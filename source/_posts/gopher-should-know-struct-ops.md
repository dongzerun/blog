---
title: Gopher 需要知道的几个结构体骚操作
categories: go
toc: true
---

![](/images/go-struct-cover.jpeg)

我们知道 Go 没有继承的概念，接口结构体多使用组合，很多开源产品或是源代码都有大量的内嵌 (embeded field) 字段，用于特殊目的。本次分享的内容来自 grpc 与 go 源码

### NoCopy
```go
package main

import (
	"sync"
)

func test(wg sync.WaitGroup) {
	defer wg.Done()
	wg.Add(1)
}

func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go test(wg)
	wg.Wait()
}
```
这是非常经典的 case, 程序执行报错 `all goroutines are asleep - deadlock!`, 解决也很简单，把 `wg` 由值传递变成指针类型即可。本质是 `WaitGroup` 内部维护了计数，不允许 copy 变量，还有 `sync.Mutex` 锁也是不允许 copy 的

解决办法很简单，需要 CI 时由 linter 检测出来，最好运行时也能有检测机制，这方面的讨论请参考[issue 8005](https://github.com/golang/go/issues/8005, "nocopy issue 8005")

```shell
zerun.dong$ go vet aaa.go
# command-line-arguments
./aaa.go:7:14: test passes lock by value: sync.WaitGroup contains sync.noCopy
./aaa.go:15:10: call of test copies lock value: sync.WaitGroup contains sync.noCopy
```
这是 go vet 结果，报错己经很明显了
```
type noCopy struct{}
```
`noCopy` 定义非常简单，空结构体，zero size 不占用空间(前提是非结构体的最后一个字段，否则还要是有 8 byte 空间开销)

[sync.WaitGroup](https://github.com/golang/go/blob/master/src/sync/waitgroup.go#L212, "sync.WaitGroup noCopy") 内嵌 `noCopy` 字段，防止 `Cond` 变量被复制

```go
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers only guarantee that 64-bit fields are 32-bit aligned.
	// For this reason on 32 bit architectures we need to check in state()
	// if state1 is aligned or not, and dynamically "swap" the field order if
	// needed.
	state1 uint64
	state2 uint32
}
```
上面是 `sync.WaitGroup` 结构体的定义，同时注意 `noCopy` 是源码中不可导出的定义。如果用户代码也想实现 NoCopy 呢？可以参考 [grpc DoNotCopy](https://pkg.go.dev/google.golang.org/protobuf@v1.27.1/internal/pragma#DoNotCopy, "pprotobuf DoNotCopy")
```go
// DoNotCopy can be embedded in a struct to help prevent shallow copies.
// This does not rely on a Go language feature, but rather a special case
// within the vet checker.
type DoNotCopy [0]sync.Mutex
```
非常简单，`Mutex` 零长数组，不占用空间。由于 vet checker 会检测 `Mutex`，相当于替我们实现了 `noCopy` 功能

### DoNotCompare
[Golang Sepc Comparison_operators](https://go.dev/ref/spec#Comparison_operators, "官方文档Comparison_operators") 官方文档描述常见类型比较运算( == != > < <= >=)的结果，详细内容看官方文档 https://go.dev/ref/spec#Comparison_operators

1. In any comparison, the first operand must be assignable to the type of the second operand, or vice versa.

2. The equality operators == and != apply to operands that are comparable. The ordering operators <, <=, >, and >= apply to operands that are ordered. 

3. Struct values are comparable if all their fields are comparable. Two struct values are equal if their corresponding non-blank fields are equal.

4. Slice, map, and function values are not comparable. However, as a special case, a slice, map, or function value may be compared to the predeclared identifier nil. Comparison of pointer, channel, and interface values to nil is also allowed and follows from the general rules above.

对于 struct 来讲，**只有所有字段全部 comparable 的(不限大小写是否导出)，那么结构体才可以比较。同时只比较 non-blank 的字段**，举个例子：
```go
type T struct {
    name string
    age int
    _ float64
}
func main() {
   x := [...]float64{1.1, 2, 3.14}
   fmt.Println(x == [...]float64{1.1, 2, 3.14}) // true
   y := [1]T{{"foo", 1, 0}}
   fmt.Println(y == [1]T{{"foo", 1, 1}}) // true
}
```
运行后，结果均为 true

Slice, Map, Function 均是不可比较的，只与判断是否为 nil. 所以我们可以利用这两个特性，内嵌函数来实现不可比较，参考 [protobuf DoNotCompare](https://pkg.go.dev/google.golang.org/protobuf@v1.27.1/internal/pragma#DoNotCompare, "DoNotCompare")

```go
// DoNotCompare can be embedded in a struct to prevent comparability.
type DoNotCompare [0]func()
```
如果比较会报错
```go
type DoNotCompare [0]func()

type T struct {
    name string
    age int
    DoNotCompare
}
func main() {
// ./cmp.go:13:21: invalid operation: T{} == T{} (struct containing DoNotCompare cannot be compared)
    fmt.Println(T{} == T{})
}
```

### NoUnkeyedLiterals
结构体初始化有两种：指定字段名称，或者按顺序列出所有字段，不指定名称
```go
type User struct{
    Age int
    Address string
}

u := &User{21, "beijing"}
```
这样写的问题非常大，如果新增字段会不兼容
```go
type User struct{
    Age int
    Address string
    Money int
}

func main(){
// ./struct.go:11:15: too few values in User{...}
  _ = &User{21, "beijing"}
}
```
上面的例子，能在编译期报错还是可接受的，如果同类型的调换顺序，那才叫坑爹... 所以这时需要 [NoUnkeyedLiterals](https://github.com/protocolbuffers/protobuf-go/blob/v1.27.1/internal/pragma/pragma.go#L12, "protobuf NoUnkeyedLiterals")
```go
// NoUnkeyedLiterals can be embedded in a struct to prevent unkeyed literals.
type NoUnkeyedLiterals struct{}
```
很简单，就是一个空结构体，这是 Protobuf 的实现。很多时候我们都用空的结构体占位符实现
```go
type User struct{
    _ struct{}
    Age int
    Address string
}

func main(){
// ./struct.go:10:11: cannot use 21 (type int) as type struct {} in field value
// ./struct.go:10:15: cannot use "beijing" (type untyped string) as type int in field value
// ./struct.go:10:15: too few values in User{...}
_ = &User{21, "beijing"}
}
```
报错很明显了，字段类型不匹配，有人会说初始化写上 `struct{}` 不就可以了？
```go
_ = &User{struct{}{}, 21, "beijing"}
```
这样确实可以工作，但是占位符 `_` 的字段是不可导出的，所以 import 其它包的 `NoUnkeyedLiterals` 结构体同样会报错

### Copier 库
最后推荐一个非常实用的 [copier](https://github.com/jinzhu/copier, "jinzhu copier") 库，CRUD Boy 经常结构体转来转去的，比如 dto, dao 互转，或是 dao 与其它互转，如果修改了 dao 结构体，还要记得修改其它转换逻辑，非常繁琐
```go
package main
import (
  "fmt"
  "github.com/jinzhu/copier"
)

type User struct {
  Name string
  Age  int
}

type Employee struct {
  Name string
  Age  int
  Role string
}

func main() {
  user := User{Name: "dj", Age: 18}
  employee := Employee{Role: "admin"}

  copier.Copy(&employee, &user)
  // main.Employee{Name:"dj", Age:18, Role:"admin"}
  fmt.Printf("%#v\n", employee)
}
```
打印 `Employee` 发现 name, age 字段己经赋值了，非常好用。感兴趣的可以查看官网，支持非常多的高级玩法

注意：**这里是隐式的，有人觉得所有字段都要显示赋值，大家怎么看？**

### 小结
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `struct 骚操作` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)