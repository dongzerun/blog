---
title: 聊聊最近基于 S3 的项目
categories: S3
toc: true
---

![](/images/aws-s3-cover.jpg)

提起对象存储，业务唯一扛把子就是 AWS Simple Storage Service (S3), 国内云厂商不需要做什么，要什么创新，直接抄就完事。协义都是现成的，哪家厂商敢不支持 s3 协义，都会被现实打脸，纯纯的开卷考试

对于公司来讲，无论是否多云架构，还是自建 IDC, 对象存储选型也只能是支持 S3 协义的，比如探探选型的 `Minio`, 还有基于 `SeaweedFS` 改造的，必选项就是协义

业务也有很多基于 S3 的创业项目，比如基于 Fuse 实现的 `juicefs` 分布式文件系统，目前大数据领域用的非常多，听说前两年实现了收支平衡，当然收入来自国外 ^^ 还有 olap 数据库 databend 等等

### 基础设施
S3 己经是 IT 的基础设施，想象不到离开 S3, 自建成本有多离谱。做为程序员自建轮子飞起，技术实现上也没什么秘密，都是公开的，锻炼技术。就看谁能把稳定性做好，把成本降下来

![](/images/s3-product-photo.jpg)

上面截图来自 aws 官网，大数据分析、日志存储、应用数据、视频图片、备份都围绕 S3, 对于应用来讲，屏蔽了底层细节，无容量限制

而且 aws 大部分产品也都构建在 s3 之上，围绕对象存储基础设施，构建庞大的生态。这也是为什么单独做云存储的厂商，无法存储，无法盈利的原因 ... 比如七牛，如果用阿里云的大数据平台，存取七牛对象存储还要跨公网，谁也接受不了这个成本，这家公司半死不活，己经沦落为某云厂商 cdn 的二道贩子

也可以想象一下，如果我司自建 IDC, 现有的 infra 团队人员至少翻倍，这里羡慕国外的 infra 创业，能买 saas 服务，绝不造轮子。从财务到办公软件，从监控系统到数据库，都是买的服务

刚毕业工作时，数据库的备份只能写到磁盘上，然后把磁盘存储到保险柜，没得选，当时国内公有云厂商接受度还不高

### 成本与选型
S3 好用，但代价也是真的贵，需要根据实用使用选择不同的类型：`Standard`, `Intelligent-Tiering` 与 `Glacier`

标准的最贵，对于频繁读取的默认即可。如果数据读很少，可以选择 `Intelligent-Tiering` 智能存储：`Frequent Access Tier`, `Infrequent Access Tier`, `Archive Instant Access Tier`

对于不追求多活的可以选择 one-zone, 去掉 versioning 多版本存储，进一步降低成本

对于大量的历史审记数据，选择 `Glacier` 冰川类型，缺点是转成冰川类型，后续无法读取，需要手动恢复成正常类型。

![](/images/aws-s3-prcing.jpg)

AWS 当然不是大善人，冰川类型存取都要花钱，一次性转换成本非常高。对于大量小对象，无法利用 `Intellint-Tiering` 类型，最小对象大小要求 128KB， 同时每个对象还有额外 40KB 开销，用于维护对象元数据信息。我们服务的 10% 成本来自这块的开销，惊不惊喜，意不意外 ... 

至于技术实现细节，估计也是多副本离线转 EC 纠删码, 但是 aws 拥有庞大的机器规模，海量的过保机器，闲着也是闲着。这里推荐[美团对象存储系统]("美团对象存储系统", "https://www.bilibili.com/video/BV1za4y1v7Tb"), 干活十足，技术点都讲到了

### 小对象合并
这里介绍下我们的场景，订单完成或退货，都要拍照生成 proof 凭据，供后续审计使用。订单完成还要生成发票收据，这些都是图片的形式存储到 S3, 海量的小对象，成本非常高

为了降低成本己经选择了智能存储与冰川类型，但是这还远远不够。这类场景有个前提：

* 历史图片访问的概率非常低，一周前的图片基本不会读取
* 单独使用图片没用，只有绑定到订单才有意义

经过调研我们可以把图片压缩，并且合并成大文件按历史时间存储，好处是可以充分利用智能类型，使用便宜的 tier, 同时图片经过压缩后大小只有原来的一半

原有逻辑，后端服务生成临时 presigned s3 public URL, 客户端公网访问即可，那么如何读取历史订单数据呢？这里用到了 s3 的技术 [object accesspoint]("https://aws.amazon.com/blogs/aws/introducing-amazon-s3-object-lambda-use-your-code-to-process-data-as-it-is-being-retrieved-from-s3/", "AWS object accesspoint") 与 [select api]("https://docs.aws.amazon.com/AmazonS3/latest/API/API_SelectObjectContent.html", "S3 Select API")

![](/images/object-accesspoint.jpg)

简单来讲，**历史数据合并成 `parquet` 格式大文件，上传到 S3, 用户访问 encoded s3 url 时，access point 调用 lambda 解析 url, 定位到 parquet 文件位置，然后使用 S3 Select API 查询图片数据**

```
message Object {
	required int64 ID;
	required int64 DriverID;
	required int64 CityID;
	required int64 Index;
	required binary Payload;
}
```
可以把 `parquet` 想象成数据库的表，提前定义好 schema, 使用 SQL 来查询数据。`Object` 由 thrift 定义 IDL, `Payload` 是我们的历史图片 binary 数据

```
select Payload from S3Object where ID=XXX limit 1
```
这是 select api 定义的查询 SQL, 至于 AWS 底层如何存储 `parquet` 文件仍然是黑盒，`limit 1` 算子好像不能 condition push down 到底层存储，无法提前返回，也就是说加不加 `limit 1` 效果可能一样

```go

func (impl *implDecoder) Decode(bucket, key string, filters map[string]string) ([]byte, error) {

	sql := fmt.Sprintf("select Payload from S3Object where  %s limit 1", strings.Join(clauses, " and "))
	logger.Info(logTag, "Decode bucket %s key %s filters %v sql: %s", bucket, key, filters, sql)

	// small objects default 50KB
	writer := bytes.NewBuffer(make([]byte, 0, 50*1024))

	if err := selectFromObjects(context.Background(), impl.selector, writer, bucket, key, sql); err != nil {
		return nil, err
	}
	return writer.Bytes(), nil
}
```
上面是 lambda 回调的伪代码示例，像不像读取数据库?

理想情况下，aws 应该将 `parquet` 按照 Row Group 分开存储数据，并且用 parquet index 加速查询，或者是自建 `Object` 索引，类似于 mysql table. 或者添加 bloomfilter, 总之就是减少数据扫描，一个是减小 access point 查询时间，一个是减少成本，aws 奸商是按照扫描数据大小收费的！！！

目前 aws 应该没实现算子下推，了解这块的朋友可以说下 ^^ 再次感慨 aws 生态的强大，收费也是真贵

### 小结
推荐大家读下 minio 或者 seaweedfs 源码，本篇分享的优化 idea 启发于 minio 源码。多读读业界实现，没坏处，说不定哪个带来了灵感

**分享知识，长期输出价值，这是我做公众号的目标**。同时写文章不容易，如果对大家有所帮助和启发，请帮忙点击`在看`，`点赞`，`分享` 三连

![](/images/dongzerun-weixin-code.png)