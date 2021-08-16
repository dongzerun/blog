---
title: Rust ownership 三原则
toc: true
---

本文参考 [The Book Understanding Ownership 4.1](https://doc.rust-lang.org/stable/book/ch04-01-what-is-ownership.html, "The Book Understanding Ownership 4.1"), 还在更就说明没劝退^_^

Rust 最核心的概念就是 `Ownership` 所有权，带 GC 功能的语言，可以在运行期由 runtime 扫堆内存，释放没有被引用的垃圾对象，比如现在用的 go. 对于像 c/c++ 这类语言，就需要用户自行管理内存的分配与释放

而 Rust 采用 `Ownership` 的概念，并在编译器附加各种检查规则，来实现内存管理。注意，Rust 大部分工作都是在编译期完成，所以运行时没有额外开销。下面是三原则：

* **每个值 value, 都有一个所有者 owner**
* **同一时间，一个值只能有一个所有者 owner**
* **当所有者 owner 离开作用域，对应的值会自动 dropped**

### RAII
先来看一下离开作用域，自动释放
```rust
fn main() {
    let _s = String::from("hello");
}
```
最简单的代码，只有一行，在堆上分配字符串，反汇编观察下如何管理内存
```
Dump of assembler code for function hello_cargo::main:
src/main.rs:
1	fn main() {
   0x000055555555d190 <+0>:	sub    $0x18,%rsp

2	    let _s = String::from("hello");
   0x000055555555d194 <+4>:	mov    %rsp,%rdi
   0x000055555555d197 <+7>:	lea    0x2bf86(%rip),%rsi        # 0x555555589124
   0x000055555555d19e <+14>:	mov    $0x5,%edx
   0x000055555555d1a3 <+19>:	callq  0x55555555c480 <<alloc::string::String as core::convert::From<&str>>::from>

3	}
=> 0x000055555555d1a8 <+24>:	mov    %rsp,%rdi
   0x000055555555d1ab <+27>:	callq  0x55555555c700 <core::ptr::drop_in_place<alloc::string::String>>
   0x000055555555d1b0 <+32>:	add    $0x18,%rsp
   0x000055555555d1b4 <+36>:	retq
End of assembler dump.
```
第 2 行的汇编，调用 `core::convert::From` 创建字符串变量。然后第 3 行，main 结束时，自动调用 `core::ptr::drop_in_place` 释放字符串

离开作用域自动析构，这一点很像 c++ 的 `RAII` (Resource Acquisition Is Initialization), 只不过 rust 是离开作用域调用 `drop trait`

### 所有权
```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;

    println!("s1 = {}, s2 = {}", s1, s2);
}
```
这段代码，如果是 go 肯定可以运行，但是 rust 不行
```
hello_cargo# cargo run
   Compiling hello_cargo v0.1.0 (/root/zerun.dong/code/rusttest/hello_cargo)
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:34
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 |
5 |     println!("s1 = {}, s2 = {}", s1, s2);
  |                                  ^^ value borrowed here after move
```
执行报错，因为 s1 值的所有权己经 move 给 s2 了，原来的 s1 己经处于不可用状态。**move occurs because `s1` has type `String`, which does not implement the `Copy` trait**

![](https://gitee.com/dongzerun/images/raw/master/img/s1-s2.jpg)

可以看一下字符串的结果，和 go 一样，string header 里面有 pointer 指向堆内存。如果 s1, s2 浅拷贝，同时将 pointer 指向堆上的数据，那么离开作用域后，会将同一片内存释放两次！！！

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();

    println!("s1 = {}, s2 = {}", s1, s2);
}
```
可以调用 `s1.clone()` 进行深拷贝，解决这个问题。但是每次都进行内存复制很低效，所以后面会介绍 `引用 references` 的概念

### 所有权与函数
```rust
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it's okay to still
                                    // use x afterward

} // Here, x goes out of scope, then s. But because s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```
这是官网的一个例子，当 `takes_ownership` 执行时，s 对应值的所有权就转移到函数中了，离开作用域后进行释放，如果 main 函数想再使用就会报错

但是同时，x 是一个 int 值，`makes_copy` 函数执行会 copy 这个 value, 而不是转移所有权
### Move,Copy,Clone
在 Rust 中，如果类型没有实现 `Copy`, 那么该类型进行赋值，传参，返回值时都是 `Move` 语义，有点拗口，不直观

上一段字符串就是一个 `Move`, 来看一个整型例子
```rust
zerun.dong$ cat src/main.rs
fn main() {
    let s1 = 1234;
    let s2 = s1;

    println!("s1 = {}, s2 = {}", s1, s2);
}

zerun.dong$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/hello_cargo`
s1 = 1234, s2 = 1234
```
这里 s1 默认就是 i32 类型，实现了 `Copy` 语义，所以之后 s1 也可以使用

来看一下哪些类型默认实现了 `Copy` 语义，一般都是标量，也就是栈上分配，在编译期就确定大小的类型：
* All the integer types, such as u32.
* The Boolean type, bool, with values true and false.
* All the floating point types, such as f64.
* The character type, char.
* Tuples, if they only contain types that also implement Copy. For example, (i32, i32) implements Copy, but (i32, String) does not.

那么对于自定义类型，比如 struct 是如何处理的呢？？？
```rust
struct Data {
    a: i64,
    b: i64,
}

fn test(d: Data) {
    let _d = d;
}

fn main() {
    let d = Data{a:1,b:1};
    test(d);
    println!("a is {}, b is {}", d.a, d.b);
}

zerun.dong$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
error[E0382]: borrow of moved value: `d`
  --> src/main.rs:14:39
   |
12 |     let d = Data{a:1,b:1};
   |         - move occurs because `d` has type `Data`, which does not implement the `Copy` trait
13 |     test(d);
   |          - value moved here
14 |     println!("a is {}, b is {}", d.a, d.b);
   |                                       ^^^ value borrowed here after move
```
可以看到，运行报错，原因是 d 在调用 `test` 时己经 move 走了，所以 main 里不能再使用变量 d. 但问题在于 struct Data 成员都是整型啊，默认都是实现了 `Copy` 的
```rust
#[derive(Copy, Clone)]
struct Data {
    a: i64,
    b: i64,
}
```
这里需要在 struct 标注一下 `#[derive(Copy, Clone)]`, 这样该自定义类型自动实现 `Copy trait`, 这里要求所有字段都己经实现了 `Copy`
```rust
enum E1 {
    Text,
    Digit,
}
struct S2 {
    u: usize,
    e: E1,
    s: String,
}
impl Clone for S2 {
    fn clone(&self) -> Self {
        // 生成新的E1实例
        let e = match self.e {
            E1::Text => E1::Text,
            E1::Digit => E1::Digit,
        };
        Self {
            u: self.u,
            e,
            s: self.s.clone(),
        }
    }
}
```
对于 `struct S2`, 由于 S2 的 field 中有 String 类型，String 类型没有实现 `Copy trait`, 所以 S2 类型就不能实现 `Copy trait`

S2 中也包含了 E1 类型，E1 类型没有实现 `Clone` 和 `Copy trait`，但是我们可以自己实现S2类型的 `Clone trait`, 在 `Clone::clone` 方法中生成新的 E1 实例，这就可以 clone 出新的 S2 实例

这块概念参考[rust中move、copy、clone、drop和闭包捕获](https://www.jianshu.com/p/ea1b96cbf0a1, "rust中move、copy、clone、drop和闭包捕获"), 太复杂了，涉及到闭包的，后面再讲。多个语言，对比学习，还是蛮有意思的 ^_^

### 小结
今天的分享就这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连

关于 Rust Ownership 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)