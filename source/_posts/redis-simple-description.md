---
title: Redis简述
---
REmote DIctionary Server(Redis) 是一个由Salvatore Sanfilippo写的key-value存储系统。   
Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。   
它通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Hash), 列表(list), 集合(sets) 和 有序集合(sorted sets)等类型。
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

### Redis List类型
List数据结构是链表结构，是双向的，可以在链表左，右两边分别操作，列表允许重复数据；
也可以把list看成一种队列，所以在很多时候可以用redis用作消息队列，这个时候它的作用就类似于activeMq；
应用案例有时间轴数据，评论列表，消息传递等等，它可以提供简便的分页，读写操作。

#### 操作命令
{% codeblock [LPUSH操作] %}
redis> LPUSH "key" "value" ["value" ...]
(integer) 1
{% endcodeblock %}

将一个或多个值 value 插入到列表 key 的表头：
1. 如果有多个 value 值，那么各个 value 值按从左到右的顺序依次插入到表头： 比如说，对空列表 mylist 执行命令 LPUSH mylist a b c ，列表的值将是 c b a ，这等同于原子性地执行 LPUSH mylist a 、 LPUSH mylist b 和 LPUSH mylist c 三个命令。
2. 如果 key 不存在，一个空列表会被创建并执行 LPUSH 操作。
3. 当 key 存在但不是列表类型时，返回一个错误。

#### 参考链接
[redis 五种数据结构详解](https://www.cnblogs.com/sdgf/p/6244937.html)
[Redis命令参考](http://redisdoc.com/string/index.html)

### Redis Hash类型
### Redis Set类型
### Redis Zset类型