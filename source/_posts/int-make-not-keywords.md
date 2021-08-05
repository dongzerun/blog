---
title: int make 居然不是关键字？
---

这是一个小白问题，有多少人知道 `int` 居然不是关键字？`make` 也不是关键字？

我们知道每种语言都有关键字和保留字的，而 go 以[关键字](https://golang.org/ref/spec#Keywords)少著称，只有25个
```shell
break        default      func         interface    select
case         defer        go           map          struct
chan         else         goto         package      switch
const        fallthrough  if           range        type
continue     for          import       return       var
```
也就是说，**我们常用的 `make`, `cap`, `len`不是关键字，就连基本数据类型 `int`, `int64`, `float` 也都不是** 但是 [C 语言中关键字](https://en.cppreference.com/w/c/keyword)可是非常多的

### make 内置函数
```go
package main

import "fmt"

func main(){
    make := func() string {
        return "hijacked"
    }

    int := make()    // Completely OK, variable 'int' will be a string
    fmt.Println(int) // Prints "hijacked"
}
```
这段代码 `make` 变量是一个闭包，返回一个字符串，而 `int` 变量类型是字符串。最后函数打印 hijacked. 显然这段代码很神经病，谁要这么写会被打死，但确是可以编译成功的

同时如果想继续用 `make` 创建 map, 或是用 `int` 声明变量就会报错。本质上 `make`, `cap`, `len` 都是 go 源码中的函数名，有点泛型的意思

```go
// The make built-in function allocates and initializes an object of type
// slice, map, or chan (only). Like new, the first argument is a type, not a
// value. Unlike new, make's return type is the same as the type of its
// argument, not a pointer to it. The specification of the result depends on
// the type:
//	Slice: The size specifies the length. The capacity of the slice is
//	equal to its length. A second integer argument may be provided to
//	specify a different capacity; it must be no smaller than the
//	length. For example, make([]int, 0, 10) allocates an underlying array
//	of size 10 and returns a slice of length 0 and capacity 10 that is
//	backed by this underlying array.
//	Map: An empty map is allocated with enough space to hold the
//	specified number of elements. The size may be omitted, in which case
//	a small starting size is allocated.
//	Channel: The channel's buffer is initialized with the specified
//	buffer capacity. If zero, or the size is omitted, the channel is
//	unbuffered.
func make(t Type, size ...IntegerType) Type
```
```go
func len(v Type) int
```
```go
func cap(v Type) int
```
上面是 runtime 中对 `make`, `len`, `cap` 的函数定义，大家可以看注释或是看 [builtin.go](https://github.com/golang/go/blob/master/src/builtin/builtin.go#L189)

也就是说，变量名用 `make`, 只是在 main 函数这个词法块中普通的局部变量而己，同时遮蔽了 runtime 的 `make` 函数名

### Predeclared identifiers
前面说的是 `make`, 那么对于 `int` 呢？其实道理也一样，这些都是 go 预定义的标识符 [Predeclared identifiers](https://golang.org/ref/spec#Predeclared_identifiers)
```
Types:
	bool byte complex64 complex128 error float32 float64
	int int8 int16 int32 int64 rune string
	uint uint8 uint16 uint32 uint64 uintptr

Constants:
	true false iota

Zero value:
	nil

Functions:
	append cap close complex copy delete imag len
	make new panic print println real recover
```
其实这些都 document 在 [builtin.go](https://github.com/golang/go/blob/master/src/builtin/builtin.go#L189)，包括常见的类型，`true`, `false`, `iota`, `nil` 以及常用的函数 `make`, `new`, `copy` 等等，这些在其它语言可能都对应着关键词 keywords 或是保留词

从编译原理的角度看，`identifiers` 和 `keywords` 关键词没有本质的区别，都是一个一个 token 而己

**官方告诉我们，这些预定义的标识符在 [universe block](https://golang.org/ref/spec#Blocks) 块中都是隐式定义的，所以我们才能直接用**。那么什么是 `universe block` 呢？

```
Block = "{" StatementList "}" .
StatementList = { Statement ";" } .
```

除了上面这种显示的语句块，还有很多隐式的语句块。大家要小心，因为很多时候 variable shadow 就是因为这个隐式的

1. The universe block encompasses all Go source text. 通用块包括 go 源码文本

2. Each package has a package block containing all Go source text for that package. 每个包都有一个块，包含该包的所有 Go 源代码

3. Each file has a file block containing all Go source text in that file. 每个文件都有一个文件块，包含该文件中的所有 Go 源码

4. Each "if", "for", and "switch" statement is considered to be in its own implicit block. 每个 `if`、`for` 和 `switch` 语句都被认为是在自己的隐式块中

5. Each clause in a "switch" or "select" statement acts as an impliscit block. `switch` 或 `select` 语句中的每个子句都是一个隐式块

我们就犯过错误，命中了最后一条导致了变量 shadow

### 小结
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `关键字` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^s

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)