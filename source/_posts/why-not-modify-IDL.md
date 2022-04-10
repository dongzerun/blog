---
title: 聊聊为什么 IDL 只能扩展字段而非修改
categories: go
toc: true
---

![](/images/idl-cover.png)

前几年业界流行使用 [thrift](https://github.com/apache/thrift), 比如滴滴。这几年 [grpc](https://github.com/grpc/grpc-go) 越来越流行，很多开源框架也集成了，我司大部分服务都同时开放 grpc 和 http 接口

相比于传统的 http1 + json 组合，这两种技术都用到了 [IDL](https://en.wikipedia.org/wiki/Interface_description_language), 即 `Interface description language` 接口描术语言，相当于增加了 endpoint schema 约束，不同语言只需要一份相同的 IDL 文件即可生成接口代码。

很多人喜欢问：proto buf 与 json 比起来有哪些优势？比较经典的面试题

IDL 文件管理每个公司不一样，有的保存在单独 gitlab 库，有的是 mono repo 大仓库。当业务变更时，IDL 文件经常需要修改，很多新手总是容易踩坑，本文聊聊 grpc proto 变更时的兼容问题，核心只有一条：**对扩展开放，对修改关闭，永远只增加字段而不修改**

### 测试修改兼容性
本文测试使用 [grpc-go example](https://github.com/grpc/grpc-go/tree/master/examples/helloworld) 官方用例，感兴趣自查
```proto
syntax = "proto3";

option go_package = "google.golang.org/grpc/examples/helloworld/helloworld";
package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
  string additional = 2;
  int32 age = 3;
  int64 id = 4;
}
```
每次修改后使用 `protoc` 重新生成代码
```shell
protoc --go_out=. --go_opt=paths=source_relative     --go-grpc_out=. --go-grpc_opt=paths=source_relative     helloworld/helloworld.proto
```
Server 每次接受请求后，返回 `HelloReply` 结构体
```go
// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{
		Message:    "Hello addidional " + in.GetName(),
		Additional: "this is addidional field",
		Age:        10,
		Id:         12345,
	}, nil
}
```
Client 每次只打印 Server 返回的结果
#### 修改字段编号
将 `HelloReply` 结构体字段 `age` 编号变成 12, 然后 server 使用新生成的 IDL 库，client 使用旧版本

```shell
zerun.dong$ ./greeter_client
......
2021/12/08 22:23:38 Greeting: {
 "message": "Hello addidional world",
 "additional": "this is addidional field",
 "id": 12345
}
```
可以看到 client 没有读到 age 字段，**因为 IDL 是根据序号传输的，client 读不到 seq 3, 所以修改序号不兼容**
#### 修改字段 name
修改 `HelloReploy` 字段 `id`, 变成 `score` 类型和序号不变
```go
// The response message containing the greetings
message HelloReply {
  string message = 1;
  string additional = 2;
  int32 age = 3;
  int64 score = 4;
}
```
重新编译 server, 并用旧版本 client 访问
```shell
zerun.dong$ ./greeter_client
......
2021/12/08 22:29:18 Greeting: {
 "message": "Hello addidional world",
 "additional": "this is addidional field",
 "age": 10,
 "id": 12345
}
```
可以看到，虽然修改了字段名，但是 client 仍然读到了正确的值 12345, **如果字段含义不变，那么只修改名称是兼容的**
#### 修改类型
有些类型是兼容的，有些不可以，而且还要考滤不同的语言。这里测试三种
##### 1.字符串与字节数组
```IDL
// The response message containing the greetings
message HelloReply {
  string message = 1;
  bytes additional = 2;
  int32 age = 3;
  int64 id = 4;
}
```
我们将 `additional` 字段由 string 类型修改为 bytes
```go
// The response message containing the greetings
type HelloReply struct {
	Message    string `protobuf:"bytes,1,opt,name=message,proto3" json:"message,omitempty"`
	Additional []byte `protobuf:"bytes,2,opt,name=additional,proto3" json:"additional,omitempty"`
	Age        int32  `protobuf:"varint,3,opt,name=age,proto3" json:"age,omitempty"`
	Id         int64  `protobuf:"varint,4,opt,name=id,proto3" json:"id,omitempty"`
}
```
可以看到 go 结构体由 string 变成了 []byte, 我们知道这两个其实可以互换
```shell
zerun.dong$ ./greeter_client
......
2021/12/08 22:35:43 Greeting: {
 "message": "Hello addidional world",
 "additional": "this is addidional field",
 "age": 10,
 "id": 12345
}
```
最后结果也证明 client 可以正确的处理数据，即**修改成兼容类型没有任何问题**
##### 2.int32 int64 互转
```IDL
message HelloReply {
  string message = 1;
  string additional = 2;
  int64 age = 3;
  int64 id = 4;
}
```
这里我们将 `age` 由 int32 修改成 int64 字段，位数不一样，如果同样小于 int32 最大值没有问题，此时我们在 server 端将 age 赋于 2147483647 + 1 刚好超过最大值
```shell
zerun.dong$ ./greeter_client
......
2021/12/08 22:43:32 Greeting: {
 "message": "Hello addidional world",
 "additional": "this is addidional field",
 "age": -2147483648,
 "id": 12345
}
```
我们可以看到 `age` 变成了负数，如果业务刚好允许负值，那么此时一定会出逻辑问题，而且难以排查 bug, 这其实是非常典型的向上向下兼容问题
##### 3.非兼容类型互转
```IDL
message HelloReply {
  string message = 1;
  string additional = 2;
  string age = 3;
  int64 id = 4;
}
```
我们将 `age` 由 int32 变成 string 字符串，依旧使用 client 旧版本测试
```shell
zerun.dong$ ./greeter_client
......
2021/12/08 22:55:21 Greeting: {
 "message": "Hello addidional world",
 "additional": "this is addidional field",
 "id": 12345
}
2021/12/08 22:55:21 message:"Hello addidional world" additional:"this is addidional field" id:12345 3:"this is age"
2021/12/08 22:57:56 r.Age is 0
```
可以看到结构体 json 序列化打印时不存在 `Age` 字段，但是 log 打印时发现了不兼容的 `3:"this is age"`, 注意 grpc 会保留不兼容的数据

同时 `r.Age` 默认是 0 值，即**非兼容类型修改是有问题的**
#### 删除字段
```IDL
message HelloReply {
  string message = 1;
  string additional = 2;
  // string age = 3;
  int64 id = 4;
}
```
删除字段 `age` 也就是说序号此时有空洞，运行 client 旧版本协义
```shell
zerun.dong$ ./greeter_client
......
2021/12/08 23:02:12 Greeting: {
 "message": "Hello addidional world",
 "additional": "this is addidional field",
 "id": 12345
}
2021/12/08 23:02:12 message:"Hello addidional world"  additional:"this is addidional field"  id:12345
2021/12/08 23:02:12 0
```
没有问题，打印 `r.Age` 当然是默认值 0, 即删除字段是兼容的

### 为什么 required 在 proto3 中取消了？
```IDL
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}
```
熟悉 `thrift` 或是使用 `proto2` 协义的都习惯使用 `required` `optional` 来定义字段属于，扩展字段一般标记为 `optional`, 必传字段使用 `required` 来约束

[官方解释如下 issues2497](https://github.com/protocolbuffers/protobuf/issues/2497, "why remove required")，简单说就是 `required` 打破了更新 IDL 时的兼容性

1. 永远不能安全地向 proto 定义添加 required 字段，也不能安全地删除现有的 required 字段，因为这两个操作都会破坏兼容性

2. 在一个复杂的系统中，proto 定义在系统的许多不同组件中广泛共享，添加/删除 required 字段可以轻松地降低系统的多个部分

3. 多次看到由此造成的生产问题，并且 Google 内部几乎禁止任何人添加/删除 required 字段

上面是谷歌得出的结论，大家可以借鉴一下，但也不能唯 G 家论

### 小结
IDL 修改还有很多测试用例，感兴趣的可以多玩玩，比如结构体间的转换问题，比如 enum 枚举类型。上文测试的都是 server 端使用新协义，client 使用旧协义，如果反过来呢？想测试 thrift 的可以看看这篇 [thrift missing guide](https://diwakergupta.github.io/thrift-missing-guide/, "thrift missing guide")

本文能过测试 case 想告诉大家，**IDL 只能追加杜绝修改**(产品测试阶段随变改，无所谓)

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `IDL 兼容问题` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)