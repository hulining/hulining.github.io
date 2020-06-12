---
title: go 学习笔记之表达式
date: 2020/04/30
tags:
  - go
categories:
  - go
abbrlink: 46843
description: >-
  记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中指针, for 循环, if-else,switch, select 流程控制及
  fallthrough,goto,break,continue 等关键字
---

## 指针

指针会分配内存空间, 相当于一个专门用来保存对象地址的整型变量. 它有自身的内存地址, 其值为另外一个对象的内存地址

- 取址运算符 `&` 用于获取对象地址.
- 指针运算符 `*` 用于间接引用目标对象.

```go
import (
    "fmt"
)

func main() {
    x := 100
    p := &x
    fmt.Printf("p 的类型为:%T, p 的地址为:%p\n", p, &p) // p 为 *int 类型,有自己的地址,
    fmt.Printf("p 的值为:%v, x 的地址为:%p\n", p, &x)  // 变量 p 保存了 x 的内存地址
    fmt.Printf("x 的值为:%v, p 的引用值为:%v\n", x, *p) // p 的引用为变量 x 的值
}
```

在结构体指针引用使用过程中, 由于 Go 语言底层做了优化, 对结构体属性的访问可直接使用指针变量进行, 如:

```go
func main() {
    type Student struct {
        Name string
        age int
    }
    stu := Student{"tom", 20}
    p := &stu
    (*p).Name="jack"  // 先使用指针进行引用后, 对引用对象属性进行修改, 但是看起来比较复杂
    p.Name = "jack"   // 与上述引用相同, Go 语言在底层做了优化, 看起来/写起来比较简单
}
```

## `for` 循环

主要有两种形式

```go
for i := 0; i < 10; i++ {
    // do something
}

for index, value := range x {
    // do something
    // 其中, index 表示 x 的索引下标, value 为 x 对应索引下标的值
    // 可使用 _ 对其中元素进行站位操作
}
```

Go 中没有 `while`, `until` 等关键字, 因此没有 `while true` 或 `do...until` 等形式, 我们可以使用 `for` + `if` 条件判断达到类似的效果. 如下:

```go
// 一直循环
for {
    // do something
}

// do...while 形式
for {
    // do something
    if expression {
        break
    }
}
```

## `if-else` 流程控制

`if-else` 控制比较简单, 一般形式如下

```go
if expression {
    // do something
} else if {
    // do something
} else {
    // do something
}
```

## `switch` 流程控制

与 `if-else` 类似相同, `switch` 语句也用于选择执行, 但具体使用场景会有所不同

一般来说具有以下特点:

- 支持 `switch` 代码块内赋值等初始化语句
- `case` 语句支持多个匹配条件逻辑或
- 只有全部 `case` 语句匹配失败时才会执行 `default` 块
- 相邻的空 `case` 表示执行的代码块为空
- 无需显示指定 `break` 语句, `case` 执行完毕后自动中断

```go
import (
    "fmt"
)

func main() {
    switch x := 5; x {
    case 0:     // 隐式 "case 0: break;"
    case 1, 2, 3, 4, 5:
        fmt.Println("工作日")
    case 6, 7:
        fmt.Println("周末")
    default:
        fmt.Println("错误输入")
    }
}
```

`switch` 可以用来替换 `if` 语句, 被省略的 `switch` 条件表达式默认为 `true`, 继而与 `case` 表达式结果匹配: 如

```go
import (
    "fmt"
)

func main() {
    x := 20
    switch  {
    case x > 0 && x < 18:
        fmt.Println("未成年人")
    case x >= 18:
        fmt.Println("成年人")
    default:
        fmt.Println("错误输入")
    }
}
```

## `select` 随机选择控制

`select` 是 Go 中的一个控制结构,其语法与 `switch` 流程控制类似.`select` 会随机执行一个可运行的 case.如果没有 case 可执行,则执行 default 语句块或阻塞直到有 case 可运行

一般来说语法结构如下:

```go
select {
case communicating_1:
    statements
case communicating_2:
    statements
default: // 可选
    statements
}
```

一般来说具有以下特点:

- 每个 case 都必须是一个通信过程
- 如果有多个 case 可运行, select 会随机选择一个执行,其它不会被执行
- 如果没有 case 可执行,则如果有 default 子句,执行该语句块;否则阻塞,直到某个通信可进行

示例如下

```go
import (
    "fmt"
)

func main() {
    done := make(chan int)
    data := []chan int{make(chan int, 3)}

    go func() {
        defer close(done)
        for i := 0; i < 10; i++ {
            select {
            case data[len(data)-1] <- i:
                fmt.Println("case 1 执行")
            case data[len(data)-1] <- i:
                fmt.Println("case 2 执行")
            default:
                data = append(data, make(chan int, 3))  // default 语句用于添加新的 channel 等.
            }
        }
    }()
    <-done // 阻塞, 直到 goroutine 执行结束
    for i := 0; i < len(data); i++ {
        c := data[i]
        close(c)
        for x := range c {
            fmt.Println(x)
        }
    }
}
// 可看到 case 1 2 随机执行
```

### `fallthrough` 关键字

`fallthrough` 关键字用于继续执行**下一个** `case` 语句, 必须放在 `case` 代码块末尾

```go
import (
    "fmt"
)

func main() {
    switch x := 0; x {
    case 0:
        fmt.Println("0")
        fallthrough
    case 1, 2, 3, 4, 5:
        fmt.Println("工作日")
    case 6, 7:
        fmt.Println("周末")
    default:
        fmt.Println("错误输入")
    }
}
// 输出如下
// 0
// 工作日
```

## `goto`, `continue`, `break` 关键字

- `goto` 用于定点跳转, 代码跳转到指定标签的代码位置, 但不能跳转到其他函数或内层代码块内
- `continue` 仅用于 `for` 循环, 终止本次循环逻辑, 立即进入下一轮循环.
- `break` 用于 `switch`, `for`, `select` 语句, 跳出整个语句块

配合标签, `continue` 和 `break` 可在多层嵌套中指定目标层级.

```go
import (
    "fmt"
)

func main() {
outer:
    for x := 0; x < 5; x++ {
        for y := 0; y < 10; y++ {
            if y > 2 {
                fmt.Println()
                continue outer
            }
            if x > 2 {
                break outer
            }
            fmt.Printf("%v:%v", x, y)
        }
    }
}
// 输出如下:
// 0:0 0:1 0:2
// 1:0 1:1 1:2
// 2:0 2:1 2:2
```
