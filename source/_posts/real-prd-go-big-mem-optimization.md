---
title: 真实环境下 Go 大内存服务优化实践
categories: go
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/tikv-flame.png)

本文是在上家的 case, 以前很多人在公开大会上拿该案例做分享，所以觉得有印象的同学勿喷，虽然冷饭，但是原创

有时大字很不理解的现象，明明 call RPC 时设置了超时时间 timeout, 但是 Grafna 看到 P99 latency 很高，why ???

不要犹豫，要么是 timeout 设置不合理，比如只设置了单次 socket timeout, 并没有设置 circuit breaker 外层超时。参考 [你真的了解 timeout 嘛](https://mp.weixin.qq.com/s/GihBqN5m0vGxxvFdHWRc7Q, "你真的了解 timeout 嘛")

![](https://gitee.com/dongzerun/images/raw/master/img/trick-or-treat.jpg)

还有一种情况就是 GC 在捣乱，我们知道 Go GC 使用三色标记法，在 GC 压力大时用户态 goroutine 是要 assit 协助标记对象的，同时 GC STW 时间如果非常高，那么业务看起来 latency 就会得比 timeout 大很多

### 毛刺
该服务使用 go1.7, 需要加载海量的机器学习词表，标准的 Go 大内存服务，优化前表现为 latency 非常高

![](https://gitee.com/dongzerun/images/raw/master/img/io-big-latency.jpg)

![](https://gitee.com/dongzerun/images/raw/master/img/io-big-latency2.jpg)

可以看到最大的己经到了 2s

![](https://gitee.com/dongzerun/images/raw/master/img/go-gc-pausens.jpg)

同时查看 GC PauseNS 也非常可怕，基本接近 1s, 服务处理不可用状态

### Pprof
如何开启 pprof 这里就不写了，网上有很多，大家可以自行查看
```shell
go tool pprof bin/dupsdc http://127.0.0.1:6060/debug/pprof/profile
```

![](https://gitee.com/dongzerun/images/raw/master/img/cpu-pprof-gc-now.jpg)

可以看到 `runtime.greyobject`, `runtime.mallocgc`, `runtime.heapBitsForObject`, `runtime.scanobject`, `runtime.memmove` 就些与 GC 相关的占据了 CPU 消耗的 TOP 6
```shell
go tool pprof -inuse_objects http://127.0.0.1:6060/debug/pprof/heap
```

![](https://gitee.com/dongzerun/images/raw/master/img/inuse-objects-gc.jpg)

再查看下常驻对像个数，发现 1kw 常驻内存对像(现在来看很小了，不多)，这些都是词表加载的小对像
### 优化对像
词表主要使用两种类型，`map[int64][]float32` 和 `map[string]int`

![](https://gitee.com/dongzerun/images/raw/master/img/three-biaoji.gif)

让我们看一下三色标记，本质是递归扫描所有的指针类型，遍历确定有没有被引用

那么问题来了，什么是指针类型呢？？？**所有显示 \*T 以及内部有 pointer 的对像都是指针类型，比如 `map[int64][]float32` 因为值是 slice, 内部包含了指针，如果 map 有 1kw 个元素，那么 GC 也要递归所描所有 value**

了解这些，优化方法就来了，把 `map[int64][]float32` 变成 `map[int64][6]float32`, 这里 slice 变成 6 个元素的数组，业务可以接受

同时把 `map[string]int` 里的 key 由 string 类型换成 int 枚举值
### 优化效果
上线后优化效果很明显

![](https://gitee.com/dongzerun/images/raw/master/img/after-optimize-inuse-objects.jpg)

可以看到，常驻内存对像由 1kw 降低到 200w

![](https://gitee.com/dongzerun/images/raw/master/img/after-optimize-cpu-pprof.jpg)

同时 cpu pprof 也能看到，排名第一的是 syscall, GC 相关的己经降低很多

![](https://gitee.com/dongzerun/images/raw/master/img/after-optimize-io-latency.jpg)

查看 Grafana 外围 IO latency 降低非常明显。整体优化效果不错
### 例外
这里也有例外，比如 map 内部的实现，如果 key/value 值类型大小超过 128 字节，就会退化成指针
```go
// Builds a type representing a Bucket structure for
// the given map type. This type is not visible to users -
// we include only enough information to generate a correct GC
// program for it.
// Make sure this stays in sync with runtime/map.go.
const (
	BUCKETSIZE  = 8
	MAXKEYSIZE  = 128
	MAXELEMSIZE = 128
)
```
```go
// bmap makes the map bucket type given the type of the map.
func bmap(t *types.Type) *types.Type {
	if t.MapType().Bucket != nil {
		return t.MapType().Bucket
	}

	bucket := types.New(TSTRUCT)
	keytype := t.Key()
	elemtype := t.Elem()
	dowidth(keytype)
	dowidth(elemtype)
	if keytype.Width > MAXKEYSIZE {
		keytype = types.NewPtr(keytype)
	}
	if elemtype.Width > MAXELEMSIZE {
		elemtype = types.NewPtr(elemtype)
	}

	field := make([]*types.Field, 0, 5)
  ......
}
```
### 思考
Go 每个版本性能都会提升很多，go1.7 1kw 对像服务压力非常大，但是我司现在 go1.15 2kw 对像未优化也毫无压力

Go 在吞吐量方面优化非常显著。还是那句话，本文只做为 GC 性能分析参考，不要提前优化

另外一方面也说明，Go 三色标记并不适合所有场景，本次分享的大词表常驻内存就是一个典型：

很明显的 old objects, 不需要 GC 每次都扫描，这里羡慕 java 的分代 GC
### 小结
今天的分享就这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连

关于`性能优化`大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)