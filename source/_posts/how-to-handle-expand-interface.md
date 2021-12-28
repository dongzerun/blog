---
title: 如何应对不断膨胀的接口
categories: interface
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/go-interface-cover.jpg)

难怪码农自嘲是 CRUD boy, 每天确实在不断的堆屎，在别人的屎山上缝缝补补。下面的案例并没有 blame 任何人的意思，我也是堆屎工^^ 如有雷同，请勿对号入座

### 案例
最近读一个业务代码，状态机接口定义有 40 个函数，查看 commit log, 初始只有 10 个，每当增加新的业务需求时，就不断的在原接口添加

```go
// OrderManager handles operation on order entity
type OrderManager interface {
	LoadOrdersByIDs(ctx context.Context, orderIDs []string) ([]*dbentity.Order, error)
  ......
  TransitOrdersToState(ctx context.Context, orderIDs []string, toState orderstate.OrderState) ([]*dbentity.Order, error)
  ......
	Stop() error
}
```

业务中很多 interface 都是用来 mock UT, 实现依赖反转，而不是业务多态。`OrderManager` 就属于这类，所以接口膨胀后对工程质量影响并不大，就是看着不内聚...

### 接口为什么要小

> The bigger the interface, the weaker the abstraction.

[Go Proverbs](https://go-proverbs.github.io/, "Go Proverbs") Rob Pike 提到：**接口越大，抽像能力越弱**，比如系统库中的 `io.Reader`, `io.Writer` 等等接口定义只有一两个函数。为什么说接口要小呢？举个例子

```go
type FooBeeper interface {
  Bar(s string) (string, error)
  Beep(s string) (string, error)
}

type thing struct{}

func (l *thing) Bar(s string) (string, error) {
  ...
}

func (l *thing) Beep(s string) (string, error) {
  ...
}

type differentThing struct{}

func (l *differentThing) Bar(s string) (string, error) {
  ...
}

type anotherThing struct{}

func (l *anotherThing) Beep(s string) (string, error) {
  ...
}
```
接口 `FooBeeper` 定义有两个函数: `Bar`, `Beep`. 由于接口实现是隐式的，我们有如下结论：
* `thing` 实现了 `FooBeeper` 接口
* `differentThing` 没有实现，缺少 `Bar` 函数
* `anotherThing` 同样没有实现，缺少 `Beep` 函数

但是如果我们把 `FooBeeper` 打散也多个接口的组合
```go
type FooBeeper interface {
	Bar
	Beep
}

type Bar interface {
	Bar(s string) (string, error)
}

type Beep interface {
	Beep(s string) (string, error)
}
```
如上述定义，就可以将接口做小，使得 `differentThing` `anotherThing` 可以复用接口

### 组合改造
关于如何改造 `OrderManger` 可以借鉴 [etcd client v3](https://github.com/etcd-io/etcd/blob/main/client/v3/client.go#L44, "etcd client v3") 定义的思想，将相关的功能取合成小接口，通过接口的组合实现

```go
type Client struct {
	Cluster
	KV
	Lease
	Watcher
	Auth
	Maintenance

	conn *grpc.ClientConn

	cfg      Config
  ......
}
```
上面是 clientV3 结构体定义，虽然不是接口，但是思想可以借鉴

```go
// OrderManager handles operation on order entity
type OrderManager interface {
	OrderOperator
	TransitOrders
	Stop() error
}
```
实际上可能接口只需抽成三个，`OrderOperator` 负责对 orders 的 CRUD 操作，`TransitOrders` 负责转态机流转，原来的 40 个函数函数都放到小接口里面

### 冗余改造
只抽成小接口是不行的，`LoadOrderByXXXX` 有一堆定义，根据不同条件获取定单，但实际上这些都是可以转换的

```go
func LoadOrders(ctx context.Context, FiltersParams options...)
```

针对这种情况可以传入 option, 或是用结构体当成参数容器。再比如状态机流转有 `TransitOrdersToState`, `TransitOrdersToStateByEntity`, `TransitOrdersStateByEntityForRegularDelivery` 均属于冗余定义

还有一种冗余接口是根本没人用，或是不该这层暴露的

### 预防
接口拆分本质上是 ISP [Interface segregation principle](https://en.wikipedia.org/wiki/Interface_segregation_principle, "Interface segregation principle"), 不应该强迫任何代码依赖它不使用的方法

IT 行业有一个笑话

> 当你的 MR 只有几行时，peer 会提出几十个 comment. 但是当你的 MR 几百行时，他们只会回复 LGTM

Peer review 还是要有责任心的，如果成本不高，建义顺手把老代码重构一下。重构代码有几项原则，可以参考 [重构最佳实践2](https://mp.weixin.qq.com/s/fcxg67KHj9AmN-7a3g3e0g)

CI lint 不知道是否支持检查 interface 行数，但是如果行数成为指标，可能又本末倒置了

### 小结
还是那句话，知易行难，从点滴做起吧

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `接口` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)