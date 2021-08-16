---
title: Rust 深入浅出闭包
toc: true
---

在计算机中，闭包 `Closure`, 又称词法闭包 `Lexical Closure` 或函数闭包 `function closures`, 是**引用了自由变量的函数**

被引用的自由变量将和函数一同存在，即使已经离开了创造它的环境也不例外。换句话说，**闭包是由函数和与其相关的引用环境组合而成的实体**

![](https://gitee.com/dongzerun/images/raw/master/img/closer-fn-variables.png)

### 普通函数
函数在 Rust 里是一等公民 (fist class), 即可以做为参数和返回值，和内置类型地位一样。但 Rust 函数也有一些限制，比如不支持默认参数，不支持变参，不允许多值返回等等
```rust
fn inc(x: Option<i32>) -> i32 {
    let a = match x {
        Some(v) => v,
        None => 100,
    };
    a + 1
}

fn inc2(x: Option<i32>) -> i32 {
    let a = x.unwrap_or(100);
    a + 1
}

fn inc3(x: Option<i32>) -> (i32, i32) {
    let a = x.unwrap_or(100);
    (a + 1, a-1)
}
```
但是可以通过其它方法绕过去，比如参数定义为 `Option` 类型，返回值定义 `Result`. 如果想要返回多值，可以用元组来模拟，但本质还是只返回了一个值

`Option<T>` 是泛型枚举，要么匹配成 `Some(v)` 要么是空值 `None`. `Result<T, E>` 定义要么返回泛型 `T`, 要么错误类型 `E`

### 嵌套函数
Rust 允许嵌套函数的，大家在 go 里这么写很常见
```rust
fn main() {
    let x = 1;
    fn test() {
        println!("{}", x);
    }
    test();
}
```
上面的例子，`test` 函数打印变量 `x`, 其它语言是没问题的，但是 rust 不行
```shell
$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
error[E0434]: can't capture dynamic environment in a fn item
 --> src/main.rs:4:24
  |
4 |         println!("{}", x);
  |                        ^
  |
  = help: use the `|| { ... }` closure form instead
```
编译报错，可以看到，只有**闭包能捕获自由变量，普通潜逃函数不行**
### 闭包
```rust
let f = |x: i32| -> i32 { x + 1 };
```
闭包 `||` 代表传入参数，`->` 后面代表返加值，`{}` 大括号里代表函数体
```rust
let f = |x: i32| x + 1;
```
同时如果函数体只有一行，可以省略 `{}`
```rust
let f = |x| x+1;
```
极致一点可以省去 `i32` 类型, 因为 rust 很智能，默认 `x+1` 会推导出闭包 f 返回类型是 `i32`
```rust
let f = |x| x;
```
如果这种情况，就需要根据第一次使用时推导出类型
```rust
fn main() {
    let f = |x| x;
    f(1);
    f('a');
}
```
```shell
$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
error[E0308]: mismatched types
 --> src/main.rs:4:7
  |
4 |     f('a');
  |       ^^^ expected integer, found `char`

error: aborting due to previous error
```
上面的报错，告诉我们类型是 `i32`, 而后来传入的是 `char`

### 闭包底层实现
```rust
fn main() {
    let v1 = 100;
    let v2 = 100;
    let a = |x: i32| x;
    let b = |x: i32| x + v1;
    let c = |x: i32| x + v1 + v2;
    
    assert_eq!(size_of(&a), 0);
    assert_eq!(size_of(&b), 8);
    assert_eq!(size_of(&c), 16);
}

fn size_of<T>(_: &T) -> usize {
    std::mem::size_of::<T>()
}
```
定义了三个闭包，`a` 普通的匿名函数，`b` 引用外部变量 v1, `c` 引用两个外部变量 v1, v2
```rust
(gdb)
c = hello_cargo::main::closure-2 (0x7fffffffe0e0, 0x7fffffffe0e4)
b = hello_cargo::main::closure-1 (0x7fffffffe0e0)
a = hello_cargo::main::closure-0
v2 = 100
v1 = 100
(gdb) ptype a
type = struct hello_cargo::main::closure-0
(gdb) ptype b
type = struct hello_cargo::main::closure-1 (
  *mut i32,
)
(gdb) ptype c
type = struct hello_cargo::main::closure-2 (
  *mut i32,
  *mut i32,
)
(gdb) p/x &v1
$1 = 0x7fffffffe0e0
(gdb) p/x &v2
$2 = 0x7fffffffe0e4
```
通过返汇编，我们可以看到，**rust 里匿名函数其实和闭包是一样的结构，底层实现一样的**。

![](https://gitee.com/dongzerun/images/raw/master/img/closure-struct.png)

闭包 `a` 是空结构体，所以大小 0 字节，而 `b` 拥有一个指针字段，64位平台上当然是 8 字节，`c` 就是 16 字节长度。打印地址，可以看到捕获的就是对应 v1, v2

同时要注意到，这个例子里，闭包在二进制包 text 代码段中用 `hello_cargo::main::closure-N` 结构体来表示，编号依次递增的，同时在该例子中结构体捕获的变量，其实是引用形式

### 闭包与所有权
```rust
fn main(){
    let mut a = 1;
    let mut inc = || {a+=1;a};
    inc();
    inc();
    println!("now a is {}", a);
}
```
关于所有权，可以参考[Rust Ownership 三原则](https://mp.weixin.qq.com/s/3apdPoiPWmDDU7jMRIJkrA)。闭包 `inc` 捕获自由变量 a, 然后自增
```shell
# cargo run
   Compiling hello_cargo v0.1.0 (/root/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 2.35s
     Running `target/debug/hello_cargo`
now a is 3
```
可以看到，当 `inc` 执行两次后，a 的结果是 3, 由上面可以知道此时 `inc` 以引用的方式捕获变量 a
```rust
fn main(){
    let s = String::from("test");
    let f = || {let _s = s;println!("{}", _s)};
    f();
    f();
}
```
这是转移所有权的例子，堆上的字符串变量 `s`, 所有权转给了闭包中的临时变量 `_s`
```rust
fn main(){
    let mut s = String::from("test");
    let mut f = || {s.push('a');println!("{}", s)};
    f();
    f();
}
```
例子中闭包 f 修改字符串 s, 并打印
```shell
$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 0.55s
     Running `target/debug/hello_cargo`
testa
testaa
```
执行后打印，输出和预期一样
```shell
# cargo run
   Compiling hello_cargo v0.1.0 (/root/zerun.dong/code/rusttest/hello_cargo)
error[E0382]: use of moved value: `f`
 --> src/main.rs:5:5
  |
4 |     f();
  |     --- `f` moved due to this call
5 |     f();
  |     ^ value used here after move
```
函数只能执行一次，因为当第一次执行时，`_s` 随后析构释放了内存，所以编译器报错
```rust
fn main(){
    let s = String::from("test");
    let f = move || {println!("{}", s)};
    f();
    f();
}
```
如果想把变量转移给闭包，就需要显示使用 `move` 关键字，此时字符串 `test` 所有权转给了闭包 `f`, 当然可以多次执行，直到 `f` 离开作用域后一起析构，感兴趣的可以返汇编自己看一下

可以得出结论：**闭包捕获变量优先只读引用，然后可变引用，最后 move 所有权**
### Rust 2021
[Disjoint capture in closures](https://doc.rust-lang.org/nightly/edition-guide/rust-2021/disjoint-capture-in-closures.html, "Disjoint capture in closures") 这是 rust2021 关于闭包的更新

对于闭包 `|| a.x + 1`, 2018 的实现是捕获整个结构体 a, 但是现在只捕获所需要用的 x

这个特性会导致一些对像在不同时间点被释放 dropped, 或是影响了闭包是否实现 Send 或 Clone trait, 所以 cargo 会插入语句 `let _ = &a` 引用完整结构体来修复这个问题。这个变动其实很大，细节可以参考官方文档

### 小结
本文先分享这些，下一篇讲讲闭包做为参数和返回值的限制，以及 `Fn`, `FnOnce` 和 `FnMut`, 这些点比较难理解

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `泛型` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)