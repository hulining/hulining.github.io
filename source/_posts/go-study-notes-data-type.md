---
title: go 学习笔记之常用数据类型
date: 2020/04/30
tags:
  - go
categories:
  - go
abbrlink: 16200
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中字符串, 数组, 切片, 字典, 结构体等数据类型的定义方式及其注意事项'
---

## 字符串 `string`

字符串 `string` 是 Go 中的基本类型, 它是一个不可变的 UTF-8 字符(byte)序列.

特点和注意事项如下:

- 字符串默认值不是 `nil`, 而是 `""`
- 允许以索引方式访问字节数组, 但不能获取元素地址或对其中元素修改
- 可以使用反引号(\`)定义不做任何转义处理的原始字符串, 支持跨行

```go
import (
    "fmt"
)

func main(){
    var str string
    fmt.Println(str) // ""
    str = "string"
    fmt.Print(str[1:3])       // tr
    // fmt.Printf("%p", &str[1]) // 报错 cannot take the address of 'str[1]', 不能获取 str[1] 的指针地址
    // str[1] = 'x'              // 报错, cannot assign to str[1], 字符串是不可变的, 不能对其中元素进行修改
    multiLine := `line \n
            line2`
    fmt.Println(multiLine)    // 会按照  multiLine 的原始字符输出,不做任何转义操作
}
```

- 使用 for 循环遍历字符串时, 分 `byte` 和 `rune` 两种方式

```go
import (
    "fmt"
)

func main(){
    str := "Hello 北京!"

    for i := 0; i < len(str); i++ {
        fmt.Printf("%v: %v\n", i, str[i])
    }
    for i, v := range str {
        fmt.Printf("%v: %v\n", i, v)
    }
}
// 可以看到 for-range (rune)方式输出可以看到中文字符, 而 for-i (byte)方式则不能
```

- 要修改字符串, 需要将其转换为 `[]rune` 或 `[]byte` 数组, 待修改完成后再使用 `string()` 强制转换回来.

## 数组 `array`

数组的一般形式如下:

```go
var name [len]Type = [len]Type{element1, element2, element3...}
```

特点和注意事项如下:

- 数组用于保存相同类型元素的集合, 数组是有长度的. 只有元素类型与长度都相同的数组才属于同一种类型.
- 数组支持使用索引访问元素内容
- 对于定义时不确定长度的数组, 可用 `[...]Type` 进行定义, 但一旦初始化赋值, 其长度也会随之确定. 定义多维数组时, 仅允许第一维数组长度使用 `...`
- 内置函数 `len()`, `cap()` 均返回数组的第一维长度
- 指针数组是指元素为指针的数组, 如 `arr := [3]*int{&x, &y}` , 数组指针是内存中数组的地址, 如`&arr`. 数组指针可来操作元素
- 数组是值类型, 赋值和传参都会复制整个数组数据, 可以使用指针或切片,避免数据复制

代码示例如下:

```go
import (
    "fmt"
)

func test1(x [2]int) {
    fmt.Printf("x: %p, %v\n", &x, x)
}

func test2(p *[2]int) {
    fmt.Printf("p: %p, %v\n", p, *p)
    p[1] += 100
}

func main() {
    a := [2]int{1, 2}
    b := a
    fmt.Printf("a: %p, %v\n", &a, a)
    fmt.Printf("b: %p, %v\n", &b, b)
    test1(a)
    test2(&a)
    fmt.Printf("a: %p, %v\n", &a, a)
}
// a: 0xc00000a0d0, [1 2]
// b: 0xc00000a0e0, [1 2]
// x: 0xc00000a130, [1 2]
// p: 0xc00000a0d0, [1 2]
// a: 0xc00000a0d0, [1 102]
```

## 切片 `slice`

切片的一般形式如下

```go
var sliceName []Type = []Type{element1, element2, element3...}
```

特点和注意事项如下:

- 可使用内置函数 `make([]Type, len, cap)` 初始化一个长度为 `len` 元素值为默认值的切片, 并完成长度为 `cap` 用于存储底层数据的数组的内存分配. 其中 `cap` 为容量, 可以省略(默认为 `len`), 否则必须大于等于 `len`. 切片的长度及容量均可超过其初始定义时的长度和容量, 此时会为底层数组重新分配内存地址空间
- 切片是**引用类型**, 所有在函数内的对其元素的操作都会作用到其底层数据结构上
- 可基于数组或数组指针创建切片, 以开始索引或结束索引确定切片所引用的数据字段. 不支持反向索引
- 使用形如 `var s []int` 创建的切片为 `nil`
- 切片支持使用索引号访问元素内容
- `append()` 可用于向切片尾部添加数据, 返回对象的内存地址不会发生改变, 但是如果 `append` 后的切片超过 `cap` 容量, 则会为底层数组重新分配内存空间. 新分配的 `cap` 容量为一般初始 `cap` 容量的整数倍
- `copy(dst, src []Type)` 可用于在两个切片对象间复制数据, 最终所复制的数据以较短的切片长度为准.

代码示例如下:

```go
import (
    "fmt"
)

func main() {
    var nilSli []int
    fmt.Println(nilArr == nil)  // true
    sli := make([]int, 2, 5)
    fmt.Printf("%p,%v,%v,%p\n", &sli, len(sli), cap(sli), &sli[0])
    arr = append(arr, []int{1, 2, 3, 4, 5, 6}...)
    fmt.Printf("%p,%v,%v,%p\n", &sli, len(sli), cap(sli), &sli[0])  
    // 可以看到 append 前后 `&arr` 没有变化, 但是 `cap(arr) 变为原来的 2 倍,实际上是对底层数组重新分配了新的内存空间(原来数组的内存空间也会被重新分配)
    p := &sli
    p0 := &sli[0]
    p1 := &sli[1]
    fmt.Printf("%p,%p,%p\n", p, p0, p1)
    (*p)[0] += 100
    *p1 += 100
    fmt.Printf("%v\n", sli) // [100 100 1 2 3 4 5 6]
}
// 输出
// true
// 0xc0000044c0,2,5,0xc00000c300
// 0xc0000044c0,8,10,0xc000014190
// 0xc0000044c0,0xc000014190,0xc000014198
// [100 100 1 2 3 4 5 6]
```

```go
import "fmt"

func change(s []int) {
    s[0] += 100
    fmt.Printf("%v,%p\n", s, &s)
}

func main() {
    s := []int{1, 20, 32}  // 等价于 var s = make([]int, 3, 3); s = []int{1, 20, 32}
    fmt.Printf("%v,%p\n", s, &s)
    change(s)
    fmt.Printf("%v,%p\n", s, &s)
}
// 输出
// [1 20 32],0xc0000044c0
// [101 20 32],0xc000004520
// [101 20 32],0xc0000044c0
```

## 字典 `map`

字典的一般形式如下:

```go
var mapName map[KeyType]ValueType = map[KeyType]ValueType{
    KeyType: ValueType,
    // ...
}
```

特点和注意事项如下:

- 字典是引用类型, 一般使用 `make(map[KeyType]ValueType, size)` 或初始化语句表达式创建. 内容为空的字典已经做了初始化操作, 与 `nil` 是不同的
- 函数 `len()` 返回当前键值对数量
- Key 必须支持等值比较(`==`, `!=`), 类型可以为数字,字符串,指针,数组,结构体,接口等类型.常见的一般为字符串
- 访问键值时, 推荐使用 `v, ok := m[key]` 模式. 可以通过 `ok` 判断键是否存在. 如果存在, `ok` 会返回 `true`; 否则, 返回 `false`
- 字典是一种无序键值对集合, 对字典进行 for-range 遍历, 每次遍历的次序都不相同
- 因为访问安全和哈希算法的缘故, 字典被设计为 "not addressable", 因此不能直接修改 `value` 成员(结构体或数组). 只能返回整个 `value`, 修改后重新对字典赋值. 或直接使用指针类型

示例代码如下:

```go
import (
    "fmt"
)

func main() {
    var nilMap map[string]string
    fmt.Println(nilMap == nil)
    m := make(map[string]user, 2)
    fmt.Printf("%p,%v\n", &m, m)
    m["tom"] = user{"tom", 20}
    m["jack"] = user{"jack", 21}
    m["lucy"] = user{"lucy", 19}
    v, ok := m["liming"]
    if ok {
        fmt.Println(v)
    }
    fmt.Printf("%p,%v\n", &m, m)
    for k, v := range m {
        fmt.Println(k, v)  // 每次输出循序都不一样
    }
    //m["lucy"].age += 1 // 报错, cannot assign to `m["lucy"].age`, 不能对字典值的成员变量直接赋值
    lucy := m["lucy"]
    lucy.age += 1
    m["lucy"] = lucy
    fmt.Println(m)
}
// 输出如下
// true
// 0xc000006030,map[]
// 0xc000006030,map[jack:{jack 21} lucy:{tom 19} tom:{tom 20}]
// tom {tom 20}
// jack {jack 21}
// lucy {tom 19}
// map[jack:{jack 21} lucy:{tom 20} tom:{tom 20}]
```

## 结构体 `struct`

`struct` 将多个不通类型的字段序列打包成一个复合类型, 类似于`类`的概念. 一般定义方式如下

```go
type structName struct{
    fieldName fieldType
    // ...
}
```

特点和注意事项如下:

- 字段名必须唯一, 支持使用 `_` 补位字段名(忽略该字段). 支持直接指定字段类型, 从而使用匿名字段, 实际上是使用与类型名相同的字段
- 可直接使用 `structName.fieldName` 访问结构体字段. 对于匿名字段, 可以使用 `structName.fieldType` 访问该匿名字段
- 由于 Go 语言底层的优化, 可使用结构体指针直接操作结构体字段 如, 如果 `p := &structName`, 则 `*(p).fieldName` 等价于 `p.fieldName`
- 结构体支持多个结构体嵌套, 可以理解为:**内层结构体作为外层结构体一个或多个匿名或非匿名字段**.
- **结构体嵌套过程中, 如果内层结构体(innerStructName)与外层结构体(outerStructName)有相同的字段 `fieldName`, 则 `outerStructName.fieldName` 表示访问外层结构体. 内层结构体 `fieldName` 字段只能通过 `outerStructName.innerStructFieldName.fieldName` 访问.
如果外层结构体没有内层结构体字段`innerFieldName`, 则可以通过 `outerStructName.innerFieldName` 访问 `innerFieldName` 字段.其实就是就近原则**
- 如果多个内层结构体有相同的字段, 则必须指定内层结构体名称才能访问到该字段,否则会编译报错

示例代码如下:

```go
import (
    "fmt"
)

type test struct {
    Name string
    Age  int
    _    string
    _    string
    int
    //int  // 报错, 重复定义 int
}
type user struct {
    name string
    inneruser1
    inneruser2
}
type inneruser1 struct {
    name string
    age  int
}
type inneruser2 struct {
    name  string
    age   int
    score float64
}

func main() {
    t := test{
        Name: "test",
        Age:  20,
        int:  4,
    }
    fmt.Println(t)
    u := user{
        name:       "outeruser",
        inneruser1: inneruser1{"inneruser1", 10},
        inneruser2: inneruser2{"inneruser2", 20, 89.5},
    }
    fmt.Println(u.name)
    // fmt.Println(u.age) // 报错, Ambiguous reference 'age', 编译器搞不清使用哪个 age
    fmt.Println(u.inneruser1.age)
    fmt.Println(u.score)
}
// 输出
// {test 20   4}
// outeruser
// 10
// 89.5
```

### 字段标签

字段标签并不是注释, 而是用来对字段进行描述的元数据.

- 在运行期间, 可用反射获取标签信息, 常被用作格式校验, 数据库关系映射等.
- 由于 Go 中私有变量与可导入变量是通过首字母大小写区分的. 因此对于可导入变量, 还可以用作 json 格式化输出字段

示例如下:

```go
import (
    "fmt"
    "reflect"
    "encoding/json"
)

type user struct {
    Name string `json:"name"` // `` 反引号中的内容为该字段的 tag 标签
    Age  int    `json:"age"`
}

func main() {
    u := user{
        Name: "tom",
        Age:  10,
    }
    val := reflect.ValueOf(u)
    valType := val.Type()
    for i := 0; i < val.NumField(); i++ {
        fmt.Printf("%v: %v\n", valType.Field(i).Tag.Get("json"), val.Field(i))
    }

    str, err := json.Marshal(u)
    if err != nil {
        fmt.Println("格式转换出错")
    }
    fmt.Println(string(str))

}
// 输出如下:
// name: tom
// age: 10
// {"name":"tom","age":10}
```
