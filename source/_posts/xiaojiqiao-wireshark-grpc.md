---
title: 小技巧！Wireshark 让调试 GRPC 不再困难
categories: 小技巧
toc: true
---

以前用 wireshark 分析过 GRPC 流量，非常方便，年初用同样方法分析了HOL blocking 问题，感兴趣的可以看看。今天记录下全过程，分享给大家，贼好用^^

```shell
ssh root@some.host 'tcpdump -i eth0 port 80 -s 0 -l -w -' | wireshark -k -i -
```

还有一种骚操作是 ssh 实时用 wireshark 解析，好处是不占用磁盘空间，但不是所有人都有权限

以前用 [wireshark](https://www.wireshark.org/, "wireshark") 分析过 `GRPC` 流量，非常方便。今天记录下全过程，分享给大家，贼好用^^

### tcpdump
```shell
tcpdump -i eth0 -w tcpdump.log
```
上面是直接 dump 整个网卡的流量，如果太大的话，可以只 dump 固定 ip 或端口的

我们线上有脚本，可以整天 dump 数据，然后按文件大小进行切割。大家可以自己写，还蛮方便的

### 配置 wireshark
因为 `GRPC` 是在 `http2` 之上运行的，协议是 `protobuf`, 所以需要加载 pb 文件，否则 wireshark 无法识别自定义内容

另外，如果走了 `tls` 加密，还需要在 `wireshark` 上加载 rsa key 解密流量

打开 `Wireshark->Preference->Protocols->Protobuf`

![](https://gitee.com/dongzerun/images/raw/master/img/protobuf.jpg)

然后打开 `Edit`, 输入本次测试用的 proto 文件

![](https://gitee.com/dongzerun/images/raw/master/img/protobuf-source-path.jpg)

`proto` 文件可能引用其它 pb 文件，所以也需要填写搜索路径，然后`确定` 这就配置完成

### 解析 GRPC
打开 tcpdump.log 数据文件以后，打开 `Wireshark->Analyze->Decode As`

![](https://gitee.com/dongzerun/images/raw/master/img/decode-as.jpg)

如上所示，因为我要解析 10177 http2, 添加后确定

![](https://gitee.com/dongzerun/images/raw/master/img/tcp-http2.jpg)

这时会发现，己经能看到 http2 包数据了。如果你的数据是加密的，记得配置 tls
### 过滤
Wireshark 非常强大，可以根据 http2 header 来过滤，由可以跟据 body 来过滤，很方便

![](https://gitee.com/dongzerun/images/raw/master/img/header-filter.jpg)

如上图，可以看到解析出了业务 endpoints, header 以由 request 内容。撒花 ~~
### 小结
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `调试 GRPC` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)