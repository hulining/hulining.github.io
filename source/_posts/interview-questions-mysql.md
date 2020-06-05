---
title: 运维面试题之 MySQL
date: 2020/06/04
tags:
  - MySQL
categories:
  - 面试题
  - MySQL
abbrlink: 
abbrlink: 
description: 总结整理常见 MySQL 面试题,以作备忘
---

## InnoDB 和 MyIsam 存储引擎的区别

维度 | InnoDB | MyIsam
:---: | :---: | :---:
事务支持 | 支持(ACID) | 不支持
锁 | 事务和行级锁 | 表级锁
全文索引 | 不支持 | 支持 FULLTEXT 类型的全文索引
主键/外键 | 支持 | 不支持
存储结构 | 表数据保存在一个数据文件中 | 存储为3个文件, .frm 文件存储表定义; .MDY 数据文件; .MYI 索引文件
存储空间 | 需要更多的内存和存储 | 可被压缩
备份及恢复 | 拷贝数据文件,备份 binlog,使用 mysqldump 进行备份 | 

InnoDB 存储引擎支持事务、支持外键、支持非锁定读、行锁设计

## 如何判断 MySQL 是否同步,该如何使其同步

`show slave status\G` 查看同步信息

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

## 如何定位锁问题

查看命令 `show engine innodb status` 的输出,并通过查询 `information_schema` 库中三个有关锁的表进行查看锁的详情

- `innodb_trx`: 当前运行的所有事务
- `innodb_locks`: 当前出现的锁
- `innodb_lock_waits`: 锁等待的对应关系

## MySQL 如何减少主从复制延迟

- 慢 SQL 语句过多,在开发或架构上做优化,减少慢 SQL 语句
- 主从复制单线程,主库写入过快.主库写,安全性较高,建议开启`sync_binlog=1`,`innodb_flush_log_at_trx_commit=1`之类的设置,而从库可以不开启
- master负载过大.架构的前端要加buffer及缓存层,如 redis
- slave负载过大,使用多台slave来分摊读请求.再从这些slave中取一台专用的服务器
- 网络延迟
- 从库硬件性能较差

## MySQL 重置密码

```bash
mysqld --skip-grant-tables --skip-networking
mysql> use mysql;
mysql> update user set password=password("new_password") where user="root";
mysql> flush privileges;
mysqld --正常启动测试
```

## MySQL 复制过程及原理

过程:

- 主库把数据更改记录在二进制日志中
- 备库将主库上的日志复制到自己的中继日志中
- 备库读取中继日志中的事件,并将其重放到备库上

原理:

- 主库记录二进制日志,在每次提交事务完成数据更新之前,主库将数据更新的事件记录到二进制日志中.
- 备库将主库的二进制日志复制到本地中继日志中.备库会启动一个 I/O 线程,并使用该线程与主库建立连接,读取二进制日志中的事件,复制到备库的本地中继日志中.如果该线程追上了主库,则进入睡眠状态,知道有信号通知.
- 备库的 SQL 线程从中继日志中读取事件并在备库中执行,直到 SQL 线程追上 I/O 线程,从而实现备库的更新

## 为什么MySQL数据库索引选择使用B+树?

- MySQL 的数据存放在磁盘中,读取数据时有访问磁盘的操作.在查找时,存在一个定位到磁盘中的块的过程,通过B树进行优化,提高磁盘读取定位的效率
- B+ 树是应文件系统所需而产生的一种B树的变形树,只有叶子节点保存数据,非叶子节点只保存索引.
- B+ 树为所有叶子节点增加一个链指针组成链表,方便顺序查找

## MySQL 事务有哪些特性

- 原子性(atomicity): 事务是一个不可分割的操作,要么全部正确执行,要么全部不执行
- 一致性(consistency): 事务把数据从一种一致性状态转化为另一种一致性状态,事务开始前后,数据库完整性没有被破坏
- 隔离性(isolation): 要求每个读写事务之间是分开的,在提交事务之前对其它事务是不可见的
- 持久性(durability): 事务一旦提交,结果是永久性的

## 事务的隔离级别

- 读未提交: 能够读取到未提交的数据,脏读
- 读已提交: 只能读取到已经提交的数据,解决脏读问题,产生不可重读问题.两次同样的查询,可能得到不一样的结果.
- 可重读(默认): 解决不可重读问题,但仍然存在幻读,某个事务在读取范围内记录时,另一个事务又在该范围内插入新纪录,之前事务再次读取该范围的记录时,会产生幻行.
- 串行化: 强制事务串行执行,避免幻读问题

## SQL 优化

1. 通过 `show status like 'Com_%'` 查看各种语句执行的次数,其中 `Slow_queries` 表示慢查询次数
2. 通过 `show processlist` 命令查看当前 MySQL 正在进行的线程状态,是否锁表等
3. 通过 `explain` 分析低效 SQL 的执行计划
    1. `select_type` 表示选择的类型
    2. `type` 表示 MySQL 的访问方式,all(全表扫描),index(索引),range(索引范围),
    3. `possible_key` 表示查询时可能用到的索引
    4. `key` 表示实际用到的索引
    5. `key_len` 表示用到索引字段的长度
    6. `rows`: 扫描行的数量

## 常用 SQL 优化

1. 大量插入数据

- 使用多个值的 insert 语句,避免连接关闭等消耗
- 使用 `insert delayed` 获取更高速度
- 使用 `load data infile` 代替 `insert`
- 将索引文件和数据文件放在不通磁盘上,增大数据写入速率

2. 创建合适的索引,并按照索引顺序进行查询

- 考虑在 where 及 order by 列上创建索引
- 避免在 where 字句中进行null判断,或使用 !=,<>,in,or,not in,like '%xxx',函数/计算,,否则将导致放弃使用索引而进行全表扫描

## drop,delete,truncate 的区别

- drop 删除表有关的一切(数据,结构,约束,键),为 DDL 操作
- delete 删除表中所有数据,但每次从表中删除一行,较慢,可增加 where 语句
- truncate 一次性清空表数据,保留表结构,约束,键
