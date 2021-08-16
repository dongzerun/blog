---
title: 揭开智能指针 Box 的神秘面纱
toc: true
---

熟悉 c++ 的肯定知道 `shared_ptr`, `unique_ptr`, 而 Rust 也有智能指针 `Box`, `Rc`, `Arc`, `RefCell` 等等，本文分享 `Box` 底层实现

`Box<T>` 会在堆上分配空间，存储 T 值，并返回对应的指针。同时 `Box` 也实现了 trait `Deref` 解引用和 `Drop` 析构，当 `Box` 离开作用域时自动释放空间

### 入门例子
例子来自 [the rust book](https://doc.rust-lang.org/book/ch15-01-box.html), 为了演示方便，去掉打印语句

```rust
fn main() {
    let _ = Box::new(0x11223344);
}
```
将变量 0x11223344 分配在堆上，所谓的装箱，java 同学肯定很熟悉。让我们挂载 docker, 使用 rust-gdb 查看汇编实现
```
Dump of assembler code for function hello_cargo::main:
   0x000055555555bdb0 <+0>:	sub    $0x18,%rsp
   0x000055555555bdb4 <+4>:	movl   $0x11223344,0x14(%rsp)
=> 0x000055555555bdbc <+12>:	mov    $0x4,%esi
   0x000055555555bdc1 <+17>:	mov    %rsi,%rdi
   0x000055555555bdc4 <+20>:	callq  0x55555555b5b0 <alloc::alloc::exchange_malloc>
   0x000055555555bdc9 <+25>:	mov    %rax,%rcx
   0x000055555555bdcc <+28>:	mov    %rcx,%rax
   0x000055555555bdcf <+31>:	movl   $0x11223344,(%rcx)
   0x000055555555bdd5 <+37>:	mov    %rax,0x8(%rsp)
   0x000055555555bdda <+42>:	lea    0x8(%rsp),%rdi
   0x000055555555bddf <+47>:	callq  0x55555555bd20 <core::ptr::drop_in_place<alloc::boxed::Box<i32>>>
   0x000055555555bde4 <+52>:	add    $0x18,%rsp
   0x000055555555bde8 <+56>:	retq
End of assembler dump.
```
关键点就两条，`alloc::alloc::exchange_malloc` 在堆上分配内存空间，然后将 `0x11223344` 存储到这个 malloc 的地址上

函数结束时，将地址传递给 `core::ptr::drop_in_place` 去释放，因为编译器知道类型是 `alloc::boxed::Box<i32>`, 会掉用 `Box` 相应的 drop 函数

单纯的看这个例子，`Box` 并不神秘，对应汇编实现，和普通指针没区别，一切约束都是编译期行为

### 所有权
```rust
fn main() {
    let x = Box::new(String::from("Rust"));
    let y = *x;
    println!("x is {}", x);
}
```
这个例子中将字符串装箱，其实没必要这么写，因为 `String` 广义来讲本身就是一种智能指针。这个例子会报错
```
3 |     let y = *x;
  |             -- value moved here
4 |     println!("x is {}", x);
  |                         ^ value borrowed here after move
```
`*x` 解引用后对应 `String`, 赋值给 y 时执行 move 语义，所有权不在了，所以后续 println 不能打印 x

```rust
let y = &*x;
```
可以取字符串的不可变引用来 fix

### 底层实现
```rust
pub struct Box<
    T: ?Sized,
    #[unstable(feature = "allocator_api", issue = "32838")] A: Allocator = Global,
>(Unique<T>, A);
```
上面是 `Box` 的定义，可以看到是一个元组结构体，有两个泛型参数：`T` 代表任意类型，`A` 代表内存分配器。标准库里 `A` 是 Gloal 默认值。其中 `T` 有一个泛型约束 `?Sized`, 表示在编译时可能道理类型大小，也可能不知道

```rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<#[may_dangle] T: ?Sized, A: Allocator> Drop for Box<T, A> {
    fn drop(&mut self) {
        // FIXME: Do nothing, drop is currently performed by compiler.
    }
}
```
这是 `Drop` 实现，源码里也说了，由编译器实现
```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized, A: Allocator> Deref for Box<T, A> {
    type Target = T;

    fn deref(&self) -> &T {
        &**self
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized, A: Allocator> DerefMut for Box<T, A> {
    fn deref_mut(&mut self) -> &mut T {
        &mut **self
    }
}
```
实现了 `Deref` 可以定义解引用行为，`DerefMut` 可变解引用。所以 `*x` 对应着操作 `*(x.deref())`

### 适用场景
官网提到以下三个场景，本质上 `Box` 和普通指针区别不大，所以用处不如 `Rc`, `Arc`, `RefCell` 广

* 当类型在编译期不知道大小，但代码场景还要求确认类型大小的时候
* 当你有大量数据，需要移动所有权，而不想 copy 数据的时候
* trait 对象，或者称为 dyn 动态分发常用在一个集合中存储不同的类型上，或者参数指定不同的类型

官网提一一个链表的实现
```rust
enum List {
    Cons(i32, List),
    Nil,
}
```
上面代码是无法运行的，道理也很简单，这是一种递归定义。对应 c 代码也是不行的，我们一般要给 next 类型定义成指针才行
```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
}
```
官网给的解决方案，就是将 next 变成了指针 `Box<List>`, 算是常识吧，没什么好说的

### 小结
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Box` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)
