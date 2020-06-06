---
title: go 学习笔记之 sql 包
date: 2020/05/10
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 6060
description: '本文章主要包含 Go database/sql 包及其 driver 子包的内置类型和方法的使用.database/sql 包提供了有关 SQL 的数据库通用接口.driver包定义了 sql 包使用的数据库程序要实现的接口.'
---

`database/sql/driver` 包定义了 `database/sql` 包使用的数据库程序要实现的接口
`database/sql` 包提供了有关 SQL 的数据库通用接口. sql 包必须与数据库驱动程序一起使用.相关程序列表,参见 [https://github.com/golang/go/wiki/SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers)

如连接 mysql 需要使用导入 mysql 的驱动程序包

```go
import "database/sql"
import _ "github.com/go-sql-driver/mysql"

db, err := sql.Open("mysql", "user:password@/dbname")
```

# 常用类型定义

```go
// 列的类型
type ColumnType struct {
    // contains filtered or unexported fields
}
// 单个数据库连接.除非特别需要连续的单个数据库连接,否则最好 DB 对象
type Conn struct {
    // contains filtered or unexported fields
}
// 数据库对象,对于多个 goroutine 时并发安全的.会自动创建并释放连接
type DB struct {
    // contains filtered or unexported fields
}

// 数据库统计信息
type DBStats struct {
    MaxOpenConnections int // 最大连接数

    // 连接池状态
    OpenConnections int // 使用和空闲的连接数
    InUse           int // 正在使用的连接数
    Idle            int // 空闲连接数

    // 计数器
    //等待
    WaitCount         int64         // 等待的连接总数
    WaitDuration      time.Duration // 等待新连接的总时间
    MaxIdleClosed     int64         // 由于 SetMaxIdleConns 而关闭的连接总数
    MaxLifetimeClosed int64         // 由于 SetConnMaxLifetime 而关闭的连接总数
}

// 结果信息接口
type Result interface {
    // 返回数据库为响应命令而生成的整数.通常用于插入新行的自增列
    LastInsertId() (int64, error)
    // 受更新,插入或删除影响的行数.
    RowsAffected() (int64, error)
}
// QueryRow 产生的单行对象
type Row struct {
    // contains filtered or unexported fields
}
// 查询的多行结果.它的光标从第一行开始,使用 `Next()` 逐行获取下一行的值
type Rows struct {
    // contains filtered or unexported fields
}

// 预处理的语句.可以被多个 goroutine 安全并发使用
type Stmt struct {
    // contains filtered or unexported fields
}

// 事务
type Tx struct {
    // contains filtered or unexported fields
}
// 事务选项,定义了事务的隔离级别
type TxOptions struct {
    // 事务的隔离级别
    Isolation IsolationLevel
    ReadOnly  bool
}
```

# 常量及变量

```go
// 隔离级别
const (
    LevelDefault IsolationLevel = iota
    LevelReadUncommitted
    LevelReadCommitted
    LevelWriteCommitted
    LevelRepeatableRead
    LevelSnapshot
    LevelSerializable
    LevelLinearizable
)
```

# 常用函数

## `sql` 包函数

```go
// 返回已注册的驱动列表
func Drivers() []string
// 注册驱动
func Register(name string, driver driver.Driver)

// 打开一个由其数据库驱动名称 driverName 和特定驱动程序的数据源名称 dataSourceName 指定的数据库
// 该函数可能只验证参数而不创建与数据库的连接.要验证数据源名称是否有效,调用 Ping 函数
// 返回的数据库可安全的供多个 goroutine 并发使用,并维护自己的空闲连接池.因此此函数只需要调用一次
// dataSourceName 一般形式为 `username:password@protocol(address)/dbname?param=value`
func Open(driverName, dataSourceName string) (*DB, error)
// 使用连接器打开数据库,从而避免字符串的数据源名称
func OpenDB(c driver.Connector) *DB
```

## `ColumnType` 结构体方法

`ColumnType` 定义了列的类型,它包含以下方法

```go
// 返回数据库中对应的列的类型
func (ci *ColumnType) DatabaseTypeName() string
// 返回小数类型的小数精度和位数
func (ci *ColumnType) DecimalSize() (precision, scale int64, ok bool)
// 返回可变类型长度列类型(如 text 和 binary)的列类型长度
func (ci *ColumnType) Length() (length int64, ok bool)
// 返回列的名称或列的别名
func (ci *ColumnType) Name() string
// 该列是否可为空
func (ci *ColumnType) Nullable() (nullable, ok bool)
// 返回适合使用 Rows.Scan 进行扫描的 Go 反射类型
func (ci *ColumnType) ScanType() reflect.Type
```
## `DB` 结构体方法

`DB` 定义了数据库的实例对象.

```go
// 开始事务,默认的隔离级别取决与驱动程序
func (db *DB) Begin() (*Tx, error)
// 开始事务,在事务提交或回滚之前,将使用上下文.如果上下文被取消,事务将回滚
func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error)
// 关闭数据库连接,阻止启动新查询.很少用
func (db *DB) Close() error
// 打开新连接或从连接池返回现有连接. Conn 将阻塞,直到返回连接池或取消 ctx.使用后必须调用 Conn.Close 将每个 Conn 返回到数据库连接池
// 在同一 Conn 上运行的查询将在同一数据库会话中运行.
func (db *DB) Conn(ctx context.Context) (*Conn, error)
// 底层驱动
func (db *DB) Driver() driver.Driver
// 执行 query 语句,但不返回任何行.args 参数用于查询中的占位符
func (db *DB) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
// 验证与数据库的连接是否存在,必要时建立连接
func (db *DB) PingContext(ctx context.Context) error
// 为后续查询或执行做好准备.Stmt 可以并发执行多个查询或执行,当不需要 Stmt 时,需要调用 Stmt.Close 显式关闭
func (db *DB) PrepareContext(ctx context.Context, query string) (*Stmt, error)
// 执行查询语句,返回多行
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
// 执行查询语句,返回单行
func (db *DB) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
// 设置可重用连接的最长时间
func (db *DB) SetConnMaxLifetime(d time.Duration)
// 设置空闲连接池中的最大连接数
func (db *DB) SetMaxIdleConns(n int)
// 设置数据库打开的最大连接数
func (db *DB) SetMaxOpenConns(n int)
// 返回数据库统计信息
func (db *DB) Stats() DBStats
```

## `Rows` 结构体方法

```go
// 返回列的信息,如类型,长度,是否可为空.
func (rs *Rows) ColumnTypes() ([]*ColumnType, error)
// 返回列的名称
func (rs *Rows) Columns() ([]string, error)
// 准备下一行,以使用 Scan 方法获取.如果策成功,返回 ture.每次调用 Scan 时,都必须先调用该函数
func (rs *Rows) Next() bool
// 准备下一个要读取的结果集.
func (rs *Rows) NextResultSet() bool
// 将当前行的列复制到 dest 指向的值中. dest 中值的数量必须与"行"中列数相同.
func (rs *Rows) Scan(dest ...interface{}) error
```

## `Stmt` 结构体方法

```go
// 关闭
func (s *Stmt) Close() error
// 执行,并返回 Result 接口
func (s *Stmt) Exec(args ...interface{}) (Result, error)
func (s *Stmt) ExecContext(ctx context.Context, args ...interface{}) (Result, error)
// 查询,并返回查询到的行对象
func (s *Stmt) Query(args ...interface{}) (*Rows, error)
func (s *Stmt) QueryContext(ctx context.Context, args ...interface{}) (*Rows, error)
func (s *Stmt) QueryRow(args ...interface{}) *Row
func (s *Stmt) QueryRowContext(ctx context.Context, args ...interface{}) *Row
```

## `Tx` 结构体方法

```go
// 提交事务
func (tx *Tx) Commit() error
// 执行查询,并返回查询结果
func (tx *Tx) Exec(query string, args ...interface{}) (Result, error)
func (tx *Tx) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
// 事务中创建 Stmt
func (tx *Tx) Prepare(query string) (*Stmt, error)
func (tx *Tx) PrepareContext(ctx context.Context, query string) (*Stmt, error)
// 事务中查询
func (tx *Tx) Query(query string, args ...interface{}) (*Rows, error)
func (tx *Tx) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
func (tx *Tx) QueryRow(query string, args ...interface{}) *Row
func (tx *Tx) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
// 回滚事务
func (tx *Tx) Rollback() error

// 使用已有的 Stmt 创建 Stmt
func (tx *Tx) Stmt(stmt *Stmt) *Stmt
func (tx *Tx) StmtContext(ctx context.Context, stmt *Stmt) *Stmt
```

# 示例

```go
import (
    "database/sql"
    "fmt"
    "github.com/go-sql-driver/mysql"
)

func main() {
    cfg := mysql.Config{
        User:                 "username",
        Passwd:               "password",
        Net:                  "tcp",
        Addr:                 "10.71.1.27:3306",
        DBName:               "datastream-info",
        AllowNativePasswords: true,
    }
    dataSourceName := cfg.FormatDSN() // 支持使用 mysql 包中的配置转换方法转换 dataSourceName,也可以使用如下方式直接定义
    // dataSourceName := "username:password@tcp(10.71.1.27:3306)/test"
    db, err := sql.Open("mysql", dataSourceName)
    rows, err := db.Query("select * from province_num")
    var (
        id    int
        name  string
        value int
    )
    for rows.Next() {
        if err = rows.Scan(&id, &name, &value); err != nil {
            fmt.Println(err)
        }
        fmt.Println(id, name, value)
    }
}
```