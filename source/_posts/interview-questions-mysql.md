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

- IO 次数少: B+ 树的中间结点只存放索引,数据都存在叶子结点中,因此中间结点可以存更多的数据,让索引树更加矮胖
- 范围查询效率更高: B 树需要中序遍历整个树,而 B+ 树需要遍历叶结点中的链表
- 查询效率更加稳定: 每次查询都需要从根结点到叶结点,路径长度相同,所以每次查询的效率都差不多

### 聚簇索引与非聚簇索引

- 聚簇索引

聚簇索引按照每张表的主键构造一棵 B+ 树,叶子节点存放整张表的行记录数据.每个数据页都通过一个双向链表来进行链接.

聚簇索引对于主键的排序查找和范围查找非常快.叶子节点就是用户所要查询的数据.

- 非聚簇索引

非聚簇索引中叶子节点不包含行记录的全部数据,而是带有指向数据的指针.

当通过非聚簇索引查找数据时,InnoDB 存储引擎会遍历非聚簇索引并通过叶级别的指针获得指向主键索引的主键,然后再通过主键索引找到一个完整的行记录.

### 在哪些地方适合创建索引

- 经常被查询的字段
- 某列常作为最大值,最小值
- 经常用作表连接的字段
- 经常出现在 `ORDER BY`/`GROUP BY`/`DISDINCT` 后面的字段

#### 使用索引时的注意事项

- 建立索引的字段应该非空
- 选择数据重复率低的字段添加索引,如性别不适合做索引

## 锁

### 什么是乐观锁和悲观锁？

- 悲观锁: 认为数据随时会被修改,因此每次读取数据之前都会上锁,防止其它事务读取或修改数据.应用于数据更新比较频繁的场景
- 乐观锁: 操作数据时不会上锁,但是更新时会判断在此期间有没有别的事务更新这个数据.若被更新过,则失败重试.适用于读多写少的场景.乐观锁的实现方式有:
  - 加一个版本号或者时间戳字段,每次数据更新时同时更新这个字段
  - 先读取想要更新的字段或者所有字段,更新的时候比较一下,只有字段没有变化才进行更新

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

- 读未提交: 能够读取到未提交的数据,产生脏读问题
- 读已提交: 只能读取到已经提交的数据,解决脏读问题,产生不可重读问题.两次同样的查询,可能得到不一样的结果(在读过程中,其它事务修改了数据).
- 可重读(默认): 在事务开启时,不再允许修改操作.解决不可重读问题,但仍然存在幻读.幻读是指某个事务在读取范围内记录时,另一个事务又在该范围内插入新纪录,之前事务再次读取该范围的记录时,会产生幻行.
- 串行化: 强制事务串行执行,避免幻读问题

## 复制

### MySQL 复制过程及原理

过程:

- 主库把数据更改记录在二进制日志中
- 备库将主库上的日志复制到自己的中继日志中
- 备库读取中继日志中的事件,并将其重放到备库上

原理:

- 主库记录二进制日志,在每次提交事务完成数据更新之前,主库将数据更新的事件记录到二进制日志中.
- 备库将主库的二进制日志复制到本地中继日志(Relay log)中.备库会启动一个 I/O 线程,并使用该线程与主库建立连接,读取二进制日志中的事件,复制到备库的本地中继日志中.如果该线程追上了主库,则进入睡眠状态,直到有信号通知.
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

```conf
# 主库配置
log_bin = mysql-bin # 开启二进制日志并设置二进制文件前缀名
server_id = 10 # 唯一服务器ID,可自行设置
sync_binlog = 1 # MySQL 每次在提交事务之前会将二进制日志同步到磁盘上,保证服务器在崩溃时不丢失事件
innodb_flush_logs_at_trx_commit = 0,1,2  #  每次提交事务时会记录日志,开启会对性能产生影响,但是提升准确性

# 主库执行
create user repl_user@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE ON *.* TO repl_user IDENTIFIED BY '123456';
flush privileges;
show master status; # 获取到当前 binlog 及 position
```

```conf
# 从库配置
relay_log = /path/to/relay_log/relay-bin
log_slave_updates = 1 # 允许从节点在二进制日志中记录更新事务
log_bin = mysql-bin
skip_slave_start # 阻止从库在崩溃后自动启动复制
read_only = 1

# 从库执行
CHANGE MASTER TO MASTER_HOST='server1',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='123456',
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
4. 应尽量避免在 where 子句中使用 `!=,<,>` 操作符或对字段进行 null 值判断,否则将放弃使用索引而进行全表扫描
5. 只返回必要的列: 最好不要使用 SELECT * 语句
6. 将一个大连接查询分解成对每一个表进行一次单表查询,然后在应用程序中进行关联.好处有:
   1. 高效利用缓存: 单表查询的缓存结果更可能被其它查询使用到,连接查询时,其中一个表发生变化,则整个缓存都会失效.
   2. 减少锁竞争

### 常用 SQL 优化

1. 大量插入数据

   - 使用多个值的 insert 语句,避免连接关闭等消耗
   - 使用 `insert delayed` 获取更高速度
   - 使用 `load data infile` 代替 `insert`
   - 将索引文件和数据文件放在不通磁盘上,增大数据写入速率

2. 创建合适的索引,并按照索引顺序进行查询

   - 考虑在 where 及 order by 列上创建索引
   - 避免在 where 字句中进行空值判断,或使用 `!=`,`<>`,`or`,`not in`,`like '%xxx'`,函数/计算,否则将导致放弃使用索引而进行全表扫描

### drop,delete,truncate 的区别

- drop 删除表有关的一切(数据,结构,约束,键),为 DDL 操作.**不能回滚**
- delete 删除表中数据,可增加 where 语句,但每次从表中删除一行,较慢.**每行删除操作都会记录 binlog,可以回滚**
- truncate 一次性清空表数据,保留表结构,约束,键.**只有一条 binlog 日志记录此操作不能回滚**

## 表连接方式有哪些?

- 内连接(inner join): 仅将两个表中满足连接条件的行组合起来作为结果集
  - 等值连接：给定条件进行查询
- 外连接(outer join)
  - 左连接: 左边表的所有数据都有显示出来,右边的表数据只显示共同有的那部分,没有对应的部分补 NULL
  - 右连接: 和左连接相反
  - 全外连接(Full Outer Join)查询出左表和右表所有数据,但是去除两表的重复数据
- 交叉连接(Cross Join): 返回两表的笛卡尔积(对于所含数据分别为 m,n 的表,返回 m*n 行结果)

## 什么是 SQL 注入,如何防止

如果用户在提交参数时,在其中掺杂了一些 SQL 关键字或者特殊符号(比如,or # --),就可能会导致SQL语句的语意发生变化.从而执行一些意外的操作(在不知道密码的情况下也能登陆,甚至在不知道用户名和密码的情况下也能登陆),这就是 SQL 注入攻击.

有如下解决办法:

- 检查用户输入的合法性
- 对进入数据库的特殊字符进行转义处理,或编码转换
- 预编译 SQL,使用 `PreparedStatement` 对象来替代 `Statement` 对象.`PreparedStatement` 对象会将要执行的 SQL 发送到数据库进行预处理
- 限制 web 应用连接数据库的权限
