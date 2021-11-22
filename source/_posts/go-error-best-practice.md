---
title: Go error 处理最佳实践
categories: go
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/error-cover.jpg)

今天分享 go 语言 error 处理的最佳实践，了解当前 error 的缺点、妥协以及使用时注意事项。文章内容较长，干货也多，建义收藏

### 什么是 error
大家都知道 [error](https://github.com/golang/go/blob/master/src/builtin/builtin.go#L260, "builting.go error interface") 是源代码内嵌的接口类型。根据导出原则，只有大写的才能被其它源码包引用，但是 error 属于 predeclared identifiers 预定义的，并不是关键字，细节参考[int make 居然不是关键字？](https://mp.weixin.qq.com/s/OxLtvOPyeYMV34EDEXEFlA)
```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```
`error` 只有一个方法 `Error() string` 返回错误消息
```go
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```
一般我们创建 error 时只需要调用 `errors.New("error from somewhere")` 即可，底层就是一个字符串结构体 `errorString`s

### 当前 error 有哪些问题

```go
func Test() error {
	if err := func1(); err != nil {
		return err
	}
  ......
}
```
这是常见的用法，也最被人诟病，很多人觉得不如 `try-catch` 用法简洁，有人戏称 go 源码错误处理占一半
```go
import sys

try:
    f = open('myfile.txt')
    s = f.readline()
    i = int(s.strip())
except OSError as err:
    print("OS error: {0}".format(err))
except ValueError:
    print("Could not convert data to an integer.")
except BaseException as err:
    print(f"Unexpected {err=}, {type(err)=}")
    raise
```
比如上面是 python `try-catch` 的用法，先写一堆逻辑，不处理异常，最后统一捕获
```rust
let mut cfg = self.check_and_copy()?;
```
相比来说 rust Result 模式更简洁，一个 ? 就代替了我们的操作。但是 error 的繁琐判断是当前的痛点嘛？显然不是，尤其喜欢 c 语言的人，反而喜欢每次都做判断

**在我看来 go 的痛点不是缺少泛型，不是 error 太挫，而是 GC 太弱，尤其对大内存非常不友好**，这方面可以参考[真实环境下大内存 Go 服务性能优化一例](https://mp.weixin.qq.com/s/jGGCccMOx4s5asG2IXWNMQ)

当前 error 的问题有两点：
1. 无法 wrap 更多的信息，比如调用栈，比如层层封装的 error 消息
2. 无法很好的处理类型信息，比如我想知道错误是 io 类型的，还是 net 类型的

#### 1.Wrap 更多的消息
这方面有很多轮子，最著名的就是 `https://github.com/pkg/errors`, 我司也重度使用，主要功能有三个：

1. `Wrap` 封装底层 error, 增加更多消息，提供调用栈信息，这是原生 error 缺少的
2. `WithMessage` 封装底层 error, 增加更多消息，但不提供调用栈信息
3. `Cause` 返回最底层的 error, 剥去层层的 wrap
```go
import (
   "database/sql"
   "fmt"

   "github.com/pkg/errors"
)

func foo() error {
   return errors.Wrap(sql.ErrNoRows, "foo failed")
}

func bar() error {
   return errors.WithMessage(foo(), "bar failed")
}

func main() {
   err := bar()
   if errors.Cause(err) == sql.ErrNoRows {
      fmt.Printf("data not found, %v\n", err)
      fmt.Printf("%+v\n", err)
      return
   }
   if err != nil {
      // unknown error
   }
}
/*Output:
data not found, bar failed: foo failed: sql: no rows in result set
sql: no rows in result set
foo failed
main.foo
    /usr/three/main.go:11
main.bar
    /usr/three/main.go:15
main.main
    /usr/three/main.go:19
runtime.main
    ...
*/
```
这是测试代码，当用 `%v` 打印时只有原始错误信息，`%+v` 时打印完整调用栈。当 go1.13 后，标准库 errors 增加了 `Wrap` 方法
```go
func ExampleUnwrap() {
	err1 := errors.New("error1")
	err2 := fmt.Errorf("error2: [%w]", err1)
	fmt.Println(err2)
	fmt.Println(errors.Unwrap(err2))
	// Output
	// error2: [error1]
	// error1
}
```
标准库没有提供增加调用栈的方法，`fmt.Errorf` 指定 `%w` 时可以 wrap error, 但整体来讲，并没有 `https://github.com/pkg/errors` 库好用
#### 2.错误类型
这个例子来自 [ITNEXT](https://itnext.io/golang-error-handling-best-practice-a36f47b0b94c, "ITNEXT")
```go
import (
   "database/sql"
   "fmt"
)

func foo() error {
   return sql.ErrNoRows
}

func bar() error {
   return foo()
}

func main() {
   err := bar()
   if err == sql.ErrNoRows {
      fmt.Printf("data not found, %+v\n", err)
      return
   }
   if err != nil {
      // Unknown error
   }
}
//Outputs:
// data not found, sql: no rows in result set
```
有时我们要处理类型信息，比如上面例子，判断 err 如果是 `sql.ErrNoRows` 那么视为正常，data not found 而己，类似于 redigo 里面的 `redigo.Nil` 表示记录不存在
```go
func foo() error {
   return fmt.Errorf("foo err, %v", sql.ErrNoRows)
}
```
但是如果 `foo` 把 error 做了一层 wrap 呢？这个时候错误还是 `sql.ErrNoRows` 嘛？肯定不是，这点没有 python `try-catch` 错误处理强大，可以根据不同错误 class 做出判断。那么 go 如何解决呢？答案是 go1.13 新增的 [Is](https://github.com/golang/go/blob/master/src/errors/wrap.go#L40, "errors.Is") 和 [As](https://github.com/golang/go/blob/master/src/errors/wrap.go#L78)
```go
import (
   "database/sql"
   "errors"
   "fmt"
)

func bar() error {
   if err := foo(); err != nil {
      return fmt.Errorf("bar failed: %w", foo())
   }
   return nil
}

func foo() error {
   return fmt.Errorf("foo failed: %w", sql.ErrNoRows)
}

func main() {
   err := bar()
   if errors.Is(err, sql.ErrNoRows) {
      fmt.Printf("data not found,  %+v\n", err)
      return
   }
   if err != nil {
      // unknown error
   }
}
/* Outputs:
data not found,  bar failed: foo failed: sql: no rows in result set
*/
```
还是这个例子，`errors.Is` 会递归的 Unwrap err, 判断错误是不是 `sql.ErrNoRows`，这里个小问题，`Is` 是做的指针地址判断，如果错误 `Error()` 内容一样，但是根 error 是不同实例，那么 `Is` 判断也是 false, 这点就很扯
```go
func ExampleAs() {
	if _, err := os.Open("non-existing"); err != nil {
		var pathError *fs.PathError
		if errors.As(err, &pathError) {
			fmt.Println("Failed at path:", pathError.Path)
		} else {
			fmt.Println(err)
		}
	}

	// Output:
	// Failed at path: non-existing
}
```
[errors.As](https://github.com/golang/go/blob/master/src/errors/wrap_test.go#L255, "errors.As example") 判断这个 err 是否是 `fs.PathError` 类型，递归调用层层查找，源码后面再讲解

另外一个判断类型或是错误原因的就是 `https://github.com/pkg/errors` 库提供的 `errors.Cause`
```go
switch err := errors.Cause(err).(type) {
case *MyError:
        // handle specifically
default:
        // unknown error
}
```

在没有 `Is` `As` 类型判断时，需要很恶心的去判断错误自符串
```go
func (conn *cendolConnectionV5) serve() {
	// Buffer needs to be preserved across messages because of packet coalescing.
	reader := bufio.NewReader(conn.Connection)
	for {
		msg, err := conn.readMessage(reader)
		if err != nil {
			if netErr, ok := strings.Contain(err.Error(), "temprary"); ok   {
					continue
			}
		}

		conn.processMessage(msg)
	}
}
```
想必接触 go 比较早的人一定很熟悉，如果 conn 从网络接受到的连接错误是 `temporary` 临时的那么可以 continue 重试，当然最好 backoff sleep 一下

当然现在新增加了 `net.Error` 类型，实现了 `Temporary` 接口，不过也要废弃了，请参考[#45729](https://github.com/golang/go/issues/45729, "#45729")

### 源码实现
#### 1.`github.com/pkg/errors` 库如何生成 warapper error
```go
// Wrap returns an error annotating err with a stack trace
// at the point Wrap is called, and the supplied message.
// If err is nil, Wrap returns nil.
func Wrap(err error, message string) error {
	if err == nil {
		return nil
	}
	err = &withMessage{
		cause: err,
		msg:   message,
	}
	return &withStack{
		err,
		callers(),
	}
}
```
主要的函数就是 `Wrap`, 代码实现比较简单，查看如何追踪调用栈可以查看源码
#### 2.`github.com/pkg/errors` 库 `Cause` 实现
```go
type withStack struct {
	error
	*stack
}

func (w *withStack) Cause() error { return w.error }

func Cause(err error) error {
	type causer interface {
		Cause() error
	}

	for err != nil {
		cause, ok := err.(causer)
		if !ok {
			break
		}
		err = cause.Cause()
	}
	return err
}
```
`Cause` 递归调用，如果没有实现 `causer` 接口，那么就返回这个 err
#### 3.官方库如何生成一个 wrapper error
官方没有这样的函数，而是 `fmt.Errorf` 格式化时使用 `%w`
```go
e := errors.New("this is a error")
w := fmt.Errorf("more info about it %w", e)
```
```go
func Errorf(format string, a ...interface{}) error {
	p := newPrinter()
	p.wrapErrs = true
	p.doPrintf(format, a)
	s := string(p.buf)
	var err error
	if p.wrappedErr == nil {
		err = errors.New(s)
	} else {
		err = &wrapError{s, p.wrappedErr}
	}
	p.free()
	return err
}

func (p *pp) handleMethods(verb rune) (handled bool) {
	if p.erroring {
		return
	}
	if verb == 'w' {
		// It is invalid to use %w other than with Errorf, more than once,
		// or with a non-error arg.
		err, ok := p.arg.(error)
		if !ok || !p.wrapErrs || p.wrappedErr != nil {
			p.wrappedErr = nil
			p.wrapErrs = false
			p.badVerb(verb)
			return true
		}
		p.wrappedErr = err
		// If the arg is a Formatter, pass 'v' as the verb to it.
		verb = 'v'
	}
  ......
}
```
代码也不难，`handleMethods` 时特殊处理 `w`, 使用 `wrapError` 封装一下即可
#### 4.官方库 `Unwrap` 实现
```go
func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	})

	if !ok {
		return nil
	}
	return u.Unwrap()
}
```
也是递归调用，否则接口断言失败，返回 nil
```go
type wrapError struct {
	msg string
	err error
}

func (e *wrapError) Error() string {
	return e.msg
}

func (e *wrapError) Unwrap() error {
	return e.err
}
```
上文 `fmt.Errof` 时生成的 error 结构体如上所示，`Unwrap` 直接返回底层 err
#### 5.官方库 `Is` `As` 实现
本段源码分析来自 [flysnow](https://www.flysnow.org/2019/09/06/go1.13-error-wrapping.html, "flysnow error 分析")
```go
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	
	//for循环，把err一层层剥开，一个个比较，找到就返回true
	for {
		if isComparable && err == target {
			return true
		}
		//这里意味着你可以自定义error的Is方法，实现自己的比较代码
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true
		}
		//剥开一层，返回被嵌套的err
		if err = Unwrap(err); err == nil {
			return false
		}
	}
}
```
`Is` 函数比较简单，递归层层检查，如果是嵌套 err, 那就调用 `Unwrap` 层层剥开找到最底层 err, 最后判断指针是否相等
```go
var errorType = reflectlite.TypeOf((*error)(nil)).Elem()

func As(err error, target interface{}) bool {
    //一些判断，保证target，这里是不能为nil
	if target == nil {
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	
	//这里确保target必须是一个非nil指针
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
		panic("errors: target must be a non-nil pointer")
	}
	
	//这里确保target是一个接口或者实现了error接口
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
	    //关键部分，反射判断是否可被赋予，如果可以就赋值并且返回true
	    //本质上，就是类型断言，这是反射的写法
		if reflectlite.TypeOf(err).AssignableTo(targetType) {
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
		//这里意味着你可以自定义error的As方法，实现自己的类型断言代码
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
			return true
		}
		//这里是遍历error链的关键，不停的Unwrap，一层层的获取err
		err = Unwrap(err)
	}
	return false
}
```
代码同样是递归调用 `As`, 同时 `Unwrap` 最底层的 error, 然后用反射判断是否可以赋值，如果可以，那么说明是同一类型
### ErrGroup 使用
提到 error 就必须要提一下 `golang.org/x/sync/errgroup`, 适用如下场景：**并发场景下，如果一个 goroutine 有错误，那么就要提前返回，并取消其它并行的请求**
```go
func ExampleGroup_justErrors() {
	g := new(errgroup.Group)
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somestupidname.com/",
	}
	for _, url := range urls {
		// Launch a goroutine to fetch the URL.
		url := url // https://golang.org/doc/faq#closures_and_goroutines
		g.Go(func() error {
			// Fetch the URL.
			resp, err := http.Get(url)
			if err == nil {
				resp.Body.Close()
			}
			return err
		})
	}
	// Wait for all HTTP fetches to complete.
	if err := g.Wait(); err == nil {
		fmt.Println("Successfully fetched all URLs.")
	}
}
```
上面是官方给的例子，底层使用 `context` 来 cancel 其它请求，同步使用 `WaitGroup`, 原理非常简单，代码量非常少，感兴趣的可以看源码

这里一定要注意三点：
1. `context` 是谁传进来的？其它代码会不会用到，cancel 只能执行一次，瞎比用会出问题
2. `g.Go` 不带 recover 的，为了程序的健壮，一定要自行 recover
3. 并行的 goroutine 有一个错误就返回，而不是普通的 fan-out 请求后收集结果
### 线上实践注意的几个问题
#### 1.error 与 panic
查看 go 源代码会发现，源码很多地方写 panic, 但是工程实践，尤其业务代码不要主动写 panic

理论上 panic 只存在于 server 启动阶段，比如 config 文件解析失败，端口监听失败等等，**所有业务逻辑禁止主动 panic**

根据 `CAP` 理论，当前 web 互联网最重要的是 `AP`, 高可用性才最关键(非银行金融场景)，程序启动时如果有部分词表，元数据加载失败，都不能 panic, 提供服务才最关键，当然要有报警，让开发第一时间感知当前服务了的 QOS 己经降低

最后说一下，所有异步的 goroutine 都要用 `recover` 去兜底处理
#### 2.错误处理与资源释放
```go
func worker(done chan error) {
    err := doSomething()
    result := &result{}
    if err != nil {
        result.Err = err
    }
    done <- result
}
```
一般异步组装数据，都要分别启动 goroutine, 然后把结果通过 channel 返回，result 结构体拥有 err 字段表示错误

这里要注意，main 函数中 `done` channel 千万不能 close, 因为你不知道 `doSomething` 会超时多久返回，写 closed channel 直接 panic 

所以这里有一个准则：**数据传输和退出控制，需要用单独的 channel 不能混, 我们一般用 context 取消异步 goroutine, 而不是直接 close channels**

#### 3.error 级联使用问题
```go
package main

import "fmt"

type myError struct {
	string
}

func (i *myError) Error() string {
	return i.string
}

func Call1() error {
	return nil
}

func Call2() *myError {
	return nil
}

func main() {
	err := Call1()
	if err != nil {
		fmt.Printf("call1 is not nil: %v\n", err)
	}

	err = Call2()
	if err != nil {
		fmt.Printf("call2 err is not nil: %v\n", err)
	}
}
```
这个问题非常经典，如果复用 err 变量的情况下， Call2 返回的 error 是自定义类型，此时 err 类型是不一样的，导致经典的 error is not nil, but value is nil

非常经典的 [Nil is not nil](https://yourbasic.org/golang/gotcha-why-nil-error-not-equal-nil/, "Nil is not nil") 问题。解决方法就是 `Call2` err 重新定义一个变量，当然最简单就是统一 error 类型。有点难，尤其是大型项目
#### 4.并发问题
go 内置类型除了 channel 大部分都是非线程安全的，error 也不例外，先看一个例子
```go
package main
import (
   "fmt"
   "github.com/myteksi/hystrix-go/hystrix"
   "time"
)
var FIRST error = hystrix.CircuitError{Message:"timeout"}
var SECOND error = nil
func main() {
   var err error
   go func() {
      i := 1
      for {
         i = 1 - i
         if i == 0 {
            err = FIRST
         } else {
            err = SECOND
         }
         time.Sleep(10)
      }
   }()
   for {
      if err != nil {
         fmt.Println(err.Error())
      }
      time.Sleep(10)
   }
}
```
运行之前，大家先猜下会发生什么？？？
```shell
zerun.dong$ go run panic.go
hystrix: timeout
panic: value method github.com/myteksi/hystrix-go/hystrix.CircuitError.Error called using nil *CircuitError pointer

goroutine 1 [running]:
github.com/myteksi/hystrix-go/hystrix.(*CircuitError).Error(0x0, 0xc0000f4008, 0xc000088f40)
 <autogenerated>:1 +0x86
main.main()
 /Users/zerun.dong/code/gotest/panic.go:25 +0x82
exit status 2
```
上面是测试的例子，只要跑一会，就一定发生 panic, 本质就是 error 接口类型不是并发安全的
```go
// 没有方法的interface
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
// 有方法的interface
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```
所以不要并发对 error 赋值

#### 5.error 要不要忽略
```go
func Test(){
	_ = json.Marshal(xxxx)
	......
}
```
有的同学会有疑问，error 是否一定要处理？其实上面的 `Marshal` 都有可能失败的

如果换成其它函数，当前实现可以忽略，不能保证以后还是兼容的逻辑，一定要处理 error，至少要打日志
#### 6.errWriter
本例来自[官方 blog](https://blog.golang.org/errors-are-values, "errors are values"), 有时我们想做 pipeline 处理，需要把 err 当成结构体变量
```go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```
上面是原始例子，需要一直做 `if err != nil` 的判断，官方优化的写法如下
```go
type errWriter struct {
    w   io.Writer
    err error
}

func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}

// 使用时
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```
清晰简洁，大家平时写代码可以多考滤一下
#### 7.何时打印调用栈
官方库无法 wrap 调用栈，所以 `fmt.Errorf %w` 不如 `pkg/errors` 库实用，**但是 `errors.Wrap` 最好保证只调用一次，否则全是重复调用栈**

我们项目的使用情况是 log error 级别的打印栈，warn 和 info 都不打印，当然 case by case 还得看实际使用情况
#### 8.Wrap前做判断
```go
errors.Wrap(err, "failed")
```
通过查看源码，如果 err 为 nil 的时候，也会返回 nil. 所以 `Wrap` 前最好做下判断，建议来自 xiaorui.cc
### 小结
上面提到的线上实践注意的几个问题，都是实际发生的坑，惨痛的教训，大家一定要多体会下。错误处理涵盖内容非常广，本文不涉及分布式系统的错误处理、gRPC 错误传播以及错误管理

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `error` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)
