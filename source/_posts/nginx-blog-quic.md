---
title: 基于 nginx quic 分支体验 http3 
categories: quic
toc: true
---
![](https://gitee.com/dongzerun/images/raw/master/img/quic-cover.jpg)

去年发过文章 [HOL blocking 困扰两个月的问题](https://mp.weixin.qq.com/s/pILicxg81p3FVT07MCfJNw), http2 通过多路复用虚拟 stream 最大化的利用了 tcp 连接的性能，并且解决了七层的 HOL 问题，但是没有解决四层 tcp 的 HOL, 如果有丢包，那么同一 tcp 上的所有业务都会产生影响，QPS 高的时候非常明显

而 [QUIC](https://www.chromium.org/quic/, "chromium quic") 解决了这个问题，底层基于 UDP. 本文提到的 http3 也是基于 google 的 quic, IETF 去年发布了完整版的 [RFC9000](https://datatracker.ietf.org/doc/html/rfc9000, "HTTP3 RFC9000")

[how-facebook-is-bringing-quic-to-billions](https://engineering.fb.com/2020/10/21/networking-traffic/how-facebook-is-bringing-quic-to-billions/, "how-facebook-is-bringing-quic-to-billions") 提到了性能优化，**整体来讲弱网络下，用户体验提升非常明显**，但是有一些基于 tcp 做的优化就不再适用了，需要重新审视一下。阿里也开源了自己的 [XQUIC](https://mp.weixin.qq.com/s/RdR-7hPfY3tckHt3c3bw7Q, "阿里正式开源自研 XQUIC"), 手淘大规模应用，网络耗时降低超 20%

性能测试也可以参考 [HTTP3/QUIC 性能测试与配套组件](https://segmentfault.com/a/1190000040394845, "HTTP3/QUIC 性能测试与配套组件") 和 [measuring-quic-vs-tcp-mobile-desktop](https://blog.apnic.net/2018/01/29/measuring-quic-vs-tcp-mobile-desktop/, "measuring-quic-vs-tcp-mobile-desktop")

### 背景
微信群有朋友将博客升级到了 http3, 咨询一翻后我也升级了，基于 nginx 的 quic branch, 重新编译即可，参数改动非常少，这里得提一下芮神的 xiaorui.cc 到现在都没有 tls... 感兴趣的可以看看 `https://mytechshares.com`

### 编译 H3
整体参考 [nginx quic roadmap](https://www.nginx.com/blog/our-roadmap-quic-http-3-support-nginx/, "nginx quic roadmap") 的 dockerfile

nginx quic 使用 boringssl, 无法科学上网的单独下载吧
```shell
# apt-get install -y git gcc make g++ cmake perl libunwind-dev golang
# git clone https://boringssl.googlesource.com/boringssl
# mkdir boringssl/build
# cd boringssl/build
# cmake ..
# make
```
nginx 源码并没有用 github 管理，需要下载 `hg`

```shell
# apt-get install -y mercurial libperl-dev libpcre3-dev zlib1g-dev libxslt1-dev libgd-ocaml-dev libgeoip-dev
# hg clone https://hg.nginx.org/nginx-quic
```
编译也比较简单，没啥特殊的
```shell
./auto/configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_realip_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt=-I../boringssl/include --with-ld-opt='-L../boringssl/build/ssl -L../boringssl/build/crypto' --with-http_v3_module  --with-stream_quic_module
```
重点是几个 http3 相关的
```shell
--with-ld-opt='-L../boringssl/build/ssl -L../boringssl/build/crypto' --with-http_v3_module  --with-stream_quic_module
```
make 完成后将 `nginx-quic/objs/nginx` 命令复制到 /usr/sbin 即可
```shell
# nginx -V
nginx version: nginx/1.21.6
built by gcc 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)
built with OpenSSL 1.1.1 (compatible; BoringSSL) (running with BoringSSL)
TLS SNI support enabled
......
ld-opt='-L../boringssl/build/ssl -L../boringssl/build/crypto' --with-http_v3_module --with-stream_quic_module
```
查看是编译成功及参数
### 配置 H3
只需要在 nginx.conf 中添加下面四行即可，如果有多个 virtual server, 根据自己的情况调整
```
listen 443 http3 reuseport;
listen 443 ssl http2;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
add_header Alt-Svc 'h3=":443"; ma=86400, h3-29=":443"; ma=86400';
```
使用 `nginx -T` 检测配置无后，重启 nginx 即可
```shell
# ss -anlp | grep -i nginx
udp               UNCONN              0                    0                                                                                 0.0.0.0:443                                           0.0.0.0:*                                     users:(("nginx",pid=30595,fd=10),("nginx",pid=30594,fd=10),("nginx",pid=30593,fd=10))
udp               UNCONN              0                    0                                                                                 0.0.0.0:443                                           0.0.0.0:*                                     users:(("nginx",pid=30595,fd=6),("nginx",pid=30594,fd=6),("nginx",pid=30593,fd=6))
tcp               LISTEN              0                    128                                                                               0.0.0.0:443                                           0.0.0.0:*                                     users:(("nginx",pid=30595,fd=7),("nginx",pid=30594,fd=7),("nginx",pid=30593,fd=7))
tcp               LISTEN              0                    128                                                                               0.0.0.0:80                                            0.0.0.0:*                                     users:(("nginx",pid=30595,fd=8),("nginx",pid=30594,fd=8),("nginx",pid=30593,fd=8))
tcp               LISTEN              0                    128                                                                                  [::]:80                                               [::]:*                                     users:(("nginx",pid=30595,fd=9),("nginx",pid=30594,fd=9),("nginx",pid=30593,fd=9))
```
可以看到我们同时支持了 udp/tcp 的 443 和 80
### 开放防火墙
注意 iptables 是否开放了 udp 443, 以及云厂商的安全组规则

![](https://gitee.com/dongzerun/images/raw/master/img/udp443-aliyun.jpg)

我这里需要开通阿里云的
### 浏览器支持
对于 chrome 需要打开 `chrome://flags/`, 然后 enable QUIC

![](https://gitee.com/dongzerun/images/raw/master/img/chrome-quic-enable.jpg)
### 测试 H3
可以通过网站 `https://http3check.net` 来测试你的 blog 是否己支持 http3, 否则可能退化到 h2 或是 h1

![](https://gitee.com/dongzerun/images/raw/master/img/http3-check.jpg)

最后打开 chrome 或是 firefox 测试完成 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/blog-nginx-http3.jpg)
### 小结
想了解 quic 协议的可以参考 [a-quick-look-at-quic](https://blog.apnic.net/2019/03/04/a-quick-look-at-quic/, "a-quick-look-at-quic") 和 [Everything You Need to Know About QUIC and HTTP3](https://www.youtube.com/watch?v=_QQX0Ezpq8U, "Everything You Need to Know About QUIC and HTTP3")

不是很清楚我司移动端是否使用 quic, 明天去问问，可能工具类的 APP 不如 facebook/youtube 对用户体验要求高。IDC 内网暂时没升级 quic 的痛点，但也值得关注

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `HTTP3` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)