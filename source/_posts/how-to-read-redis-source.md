---
title: 如何阅读 redis 源码
categories: redis
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/redis-cover.jpg)

有的网友想要学习 `redis` 源码的方法，鸽了一个月，今天分享我的学习方法以及路径，学习步骤不限于 `redis`, 换成其它开源软件套路也是一样。强调一下，**没有速成方法，没有捷径，只有苦行僧一般的坚持才能做好任何一件事情**，与君共勉 ^^ 以前写过 redis 系列，感兴趣的可以[订阅话题](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg5MTYyNzM3OQ==&action=getalbum&album_id=1899308563211091969#wechat_redirect)

![](https://gitee.com/dongzerun/images/raw/master/img/duozhe-redis-tui-gao.jpeg)

### 先导
首先要知道什么是 `redis`, 最好的莫过于[官方文档](https://redis.io/documentation) https://redis.io/documentation 这是英文版的，也可以参考 [中文文档](http://redisdoc.com/)

![](https://gitee.com/dongzerun/images/raw/master/img/redis-io-document.jpg)

通过文档，我们会了解 `redis` 基本的数据结构以及用法：`string`, `set`, `zset`, `hset`, `list` 等等，有一个直观的感受，如果连 `redis` 是做什么用的都不知道，那直接看源码有何意义呢？比如 [An introduction to Redis data types and abstractions](https://redis.io/topics/data-types-intro)

新手强烈建义通读一遍官方文档，知道什么是 `Pipeline`, `Lua scrpit`, `LRU`, `Replication`, `Cluster`, 还要多练习才能加深理解。比如我们常用 `redis` 做分布式锁，那这里就离不开 `Lua script`, 只有用 `Lua` 包装才能保证逻辑的原子性。再比如什么场景下用 `Pipeline`, 为什么会比普通的 PING/PONG 模式快

### 源码学习

![](https://gitee.com/dongzerun/images/raw/master/img/redis-sheji-and-shixian.jpg)

《Redis设计与实现》唯一推荐剖析源码的书，内容不错受益匪浅，不介意给这个广告位^^ 

![](https://gitee.com/dongzerun/images/raw/master/img/san-jiao.jpg)

边看书，边看源码，再结合文档，加深理解，学习非常快。只有源码才是最准确的，书不能一直更新，文档也会有错误，而且非常低级的都出现过
#### 1.分模块学习
![](https://gitee.com/dongzerun/images/raw/master/img/redis-arch-2015.jpg)

整体来讲，redis 架构非常清晰，模块大致有：cluster 集群实现、replication 复制、数据结构、networking epoll、persist 持久化 等等

起手式数据结构从 [sds](https://mp.weixin.qq.com/s/sNCUs-S1t46qFJ3uus5JmA) 开始学习，dict, zipList, skipList 等等，这些代码非常独立，不需要了解其它模块，非常适合入门开始看。重点关注基本操作，比如 [dict](https://mp.weixin.qq.com/s/OwcUHaYmAPkTuL47OO8FyQ) 扩缩容，非常经典的面试题

当我们了解基础数据结构后，就可以看 redis 数据类型了，以 `t_` 开头的源码文件，t_hash.c、t_list.c、t_set.c、t_string.c、t_zset.c 等等，比如 `zset` 由 `dict`、`skipList` 构成，`dict` 里存储什么？为什么需要用到 `dict` 数据结构？`skipList` 存储什么？相比其它平衡树有什么优劣势？

Replication 复制模块非常复杂，完整流程被割裂到多个文件，同时由于 epoll 的存在，各种 event 事件驱动，新手很容易蒙逼。建义从 redis 2.4 开始看

#### 2.跟踪请求流程
![](https://gitee.com/dongzerun/images/raw/master/img/redis-cmd-work-flow.jpg)

redis 数据处理流程比较复杂，代码割裂，epoll 事件。网上有很多分析这方面的文章，建义多通读几遍，然后再阅读源码，可以重新编译源码，debug 模式，并添加更多的日志来验证
#### 3.通过 issue 学习
举一个 [9323](https://github.com/redis/redis/pull/9323) Swap db only at end of diskless replication for better availability 的例子，里面的讨论也蛮有价值的，对于理解 bug 以及 replication 帮助非常大

![](https://gitee.com/dongzerun/images/raw/master/img/9323-swap-db-only.jpg)

歪个楼，学习 etcd 时腾讯云贡献了一个 [issue11651](https://github.com/etcd-io/etcd/issues/11651), 解决了三年之久的 etcd3 数据不一致 bug, 分析验证过程干货十足。强烈建义有空多看看 issue
#### 4.多版本实现对比
开源代码持续改进与演化，最小的比如 `sds` 为了节省空间，增加 flag 标记。大一些的比如复制原理：2.6 远古版本复制，2.8 PSYNC 增强版本，4.0 引入的 PSYNC2 复制协议

横向对比 go Mutex, 第一版本普通的向号量实现，第二版引入了 spinlock, 减少上下文切换，第三版引入公平锁逻辑

横向对比 etcd [fully concurrent](https://mp.weixin.qq.com/s/7jhtXSXvdp1kdceNmRqBvQ) 实现，也是由读写共用一把锁，到读写锁拆分，再到最后写不影响读，是不是特别像 mvcc 原理？

源码不断优化，功能也在持续演进，强烈建义从动态与发展的角度学习
#### 5.多做压测
虽然 redis 运行时不止一个线程，但数据处理只有一个，所以说 redis 是典型的单线程模型，基于 epoll 实现多路复用。市面上类似 redis 的轮子有很多，关于多线程版本实现也很多，阿里也有自研的

![](https://gitee.com/dongzerun/images/raw/master/img/redis-multi-io.jpg)

那么多线程版本性能提升多少呢？性能提升的点在哪里呢？

![](https://gitee.com/dongzerun/images/raw/master/img/redis-multi-io-bench.jpg)

上图是之前做的压测，多线程版本性能提高很明显，但也占用了很多资源

![](https://gitee.com/dongzerun/images/raw/master/img/old-benchmark.jpg)

至于性能问题，可以通过 pprof 查看瓶颈点，感兴趣的可以自行实验
#### 6.通过故障学习
网上有很多同学，分享的 redis 故障资料，他山之石可以攻玉。强烈建义搜搜学习
#### 7.开发工具
条件允许，可以开发小工具，加深对 redis 理解。比如伪装 slave 获取 redis 数据，然后分析 big key, 比如开发 redis 异构数据同步工具，将 redis 数据实时更新到 mysql 或是其它 redis 集群

当然，要说轮子，最大的还是写 kv store, 持久化保存数据，每个公司都有自己的轮子。学习如何编码 rocksdb 的 key/value 构建 redis 各种数据类型
### 小结
总结一下学习流程：**阅读文档了解如何使用、分模块学习、阅读 issue、多版本对比分析**。坚持半年一定有所收获

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `redis` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)


