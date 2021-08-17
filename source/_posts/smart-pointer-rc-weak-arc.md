---
title: 剖析智能指针 Rc Weak 与 Arc
categories: rust
toc: true
---
![](https://gitee.com/dongzerun/images/raw/master/img/rust-cover.jpeg)

我们知道 rust ownership 有[三原则](https://mp.weixin.qq.com/s/3apdPoiPWmDDU7jMRIJkrA):

* 每个值 value, 都有一个所有者 owner
* 同一时间，一个值只能有一个所有者 owner
* 当所有者 owner 离开作用域，对应的值会自动 dropped

但有些时候一个值，要被多个变量共享。同时无法用引用来解决，因为不确定哪个变量最后结束，无法确定生命周期。所以这时候 `Rc` reference count 智能指针派上用场了， **`Rc` 共享所有权，每增加一个引用时，计数加一，离开作用域时减一，当引用计数为 0 时执行析构函数**

`Rc` 线程不安全，跨线程时需要使用 `Arc`, 其中 `A` 是 atomic 原子的意思

### 查看引用计数
```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("https://mytechshares.com/"));
    println!("ref count is {}", Rc::strong_count(&a));
    let b = Rc::clone(&a);
    println!("ref count is {}", Rc::strong_count(&a));
    {
        let c = Rc::clone(&a);
        println!("ref count is {}", Rc::strong_count(&a));
        println!("ref count is {}", Rc::strong_count(&c));
    }
    println!("ref count is {}", Rc::strong_count(&a));
}
```
`strong_count` 查看引用计数，变量 `c` 在 inner 词法作用域内，在离开作用域前打印查看引用计数
```shell
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/hello_cargo`
ref count is 1
ref count is 2
ref count is 3
ref count is 3
ref count is 2
```
每增加一次引用，计数都会加一，在 inner 语句块中，count 是 3，但当离开后，计数减为 2

### 循环引用
熟悉 Garbage Collection 的都知道，有的 GC 算法就靠的引用计数，比如 python 的实现。比如 Redis object 同样用 rc 来实现，由于 redis `rc` 并不暴露给用户，只要写代码时注意引用方向，就不会写出循环引用，但是 rust `Rc` 就无法避免，先来看个例子

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    next: Option<Rc<RefCell<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("Dropping https://mytechshares.com 董泽润的技术笔记");
    }
}

fn main() {
    let first = Rc::new(RefCell::new(Node {next: None}));
    let second = Rc::new(RefCell::new(Node {next: None}));
    let third = Rc::new(RefCell::new(Node {next: None}));
    (*first).borrow_mut().next = Some(Rc::clone(&second));
    (*second).borrow_mut().next = Some(Rc::clone(&third));
    (*third).borrow_mut().next = Some(Rc::clone(&first));
}
```
这是一个环形链表的代表，稍微有点绕，原理就是 first -> second -> third, 同时 third -> first, 代码运行后，并没有打印 **Dropping https://mytechshares.com 董泽润的技术笔记**

这里面 `RefCell`, `borrow_mut` 有点绕，刚学 rust 时一直读不懂，下一篇会详细讲解。**简单说就是 `RefCell`, `Cell` 提供了一种机制叫 `内部可变性`，因为 `Rc` 共享变量所有权，所以要求只能读不允许修改。那么 `Rc` 里面的值包一层 `RefCell`, `Cell` 就可以避开编译器检查，变成运行时检查，相当于开了个后门**。Rust 里大量使用这种设计模式

那么如何 fix 这个循环引用呢？答案是 `Weak` 指针，只增加引用逻辑，不共享所有权，即不增加 strong reference count

```rust
use std::rc::Rc;
use std::rc::Weak;
use std::cell::RefCell;

struct Node {
    next: Option<Rc<RefCell<Node>>>,
    head: Option<Weak<RefCell<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("Dropping https://mytechshares.com 董泽润的技术笔记");
    }
}

fn main() {
    let first = Rc::new(RefCell::new(Node {next: None, head: None}));
    let second = Rc::new(RefCell::new(Node {next: None, head: None}));
    let third = Rc::new(RefCell::new(Node {next: None, head: None}));
    (*first).borrow_mut().next = Some(Rc::clone(&second));
    (*second).borrow_mut().next = Some(Rc::clone(&third));
    (*third).borrow_mut().head = Some(Rc::downgrade(&first));
}
```
这是修复后的代码，增加一个 `head` 字段，`Weak` 类型的指针，通过 `Rc::downgrade` 来生成 first 的弱引用

```rust
# cargo run
   Compiling hello_cargo v0.1.0 (/root/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 2.63s
     Running `target/debug/hello_cargo`
Dropping https://mytechshares.com 董泽润的技术笔记
Dropping https://mytechshares.com 董泽润的技术笔记
Dropping https://mytechshares.com 董泽润的技术笔记
```
运行后看到，打印了三次 Dropping 信息，符合预期。另外由于 `Weak` 指针指向的对象可能析构了，所以不能直接解引用，要模式匹配，再 upgrade

### 线程安全 Arc
让我们看一个并发修改变量的例子，来自[the rust book 官网](https://doc.rust-lang.org/book/ch16-03-shared-state.html#atomic-reference-counting-with-arct)
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("董泽润的技术笔记 Got Result: {}", *counter.lock().unwrap());
}
```
spawn 开启 10 个线程，并发对 counter 加一，最后运行后打印
```shell
# cargo run
   Compiling hello_cargo v0.1.0 (/root/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 3.72s
     Running `target/debug/hello_cargo`
董泽润的技术笔记  Got Result: 10
```
刚学的时候确实很绕，连并发修改变量都这么麻烦，各种 `Arc` 套 `Mutex`, 其实这都是为了实现零成本的运行时安全，总要有人做 GC 方面的工作

而且**各种 wrapper 并不妨碍阅读源码，忽略就好，关注代码逻辑，只有写的时候才需要仔细思考**

### 底层实现
```rust
#[stable(feature = "rust1", since = "1.0.0")]
pub struct Rc<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
    phantom: PhantomData<RcBox<T>>,
}

#[repr(C)]
struct RcBox<T: ?Sized> {
    strong: Cell<usize>,
    weak: Cell<usize>,
    value: T,
}
```
`Rc` 是一个结构体，两个字段 `ptr`, `phantom` 成员都是 `RcBox` 类型，注意看里面有 `strong`, `weak` 计数字段和真实数据 `value` 字段，就里再次看到了 `Cell` 来实现的内部可变量

`PhantomData` 是幽灵数据类型，参考 nomicon 文档，大致场景就是：

> 在编写非安全代码时，我们常常遇见这种情况：类型或生命周期逻辑上与一个结构体关联起来了，但是却不属于结构体的任何一个成员。这种情况对于生命周期尤为常见。

> PhantomData 不消耗存储空间，它只是模拟了某种类型的数据，以方便静态分析。这么做比显式地告诉类型系统你需要的变性更不容易出错，而且还能提供 drop 检查需要的信息

```
Zero-sized type used to mark things that "act like" they own a `T`.

Adding a `PhantomData<T>` field to your type tells the compiler that your
type acts as though it stores a value of type `T`, even though it doesn't
really. This information is used when computing certain safety properties.

```

简单说 `PhantomData` 就是零长的占位符，告诉编译器看起来我拥有这个 T, 但不属于我，同时如果析构时，也要调用 drop 释放 T. 因为幽灵字段的存在，`Rc` 是不拥有 `RcBox` 的，`NonNull` 告诉编译器，这个指针 ptr 一定不为空，你可以优化，`Option<Rc<T>>` 跟 `Rc<T>` 占用相同的大小，去掉了是否为空的标记字段

```rust
    pub fn new(value: T) -> Rc<T> {
        // There is an implicit weak pointer owned by all the strong
        // pointers, which ensures that the weak destructor never frees
        // the allocation while the strong destructor is running, even
        // if the weak pointer is stored inside the strong one.
        Self::from_inner(
            Box::leak(box RcBox { strong: Cell::new(1), weak: Cell::new(1), value }).into(),
        )
    }
```
从初始化 `Rc::new` 代码中看到，初始 strong, weak 计数值均为 1

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Clone for Rc<T> {
    #[inline]
    fn clone(&self) -> Rc<T> {
        self.inner().inc_strong();
        Self::from_inner(self.ptr)
    }
}

impl<T: ?Sized> Rc<T> {
    #[inline(always)]
    fn inner(&self) -> &RcBox<T> {
        // This unsafety is ok because while this Rc is alive we're guaranteed
        // that the inner pointer is valid.
        unsafe { self.ptr.as_ref() }
    }
}

fn inc_strong(&self) {
  let strong = self.strong();

  // We want to abort on overflow instead of dropping the value.
  // The reference count will never be zero when this is called;
  // nevertheless, we insert an abort here to hint LLVM at
  // an otherwise missed optimization.
  if strong == 0 || strong == usize::MAX {
      abort();
  }
  self.strong_ref().set(strong + 1);
} 
```
`Rc::clone` 并不会拷贝数据，只增加 strong ref count 强引用计数

```rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<#[may_dangle] T: ?Sized> Drop for Rc<T> {

    fn drop(&mut self) {
        unsafe {
            self.inner().dec_strong();
            if self.inner().strong() == 0 {
                // destroy the contained object
                ptr::drop_in_place(Self::get_mut_unchecked(self));

                // remove the implicit "strong weak" pointer now that we've
                // destroyed the contents.
                self.inner().dec_weak();

                if self.inner().weak() == 0 {
                    Global.deallocate(self.ptr.cast(), Layout::for_value(self.ptr.as_ref()));
                }
            }
        }
    }
}
```
可以看到 `drop` 时先做计数减一操作，如果 strong ref count 到 0，开始析构释放对象。同时 weak 减一，如果为 0, 就要释放 `RcBox`, 也就是 `RcBox` 和 `Value T` 是单独释放的

```rust
pub struct Arc<T: ?Sized> {
    ptr: NonNull<ArcInner<T>>,
    phantom: PhantomData<ArcInner<T>>,
}

struct ArcInner<T: ?Sized> {
    strong: atomic::AtomicUsize,

    // the value usize::MAX acts as a sentinel for temporarily "locking" the
    // ability to upgrade weak pointers or downgrade strong ones; this is used
    // to avoid races in `make_mut` and `get_mut`.
    weak: atomic::AtomicUsize,

    data: T,
}
```
而 `Arc` 里面的计数是用 atomic 来保证原子的，所以是并发字全。还有很多源码实现，理解有点难，多 google 查查，再读读注释，感兴趣自行查看
### 小结
本文先分享这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Rc/Arc 智能指针` 大家有什么看法，欢迎留言一起讨论，如果理解有误，请大家指正 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)

