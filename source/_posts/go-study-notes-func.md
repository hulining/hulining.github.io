---
title: go 学习笔记之函数
date: 2020/04/30
tags:
  - go
categories:
  - go
abbrlink: 47797
description: >-
  记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中函数的定义, 参数, 返回值等相关内容,
  简单描述了匿名函数和闭包的定义方式及其可能发生的错误. 本文还记录了 defer 关键字的使用及函数在发送错误时的处理办法.
---


## 定义

关键字 `func` 用于定义函数. Go 语言中函数有以下特点:

- 无须前置声明
- 不支持命名嵌套定义
- 不支持同名函数重载
- 不支持默认参数
- 支持不定长变参
- 支持多返回值
- 支持命名返回值
- 支持匿名函数和闭包

一般来说,表示方式如下:

```go
func 函数名(函数参数) (返回值列表) {
    // expression
    return
}
// 带参数
// `args... argsType` 本质上是一个切片, 表示可以传入多个参数, 该参数形式只能放在参数的最后
func FuncName(arg argType, args ...argsType) {  
    // expression
}
// 带返回值
// returnName 可省略, 返回值可以为逗号 `,` 分割的多个值
func FuncName() (returnName returnType) {
    // expression
}
// 匿名函数
// 匿名函数只是没有名称, 其使用方式基本与普通函数没有区别, 多用于只调用一次或定义后立即调用的情况
a := func (arg argType) (returnName returnType) {  
    // expression
}

// 调用方式
FuncName(args)
a(args)
```

## 参数

函数的参数可视作局部变量, 因此不能在相同层次定义同名变量

> 函数的形参是指函数中定义的参数, 实参则是函数调用时所传递的参数.

不管传入的参数是指针,引用类型,还是其它类型参数,都是值拷贝传递.区别在于是拷贝指针,还是拷贝目标对象.

```go
import (
    "fmt"
)

func Change(x int) {
    x = 100
    fmt.Println(x)
}
func ChangePtr(x *int) {
    *x = 200
    fmt.Println(*x)
}
func main() {
    x := 10
    fmt.Println(x)  // 10
    Change(x)       // 100
    fmt.Println(x)  // 10
    x = 10
    fmt.Println(x)  // 10
    ChangePtr(&x)   // 200
    fmt.Println(x)  // 200
}
```

### 不定长变参

变参本质上是一个切片,只能接收一到多个同类型的参数,且必须放在参数列表末尾

```go
// `args... argsType` 本质上是一个切片, 表示可以传入多个参数, 该参数形式只能放在参数的最后
func FuncName(arg argType, args ...argsType) {  
    // expression
}
```

将切片作为变参传入函数时,需进行展开操作.如果是数组,则需要将其转化为切片. 切片作为引用类型, 其在函数中所做的一切操作会影响到底层的数据. 如果需要可以使用内置函数 `copy()` 复制底层数据.

```go
import (
    "fmt"
)

func test(a ...int){
    for i, _ := range a {
        a[i] += 100
    }
}
func main() {
    arr := [3]int{10, 20, 30}
    s := arr[:]
    scopy := make([]int, len(s))
    copy(scopy, s)  // 将切片 s 底层数据复制一份到 scopy
    test(s...)
    fmt.Println(arr)
    fmt.Println(s)
}
```

## 返回值

- 有返回值的语句, 必须有明确的 `return` 终止语句
- Go 函数支持多返回值模式
- 返回值在命名时, 其实已经在函数内部隐式创建了指定类型和名称的变量, 可当作局部变量使用, 且不能在函数体内对已经命名的返回值变量 `a` 使用形如 `a := xxx` 的变量定义表达式
- 返回值在命名时, 需要对全部返回值命名, 否则会编译出错

```go
import (
    "fmt"
)

func test() (a int) {
    fmt.Println(a) // 输出 0
    a := 10        // 报错,No new variables on left side of :=, 表示 a 不是一个新定义的变量, 只能使用 = 对其赋值
    return
}
```

## 匿名函数

除没有名字外, 匿名函数与普通函数完全相同. 匿名函数可以直接被调用, 保存到变量, 作为参数或返回值

- 直接执行或赋值给变量

```go
import (
    "fmt"
)

func main() {
    // 直接执行
    func (s string){
        fmt.Println(s)
    }("hello world")

    // 赋值给变量
    add := func (x, y int) int {
        return x + y
    }
    fmt.Println(add(1, 2))
}
```

- 作为参数传递

```go
import (
    "fmt"
)

func test(f func()) {
    f()
}
func main() {
    test(func (){
        fmt.Println("hello world")
    })
}
```

- 作为返回值

```go
import (
    "fmt"
)

func test() func(int, int) int {
    return func(x, y int) int {
        return x + y
    }
}
func main() {
    add := test()
    fmt.Println(add(1, 2))
}
```

## 闭包

- 示例

```go
import (
    "fmt"
)

func test(x int) func(){
    return func(){
        fmt.Println(x)
    }
}
func main(){
    f := test(123)
    f()  // 输出 123
}
```

就上述代码而言, `test` 返回的匿名函数会引用上下文环境变量 `x`. 当该函数在 `main` 中执行时, 它依然可以读取 `x` 的值, 这种现象就称作闭包.

闭包通过指针引用环境变量, 那么可能导致其生命周期延长, 还有可能发生 "延迟求值"

```go
import (
    "fmt"
)

func test() []func() {
    var s []func()
    for i := 0; i < 2; i++ {
        s = append(s, func() {
        fmt.Println(&i, i)
        })
    }
    return s
}

func main() {
    for _, f := range test() {
        f()
    }
}
// 输出如下:
// 0xc000062090 2
// 0xc000062090 2
```

在以上示例中, for 循环复用局部变量 i, 每次添加的匿名函数引用的变量是同一变量. 添加仅仅是将匿名函数放入列表, 并未执行. 因此在 main 函数调用这些函数时, 它们读取的是环境变量 i 最后一次循环时的值,为 2.

解决方法就是每次使用不同的环境变量或传参复制, 让各自的闭包环境各不相同

```go
import (
    "fmt"
)

func test() []func() {
    var s []func()
    for i := 0; i < 2; i++ {
        x := i // x 在每次循环都重新定义
        s = append(s, func() {
            fmt.Println(&x, x)
        })
    }
    return s
}
```

## `defer` 语句延迟调用

- `defer` 语句定义的语句直到当前函数执行结束前在被执行, 常用于资源释放, 解除绑定, 错误处理等操作. 多个 `defer` 语句会按照"先进后出"(FILO)的次序执行.
- `defer` 语句定义的语句会被延迟调用, 其中传入的参数被复制并被缓存起来. 调用时使用的参数为 `defer` 语句定义时的参数值. 如

```go
import (
    "fmt"
)

func main() {
    x, y := 1, 2
    defer func(a, b int) {
        fmt.Println("传入 defer 语句中 a b 值分别为 ", a, b)  // 只有传入的参数保存了当时的状态
        fmt.Println("不以 defer 语句参数方式输出 x y 值分别为", x, y)
    }(x, y)
    x += 100
    y += 100
    fmt.Println("函数结束前 x y 值为", x, y)
}
// 输出如下:
// 函数结束前 x y 值为 101 102
// 传入 defer 语句中 a b 值分别为  1 2
// 不以 defer 语句参数方式输出 x y 值分别为 101 102
```

### 误用

`defer` 语句在函数结束时才被执行. 不合理的使用方式会浪费资源.

如下案例是在 for 循环中不恰当使用 `defer` 语句导致文件关闭时间延长

```go
import (
    "fmt"
    "os"
)

func main() {
    for i := 0; i < 100; i++ {
        path := fmt.Sprintf("log/%d.txt", i)
        file, err := os.Open(path)
        if err != nil {
            fmt.Println(err)
        }
        // 这个文件关闭操作在 main 函数结束时才会执行,而不是在当前循环结束后执行
        defer file.Close()
        // do something...
    }
}
```

我们应该将带有 `defer` 语句的循环体重构为函数,在循环中调用. 这样, 在每次循环执行后, `defer` 语句都会被执行一次, 即时释放资源

```go
import (
    "fmt"
    "os"
)

func main() {
    do := func(n int){
        path := fmt.Sprintf("log/%d.txt", n)
        file, err := os.Open(path)
        if err != nil {
            fmt.Println(err)
        }
        // 这个文件关闭操作在该函数结束时调用
        defer file.Close()
        // do something...
    }

    for i := 0; i < 100; i++ {
        do()
    }
}
```

## 错误处理

## `error`

官方推荐的做法是返回 `error`

标准库将 `error` 定义为接口类型, 以便实现自定义错误类型. 我们在自定义错误类型时, 只需实现该接口即可

```go
// Go 内置的 error 接口
type error interface {
    Error() string
}

// 自定义 Error
type DivError struct {
    s string
}

func (divError *DivError) Error() string {
    return divError.s
}
```

标准库也提供了创建 `error` 的函数, 可以方便地创建包含错误文本的 `error` 对象

```go
err := errors.New("some description for error")
```

### `panic()`, `revover()` 内置函数

- `panic()` 内置函数接收 `interface` 作为参数, 会立即中断当前函数流程, 执行延迟调用并将 `panice` 向外传递
- `revover()` 内置函数返回 `interface` 对象, 多用于捕获 `panic()` 函数引发的错误. **该函数必须在延迟调用函数中才能正常工作**

```go
import (
    "fmt"
)

func catch() {
    recover()
    fmt.Println("捕获成功")
}

func main() {
    defer catch()                // 捕获成功
    defer fmt.Println(recover()) // 捕获失败
    defer recover()              // 捕获失败
    panic("error")
}
```
