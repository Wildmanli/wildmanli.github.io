---
title: 记录：mysql 中文乱码
date: 2019-04-06
tags: [database,mysql,encode,docker]
---
mysql数据库服务中文乱码问题已经遇见并处理过两次。
第一次是新换用mac，配置开发环境时本地数据库服务中文乱码；
第二次是前几天测试服务器环境重装后，docker运行的mysql服务中文乱码。
虽然两次在问题原因上一致，但在处理细节上还是存在明显不同。
打算记录在案，为可能存在的第三次加深印象。

## mac mysql 乱码事件
在讲述乱码事件前，补充部分关于mac中安装mysql建议

### 下载dmg安装包
访问[MYSQL下载社区](https://dev.mysql.com/downloads/mysql/)，为了安装更方便，建议下载dmg安装包。
![mac-mysql-download](/images/mysql-database-encode/mac-mysql-download.png)

### 安装 MYSQL
然后点击dmg安装包，按照提示正常安装即可。
特别注意的是：安装完成之后会弹出一个对话框，告诉我们生成了一个root账户的临时密码。请注意保存，否则重设密码会比较麻烦。
![mac-mysql-install](/images/mysql-database-encode/mac-mysql-install.png)

### 启动 MYSQL
打开系统偏好设置，正常安装后会在界面中新增一个MYSQL图标。
![mac-mysql-start](/images/mysql-database-encode/mac-mysql-start1.png)
点击将进入MYSQL设置界面。
安装之后，默认MySQL的状态是stopped，需要点击“Start MySQL Server”按钮来启动它；
启动之后，状态会变成running。
下方还有一个复选框按钮，可以设置是否在系统启动的时候自动启动MySQL，默认是勾选。如果电脑性能不错，就没必要关闭来节省开机时间。
![mac-mysql-start](/images/mysql-database-encode/mac-mysql-start2.png)

### 终端连接
{% codeblock [连接数据库] %}
$ mysql -u root -p
Enter password: 
{% endcodeblock %}
登陆成功，但是运行命令的时候会报错，提示我们需要重设密码。
{% codeblock [提示重设密码] %}
mysql> show databases;ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.mysql
{% endcodeblock %}
修改密码，新密码为666666
{% codeblock [修改密码] %}
mysql> set PASSWORD =PASSWORD('666666');
{% endcodeblock %}
修改完成后就能正常运行命令了。

### 查看MYSQL编码
运行命令：show variables like '%char%',将会显示如下：(由于截图是已经修改过编码，显示可能与真实情况不符合)
![mysql encode](/images/mysql-database-encode/mysql-encode.png)
1. character_set_client :   utf8
2. character_set_connection :   utf8
3. character_set_database   :   latin1
4. character_set_filesystem :   binary
5. character_set_results    :   uft8
6. character_set_server :   latin1
7. character_set_system :   uft8
8. character_sets_dir   :   ??

而导致中文乱码问题的原因就在与3与6的Latin1编码。需要将其修改为uft8（至于为什么初始设置为Latin1，待有机会再深究）

### 修改MYSQL编码
通过终端连接MYSQL，使用set命令逐个设置编码格式为uft8。
{% codeblock [set命令修改编码] %}
mysql> set character_set_database = utf8;
OK
mysql> set character_set_server = uft8;
OK
{% endcodeblock %}
并再次使用show variables like '%char%'查看时，发现编码已经变更。但这只是在当前会话中生效，当重启MYSQL服务后变更将失效，为了保证后续也保持修改，也只能通过修改配置文件my.cnf。
遗憾的是mac中安装的MYSQL服务并没有my.cnf配置文件（以及相关的配置文件），幸亏仍然支持，简单粗暴，直接在/etc文件夹中创建一个my.cnf，并修改配置。    
⚠️the cnf search path is: /etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf    
将my.cnf文件放入这些路径下都可有效。
{% codeblock [创建my.cnf配置文件] %}
$ cd /etc
$ mkdir my.cnf
$ vim my.cnf
//内容如下

# MySQL Server Instance Configuration File
# ----------------------------------------------------------------------
# Generated by the MySQL Server Instance Configuration Wizard
#
#
# Installation Instructions
# ----------------------------------------------------------------------
#
# On Linux you can copy this file to /etc/my.cnf to set global options,
# mysql-data-dir/my.cnf to set server-specific options
# (@localstatedir@ for this installation) or to
# ~/.my.cnf to set user-specific options.
#
# On Windows you should keep this file in the installation directory 
# of your server (e.g. C:\Program Files\MySQL\MySQL Server 4.1). To
# make sure the server reads the config file use the startup option 
# "--defaults-file". 
#
# To run run the server from the command line, execute this in a 
# command line shell, e.g.
# mysqld --defaults-file="C:\Program Files\MySQL\MySQL Server 4.1\my.ini"
#
# To install the server as a Windows service manually, execute this in a 
# command line shell, e.g.
# mysqld --install MySQL41 --defaults-file="C:\Program Files\MySQL\MySQL Server 4.1\my.ini"
#
# And then execute this in a command line shell to start the server, e.g.
# net start MySQL41
#
#
# Guildlines for editing this file
# ----------------------------------------------------------------------
#
# In this file, you can use all long options that the program supports.
# If you want to know the options a program supports, start the program
# with the "--help" option.
#
# More detailed information about the individual options can also be
# found in the manual.
#
#
# CLIENT SECTION
# ----------------------------------------------------------------------
#
# The following options will be read by MySQL client applications.
# Note that only client applications shipped by MySQL are guaranteed
# to read this section. If you want your own MySQL client program to
# honor these values, you need to specify it as an option during the
# MySQL client library initialization.
#
[client]

port=3306

[mysql]

default-character-set=utf8mb4


# SERVER SECTION
# ----------------------------------------------------------------------
#
# The following options will be read by the MySQL Server. Make sure that
# you have installed the server correctly (see above) so it reads this 
# file.
#
[mysqld]
secure-file-priv=""
# The TCP/IP Port the MySQL Server will listen on
port=3306


#Path to installation directory. All paths are usually resolved relative to this.
basedir="/usr/local/mysql/"

#Path to the database root
datadir="/usr/local/mysql/data/"

# The default character set that will be used when a new schema or table is
# created and no character set is defined
# default-character-set=utf8mb4
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci

# The default storage engine that will be used when create new tables when
default-storage-engine=INNODB

# Set the SQL mode to strict
sql-mode="STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"

# The maximum amount of concurrent sessions the MySQL server will
# allow. One of these connections will be reserved for a user with
# SUPER privileges to allow the administrator to login even if the
# connection limit has been reached.
max_connections=100

# Query cache is used to cache SELECT results and later return them
# without actual executing the same query once again. Having the query
# cache enabled may result in significant speed improvements, if your
# have a lot of identical queries and rarely changing tables. See the
# "Qcache_lowmem_prunes" status variable to check if the current value
# is high enough for your load.
# Note: In case your tables change very often or if your queries are
# textually different every time, the query cache may result in a
# slowdown instead of a performance improvement.
query_cache_size=14M

# The number of open tables for all threads. Increasing this value
# increases the number of file descriptors that mysqld requires.
# Therefore you have to make sure to set the amount of open files
# allowed to at least 4096 in the variable "open-files-limit" in
# section [mysqld_safe]
table_open_cache=1024

# Maximum size for internal (in-memory) temporary tables. If a table
# grows larger than this value, it is automatically converted to disk
# based table This limitation is for a single table. There can be many
# of them.
tmp_table_size=17M


# How many threads we should keep in a cache for reuse. When a client
# disconnects, the client's threads are put in the cache if there aren't
# more than thread_cache_size threads from before.  This greatly reduces
# the amount of thread creations needed if you have a lot of new
# connections. (Normally this doesn't give a notable performance
# improvement if you have a good thread implementation.)
thread_cache_size=8

#*** MyISAM Specific options

# The maximum size of the temporary file MySQL is allowed to use while
# recreating the index (during REPAIR, ALTER TABLE or LOAD DATA INFILE.
# If the file-size would be bigger than this, the index will be created
# through the key cache (which is slower).
myisam_max_sort_file_size=100G

# If the temporary file used for fast index creation would be bigger
# than using the key cache by the amount specified here, then prefer the
# key cache method.  This is mainly used to force long character keys in
# large tables to use the slower key cache method to create the index.
# myisam_max_extra_sort_file_size=100G

# If the temporary file used for fast index creation would be bigger
# than using the key cache by the amount specified here, then prefer the
# key cache method.  This is mainly used to force long character keys in
# large tables to use the slower key cache method to create the index.
myisam_sort_buffer_size=33M

# Size of the Key Buffer, used to cache index blocks for MyISAM tables.
# Do not set it larger than 30% of your available memory, as some memory
# is also required by the OS to cache rows. Even if you're not using
# MyISAM tables, you should still set it to 8-64M as it will also be
# used for internal temporary disk tables.
key_buffer_size=22M

# Size of the buffer used for doing full table scans of MyISAM tables.
# Allocated per thread, if a full scan is needed.
read_buffer_size=64K
read_rnd_buffer_size=256K

# This buffer is allocated when MySQL needs to rebuild the index in
# REPAIR, OPTIMZE, ALTER table statements as well as in LOAD DATA INFILE
# into an empty table. It is allocated per thread so be careful with
# large settings.
sort_buffer_size=256K


#*** INNODB Specific options ***


# Use this option if you have a MySQL server with InnoDB support enabled
# but you do not plan to use it. This will save memory and disk space
# and speed up some things.
#skip-innodb

# Additional memory pool that is used by InnoDB to store metadata
# information.  If InnoDB requires more memory for this purpose it will
# start to allocate it from the OS.  As this is fast enough on most
# recent operating systems, you normally do not need to change this
# value. SHOW INNODB STATUS will display the current amount used.
# innodb_additional_mem_pool_size=2M

# If set to 1, InnoDB will flush (fsync) the transaction logs to the
# disk at each commit, which offers full ACID behavior. If you are
# willing to compromise this safety, and you are running small
# transactions, you may set this to 0 or 2 to reduce disk I/O to the
# logs. Value 0 means that the log is only written to the log file and
# the log file flushed to disk approximately once per second. Value 2
# means the log is written to the log file at each commit, but the log
# file is only flushed to disk approximately once per second.
innodb_flush_log_at_trx_commit=1

# The size of the buffer InnoDB uses for buffering log data. As soon as
# it is full, InnoDB will have to flush it to disk. As it is flushed
# once per second anyway, it does not make sense to have it very large
# (even with long transactions).
innodb_log_buffer_size=1M

# InnoDB, unlike MyISAM, uses a buffer pool to cache both indexes and
# row data. The bigger you set this the less disk I/O is needed to
# access data in tables. On a dedicated database server you may set this
# parameter up to 80% of the machine physical memory size. Do not set it
# too large, though, because competition of the physical memory may
# cause paging in the operating system.  Note that on 32bit systems you
# might be limited to 2-3.5G of user level memory per process, so do not
# set it too high.
innodb_buffer_pool_size=40M

# Size of each log file in a log group. You should set the combined size
# of log files to about 25%-100% of your buffer pool size to avoid
# unneeded buffer pool flush activity on log file overwrite. However,
# note that a larger logfile size will increase the time needed for the
# recovery process.
innodb_log_file_size=10M

# Number of threads allowed inside the InnoDB kernel. The optimal value
# depends highly on the application, hardware as well as the OS
# scheduler properties. A too high value may lead to thread thrashing.
innodb_thread_concurrency=10
{% endcodeblock %}

其中设置编码作用的内容如下：
1. 在[mysql]下方添加 default-character-set=utf8mb4
2. 在[mysqld] 下方添加 character_set_server=utf8mb4

### 特别提示
unix、mac、windows 系统中mysql的大小写区分规则默认配置不同[详细参考](https://dev.mysql.com/doc/refman/8.0/en/identifier-case-sensitivity.html)
1. unix 系统下：库名、表名、表别名、变量名严格区分大小写；列名、列别名忽略大小写   lower_case_table_names=1
2. windows 系统下：都不区分大小写  lower_case_table_names=0
3. mac OS系统下：都不区分大小写    lower_case_table_names=2