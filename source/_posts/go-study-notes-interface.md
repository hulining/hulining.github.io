---
title: go 学习笔记之接口
date: 2020/05/02
tags:
  - go
categories:
  - go
abbrlink: 34261
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中接口的定义及使用'
---

## 定义

一般来说, 定义方式如下:

```go
type interfaceName interface {
    // 方法名(参数列表) (返回值列表)
}
```

- 接口内如果没有方法声明, 则该接口为空接口(`interface{}`). 空接口是所有类型均实现的接口, 可被赋值为任何类型的对象
- 接口的结构体内不能有字段
- 只能声明方法, 不能实现, 不能包含任何方法体
- 可嵌入其它接口类型, 相当于将其其它接口申明的方法集导入, 这要求不能有同名方法.
- 目标类型必须定义了接口中包含的全部方法才算实现了该接口. 如果两个接口定义的方法相同, 且目标类型实现了其中方法, 则该类型同时实现了两个接口
- 通过 `var i1 = interface{}` 声明的接口, 默认值为 `nil`
- 将对象赋值给接口变量时, 会复制该对象, 此时不能通过接口修改原始对象的值; 将对象指针赋值给接口, 那么接口内存储的就是指针的复制品, 可以通过接口修改原始对象的值

## 类型转换/断言

类型推断可将接口变量转换为原始类型, 或判断是否实现了某个更具体的接口类型. 一般来说类型推断表达式如下:

```go
Type, ok := x.(Type)    // 判断是否可以转换为原始类型
Type, ok := x.(interfaceName)  // 判断是否实现某个接口
```

示例代码如下:

```go
import (
    "fmt"
)

type N int
type interfaceName interface {
  test() int
}

func (n N) test() int {
  return 0
}

func main() {
  var n N = 100
  var x interface{} = n
    // 在 `switch` 语句中使用 `x.(type)` 对 x 类型进行判断
    switch x.(type) {
    case nil:
      fmt.Println("nil")
    case int, string, N:
      fmt.Println(x)
    case func(int) string:
      fmt.Println("func")
    default:
      fmt.Println("unknown")
    }
    // 判断 x 是否可以转换为 N 类型
  if n, ok := x.(N); ok {
    fmt.Printf("%T, %v\n", n)
  }
    // 判断 n 是否也实现了 interfaceName 接口
  if n, ok := x.(interfaceName); ok {
    fmt.Printf("%T, %v\n", n, n)
  }  
}
```
