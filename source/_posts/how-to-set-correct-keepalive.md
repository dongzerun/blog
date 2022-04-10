---
title: 如何正确设置连接保活
categories: go
toc: true
---

![](/images/enable-keep-alive-apache.jpg)

本文来自十年老粉`小六子`投稿，内容比较干，大家平时肯定也会遇到

由于线上存在网络问题，会导致 `GRPC HOL blocking`, 于是决定把 `GRPC client`改写成 `HTTP client`

改写 `HTTP client` 的过程还算顺利，但是搜索日志里面会发现有极少数的 `EOF` 错误

```
call xxx failed: Post "http://localhost:8080": EOF
```

`EOF` 这个东西一般是跟 `IO` 关闭有关系的，`Google` 了下相关的错误，在 `stackoverflow` 找到相关的参考

> Go by default will send requests with the header Connection: Keep-Alive and persist connections for re-use. The problem that I ran into is that the server is responding with Connection: Keep-Alive in the response header and then immediately closing the connection.

粗略看了下，问题很清晰，就是 `server` 和 `client` 的 `Keep-Alive` 机制的问题，去看下 `client` 和 `serve` 的设置参数再去调下应该就可以解决问题

## Keep-Alive parameter
### HTTP Client
线上在使用的`http.Client`的参数如下:
```Go

func main() {
	c := &http.Client{
		Transport: &http.Transport{
			MaxIdleConnsPerHost: 1,
			DialContext: (&net.Dialer{
				Timeout:   time.Second * 2,
				KeepAlive: time.Second * 60,
			}).DialContext,
			DisableKeepAlives: false,
			IdleConnTimeout:   90 * time.Second,
		},
		Timeout: time.Second * 2,
	}
	// c := &http.Client{}
	// sendRequest(c)
}
```

- Dial中的`DisableKeepAlives`为开启状态

- KeepAlive: 官方文档介绍是一个用于`TCP Keep-Alive`的`probe`指针，间隔一定的时间发送心跳包。每间隔60S进行一次`Keep-Alive`

``` Go
type Dialer struct {
...
	// KeepAlive specifies the interval between keep-alive
	// probes for an active network connection.
	// If zero, keep-alive probes are sent with a default value
	// (currently 15 seconds), if supported by the protocol and operating
	// system. Network protocols or operating systems that do
	// not support keep-alives ignore this field.
	// If negative, keep-alive probes are disabled.
	KeepAlive time.Duration
...
}
```

### HTTP Server
线上在使用的`http server`的参数
```Go
func main() {
	s := http.Server{
		Addr:        ":8080",
		Handler:     http.HandlerFunc(Index),
		ReadTimeout: 10 * time.Second,
		// IdleTimeout: 10 * time.Second,
	}
	s.SetKeepAlivesEnabled(true)
	s.ListenAndServe()
}
```

Server的 `KeepAlive` 主要是通过 `IdleTimeout` 来进行控制的，`IdleTimeout` 如果为空则使用 `ReadTimeout`

```Go
type Server struct {
...
	// IdleTimeout is the maximum amount of time to wait for the
	// next request when keep-alives are enabled. If IdleTimeout
	// is zero, the value of ReadTimeout is used. If both are
	// zero, there is no timeout.
	IdleTimeout time.Duration
...
}
```

### Debug again
可以看到，`client` 侧的 `Keep-Alive` 是60s，但是 `server` 侧的时间是间隔10s就去关掉空闲的连接。所以这里很容易就认为是：`client` 侧的 `Keep-Alive` 心跳间隔时间太长了，`server` 侧提前关闭了连接。

于是作出更改：调整 `client Keep-Alive` 为1s，这个时候感觉就不会出现 `EOF` 的错误了。

于是修改参数，重新上线，持续观察一段时间发现还是有 `EOF` 错误。看来只有进行本地复现看看究竟发生了什么

## 本地复现
### Mock EOF
在尝试复现 `EOF` 错误的时候，看到有 `Hijack` 这种东西，还是挺好用的。可以看到直接在 `server` 侧关掉连接, `client` 侧感知不到连接关闭确实是会有 `EOF` 错误发生的。
``` Go
func test(w http.ResponseWriter, r *http.Request) {
	log.Println("receive request from:", r.RemoteAddr, r.Header)
	if count%2 == 1 {
		conn, _, err := w.(http.Hijacker).Hijack()
		if err != nil {
			return
		}

		conn.Close()
		count++
		return
	}
	w.Write([]byte("ok"))
	count++
}

func main() {
	s := http.Server{
		Addr:        ":8080",
		Handler:     http.HandlerFunc(test),
		ReadTimeout: 10 * time.Second,
	}
	// s.SetKeepAlivesEnabled(false)
	s.ListenAndServe()
}

```
`EOF` 的原因知道了，在这里应该就是 `Server` 侧主动关闭了连接，至于为什么关闭连接，可以再继续往下看

### Mock Keep-Alive 
然后先在本地开始尝试复现 `Keep-Alive` 的问题，`client` 侧使用 `KeepAlive: time.Second`, 每间隔一秒钟的 `keep-alive`, `server` 侧同样使用两秒 `IdleTimeout: time.Second`。

`Client` 侧代码的 `Keep-Alive`
```go
func sendRequest(c *http.Client) {
	req, err := http.NewRequest("POST", "http://localhost:8080", nil)
	if err != nil {
		panic(err)
	}
	resp, err := c.Do(req)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	buf := &bytes.Buffer{}
	buf.ReadFrom(resp.Body)

}

func main() {
	c := &http.Client{
		Transport: &http.Transport{
			MaxIdleConnsPerHost: 1,
			DialContext: (&net.Dialer{
				Timeout:   time.Second * 2,
				KeepAlive: time.Second,
			}).DialContext,
			DisableKeepAlives: false,
			IdleConnTimeout:   90 * time.Second,
		},
		Timeout: time.Second * 2,
	}
	// c := &http.Client{}
	sendRequest(c)
	time.Sleep(time.Second * 3)
	sendRequest(c)

}
```

Server 侧的代码:
```Go
func echo(w http.ResponseWriter, r *http.Request) {
	log.Println("receive a request from:", r.RemoteAddr, r.Header)
	w.Write([]byte("ok"))
}

func main() {
	var s = http.Server{
		Addr:        ":8080",
		Handler:     http.HandlerFunc(echo),
		IdleTimeout: time.Second * 2,
	}
	s.ListenAndServe()
}

```
理论上来讲，`client`间隔一秒发送`probe`，`server`的`idle`为两秒是不会关闭连接的，但是实际却是关闭了旧的连接，重新创建了新的连接。

Server侧输出:
```
➜  http-client-server go run http-server-simple.go
2021/08/07 19:46:47 receive a request from: [::1]:53196 map[Accept-Encoding:[gzip] Content-Length:[0] User-Agent:[Go-http-client/1.1]]
2021/08/07 19:46:50 receive a request from: [::1]:53197 map[Accept-Encoding:[gzip] Content-Length:[0] User-Agent:[Go-http-client/1.1]]
```
### 抓包分析
结果有些出乎意料，因为是在本地进行代码复现的，所以去看下抓包分析结果。

![](/images/tcpdump-jiangge.jpg)s

- `Client` 使用的基于`TCP`层面的`Keep-alive`协议，针对的是整条`TCP`连接
- `Server` 侧明显是基于应用层协议做的判断

所以初步的结论就是两者的 `Keep-Alive` 是工作在不同层面，让人产生了误解。

### 源码分析
#### Client 
`Client` 侧的代码在 `net/dial.go` 里面，主要进行 `Keep-Alive` 的逻辑如下

```Go
func (d *Dialer) DialContext(ctx context.Context, network, address string) (Conn, error) {
...
	if tc, ok := c.(*TCPConn); ok && d.KeepAlive >= 0 {
		setKeepAlive(tc.fd, true)
		ka := d.KeepAlive
		if d.KeepAlive == 0 {
			ka = defaultTCPKeepAlive
		}
		setKeepAlivePeriod(tc.fd, ka)
		testHookSetKeepAlive(ka)
	}
...
}

func setKeepAlivePeriod(fd *netFD, d time.Duration) error {
	// The kernel expects seconds so round to next highest second.
	secs := int(roundDurationUp(d, time.Second))
	if err := fd.pfd.SetsockoptInt(syscall.IPPROTO_TCP, syscall.TCP_KEEPINTVL, secs); err != nil {
		return wrapSyscallError("setsockopt", err)
	}
	err := fd.pfd.SetsockoptInt(syscall.IPPROTO_TCP, syscall.TCP_KEEPIDLE, secs)
	runtime.KeepAlive(fd)
	return wrapSyscallError("setsockopt", err)
}
```

上面的代码可以看到，最后调用的是 `SetsockoptInt`，这个函数就不在这具体的展开了，本质上来讲 `Client` 侧是在TCP 4层让 `OS` 来帮忙进行的 `Keep-Alive`。

因为网络的环境是比较复杂的，有很多的请求是跨 `LB` 进行的，**比如 `AWS` 的 `ELB`之类的，所以这个 `keep-alive` 在这里也显得合理**。

#### Server
Server侧的代码在 `net/http/server.go`里:
```Go
func (c *conn) serve(ctx context.Context) {
...
    defer func() {
        if !c.hijacked() {
			c.close()
			c.setState(c.rwc, StateClosed, runHooks)
		}
    }
    
    for {
        w, err := c.readRequest(ctx)
        ...
        serverHandler{c.server}.ServeHTTP(w, w.req)
        ...
		if d := c.server.idleTimeout(); d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
		}
    }
...
}
```
简单的说明下， `defer` 就是关闭连接用的，当函数退出的时候`server`会关闭连接。

`for`循坏是处理连接请求用的，可以看出来`HTTP server`本身其实是不支持处理多个请求的，并没有实现`HTTP 1.1`协议中的`Pipeline`。

然后再看`keep-alive`的操作，先设置`ReadDeadline`，然后调用`c.bufr.Peek`这里的调用流程比较长，其实最后会落到`conn.Read`,本质上是一个阻塞操作。然后开始等待`bufr`里面的数据，如果`client`在这个时间段没有发送数据过来，则会退出`for`循环然后关闭连接

### conclusion
所以在上述的场景下想要`reuse`一个`conn`主要还是取决于`server` 侧的`idleTimeout`。如果没收到`client`发送的请求是会主动发送`fin`包进行`close`的。

### 如何修复
#### 1.Retry
其实解决方案有很多种，在这里线上采用的是客户端进行重试。这里引申一下，像上面这种错误，如果是`GET`,`HEAD`等一些幂等操作的话，`client`代码库会自动进行重试。我们线上使用的是`POST`, 所以直接在业务侧进行重试。

### 2.Increase IdleTimeout
另外一个解决方案就是增加`server`的`IdleTimeout`，但是这样一来会消耗更多的`server`资源。

### 3.Short-lived conn
还有一种方法就是短连接，这样对`server`的资源浪费就减轻了，但是不能重用连接。整体`latency`会受到影响。

### 小结
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `http 超时` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)