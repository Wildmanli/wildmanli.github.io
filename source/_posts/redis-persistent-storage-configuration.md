---
title: Redis持久化存储配置简述
date: 2019-05-21 15:28:37
tags: [redis,RDB,AOF]
---
## Redis持久化存储配置说明
Redis支持两种持久化存储形式：RDB、AOF。
前者采用snapshot方式，是通过周期性的将内存中的数据快照写入RDB文件来实现；
后者采用command-log方式，是通过记录Redis进程的接收到的"写操作"（update）来实现。
下面将分别介绍两种方式在redis.conf文件中的配置。
### RDB相关配置
- save <seconds> <changes>  snapshot方式持久化的save策略，支持多设置混用（只要满足其中之一就可触发操作），例如：
{% codeblock [save 示例] %}
save 900 1  #表示900秒内至少1个key发生变化
save 300 100    #表示300秒内至少100个key发生变化
save 60 1000    #表示60秒内至少1000个key发生变化
{% endcodeblock %}
只要上述3个条件其中一个满足，将会触发save。另外，如果想要关掉RDB持久化存储，只需要将所有save配置注释即可
- stop-writes-on-bgsave-error   指定Redis在后台dump磁盘出错时的行为，默认为yes，表示若后台dump出错，则RedisServer拒绝新的写入请求，
通过这种方式来引起用户警觉，避免因用户未发现异常而引起更大的事故。
- rdbcompression    RDB文件是否压缩存储，默认为yes。压缩时会消耗CPU资源，却可以节省磁盘空间。
- rdbchecksum   RDB文件是否需要CRC64校验, 默认为yes。
校验时会在生成RDB文件后计算其CRC64并将结果追加至文件尾。
同样，Redis启动Load RDB时，也会先计算该文件的CRC64并与dump时的计算结果对比。
    1. 好处：严格保证RDB的完整性及安全性
    2. 代价：会在dump/load时损失10%的性能。如果要最大化Redis的性能，这个配置项应该用no关掉
- dbfilename    指定RDB文件名，默认为dump.rdb 
- dir   指定RDB文件存放目录的"路径"。
特别提示：若包含多级路径，则相关父路径需事先mkdir出来，否则启动失败。
{% codeblock [RDB存储配置块] %}
################################ SNAPSHOTTING  ################################
#
# Save the DB on disk:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   In the example below the behaviour will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#
#   save ""

save 900 1
save 300 10
save 60 10000

# By default Redis will stop accepting writes if RDB snapshots are enabled
# (at least one save point) and the latest background save failed.
# This will make the user aware (in a hard way) that data is not persisting
# on disk properly, otherwise chances are that no one will notice and some
# disaster will happen.
#
# If the background saving process will start working again Redis will
# automatically allow writes again.
#
# However if you have setup your proper monitoring of the Redis server
# and persistence, you may want to disable this feature so that Redis will
# continue to work as usual even if there are problems with disk,
# permissions, and so forth.
stop-writes-on-bgsave-error yes

# Compress string objects using LZF when dump .rdb databases?
# For default that's set to 'yes' as it's almost always a win.
# If you want to save some CPU in the saving child set it to 'no' but
# the dataset will likely be bigger if you have compressible values or keys.
rdbcompression yes

# Since version 5 of RDB a CRC64 checksum is placed at the end of the file.
# This makes the format more resistant to corruption but there is a performance
# hit to pay (around 10%) when saving and loading RDB files, so you can disable it
# for maximum performances.
#
# RDB files created with checksum disabled have a checksum of zero that will
# tell the loading code to skip the check.
rdbchecksum yes

# The filename where to dump the DB
dbfilename dump.rdb

# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir /usr/local/var/db/redis/
{% endcodeblock %}
### AOF相关配置
- appendonly    配置是否启用AOF持久化，默认为no
- appendfilename     指定aof文件名，默认为appendonly.aof 
- appendfsync   配置aof文件的同步方式，Redis支持3种方式：
    1. no => redis不主动调用fsync，何时刷盘由OS来调度；
    2. always => redis针对每个写入命令均会主动调用fsync刷磁盘；
    3. everysec => 每秒调一次fsync刷盘。
- no-appendfsync-on-rewrite 指定是否在后台aof文件rewrite期间调用fsync，默认为no，表示要调用fsync（无论后台是否有子进程在刷盘）。
备注：Redis在后台写RDB文件或重写afo文件期间会存在大量磁盘IO，此时，在某些linux系统中，调用fsync可能会阻塞。
- auto-aof-rewrite-percentage   指定Redis重写aof文件的条件，默认为100，表示与上次rewrite的aof文件大小相比，
当前aof文件增长量超过上次afo文件大小的100%时，就会触发background rewrite。若配置为0，则会禁用自动rewrite。
- auto-aof-rewrite-min-size 指定触发rewrite的aof文件大小，默认为60mb。若aof文件小于该值，即使当前文件的增量比例达到auto-aof-rewrite-percentage的配置值，
也不会触发自动rewrite。即这两个配置项同时满足时，才会触发rewrite。
- aof-load-truncated    控制是否忽略最后一条可能存在问题的指令，默认为yes。
这是应对AOF文件被截断（可能是操作过程崩溃或卷存储已满）情况发生时，"启动Redis阶段"，加载AOF文件时的一种策略：
若为yes，则会忽略掉错误，尽可能加载较多的数据，若为no，则会直接报错退出。
- aof-use-rdb-preamble  是否启用Redis 4.x提供的AOF+RDB的混合持久化方案，
若为yes，在重写AOF文件时，Redis会将数据以RDB的格式作为AOF文件的开始部分。
在重写之后，Redis会继续以AOF格式持久化写入操作。默认值为yes。
{% codeblock [AOF存储配置块] %}
############################## APPEND ONLY MODE ###############################

# By default Redis asynchronously dumps the dataset on disk. This mode is
# good enough in many applications, but an issue with the Redis process or
# a power outage may result into a few minutes of writes lost (depending on
# the configured save points).
#
# The Append Only File is an alternative persistence mode that provides
# much better durability. For instance using the default data fsync policy
# (see later in the config file) Redis can lose just one second of writes in a
# dramatic event like a server power outage, or a single write if something
# wrong with the Redis process itself happens, but the operating system is
# still running correctly.
#
# AOF and RDB persistence can be enabled at the same time without problems.
# If the AOF is enabled on startup Redis will load the AOF, that is the file
# with the better durability guarantees.
#
# Please check http://redis.io/topics/persistence for more information.

appendonly no

# The name of the append only file (default: "appendonly.aof")

appendfilename "appendonly.aof"

# The fsync() call tells the Operating System to actually write data on disk
# instead of waiting for more data in the output buffer. Some OS will really flush
# data on disk, some other OS will just try to do it ASAP.
#
# Redis supports three different modes:
#
# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log. Slow, Safest.
# everysec: fsync only one time every second. Compromise.
#
# The default is "everysec", as that's usually the right compromise between
# speed and data safety. It's up to you to understand if you can relax this to
# "no" that will let the operating system flush the output buffer when
# it wants, for better performances (but if you can live with the idea of
# some data loss consider the default persistence mode that's snapshotting),
# or on the contrary, use "always" that's very slow but a bit safer than
# everysec.
#
# More details please check the following article:
# http://antirez.com/post/redis-persistence-demystified.html
#
# If unsure, use "everysec".

# appendfsync always
appendfsync everysec
# appendfsync no

# When the AOF fsync policy is set to always or everysec, and a background
# saving process (a background save or AOF log background rewriting) is
# performing a lot of I/O against the disk, in some Linux configurations
# Redis may block too long on the fsync() call. Note that there is no fix for
# this currently, as even performing fsync in a different thread will block
# our synchronous write(2) call.
#
# In order to mitigate this problem it's possible to use the following option
# that will prevent fsync() from being called in the main process while a
# BGSAVE or BGREWRITEAOF is in progress.
#
# This means that while another child is saving, the durability of Redis is
# the same as "appendfsync none". In practical terms, this means that it is
# possible to lose up to 30 seconds of log in the worst scenario (with the
# default Linux settings).
#
# If you have latency problems turn this to "yes". Otherwise leave it as
# "no" that is the safest pick from the point of view of durability.

no-appendfsync-on-rewrite no

# Automatic rewrite of the append only file.
# Redis is able to automatically rewrite the log file implicitly calling
# BGREWRITEAOF when the AOF log size grows by the specified percentage.
#
# This is how it works: Redis remembers the size of the AOF file after the
# latest rewrite (if no rewrite has happened since the restart, the size of
# the AOF at startup is used).
#
# This base size is compared to the current size. If the current size is
# bigger than the specified percentage, the rewrite is triggered. Also
# you need to specify a minimal size for the AOF file to be rewritten, this
# is useful to avoid rewriting the AOF file even if the percentage increase
# is reached but it is still pretty small.
#
# Specify a percentage of zero in order to disable the automatic AOF
# rewrite feature.

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# An AOF file may be found to be truncated at the end during the Redis
# startup process, when the AOF data gets loaded back into memory.
# This may happen when the system where Redis is running
# crashes, especially when an ext4 filesystem is mounted without the
# data=ordered option (however this can't happen when Redis itself
# crashes or aborts but the operating system still works correctly).
#
# Redis can either exit with an error when this happens, or load as much
# data as possible (the default now) and start if the AOF file is found
# to be truncated at the end. The following option controls this behavior.
#
# If aof-load-truncated is set to yes, a truncated AOF file is loaded and
# the Redis server starts emitting a log to inform the user of the event.
# Otherwise if the option is set to no, the server aborts with an error
# and refuses to start. When the option is set to no, the user requires
# to fix the AOF file using the "redis-check-aof" utility before to restart
# the server.
#
# Note that if the AOF file will be found to be corrupted in the middle
# the server will still exit with an error. This option only applies when
# Redis will try to read more data from the AOF file but not enough bytes
# will be found.
aof-load-truncated yes
# When rewriting the AOF file, Redis is able to use an RDB preamble in the
# AOF file for faster rewrites and recoveries. When this option is turned
# on the rewritten AOF file is composed of two different stanzas:
#
#   [RDB file][AOF tail]
#
# When loading Redis recognizes that the AOF file starts with the "REDIS"
# string and loads the prefixed RDB file, and continues loading the AOF
# tail.
aof-use-rdb-preamble yes
{% endcodeblock %}