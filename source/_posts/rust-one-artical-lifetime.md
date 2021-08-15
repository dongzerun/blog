---
title: 一文了解 rust lifetime
categories: Rust
toc: true
---

本篇分享部分案例来自 [The Rust Book](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html, "the rust book"), 在很多模糊的地方增加自己的理解

上次分享了 [Rust 引用](https://mp.weixin.qq.com/s/F7nkdhUed-M2bhWnN3hiEg), 不熟悉的可以先回顾下前文。**首先什么是 `lifetimes`? 生命周期定义了一个`引用`的有效范围，换句话说 `lifetimes` 是编译器用来比对 owner 和 borrower 存活时间的工具，目的是尽可能的避免悬垂引用(dangling pointer)**

```rust
fn main() {
    {
        let r;
        {
            let x = 5;
            r = &x;
            // ^^ borrowed value does not live long enough
        }
        // - `x` dropped here while still borrowed
        println!("r: {}", r);
        // - borrow later used here
    }
}
```
`let r;` 声名了一个变量，在内层语句块中变成对 x 变量的引用，当内层语句块结束后，变量 x (owner) 脱离作用域释放，此时 r 成了悬垂引用，所以编译器报错

### 借用检查器

![](https://gitee.com/dongzerun/images/raw/master/img/the-borrower-checker.png)

编译器有一个借用检查器 (borrow checker) 用来对比作用域，来决定这个引用是否有效。上图有两个注释 `'a` `'b` 来分别代表 r, x 的作用域，内存语句块的 `'b` 远远小于外层的 `'a`, 编译阶段发现 x 生命周期短于 r, 所以报错阻止编译

![](https://gitee.com/dongzerun/images/raw/master/img/borrow-checker-fix.png)

如果要修复也很简单，把 `println` 放到内层语句块即可。大多数时候，我们不需要显示指定 lifetimes, 编译器很智能，会自动帮我们推断，但也有例外
### 看个例子
```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
来看一个需要指定 lifetimes 的例子，`longest` 返回字符串最长的引用，编译时报错

![](https://gitee.com/dongzerun/images/raw/master/img/longest-error2.jpg)

编译器蒙逼了，他不知道函数返回的引用到底是哪一个，需要指定生命周期。并且很贴心的给了提示 
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str
```
这里引出 lifetimes 的一个规则：**如果函数输入参数有引用(凡是引用必然有 lifetimes), 返回结果如果不是引用，那么可以省略标注生命周期，反之必须标注**

道理很简单，如果返回结果是引用，那么根据 ownership 的三原则，他引用的对象一定不是函数内部创建的，因为函数返回后，该引用的对象会被释放掉，返回的引用就成了悬垂引用

所以，返回引用的生命周期必然和输入参数的一致，这就引出 lifetimes 第二个规则：**如果输入参数只有一个是引用，带有 lifetimes, 且返回值也是引用，那么这两个生命周期必然一致，可以省略标注 lifetimes, 反之必须标注**

这里稍微有些绕，大家需要仔细想想并且多跑跑测试例子，加深理解，我刚开始接触这里也走了很多弯路
### 生命周期语法
```rust
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```
语法没什么特别的，就是泛型语法，通常从 `'a` 开始，`'b`, `'c` 都行，写成别的也可以
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
上面的例子 `<'a>` 是泛型语法，函数签名表示 x, y 生命周期是一样的，那么返回引用自然也是 `'a`
```rust
fn main() {
  let s: &'static str = "I have a static lifetime.";
}
```
但是上面的静态生命周期的要用 `'static` 关键字，表示该引用存活在整个程序运行期间。该字符串会被编译到二进制 data 数据段中
```rust
fn longest_with_an_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
    where T: Display
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
上面是和正常泛型参数结合的例子，`<'a, T>`, 第一个 `'a` 是生命周期，第二个是泛型 `T`, 要求实现 `Display` trait. 注意这里面顺序不能颠倒，如果 `T` 放到前面会报错
```shell
error: lifetime parameters must be declared prior to type parameters
 --> src/main.rs:6:36
  |
6 | fn longest_with_an_announcement<T, 'a>(x: &'a str, y: &'a str, ann: T) -> &'a str
  |                                ----^^- help: reorder the parameters: lifetimes, then types: `<'a, T>`

error: aborting due to previous error
```
### Lifetime Annotations in Function Signatures
```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}

fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
![](https://gitee.com/dongzerun/images/raw/master/img/string2-not-live-enough.png)

这个例子就会报错，虽然我们指定了生命周期，但没什么用。`string2` 在语句块结束后事就被释放了，println resut 时继续使用 string2 的引用就是非法的。把 println 放在语句块内部就可以了
### 多个生命周期参数
```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str
```
如果函数有多个生命周期参数，`'a`, `'b`, 返回引用是 `'a`, 此时编译会报错
```shell
11 | fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
   |                                   -------     -------
   |                                   |
   |                                   this parameter and the return type are declared with different lifetimes...
...
15 |       y
   |       ^ ...but data from `y` is returned here
```
原理很简单，编译器无法确定这两个 lifetimes 的有效长度，需要指定约束
```rust
fn longest<'a, 'b: 'a>(x: &'a str, y: &'b str) -> &'a str {
    if x.len() > y.len() {
      x
    } else {
      y
    }
}
```
最终函数如上所示，其中 `'b: 'a` 表示 `'b` 一定包括 `'a` 生命周期长度
### 结构体内的标注
```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.')
        .next()
        .expect("Could not find a '.'");
    let i = ImportantExcerpt { part: first_sentence };
}
```
在结构体定义时，如果存在引用，也要写上 `'a` 注释，泛型语法这里不再赘述了。`part` 是一个字符串引用，在实例 `i` 创建前就存在了，并且和 `i` 同时离开作用域被释放

如果去掉泛型的 lifetime 注释，就会报错
```shell
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:11
  |
2 |     part: & str,
  |           ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
```
### 结构体方法的标注
```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```
结构体方法的标注语法要在 `impl` 关键字后写上 `<'a>`, 并在结构体名后使用。这个例子中并没有在 `announce_and_return_part` 中标注，这里引出另一条规则 **如果方法有多个输入生命周期参数并且其中一个参数是 &self 或 &mut self，说明是个对象的方法(method), 那么所有输出生命周期参数被赋予 self 的生命周期**

假如把 `announce_and_return_part` 返回值换成 `announcement` 就会报错
```rust
9  |     fn announce_and_return_part(&self, announcement: &str) -> &str {
   |                                                      ----     ----
   |                                                      |
   |this parameter and the return type are declared with different lifetimes...
...
12 |announcement
   |         ^^^^^^^^^^^^ ...but data from `announcement` is returned heres
```
这时需要显示的指定来协助编译器来完成检查，指定 lifetime. 另外涉及子类型，协变，逆变时生命周期会更复杂一些，感兴趣的可以参考 [nomicon](https://doc.rust-lang.org/nomicon/subtyping.html#subtyping-and-variance, "subtyping-and-variance") 官方文档

### 小结
杂七杂八写了一大堆，建义大家还是上手多练，多琢磨。以前刚学 rust 时，有人说编译不过的话，生命周期就加 `'a'`, 数据就多用 `clone`

其实呢，**还是要理解本质，和 Go GC 运行期遍历不同，检查器要在编译期确定资源何时何处释放，就需要收集额外的信息。比如说，结构对象有个引用字段。如果无法确认它的生命周期，那么结构对象释放时，是否要释放该字段？还是说该字段可以提前自动释放，那么是否导致悬垂引用？显然，这违反了安全规则**

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Rust lifetime` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)