---
title: 运维面试题之 MySQL
date: 2020/06/04
tags:
  - MySQL
  - 面试
categories:
  - MySQL
abbrlink: 
description: 总结整理常见 MySQL 面试题,以作备忘
---

## 存储引擎

### InnoDB 和 MyIsam 存储引擎的区别

维度 | InnoDB | MyIsam
:---: | :---: | :---:
事务支持 | 支持(ACID) | 不支持
锁 | 事务和行级锁 | 表级锁
全文索引 | 不支持 | 支持 FULLTEXT 类型的全文索引
主键/外键 | 支持 | 不支持
存储结构 | 表数据保存在一个数据文件中 | 存储为3个文件, .frm 文件存储表定义; .MDY 数据文件; .MYI 索引文件
存储空间 | 需要更多的内存和存储 | 可被压缩
备份及恢复 | 拷贝数据文件,备份 binlog,使用 mysqldump 进行备份 |

InnoDB 存储引擎支持事务,支持外键,支持非锁定读,行锁设计

## 文件

### 配置文件

配置文件读取顺序如下: `/etc/my.cnf -> /etc/mysql/my.cnf -> /usr/local/mysql/etc/my.cnf ->  ~/.my.cnf`.后面的配置会覆盖前面的配置.如果忘记,可通过 `mysql --help | grep my.cnf` 进行查看.

### 日志文件

- 错误日志: 记录 MySQL 启动,运行,关闭过程中发生的错误.可通过 `show variables like '%log_error%';` 查看错误日志文件位置.
- 慢查询日志: 记录查询时间超过 `long_query_time` 参数值(默认为 10)的所有 SQL 语句
  - `slow_query_log`: 设置是否开启慢查询日志
  - `slow_query_log_file`: 慢查询日志文件
- 二进制日志(binlog): 记录对 MySQL 数据库执行更改的所有操作,不包括 `SELECT` 和 `SHOW` 之类的操作.可用于复制备份恢复数据
  - `log_bin`: 是否记录 binlog
  - `log_bin_index`: binlog 索引
  - `binlog_format`: 记录二进制日志的格式. `STATEMENT` 记录逻辑 SQL 语句.`ROW` 格式记录表行更改情况(在执行 UPDATE 时,数据与原来一致,则不不会执行).`MIXED` 默认使用 `STATEMENT` 格式,一些情况使用 `ROW` 格式.
  - `max_binlog_size`: binlog 文件最大大小,若超过该值,则产生新的 binlog 文件,并使索引加 1.
  - `binlog_cache_size`: 未提交的 binlog 会被记录到缓存中,该选项配置会话缓存大小,默认为 32 KB.
  - `sync_binlog`: 表示缓冲数据写入磁盘的方式.默认为 0.1 表示同步写磁盘来写二进制日志.

## 索引

### 为什么MySQL数据库索引选择使用 B+ 树

- B+ 树中父节点元素都出现在子节点,所有叶子节点包含了全量元素信息,每个叶子节点都带有指向下一个节点的指针,形成了一个有序链表。相比于 B 树,更易于范围查找
- B 树中的每个节点带有卫星数据(所谓卫星数据,指的是索引元素指向的数据记录,比如数据库中的一行).而 B+ 树中只有叶子节点带有卫星数据(中间节点仅仅是索引),因此相同的磁盘页可以容纳更多的节点元素.这就意味着,相同数据量的情况下,B+ 树比 B 树更加"矮胖",查询 IO 次数更少.

### 聚簇索引与非聚簇索引

- 聚簇索引

聚簇索引按照每张表的主键构造一棵 B+ 树,叶子节点存放整张表的行记录数据.每个数据页都通过一个双向链表来进行链接.

聚簇索引对于主键的排序查找和范围查找非常快.叶子节点就是用户索要查询的数据.

- 非聚簇索引

非聚簇索引中叶子节点不包含行记录的全部数据,而是带有指向数据的指针.

当通过非聚簇索引查找数据时,InnoDB 存储引擎会遍历非聚簇索引并通过叶级别的指针获得指向主键索引的主键,然后再通过主键找到一个完整的行记录.

## 锁

### 如何定位锁问题

查看命令 `show engine innodb status` 的输出,并通过查询 `information_schema` 库中三个有关锁的表进行查看锁的详情

- `innodb_trx`: 当前运行的所有事务
- `innodb_locks`: 当前出现的锁
- `innodb_lock_waits`: 锁等待的对应关系

## 事务

### MySQL 事务有哪些特性

- 原子性(atomicity): 事务是一个不可分割的操作,要么全部正确执行,要么全部不执行
- 一致性(consistency): 事务把数据从一种一致性状态转化为另一种一致性状态,事务开始前后,数据库完整性没有被破坏
- 隔离性(isolation): 要求每个读写事务之间是分开的,在提交事务之前对其它事务是不可见的
- 持久性(durability): 事务一旦提交,结果是永久性的

### 事务的隔离级别

- 读未提交: 能够读取到未提交的数据,脏读
- 读已提交: 只能读取到已经提交的数据,解决脏读问题,产生不可重读问题.两次同样的查询,可能得到不一样的结果.
- 可重读(默认): 解决不可重读问题,但仍然存在幻读,某个事务在读取范围内记录时,另一个事务又在该范围内插入新纪录,之前事务再次读取该范围的记录时,会产生幻行.
- 串行化: 强制事务串行执行,避免幻读问题

## 复制

### MySQL 复制过程及原理

过程:

- 主库把数据更改记录在二进制日志中
- 备库将主库上的日志复制到自己的中继日志中
- 备库读取中继日志中的事件,并将其重放到备库上

原理:

- 主库记录二进制日志,在每次提交事务完成数据更新之前,主库将数据更新的事件记录到二进制日志中.
- 备库将主库的二进制日志复制到本地中继日志中.备库会启动一个 I/O 线程,并使用该线程与主库建立连接,读取二进制日志中的事件,复制到备库的本地中继日志中.如果该线程追上了主库,则进入睡眠状态,直到有信号通知.
- 备库的 SQL 线程从中继日志中读取事件并在备库中执行,直到 SQL 线程追上 I/O 线程,从而实现备库的更新

### 如何判断 MySQL 是否同步,该如何使其同步

使用 `show slave status\G` 查看同步信息

```text
Slave_IO_State:
Slave_IO_Running:
Slave_SQL_Running:
Slave_SQL_Running_State:
Last_IO_Errno:
Last_IO_Error:
Last_SQL_Errno:
Last_SQL_Error:
Read_Master_Log_Pos: 256364229
Exec_Master_Log_Pos: 256364229
Seconds_Behind_Master: 0
SQL_Delay: 0
```

```text
# 主库配置
log_bin = mysql-bin # 开启二进制日志并设置二进制文件前缀名
server_id = 10 # 唯一服务器ID,可自行设置
sync_binlog = 1 # MySQL 每次在提交事务之前会将二进制日志同步到磁盘上,保证服务器在崩溃时不丢失事
innodb_flush_logs_at_trx_commit  #  每次提交事务时会记录日志,开启会对性能产生影响,但是提升准确性

# 从库配置
relay_log = /path/to/relay_log/relay-bin
log_slave_updates = 1 # 允许从节点在二进制日志中记录更新事务
log_bin = mysql-bin
skip_slave_start # 阻止从库在崩溃后自动启动复制
read_only = 1

# 从库执行
CHANGE MASTER TO MASTER_HOST='server1',
    MASTER_USER='repl',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=0;
```

### MySQL 如何减少主从复制延迟

- 慢 SQL 语句过多,在开发或架构上做优化,减少慢 SQL 语句
- 主从复制单线程,主库写入过快.主库写,安全性较高,建议开启`sync_binlog=1`,`innodb_flush_log_at_trx_commit=1` 之类的设置,而从库可以不开启
- master 负载过大.架构的前端要加 buffer 及缓存层,如 redis
- slave 负载过大,使用多台slave来分摊读请求.再从这些 slave 中取一台专用的服务器
- 网络延迟
- 从库硬件性能较差

## 优化

### MySQL 重置密码

```bash
mysqld --skip-grant-tables --skip-networking
mysql> use mysql;
mysql> update user set password=password("new_password") where user="root";
mysql> flush privileges;
mysqld --正常启动测试
```

### SQL 优化

1. 通过 `show status like 'Com_%'` 查看各种语句执行的次数,其中 `Slow_queries` 表示慢查询次数
2. 通过 `show processlist` 命令查看当前 MySQL 正在进行的线程状态,是否锁表等
3. 通过 `explain` 分析低效 SQL 的执行计划
    1. `select_type` 表示选择的类型
    2. `type` 表示 MySQL 的访问方式,all(全表扫描),index(索引),range(索引范围),
    3. `possible_key` 表示查询时可能用到的索引
    4. `key` 表示实际用到的索引
    5. `key_len` 表示用到索引字段的长度
    6. `rows`: 扫描行的数量

### 常用 SQL 优化

1. 大量插入数据

- 使用多个值的 insert 语句,避免连接关闭等消耗
- 使用 `insert delayed` 获取更高速度
- 使用 `load data infile` 代替 `insert`
- 将索引文件和数据文件放在不通磁盘上,增大数据写入速率

2. 创建合适的索引,并按照索引顺序进行查询

- 考虑在 where 及 order by 列上创建索引
- 避免在 where 字句中进行空值判断,或使用 !=,<>,in,or,not in,like '%xxx',函数/计算,否则将导致放弃使用索引而进行全表扫描

### drop,delete,truncate 的区别

- drop 删除表有关的一切(数据,结构,约束,键),为 DDL 操作
- delete 删除表中所有数据,但每次从表中删除一行,较慢,可增加 where 语句
- truncate 一次性清空表数据,保留表结构,约束,键
