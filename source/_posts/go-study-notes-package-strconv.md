---
title: go 学习笔记之 strconv 包
date: 2020/05/10
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 42830
description: 本文章主要包含 Go strconv 包及其内置类型和方法的使用.
---

`strconv` 包实现了基本数据类型和其字符串表示的相互转换.导入方式为 `import "strconv"`


# 常用函数

以下是 `strconv` 包中常用的函数

```go
// 尝试将字符串转换为bool类型,否则返回错误
// 支持如下字符串`1,0,t,f,T,F,true,false,True,False,TRUE,FALSE` 
func ParseBool(str string) (value bool, err error)
// 将字符串转换为整数,支持正负号. 
// base 指定进制, 如果为0,会根据 s 自行判断("0x"是16进制,"0"是8进制,否则是10进制)
func ParseInt(s string, base int, bitSize int) (i int64, err error)
// 将字符串转换为浮点型
func ParseFloat(s string, bitSize int) (f float64, err error)
// 将布尔值转换为字符串, "true" 或 "false"
func FormatBool(b bool) string
// 返回i的base进制的字符串表示
func FormatInt(i int64, base int) string
// 将浮点数弄表示为字符串
func FormatFloat(f float64, fmt byte, prec, bitSize int) string
// 是 ParseInt(s, 10, 0)的简写.字符串转 int
func Atoi(s string) (i int, err error)
// 是 FormatInt(i, 10) 的简写. int 转字符串
func Itoa(i int) string
```