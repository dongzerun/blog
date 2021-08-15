---
title: Rust 让人头大的引用与借用
categories: rust
toc: true
---

本篇尽量深入浅出，不想学 `Rust` 的也可以读读，多种语言对比很有很大的收获，`Go` 再好也不是所有场景通吃^_^

上周分享了[Rust Ownership 三原则](https://mp.weixin.qq.com/s/3apdPoiPWmDDU7jMRIJkrA), 要谨记这三点：
* 每个值 value, 都有一个所有者 owner
* 同一时间，一个值只能有一个所有者 owner
* 当所有者 owner 离开作用域，对应的值会自动 dropped

![](https://gitee.com/dongzerun/images/raw/master/img/func-take-ownership.jpg)

`hello` 是在堆上分配的字符串，owner 是 `s`, 当以参数传给函数 `takes_ownership` 后，所有权 Move 给了 `some_string`. 也就是当函数返回时 `hello` 字符串会被 dropped，所以 main 第 4 行如果打印会报错，因为己经 Move 走了

如果 main 想继续打印 `hello` 怎么办呢？`takes_ownership` 可以将 `hello` return 出去，这样 owner 会再度 Move 给 `s`. 但真的很不方便

### 引用介绍
这时`引用`出场了，不移动所有权，只出借数据
```rust
fn main() {
    let s = String::from("hello");
    no_takes_ownership(&s);
    println!("main {}", s)
}

fn no_takes_ownership(some_string: &str) {
    println!("func {}", some_string);
}
```
```rust
zerun.dong$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.86s
     Running `target/debug/hello_cargo`
func hello
main hello
```
最后运行都可以打印，但是 `no_takes_ownership` 并没有拥有 owner, `&str` 表示对字符串的只读引用。`no_takes_ownership(&s)`, 这里没有直接写 `s`, 而是取引用 `&s`
### 指针与引用
```rust
fn main() {
    let a = 0;
    let _r = &a;
}
```
上面是最简单的引用例子，`a` 是一个 int32 类型，值为 0. 然后 `_r` 是对 a 的引用
```
Dump of assembler code for function hello_cargo::main:
src/main.rs:
1	fn main() {
   0x000055555555b6c0 <+0>:	sub    $0x10,%rsp

2	    let a = 0;
   0x000055555555b6c4 <+4>:	movl   $0x0,0x4(%rsp)

3	    let _r = & a;
=> 0x000055555555b6cc <+12>:	lea    0x4(%rsp),%rax
   0x000055555555b6d1 <+17>:	mov    %rax,0x8(%rsp)

4	}
   0x000055555555b6d6 <+22>:	add    $0x10,%rsp
   0x000055555555b6da <+26>:	retq
End of assembler dump.
```
可以看到 `a` 变量分配在栈上 rsp + 0x4, 初始值是 0, 然后第 3 行反汇编可以看到，`lea` 取了 `a` 的地址，然后将地址传递给栈上的 `_r`

**本质上 rust 引用和普通的指针区别不大，只是在编译阶段，附加了各种规则而己，比如引用的对像不能为空**
### 借用规则
`引用 (reference)` 不获取所有权，坚持单一所有者和单一职责，解决了共享访问障碍。
按引用传递对象的方式称作`借用 (borrow)`, 这比转移所有权更有效

* **一个引用的生命周期，一定不会超过其被引用的时间。这显而易见的，为了防止悬垂引用**
* **如果存在一个值的可变借用，那么在该借用作用域内，不允许有其它引用(读或写)**
* **没有可变借用的情况下，允许存在多个对同一值的不可变借用**

```rust
fn main() {
    let y: &i32;
    {
        let x = 5;
        y = &x;
    }

    println!("{}", y);
}
```
```
zerun.dong$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
5 |         y = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |
8 |     println!("{}", y);
  |                    - borrow later used here

error: aborting due to previous error
```
这个例子执行会报错，原因显而易见，`y` 要引用变量 `x`, 但是 `x` 在语句块 {} 中，离开这个词法作用域后就失效了，`y` 就成了悬垂空引用肯定不行。
```rust
fn main(){
    let mut a = String::from("hello");
    let a_ref = &mut a;
    println!("a is {}", a);
    a_ref.push('!');
}
```
```
zerun.dong$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
error[E0502]: cannot borrow `a` as immutable because it is also borrowed as mutable
 --> src/main.rs:4:25
  |
3 |     let a_ref = &mut a;
  |                 ------ mutable borrow occurs here
4 |     println!("a is {}", a);
  |                         ^ immutable borrow occurs here
5 |     a_ref.push('!');
  |     ----- mutable borrow later used here
```
上面的代码，`a_ref` 是可变借用，然后调用 `a_ref.push` 修改字符串，同时第 4 号要打印原来的 owner `a`, 这时报错

**原因在于，`a_ref` 是可变借用，在他的的作用域内，不允许存在其它不可变借用或是可变借用，这里 `println!` 是对 `a` 的不可变借用**

我一开始困惑的点在于，**这个作用域到底有多大！！！远古版本是词法作用域，后来改进了，变成第一次借用开始，直到最后一次调用结束，这样作用域小很多**
```rust
fn main(){
    let mut a = String::from("hello");
    let a_ref = &mut a;
    a_ref.push('!');
    println!("a is {}", a);
}
```
```
zerun.dong$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.51s
     Running `target/debug/hello_cargo`
a is hello!
```
经过修改之后，把 `println` 放到 `push` 后面就可以了
### 容易困惑的点
一个结构体里也会有引用，初学都很容易晕
```rust
struct Stu{
  Age: int32
  Name: &str
}
```
这里面 `Name` 是一个字符串的引用，所以实例化的 Stu 对象没有 Name 的所有权，那么就要符合上面的借用规则。说白了，就是内容谁负责释放的问题

还有一个是类型的方法，第一个参数要写成 `&self` 或是 `&mut self`, 如果写成 `self` 那么函数就会捕捉类型的所有权，函数执行一次，就无法再使用这个类型
```rust
struct Number{
    num: u8
}

impl Number {
    fn get_num(self) -> u8 {
        self.num
    }
}

fn main(){
    let num = Number{num:10};
    println!("get num {}", num.get_num());
    println!("get num {}", num.get_num());
}
```
上面代码很简单，`Number` 类型有一个 `get_num` 方法获取变量 num 值，但是 main 如果调用两次 println 编译时就会报错
```
zerun.dong$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
error[E0382]: use of moved value: `num`
  --> src/main.rs:15:28
   |
13 |     let num = Number{num:10};
   |         --- move occurs because `num` has type `Number`, which does not implement the `Copy` trait
14 |     println!("get num {}", num.get_num());
   |                                --------- `num` moved due to this method call
15 |     println!("get num {}", num.get_num());
   |                            ^^^ value used here after move
   |
note: this function consumes the receiver `self` by taking ownership of it, which moves `num`
  --> src/main.rs:7:16
   |
7  |     fn get_num(self) -> u8 {
   |                ^^^^
```
可以看到编译器给了提示，因为 `get_num(self)` 定义为 self 就会 takeing ownership, which moves `num`

所以类型方法的第一个参数要用引用，写 Go 的人肯定容易蒙蔽。同理还有 match 的匹配语句块，也会 move 对像，需要引用
### 小结
只要我还在更，就说明没被劝退。在学 rust 时如果有困惑，一定要记住，所有权，借用与生命周期就是为了保证内存安全。Go 语言有 GC 可以自动清理垃圾，而 rust 为了零运行时成本，把这一部份工作移到编译器

今天的分享就这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连

关于 `Rust 引用借用` 大家有什么看法，欢迎留言一起讨论，大牛多留言，下一篇分享生命周期 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)