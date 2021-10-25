---
title: 实践出真知，聊聊 HTTP 鉴权那些事
categories: go
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/auth-cover.jpg)

上半年参与的项目涉及到 gateway 和 id 权限认证系统，通过系统性的学习与接触，了解很多 HTTP 鉴权的那些事。分享实践的细节，都是通用做法，符合标准协义，不涉及公司机密

本文主要讲如何给第三方服务，即 API 做鉴权，而不是用户登录系统。一般做后端微服务的很少接触这方面的概念，网关层或是入口做好认证后，链路下游都是默认开放的，最多用 iptables 或 aws securit group 类似的做网络可达的限制。用户信息传到 `Context` 中即可

如果业务需要与第三方公司 partner 合作，或是开放 API 给外部，那么需要全面了解 HTTP 鉴权的知识。本文参考了[凤凰架构](http://icyfenix.cn/summary/, "凤凰架构") 和 [HTTP API 认证授权术](https://coolshell.cn/articles/19395.html, "HTTP API 认证授权术")

### 基本概念
鉴权的本质：**用户 (`user / service`) 是否有以及如何获得权限 (`Authority`) 去操作 (`Operate`) 哪些资源 (`Resource`)**

认证（`Authentication`）：系统如何正确分辨出操作用户的真实身份？认证方式有很多种，后面会讲解

授权（`Authorization`）：系统如何控制一个用户该看到哪些数据、能操作哪些功能？授权与认证是硬币的两面

凭证（`Credential`）：系统如何保证它与用户之间的承诺是双方当时真实意图的体现，是准确、完整且不可抵赖的？我们一般把凭证代称为 `TOKEN`

保密（`Confidentiality`）：系统如何保证敏感数据无法被包括系统管理员在内的内外部人员所窃取、滥用？这里要求我们不能存储明文到 DB 中，不能将密码写到 http url 中，同时要求 id 服务公有少部份人能够访问，并且有审计

传输（`Transport Security`）：系统如何保证通过网络传输的信息无法被第三方窃听、篡改和冒充？内网无所谓了，外网一般我们都用 https

验证（`Verification`）：系统如何确保提交到每项服务中的数据是合乎规则的，不会对系统稳定性、数据一致性、正确性产生风险？

### 验证方式
根据协义要求，需要将凭证 Credential 放到 header `Authorization` 里，凭证可是是用户名密码，也可以是自定义生成的 `TOKEN`

主流验证方式有三大类：`Basic/Digest`, `HMAC` 和 `Oauth2`
#### 1.Basic/Digest
`Digest` 翻译成摘要，是 `Basic` 的加强版，放在一起讨论，完整定义参考 [RFC2617 Basic and Digest Access Authentication](https://datatracker.ietf.org/doc/html/rfc2617, "Basic Digest RFC2617")

![](https://gitee.com/dongzerun/images/raw/master/img/basic-access-authentication.jpg)

Basic 认证非常简单，Server 发现没有登录返回 401 Unauthorized 并且携带 header `WWW-Authenticate: Basic realm="User Visible Realm"`

Client 用户需要将 `user:password` 用户名秘密用分号 : 组合在一起，然后求 Base64 编码，放到 header 中发送给服务端 

比如 user 是 Aladdin, 密码是 open sesame, 此时将 (Aladdin:open sesame) 求 Base64 得到 header `Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==`

但 Base64 只是编码方式而己，和明文一样没有区别。所以引出了摘要算法， `Digest` 认证把用户名和密码加盐（一个被称为 Nonce 的变化值作为盐值）后再通过 MD5/SHA 等哈希算法取摘要发送出去

但是这种认证方式依然是不安全的，无论客户端使用何种加密算法加密，无论是否采用了 Nonce 这样的动态盐值去抵御重放和冒认，遇到中间人攻击时依然存在显著的安全风险

公司内网默认可信的，可以用 `Basic/Digest` 随变搞搞，但也就局限于此了
#### 2.HMAC
`HMAC` [hash-based message authentication code](https://en.wikipedia.org/wiki/HMAC, "hash-based message authentication code") 是市面上使用最广泛的认证技术，源自于 [message authentication code](https://en.wikipedia.org/wiki/Message_authentication_code, "message authentication code"), 主要用于消息签名，防止被第三方修改

![](https://gitee.com/dongzerun/images/raw/master/img/message-authentication-code.jpg)

使用 HMAC 需求提前生成 app key 与 access secret key 给第三方，国内习惯称之为 `AK/SK`, 我司叫做 `partner-id/secret`, 叫什么不重要

`AK/SK` 是需要由 server 提前生成给第三方客户端. client 获取待签名的消息，使用 SK 加密后，将签名一起发送到服务。Server 用同样的签名算法 + SK 加密息息，如果一致说明访问有效

```go
	stringToSign := verb +
		newLineBreak +

		contentType +
		newLineBreak +

		date +
		newLineBreak +

		path +
		newLineBreak +

		contentDigest +
		newLineBreak +

		//optional, if no customised header, this part will be left blank.
		partnerTokenComponent +

		//optional, if no customised header, this part will be left blank.
		customisedHeaderComponent
```
如何构建消息呢？业界没有统一标准，但一般都是将 `metyhod`, `path`, `date`, `content-type`, `body` 柔和在一起, 其中 `date` 起到防止重放攻击的作用。有时我们可以不把 `body` 放到消息中，有时还要把 http params 排序放到消息体中

```go
func ComputeBase64EncodedHMACSHA256Signature(message string, secret string) (string, error) {
	hash := hmac.New(sha256.New, []byte(secret))

	if _, err := hash.Write([]byte(message)); err != nil {
		return "", err
	}

	val := base64.StdEncoding.EncodeToring(hash.Sum(nil))

	return val, nil
}
```
至于签名算法，我们一般用 `SHA256`，同时需要传入 secret 密钥

感兴趣的可以参考 [aws s3](https://czak.pl/2015/09/15/s3-rest-api-with-curl.html, "s3 rest api with hmac") 的玩法，原理是一样的

我司正在废弃遗留的 `HMAC` 认证方式，改用统一，流程更规范的 `Two-Legged` Oauth2 Client 认证模式
#### 3.Oauth2
[Oauth2](https://datatracker.ietf.org/doc/html/rfc6749, "Oauth2 RFC6749") 是业界标准的授权协议，专注于客户端开发人员的简单性，同时为网络应用、桌面应用、手机和客厅设备提供特定的授权流程。

![](https://gitee.com/dongzerun/images/raw/master/img/oauth2.jpg)

主要有以下四种模式：

* 授权码模式（Authorization Code）
* 隐式授权模式（Implicit）
* 密码模式（Resource Owner Password Credentials）
* 客户端模式（Client Credentials）

其中最常用的是 `Authorization Code` 与 `Client Credentials` 模式，Oauth2 涉及三个角色 `owner` 所有者(一般指人), `server` 服务端(一般指资源服务器和资源认证服务器), `client` 第三方客户端，所以第一种的 `Authorization Code` 称之为 **Three-Legged** 三腿模式

而 `Client Credentials` 其实更像是给服务端做认证，只有 `client`, `server` 所以称之为 **Two-Legged** 两腿模式，我司就使用这种做 API 鉴权

![](https://gitee.com/dongzerun/images/raw/master/img/oauth2-client.jpg)

客户端模式非常简单，`server` 提前创建好 ClientID&ClientSecret 给第三方服务，然后第三方通过 ClientID&ClientSecret 调用 `/oauth2/token` 接口生成 token, 后续所有访问携带这个 token 即可，每次由 id 服务了调用 `/oauth2/verify` 去验证

举个测试的例子：
```shell
curl -X POST https://xxxxxxxx/oauth2/token -H 'Content-Type: application/x-www-form-urlencoded' -d 'client_id=c45594c80f3342ed93eea95ad4ab18bc&grant_type=client_credentials&client_secret=5aL1LOhXHi6X7E2r&scope=test.scope'

{"access_token":"this is jwt token","token_type":"Bearer","expires_in":107999}
```

```shell
curl -i https://xxxxxxxx/v1/styles/dark.json -H 'Authorization: Bearer this-is-a-jwt-token'
```
指定 grant_type 为 `client_credentials` 同时需要填写 scope, 非常重要，用于控制该 user 能访问哪些资源

![](https://gitee.com/dongzerun/images/raw/master/img/oauth2-code.jpg)

和两腿的一样，开始进行授权过程以前，第三方应用先要到授权服务器上进行注册，所谓注册，是指向认证服务器提供一个域名地址（用于回调），然后从授权服务器中获取 ClientID 和 ClientSecret

这里有几个概念：
* 资源所有者 owner: 一般是指我们用户
* 操作代理 delelegate: 一般指浏览器 chrome edge ...
* 资源服务器：比如微信，比如新浪等等
* 授权服务器：用于鉴权的，有时和资源服务器是一台
* 第三方应用：比如一些小程序，想访问我的微信头像等等

授权过程如下：

1. 第三方应用将资源所有者（用户）导向授权服务器的授权页面，并向授权服务器提供 ClientID 及用户同意授权后的回调 URI，这是一次客户端页面转向

2. 授权服务器根据 ClientID 确认第三方应用的身份，用户在授权服务器中决定是否同意向该身份的应用进行授权，用户认证的过程未定义在此步骤中，在此之前应该已经完成

3. 如果用户同意授权，授权服务器将转向第三方应用在第 1 步调用中提供的回调 URI，并附带上一个授权码和获取令牌的地址作为参数，这是第二次客户端页面转向

4. 第三方应用通过回调地址收到授权码，然后将授权码与自己的 ClientSecret 一起作为参数，通过服务器向授权服务器提供的获取令牌的服务地址发起请求，换取令牌。该服务器的地址应与注册时提供的域名处于同一个域中

5. 授权服务器核对授权码和 ClientSecret，确认无误后，向第三方应用授予令牌。令牌可以是一个或者两个，其中必定要有的是访问令牌（Access Token），可选的是刷新令牌（Refresh Token）。访问令牌用于到资源服务器获取资源，有效期较短，刷新令牌用于在访问令牌失效后重新获取，有效期较长

6. 资源服务器根据访问令牌所允许的权限，向第三方应用提供资源。

理解 Oauth2 时一定要明确，哪些是通过浏览器方问的，哪些是第三方服务器直接与授权服务器关互，还要注入两次页面转向。建义看一下 github oauth2 或者微信的开发文档

### JSON Web Tokens
上面是三种主流的验证方式，其实 Oauth2 只规定了大致框架，并没有规定 token 如何生成。我们一般使用 [JWT](https://jwt.io/, "JWT") 开放的标准(RFC 7519), 它定义了一种紧凑和独立的方式，以 JSON 对象的形式在各方之间安全地传输信息。这种信息可以被验证和信任，因为它是经过数字签名的，一般我们使用 RSA private key 进行签名。

JWT 格式由三段组成，**header.payload.signature**, 让我们看个例子

![](https://gitee.com/dongzerun/images/raw/master/img/jwt-test-case.jpg)

header 头表示用什么算法进行签名，payload 是一堆 json 可以处定义，但一些公用的己经定义好，signature 是签名

```shell
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  your-256-bit-secret
)
```
大致算法也比较简单，`your-256-bit-secret` 是你的密钥，相同算法生成的 signature 一致，说明 JWT 没有被篡改。一般都用 rsa private 去加密，然后用 public key 解密，好处是可以把 public key 放到 cdn lambda 去验证请求是否有效。如果用对称算法，就做不到这样的效果

```
{
  "alg": "RS256",
  "kid": "_default",
  "typ": "JWT"
}
```
我司对 header/payload 都进行了定制，比如 Header 增加 `kid` 字段，代表是用哪个 rsa key pair 进行的加密，可以用于 rotate rsa 密钥，非常方便

```
{
  "aud": "c45594cXXXXXXXXXadb18bc",
  "exp": 1635263037,
  "iat": 1635155037,
  "iss": "https://idp.grab.com",
  "jti": "oD-VHixWQXXXXXXXXXXXAR6ctYg",
  "nbf": 1635154857,
  "pid": "e0923XXXXXXXXXX943690d4b",
  "pst": 1,
  "scp": "[\"7c14974d3d0e462bXXXXXXXXXXXXXXXbf7bc6\"]",
  "sub": "TWO_LEGGED_OAUTH",
  "svc": "",
  "tk_type": "access"
}
```
payload 里面 iss 指搬发者信息，exp 是指 JWT 过期时间，如果允许篡改，那就可以无限续约了，同时增加了 partner-id, scope, two-legged 等等输助信息。**其实 TOKEN 用什么实现无所谓，能防止篡改的都可以** token 搬发出去就很难控制，为了防止重放攻击，一般要对 token 做控制

### 稳定性
以前滴滴出过很多 ID 鉴权服务故障，原因五花八门，现在回头看，都是稳定性建设不到位

* uuid 生成，通过 shell exec 做系统调用，网络抖动时，司机频繁登入登出，系统负载增加，直接打挂服务

* token 相关的接口没有 ratelimter, 直接打挂 DB

其它的想不起来了，stability 永远的难题

### 杂谈
1. RSA public private 在 DB 不能存储明文，要用 vault 或是 kms 加密

2. Base64 是为了传输方便，省去空格特殊字符等等，但仍然是明文。hash 才是为了加密

3. RSA 公钥加密是不想让别人看到内容，因为只有私钥才能解开。私钥加密是为了传递数据，不想让别人篡改

4. JWT TOKEN 能防篡改但是不能防重放攻击，所以 exp 要短，同时要有 token 黑名单，还得有限流，哪怕是一小时也能把服务打爆

5. TOKEN 是否存储 DB 呢？存有好处，也可以选择不存

6. RSA 加密 Encrypt 和摘要 Digest 的区别，前者可逆，后者不可逆

7. JWT payload 自定义内容不易过多，一般 http header 都是有大小限制的

8. 三个概念：编码 Base64Encode、签名 HMAC、加密 RSA。编码是为了更的传输，等同于明文，签名是为了信息不能被篡改，加密是为了不让别人看到是什么信息

9. 本文不涉及 TLS, 历史上很多鉴权的方案都是为了应对没有 TLS 的情况，外网基本都是 https, 所以很多方案现在己经不适合

10. Scope 非常重要，基于 least privilege 原则，只允许最小访问权限，一定要控制第三方能访问的资源

11. 运维能力，ID 服务一般访问受限，只有特定服务或是 admin header 才能访问，需要提供 bot 或是网页运维能力

12. 密钥需要定期 rotate, 业务代码当然也要适配

### 小结
非安全及 web authentication 专业，如果有描述错误的请大家指出~

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `HTTP 鉴权` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)


