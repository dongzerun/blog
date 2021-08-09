---
title: 你真的了解泛型嘛
categories: go
toc: true
---

[泛型 Generic Programming](https://zh.wikipedia.org/wiki/%E6%B3%9B%E5%9E%8B%E7%BC%96%E7%A8%8B) 通常指**允许程序员在强类型程序设计语言中，编写代码时使用一些以后才指定的类型，在实例化时作为参数指明这些类型，即类型参数化**

首先我们不是科班讨论学术，有些概念比较模糊也正常，本文讨论的内容意在给大家提供全局的视野看待`泛型 Generic`, 大致了解主流语言的实现

泛型会提供哪些便利呢？上面的动图非常经典，比如 `min` 函数，如果没有泛型，需要针对 int, float, string 分别写出不同的特定实现，代码非常冗余。本文会讨论下为什么 go 需要泛型以及 [go2 泛型的 proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)
### CPP 模板
```c++
#include <iostream>
template <class T>
T max(T a,T b){
  T ret = a > b? a : b;
  std::cout<< a << " and " << b <<" max is " << ret << std::endl;
  return ret;
}
int main(){
  max(1,2);  // 整数
  max(1.2,2.3); // 浮点数
  return 0;
}
```
上面是 cpp 模板实现的 max 泛型函数，传入 int, float 都可以工作
```shell
$ c++ max.cpp  -o max && ./max
1 and 2 max is 2
1.2 and 2.3 max is 2.3
```
运行结果如上所示，同样代码换成 go 肯定就的错了，本质是模板在编译期单态化，针对每个特定类型生成了特定函数
```shell
$ nm max | grep -i max
0000000100000f90 T __Z3maxIdET_S0_S0_
0000000100000ef0 T __Z3maxIiET_S0_S0_
$ c++filt __Z3maxIdET_S0_S0_
double max<double>(double, double)
$ c++filt __Z3maxIiET_S0_S0_
int max<int>(int, int)
```
通过 `nm` 查看二进制的符号表，生成了两个函数，签名分别是 `double max<double>(double, double)`, `int max<int>(int, int)`

CPP 模板非常强大，但是也有缺点，比如二进制膨胀的厉害，有可能影响到 cpu icache, 业务代码无所谓了。另外 `T` 没有其它语言的 constraint 或 bounded, 比如要求 `T` 必须实现某些方法，然后函数内只能调用这些显示的约束

### C 怎么实现
C 严格意义上没有实现，但在 C11 标准中引入了 `_Generic` 泛型选择器，阉割版的实现
```c
#include <stdio.h>
// int类型加法
int addI(int a, int b)
{
    printf("%d + %d = %d\n",a,b, a + b );
    return (a + b);
}
// double类型加法
double addF(double a, double b)
{
    printf("%f + %f = %f\n",a,b, a + b );
    return (a + b);
}
void unsupport(int a,int b)
{
    printf("unsupport type\n");
}
#define ADD(a,b) _Generic((a), \
    int:addI(a,b),\
    double:addF(a,b), \
    default:unsupport(a,b))
int main(void)
{
    ADD(1 , 2);
    ADD(1.1,2.2);
    return 0;
}
```
上面是实现的 `ADD` 方法，只是调用的时候看起来是泛型了。 这种实现是有缺点的，c 会做很多隐式转换，即使你传入字符串，也会编译成功，并且取得一个值。**并没有在编译期确保类型 a, b 一致**

同时由于 c 不支持函数名重载，还要人工编写特定实例的函数 `addF`, `addI`, 并且二进制的符号表并没有 `ADD`. 类比 cpp 是由编译器替我们完成

在 C11 以前的标准中，很多代码都是用 `void *` 实现，问题在于 void 是没有类型信息的，需要增加特定的函数指针回调，让我们来看 redis 例子
```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;


typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
上面的代码来自 [redis dict](https://github.com/redis/redis/blob/unstable/src/dict.h#L61), 字典是泛型的，value 可以是任意类型，只要实现了 `dictType` 定义的函数指针。内核里面也有大量类似实现，非常通用

### Rust 实现
Rust 泛型也同样编译期单态化，运行时没有开销
```rust
fn printer<T: Display>(t: T) {
    println!("{}", t);
}
```
这里定义函数 `printer`, 类型 `T` 必须实现了 `Display` trait, 这就是所谓的 constraint 或者 bounded
```rust
impl <A, D> MyTrait<A, D> for YourType where
    A: TraitB + TraitC,
    D: TraitE + TraitF {
    
}
```
但假如需要满足多个约束的时候，就要使用 [where](https://rustwiki.org/zh-CN/rust-by-example/generics/where.html) 关键字

**Rust 很多时候难懂，就是因为叠加了泛型，生命周期，所有权，还有很多的 wrapper, 别说写了，读都读不懂**
### Java 版本
Java 在 1.5 版本引入了泛型，它的泛型是用类型擦除实现的。Java 的泛型只是在编译期间用于检查类型的正确，为了保证与旧版本 JVM 的兼容，类型擦除会删除泛型的相关信息，导致其在运行时不可用。关于这块可以参考 [大神R大的回答](https://www.zhihu.com/question/28665443/answer/118148143), 早就有类似 cpp 实现，只是涉及兼容上的取舍，不得不这么做

编译器会插入额外的类型转换指令，与 C 语言和 C++ 在运行前就已经实现或者生成代码相比，Java 类型的装箱和拆箱会降低程序的执行效率。非 Java 党就不写太多了
### Go 为什么需要泛型
官方要在 go1.18 引入泛型，那现在我们是怎么用的呢？很多时候就是 copy 代码，或者用 `interface{}` 代替
```go
func Max(a , b Comparable) Comparable
```
比如常见的 `Max` 函数，传入参数 a, b 都实现了 `Comparable` 接口，然后返回最大值。能工作不？能，但是有问题

1. **丢失了类型信息**，原来 cpp/rust 的实现，在函数中操作的都是原始类型，如果用 interface 还需要断言，这是运行时行为，性能损失很大
2. **表达力不够**，比如 a, b 不能确保是同一个类型，可能一个是 int, 另外一个是 struct. 如果想传入的是类型组合呢？

再看一个 interface{} 不能当成泛型的例子
```go
func sort(arr []interface{})
```
这个排序函数如果传入 sort([]int{1,2,3}) 或是 sort([]float64{1.032, 3.012}) 都会报错，原因是什么呢？？？

在 go 中 `[]interface{}` 类型是 slice 不是 `interface{}`, go 中对比接口相等时，是判断 itab 中 type 和 data 需要都一致。换句话说，不允许类型的协变(没其它语言背景的不必纠结概念，在 Go2 中也不打算支持协变和逆变)

### Go2 泛型
泛型讨论了很久，参见 [generic programming facilities](https://github.com/golang/go/issues/15292), [Why Generics](https://blog.golang.org/why-generics#TOC_1.), [Next Step](https://blog.golang.org/generics-next-step), 以及[官方 proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md), 这是最终版，语法上不会再改变

#### 1.Multiple type parameters
```go
// Print prints the elements of any slice.
// Print has a type parameter T and has a single (non-type)
// parameter s which is a slice of that type parameter.
func Print[T any](s []T) {
	......
}
```
类型参数列表，放到中括号里 `[]`, 其中 `T` 是类型，`any` 关键字是表示可以传入任意类型。居然不是业务通用的尖括号 `<>` 还是感觉丑的... 其中 `any` 其实就是 `interface{}` 空接口，写起来相对方便，而且语意上表达更准确一些

```go
// Print2 has two type parameters and two non-type parameters.
func Print2[T1, T2 any](s1 []T1, s2 []T2) { ... }
```
```go
// Print2Same has one type parameter and two non-type parameters.
func Print2Same[T any](s1 []T, s2 []T) { ... }
```
在 `Print2` 中可以传入相当或不同的类型，比如 `Print2[int, int]` 者 `Print2[int, string]` 都是允许的

但是 `Print2Same` 的参数 s1, s2 类别必然要相同，编译期保证。这也就是上面提到 go1 时 `interface{}` 模拟泛型的问题
#### 2.Constraint
```go
func Stringify[T any](s []T) (ret []string) {
	for _, v := range s {
		ret = append(ret, v.String()) // INVALID
	}
	return ret
}
```
这个例子是不会编译成功的，因为 `T` 是任意类型，并没有实现 `String()` 方法。也就是说，**泛型函数只允许调用 constraint 约束里显示指定的方法** 这样的好处是，对于大型项目的构建，不会因为隐式的改动，而改变兼容性

相比于 rust `where` 语句，go 中增加了类型参数列表和接口的组合，做为约束
```go
package constraints

// Ordered is a type constraint that matches any ordered type.
// An ordered type is one that supports the <, <=, >, and >= operators.
type Ordered interface {
	type int, int8, int16, int32, int64,
		uint, uint8, uint16, uint32, uint64, uintptr,
		float32, float64,
		string
}
```
上面是约束的定义，关键字 `type` 后面跟上**允许的类型列表，不知道为啥就是丑。其它语言用到哪个类型，编译期单态就好了，go 需要人工枚举所有可能...**
```go
// Smallest returns the smallest element in a slice.
// It panics if the slice is empty.
func Smallest[T constraints.Ordered](s []T) T {
	r := s[0] // panics if slice is empty
	for _, v := range s[1:] {
		if v < r {
			r = v
		}
	}
	return r
}
```
#### 3.泛型类型
上面提到的都是 `Generic Func`, 另一个使用场就就是泛型类型 `Generic Type`
```go
// Vector is a name for a slice of any element type.
type Vector[T any] []T
```
上面这个就是标准的泛型容器定义，类似其它语言的 `Vector`
```go
// Push adds a value to the end of a vector.
func (v *Vector[T]) Push(x T) { *v = append(*v, x) }
```
同样，泛型类型还可以定义方法
```go
// List is a linked list of values of type T.
type List[T any] struct {
	next *List[T] // this reference to List[T] is OK
	val  T
}

// This type is INVALID.
type P[T1, T2 any] struct {
	F *P[T2, T1] // INVALID; must be [T1, T2]
}
```
上面是定义的 struct 语法，比较相似。但如果泛型类型，有指向自己的指针，那么注意参数顺序也要一样，`P[T1, T2 any]` 这样写的，那么指针也要是 `*[T1, T2]`. 官方说可能以后会改  
#### 4.Comparable types in constraints
Go 不允许重载操作符，这样好处多多。但同时代表着大于，小于这些只适用于基本类型。但也有例外 `==`, `!=` 这是可以比较 struct, array, interface. 官方在标准库预定义了约束 `comparable` 用于比较约束
```go
// Index returns the index of x in s, or -1 if not found.
func Index[T comparable](s []T, x T) int {
	for i, v := range s {
		// v and x are type T, which has the comparable
		// constraint, so we can use == here.
		if v == x {
			return i
		}
	}
	return -1
}
```
上面的例子很清晰明了，T 实现了比较接口，就可以用 `==`
```go
// ComparableHasher is a type constraint that matches all
// comparable types with a Hash method.
type ComparableHasher interface {
	comparable
	Hash() uintptr
}
```
```go
// ImpossibleConstraint is a type constraint that no type can satisfy,
// because slice types are not comparable.
type ImpossibleConstraint interface {
	comparable
	type []int
}
```
用 interface 实现约束，上面的例子是 `hasher` 接口，下面的接口是永远不能实现的，很好理解，类型 `[]int` 是不能比较的

#### 5.类型推导 type inference
类型推导很重要，可以省略很多无用代码，编译器自动识别类型
```go
func Map[F, T any](s []F, f func(F) T) []T { ... }
```
这是一个 Map 算子函数的签名，f 匿名函数，将 `[]F` 转换成 `[]T` 类型 slice
```go
	var s []int
	f := func(i int) int64 { return int64(i) }
	var r []int64
	// Specify both type arguments explicitly.
	r = Map[int, int64](s, f)
	// Specify just the first type argument, for F,
	// and let T be inferred.
	r = Map[int](s, f)
	// Don't specify any type arguments, and let both be inferred.
	r = Map(s, f)
```
最理想的是只需要调用 `r=Map(s, f)` 就可以，否则还要写 `r = Map[int, int64](s, f)`, 能推导的就不要让人写

除了上面的，再举个例子比如 `T1`, `T2` 是两个类型参数，那么 `[]map[int]bool` 可以和以下类型统一 unified 起来

* []map[int]bool 这个就是类型自身，很好理解
* `T1` 当然可以匹配
* `[]T1` T1 匹配到 `map[int]bool`
* `[]map[T1]T2` 这里 T1 是 int, T2 是 bool

上面只是几种可能，但同时不能匹配到 `int`, `struct{}`, `[]map[T1]string` 这也是显而易见的。虽然 go2 不支持协变，但是基于类型推导还是比 go1 方便的多

#### 6.type lists 组合
```go
// StringableSignedInteger is a type constraint that matches any
// type that is both 1) defined as a signed integer type;
// 2) has a String method.
type StringableSignedInteger interface {
	type int, int8, int16, int32, int64
	String() string
}
```
约束还可以将 `类型列表` 与普通的接口函数组合起来。上面的 `StringableSignedInteger` 要求约束，必须是 type list 里的任一类型，同时实现了 `String() string` 函数

问题是 `int` 这些都没有 string 方法，所以上面约束其实是 impossible 的，只是举例子说明如何使用
#### 7.未实现部份
官方还列举了很多 [omissions](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#omissions) 未实现的点

比如不支持：特化，元编程(cpp 玩烂了)，curry 柯立化等等，感兴趣直接看官网好了
### 泛型代价
引入泛型不是没有代价的，都是取舍，好在 go 语言诞生才十年，没有很重的历史包袱
* **Slower Compiler**: 这是 c++/rust 泛型
* **Slower Performance**: 这是 java/scale 泛型
* **Slower Programmer**: 这是 go1 

但 go 真的需要泛型嘛？虽然 go 定位是系统语言，用的最多还是偏业务层，中间件以下还是 c/c++ 的舞台，哪怕 go 己经有了杀手级应用 docker/k8s

吸引大公司选择 go 的原因，就是简单，快速上手，goroutine 用户态高并发。如果 GA 后，会不会导致代码库里泛型使用泛滥呢？

支持泛型也会让 go 很好的处理数据，要是再搞个分代 GC, 是不是可以造大数据的轮子了^^
### 小结
引用[曹大](https://xargin.com/why-do-we-need-generics/)的一句话：敏捷大师们其实非常双标，他们给出的方法论也不一定靠谱，反正成功了就是大师方法得当，失败了就是我们执行不力没有学到精髓。正着说反着说都有道理。

再看看现在的 Go 社区，buzzwords 也很多，**如果一个特性大师不想做，那就是 less is more. 如果一个特性大师想做，那就是 orthogonal, 非常客观**

**分享知识，长期输出价值，这是我做公众号的目标**。写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `泛型` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)