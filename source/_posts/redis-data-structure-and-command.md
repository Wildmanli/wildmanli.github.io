---
title: Redis五种常用数据结构及简单操作指令
date: 2019-04-03
tags: [data structures]
---
REmote DIctionary Server(Redis) 是一个由Salvatore Sanfilippo写的key-value存储系统。   
Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。   
它通常被称为数据结构服务器，因为值（value）可以是 字符串(String)，哈希(Hash)，列表(list)，集合(sets) 和 有序集合(sorted sets)等类型。
## 数据结构
Redis支持五种类型数据结构：string，list，hash，set，zset。
### Redis String类型
string类型的数据存储是最简单的key-value存储
#### 操作命令
{% codeblock [SET操作] %}
redis> SET "key" "value" [EX seconds] [PX milliseconds] [NX|XX]
OK
{% endcodeblock %}

从 Redis 2.6.12 版本开始， SET 命令的行为可以通过一系列参数来修改:
1. EX seconds ： 将键的过期时间设置为 seconds 秒。 执行 SET key value EX seconds 的效果等同于执行 SETEX key seconds value 。  
2. PX milliseconds ： 将键的过期时间设置为 milliseconds 毫秒。 执行 SET key value PX milliseconds 的效果等同于执行 PSETEX key milliseconds value 。  
3. NX ： 只在键不存在时， 才对键进行设置操作。 执行 SET key value NX 的效果等同于执行 SETNX key value 。  
4. XX ： 只在键已经存在时， 才对键进行设置操作。

{% codeblock [GET操作] %}
redis> GET "key"
"value"
{% endcodeblock %}

{% codeblock [EXISTS操作] %}
redis> EXISTS "key" ["key" ...]
(integer) 1
{% endcodeblock %}

{% codeblock [DEL操作] %}
redis> DEL "key" ["key" ...]
(integer) 1
{% endcodeblock %}

#### 参考链接
[redis 五种数据结构详解](https://www.cnblogs.com/sdgf/p/6244937.html)
[Redis命令参考](http://redisdoc.com/string/index.html)
[更多strings介绍](https://redis.io/topics/data-types#strings)

### Redis List类型
List数据结构是链表结构，是双向的，可以在链表左，右两边分别操作，允许重复数据；
也可以把list看成一种队列，所以在很多时候可以用redis用作消息队列，这个时候它的作用就类似于activeMq；

#### 操作命令
{% codeblock [LPUSH操作] %}
redis> LPUSH "key" "value" ["value" ...]
(integer) 1
{% endcodeblock %}

{% codeblock [RPUSH操作] %}
redis> RPUSH "key" "value" ["value" ...]
(integer) 1
{% endcodeblock %}

将一个或多个值 value 插入到列表 key 的表头：
1. 如果有多个 value 值，那么各个 value 值按从左到右的顺序依次插入到表头： 比如说，对空列表 mylist 执行命令 LPUSH mylist a b c ，列表的值将是 c b a ，这等同于原子性地执行 LPUSH mylist a 、 LPUSH mylist b 和 LPUSH mylist c 三个命令。
2. 如果 key 不存在，一个空列表会被创建并执行 LPUSH/RPUSH 操作。
3. 当 key 存在但不是列表类型时，返回一个错误。
4. LPUSHX/RPUSHX功能同LPUSH/RPUSH，但是只能对已存在的key进行有效操作（不会返回错误）,且只能单值操作。

{% codeblock [LRANGE操作] %}
redis> LRANGE "key" start stop
"value1"
"value2"
"value3"
...
{% endcodeblock %}

1. 当start为0，stop为 1：返回数组包含下标为0到1共2个元素。
2. 当start为0，stop为-1：返回第0个至倒数第一个,相当于返回所有元素,注意redis中很多时候会用到负数。

{% codeblock [LPOP操作] %}
redis> LPOP "key"
"value"
{% endcodeblock %}

{% codeblock [RPOP操作] %}
redis> RPOP "key"
"value"
{% endcodeblock %}

{% codeblock [LREM操作] %}
redis> LREM "key" "count" "value"
(integer) 3
{% endcodeblock %}

根据参数 count 的值，移除列表中与参数 value 相等的元素。
count 的值可以是以下几种：
1. count > 0 : 从表头开始向表尾搜索，移除与 value 相等的元素，数量为 count；
2. count < 0 : 从表尾开始向表头搜索，移除与 value 相等的元素，数量为 count 的绝对值；（所以不存在RREM命令）
3. count = 0 : 移除表中所有与 value 相等的值。


#### 参考链接
[redis 五种数据结构详解](https://www.cnblogs.com/sdgf/p/6244937.html)
[Redis命令参考](http://redisdoc.com/string/index.html)
[更多lists介绍](https://redis.io/topics/data-types#lists)

### Redis Hash类型
redis中hash表存储数据，比较类似数据库中表的一条记录。

#### 操作命令
{% codeblock [HSET操作] %}
redis> HSET "key" "field" "value" ["field" "value" ...]
(integer) 1
{% endcodeblock %}

{% codeblock [HKEYS操作] %}
redis> HKEYS "key"
"field1"
"field2"
...
{% endcodeblock %}

{% codeblock [HGET操作] %}
redis> HGET "key" "field"
"value"
{% endcodeblock %}

{% codeblock [HEXISTS操作] %}
redis> HEXISTS "key" "field"
(integer) 0/1   //表示false/ture
{% endcodeblock %}

#### 参考链接
[redis 五种数据结构详解](https://www.cnblogs.com/sdgf/p/6244937.html)
[Redis命令参考](http://redisdoc.com/string/index.html)
[更多hashes介绍](https://redis.io/topics/data-types#hashes)

### Redis Set类型
Set 就是一个集合，集合的概念就是一堆不重复值的组合。
Redis 为集合提供了求交集、并集、差集等操作。

#### 操作命令
{% codeblock [SADD操作] %}
redis> SADD "key" "member" ["member" ...]
(integer) 2 //当设置重复数据，操作无效，但不会报错
{% endcodeblock %}

{% codeblock [SMEMBERS操作] %}
redis> SMEMBERS "key"
"member1"
"member2"
...
{% endcodeblock %}

{% codeblock [SREM操作] %}
redis> SREM "key" "member" ["member" ...]
(integer) 2
{% endcodeblock %}

#### 参考链接
[redis 五种数据结构详解](https://www.cnblogs.com/sdgf/p/6244937.html)
[Redis命令参考](http://redisdoc.com/string/index.html)
[更多sets介绍](https://redis.io/topics/data-types#sets)

### Redis Zset类型
zset是set的一个升级版本，他在set的基础上增加了一个顺序属性，这一属性在添加修改元素的时候可以指定，每次指定后，zset会自动重新按新的值调整顺序。 可以对指定键的值进行排序权重的设定，它应用排名模块比较多。    
zset集合可以完成有序执行、按照优先级执行的情况。

#### 操作命令
{% codeblock [ZADD操作] %}
redis> ZADD "key" [NX|XX] [CH] [INCR] score member [score member ...]
(integer) 2
{% endcodeblock %}

将一个或多个 member 元素及其 score 值加入到有序集 key 当中:
1. 在 Redis 2.4 版本以前，ZADD 每次只能添加一个元素。
2. 如果某个 member 已经是有序集的成员，那么更新这个 member 的 score 值，并通过重新插入这个 member 元素，来保证该 member 在正确的位置上。
3. score 值可以是整数值或双精度浮点数。
4. 如果 key 不存在，则创建一个空的有序集并执行 ZADD 操作。
5. 当 key 存在但不是有序集类型时，返回一个错误。

{% codeblock [ZRANGE操作] %}
redis> ZRANGE "key" "start" "stop" [WITHSCORES]
"member1"
[score1]
"member2"
[score2]
...
{% endcodeblock %}

{% codeblock [ZSCORE操作] %}
redis> ZSCORE "key" "member"
"score"
{% endcodeblock %}

返回有序集 key 中，成员 member 的 score 值:
1. 如果 member 元素不是有序集 key 的成员，或 key 不存在，返回 nil/(nil) 。
2. member 成员的 score 值，以字符串形式表示。

{% codeblock [ZREM操作] %}
redis> ZREM "key" "member" ["member" ...]
(integer) 2
{% endcodeblock %}

#### 参考链接
[redis 五种数据结构详解](https://www.cnblogs.com/sdgf/p/6244937.html)
[Redis命令参考](http://redisdoc.com/string/index.html)
[更多sorted-sets介绍](https://redis.io/topics/data-types#sorted-sets)