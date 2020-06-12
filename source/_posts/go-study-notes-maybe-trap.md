---
title: go 学习笔记之语言陷阱
date: 2020/05/17
tags:
  - go
categories:
  - go
abbrlink: 42823
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中存在的"陷阱",加深对 Go 语言相关知识的理解,避免犯错.'
---

## `range` 复用临时变量

先来看一段简单的代码

```go
import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    arr := []int{0, 1, 2, 3}
    for i, v := range arr {
        wg.Add(1)
        go func() {
            fmt.Printf("(%v,%v)", i, v)
            wg.Done()
        }()
    }
    wg.Wait()
}
// 输出如下:
// (3,3)(3,3)(3,3)(3,3)
```

程序并没有像我们预期的一样遍历切片 `arr`,而是全部打印其索引下标.其实有两点原因会导致这个问题

- `for range` 下的迭代变量 i, v 的值是共用的
- main 函数所在的 goroutine 与后续启动的 goroutine 存在竞争关系,可通过 `go run -race main.go` 看到 goroutine 之间的竞争关系

因此,`range` 在迭代写过程中,多个 goroutine 并发地去读,导致传入闭包中 i,v 数据一直在做更改,从而出现上述情况

可以使用函数函数参数做一次数据复制,而不是闭包.如下:

```go
import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    arr := []int{0, 1, 2, 3}
    for i, v := range arr {
        wg.Add(1)
        // 这里有一个实参到形参的值拷贝
        go func(i, v int) {
            fmt.Printf("(%v,%v)", i, v)
            wg.Done()
        }(i, v)
    }
    wg.Wait()
}
// 输出如下: 随着 goroutine 执行完成顺序不同,输出顺序也会发生改变
(3,3)(0,0)(1,1)(2,2)
```

可以看到新程序的结果符合预期.其实质是在迭代过程中,向函数传参时, `i,v` 的值已经传递给 `a,b`(值拷贝),因此输出的值也是遍历之后的值

## `defer` 陷阱

先来看一下如下几个函数的执行结果:

```go
import "fmt"

func f1() (r int) {
    defer func() {
        r++
    }()
    return 2
}

func f2() (r int) {
    t := 5
    defer func() {
        t += 5
    }()
    return t
}

func f3() (r int) {
    defer func(r int) {
        r += 5
    }(r)
    return 1
}

func main() {
    fmt.Printf("f1=%v,f2=%v,f3=%v", f1(), f2(), f3())
}
// 输出如下:
f1=3,f2=5,f3=1
```

这个理解起来可能有些难度,我们逐个进行分析

首先,我们要明白在以上函数中, `r` 作为返回值,在函数定义时,已经被定义并赋值为 `return` 关键字返回的值.因此

1. 在 `f1` 中, `r` 的初始值为 2.在函数返回之前 `defer` 修饰的闭包函数对 `r` 的值做了修改.通过本篇文章中第一个示例可以看出,闭包函数中的操作会影响到函数的返回值.因此 `f1` 在返回之前会执行闭包函数而修改 `r` 的值(自增).因此返回 3
2. 在 `f2` 中, `r` 的初始值为变量 `t` 的值(变量 `t` 做值拷贝后将值传给 `r`), 为 5.在函数返回之前 `defer` 修饰的闭包函数对 `t` 的值做了修改,而 `r` 的值没有受到影响.因此 `f2` 的返回值是 5
3. 在 `f3` 中, `r` 的初始值为 1.在函数返回之前, 将 `r` 做值拷贝后将值作为参数传递给 `defer` 修饰的函数(非闭包),`defer` 修饰的函数中的操作对外部参数 `r` 没有影响.因此 `f3` 的返回值是 1

其实以上主要是闭包函数在使用过程中可能忽略的陷阱,而我们要始终知道的是 **Go 中所有的变量赋值及参数传递均为值拷贝,变量值可能是具体的一个对象,也可能是指向某一对象的内存地址.而闭包不会进行参数传递,闭包函数内与外的变量都使用同一内存地址**

## 切片

## 创建方式与底层数据结构

切片的创建方式如下:

- 通过数组创建
- 通过内置的 make 函数创建
- 直接声明

```go
arr := [5]int{0, 1, 2, 3, 4}
s1 := arr[:]
s2 := make([]int, 3, 5)
s3 := []int{0, 1, 2}
var s4 []int  // s4 = nil
```

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

不管是以哪种类型创建,其数据的底层存储都是数组.且由 `runtime/slice.go` 可以看出切片的数据结构有 3 个成员,分别是指向底层数组的指针,切片的当前大小和底层数组的大小.当 len 增长超过 cap 时,会申请一个更大容量的底层数组,并将数据复制过来

需要注意的是 `var s = make([]int, 0)` 与 `var s []int` 创建的对象是有区别的,前者会对底层数组进行内存分配,并初始化为没有值的切片,后者不会进行内存分配,其实为 nil.

### 多个切片引用同一底层数组引发的混乱

使用内置函数 `append` 扩展切片过程中可能会修改底层数组的元素,间接影响其它切片的值,也可能引发数组重建,可能会引发意想不到的错误

```go
import (
    "fmt"
    "reflect"
    "unsafe"
)

func main() {
    arr := [7]int{0, 1, 2, 3, 4, 5, 6}
    a := arr[:]
    b := arr[0:4]
    as := (*reflect.SliceHeader)(unsafe.Pointer(&a))
    bs := (*reflect.SliceHeader)(unsafe.Pointer(&b))

    fmt.Printf("a=%v,len=%d,cap=%d,pointer=%p, type=%d\n", a, len(a), cap(a), &a, as.Data)
    fmt.Printf("b=%v,len=%d,cap=%d,pointer=%p, type=%d\n", b, len(b), cap(b), &b, bs.Data)

    b = append(b, 10, 11, 12)
    fmt.Printf("arr=%v,len=%d,cap=%d,pointer=%p\n", arr, len(arr), cap(arr))
    fmt.Printf("a=%v,len=%d,cap=%d,pointer=%p, type=%d\n", a, len(a), cap(a), &a, as.Data)
    fmt.Printf("b=%v,len=%d,cap=%d,pointer=%p, type=%d\n", b, len(b), cap(b), &b, bs.Data)

    b = append(b, 13, 14)
    fmt.Printf("a=%v,len=%d,cap=%d,pointer=%p, type=%d\n", a, len(a), cap(a), &a, as.Data)
    fmt.Printf("b=%v,len=%d,cap=%d,pointer=%p, type=%d\n", b, len(b), cap(b), &b, bs.Data)
}
// 输出如下
// arr=[0 1 2 3 4 5 6],len=7,cap=7,pointer=0xc00000e280
// a=[0 1 2 3 4 5 6],len=7,cap=7,pointer=0xc0000044a0, type=824633778816
// b=[0 1 2 3],len=4,cap=7,pointer=0xc0000044c0, type=824633778816
// arr=[0 1 2 3 10 11 12],len=7,cap=7,pointer=0xc00000e280
// a=[0 1 2 3 10 11 12],len=7,cap=7,pointer=0xc0000044a0, type=824633778816
// b=[0 1 2 3 10 11 12],len=7,cap=7,pointer=0xc0000044c0, type=824633778816
// arr=[0 1 2 3 10 11 12],len=7,cap=7,pointer=0xc00000e280
// a=[0 1 2 3 10 11 12],len=7,cap=7,pointer=0xc0000044a0, type=824633778816
// b=[0 1 2 3 10 11 12 13 14],len=9,cap=14,pointer=0xc0000044c0, type=824633786816
```

由输出可以看出:

- 在第一次调用 `append` 函数向切片 b 追加元素后,底层数组 arr 的值发生改变.从而导致切片 a 的值也发生改变
- 第二次调用 `append` 函数向切片 b 追加元素后,由于 `len(b) > cap(b)`,底层数组重新分配内存空间,产生更大容量的数组结构,并将原来数组值复制到新数组

## 传值还是传引用

Go 中所有的变量赋值及参数传递均为值拷贝.

- 当变量赋值或参数传递为指针时,同样是值拷贝,但是指针及其副本指向的地址是同一个地址,因此操作的是同一数据
- 当变量赋值或参数传递为引用类型数据时(`chan,map,slice`),其内部都是通过指针指向具体的数据,实际上相当于传递了指针的副本
