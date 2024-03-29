---
title: Redis lua 脚本简述
date: 2019-05-27 15:15:49
tags: [redis,lua,script]
---

## 前言
从2.6.0版开始，Redis增加了对Lua运行环境的支持。在了解Redis lua 脚本使用前，最好能够了解 lua 的语言基础。

本篇包含如下 lua 脚本内容：
- Redis加载（初始化lua运行环境）
- Lua与Redis数据类型的转换
- 脚本命令执行分析
- 脚本执行过程分析

## Redis Lua运行环境
Lua 具有原生的运行环境，提供了基本函数库，table函数库，OS函数库等。
为了保障 Lua 脚本的安全性运行问题并提供对Redis的操作，在初始化Redis服务器的同时Lua环境也一并进行了系列适用于Redis的修改。
包括添加函数库、更换随机函数、保护全局变量等。

### 创建 Lua 基本运行环境
在初始化的第一步，服务器首先会调用Lua的C API 函数 lua_open，创建一个新的 Lua 基本运行环境。

### 载入函数库
- 基本库：包含 Lua 的核心函数，如 assert、error、pcall、pairs。
为了防止用户从外部文件引入不安全代码，将库中的 loadfile 函数剔除；
- table 库：提供了处理 table 类型的通用函数，如 table.concat、table.remove、table.sort；
- string 库：提供了处理 字符串 类型的通用函数，如 string.len、string.reverse、string.format；
- math 库：提供标准 C 语言数学库接口，如 math.abs、math.max、math.min、math.sqrt、math.log；
- debug 库：提供了对程序进行调试所需函数，如 debug.sethook、debug.geghook；
- cjson 库：用于处理 utf-8 编码的 JSON 格式，如 cjson.encode 将一个Lua值序列化为 JSON 格式字符串、cjson.decode 将 JSON 格式字符串转换为 Lua 值；
- struct 库：用于处理 Lua 值和 C 结构（struct）之前进行转换，如 struct.pack 将多个 Lua 值打包成一个类结构（struct-like）字符串、struct.unpack 将一个类结构字符串解包出多个 Lua 值；
- cmsgpack 库：用于处理 MessagePack 格式的数据，如 cmsgpack.pack 将 Lua 值转换为 MessagePack 数据、cmsgpack.unpack 将 MessagePack 数据转换为 Lua 值。    

### 创建全局表 redis
全局表 redis 中包含了各种对Redis进行操作的函数，包括：
- 用于执行 Redis 命令的 redis.call 和 redis.pcall 函数
- 用于发送日志的 redis.log 函数，以及相应的日志级别：
    - redis.LOG_DEBUG
    - redis.LOG_VERBOSE
    - redis.LOG_NOTICE
    - redis.LOG_WARNING
- 用于计算 SHA1 校验和的 redis.sha1hex 函数
- 用于返回错误信息的 redis.error_reply 函数和 redis.status_reply 函数

### 替换 Lua 原有随机函数
为了保证相同的脚本可以在不同的机器上产生相同的结果，Redis 要求所有传入服务器的 Lua 脚本，以及 Lua 环境中的所有函数，都必须是无副作用（side effect）的纯函数（pure function）。
Lua 原有随机函数是基于 OS，其 seed 往往是基于时钟 ，不符合 Redis 对 Lua 环境的无副作用要求。
Redis 使用自制的函数替换了 math 库中原有的 math.random 函数和 math.randomseed 函数。替换后的函数具有如下特征：
- 对于相同的 seed 来说， math.random 总产生相同的随机数序列
- 除非在脚本中使用 math.randomseed 显式地修改 seed ，否则每次运行脚本时，Lua 环境都使用固定的 math.randomseed(0) 语句来初始化 seed

### 创建排序辅助函数
Redis 要求所有传入服务器的 Lua 脚本无副作用，就需要处理 Lua 脚本中可能导致数据不一致的情况。
除了原有随机函数会导致数据不一致外，还存在一些带有不确定性质的命令：
- SINTER
- SUNION
- SDIFF
- SMEMBERS
- HKEYS
- HVALS
- KEYS

以 SMEMBERS 对集合的操作为例：
{% codeblock %}
127.0.0.1:6379> SADD fruit apple banana cherry
(integer) 3
127.0.0.1:6379> SMEMBERS fruit
1) "cherry"
2) "banana"
3) "apple"
127.0.0.1:6379> SADD another-fruit cherry banana apple
(integer) 3
127.0.0.1:6379> SMEMBERS another-fruit
1) "apple"
2) "banana"
3) "cherry"
127.0.0.1:6379> 
{% endcodeblock %}

例子中 fruit 集合和 another-fruit 集合包含的元素完全相同（集合 list 是无序的）。
只因为集合添加元素的顺序不同，SMEMBERS 命令的输出就产生了不同的结果，是不满足 Lua 脚本无副作用要求。

为了消灭这些命令带来的不确定性，Redis 服务器为 Lua 环境创建了一个排序辅助函数  __redis__compare_helper，
当 Lua 脚本执行完一个带有不确定性的命令之后，程序会使用 __redis__compare_helper 作为对比函数，自动调用 table.sort 函数对命令的返回值做一次排序，以此来保证相同的数据集总是产生相同的输出。
使用 lua 脚本形式执行示例：
{% codeblock %}
127.0.0.1:6379> eval "return redis.call('SMEMBERS', KEYS[1])" 1 fruit
1) "apple"
2) "banana"
3) "cherry"
127.0.0.1:6379> eval "return redis.call('SMEMBERS', KEYS[1])" 1 another-fruit
1) "apple"
2) "banana"
3) "cherry"
127.0.0.1:6379> 
{% endcodeblock %}

### 创建错误报告辅助函数
服务器为 Lua 环境创建了一个 _redis_err_handler 的错误处理函数,
当脚本运行出现错误时，_redis_err_handler 就会打印出错误代码来源与发生错误行数。
{% codeblock %}
127.0.0.1:6379> eval "local a = redis.call('get', KEYS[1]), return a" 1 haha
(error) ERR Error compiling script (new function): user_script:1: unexpected symbol near 'return' 
127.0.0.1:6379> eval "local a = redis.pcall('get', KEYS[1]), return a" 1 haha
(error) ERR Error compiling script (new function): user_script:1: unexpected symbol near 'return' 
127.0.0.1:6379>
{% endcodeblock %}

### 保护 Lua 全局环境
因为 Lua 变量定义默认为全局变量，为了避免脚本中创建的变量对 Lua全局环境造成影响，Redis 服务器禁用了脚本中全局变量的创建。
1. 当脚本试图创建一个全局变量时，服务将会报告一个错误
{% codeblock %}
127.0.0.1:6379> eval "a = 'my a'" 0
(error) ERR Error running script (call to f_842595f923de966a2f0b2cd2b8a01ae1fb074c53): @enable_strict_lua:8: user_script:1: Script attempted to create global variable 'a' 
127.0.0.1:6379> 
{% endcodeblock %}

2. 当脚本视图获取一个不存在的全局变量也会引发错误
{% codeblock %}
127.0.0.1:6379> eval "return histo" 0
(error) ERR Error running script (call to f_e3299dfc93671ffbb8061eb25dc195c8547b0f7f): @enable_strict_lua:15: user_script:1: Script attempted to access nonexistent global variable 'histo' 
127.0.0.1:6379> 
{% endcodeblock %}

3. 但是 Redis 并不禁止修改已经存在的全局变量，例如修改 全局table redis
{% codeblock %}
127.0.0.1:6379> eval "redis = 110 return redis" 0
(integer) 110
127.0.0.1:6379> keys *
1) "sd"
2) "ft"
3) "aft"
127.0.0.1:6379> eval "return redis.call('SMEMBERS' KEYS[1])" 1 sd
(error) ERR Error compiling script (new function): user_script:1: ')' expected near 'KEYS' 
127.0.0.1:6379> 
{% endcodeblock %}

## Lua与Redis数据类型的转换
Redis 与 Lua 各自具有"数据类型"定义，以下转换规则确保了数据转换的一对一关系。
⚠️这里的 Redis 数据类型实质上是只 Redis 服务对请求的 reply 数据。
而 Redis 是采用 C/S 架构，客户端请求，服务端响应。其中的数据交互可以参考[通信协议](http://redisdoc.com/topic/protocol.html)了解。

### Redis数据转换为 Lua 数据

| Redis Reply | Lua Type | 补充说明 |
| :--------: |:--------:|:--------|
| integer | number | - |
| bulk | string | - |
| multi bulk | table | - |
| status | table | 包含单个 'ok' 键对应值为其 status 的 table 类型 |
| error | table | 包含单个 'err' 键对应值为其 error 信息的 table 类型 |
| Nil bulk / Nil multi bulk | boolean | 值为 false 的 boolean 类型 |

### Lua 数据转换 Redis 数据

| Lua Type | Redis Reply | 补充说明 |
| :--------: |:--------:|:--------|
| number | integer | Lua 的小数 (number) 会被转换为 Redis 整型 |
| string | bulk | - |
| table(array) | multi bulk | 转换过程中会以 Lua array 中的第一个 nil 作为结束标志|
| table with a single ok field | status | - |
| table with a single err field | error | - |
| boolean(false) | Nil bulk | - |

### 补充转换说明
- Lua 的 boolean 类型 true 将会转换为值为 1 的 Redis integer reply
- Lua 的 number 类型可表示整数与小数，在转换为 Redis integer reply 时会忽略小数部分，这点需要特别注意。基于此在脚本中想要返回小数应该将其转换为string
- Lua 中的数组（table）存在一个定义——以第一个 nil 元素为结束标志。这里存在的缺陷是无法拥有一个包含 nil 元素的数组

## 脚本命令执行分析
Redis 服务器提供两种执行 Lua 脚本的命令：EVAL 与 EVALSHA 。主要功能是调用从 Redis 2.6.0 版本内置的 Lua 解释器对脚本进行评估分析。
以下将分别介绍 EVAL 与 EVALSHA 的使用。

### EVAL
{% codeblock [EVAL 基本语法] %}
127.0.0.1:6379> EVAL script numkeys key [key ...] arg [arg ...]
{% endcodeblock %}
- 第一个参数 script 是 Lua 5.1脚本（一个将要在 Redis 上下文运行的程序）
- 第二个参数是脚本后面的 Redis 键名参数数量。
- 第三个参数开始直至达到键名参数定义数量，都为键名，可以在脚本 script 中使用全局变量 KEYS 获取（KEYS[1],KEYS[2]...的形式）
- 剩下的就是非键名参数，可以在脚本 script 中使用全局变量 ARGV 获取（ARGV[1]，ARGV[2]...的形式）

示例如下：
{% codeblock [EVAL 使用示例] %}
127.0.0.1:6379> EVAL "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 value1 value2
1) "key1"
2) "key2"
3) "value1"
4) "value2"
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 mykey myvalue
OK
127.0.0.1:6379> keys *
1) "mykey"
127.0.0.1:6379> EVAL "return redis.call('get', KEYS[1])" 1 mykey
"myvalue"
127.0.0.1:6379> 
{% endcodeblock %}

### EVALSHA
{% codeblock [EVALSHA 基本语法] %}
127.0.0.1:6379> EVALSHA sha1 numkeys key [key ...] arg [arg ...]
{% endcodeblock %}
- 第一个参数 sha1 为 Lua 脚本的 SHA1 校验和，服务器会执行 'f_' + sha1 名称的 function
- 第二个参数是脚本后面的 Redis 键名参数数量。
- 第三个参数开始直至达到键名参数定义数量，都为键名
- 剩下的就是非键名参数

示例如下：
{% codeblock %}
127.0.0.1:6379> SCRIPT LOAD "return {KEYS[1], KEYS[2], ARGV[1], ARGV[2]}"
"c0d2d6f81be75d67523d7c8ac69a932fbe1aa4e2"
127.0.0.1:6379> EVALSHA c0d2d6f81be75d67523d7c8ac69a932fbe1aa4e2 2 k1 k2 v1 v2
1) "k1"
2) "k2"
3) "v1"
4) "v2"
127.0.0.1:6379> 
{% endcodeblock %}

## 脚本执行过程分析
EVALSHA 命令是基于 EVAL 命令构建，关于脚本执行过程分析主要对 EVAL 命令执行过程进行分析。

EVAL 命令执行会分为两步：
1. 为输入脚本定义一个 Lua 函数（function）
2. 执行这个 Lua 函数

### 定义 Lua 函数
所有被 Redis 执行的 Lua 脚本，在 Lua 环境中都会有一个和该脚本对应的无参数函数（目的是：以函数为单位的形式保存 Lua 脚本）。
当调用 EVAL 命令执行脚本时，程序第一步要完成的工作就是为传入的脚本创建一个相应的 Lua 函数（保存在 lua_scripts 字典）。
例如脚本 "return {KEY[1],KEY[2],ARGV[1],ARGV[2]}" ，其生成的 SHA1 校验和为 d8f14ae7100459bda992510e1304e4217cb42234。那么就会创建一个如下的对应函数：
{% codeblock %}
function f_d8f14ae7100459bda992510e1304e4217cb42234()
    return {KEY[1],KEY[2],ARGV[1],ARGV[2]}
end
{% endcodeblock %}
可以看出，函数名以 f_ 为前缀，后根脚本的 SHA1 校验和拼接而成，而函数体则是用户输入的脚本。
如果定义的脚本在编译过程中出错（语法错误），程序将直接返回脚本错误，并不再继续执行后续步骤

### 执行 Lua 函数
在定义好 Lua 函数后，程序就可以通过运行这个函数来达到运行输入脚本的目的。

不过，在此之前，为了确保脚本的正确和安全执行，需要执行一些设置钩子、传入参数之类的操作，整个执行函数的过程如下：
1. 将 EVAL 命令中输入的 KEYS 参数和 ARGV 参数以全局数组的方式传入到 Lua 环境中。
2. 设置伪客户端的目标数据库为调用者客户端的目标数据库：fake_client->db = caller_client->db，确保脚本中执行的 Redis 命令访问的是正确的数据库。（Redis 是一种C/S架构，对服务器的访问入口限制为客户端）
3. 为 Lua 环境装载超时钩子，保证在脚本执行出现超时时可以杀死脚本，或者停止 Redis 服务器。
4. 执行脚本对应的 Lua 函数。
5. 如果被执行的 Lua 脚本中带有 SELECT 命令，那么在脚本执行完毕之后，伪客户端中的数据库可能已经有所改变，所以需要对调用者客户端的目标数据库进行更新： caller_client->db = fake_client->db 。
6. 执行清理操作：清除钩子、清除指向调用者客户端的指针等。
7. 将 Lua 函数执行所得的结果转换成 Redis 回应，然后传给调用者客户端。
8. 对 Lua 环境进行一次 GC —— [参考：Lua GC 的工作原理](https://blog.codingnow.com/2018/10/lua_gc.html)。

特别提示：Redis 使用串行化的方式来执行 Redis 命令，在任何特定时间段，最多只会有一个脚本在 Lua 环境里运行。因此，整个 Redis 服务器只需要创建一个 Lua 环境，并且很多对脚本的控制直接转移到了对 Lua 环境的设置。
（每次执行脚本，是否都要初始化 Lua 环境，如果不是，那么是怎么做到环境不被污染的相关资料未找到）

![Lua script 执行过程](/images/redis-simple-description/Lua-script-process.png)


## 参考资料
[Redis 命令参考——功能文档](http://redisdoc.com/topic/index.html)
[创建并修改 Lua 环境](http://redisbook.com/preview/script/init_lua_env.html)
[Lua 脚本](https://redisbook.readthedocs.io/en/latest/feature/scripting.html#id1)
[Redis 官方文档—— Redis Lua scripting 篇](https://redis.io/commands/eval)