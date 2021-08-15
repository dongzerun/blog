---
title: 再批 MySQL JSON
categories: mysql
toc: true
---

上一篇[弱智的 MySQL NULL](https://mp.weixin.qq.com/s/hmro3mAmEWDgpanH3DsPpQ), 居然有小伙伴在业务中依赖 `NULL` 使联合索引不唯一的特性，比如有的用户就要多条记录，有的仅一条

**我看了差点一口老血喷出来，把业务逻辑耦合在 DB 中这样真的合适嘛？** 要是外包另当别论，正常项目谁接手谁倒霉

### 讨伐 json
今天的分享是再批 `json`, 去年分享过因为 mysql json 导致的故障，今天的 case 其实是去年的姊妹偏，原理一模一样。有两个原因不建义用 json:

1. **Table Schema 就是强一致的，约束开发不要乱搞，json 这种弱约束的就是开后门，时间一长 json 字段就成了下水道**
2. **MySQL JSON 很垃圾，5.7 系列都有性能问题，测试 8.0 好很多。强烈建义大家，使用前压测一下**

上面提到的两点有争议？有争议就对了
### 实现
`JSON` 有两种表示方法：文本可读的在 mysql 中对应 `json_dom.cc`, binary 二进制表示的对应 `json_binary.cc`

![](https://gitee.com/dongzerun/images/raw/master/img/json-objects.png)

```
If the value is a JSON object, its binary representation will have a
header that contains:

- the member count
- the size of the binary value in bytes
- a list of pointers to each key
- a list of pointers to each value

The actual keys and values will come after the header, in the same
order as in the header.

Similarly, if the value is a JSON array, the binary representation
will have a header with

- the element count
- the size of the binary value in bytes
- a list of pointers to each value
```
源码中注释也写的比较清楚，header + element. 实际上 mysql 只是要 server 识别了 json, 各个存储引擎是存储的二进制 blob

[json-function-reference](https://dev.mysql.com/doc/refman/5.7/en/json-function-reference.html, "json-function-reference") 官方有好多在 server 层操作 json 的方法，感兴趣的可以看一下
### 我们的问题
MySQL 读取 json 时是 json_dom 调用 `wrapper_to_string` 方法，序列化成可读格数数据

写入 json 时，是由 json_binary 调用 `serialize_json_value` 方法，序列化成上面图表示的 binary 数据，然后由引擎层存储成 blob 格式

![](https://gitee.com/dongzerun/images/raw/master/img/wrappeertostring2.jpg)

去年故障有服务端的问题，加载单条数据失败就会走动 panic, 坑人不浅。原因是 `wrapper_to_string` 遇到 json array 特别多的情况下反复 mem_realloc 创建内存空间，导致性能下降

![](https://gitee.com/dongzerun/images/raw/master/img/json-binary2.jpg)

其实去年没有 fix 完整，最近发现写入也有类似问题，只不过是 `serialize_json_value` 写入引擎前 mem_realloc 耗时，这时前端页面发现写入超时了，重试继续写入 json 数据

**恰好赶上联合索引中有 `NULL` 字段，由此引出了唯一索引不唯一的现象**。那怎么解决呢？前端按钮 cooldown 治标不治本，sql 执行 12s 前端肯定又点击提交了，治本还得升级 mysql 8.0, 那会不会又引入其它问题呢？

**要不下掉 json 不用？项目初期做了错误的决定，后人很容易买单**
### 8.0 fix
在测试机上发现 8.0 是 ok 的，没有性能问题，查看提交的 commit, 2016 年就有人发现并 fix 了，不知道有没有 back port 到 mysql 5.7 那几个版本
```
commit a2f9ea422e4bdfd65da6dd0c497dc233629ec52e
Author: Knut Anders Hatlen <knut.hatlen@oracle.com>
Date:   Fri Apr 1 12:56:23 2016 +0200

    Bug#23031146: INSERTING 64K SIZE RECORDS TAKE TOO MUCH TIME

    If a JSON value consists of a large sub-document which is wrapped in
    many levels of JSON arrays or objects, serialization of the JSON value
    may take a very long time to complete.

    This is caused by how the serialization switches between the small
    storage format (used by documents that need less than 64KB) and the
    large storage format. When it detects that the large storage format
    has to be used, it redoes the serialization of the current
    sub-document using the large format. But this re-serialization has to
    be redone again when the parent of the sub-document is switched from
    small format to large format. For deeply nested documents, the inner
    parts end up getting re-serializing again and again.

    This patch changes how the switch between the formats is done. Instead
    of starting with re-serializing the inner parts, it now starts with
    the outer parts. If a sub-document exceeds the maximum size for the
    small format, we know that the parent document will exceed it and need
    to be re-serialized too. Re-serializing an inner document is therefore
    a waste of time if we haven't already expanded its parent. By starting
    with expanding the outer parts of the JSON document, we avoid the
    wasted work and speed up the serialization.
```
### 小结
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `MySQL JSON` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)