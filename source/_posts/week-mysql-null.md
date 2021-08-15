---
title: 弱智的 MySQL NULL
categories: mysql
toc: true
---

**MySQL 字段一定要 NOT NULL, 并且设置合理的 default 值！！！**

**MySQL 字段一定要 NOT NULL, 并且设置合理的 default 值！！！**

**MySQL 字段一定要 NOT NULL, 并且设置合理的 default 值！！！**

重要的事情提前说三遍，靠人约束是不行的，SQL 上线平台一定要检查语句是否规范。直接一棒子打死，省得以后祸害人间 ^^

### 背景
这几天同事遇到小问题，明明表结构中有 Unique 复合唯一索引，但是数据居然有重复，百思不得其解。因为涉及 json 字段，所以稍走些弯路，以为是 json 引入的问题

这里强调一下，MySQL json 功能很弱，大家不要用，会有性能问题(去年分享过)。同时，`table schema` 本身是一种强约束，字段 json 大家都往里塞，新功能开发，服务易手几次，json 就成了下水道，没人说得清里面存的都是啥

这点很像开发写的 `protobuf` 或是 `thrift` IDL, 定义一个 Map 字段，以后新加字段直接写进 Map 里 ...

### 复现
```shell
mysql> show create table players\G
*************************** 1. row ***************************
       Table: players
Create Table: CREATE TABLE `players` (
  `id` int(10) unsigned NOT NULL,
  `delete_at` timestamp NULL DEFAULT NULL,
  `rname` varchar(30) DEFAULT NULL,
  `rage` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `delete_at` (`delete_at`,`rname`,`rage`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
```
这是表结构，`delete_at`,`rname`,`rage` 三个字段构成一个复合的唯一索引
```shell
mysql> select * from players;
+----+---------------------+-------+------+
| id | delete_at           | rname | rage |
+----+---------------------+-------+------+
|  4 | NULL                | aaaa  | NULL |
|  2 | NULL                | aaaa  |   19 |
|  3 | NULL                | aaaa  |   19 |
|  1 | 2021-09-09 00:00:00 | aaaa  |   19 |
+----+---------------------+-------+------+
4 rows in set (0.01 sec)
```
上面是测试数据，很容易复现数据冗余问题，id 为 2,3 的两条数据是一样的。为什么？

而且索引顺序也不对，`delete_at` 业务逻辑表示记录被删除的时间点，未被删除的话默认为 `NULL`

```SQL
select * from players where delete_at is not null and rname='xx' and range=xx;
```
这是模拟的线上 SQL, 看出问题了吧，应该唯一性高的放到前面才对
### 分析
不绕弯子，数据冗余原因在于 `NULL` 值。在 2005 年的时候就有人提了 Bug [unique index allows duplicates if at least one of the columns is null](https://bugs.mysql.com/bug.php?id=8173, "unique index allows duplicates if at least one of the columns is null"), 大家可以看看讨论，争议很多

我测试后也得出结论，现在的 MySQL 版本也有这个现象，并且 `NULL` 值的索引顺序无关。但是官方并不认为这是一个 issue

为什么？因为 [SQL92 标准](https://github.com/crate/sql-99/blob/master/docs/chapters/13.rst#unique-predicate, "SQL92 标准") 定义了 `NULL` 不与任何值相等, `NULL` 只是代表 missing values 
>NULLs cannot equal anything else, so can't stop UNIQUE from being TRUE. For example, a series of rows containing {1,NULL,2,NULL,3} is UNIQUE. UNIQUE never returns UNKNOWN.

另外关于唯一索引也有相关描述

>A UNIQUE Constraint makes it impossible to COMMIT any operation that would cause the unique key to contain any non-null duplicate values. (Multiple null values are allowed, since the null value is never equal to anything, even another null value.) A UNIQUE Constraint is violated if its condition is FALSE for any row of the Table it belongs to. 

### 使用问题
抛开上文说的索引问题，MySQL 官方文档也指出了 `NULL` 使用让人困惑的地方 [Working with NULL Values](https://dev.mysql.com/doc/refman/5.6/en/working-with-null.html, "Working with NULL Values") 和 [Problems with NULL Values](https://dev.mysql.com/doc/refman/5.6/en/problems-with-null.html, "Problems with NULL Values")

```shell
mysql> select count(*), count(delete_at) from players;
+----------+------------------+
| count(*) | count(delete_at) |
+----------+------------------+
|        4 |                1 |
+----------+------------------+
1 row in set (0.00 sec)
```
count(\*) 和 count(column) 是不一样的，后者会过滤掉 `NULL` 值
```shell
mysql> SELECT 1 = NULL, 1 <> NULL, 1 < NULL, 1 > NULL;
+----------+-----------+----------+----------+
| 1 = NULL | 1 <> NULL | 1 < NULL | 1 > NULL |
+----------+-----------+----------+----------+
|     NULL |      NULL |     NULL |     NULL |
+----------+-----------+----------+----------+
1 row in set (0.01 sec)

mysql> select NULL = NULL, 1 is NULL, 1 is NOT NULL;
+-------------+-----------+---------------+
| NULL = NULL | 1 is NULL | 1 is NOT NULL |
+-------------+-----------+---------------+
|        NULL |         0 |             1 |
+-------------+-----------+---------------+
1 row in set (0.00 sec)
```
可以看到，只能使用 `is NULL` 或者 `is NOT NULL` 来判断是否等于 `NULL`
### 反思
除了上面使用的困惑，`NULL` 值过多会影响统计信息，可能影响执行计划。MySQL 很不负责的把对 `NULL` 值的统计方式交给了用户 `innodb_stats_method`, 默认值是 `nulls_equal`

>Specifies how InnoDB index statistics collection code should treat NULLs. Possible values are NULLS_EQUAL (default), NULLS_UNEQUAL and NULLS_IGNORED

* 当变量设置为 nulls_equal 时，所有 NULL 值都被视为相同(即，它们都形成一个 value group)
* 当变量设置为 nulls_unequal 时，NULL 值不被视为相同。相反，每个 NULL value 形成一个单独的 value group, 大小为 1
* 当变量设置为 nulls_ignored 时，将忽略 NULL 值

阿里也有过一篇文章，讲 [unique 索引有 NULL 值导致主备延迟](http://mysql.taobao.org/monthly/2018/01/04/, "unique 索引有 NULL 值导致主备延迟") 感兴趣的可以看看

最骚的是有人，给一个字段赋值 `'NULL'`, 注意是 `'NULL'` 不是 `NULL`
### 小结
今天的分享就这些，写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`再看`，`点赞`，`分享` 三连

关于 MySQL NULL 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](https://gitee.com/dongzerun/images/raw/master/img/dongzerun-weixin-code.png)