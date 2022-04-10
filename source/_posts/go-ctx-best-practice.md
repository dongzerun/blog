---
title: Go Context 最佳实践
categories: go
toc: true
---

![](/images/context-cover.jpg)

去年写了[也许是 Context 最佳实践](https://mp.weixin.qq.com/s/UmqjIqesXddotHpNh02VCw), 回头看有些遗漏，重新编辑整理，总结截至 go 1.17 的最佳实践

尽管有人说[Context should go away in GO2](https://faiface.github.io/post/context-should-go-away-go2/, "Context should go away in GO2"), 但是现有的代码中还是大量使用 `Context`, 并不是每个人都了解 `Context`, 从去年到现在就见过两次因为错误使用导致的问题。每个同学都会踩到坑，今天分享下 `Context` 库使用的 Dos and Don'ts

### 使用场景
`Context` 主要有以下三种使用场景
* 传递超时信息，这点用的最多
* 传递信号，用于消息通知，处理多协程通信
* 传递数据，常用的框架层 trace-id, metadata

举一个 etcd watch 的例子，我们加深了解
```go
func watch(ctx context.Context, revision int64) {
 ctx, cancel := context.WithCancel(ctx)
 defer cancel()

 for {
  rch := watcher.Watch(ctx, watchPath, clientv3.WithRev(revision))
  for wresp := range rch {
    ......
      doSomething()
  }

  select {
  case <-ctx.Done():
   // server closed, return
   return
  default:
  }
 }
}
```
首先基于参数传进来的 `parent ctx` 生成了 `child ctx` 与 `cancel` 函数。然后 `Watch` 时传入 `child ctx`, 如果此时 `parent ctx` 被外层 `cancel`, `child ctx` 也会被级联 `cancel`, `rch` 会被 `etcd` 关闭，然后 `for` 循环走到 `select` 逻辑，此时 `child ctx` 被取消了，所以 <-ctx.Done() 生效，`watch` 函数返回

其于 context 可以很好的做到多个 goroutine 协作，超时管理，大大简化了开发工作。这也是 Go 的魅力

### 原理
```go
type Context interface {
 Deadline() (deadline time.Time, ok bool)
 Done() <-chan struct{}
 Err() error
 Value(key interface{}) interface{}
}
```
`Context` 是一个接口

* `Deadline` ctx 如果在某个时间点关闭的话，返回该值。否则 ok 为 false
* `Done` 返回一个 channel, 如果超时或是取消就会被关闭，实现消息通讯
* `Err` 如果当前 ctx 超时或被取消了，那么 Err 返回错误
* `Value` 根据某个 key 返回对应的 value, 功能类似字典

目前的实现有 `emptyCtx`, `valueCtx`, `cancelCtx`, `timerCtx`. 可以基于某个 Parent 派生成 Child Context
```go
func WithValue(parent Context, key, val interface{}) Context
func WithCancel(parent Context) (Context, CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

![](/images/ctx-cancel.jpg)

经过多次派生后，ctx 是一个类似多叉树的结构。当 ctx-1 被 cancel 时，会级联 cancel 以 ctx-1 为根的整棵树，但是原来的 root, ctx2 ctx3 不受影响
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
首先检测 done channel, 如果有人监听，那么 close 掉，这时所有 wait 这个 ctx 的 goroutines 都会收到消息

然后遍历 children map, 依次 cancel 所有 child, 这里类似树的先序遍历。最后 `removeFromParent` 将自己从父节点中摘除

### 几个问题
#### 打印 Ctx
以 `WithCancel` 为例子，可以看到 child 同时引用了 parent, 而 `propagateCancel` 函数的存在，parent 也会引用 child(当 parent 是 cancelCtx 类型时)
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
 c := newCancelCtx(parent)
 propagateCancel(parent, &c)
 return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
 return cancelCtx{Context: parent}
}
```
如果此时打印 ctx, 就会递归调用 `String()` 方法，就会把 `key/value` 打印出来。如果此时 value 是非线程安全的，比如 map, 就会引发 `concurrent read and write panic`

这个案例就是 http 标准库的实现 [server.go:2906]( https://github.com/golang/go/blob/master/src/net/http/server.go#L2878, "http server") 行代码，把 http server 保存到 ctx 中
```go
ctx := context.WithValue(baseCtx, ServerContextKey, srv)
```
最后调用业务层代码时把 ctx 传给了用户
```go
go c.serve(connCtx)
```
如果此时打印 ctx, 就会打印 http srv 结构体，这里面就有 map. 感兴趣的可以做个实验，拿 ab 压测很容易复现

```go
func stringify(v interface{}) string {
	switch s := v.(type) {
	case stringer:
		return s.String()
	case string:
		return s
	}
	return "<not Stringer>"
}

func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(type " +
		reflectlite.TypeOf(c.key).String() +
		", val " + stringify(c.val) + ")"
}
```

![](/images/context-fmt.jpg)

同时注意，后来 go 对此做了部份修复，一定程序上解决了问题。但也记住不要打印 ctx

#### Key/Value 类型不安全
```go
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}
```
强烈不建义使用 Context 传递过多数据，这里可以看到 `key`/`value` 类型都是 `interface{}`, 编译期无法确定类型，运行期需要断言，有性能和安全问题
#### 关闭底层连接
`Context` 超时会触发 http pool 关闭掉底层 connection, 导致连接频繁销重建，参考之前的文章[超时控制一个反例](https://mp.weixin.qq.com/s/Mwhy2JllWTIxKK5U-zCgiA)

问题在于，要在哪层处理 tcp 无用数据，如果应用层读完再丢掉，此时连接还是可用的，但是操作系统 tcp stack 处理无用数据，那直接就 close. 而 grpc 就没这个问题，因为多路复用，每个请求都是虚拟的 stream, 如果超时，只需关闭 stream, 无需关闭底层 tcp 连接
#### 双向链表
当 `Context` 派生层数比较多时，构成了一个双向链表，`key`/`value` 获取很有可能退化成 O(N) 操作，非常慢
```go
type valueCtx struct {
	Context
	key, val interface{}
}

func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```
每当添加一个 `key`/`value` 时都会生成新的 `valueCtx`, 查询时，如果当前 ctx 不存在 `key`, 则递归查询 `c.Context`
#### 提前超时
```go
func test(){
 ctx, cancel := context.WithCancel(ctx)
 defer cancel()
  
  doSomething(ctx)
}

func doSomething(ctx){
  go doOthers(ctx)
}
```
当调用栈较深，多人合作时很容易产生这种情况。其实还是没明白 ctx cancel 工作原理，异步 go 出去的业务逻辑需要基于 `context.Background()` 再派生 child ctx, 否则就会提前超时返回

另外大家容易忽略的点，默认情况下 `grpc` 会透传超时时间的，比如入口 A 服务调 B, 超时设置了 2s, B 如果用同一个 `Context` 去调下游 C, 那么超时就要减去 B 自己处理的时间。如果链路比较长，很可能到达 G 服务时就己经超时了

传递超时可以提前释放资源，否则入口超时了，后端还在处理请求
#### 自定义 Ctx
非常不建义自定义 `Context`, 原因在于源码中处理是不同的
```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
  ......
 if p, ok := parentCancelCtx(parent); ok {
  p.mu.Lock()
  if p.err != nil {
  ......
  } else {
  ......
   p.children[child] = struct{}{}
  }
  p.mu.Unlock()
 } else {
  atomic.AddInt32(&goroutines, +1)
  go func() {
   select {
   case <-parent.Done():
    child.cancel(false, parent.Err())
   case <-child.Done():
   }
  }()
 }
}

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
  ......
 p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
 if !ok {
  return nil, false
 }
  ......
 return p, true
}
```
通过源码可知，`parent` 引用 `child` 有两种方式，官方 `cancelCtx` 类型的是用 map 保存。但是非官方的需要开启 `goroutine` 去监测。本来业务代码己经 `goroutine` 满天飞了，不加节制的使用只会增加系统负担

### 使用建议
最后来总结下 context 使用的几个原则：

* 除了框架层不要使用 `WithValue` 携带业务数据，这个类型是 `interface{}``, 编译期无法确定，运行时 `assert` 有开销。如果真要携带也要用 thread-safe 的数据
* 一定不要打印 `Context`, 尤其是从 `http` 标准库派生出来的，谁知道里面存了什么
* `Context` 通常做为第一个参数传给函数，但如果 `Context` 生命周期等同于结构体，当成结构体成员也可以
* 尽可能不要自定义用户层 `Context`, 除非收益巨大
* 异步 goroutine 逻辑使用 `Context` 时要清楚谁还持有，会不会提前超时，尤其调 rpc, db, redis 时
* 派生出来的 child ctx 一定要配合 defer cancel() 使用，释放资源


### 小结 
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Context` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)