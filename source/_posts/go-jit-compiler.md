---
title: 如何用 Go 实现 Jit compiler
categories: go
toc: true
---

原文作者是 **Sidhartha Mani** 首发于 [Medium](https://medium.com/kokster/writing-a-jit-compiler-in-golang-964b61295f),曾由 [jiangwei161002010](https://github.com/jiangwei161002010) 翻译后发布在 [Go 语言中文网](https://studygolang.com/articles/12730, "Go 语言中文网")

---
JIT([Just-In-Time](https://en.wikipedia.org/wiki/Just-in-time_compilation, "Just-In-Time")) Compiler 是指在运行期，实时生成机器码的服务。机器码 (machine code) 是 cpu 识别的机器指令

![](https://gitee.com/dongzerun/images/raw/master/img/assember-machine-code.jpg)

一般我们都是 Go 代码翻译成汇编，然后转换成 cpu 可识别的机器码。相比于其它代码 (`fmt.Printf`), Jit Compiler 是在运行期生成，而不是编译期 (即我们常用的 go build)

Go 是静态类型，编译完后生成二进制可执行文件。看起来不可能生成任意代码，更不用说去执行这些代码。然而，向运行中的程序注入指令是可行的，这就是所谓的 `Type Magic`, 类型魔法，可以将任意类型转换成其它类型

### 生成 x64 机器码
机器码是一串字节数据，对于 cpu 有特殊的意义(识别后执行)。本文用来测试的是 x86 处理器，因此，采用 [x64 instruction set](https://software.intel.com/content/www/us/en/develop/articles/introduction-to-x64-assembly.html, "x86 instruction set") 指令集，平台相关，所以你在非 x86 上肯定无法运行

本文测试 Jit 让我们用 x86 指令，打印 `Hello World!`
```c
write(int fd, const void *buf, size_t count)
```
众所周知，肯定是一个 `write` 的系统调用，`fd` 是输出的文件描述符，标准输出是 1, 第二个参数是 `Hello World!` 字符串指针，第三个参数 count 是要打印的字符数 12

![](https://gitee.com/dongzerun/images/raw/master/img/syscall-register.jpg)

同时我们也要知道，系统调用是 `Syscall` 指令，rax 告诉 [linux 要执行的具体系统调用函数编号](https://filippo.io/linux-syscall-table/https://filippo.io/linux-syscall-table/, "系统调用函数编号")，`write` 是 1, 函数的参数由 6 个寄存器完成，由于 `write` 只有三个参数，所有第一个参数 rdi 是 1 (文件描述符)，第二个参数 rsi 是字符串地址，但是暂时无法确定，稍后咱们再看。第三个参数 rdx 是字符串长度 12

```
0:  48 c7 c0 01 00 00 00  mov rax,0x1 
7:  48 c7 c7 01 00 00 00  mov rdi,0x1
e:  48 c7 c2 0c 00 00 00  mov rdx,0xc
```
放在一起，上面就是对应的机器码，右面是汇编代码 (网上有很多，根据汇编指令生成机器码的，大家不用过于考究，知道意思就行)。**那么现在唯一的问题在于，如何确定字符串指针的地址**

这个地址一定是运行时有效的，否则就会发生 segmentfault 错误。这个例子中，我们可以把 `Hello world!` 字符串放到可执行的执令 return 的后面。就是安全的，因为 cpu 执行完就返回了，不会走到后面

由于在返回指令下达之前不知道返回后的地址，所以可以使用一个临时的位置占位符 placeholder, 一旦知道了数据的地址，就用正确的地址来代替。这就是链接器所遵循的确切程序。链接的过程只是将这些地址填入正确的数据或函数。
```
15: 48 8d 35 00 00 00 00 lea rsi,[rip+0x0] # 0x15
1c: 0f 05                syscall
1e: c3                   ret
```
上面的代码中，`lea` 指令是用来取字符串 `Hello World!` 地址的，但是这里是当前的位置 (我们知道 rip 寄存器保存当前执行代码的地址，0x0 是偏移量)。为什么指向自己呢？因为字符串还没有保存呢，不知道具体位置。其中 `0F 05` 是 `Syscall` 对应的机器码
```
1f: 48 65 6c 6c 6f 20 57 6f 72 6c 64 21   // Hello World!
```
我们现在把字符串放到 `ret` 指令后面，那么此时字符串的相对地址就确定了
```
0:  48 c7 c0 01 00 00 00 mov rax,0x1
7:  48 c7 c7 01 00 00 00 mov rdi,0x1
e:  48 c7 c2 0c 00 00 00 mov rdx,0xc
15: 48 8d 35 03 00 00 00 lea rsi,[rip+0x3]# 0x1f
1c: 0f 05                syscall
1e: c3                   ret
1f: 48 65 6c 6c 6f 20 57 6f 72 6c 64 21   // Hello World! 
```
然后为了可读性我们用 golang int16 slice 保存上面的机器码
```go
printFunction := []uint16{
0x48c7, 0xc001, 0x0, // mov %rax,$0x1
0x48, 0xc7c7, 0x100, 0x0, // mov %rdi,$0x1
0x48c7, 0xc20c, 0x0, // mov 0x13, %rdx
0x48, 0x8d35, 0x400, 0x0,// lea 0x4(%rip), %rsi
0xf05,// syscall
0xc3cc,// ret
0x4865, 0x6c6c, 0x6f20,// Hello_(whitespace)
0x576f, 0x726c, 0x6421, 0xa,// World!
} 
```
切片保存的和上面机器码有些偏差，是为了对齐，更好看一些 (ret 机器码是 c3, 后面多了一个 cc, 这是一个 no-op 指令)。所以 lea 取地址即为 rip+0x4

### 将 slice 转换成函数
切片中的指令，必须转换成函数才能调用，下面的 go 代码展示了如何转换
```go
type printFunc func()
unsafePrintFunc := (uintptr)(unsafe.Pointer(&printFunction)) 
printer := *(*printFunc)(unsafe.Pointer(&unsafePrintFunc)) 
printer()
```
Go 函数值只是一个指向 C 函数指针的指针（注意两级指针。从切片到函数的转换，首先要提取一个指向存放可执行代码的数据结构的指针（这里就是 slice 的指针）。这真指针被存储在 `unsafePrintFunc` 中。指向 `unsafePrintFunc` 的指针可以被转换成所需要的函数类型

**这种方法只适用于没有参数或返回值的函数**。在调用有参数或返回值的函数时，需要提前创建一个栈空（go 是 stack-based 函数调用规约，感兴趣的网上搜一下）

### 让函数执行
上述函数实际上不会运行。这是因为 Go 将所有的数据存储在二进制文件的 data 段。这一部分的数据被设置了 `No-Execute` 标志，防止它被执行(这不只是 GO, 所有都是这样)

`printFunction` 片段中的数据需要存储在一块可执行的内存中。这可以通过移除 `printFunction` 片断上的 `No-Execute` 标志或将其复制到可执行的内存位置来实现

在下面的代码中，数据被复制到一个新分配的可执行的内存中（使用 `mmap`）。这种方法比较好，因为只有在整个页面上才能设置不执行标志--很容易无意中使数据部分的其他部分成为可执行的，变得很不安全
```go
 executablePrintFunc, err := syscall.Mmap(
  -1,
  0,
  128,
  syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC,
  syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS)
 if err != nil {
  fmt.Printf("mmap err: %v", err)
 }
j := 0
 for i := range printFunction {
  executablePrintFunc[j] = byte(printFunction[i] >> 8)
  executablePrintFunc[j+1] = byte(printFunction[i])
  j = j + 2
 }
```
上面代码用 `Mmap` 创建了一块匿名的可执行私有内存区域，然后把 `printFunction` 代码按照机器码顺序复制到该区域。重点是 `syscall.PROT_EXEC` 标记

```go
package main

import (
	"fmt"
	"syscall"
	"unsafe"
)

type printFunc func()

func main() {
	printFunction := []uint16{
		0x48c7, 0xc001, 0x0, // mov %rax,$0x1
		0x48, 0xc7c7, 0x100, 0x0, // mov %rdi,$0x1
		0x48c7, 0xc20c, 0x0, // mov 0x13, %rdx
		0x48, 0x8d35, 0x400, 0x0, // lea 0x4(%rip), %rsi
		0xf05,                  // syscall
		0xc3cc,                 // ret
		0x4865, 0x6c6c, 0x6f20, // Hello_(whitespace)
		0x576f, 0x726c, 0x6421, 0xa, // World!
	}
	executablePrintFunc, err := syscall.Mmap(
		-1,
		0,
		128,
		syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC,
		syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS)
	if err != nil {
		fmt.Printf("mmap err: %v", err)
	}

	j := 0
	for i := range printFunction {
		executablePrintFunc[j] = byte(printFunction[i] >> 8)
		executablePrintFunc[j+1] = byte(printFunction[i])
		j = j + 2
	}

	type printFunc func()

	unsafePrintFunc := (uintptr)(unsafe.Pointer(&executablePrintFunc))
	printer := *(*printFunc)(unsafe.Pointer(&unsafePrintFunc))
	printer()
}
```
上面就是最终可执行的代码，执行后只是打印 `Hello World!`
```shell
~# strace ./jit
execve("./jit", ["./jit"], 0x7ffe27f7ef70 /* 17 vars */) = 0
arch_prctl(ARCH_SET_FS, 0x54e5d0)       = 0
sched_getaffinity(0, 8192, [0, 1])      = 8

......

readlinkat(AT_FDCWD, "/proc/self/exe", "/root/jit", 128) = 9
fcntl(0, F_GETFL)                       = 0x2 (flags O_RDWR)
futex(0xc000036950, FUTEX_WAKE_PRIVATE, 1) = 1
fcntl(1, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(2, F_GETFL)                       = 0x2 (flags O_RDWR)
mmap(NULL, 128, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5eabb00000
write(1, "Hello World!", 12Hello World!)            = 12
exit_group(0)                           = ?
+++ exited with 0 +++
```
`strace` 可以看到完整系统调用，先是 `mmap` 然后打印字符串到标准输出。就这些，**祝大家玩的开心！！**
### 小结
今天的分享就这些，`JIT` 是一个非常庞大的 topic, 还是蛮有意思的。写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Go JIT` 大家有什么看法，欢迎留言一起讨论，大牛多留言，下一篇分享生命周期 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)


