---
title: 浅析 redis lua 实现
categories: redis
toc: true
---

![](/images/redis-lua-cover.jpg)

关于 redis lua 的使用大家都不陌生，应用场景需要把复杂逻辑的原子性，比如计数器，分布式锁。见过没用 lua 实现的锁，不出 bug 也算是神奇

好奇实现的细节，阅读了几个版本，本文源码展示为 3.2 版本, 7.0 重构比较多，看着干净一些

### 一致性
```
redis> EVAL "return { KEYS[1], KEYS[2], ARGV[1], ARGV[2], ARGV[3] }" 2 key1 key2 arg1 arg2 arg3
1) "key1"
2) "key2"
3) "arg1"
4) "arg2"
5) "arg3"
```
上面是简单的测试用例，其中 2 表示紧随其后的两个参数是 key, 我们通过 `KEYS[i]` 来获取，后面的是参数，通过 `ARGV[i]` 获取。我司历史上遇到过一次 redis 主从数据不一致的情况，原因比较简单：

**Lua 脚本需要 `hgetall` 拿到所有数据，但是依赖顺序，恰好此时底层结构 master 己经变成了 `hashtable`, 但是 slave 还是 `ziplist`, 获取到的第一个数据当成 key 去做其它逻辑，导致主从不一致发生**

引出使用 redis lua 最佳实践之一：**无论单机还是集群模式，对于 key 的操作必须通过参数列表，显示的传进去，而不能依赖脚本或是随机逻辑**

结论其实显而易见，会有数据不一致的风险，同时对于 cluster 模式，要求所有 keys 所在的 slot 必须在同一个 shard 内，这个检测是在 smart client 或者是 cluster proxy 端

导致问题的原因在于，redis 旧版本同步时，本质上还是直接执行的 lua 脚本，这种模式叫做 `verbatim replication`. 如果只同步 lua 脚本修改的内容可以避免这类 issue, 类似于 mysql binlog 的 SQL 模式和 ROW 模式的区别(也不完全一样)

实际上 redis 也是这么做的，3.2 版本引入 `redis.replicate_commands()`, 只同步变更的内容，称为 `effects replication` 模式。5.0 lua 默认为该模式，在 7.0 中移除了旧版本的 `verbatim replication` 的支持

```c
int luaRedisGenericCommand(lua_State *lua, int raise_error) {
......
    /* If we are using single commands replication, we need to wrap what
     * we propagate into a MULTI/EXEC block, so that it will be atomic like
     * a Lua script in the context of AOF and slaves. */
    if (server.lua_replicate_commands &&
        !server.lua_multi_emitted &&
        server.lua_write_dirty &&
        server.lua_repl != PROPAGATE_NONE)
    {
        execCommandPropagateMulti(server.lua_caller);
        server.lua_multi_emitted = 1;
    }

    /* Run the command */
    int call_flags = CMD_CALL_SLOWLOG | CMD_CALL_STATS;
    if (server.lua_replicate_commands) {
        /* Set flags according to redis.set_repl() settings. */
        if (server.lua_repl & PROPAGATE_AOF)
            call_flags |= CMD_CALL_PROPAGATE_AOF;
        if (server.lua_repl & PROPAGATE_REPL)
            call_flags |= CMD_CALL_PROPAGATE_REPL;
    }
    call(c, call_flags);
......
}
```
3.2 版本中，当 lua 虚拟机执行 redis.call 或者 redis.pcall 时调用 `luaRedisGenericCommand`, 如果开启了 `lua_replicate_commands` 选项，那么生成一个 `multi` 事务命令用于复制

同时 `call` 去真正执行命令时，call_flags 打上 `CMD_CALL_PROPAGATE_AOF` 与 `CMD_CALL_PROPAGATE_REPL` 标签，执行命令时生成同步命令

```c
void evalGenericCommand(client *c, int evalsha) {
......
    /* If we are using single commands replication, emit EXEC if there
     * was at least a write. */
    if (server.lua_replicate_commands) {
        preventCommandPropagation(c);
        if (server.lua_multi_emitted) {
            robj *propargv[1];
            propargv[0] = createStringObject("EXEC",4);
            alsoPropagate(server.execCommand,c->db->id,propargv,1,
                PROPAGATE_AOF|PROPAGATE_REPL);
            decrRefCount(propargv[0]);
        }
    }
......
}
```
略去无关代码，`evalGenericCommand` 函数最后判断，如果处于 `effects replication` 模式，那么只通过事务去执行产生的命令，而不是同步 lua 脚本，生成一个 `exec` 命令

另外为了保证 `deterministic` 确定性，redis lua 做了以下事情：

1. redis lua 不允许获取系统时间或者外部状态
2. 修改了伪随机函数 `math.random`, 使用同一个种子，使得每次获取得到随机序列是一样的(除非指定了 math.randomseed)
3. 有些命令返回的结果是没有排序的，比如 `SMEMBERS`, 4.0 版本 redids lua 会额外的做一次排序再返回。但是 5.0 后去掉了这个排序，因为前面提到的 `effects replication` 避免了这个问题，但是使用时不要假设有任何排序，是否排序要看普通命令的文档说明
4. 当用户的脚本调用 `RANDOMKEY`, `SRANDMEMBER`, `TIME` 随机命令后，尝试去修改数据库，会报错。但是只读的 lua 脚本可以调用这些 non-deterinistic 命令

### 缓存
一般我们用 `eval` 命令执行 lua 脚本内容，但是对于高频执行的脚本，每次都要从文本中解析生成 function 开销会很高，所以引入了 `evalsha` 命令
```
> script load "redis.call('incr', KEYS[1])"
"072f8f1f3cac2bbf3017c4af7c73e742aa085ac5"
```
先调用 `script load` 生成对应脚本的 hash 值，每次执行时只需要传入 hash 值即可
```
> EVALSHA da0bf4095ef4b6f337f03ba9dcd326dbc5fc8ace 1 testkey
(nil)
```
对于 failover, 或第一次执行时 redis 不存在该 lua 函数则报错
```
> EVALSHA da0bf4095ef4b6f337f03ba9dcd326dbc5fc8aca 1 testkey
(error) NOSCRIPT No matching script. Please use EVAL.
```
所以，我们在封装 redis client 时要处理异常情况

1. Client 初始化计算 sha 值后，直接 `evalsha` 调用脚本
2. 如果失败，返回错误是 `NOSCRIPT`, 再调用 `SCRIPT LOAD` 创建 lua 函数，Client 再正常调用 `evalsha`

```c
void scriptCommand(client *c) {
......
    } else if (c->argc == 3 && !strcasecmp(c->argv[1]->ptr,"load")) {
        char funcname[43];
        sds sha;

        funcname[0] = 'f';
        funcname[1] = '_';
        sha1hex(funcname+2,c->argv[2]->ptr,sdslen(c->argv[2]->ptr));
        sha = sdsnewlen(funcname+2,40);
        if (dictFind(server.lua_scripts,sha) == NULL) {
            if (luaCreateFunction(c,server.lua,funcname,c->argv[2])
                    == C_ERR) {
                sdsfree(sha);
                return;
            }
        }
        addReplyBulkCBuffer(c,funcname+2,40);
        sdsfree(sha);
        forceCommandPropagation(c,PROPAGATE_REPL|PROPAGATE_AOF);
......
}
```
命令入口函数 `scriptCommand`，`LOAD` 名字很不直观，以为是个只读命令，但实际上做了很多事情：

1. 计算 sha 值
2. `luaCreateFunction` 创建运行时函数
3. `forceCommandPropagation` 设置 flag 参数用于复制到从库或者 AOF

### Lua 源码走读
#### 初始化
初始化只需看 `scriptingInit` 函数，主要功能是加载 lua 库(cjson, table, string, math ...), 移除不支除的函数(loadfile, dofile), 注册我们常用的命令表到 lua table 中(call, pcall, log, math, random ...), 最后创建虚拟的 redisClient 用于执行命令

```c
void scriptingInit(int setup) {
    lua_State *lua = lua_open();

    if (setup) {
        server.lua_client = NULL;
        server.lua_caller = NULL;
        server.lua_timedout = 0;
        server.lua_always_replicate_commands = 0; /* Only DEBUG can change it.*/
        ldbInit();
    }

    luaLoadLibraries(lua);
    luaRemoveUnsupportedFunctions(lua);

    /* Initialize a dictionary we use to map SHAs to scripts.
     * This is useful for replication, as we need to replicate EVALSHA
     * as EVAL, so we need to remember the associated script. */
    server.lua_scripts = dictCreate(&shaScriptObjectDictType,NULL);

    /* Register the redis commands table and fields */
    lua_newtable(lua);

    /* redis.call */
    lua_pushstring(lua,"call");
    lua_pushcfunction(lua,luaRedisCallCommand);
    lua_settable(lua,-3);
  ......
}
```
这里面涉及 c 如何与 lua 语言交互，如何互相调用的问题，不用深究用到了再学即可

`lua_newtable(lua);` 创建 lua table 并入栈，此时位置是  -1

`lua_pushstring(lua,"call");` 入栈字符串 `call`

`lua_pushcfunction(lua,luaRedisCallCommand);` 入栈函数 `luaRedisCallCommand`

`lua_settable(lua,-3);` 生成命令表，此时 table 位置是 -3，然后一次从栈中弹出，即伪代码为 `table["call"] = luaRedisCallCommand`

```
eval "redis.call('incr', KEYS[1])" 1 testkey
```
这也就是为什么我们的 lua 脚本可以执行 redis 命令的原因，函数查表去执行。其它命令也同理

```c
    /* Replace math.random and math.randomseed with our implementations. */
    lua_getglobal(lua,"math");

    lua_pushstring(lua,"random");
    lua_pushcfunction(lua,redis_math_random);
    lua_settable(lua,-3);

    lua_pushstring(lua,"randomseed");
    lua_pushcfunction(lua,redis_math_randomseed);
    lua_settable(lua,-3);

    lua_setglobal(lua,"math");
```
这里也看到同时修改了 random 函数行为

#### 执行命令
`eval` 函数总入口是 `evalCommand`, 这里参考 3.2 源码，非 debug 模式下执行调用 `evalGenericCommand`, 函数比较长，主要分三大块
```c
void evalGenericCommand(client *c, int evalsha) {
    lua_State *lua = server.lua;
    char funcname[43];
    long long numkeys;
    int delhook = 0, err;

    /* When we replicate whole scripts, we want the same PRNG sequence at
     * every call so that our PRNG is not affected by external state. */
    redisSrand48(0);

    /* We set this flag to zero to remember that so far no random command
     * was called. This way we can allow the user to call commands like
     * SRANDMEMBER or RANDOMKEY from Lua scripts as far as no write command
     * is called (otherwise the replication and AOF would end with non
     * deterministic sequences).
     *
     * Thanks to this flag we'll raise an error every time a write command
     * is called after a random command was used. */
    server.lua_random_dirty = 0;
    server.lua_write_dirty = 0;
    server.lua_replicate_commands = server.lua_always_replicate_commands;
    server.lua_multi_emitted = 0;
    server.lua_repl = PROPAGATE_AOF|PROPAGATE_REPL;

    /* Get the number of arguments that are keys */
    if (getLongLongFromObjectOrReply(c,c->argv[2],&numkeys,NULL) != C_OK)
        return;
    if (numkeys > (c->argc - 3)) {
        addReplyError(c,"Number of keys can't be greater than number of args");
        return;
    } else if (numkeys < 0) {
        addReplyError(c,"Number of keys can't be negative");
        return;
    }
```

命令执行前的检查阶段，设置随机种子，设置一些 flag, 并检查 keys 个数是否正确

```c
     /* We obtain the script SHA1, then check if this function is already
     * defined into the Lua state */
    funcname[0] = 'f';
    funcname[1] = '_';
    if (!evalsha) {
        /* Hash the code if this is an EVAL call */
        sha1hex(funcname+2,c->argv[1]->ptr,sdslen(c->argv[1]->ptr));
    } else {
        /* We already have the SHA if it is a EVALSHA */
        int j;
        char *sha = c->argv[1]->ptr;

        /* Convert to lowercase. We don't use tolower since the function
         * managed to always show up in the profiler output consuming
         * a non trivial amount of time. */
        for (j = 0; j < 40; j++)
            funcname[j+2] = (sha[j] >= 'A' && sha[j] <= 'Z') ?
                sha[j]+('a'-'A') : sha[j];
        funcname[42] = '\0';
    }

    /* Push the pcall error handler function on the stack. */
    lua_getglobal(lua, "__redis__err__handler");

    /* Try to lookup the Lua function */
    lua_getglobal(lua, funcname);
    if (lua_isnil(lua,-1)) {
        lua_pop(lua,1); /* remove the nil from the stack */
        /* Function not defined... let's define it if we have the
         * body of the function. If this is an EVALSHA call we can just
         * return an error. */
        if (evalsha) {
            lua_pop(lua,1); /* remove the error handler from the stack. */
            addReply(c, shared.noscripterr);
            return;
        }
        if (luaCreateFunction(c,lua,funcname,c->argv[1]) == C_ERR) {
            lua_pop(lua,1); /* remove the error handler from the stack. */
            /* The error is sent to the client by luaCreateFunction()
             * itself when it returns C_ERR. */
            return;
        }
        /* Now the following is guaranteed to return non nil */
        lua_getglobal(lua, funcname);
        serverAssert(!lua_isnil(lua,-1));
    }
```
lua 中保存脚本 funcname 格式是 `f_{evalsha hash}`, 如果每一次执行，调用 `luaCreateFunction` 让 lua 虚拟机加载 user_script 脚本

```c
   /* Populate the argv and keys table accordingly to the arguments that
     * EVAL received. */
    luaSetGlobalArray(lua,"KEYS",c->argv+3,numkeys);
    luaSetGlobalArray(lua,"ARGV",c->argv+3+numkeys,c->argc-3-numkeys);

    /* Select the right DB in the context of the Lua client */
    selectDb(server.lua_client,c->db->id);

    /* Set a hook in order to be able to stop the script execution if it
     * is running for too much time.
     * We set the hook only if the time limit is enabled as the hook will
     * make the Lua script execution slower.
     *
     * If we are debugging, we set instead a "line" hook so that the
     * debugger is call-back at every line executed by the script. */
    server.lua_caller = c;
    server.lua_time_start = mstime();
    server.lua_kill = 0;
    if (server.lua_time_limit > 0 && server.masterhost == NULL &&
        ldb.active == 0)
    {
        lua_sethook(lua,luaMaskCountHook,LUA_MASKCOUNT,100000);
        delhook = 1;
    } else if (ldb.active) {
        lua_sethook(server.lua,luaLdbLineHook,LUA_MASKLINE|LUA_MASKCOUNT,100000);
        delhook = 1;
    }

    /* At this point whether this script was never seen before or if it was
     * already defined, we can call it. We have zero arguments and expect
     * a single return value. */
    err = lua_pcall(lua,0,1,-2);

    /* Perform some cleanup that we need to do both on error and success. */
    if (delhook) lua_sethook(lua,NULL,0,0); /* Disable hook */
    if (server.lua_timedout) {
        server.lua_timedout = 0;
        /* Restore the readable handler that was unregistered when the
         * script timeout was detected. */
        aeCreateFileEvent(server.el,c->fd,AE_READABLE,
                          readQueryFromClient,c);
    }
    server.lua_caller = NULL;
```
`luaSetGlobalArray` 将 `KEYS`, `ARGS` 以参数形式入栈，设置一堆 debug/slow call 相关的参数，最后 `lua_pcall` 执行用户脚本，lua 虚拟机执行脚本时，如果遇到 `redis.call` 就会回调 redis 函数 `luaRedisCallCommand`, 对应的 `redis.pcall` 执行 `luaRedisPCallCommand` 函数


```c
    if (err) {
        addReplyErrorFormat(c,"Error running script (call to %s): %s\n",
            funcname, lua_tostring(lua,-1));
        lua_pop(lua,2); /* Consume the Lua reply and remove error handler. */
    } else {
        /* On success convert the Lua return value into Redis protocol, and
         * send it to * the client. */
        luaReplyToRedisReply(c,lua); /* Convert and consume the reply. */
        lua_pop(lua,1); /* Remove the error handler. */
    }

    /* If we are using single commands replication, emit EXEC if there
     * was at least a write. */
    if (server.lua_replicate_commands) {
        preventCommandPropagation(c);
        if (server.lua_multi_emitted) {
            robj *propargv[1];
            propargv[0] = createStringObject("EXEC",4);
            alsoPropagate(server.execCommand,c->db->id,propargv,1,
                PROPAGATE_AOF|PROPAGATE_REPL);
            decrRefCount(propargv[0]);
        }
    }

......
    if (evalsha && !server.lua_replicate_commands) {
        if (!replicationScriptCacheExists(c->argv[1]->ptr)) {
            /* This script is not in our script cache, replicate it as
             * EVAL, then add it into the script cache, as from now on
             * slaves and AOF know about it. */
            robj *script = dictFetchValue(server.lua_scripts,c->argv[1]->ptr);

            replicationScriptCacheAdd(c->argv[1]->ptr);
            serverAssertWithInfo(c,NULL,script != NULL);
            rewriteClientCommandArgument(c,0,
                resetRefCount(createStringObject("EVAL",4)));
            rewriteClientCommandArgument(c,1,script);
            forceCommandPropagation(c,PROPAGATE_REPL|PROPAGATE_AOF);
        }
    }
}
```
代码有点长，总体就是执行超时处理，生成 `exec` 用于复制，最后如果 replication 从库没有执行过这个 `evlsha` 脚本，并且当前模式不是 lua_always_replicate_commands 要把脚本真实内容也先同步到 replication

这里还有最重要的是 `luaReplyToRedisReply(c,lua);` 将 lua 返回值，转换成 redis RESP 格式 

再来看一下 `luaRedisGenericCommand` 是如何调用 redis 函数
```c
int luaRedisGenericCommand(lua_State *lua, int raise_error) {
    int j, argc = lua_gettop(lua);
    struct redisCommand *cmd;
    client *c = server.lua_client;
    sds reply;

    /* Cached across calls. */
    static robj **argv = NULL;
    static int argv_size = 0;
    static robj *cached_objects[LUA_CMD_OBJCACHE_SIZE];
    static size_t cached_objects_len[LUA_CMD_OBJCACHE_SIZE];
    static int inuse = 0;   /* Recursive calls detection. */

    /* By using Lua debug hooks it is possible to trigger a recursive call
     * to luaRedisGenericCommand(), which normally should never happen.
     * To make this function reentrant is futile and makes it slower, but
     * we should at least detect such a misuse, and abort. */
    if (inuse) {
        char *recursion_warning =
            "luaRedisGenericCommand() recursive call detected. "
            "Are you doing funny stuff with Lua debug hooks?";
        serverLog(LL_WARNING,"%s",recursion_warning);
        luaPushError(lua,recursion_warning);
        return 1;
    }
    inuse++;

    /* Require at least one argument */
    if (argc == 0) {
        luaPushError(lua,
            "Please specify at least one argument for redis.call()");
        inuse--;
        return raise_error ? luaRaiseError(lua) : 1;
    }

    /* Build the arguments vector */
    if (argv_size < argc) {
        argv = zrealloc(argv,sizeof(robj*)*argc);
        argv_size = argc;
    }

    for (j = 0; j < argc; j++) {
        char *obj_s;
        size_t obj_len;
        char dbuf[64];

        if (lua_type(lua,j+1) == LUA_TNUMBER) {
            /* We can't use lua_tolstring() for number -> string conversion
             * since Lua uses a format specifier that loses precision. */
            lua_Number num = lua_tonumber(lua,j+1);

            obj_len = snprintf(dbuf,sizeof(dbuf),"%.17g",(double)num);
            obj_s = dbuf;
        } else {
            obj_s = (char*)lua_tolstring(lua,j+1,&obj_len);
            if (obj_s == NULL) break; /* Not a string. */
        }

        /* Try to use a cached object. */
        if (j < LUA_CMD_OBJCACHE_SIZE && cached_objects[j] &&
            cached_objects_len[j] >= obj_len)
        {
            sds s = cached_objects[j]->ptr;
            argv[j] = cached_objects[j];
            cached_objects[j] = NULL;
            memcpy(s,obj_s,obj_len+1);
            sdssetlen(s, obj_len);
        } else {
            argv[j] = createStringObject(obj_s, obj_len);
        }
    }

    /* Check if one of the arguments passed by the Lua script
     * is not a string or an integer (lua_isstring() return true for
     * integers as well). */
    if (j != argc) {
        j--;
        while (j >= 0) {
            decrRefCount(argv[j]);
            j--;
        }
        luaPushError(lua,
            "Lua redis() command arguments must be strings or integers");
        inuse--;
        return raise_error ? luaRaiseError(lua) : 1;
    }

    /* Setup our fake client for command execution */
    c->argv = argv;
    c->argc = argc;

    /* Log the command if debugging is active. */
    if (ldb.active && ldb.step) {
        sds cmdlog = sdsnew("<redis>");
        for (j = 0; j < c->argc; j++) {
            if (j == 10) {
                cmdlog = sdscatprintf(cmdlog," ... (%d more)",
                    c->argc-j-1);
            } else {
                cmdlog = sdscatlen(cmdlog," ",1);
                cmdlog = sdscatsds(cmdlog,c->argv[j]->ptr);
            }
        }
        ldbLog(cmdlog);
    }
```
这里的功能，主要是从 lua 虚拟机中获取 eval 脚本的参数，赋值给 redisClient, 为以后执行命令做准备
```c
    /* Command lookup */
    cmd = lookupCommand(argv[0]->ptr);
    if (!cmd || ((cmd->arity > 0 && cmd->arity != argc) ||
                   (argc < -cmd->arity)))
    {
        if (cmd)
            luaPushError(lua,
                "Wrong number of args calling Redis command From Lua script");
        else
            luaPushError(lua,"Unknown Redis command called from Lua script");
        goto cleanup;
    }
    c->cmd = c->lastcmd = cmd;
```
`lookupCommand` 查表，找到要执行的 redis 命令

```c
    /* There are commands that are not allowed inside scripts. */
    if (cmd->flags & CMD_NOSCRIPT) {
        luaPushError(lua, "This Redis command is not allowed from scripts");
        goto cleanup;
    }
```
如果是不允许在 lua 中执行的命令，报错退出
```c
    /* Write commands are forbidden against read-only slaves, or if a
     * command marked as non-deterministic was already called in the context
     * of this script. */
    if (cmd->flags & CMD_WRITE) {
        if (server.lua_random_dirty && !server.lua_replicate_commands) {
            luaPushError(lua,
                "Write commands not allowed after non deterministic commands. Call redis.replicate_commands() at the start of your script in order to switch to single commands replication mode.");
            goto cleanup;
        } else if (server.masterhost && server.repl_slave_ro &&
                   !server.loading &&
                   !(server.lua_caller->flags & CLIENT_MASTER))
        {
            luaPushError(lua, shared.roslaveerr->ptr);
            goto cleanup;
        } else if (server.stop_writes_on_bgsave_err &&
                   server.saveparamslen > 0 &&
                   server.lastbgsave_status == C_ERR)
        {
            luaPushError(lua, shared.bgsaveerr->ptr);
            goto cleanup;
        }
    }

    /* If we reached the memory limit configured via maxmemory, commands that
     * could enlarge the memory usage are not allowed, but only if this is the
     * first write in the context of this script, otherwise we can't stop
     * in the middle. */
    if (server.maxmemory && server.lua_write_dirty == 0 &&
        (cmd->flags & CMD_DENYOOM))
    {
        if (freeMemoryIfNeeded() == C_ERR) {
            luaPushError(lua, shared.oomerr->ptr);
            goto cleanup;
        }
    }

    if (cmd->flags & CMD_RANDOM) server.lua_random_dirty = 1;
    if (cmd->flags & CMD_WRITE) server.lua_write_dirty = 1;
```
设置 cmd->flags
```c
    /* If this is a Redis Cluster node, we need to make sure Lua is not
     * trying to access non-local keys, with the exception of commands
     * received from our master or when loading the AOF back in memory. */
    if (server.cluster_enabled && !server.loading &&
        !(server.lua_caller->flags & CLIENT_MASTER))
    {
        /* Duplicate relevant flags in the lua client. */
        c->flags &= ~(CLIENT_READONLY|CLIENT_ASKING);
        c->flags |= server.lua_caller->flags & (CLIENT_READONLY|CLIENT_ASKING);
        if (getNodeByQuery(c,c->cmd,c->argv,c->argc,NULL,NULL) !=
                           server.cluster->myself)
        {
            luaPushError(lua,
                "Lua script attempted to access a non local key in a "
                "cluster node");
            goto cleanup;
        }
    }
```
如果是 cluster 模式，要保证 lua 的 keys 所在的 slots 必须在本地 shard 
```c
    /* If we are using single commands replication, we need to wrap what
     * we propagate into a MULTI/EXEC block, so that it will be atomic like
     * a Lua script in the context of AOF and slaves. */
    if (server.lua_replicate_commands &&
        !server.lua_multi_emitted &&
        server.lua_write_dirty &&
        server.lua_repl != PROPAGATE_NONE)
    {
        execCommandPropagateMulti(server.lua_caller);
        server.lua_multi_emitted = 1;
    }
```
如果是 `effect replication` 模式，生成 `multi` 事务命令用于复制
```c
    /* Run the command */
    int call_flags = CMD_CALL_SLOWLOG | CMD_CALL_STATS;
    if (server.lua_replicate_commands) {
        /* Set flags according to redis.set_repl() settings. */
        if (server.lua_repl & PROPAGATE_AOF)
            call_flags |= CMD_CALL_PROPAGATE_AOF;
        if (server.lua_repl & PROPAGATE_REPL)
            call_flags |= CMD_CALL_PROPAGATE_REPL;
    }
    call(c,call_flags);
```
这里才去真正的执行命令，`call_flags` 参数用于控制是否复制，是否生成 AOF 等等

```c
    /* Convert the result of the Redis command into a suitable Lua type.
     * The first thing we need is to create a single string from the client
     * output buffers. */
    if (listLength(c->reply) == 0 && c->bufpos < PROTO_REPLY_CHUNK_BYTES) {
        /* This is a fast path for the common case of a reply inside the
         * client static buffer. Don't create an SDS string but just use
         * the client buffer directly. */
        c->buf[c->bufpos] = '\0';
        reply = c->buf;
        c->bufpos = 0;
    } else {
        reply = sdsnewlen(c->buf,c->bufpos);
        c->bufpos = 0;
        while(listLength(c->reply)) {
            robj *o = listNodeValue(listFirst(c->reply));

            reply = sdscatlen(reply,o->ptr,sdslen(o->ptr));
            listDelNode(c->reply,listFirst(c->reply));
        }
    }
    if (raise_error && reply[0] != '-') raise_error = 0;
    redisProtocolToLuaType(lua,reply);
......
}
```
`redisProtocolToLuaType` 把 redis 结果转换成 lua 类型返回给 lua 虚拟机

### 小结
感慨一下，redis 仅有的几个数据结构就能满足 90% 的业务需求，最近几个版本优化非常明显，大家赶紧升级吧，享受新版的福利

从生产环境上看，大版本稳定半年到一年，大胆升级准没错，还在抱残守缺的用 redis 3.X 4.X 的活该遇到各种问题

写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `redis lua` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)
