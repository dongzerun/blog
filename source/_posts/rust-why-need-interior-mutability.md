---
title: Rust 为什么需要内部可变性
categories: rust
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/rust-mutability.jpg)

本文参考 [rust book ch15](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) 并添加了自己的理解，感兴趣的可以先看看官方文档

Rust 有两种方式做到可变性

* 继承可变性：比如一个 struct 声明时指定 let mut, 那么后续可以修改这个结构体的任一字段
* 内部可变性：使用 `Cell` `RefCell` 包装变量或字段，这样即使外部的变量是只读的，也可以修改

看似继承可变性就够了，那么为什么还需要所谓的 `interior mutability` 内部可变性呢？让我们分析两个例子
```rust
struct Cache {
    x: i32,
    y: i32,
    sum: Option<i32>,
}

impl Cache {
    fn sum(&mut self) -> i32 {
        match self.sum {
            None => {self.sum=Some(self.x + self.y); self.sum.unwrap()},
            Some(sum) => sum,
        }
    }
}

fn main() {
    let i = Cache{x:10, y:11, sum: None};
    println!("sum is {}", i.sum());
}
```
结构体 `Cache` 有三个字段，`x`, `y`, `sum`, 其中 `sum` 模拟 lazy init 懒加载的模式，上面代码是不能运行的，道理很简单，当 let 初始化变量 i 时，就是不可变的
```shell
17 |     let i = Cache{x:10, y:11, sum: None};
   |         - help: consider changing this to be mutable: `mut i`
18 |     println!("sum is {}", i.sum());
   |                           ^ cannot borrow as mutable
```
有两种方式修复这个问题，let 声明时指定 `let mut i`, 但具体大的项目时，外层的变量很可能是 immutable 不可变的。这时内部可变性就派上用场了。

#### 修复
```rust
use std::cell::Cell;

struct Cache {
    x: i32,
    y: i32,
    sum: Cell<Option<i32>>,
}

impl Cache {
    fn sum(&self) -> i32 {
        match self.sum.get() {
            None => {self.sum.set(Some(self.x + self.y)); self.sum.get().unwrap()},
            Some(sum) => sum,
        }
    }
}

fn main() {
    let i = Cache{x:10, y:11, sum: Cell::new(None)};
    println!("sum is {}", i.sum());
}
```
这是修复之后的代码，sum 类型是 `Cell<Option<i32>>`, 初学者很容易蒙逼，这到底是个啥啊？有的还嵌套多个 wrapper, 比如 `Rc<Cell<Option<i32>>>` 等等

**其实每一个都是有意义的，比如 `Rc` 代表共享所有权，但是因为 `Rc<T>` 里的 T 要求是只读的，不能修改，所以就要用 `Cell` 封一层，这样就共享所有权，但还是可变的，`Option` 就是常见的要么有值 `Some(T)` 要么空值 `None`**, 还是很好理解的

如果不是写 rust 代码，只想阅读源码了解流程，没必要深究这些 wrapper, 重点关注包裹的真实类型就可以

官网举的例子是 [Mock Objects](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html#a-use-case-for-interior-mutability-mock-objects), 代码比较长，但是原理一样
```rust
    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }
```
最后都是把结构体字段，使用 `RefCell` 包装一下
#### Cell
```rust
use std::cell::Cell;

fn main(){
    let a = Cell::new(1);
    let b = &a;
    a.set(1234);
    println!("b is {}", b.get());
}
```
这段代码非常有代表性，如果变量 `a` 没有用 `Cell` 包裹，那么在 `b` 只读借用存在的时间，是不允许修改 `a` 的，由 rust 编译器在 compile 编译期保证：**给定一个对像，在作用域内(NLL)只允许存在 N 个不可变借用或者一个可变借用**


`Cell` 通过 `get`/`set` 来获取和修改值，这个函数要求 value 必须实现 `Copy` trait, 如果我们换成其它结构体，编译报错
```rust
error[E0599]: the method `get` exists for reference `&Cell<Test>`, but its trait bounds were not satisfied
  --> src/main.rs:11:27
   |
3  | struct Test {
   | ----------- doesn't satisfy `Test: Copy`
...
11 |     println!("b is {}", b.get().a);
   |                           ^^^
   |
   = note: the following trait bounds were not satisfied:
           `Test: Copy`
```
从上面可以看到 `struct Test` 默认没有实现 `Copy`, 所以不允许使用 `get`. 那有没有办法获取底层 struct 呢？可以使用 `get_mut` 返回底层数据的引用，但这就要求整个变量是 `let mut` 的，所以与使用 `Cell` 的初衷不符，所以针对 `Move` 语义的场景，rust 提供了 `RefCell`

#### RefCell
与 `Cell` 不一样，我们使用 `RefCell` 一般通过 `borrow` 获取不可变借用，或是 `borrow_mut` 获取底层数据的可变借用
```rust
use std::cell::{RefCell};

fn main() {
    let cell = RefCell::new(1);

    let mut cell_ref_1 = cell.borrow_mut(); // Mutably borrow the underlying data
    *cell_ref_1 += 1;
    println!("RefCell value: {:?}", cell_ref_1);

    let mut cell_ref_2 = cell.borrow_mut(); // Mutably borrow the data again (cell_ref_1 is still in scope though...)
    *cell_ref_2 += 1;
    println!("RefCell value: {:?}", cell_ref_2);
}
```
代码来自 [badboi.dev](https://badboi.dev/rust/2020/07/17/cell-refcell.html), 编译成功，但是运行失败
```shell
# cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
#
# cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/hello_cargo`
RefCell value: 2
thread 'main' panicked at 'already borrowed: BorrowMutError', src/main.rs:10:31
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
`cell_ref_1` 调用 `borrow_mut` 获取可变借用，还处于作用域时，`cell_ref_2` 也想获取可变借用，此时运行时检查报错，直接 panic

也就是说 **`RefCell` 将借用 borrow rule 由编译期 compile 移到了 runtime 运行时**, 有一定的运行时开销
```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```
这是[官方](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html#having-multiple-owners-of-mutable-data-by-combining-rct-and-refcellt)例子，通过 `Rc`, `RefCell` 结合使用，做到共享所有权，同时又能修改 List 节点值

### 小结
内部可变性提供了极大的灵活性，但是考滤到运行时开销，还是不能滥用，性能问题不大，重点是缺失了编译期的静态检查，会掩盖很多错误

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Cell` `RefCell` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)