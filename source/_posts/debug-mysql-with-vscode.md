---
title: 新手如何调试 MySQL
categories: MySQL
toc: true
---

![](/images/mysql-vscode-cover.jpg)

前几天看到姜老师的旧文[用 VSCode 编译和调试 MySQL，每个 DBA 都应 get 的小技能](https://www.modb.pro/db/112992, "用 VSCode 编译和调试 MySQL，每个 DBA 都应 get 的小技能"), 文末留了一个思考题，如何修改源码，自定义版本，使得 `select version()` 输出自定义内容

调试过程参考[macOS VSCode 编译调试 MySQL 5.7](https://shockerli.net/post/mysql-source-macos-vscode-debug-5-7/, "macOS VSCode 编译调试 MySQL 5.7")

内部 `Item` 对象参考[从SQL语句到MySQL内部对象](https://www.orczhou.com/index.php/2012/11/mysql-innodb-source-code-optimization-1/#21_Item, "从SQL语句到MySQL内部对象")

源码面前没有秘密，建义对 DB 感兴趣的尝试 debug 调试。本文环境为 mac + vscode + lldb

### 依赖及插件
vscode 插件：
* C/C++
* C/C++ Clang Command Adapter
* CodeLLDB
* CMake Tools

mysql 源码：
* mysql-boost-5.7.35.tar.gz

补丁：
`MySQL <= 8.0.21` 需要对 cmake/mysql_version.cmake 文件打补丁 (没有严格测试所有版本)
```shell
tar -zxf mysql-boost-5.7.35.tar.gz
cd mysql-5.7.35
mv VERSION MYSQL_VERSION
sed -i '' 's|${CMAKE_SOURCE_DIR}/VERSION|${CMAKE_SOURCE_DIR}/MYSQL_VERSION|g' cmake/mysql_version.cmake
```
创建 `cmake-build-debug` 目录，后续 mysql 编译结果，以及启动后生成的文件都在这里
```shell
mkdir -p cmake-build-debug/{data,etc}
```

### 配置 CMake 与编译
在 mysql 工程目录下面创建 `.vscode/settings.json` 文件
```
{
    "cmake.buildBeforeRun": true,
    "cmake.buildDirectory": "${workspaceFolder}/cmake-build-debug/build",
    "cmake.configureSettings": {
        "WITH_DEBUG": "1",
        "CMAKE_INSTALL_PREFIX": "${workspaceFolder}/cmake-build-debug",
        "MYSQL_DATADIR": "${workspaceFolder}/cmake-build-debug/data",
        "SYSCONFDIR": "${workspaceFolder}/cmake-build-debug/etc",
        "MYSQL_TCP_PORT": "3307",
        "MYSQL_UNIX_ADDR": "${workspaceFolder}/cmake-build-debug/data/mysql-debug.sock",
        "WITH_BOOST": "${workspaceFolder}/boost",
        "DOWNLOAD_BOOST": "0",
        "DOWNLOAD_BOOST_TIMEOUT": "600"
    }
}
```
内容没啥好说的，都是指定目录及 boost 配置，其中 `WITH_DEBUG` 打开 debug 模式，会在 /tmp/debug.trace 生成 debug 信息

![](/images/mysql-cmake-configure.jpg)

`View` -> `Command Palette` -> `CMake: Configure` 执行后生成 cmake 配置

![](/images/cmake-result.jpg)

`View` -> `Command Palette` -> `CMake: Build` 编译生成最终 mysql 相关命令

![](/images/cmake-build.jpg)

发现老版本编译很麻烦，各种报错，mysql 5.7 代码量远超过 5.5, 只能硬着头皮看 5.7

### 初始化数据库
首先初始化 my.cnf 配置，简单的就可以，共它均默认
```shell
cd cmake-build-debug
cat > etc/my.cnf <<EOF
[mysqld]
port=3307
socket=mysql.sock
innodb_file_per_table=1
log_bin = on
server-id = 10086
binlog_format = ROW
EOF
```
初始化数据文件，非安全模式，调试用
```shell
./build/sql/mysqld --defaults-file=etc/my.cnf --initialize-insecure
```
```shell
tree -L 1 data
data
├── auto.cnf
├── ca-key.pem
├── ca.pem
├── client-cert.pem
├── client-key.pem
├── ib_buffer_pool
├── ib_logfile0
├── ib_logfile1
├── ibdata1
├── mysql
├── on.000001
├── on.index
├── performance_schema
├── private_key.pem
├── public_key.pem
├── server-cert.pem
├── server-key.pem
└── sys
```
### 运行 MySQL
由于用 vscode 接管 mysql, 所以需要配置 `.vscode/launch.json`
```shell
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug mysqld",
            "program": "${workspaceFolder}/cmake-build-debug/build/sql/mysqld",
            "args": [
                "--defaults-file=${workspaceFolder}/cmake-build-debug/etc/my.cnf", "--debug"
            ],
            "cwd": "${workspaceFolder}"
        },
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug mysql",
            "program": "${workspaceFolder}/cmake-build-debug/build/client/mysql",
            "args": [
                "-uroot",
                "-P3307",
                "-h127.0.0.1"
            ],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

然后点击 `run and debug mysqld`

![](/images/run-and-debug-mysqld.jpg)

![](/images/mysql-running.jpg)

mysql 启动，看到输出日志无异常，此时可以用 mysql-client 连接
```shell
mysql -uroot -S ./data/mysql.sock
```

### 调试
首先在 sql_parser.cc:5435 处打断点
```cpp
void mysql_parse(THD *thd, Parser_state *parser_state)
```
`mysql_parse` 是 sql 处理的入口，至于 tcp connection 连接先可以忽略

![](/images/sql-parser-breakpoint.jpg)

```
mysql> select version();
```
执行上述 sql 自动跳转到断点处，`Step Into`, `Step Over`, `Step Out` 这些调试熟悉下即可

接下来分别调用主要函数：`mysql_execute_command`, `execute_sqlcom_select`, `handle_query`, `select->join->exec()`, `Query_result_send::send_data`, `Item::send`, `Item_string:val_str`, `Protocol_text::store`, `net_send_ok` 

### 修改源码
启动 mysql 时 `init_common_variables` 会初始化一堆变量，其中会调用 `set_server_version` 生成版本信息，修改这个就可以
```cpp
static void set_server_version(void)
{
  char *end= strxmov(server_version, MYSQL_SERVER_VERSION,
                     MYSQL_SERVER_SUFFIX_STR, NullS);
......
#ifndef NDEBUG
  if (!strstr(MYSQL_SERVER_SUFFIX_STR, "-debug"))
    end= my_stpcpy(end, "-dongzerun");
#endif
  if (opt_general_log || opt_slow_log || opt_bin_log)
    end= my_stpcpy(end, "-log");          // This may slow down system
......
}
```
看好条件编译的是哪块，修改即可，**重新 `CMake: Build` 编译再运行**

```shell
mysql> select version();
+----------------------+
| version()            |
+----------------------+
| 5.7.35-dongzerun-log |
+----------------------+
1 row in set (0.00 sec)
```
### Item Class
这里不做过深分析，简单讲
* `select version()`, `select now()` 这些简单 sql 在 server 层就能得到结果，无需进入引擎层
* `version()`, `now()` 这些函数在 yacc&lex 词法解析时就会解析成对应的 `Item` 类
* 最后 mysql 渲染结果时，就是由 `Item::itemize` 写到 result 中

```
function_call_generic:
          IDENT_sys '(' opt_udf_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_ident_sys(@1, $1, $3);
          }
        | ident '.' ident '(' opt_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_2d(@$, $1, $3, $5);
          }
        ;
```
`sql_yacc.cc` 函数 `PTI_function_call_generic_ident_sys` 解析 sql, 识别出 `version()` 是一个函数调用
```cpp
  virtual bool itemize(Parse_context *pc, Item **res)
  {
    if (super::itemize(pc, res))
      return true;
......

    /*
      Implementation note:
      names are resolved with the following order:
      - MySQL native functions,
      - User Defined Functions,
      - Stored Functions (assuming the current <use> database)

      This will be revised with WL#2128 (SQL PATH)
    */
    Create_func *builder= find_native_function_builder(thd, ident);
    if (builder)
      *res= builder->create_func(thd, ident, opt_udf_expr_list);
......
    return *res == NULL || (*res)->itemize(pc, res);
  }
```
`find_native_function_builder` 查找 hash 表，找到对应 `version` 函数注册的单例工厂函数

```cpp
static Native_func_registry func_array[] =
{
  { { C_STRING_WITH_LEN("ABS") }, BUILDER(Create_func_abs)},
  { { C_STRING_WITH_LEN("ACOS") }, BUILDER(Create_func_acos)},
......
  { { C_STRING_WITH_LEN("VERSION") }, BUILDER(Create_func_version)},
......
  { {0, 0}, NULL}
};

```
mysql 启动时调用 `item_create_init` 将这些函数 builder 注册到 hash 表 `native_functions_hash`

```cpp
Create_func_version Create_func_version::s_singleton;

Item*
Create_func_version::create(THD *thd)
{
  return new (thd->mem_root) Item_func_version(POS());
}
```
```cpp
class Item_func_version : public Item_static_string_func
{
  typedef Item_static_string_func super;
public:
  explicit Item_func_version(const POS &pos)
    : Item_static_string_func(pos, NAME_STRING("version()"),
                              server_version,
                              strlen(server_version),
                              system_charset_info,
                              DERIVATION_SYSCONST)
  {}

  virtual bool itemize(Parse_context *pc, Item **res);
};
```
可以看到 `Item_func_version` 函数创建时传参即为 mysql `server_version` 版本信息

### 小结
MySQL 代码太庞大，5.1 大约 100w 行，5.5 130w 行，5.7 以后 330w 行，只能挑重点读源码。最近很多群里的人在背八股，没必要，有那时间学着调试下源码，读读多好

**分享知识，长期输出价值，这是我做公众号的目标**。同时写文章不容易，如果对大家有所帮助和启发，请帮忙点击`在看`，`点赞`，`分享` 三连

![](/images/dongzerun-weixin-code.png)