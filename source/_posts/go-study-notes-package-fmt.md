---
title: go 学习笔记之 fmt 包
date: 2020/05/16
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 28915
description: 本文章主要包含 Go fmt 包及其内置类型和方法的使用.fmt 包采用占位符的方式实现了格式化的 I/O.
---

fmt 包采用占位符的方式实现了格式化的 I/O.主要占位符如下:

```go
type User struct {
    Name string
}
user := User{Name: "name"}
```

- 常规占位符

占位符 | 说明 | 举例 | 输出
:---: | :---: | :---: | :---:
%v | 默认格式的值 | `Printf("%v", user)` | {name}
%+v | 打印结构体时,会添加字段名 | `Printf("%+v", user)` | {Name:name}
%#v | Go 语法表示的值 | `Printf("%#v", user)` | main.User{Name:"name"}
%T | Go 语法表示的值的类型 | `Printf("%T", user)` | main.User
%% | 百分号 | `Printf("%%")` | %

- 布尔占位符

占位符 | 说明 | 举例 | 输出
:---: | :---: | :---: | :---:
%t | true 或 false | `Printf("%t", true)` | true

- 整型占位符

占位符 | 说明 | 举例 | 输出
:---: | :---: | :---: | :---:
%b | 二进制表示(不以 0 开头,且只有 01) | `Printf("%b", 8)` | 1000
%d |  十进制表示(不以 0 开头,且只有0-9) | `Printf("%b", 8)` | 8
%o | 八进制表示(以 0 开头,且只有 0-7) | `Printf("%o", 32)` | 40
%x | 十六进制的小写表示(以 0x 开头) | `Printf("%x", 223)` | df
%X | 十六进制的大写表示(以 0X 开头) | `Printf("%X", 223)` | DF

- 浮点型占位符

占位符 | 说明 | 举例 | 输出
:---: | :---: | :---: | :---:
%e | 科学计数,小写表示 | `Printf("%.2e", 1024.5)` | 1.02e+03
%E | 科学计数,大写表示 | `Printf("%.2E", 1024.5)` | 1.02E+03
%f | 浮点数表示,多用于限制小数位数 | `Printf("%.2f", 22.2335)` | 2.23

- 字符

占位符 | 说明 | 举例 | 输出
:---: | :---: | :---: | :---:
%s | 字符串或字符切片的字符串表示 | `Printf("%s,%s", "str", []byte("str")` | str,str

- 指针

占位符 | 说明 | 举例 | 输出
:---: | :---: | :---: | :---:
%p | 对象的指针表示,切片为也为第一个元素的指针地址 | Printf("%p", []byte("string")) | 0xc0000160b0

```go
// 使用指定格式将数据进行格式化,并写入 w
func Fprint(w io.Writer, a ...interface{}) (n int, err error)
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
func Fprintln(w io.Writer, a ...interface{}) (n int, err error)

// 扫描从 r 读取的数据,以空格为分隔符存储到参数中,返回成功扫描的个数
func Fscan(r io.Reader, a ...interface{}) (n int, err error)
func Fscanf(r io.Reader, format string, a ...interface{}) (n int, err error)
func Fscanln(r io.Reader, a ...interface{}) (n int, err error)

// 输出到标准输出
func Print(a ...interface{}) (n int, err error)
func Printf(format string, a ...interface{}) (n int, err error)
func Println(a ...interface{}) (n int, err error)

// 从标准输入扫描数据
func Scan(a ...interface{}) (n int, err error)
func Scanf(format string, a ...interface{}) (n int, err error)
func Scanln(a ...interface{}) (n int, err error)
func Sprint(a ...interface{}) string

// 根据 format 给出的格式,返回格式化后的字符串
func Sprintf(format string, a ...interface{}) string
func Sprintln(a ...interface{}) string

// 扫描 str 参数,以空格为分隔符存储到参数中,返回成功扫描的个数
func Sscan(str string, a ...interface{}) (n int, err error)
func Sscanf(str string, format string, a ...interface{}) (n int, err error)
func Sscanln(str string, a ...interface{}) (n int, err error)
```
