---
title: 小技巧！Mac 环境下编译 Go 服务
categories: 小技巧
toc: true
---

> Grab 北京(格步科技)**大量招聘 后端开发、全栈、IOS/Android、SRE**, 还有少量实习生岗位开放。有意向的请联系我 ^_^
### 背景
本篇分享来源于上午和同事的讨论。大部份工程师都使用 Mac 做为开发环境，平常 local 编译 go 代码没什么问题，偶尔需要 linux binary, 交叉编译足够了
```shell
GOOS=linux GOARCH=amd64 go build main.go
```
比如上面指定 GOOS 是 linux, GOARCH 平台是 amd64. 但还是有些场景，Mac 无法解决
1. 使用 CGO 的代码
2. 想使用 gdb 去调试

第二个场景 gdb 我还折腾过一段时间，始终无法像 linux 平台那样完美。难道无法解决了嘛？
### Docker
解决办法就是：**`Docker` 启动 ubuntu 虚拟机，然后挂载本地 `GOPATH` 目录到容器中**

让我们来看下操作细节：

![](https://gitee.com/dongzerun/images/raw/master/img/docker-reference.jpg)

安装 docker for mac 可以自行 google, 这里要注意调大 cpu 和 memory, 否则编译大型代码时内存不足。

```shell
~$ docker pull ubuntu
~$ docker create -ti --cpus 6 -m 6GB --privileged --name sextant -v /Users/zerun.dong/:/root/zerun.dong ubuntu bash -l
~$ docker start -ai sextant
```

上面命令分别是下载 ubuntu 镜像，创建名为 sextant 的容器，最后再启动

这里面 cpus m 用来设置资源，少了不够用。`/Users/zerun.dong/:/root/zerun.dong` 用于将本机目录挂载到容器中的 /root/zerun.dong 下面，`privileged` 允许容器对宿机主 root 权限

进到容器后，需要再安装 go binary, 然后设置好 GOPATH, PATH, GOROOT 后即可进行编译。成功后就会在 Mac 本机留下 linux binary, 也可直接在容器中用 gdb 进行调试，非常方便

```shell
~$ docker ps -a | grep -i ubuntu
~$ docker commit d497d0fee14d ubuntu:go
```
当然也使用 docker commit 保存刚才的容器运行时，这样下次使用就可以直接编译，省去刚才的操作步骤
### 小结
这次分享就这些，以后面还会分享更多的内容。如果感兴趣，请大家`关注`, `在看`，点击左下角的`分享`素质三连哦(:

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)