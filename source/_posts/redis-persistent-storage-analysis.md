---
title: Redis持久化方案简述
date: 2019-05-21 15:24:31
tags: [redis,AOF,RDB]
---
## Redis持久存储方案
Redis服务提供四种持久化存储方案：RDB、AOF、虚拟内存（VM）和　DISKSTORE;    
[官方文档](https://redis.io/topics/persistence)仅能够看到RDB与AOF两种方案的说明;
如果您愿意，只要服务器正在运行，您就可以根据需要禁用持久性。

### RDB存储
RDB持久性以指定的时间间隔执行"数据集"的时间点快照;
![RDB-CopyOnWrite](/images/redis-simple-description/redis-RDB.png)
1. CopyOnWrite思路：Redis服务在dump过程一般还是会收到数据写操作请求，一方面保证在dump操作过程数据不会变化，另一方面保证服务响应客户端操作；
2. 快照操作过程中不能影响上一次的备份数据：Redis服务会在磁盘上创建一个临时文件进行数据操作，待操作成功后才会用这个临时文件替换掉上一次的备份（只有全部操作完成的快照才可替换原备份）

### AOF存储
1. AOF持久性记录服务器接收的每个"写入操作"，将在服务器启动时再次播放，重建原始数据集；
2. 使用与Redis协议本身相同的格式以仅"追加方式"记录命令（不进行覆盖）；
3. 当Redis太大时，Redis能够重写AOF文件(一段时间内对统一"key"进行了添加与删除操作，那么重写AOF文件将会去除改"key"的操作记录)；
4. [AOF 功能的运作机制](https://redisbook.readthedocs.io/en/latest/internal/aof.html)    

疑问：如果Redis响应客户端方式为多线程执行，AOF存储的指令集的顺序保证，以及恢复数据时执行是否会出现问题？
————Redis是单进程单线程处理客户端响应。

#### AOF文件的写入与存储
每当服务器常规任务函数被执行或者事件处理器被执行时，flushAppendOnlyFile 函数都会被调用，这个函数执行以下两个工作：
WRITE：根据条件（参数设置以及默认触发条件），将缓存写入到 AOF 文件（采用追加的方式，原AOF文件会加载到内存）
SAVE：根据条件（参数设置以及默认触发条件），将 AOF 文件保存到磁盘中（完成持久化操作）

AOF 所使用的保存模式决定步骤 WRITE 和 SAVE 的调用条件。
##### AOF_FSYNC_NO
不保存，但每次执行flushAppendOnlyFile函数 都会执行 WRITE，直到满足如下条件之一才会执行SAVE。
1. Redis 被关闭（正常关闭，非故障）
2. AOF 功能被关闭（只要服务器正在运行，您就可以根据需要禁用持久性）
3. 系统的写缓存被刷新（可能是缓存已经被写满，或者定期保存操作被执行）

##### AOF_FSYNC_EVERYSEC
SAVE 原则上每隔一秒钟就会执行一次,因为 SAVE 操作是由后台子线程调用的，不会引起服务器主进程阻塞。
![AOF_FSYNC_EVERYSEC](/images/redis-simple-description/AOF-FSYNC-EVERYSEC.png)
子线程正在执行 SAVE ，并且：
1. 这个 SAVE 的执行时间未超过 2 秒，那么程序直接返回，并不执行 WRITE 或新的 SAVE 。
2. 这个 SAVE 已经执行超过 2 秒，那么程序执行 WRITE ，但不执行新的 SAVE（因为这时 WRITE 的写入必须等待子线程先完成（旧的）SAVE，因此这里 WRITE 会比平时阻塞更长时间）

子线程没有在执行 SAVE ，并且：
3. 上次成功执行 SAVE 距今不超过 1 秒，那么程序执行 WRITE ，但不执行 SAVE
4. 上次成功执行 SAVE 距今已经超过 1 秒，那么程序执行 WRITE 和 SAVE

##### AOF_FSYNC_ALWAYS ：每执行一个命令保存一次。
在这种模式下，每次执行完一个命令之后， WRITE 和 SAVE 都会被执行。
SAVE 是由 Redis 主进程执行的，SAVE 执行期间，主进程会被阻塞，不能接受命令请求。

#### AOF文件重写
问题：AOF文件通过同步Redis服务器所执行的指令，实现了数据库状态的记录。随着运行时间越长，AOF文件会变得越来越大。
方案：AOF文件重写——创建一个新的AOF文件来代替原有的AOF文件,新AOF文件和原有AOF文件保存的数据库状态完全一样,但新 AOF 文件的体积小于等于原有AOF文件的体积。
注意：AOF重写并不需要对原有的AOF文件进行任何写入和读取，而是针对的是数据库中键的当前值。
##### 子进程执行
使用辅佐性的维护手段，将AOF文件重写程序放到"子进程"里执行，存在好处：
1. "子进程"进行AOF重写期间，"主进程"可以继续处理命令请求（如果仍然在主进程执行，对原AOF文件的重写操作将会在数据库添加"锁"，变相导致进程阻塞）
2. "子进程"复制了主进程的数据副本，但数据空间独立（使用子进程而不是线程，可以在避免锁的情况下，保证数据的安全性）

##### 数据不一致处理
子进程在进行AOF重写期间，主进程仍继续处理命令，而新的指令对现有的数据进行修改，这会让当前数据库数据和重写后的AOF文件中的数据不一致。
![AOF-REWRITE](/images/redis-simple-description/AOF-REWRITE.png)
当子进程在执行AOF重写时，"主进程"需要执行以下三个工作：
1. 处理命令请求（正常响应请求）
2. 将写命令追加到现有的"AOF文件"中（在AOF重写失败后，备份恢复仍然可用）
3. 将写命令追加到"AOF重写缓存"中（在进行AOF重写过程中，新的执行指令都记录到AOF重写缓存中，用以追加到重写后的AOF文件）

当子进程完成AOF重写之后，它会向父进程发送一个完成信号，父进程在接到完成信号之后，会调用一个信号处理函数，并完成以下工作：
1. 将AOF重写缓存中的内容全部写入到新AOF文件中（执行完毕之后，现有AOF文件、新AOF文件和数据库三者的状态就完全一致了）
2. 对新的AOF文件进行改名，覆盖原有的AOF文件（执行完毕之后，程序就完成了新旧两个AOF文件的交替）

### 参考链接
[Redis持久化存储方案](https://blog.csdn.net/liupeifeng3514/article/details/79048767)
[为什么说Redis是单线程的以及Redis为什么这么快！](https://blog.csdn.net/xlgen157387/article/details/79470556)
[fork出的子进程和父进程](https://blog.csdn.net/u013851082/article/details/76902046)