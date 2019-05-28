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
- 脚本命令执行示例
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