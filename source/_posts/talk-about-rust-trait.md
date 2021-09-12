---
title: Rust trait 入门
categories: rust
toc: true
---

![](https://gitee.com/dongzerun/images/raw/master/img/rust-trait-cover.jpg)

学 Rust 的一定离不开 `trait`, 告诉编译器某些类型拥有的，且能够被其他类型共享的功能，官方的定义叫做 [Defining Shared Behavior](https://doc.rust-lang.org/book/ch10-02-traits.html) 共享行为，同时还可以对泛型参数进行约束，将其指定为某些特定行为的类型。读过 [你真的了解泛型嘛](https://mp.weixin.qq.com/s/4PlneTYivBoBdZCBLso6jw) 朋友肯定知道，rust 的 `trait` 和 go `interface` 非常像，但是远比后者强大

[Rust Ownership 三原则](https://mp.weixin.qq.com/s/3apdPoiPWmDDU7jMRIJkrA) 开篇讲到对像在离开作用域时，会调用 `drop` 方法析构，这就是一个 drop trait 
### trait 简介
#### 定义
[Summary](https://doc.rust-lang.org/book/ch10-02-traits.html) 官方示例特征
```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```
`pub` 关键字表示能被其它模块调用，如果是私有的可以省略。`Summary` trait 拥有一个函数 `summarize`, 只需要描述函数签名即可，不需要写上函数体
```go
type Endpoint interface {
	// SetClient sets the http.Client to use.
	SetClient(client *http.Client)
}
```
可以看到和 go `interface` 的定义很像，只不过 go 的 `self` 是隐式传递的，而 rust 需要显示传递，并且只能是 `&self` 引用。为什么一定要用引用呢？这是就锈儿们吐糟和难以理解的点：如果传入 `self` 那么函数就会拥有变量所有权，离开函数后就会被析构 drop 掉
#### 实现
官网以 `NewsArticle` 和 `Twitter` 为例子，展示如何实现 `Summary` trait
```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```
`impl` 关键字，后面是要实现的 `trait` 名称，for 后面是结构体名
```rust
fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
}
```
调用的话也比较简单，初始化 `tweet` 对像后，调用相应的方法 `summarize` 即可。这里可以看到和 go `interface` 有些区别，go 不需要显示的指定实现了某些接口，而 rust 要显示声明
#### 做为入参
```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
上面是 go `net/http` 的 `ServeHTTP` 函数，入参 `ResponseWriter` 就是一个接口。其实 rust 也常这么用
```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```
这里传入参数 `item` 类型是 `impl Summary` 的引用，如果不传引用，所有权又被转移到 `notify` 函数里
#### 做为出参
```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```
同样，返回值时，指定类型是 `impl Summary`, 与 go 不一样的是，函数体内不允许返回不同类型，仅管他们都实现了 `Summary`
```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
        ......
        }
    } else {
        Tweet {
        .....
        }
    }
}
```
比如这种 if-else 编译会报错
```shell
   | |_|_________^ expected struct `NewsArticle`, found struct `Tweet`
53 |   |     }
   |   |_____- `if` and `else` have incompatible types
   |
help: you could change the return type to be a boxed trait object
   |
31 | fn returns_summarizable(switch: bool) -> Box<dyn Summary> {
   |                                          ^^^^^^^        ^
help: if you change the return type to expect trait objects, box the returned expressions
```
rust 编译器很智能，提示我们要用 `Box<dyn Summary>`, 那么最终的实现如下
```rust
fn returns_summarizable(switch: bool) -> Box<dyn Summary> {
    if switch {
        let na  = NewsArticle {
          ......
        };
        Box::new(na)
    } else {
        let t = Tweet {
          ......
        };
        Box::new(t)
    }
}
```
### trait 与泛型
```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```
官网给了例子 `largest` 泛型函数，传入类型 T, 约束是类型必须实现 `PartialOrd` 与 `Copy` 特征，用于比较运算和 copy 实现
```rust
fn largest<T>(list: &[T]) -> T
    where T: PartialOrd + Copy
```
为了简洁，可以使用 where 关键词。 那么问题来了什么时候使用泛型，什么时候使用 trait 呢？

```rust
fn largest<T>(left: T, right: T) -> T
```
这是泛型函数，由于单态化，传参 left, right 类型都是 T 然后返回 T 类型，要求类型必须一样。不可能传入 i32 返回 i8. 除非泛型签名是 (left: T, right: U) 
```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```
例子模拟了 GUI 的屏幕实现，只要实现了 `Draw` 特征的组件就可以，可以渲染方形，渲染圆形，也可以是三角形

### 高级特性
#### 默认实现方法
```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```
与 go 不同，rust `trait` 可以拥有默认实现，这样结构体 impl 时无需实现该方法
```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```
同时，`Summary` 的默认实现方法，也可以调用其它 trait 方法。如上例所示，`Tweet` 只需实现 `summarize_author` 即可
#### 有条件的实现方法
参考 [Using Trait Bounds to Conditionally Implement Methods](https://doc.rust-lang.org/book/ch10-02-traits.html#using-trait-bounds-to-conditionally-implement-methods) 章节
```rust
struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}
```
泛型结构体 `Pair` 拥有两个字段，x, y 类型都是 T. 我们可以针对部份类型实现某些方法，这在 rust 里大量应用 (rust 让人难懂的就是柔和了泛型，生命周期与所有权)
```rust
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```
此时只有实现了 `Display` 和 `PartialOrd` 的类型，才能调用 `cmd_display` 方法

#### blanket implementations
我们同样可以为实现了某个 trait 的类型有条件的实现另外一个 trait, rust 标准库里也大量应用这个方法。对于满足 trait 约束的所有类型实现 trait 也被称作 `blanket implementations`, 有的翻译成覆盖实现，有的叫一篮子实现

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```
比如标准库里，给所有实现了 `Display` 的类型，都实现了 `ToString` trait
```rust
let s = 3.to_string();
```
然后就可以直接用 `to_string` 将整型转成字符串
#### 继承
trait 同样是可以继承的
```rust
trait Vehicle {
  fn get_price(&self) -> u64;
}

trait Car: Vehicle {
  fn model(&self) -> String;
}
```
trait `Car` 实现依赖于 `Vehicle`, 也就是说，任何想实现 `Car` 的必须也同时要实现 `Vehicle`. 可以参考 `Fn`, `FnOnce`, `FnMut` [傻傻分不清](https://mp.weixin.qq.com/s/sxfkVBjGSc6J1sVIWVdvTw)
#### Marker trait
Rust 里有大量的 Marker trait, 不包含任何方法，被称为标记特征，用于告诉编译器这些类型实现了某些特征，得到编译期的某些保证。比如 `Sync`, `Send`, `Copy` 特征

* Send 表示数据能安全地被 move 到另一个线程
* Sync 表示数据能在多个线程中被同时安全地访问

还有需要用到的 `Sized`, `?Sized` 等等
#### 泛型 trait
```rust
pub trait From<T> {
  fn from(T) -> Self;
}
```
该例子来自 `From<T>` 与 `Into<T>` 特征，允许从某种类型，转换为类型 T，反之同样
#### Associated Types
参考官网 [Associated Types](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types), 上面提到了泛型特征，但是实际应用中不够方便，关联类型是更好的选择

trait 实现者需要根据自己特定场景来为联联类型指定具体的类型，通过这一技术，我们可以定义出包含某些类型的 trait, 需无须在实现前确定它们的具体类型。来看一个例子

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```
`Item` 是占位符，`next` 函数表明它返回类型是 `Option<Self::Item>`, 实现者只需要为 `Item` 指定具体的类型
```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```
看起来并没有什么特殊之处，如果我们用上面提到的泛型 trait 定义会什么样呢？
```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```
这是一个泛型 trait, 我需要为 Counter 指定 u32 的实现。对于其它的迭代器，需要指定 String 实现，等等。每次使用时，也要标注类型，非常麻烦，如果用关联类型，只需要指定一次
```rust
pub trait Add<Rhs = Self> {
    /// The resulting type after applying the `+` operator.
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}

struct Complex<T> {
    real: T,
    imag: T,
}

impl<T> Add for Complex<T>
    where T: Add<Output=T>   // or, `where T: Add<T, Output=T>`
{
    type Output = Complex<T>;

    fn add(self, rhs: Self) -> Self::Output {
        Complex {
            real: self.real + rhs.real,
            imag: self.imag + rhs.imag,
        }
    }
}
```
这是运符符重载的例子，网上引用的比较多，大家可以仔细看下实现

### 小结
本文先分享这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `trait` 大家有什么看法，欢迎留言一起讨论，如果理解有误，请大家指正 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)