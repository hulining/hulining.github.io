---
title: go 学习笔记之 time 包
date: 2020/05/01
tags:
  - go
categories:
  - go
abbrlink: 46353
description: '本文章主要包含 Go time 包及其内置类型和方法的使用.time 包提供了时间日期操作的函数,包括时间的显示和计算.'
---

`time` 包提供了时间日期操作的函数, 包括时间的显示和计算.导入方式为 `import "time"`

## 常用类型定义

```go
// 星期几封装
type Weekday int

// 月份封装
type Month int

// 地点,以及该地点所在的时区封装
type Location struct {
    // 内含隐藏或非导出字段
}

// 时间点封装
type Time struct {
    // 内含隐藏或非导出字段
}

// 两个时间点间隔封装
type Duration int64
```

## 常用内置常量及变量

`time` 包中提供了如下常用常量或变量的定义

```go
// 时间对象与字符串表示相互转换时,请记住 2006-01-02 15:04:05,有如下的对应关系
// 2006: 年, 01: 月, 02: 日
// 15: 时, 04: 分, 05: 秒
// 不管位置如何,只要看到 2006/06 就表示对年进转换
const (
    January Month = 1 + iota
    February
    March
    April
    May
    June
    July
    August
    September
    October
    November
    December
)
const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
var UTC *Location = &utcLoc  // UTC 时区表示
var Local *Location = &localLoc  // 当地时区表示
```

## 常用函数

## `time` 包函数

```go
// 返回表示当前时间的 `Time` 类型对象
func Now() Time
// 返回由给定时间创建的 `Time` 类型对象
func Date(year int, month Month, day, hour, min, sec, nsec int, loc *Location) Time
// 根据 layout 指定的格式尝试将字符串转换为 `Time` 类型对象,转换成功则 `error = nil`.
// 其中 layout 必须是 `2006-01-02 15:04:05` 日期时间点, 格式可以随意变换.示例如下
func Parse(layout, value string) (Time, error)
// 休眠指定时间
func Sleep(d Duration)
```

### `Time` 结构体方法

```go
// 返回根据 layout 指定的格式返回 t 代表的时间点的格式化文本表示. 示例如下所示
func (t Time) Format(layout string) string
// 返回从 January 1, 1970 UTC 以来经过的秒数
func (t Time) Unix() int64
// 返回从 January 1, 1970 UTC 以来经过的纳秒数
func (t Time) UnixNano() int64
// 返回 UTC `Time` 对象
func (t Time) UTC() Time
// 返回时间对象 t 的年月日三个参数
func (t Time) Date() (year int, month Month, day int)
// 返回时间对象 t 的时分秒三个参数
func (t Time) Clock() (hour, min, sec int)
// 返回 t 增加 d 时间后的时间对象
func (t Time) Add(d Duration) Time
// 返回增加指定年月日后时间对象
func (t Time) AddDate(years int, months int, days int) Time
// 返回年份
func (t Time) Year() int
//返回月份.是 Month 类型,可通过 int() 强转
func (t Time) Month() Month
// 返回日
func (t Time) Day() int
// 返回该日期是一年中的第几天,[1, 365]或[1, 366]
func (t Time) YearDay() int
// 返回小时  
func (t Time) Hour() int
// 返回分钟
func (t Time) Minute() int
// 返回秒
func (t Time) Second() int
// 返回纳秒
func (t Time) Nanosecond() int
// 返回星期几,是Weekday类型,可通过 int() 强转
func (t Time) Weekday() Weekday
// 返回 Location 信息
func (t Time) Location() *Location
// 返回 Time 实例的时区规范名(如"CET")和该时区相对于 UTC 的时间偏移量
func (t Time) Zone() (name string, offset int)
// 时间比较, 是否在 u 之后
func (t Time) After(u Time) bool
// 时间比较, 是否在 u 之前
func (t Time) Before(u Time) bool
// 是否与 u 相等
func (t Time) Equal(u Time) bool
// 返回 t-u 的时间间隔
func (t Time) Sub(u Time) Duration
```

## 示例

## 时间日志对象与字符串相互转换示例

```go
//
import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()
    // layout 支持该日期时间点的部分随机组合, 可以这么理解 1 2 3 4 5 6 分别对应 月,日 时, 分, 秒, 年
    // layout 支持该日期时间点的短格式, 转换出来的字符串也为短格式
    // 将当前时间对象转换为 "yyyy-mm-dd hh:MM:ss" 形式输出
    nowStr := now.Format("2006-01-02 15:04:05")

    // nowStr := now.Format("06-1-2 15:4:5")
    // nowStr := now.Format("01-02 15:04")

    fmt.Println(nowStr)
    str := "2020年5月1日 15"
    // 当字符串为短格式时, layout 也必须为短格式
    t := time.Parse("2006年1月2日", str)
    fmt.Printf("%v, %T\n", t, t)
```
