---
title: 什么是 Pause 容器
categories: k8s
toc: true
---

都知道 k8s 的调度最小单位是 POD, 并且每个 POD 都有一个所谓的 Infra 容器 `Pause`, 那么到底什么是 `Pause` 容器呢？长什么样？有什么做用？

### 分析源码
废话不多，直接上源码，来自官方 [pause.c](https://github.com/kubernetes/kubernetes/blob/master/build/pause/linux/pause.c, "pause.c")
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#define STRINGIFY(x) #x
#define VERSION_STRING(x) STRINGIFY(x)

#ifndef VERSION
#define VERSION HEAD
#endif

static void sigdown(int signo) {
  psignal(signo, "Shutting down, got signal");
  exit(0);
}

static void sigreap(int signo) {
  while (waitpid(-1, NULL, WNOHANG) > 0)
    ;
}

int main(int argc, char **argv) {
  int i;
  for (i = 1; i < argc; ++i) {
    if (!strcasecmp(argv[i], "-v")) {
      printf("pause.c %s\n", VERSION_STRING(VERSION));
      return 0;
    }
  }

  if (getpid() != 1)
    /* Not an error because pause sees use outside of infra containers. */
    fprintf(stderr, "Warning: pause should be the first process\n");

  if (sigaction(SIGINT, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 1;
  if (sigaction(SIGTERM, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 2;
  if (sigaction(SIGCHLD, &(struct sigaction){.sa_handler = sigreap,
                                             .sa_flags = SA_NOCLDSTOP},
                NULL) < 0)
    return 3;

  for (;;)
    pause();
  fprintf(stderr, "Error: infinite loop terminated\n");
  return 42;
}
```
可以看到 `Pause` 容器做如下两件事情：

1. 注册各种信号处理函数，主要处理两类信息：退出信号和 child 信号。收到 `SIGINT` 或是 `SIGTERM` 后，直接退出。收到 `SIGCHLD` 信号，调用 `waitpid`, 回收退出进程
2. 主进程 for 循环调用 `pause()` 函数，使进程进入休眠状态，直到被终止或是收到信号

### 可疑的 waitpid
还是 c 的基础不够扎实，一直以为 `waitpid` 是父进程等待回收退出的子进程，但是真的这样嘛？
```shell
zerun.dong$ man waitpid
WAIT(2)                     BSD System Calls Manual                    WAIT(2)

NAME
     wait, wait3, wait4, waitpid -- wait for process termination

SYNOPSIS
     #include <sys/wait.h>
```
在 mac 上查看 man 手册，`wait for process termination` 也确实这么写的。登到 ubuntu 18.04 查看一下
```shell
:~# man waitpid
WAIT(2)                                                      Linux Programmer's Manual                                                      WAIT(2)

NAME
       wait, waitpid, waitid - wait for process to change state
```
对于 linux man 手册，就变成了 `wait for process to change state
` 等待进程的状态变更！！！
```shell
All of these system calls are used to wait for state changes in a child of the calling process, and obtain information about the child whose
state has changed.  A state change is considered to be: the child terminated; the child was stopped by a signal; or the child was resumed by
a  signal.   In the case of a terminated child, performing a wait allows the system to release the resources associated with the child; if a
wait is not performed, then the terminated child remains in a "zombie" state (see NOTES below).
```

并且还很贴心的提供了测试代码
```c
#include <sys/wait.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char *argv[])
{
   pid_t cpid, w;
   int wstatus;

   cpid = fork();
   if (cpid == -1) {
       perror("fork");
       exit(EXIT_FAILURE);
   }

   if (cpid == 0) {            /* Code executed by child */
       printf("Child PID is %ld\n", (long) getpid());
       if (argc == 1)
           pause();                    /* Wait for signals */
       _exit(atoi(argv[1]));

   } else {                    /* Code executed by parent */
       do {
           w = waitpid(cpid, &wstatus, WUNTRACED | WCONTINUED);
           if (w == -1) {
               perror("waitpid");
               exit(EXIT_FAILURE);
           }

           if (WIFEXITED(wstatus)) {
               printf("exited, status=%d\n", WEXITSTATUS(wstatus));
           } else if (WIFSIGNALED(wstatus)) {
               printf("killed by signal %d\n", WTERMSIG(wstatus));
           } else if (WIFSTOPPED(wstatus)) {
               printf("stopped by signal %d\n", WSTOPSIG(wstatus));
           } else if (WIFCONTINUED(wstatus)) {
               printf("continued\n");
           }
       } while (!WIFEXITED(wstatus) && !WIFSIGNALED(wstatus));
       exit(EXIT_SUCCESS);
   }
}
```
子进程一直处于 pause 状态，而父进程则 waitpid 阻塞等待子进程状态变更。让我们开启一个 session 运行代码，另外一个 session 发送信号
```shell
~$ ./a.out
Child PID is 70718
stopped by signal 19

continued
stopped by signal 19
continued
^C
```
```shell
~# ps aux | grep a.out
zerun.d+   70717  0.0  0.0   4512   744 pts/0    S+   06:48   0:00 ./a.out
zerun.d+   70718  0.0  0.0   4512    72 pts/0    S+   06:48   0:00 ./a.out
root       71155  0.0  0.0  16152  1060 pts/1    S+   06:49   0:00 grep --color=auto a.out
~#
~# kill -STOP 70718
~#
~# ps aux | grep a.out
zerun.d+   70717  0.0  0.0   4512   744 pts/0    S+   06:48   0:00 ./a.out
zerun.d+   70718  0.0  0.0   4512    72 pts/0    T+   06:48   0:00 ./a.out
root       71173  0.0  0.0  16152  1060 pts/1    S+   06:49   0:00 grep --color=auto a.out
~#
~# kill -CONT 70718
~#
~# ps aux | grep a.out
zerun.d+   70717  0.0  0.0   4512   744 pts/0    S+   06:48   0:00 ./a.out
zerun.d+   70718  0.0  0.0   4512    72 pts/0    S+   06:48   0:00 ./a.out
root       71296  0.0  0.0  16152  1056 pts/1    R+   06:49   0:00 grep --color=auto a.out
```
通过向子进程发送信号 `STOP` `CONT` 来控制进程。

看来不同系统，同名 c 函数形为是不太一样的。是我大惊小怪了，就是菜:(
### 共享哪些 NS
一般提起 POD 就知道，同一个 POD 内的容器如果互相访问，只需调用 localhost 即可。如果把 k8s 集群想象成分布式操作系统，那么 POD 就是进程组的概念，一定要共享某些东西的，那么默认共享哪些 namespace 呢？

使用 minikube 搭建环境，先看一下 POD 定义文件
```shell
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE
    stdin: true
    tty: true
```
从 1.17 开始有参数 `shareProcessNamespace` 用来控制是否 POD 内共享 PID namespace, 1.18 之后默认是 false 的，如果有需求需要填写该字段。

```shell
~$ kubectl attach -it nginx -c shell
If you don't see a command prompt, try pressing enter.
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   41 101       0:00 nginx: worker process
   42 root      0:00 sh
   49 root      0:00 ps aux
```
attach 到 shell 容器中，可以看到该 POD 内所有进程，并且只有 `pause` 容器是 init 1 进程。

```shell
/ # kill -HUP 8
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   42 root      0:00 sh
   50 101       0:00 nginx: worker process
   51 root      0:00 ps aux
```
测试给 nginx master 发送 HUP 信号，子进程重启。

如果不共享 PID ns, 那么每个容器内的进程 pid 都是 init 1 进程。共享 PID ns 好处是什么呢？参考[这篇文章](https://kubernetes.io/zh/docs/tasks/configure-pod-container/share-process-namespace/, "share proceess namespace")

1. `容器进程不再具有 PID 1`。 在没有 PID 1 的情况下，一些容器镜像拒绝启动（例如，使用 systemd 的容器)，或者拒绝执行 kill -HUP 1 之类的命令来通知容器进程。在具有共享进程命名空间的 pod 中，kill -HUP 1 将通知 pod 沙箱（在上面的例子中是 /pause）。
2. `进程对 pod 中的其他容器可见`。 这包括 /proc 中可见的所有信息，例如作为参数或环境变量传递的密码。这些仅受常规 Unix 权限的保护。
3. `容器文件系统通过 /proc/$pid/root 链接对 pod 中的其他容器可见`。 这使调试更加容易，但也意味着文件系统安全性只受文件系统权限的保护。

在宿主机查看 nginx, sh 的进程 id, 通过 /proc/pid/ns 查看 namespace id
```shell
~# ls -l /proc/140756/ns
total 0
lrwxrwxrwx 1 root root 0 May  6 09:08 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 May  6 09:08 ipc -> 'ipc:[4026532497]'
lrwxrwxrwx 1 root root 0 May  6 09:08 mnt -> 'mnt:[4026532561]'
lrwxrwxrwx 1 root root 0 May  6 09:08 net -> 'net:[4026532500]'
lrwxrwxrwx 1 root root 0 May  6 09:08 pid -> 'pid:[4026532498]'
lrwxrwxrwx 1 root root 0 May  6 09:08 pid_for_children -> 'pid:[4026532498]'
lrwxrwxrwx 1 root root 0 May  6 09:08 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 May  6 09:08 uts -> 'uts:[4026532562]'
~# ls -l /proc/140879/ns
total 0
lrwxrwxrwx 1 root root 0 May  6 09:08 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 May  6 09:08 ipc -> 'ipc:[4026532497]'
lrwxrwxrwx 1 root root 0 May  6 09:08 mnt -> 'mnt:[4026532563]'
lrwxrwxrwx 1 root root 0 May  6 09:08 net -> 'net:[4026532500]'
lrwxrwxrwx 1 root root 0 May  6 09:08 pid -> 'pid:[4026532498]'
lrwxrwxrwx 1 root root 0 May  6 09:08 pid_for_children -> 'pid:[4026532498]'
lrwxrwxrwx 1 root root 0 May  6 09:08 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 May  6 09:08 uts -> 'uts:[4026532564]'
```
可以看到这里共享了 cgroup, ipc, net, pid, user. 这里仅限于测试案例。
### 杀掉 Pause 容器
测试一下杀掉 `Pause` 容器的话，k8s 是如何处理 POD. 使用 minikube 搭建环境，先看一下 POD 定义文件
```shell
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: false
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE
    stdin: true
    tty: true
```
启动后，查看 pause 进程 id, 然后杀掉
```shell
~$ kubectl describe pod nginx
......
Events:
  Type    Reason          Age                   From     Message
  ----    ------          ----                  ----     -------
  Normal  SandboxChanged  3m1s (x2 over 155m)   kubelet  Pod sandbox changed, it will be killed and re-created.
  Normal  Killing         3m1s (x2 over 155m)   kubelet  Stopping container nginx
  Normal  Killing         3m1s (x2 over 155m)   kubelet  Stopping container shell
  Normal  Pulling         2m31s (x3 over 156m)  kubelet  Pulling image "nginx"
  Normal  Pulling         2m28s (x3 over 156m)  kubelet  Pulling image "busybox"
  Normal  Created         2m28s (x3 over 156m)  kubelet  Created container nginx
  Normal  Started         2m28s (x3 over 156m)  kubelet  Started container nginx
  Normal  Pulled          2m28s                 kubelet  Successfully pulled image "nginx" in 2.796081224s
  Normal  Created         2m25s (x3 over 156m)  kubelet  Created container shell
  Normal  Started         2m25s (x3 over 156m)  kubelet  Started container shell
  Normal  Pulled          2m25s                 kubelet  Successfully pulled image "busybox" in 2.856292466s
```
k8s 检测到 `pause` 容器状态异常，就会重启该 `POD`, 其实也不难理解，无论是否共享 PID namespace, `infra` 容器退出了，`POD` 必然要重启，毕竟生命周期是与 `infra` 容器一致的。
### 小结
这次分享就这些，以后面还会分享更多的内容，如果感兴趣，可以关注并点击左下角的`分享`转发哦(:

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)