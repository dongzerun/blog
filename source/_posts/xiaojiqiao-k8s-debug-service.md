---
title: 小技巧！k8s 环境下调试服务
categories: 小技巧
toc: true
---

![](/images/GopherKubernetes-debug-cover.png)

本文面向初次调试 k8s 服务的新手及运维，老鸟可以跳过啦~ 但也需要了解 k8s, 比如至少知道 `service`, `endpoint`, `pod`, `node` 这些基本概念

前两年开始接触学习 k8s, 一直有纸上谈兵的感觉。最近恰好项目需要，服务要整体迁移到 aws k8s 平台，实践中才发现原来很多地方理解不到位

**比如说如何调试 k8s 里的服务呢？** 服务对外暴露了 `service`, 要查看 endpoints(就是后端的 real server) 是否挂载成功，如果没有 endpoints 那就要用 `kubectl logs` 或是 `kubectl describe` 查看服务 pod 是否启动成功。可以参考官网[应用故障排查](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/debug-application/, "应用故障排查")

![](/images/service-endpoints.jpg)

Pod 启不来的原因很多：镜像 pull 失败(墙内), 资源不足无法调度，liveness 检查失败，服务自身 panic 等等一大堆 ...
### 专用 pod
由于 k8s 内部网络和物理机不在一个网段，如果你的服务没有挂到 external lb 上面，那就需要**创建专用的调试 pod, 然后进到 k8s 网络里**
```shell
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
    image: ubuntu:18.04
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```
比如这里创建了名称是 dnsutils 的 pod, 永久 sleep
```shell
zerun.dong$ kubectl exec -i -t dnsutils -- /bin/bash
root@dnsutils:/# cat /etc/resolv.conf
nameserver 172.20.0.10
search default.svc.cluster.local svc.cluster.local cluster.local

root@dnsutils:/# curl -i http://service-name.namespace-name/xx/aa/bb/cc
HTTP/1.1 405 Method Not Allowed
Date: Tue, 22 Jun 2021 01:57:07 GMT
Content-Length: 0
```
比如这里使用 `/bin/bash` 进到调试 pod 里，然后 curl 调用我服务的地址 `http://service-name.namespace-name/xx/aa/bb/cc`。

这里要注意 `service-name.namespace-name` 是短域名，完整的应该是 `service-name.namespace-name.svc.cluster.local`. 也可以直接用 container ip 进行调试

**另外 k8s 也支持使用 `kubectl debug` 命令启动调试 pod, 也非常方便。总之有权限啥都好说，没权限干瞪眼** ...
### 登录 node
另外，如果有登录宿主机的权限，也可以使用 `nsenter` 进行调试

原理就是**用 `nsenter` attach 到目标容器的 namespaces 中，一般我们都是进到 net ns**
```shell
 ~]# ps aux | grep -i envoy
root     11023  0.0  0.0 119416   980 pts/0    S+   01:51   0:00 grep --color=auto -i envoy
root     13808  0.4  0.4 373848 67248 ?        Ssl  Jun21   3:59 envoy -c /etc/envoy/envoy.yaml --log-format
```
```shell
~]# nsenter -u -i -n -p -t 13808 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether 42:ce:9b:ce:c5:4a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.209.132/32 scope global eth0
       valid_lft forever preferred_lft forever
```
```shell
~]# nsenter -u -i -n -p -t 13808 curl -i http://127.0.0.1:8081/help
HTTP/1.1 200 OK
content-type: text/plain; charset=UTF-8
cache-control: no-cache, max-age=0
x-content-type-options: nosniff
date: Tue, 22 Jun 2021 01:52:43 GMT
server: envoy
        "domains": [
         "xxxx.domain.name"
        ],
        "routes": [
```
比如上面的例子，13808 是 envoy 在宿主机上的进程 id, 有些端口并没有暴露给缩主机，或是 lb, 只能进到 net ns 里调试
### kt-connect
阿里以前开源了一个 [kt-connect](https://github.com/alibaba/kt-connect, "kt-connect") 项目。宣称是研发侧利器，本地连通 Kubernetes 集群内网。不过一年多没有更新，猜测又是 kpi 式开源？还是项目移植走了？

理念超级棒。可以将 k8s 流量导到开发机本地，也能将本地服务暴露到 k8s 中。我们目前没有采用，现在仍然是每次修改都要 deploy 到 dev k8s 环境中

![](/images/kt-connect.jpg)

上面是架构图，可以参考[云原生环境下的开发测试](https://zhuanlan.zhihu.com/p/144273459, "云原生环境下的开发测试")。没啥黑科技，就是在集群内部设置代理影子容器，负责转发流量，有时间这块再写一篇分享
### 小结
**今天的分享就这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连**

[官方也有相关 blog](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/debug-running-pod/, "调试运行中的 Pod"), 可以参考。关于调试 k8s 服务大家有什么看法，欢迎留言一起讨论 ^_^

![](/images/dongzerun-weixin-code.png)