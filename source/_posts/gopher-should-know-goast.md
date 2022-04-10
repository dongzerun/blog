---
title: 每个 gopher 都需要了解的 Go AST
categories: go
toc: true
---

![](/images/goast-cover.jpg)

最近业务迁移，大约 100+ 个接口需要从旧的服务，迁到公司框架。遇到几个痛点：
1. 结构体 dto 做 diff, 对比结果
2. 自定义的结构体与 protobuf 生成的互相转换，基于 json tag

这类工作要么手写(编译期), 要么 reflect 反射实现(运行时)。其中 #1 考滤到性能问题，手写最优，但是结构体太大，同时 100+ 个接口迁移，工作量可以想象

google 开源的 [go-cmp](https://github.com/google/go-cmp, "go-cmp"), 输出美观，反射性能开销大了点。当前业务大量使用，堆机器吧又不是不能用

#2 目前不好解决，可以简单的 json Marshal 再 Unmarshal, 但有些字段类型不一致，同时如何做 json tag 到 pb tag 转换呢？

我们当前的方案是通过解析 ast, 读源码生成结构体树，然后 BFS 遍历自动生成转换代码

> //go:generate ast-tools --action convert --target-pkg aaa/dto/geresponse --src-pkg bbb/dto --source aaaResponse  --target bbbResponse

结合 go generate 自动生成，这是我们的目标
### Go AST 基础
不搞编译器的大多只需要懂前端，不涉及 IR 与后端，同时 go 官方还提供了大量开箱即用的库 [go/ast](https://pkg.go.dev/go/ast, "go ast")

![](/images/go-ast-types.jpg)

```go
type Node interface {
	Pos() token.Pos // position of first character belonging to the node
	End() token.Pos // position of first character immediately after the node
}
```
所有实现 `Pos` `End` 的都是 Node
* `Comments` 注释， //-style 或是 /*-style 
* `Declarations` 声明，`GenDecl` (generic declaration node) 代表 import, constant, type 或 variable declaration. `BadDecl` 代表有语法错误的 node
* `Statements` 常见的语句表达式，return, case, if 等等
* `File` 代表一个 go 源码文件
* `Package` 代表一组源代码文件
* `Expr` 表达式 ArrayExpr, StructExpr, SliceExpr 等等

我们来看一个例子吧，[goast可视化界面](https://yuroyoro.github.io/goast-viewer/index.html, "goast-viewer 可视化界面") 更直观一些
```go
// Manager ...
type Manager struct {
	Same      string
	All       bool   `json:"all"`
	Version   int    `json:"-"`
	NormalStruct  pkgcmd.RootApp
	PointerStruct *pkgcmd.RootApp
	SlicesField       []int
	MapField           map[string]string
}
```
我们定义结构体 `Manager` 来看一下 goast 输出结果

```go
29  .  1: *ast.GenDecl {
30  .  .  Doc: nil
31  .  .  TokPos: foo:7:1
32  .  .  Tok: type
33  .  .  Lparen: -
34  .  .  Specs: []ast.Spec (len = 1) {
35  .  .  .  0: *ast.TypeSpec {
36  .  .  .  .  Doc: nil
37  .  .  .  .  Name: *ast.Ident {
38  .  .  .  .  .  NamePos: foo:7:6
39  .  .  .  .  .  Name: "Manager"
40  .  .  .  .  .  Obj: *ast.Object {
41  .  .  .  .  .  .  Kind: type
42  .  .  .  .  .  .  Name: "Manager"
43  .  .  .  .  .  .  Decl: *(obj @ 35)
44  .  .  .  .  .  .  Data: nil
45  .  .  .  .  .  .  Type: nil
46  .  .  .  .  .  }
47  .  .  .  .  }
```
`*ast.GenDecl` 通用声明，`*ast.TypeSpec` 代表是个类型的定义，名称是 `Manager`
```go
48    .  Assign: -
49    .  Type: *ast.StructType {
50    .  .  Struct: foo:7:14
51    .  .  Fields: *ast.FieldList {
52    .  .  .  Opening: foo:7:21
53    .  .  .  List: []*ast.Field (len = 7) {
54    .  .  .  .  0: *ast.Field {
55    .  .  .  .  .  Doc: nil
56    .  .  .  .  .  Names: []*ast.Ident (len = 1) {
57    .  .  .  .  .  .  0: *ast.Ident {
58    .  .  .  .  .  .  .  NamePos: foo:8:2
59    .  .  .  .  .  .  .  Name: "Same"
60    .  .  .  .  .  .  .  Obj: *ast.Object {
61    .  .  .  .  .  .  .  .  Kind: var
62    .  .  .  .  .  .  .  .  Name: "Same"
63    .  .  .  .  .  .  .  .  Decl: *(obj @ 54)
64    .  .  .  .  .  .  .  .  Data: nil
65    .  .  .  .  .  .  .  .  Type: nil
66    .  .  .  .  .  .  .  }
67    .  .  .  .  .  .  }
68    .  .  .  .  .  }
69    .  .  .  .  .  Type: *ast.Ident {
70    .  .  .  .  .  .  NamePos: foo:8:12
71    .  .  .  .  .  .  Name: "string"
72    .  .  .  .  .  .  Obj: nil
73    .  .  .  .  .  }
74    .  .  .  .  .  Tag: nil
75    .  .  .  .  .  Comment: nil
76    .  .  .  .  }
77    .  .  .  .  1:
```
`*ast.StructType` 代表类型是结构体，`*ast.Field` 数组保存结构体成员声明，一共 7 个元素，第 0 个字段名称 `Same`, 类型 `string`
```
131  .  3: *ast.Field {
132  .  .  Doc: nil
133  .  .  Names: []*ast.Ident (len = 1) {
134  .  .  .  0: *ast.Ident {
135  .  .  .  .  NamePos: foo:11:2
136  .  .  .  .  Name: "NormalStruct"
137  .  .  .  .  Obj: *ast.Object {
138  .  .  .  .  .  Kind: var
139  .  .  .  .  .  Name: "NormalStruct"
140  .  .  .  .  .  Decl: *(obj @ 131)
141  .  .  .  .  .  Data: nil
142  .  .  .  .  .  Type: nil
143  .  .  .  .  }
144  .  .  .  }
145  .  .  }
146  .  .  Type: *ast.SelectorExpr {
147  .  .  .  X: *ast.Ident {
148  .  .  .  .  NamePos: foo:11:16
149  .  .  .  .  Name: "pkgcmd"
150  .  .  .  .  Obj: nil
151  .  .  .  }
152  .  .  .  Sel: *ast.Ident {
153  .  .  .  .  NamePos: foo:11:23
154  .  .  .  .  Name: "RootApp"
155  .  .  .  .  Obj: nil
156  .  .  .  }
157  .  .  }
158  .  .  Tag: nil
159  .  .  Comment: nil
160  .  }
```
`*ast.SelectorExpr` 代表该字段类型是 A.B，其中 A 代表 package, 具体 B 是什么类型不知道，还需要遍历包 A
```go
221  .  6: *ast.Field {
222  .  .  Doc: nil
223  .  .  Names: []*ast.Ident (len = 1) {
224  .  .  .  0: *ast.Ident {
225  .  .  .  .  NamePos: foo:14:2
226  .  .  .  .  Name: "MapField"
227  .  .  .  .  Obj: *ast.Object {
228  .  .  .  .  .  Kind: var
229  .  .  .  .  .  Name: "MapField"
230  .  .  .  .  .  Decl: *(obj @ 221)
231  .  .  .  .  .  Data: nil
232  .  .  .  .  .  Type: nil
233  .  .  .  .  }
234  .  .  .  }
235  .  .  }
236  .  .  Type: *ast.MapType {
237  .  .  .  Map: foo:14:21
238  .  .  .  Key: *ast.Ident {
239  .  .  .  .  NamePos: foo:14:25
240  .  .  .  .  Name: "string"
241  .  .  .  .  Obj: nil
242  .  .  .  }
243  .  .  .  Value: *ast.Ident {
244  .  .  .  .  NamePos: foo:14:32
245  .  .  .  .  Name: "string"
246  .  .  .  .  Obj: nil
247  .  .  .  }
248  .  .  }
249  .  .  Tag: nil
250  .  .  Comment: nil
251  .  }
252  }
```
`*ast.MapType` 代表类型是字段，`Key`, `Value` 分别定义键值类型

内容有点多，大家感兴趣自行实验

#### 遍历
看懂了 go ast 相关基础，我们就可以遍历获取结构体树形结构，广度 + 深度相结合
```go
func (p *Parser) IterateGenNeighbours(dir string) {
	path, err := filepath.Abs(dir)
	if err != nil {
		return
	}

	p.visitedPkg[dir] = true

	pkgs, err := parser.ParseDir(token.NewFileSet(), path, filter, 0)
	if err != nil {
		return
	}

	todo := map[string]struct{}{}
	for pkgName, pkg := range pkgs {
		nbv := NewNeighbourVisitor(path, p, todo, pkgName)
		for _, astFile := range pkg.Files {
			ast.Walk(nbv, astFile)
		}

		// update import specs per file
		for name := range nbv.locals {
			fmt.Sprintf("IterateGenNeighbours find struct:%s pkg:%s path:%s\n", name, nbv.locals[name].importPkg, nbv.locals[name].importPath)
			nbv.locals[name].importSpecs = nbv.importSpec
		}
	}

	for path := range todo {
		dir := os.Getenv("GOPATH") + "/src/" + strings.Replace(path, "\"", "", -1)
		if _, visited := p.visitedPkg[dir]; visited {
			continue
		}
		p.IterateGenNeighbours(dir)
	}
}
```
这里的工作量比较大，涉及 import 包，调试了很久，有些 linter 只需读单一文件即可，工作量没法比

#### 模板输出
最后一步就是输出结果，这里要 BFS 广度遍历结构体树，然后渲染模板
```go
var convertSlicePointerScalarTemplateString = `
    {% if ArrayLength == "" %}
    dst.{{ TargetFieldName }} = make([]{{ TargetType }}, len(src.{{ SrcFieldName }}))
    {% endif %}
    for i := range src.{{ SrcFieldName }} {
    	if src.{{ SrcFieldName }}[i] == nil {
    		continue
    	}

    	tmp := *src.{{ SrcFieldName }}[i] 
    	dst.{{ TargetFieldName }}[i] = &tmp
    }
```
上面是转换 `[8]*Scalar` 可以是数组或切片，模板使用 [pongo2](github.com/flosch/pongo2, "go pongo2 jinja2") 实现的 jinji2 语法，非常强大
```go
// ConvertDtoInsuranceOptionToCommonInsuranceOptionV2 only convert exported fields
func ConvertDtoInsuranceOptionToCommonInsuranceOptionV2(src *dto.InsuranceOption) *common.InsuranceOptionV2 {
    if src == nil {
        return nil
    }
    dst := &common.InsuranceOptionV2{}
    dst.ID = src.ID
    dst.OptionPremium = src.OptionPremium
    dst.InsuranceSignature = src.InsuranceSignature
    dst.Title = src.Title
    dst.Subtitle = src.Subtitle
    dst.ErrorText = src.ErrorText
    dst.IsIncluded = src.IsIncluded
    starCurrency := ConvertDtoCurrencyDTOToCommonCurrencyDTO(src.Currency)
    if starCurrency != nil {
        dst.Currency = *starCurrency
    }
    return dst
}
```
上面是输出结果的样例，整体来讲比手写靠谱多了，遇到个别 case 还是需要手工 fix
### AST 其它应用场景
#### 1. 规则
工作当中用到编译原理的场景非常多，比如去年高老板分享的[用规则引擎让你一天上线十个需求](https://mp.weixin.qq.com/s/gLbAZShFnkPcI937pk-wpg)

```go
If aa.bb.cc == 1  // 说明是多车型发单
  Unmarshal(bb.cc.ee)
  看type是否为 4 
else  // 单车型发单
 Unmarshal(bb.cc.ff)
  看type是否为 4 
(type = 4 的是拼车)
```

业务需要多种多样，订阅 MQ 根据需求做各种各样的统计，入库，供业务查询。如果业务类型少还好，但是 DIDI 业务复杂，如果每次都人工手写 go 代码效率太低

最后解决思路是 `JPATH + Expression Eval`, 需求只需要写表达式，服务解析表达示即可。Eval 库也是现成的 [govaluate](https://github.com/Knetic/govaluate, "govaluate")

#### 2. 模板
jinja2 就是这类的代表

![](/images/jinja2_template_rendering.jpg)

原理非常简单，感兴趣的可以看官方实现

#### 3. Inject 代码
这里要介绍两个项目 pingcap [failpoint](https://github.com/pingcap/failpoint, "pingcap failpoint") 和 uber-go 的 [gopatch](https://github.com/uber-go/gopatch)

![](/images/failpoint-inject-ast.jpg)

failpoint 实现很简单，代码里写 `Marker` 函数，这些空函数在正常编译时会被编译器优化去掉，所以正常运行时 zero-cost

```go
var outerVar = "declare in outer scope"
failpoint.Inject("failpoint-name-for-demo", func(val failpoint.Value) {
    fmt.Println("unit-test", val, outerVar)
})
```
故障注入时通过 failctl 将 `Marker` 函数转换为故障注入函数，这里就用到了 go-ast 做劫持转换

uber-go 的 gopatch 也非常强大，假如你的代码有很多 `go func` 开启的 goroutine, 你想批量加入 `recover` 逻辑，如果数据特别多人工加很麻烦，这时可以用 gopatcher
```go
var patchTemplateString = `@@
@@
+ import "runtime/debug"
+ import "{{ Logger }}"
+ import "{{ Statsd }}"

go func(...) {
+    defer func(){
+        if err := recover(); err != nil {
+            statsd.Count1("{{ StatsdTag }}", "{{ FileName }}")
+            logging.Error("{{ LoggerTag }}", "{{ FileName }} recover from panic, err=%+v, stack=%v", err, string(debug.Stack()))
+        }
+    }()
   ...
 }()
`
```
编写模板，上面的例子自动在 `go func(...) {` 开头注入 `recover` 语句块，非常方便

这个库能做的事情特别多，感兴趣自行实验

#### 4. linter
大部分 linter 工具都是用 go ast 实现的，比如对于大写的 Public 函数，如果没有注释报错
```go
// BuildArgs write a
func BuildArgs() {
    var a int
    a = a + bbb.c
    return a
}
```
我们看下该代码的 ast 代码
```
29  .  .  1: *ast.FuncDecl {
30  .  .  .  Doc: *ast.CommentGroup {
31  .  .  .  .  List: []*ast.Comment (len = 1) {
32  .  .  .  .  .  0: *ast.Comment {
33  .  .  .  .  .  .  Slash: foo:7:1
34  .  .  .  .  .  .  Text: "// BuildArgs write a"
35  .  .  .  .  .  }
36  .  .  .  .  }
37  .  .  .  }
38  .  .  .  Recv: nil
39  .  .  .  Name: *ast.Ident {
40  .  .  .  .  NamePos: foo:8:6
41  .  .  .  .  Name: "BuildArgs"
42  .  .  .  .  Obj: *ast.Object {
43  .  .  .  .  .  Kind: func
44  .  .  .  .  .  Name: "BuildArgs"
45  .  .  .  .  .  Decl: *(obj @ 29)
46  .  .  .  .  .  Data: nil
47  .  .  .  .  .  Type: nil
48  .  .  .  .  }
49  .  .  .  }
```
linter 只需要检查 `FuncDecl` 的 Name 如果是可导出的，同时 `Doc.CommentGroup` 不存在，或是注释不以函数名开头，报错即可

另外如果大家对代码 cycle 有要求，那么是不是可以 ast 扫一遍来发现呢？如果大家要求函数不能超过 100 行，是不是也可以实现呢？

玩法很多 ^^

### 小结
编译原理虽然难，但是搞业务的只需要前端知识即可，不用研究的太深，有需要的场景，知道 AST 如何解决问题就行

今天的分享就这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连

关于 `Go AST` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)