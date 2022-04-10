---
title: Rust Fn FnMut FnOnce 傻傻分不清
categories: rust
toc: true
---

![](/images/rust-cover.png)

上周文享了[闭包你了解底层实现嘛？](https://mp.weixin.qq.com/s/c1DyCemMTRfPjCsA5BBDTg) 我们要记住，**闭包是由函数和与其相关的引用环境组合而成的实体**

同时闭包引用变量也是有优先级的：优先只读借用，然后可变借用，最后转移所有权。本篇文章看下，如何将闭包当成参数或返回值

### Go 闭包调用
```go
package main

import "fmt"

func test(f func()) {
    f()
    f()
}

func main() {
    a:=1
    fn := func() {
        a++
        fmt.Printf("a is %d\n", a)
    }
    test(fn)
}
```
上面是 go 的闭包调用，我们把 `fn` 当成参数，传给函数 `test`. 闭包捕获变量 a, 做自增操作，同时函数 `fn` 可以调用多次

对于熟悉 go 的人来说，这是非常自然的，但是换成 rust 就有问题了

```rust
fn main() {
    let s = String::from("wocao");
    let f = || {println!("{}", s);};
    f();
}
```
比如上面这段 rust 代码，我如果想把闭包 `f` 当成参数该怎么写呢？[上周分享的闭包](https://mp.weixin.qq.com/s/c1DyCemMTRfPjCsA5BBDTg)我们知道，闭包是匿名的
```
c = hello_cargo::main::closure-2 (0x7fffffffe0e0, 0x7fffffffe0e4)
b = hello_cargo::main::closure-1 (0x7fffffffe0e0)
a = hello_cargo::main::closure-0
```
在运行时，类似于上面的结构体，闭包结构体命名规则 `closure-xxx`, 同时我们是不知道函数签名的

### 引出 Trait
[官方文档](https://doc.rust-lang.org/book/ch13-01-closures.html#storing-closures-using-generic-parameters-and-the-fn-traits) 给出了方案，标准库提供了几个内置的 `trait`, 一个闭包一定实现了 `Fn`, `FnMut`, `FnOnce` 其中一个，然后我们可以用泛型 + trait 的方式调用闭包

```rust
$ cat src/main.rs
fn test<T>(f: T) where
    T: Fn()
{
    f();
}

fn main() {
    let s = String::from("董泽润的技术笔记");
    let f = || {println!("{}", s);};
    test(f);
}

$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/hello_cargo`
董泽润的技术笔记
```
上面将闭包 `f` 以泛型参数的形式传给了函数 `test`, 因为闭包实现了 `Fn` trait. 刚学这块的人可能会糊涂，其实可以理解类比 go `interface`, 但本质还是不一样的
```rust
let f = || {s.push_str("不错");};
```
假如 `test` 声明不变，我们的闭包修改了捕获的变量呢？
```shell
   |
9  |     let f = || {s.push_str("不错");};
   |             ^^  - closure is `FnMut` because it mutates the variable `s` here
   |             |
   |             this closure implements `FnMut`, not `Fn`
10 |     test(f);
```
报错说 closure 实现的 trait 是 `FnMut`, 而不是 `Fn`
```rust
fn test<T>(mut f: T) where
    T: FnMut()
{
    f();
}

fn main() {
    let mut s = String::from("董泽润的技术笔记");
    let f = || {s.push_str("不错");};
    test(f);
}
```
上面是可变借用的场景，我们再看一下 move 所有权的情况
```rust
fn test<T>(f: T) where
    T: FnOnce()
{
    f();
}

fn main() {
    let s = String::from("董泽润的技术笔记");
    let f = || {let _ = s;};
    test(f);
}
```
上面我们把自由变量 s 的所有权 move 到了闭包里，此时 `T` 泛型的特征变成了 `FnOnce`, 表示只能执行一次。那如果 `test` 调用闭包两次呢？
```shell
1 | fn test<T>(f: T) where
  |            - move occurs because `f` has type `T`, which does not implement the `Copy` trait
...
4 |     f();
  |     --- `f` moved due to this call
5 |     f();
  |     ^ value used here after move
  |
note: this value implements `FnOnce`, which causes it to be moved when called
 --> src/main.rs:4:5
  |
4 |     f();
```
编译器提示第一次调用的时候，己经 move 了，再次调用无法访问。很明显此时自由变量己经被析构了 `let _ = s;` 离开词法作用域就释放了，rust 为了内存安全当然不允许继续访问
```shell
fn test<T>(f: T) where
    T: Fn()
{
    f();
    f();
}

fn main() {
    let s = String::from("董泽润的技术笔记");
    let f = move || {println!("s is {}", s);};
    test(f);
    //println!("{}", s);
}
```
那么上面的代码例子, 是否可以运行呢？当然啦，此时变量 s 的所有权 move 给了闭包 `f`, 生命周期同闭包，反复调用也没有副作用
### 深入理解
本质上 Rust 为了内存安全，才引入这么麻烦的处理。平时写 go 程序，谁会在乎对象是何时释放，对象是否存在读写冲突呢？总得有人来做这个事情，Rust 选择在编译期做检查

* `FnOnce` consumes the variables it captures from its enclosing scope, known as the closure’s environment. To consume the captured variables, the closure must take ownership of these variables and move them into the closure when it is defined. The Once part of the name represents the fact that the closure can’t take ownership of the same variables more than once, so it can be called only once.
* `FnMut` can change the environment because it mutably borrows values.
* `Fn` borrows values from the environment immutably.

上面来自官网的解释，`Fn` 代表不可变借用的闭包，可重复执行，`FnMut` 代表闭包可变引用修改了变量，可重复执行 `FnOnce` 代表转移了所有权，同时只能执行一次，再执行的话自由变量脱离作用域回收了

```rust
# mod foo {
pub trait Fn<Args> : FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

pub trait FnMut<Args> : FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait FnOnce<Args> {
    type Output;

    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
# }
```
上面是标准库中，`Fn`, `FnMut`, `FnOnce` 的实现。可以看到 `Fn` 继承自 `FnMut`, `FnMut` 继承自 `FnOnce`
```rust
Fn(u32) -> u32
```
前文例子都是无参数的，其实还可以带上参数

由于 `Fn` 是继承自 `FnMut`, 那么我们把实现 `Fn` 的闭包传给 `FnMut` 的泛型可以嘛？
```rust
$ cat src/main.rs
fn test<T>(mut f: T) where
    T: FnMut()
{
    f();
    f();
}

fn main() {
    let s = String::from("董泽润的技术笔记");
    let f = || {println!("s is {}", s);};
    test(f);
}
```
```shell
$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.47s
     Running `target/debug/hello_cargo`
s is 董泽润的技术笔记
s is 董泽润的技术笔记
```
当然可以看起来没有问题，`FnMut` 告诉函数 `test` 这是一个会修改变量的闭包，那么传进来的闭包不修改当然也没问题

![](/images/fn-context.png)

上图比较出名，由于有继承关系，实现 `Fn` 可用于 `FnMut` 和 `FnOnce` 参数，实现 `FnMut` 可用于 `FnOnce` 参数

### 函数指针
```rust
fn call(f: fn()) {    // function pointer
    f();
}

fn main() {
    let a = 1;

    let f = || println!("abc");     // anonymous function
    let c = || println!("{}", &a);  // closure

    call(f);
    call(c);
}
```
函数和闭包是不同的，上面的例子中 `f` 是一个匿名函数，而 `c` 引用了自由变量，所以是闭包。这段代码是不能执行的
```shell
9  |     let c = || println!("{}", &a);  // closure
   |             --------------------- the found closure
...
12 |     call(c);
   |          ^ expected fn pointer, found closure
```
编译器告诉我们，12 行要求参数是函数指针，不应该是闭包

### 闭包作为返回值
参考[impl Trait 轻松返回复杂的类型](https://rustwiki.org/zh-CN/edition-guide/rust-2018/trait-system/impl-trait-for-returning-complex-types-with-ease.html)，`impl Trait` 是指定实现特定特征的未命名但有具体类型的新方法。 你可以把它放在两个地方：参数位置和返回位置

```rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}

fn main() {
    let f = returns_closure();
    println!("res is {}", f(11));
}
```
在以前，从函数处返回闭包的唯一方法是，使用 `trait` 对象，大家可以试试不用 `Box` 装箱的报错提示
```rust
fn returns_closure() -> impl Fn(i32) -> i32 {
    |x| x + 1
}

fn main() {
    let f = returns_closure();
    println!("res is {}", f(11));
}
```
现在我们可以用 impl 来实现闭包的返回值声明
```rust
fn test() -> impl FnMut(char) {
    let mut s = String::from("董泽润的技术笔记");
    |c| { s.push(c); }
}

fn main() {
    let mut c = test();
    c('d');
    c('e');
}
```
来看一个和引用生命周期相关的例子，上面的代码返回闭包 `c`, 对字符串 s 进行追回作。代码执行肯定报错：
```shell
 --> src/main.rs:3:5
  |
3 |     |c| { s.push(c); }
  |     ^^^   - `s` is borrowed here
  |     |
  |     may outlive borrowed value `s`
  |
note: closure is returned here
 --> src/main.rs:1:14
  |
1 | fn test() -> impl FnMut(char) {
  |              ^^^^^^^^^^^^^^^^
help: to force the closure to take ownership of `s` (and any other referenced variables), use the `move` keyword
  |
3 |     move |c| { s.push(c); }
  |     ^^^^^^^^
```
提示的很明显，变量 `s` 脱离作用域就释放了，编译器也提示我们要 move 所有权给闭包，感兴趣的自己修改测试一下

### 小结
**分享知识，长期输出价值，这是我做公众号的目标**。同时写文章不容易，如果对大家有所帮助和启发，请帮忙点击`在看`，`点赞`，`分享` 三连

关于 `闭包` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)