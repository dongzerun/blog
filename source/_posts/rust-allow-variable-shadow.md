---
title: Rust 居然允许变量 shadow
categories: rust
toc: true
---

Rust 素来以学习曲线陡峭，语言严谨安全著称。但是最近学习 [The Book](https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html?highlight=shadowing#shadowing, "The Rust Book") 时发现，居然允许 `variable shadowing`, 即我们常说的变量遮蔽 ...

如果普通的 `shadowing` 就算了，同一个词法作用域下允许不同类型的变量，使用同一个变量名 :(

除了震惊我还能说啥？让我们来看看 rust, go 这方面的案例吧

### 普通玩法
```rust
fn main() {
    let x = 5;

    let x = x + 1;

    let x = x * 2;

    println!("The value of x is: {}", x);
}
```
我们知道 `let` 关键词用于声名一个新的变量，第二个 `let` 语句会创建新的变量名 `x`, shadowing (遮蔽)了第一个变量
```shell
$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.90s
     Running `target/debug/hello_cargo`
The value of x is: 12
```
最后 cargo run 得到的结果是 12
### 高级玩法
上面的例子是普通的变量 shadowing, 虽然变量名相同，但至少 value 的类型也是一样的

```rust
fn main() {
    let mail = "zerun.dong@grabtaxi.com";

    println!("The value of mail is: {}", mail);

    let mail = mail.len();

    println!("The value of mail is: {}", mail);
}
```
这个例子，声明变量 mail 是一个邮箱字符串，第二个 `let` 将变量名绑定到了整型值。

**这段代码在家可以自行转换成 go 代码，看看能否运行** ^_^

线上遇见这样的代码要挨锤的 ...
```shell
$ cargo run
   Compiling hello_cargo v0.1.0 (/Users/zerun.dong/code/rusttest/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.47s
     Running `target/debug/hello_cargo`
The value of mail is: zerun.dong@grabtaxi.com
The value of mail is: 23
```
重点是 **rust 居然允许同一个词法作用域下的 variable shadowing**, 运行后输出正常

什么场景下需要这样呢？？？
>> Variable shadowing has a lot of synergy with affine types (move semantics). E.g.
>>
>>let foo = foo.unwrap();
>>
>>where you're rebinding foo to refer to the result of unwrap() at the same time as the old foo becomes inaccessible because of it.

上面是我看到的一种说法，认同比较多。意思是说，variable shadowing 在仿生类型上有很多用处，这个例子是移动语义

比如重新绑定变量后，新的 foo 就是 `unwrap` 后的类型，同时旧的 foo 就不可仿问了

### 反面例子
```go
package main

import "fmt"

import "errors"


func fn1() error {
    return errors.New("this is fn1 error")
}

func fn2() (int, error) {
    return 0, errors.New("this is fn2 error")
}

func wrongCase(input int) (err error) {

    switch input {
    case 0:
        err = fn1()
    case 1:
        res, err := fn2()
        if err != nil {
            fmt.Println(res, err)
        }
    }
    return
}

func main(){
    fmt.Println(wrongCase(1))
}
```
上面是 go variable shadowing 的一个反面例子, 除中 case 语句里 err 就是变量 shadowing 的反面案例

代码运行后输出是什么？
```shell
$ go run wrongccase.go
0 this is fn2 error
<nil>
```
在 main 中看到的 nil, 而不是预期的 error. 这是真实发生过的案例，很恶心的 ...

fix 很简单，那怎么提前发现这个问题呢？除了程序员要多写代码，意识到位以外，最好有 golint 输助，这样在 ci 的时候就可以提前发现

这就印证了那句话，**高质量的软件工程质量，需要可靠的工具和标准的流程来保证**
### 小结
通过对比 go 来学习 rust, 会发现好多有趣的地方，至少没那么劝退了 ...

今天的分享就这些，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连。关于 variable shadowing 大家有什么看法，欢迎留言一起讨论 ^_^

![](/images/dongzerun-weixin-code.png)
