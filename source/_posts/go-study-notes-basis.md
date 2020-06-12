---
title: go 学习笔记之类型
date: 2020/04/30
tags:
  - go
categories:
  - go
abbrlink: 14282
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中 常量与变量的区别, iota 的使用方式, 引用类型及自定义类型等.'
---


## Go 中支持的数据类型有哪些

类型 | 长度 | 默认值 | 说明
:---: | :---: | :---: | :---:
bool | 1 | false |
byte | 1 | 0 | uint8 (alias for uint8)
int, uint | 4,8 | 0 | 默认整数类型,根据目标平台 32 位或 64 位置
int8, uint8 | 1 | 0 | -127 ~ 128, 0 ~ 255
int16, uint16 | 2 | 0 | -32768 ~ 32767, 0 ~ 65535
int32, uint32 | 4 | 0 | -2^31 ~ 2^31 -1, 0~2^32
int64, uint64 | 8 | 0 |
float32 | 4 | 0.0 |
float64 | 8 | 0.0 | 默认浮点类型数据
complex64 | 8 |  |
complex128 | 16 |  |
rune | 4 | 0 | int32 (alias for uint8)
uintptr | 4, 8 | 0 | 足够存储指针的 uint
string |  | "" | 字符串, 默认为空字符串, 而非 nil
array | | | 数组
struct | | | 结构体
function | | nil | 函数
interface | | nil | 接口
map | | nil | 字典, 引用类型
slice | | nil | 切片, 引用类型
channel | | nil | 通道, 引用类型

## 常量和变量有什么不同

- 常量在定义的时候必须初始化
- 常量在相同作用域下是唯一且不可修改的. 一经声明并赋值后, 常量变为"只读"
- 常量智能修饰 `bool`, 数值类型(int, float 系列), `string` 类型
- 常量不会被分配存储空间, 无法像变量那样通过内存寻址来取值, 因此无法获取其地址, 见如下代码

```go
import (
    "fmt"
)

const name string = "name"
func main() {
    age := 1
    fmt.Printf("%p", &name)  // cannot take the address of name
    fmt.Printf("%v", &age)  // ok
}
```

## `iota` 是什么? 如何使用

`iota` 是 Go 语言的常量计数器, 只能在 `const` 常量声明中使用, 多用于 `const` 常量声明块(`const ()` 定义多个常量的格式). 它有如下特点:

- `iota` 在 `const` 常量声明第一次出现时将被重置为 0
- `const` 常量声明块中每新增一**行**常量声明将使 `iota` 计数一次(值增加 1).

使用 `iota` 能简化常量定义, 在定义枚举时很有用. 举例如下:

- 每次 `const` 出现时, `iota` 初始化为0

```go
const a = iota // a = 0
const (
    b = iota // b = 0
    c        // c = 1   相当于 c = iota
)
```

- 每**行**变量声明计数器加 1

```go
const (
    a = iota  // iota = 0, a = 0
    b         // iota = 1, b = iota, 即 b = 1
    c = 100   // iota = 2, c = 100
    d = iota  // iota = 3, d = 3
)
const (
    A, B = iota + 1, iota + 2 // iota = 0, A = 1, B = 2
    C, D                      // iota = 1, C, D = iota + 1, iota + 2, C, D = 2, 3
    E, F                      // iota = 2, E, F = iota + 1, iota + 2, E, F = 3, 4
)
```

- 自定义枚举类型

```go
type color int

const (
    red color = iota
    yellow
    blue
)

// 可通过如下方式定义数量级
type ByteSize float64

const (
    _           = iota             // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota) // 1 << (10*1)
    MB                             // 1 << (10*2)
    GB                             // 1 << (10*3)
    TB                             // 1 << (10*4)
    PB                             // 1 << (10*5)
    EB                             // 1 << (10*6)
    ZB                             // 1 << (10*7)
    YB                             // 1 << (10*8)
)

const (
    Sunday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
    numberOfDays // 这个常量没有导出
)
```

## 引用类型

引用类型包括 `slice`, `map`, `channel` 这 3 种数据类型

相比数字, 数组等类型, 引用类型拥有更复杂的数据结构. 除了分配内存外, 还需要初始化一系列属性, 如指针, 长度, 甚至包括哈希分布, 数据队列等.

引用类型使用 `make()` 函数创建, 以确保完成内存分配和相关属性初始化.

引用类型之所以可以引用, 是因为我们创建引用类型的变量其实是一个包含指向底层数据结构指针的变量. 当我们在函数中传递引用类型时，其实传递的是这个变量的副本, 它所指向的底层结构并没有被复制传递, 函数中对引用类型所做的所有操作可以理解为操作该对象的内存地址, 都会作用到其底层数据结构上, 这也是引用类型传递高效的原因

本质上，我们可以理解函数传递参数都是值拷贝,只不过引用类型传递的是一个指向底层数据的指针, 所以我们在函数中操作的时候, 实际操作的是共享的底层数据值, 进而影响到所有引用到这个共享底层数据的变量

![值类型与引用类型的区别](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/go-study-notes-basis/difference_between_valueType_and_referenceType.jpg)

同理, 在进行拷贝操作时, 值类型和引用类型会将变量的值进行拷贝, 值类型中变量的值是真实的数据, 因此可以理解为深拷贝. 引用类型中变量的值是真实存储数据的内存地址, 拷贝后变量的值也是该内存地址, 二者仍然共享同一底层数据, 因此可以理解为浅拷贝.

## 自定义类型

使用 `type` 关键字自定义类型, 包括现有基础类型创建, 结构体, 函数类型等.

即便自定义类型使用了基础类型, 如 `type integer int`, 也只表明它们有相同的底层数据结构, 两者之间不存在任何关系, 属于完全不同的两种数据类型.
除操作符外, 自定义类型不会继承基础类型的其它信息(包括方法). 不能视作别名, 不能隐式转换, 不能直接用于比较表达式.

## 未命名类型(匿名类型)

与有明确标识符 `bool`, `int`, `string` 等类型相比, `array`, `slice`, `map`, `channel` 等类型与具体元素类型或长度等属性有关, 故称作未命名类型

具有相同声明的未命名类型被视为同一类型, 如:

- 具有相同基类型的指针
- 具有相同元素类型和长度的数组
- 具有相同元素类型的切片
- 具有相同键值类型的字典
- 具有相同数据类型及操作方向的通道
- 具有相同字段序列(字段名,字段类型,标签,字段顺序)的匿名结构体

```go
import (
    "fmt"
)

func main() {
    var a struct {
        x int
        y int
    }
    var b struct {
        x int
        y int
    }
    var a func(int, string)
    fmt.Println(a == b)  // true
}
```
