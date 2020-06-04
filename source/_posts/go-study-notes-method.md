---
title: go 学习笔记之方法
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 403
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中结构体方法的定义及使用'
---

# 定义

方法是与对象实例绑定的特殊函数. 方法与函数定义的语法区别在于前者有前置实例接收参数, 编译器以此确定方法所属类型. 一般来说定义方式如下:

```go
type structName struct{
    fieldName fieldType
    // ...
}

func (receiver structName) MethodName(参数列表) (返回值列表){
    // func expression
}
```

特点和注意事项如下:

- 可以为当前包,已经除接口和指针以外的任何类型定义方法
- 方法被看作特殊的函数, 同样不支持重载, receiver 类型可以是基础类型或指针类型, 这关系到调用时对象实例是否被复制
- 可以使用实例值或指针值调用方法, 编译器会根据方法 receiver 类型自动在基础实例和指针类型间转换
- 对于结构体嵌套, 外层结构体可以直接重载或直接调用内层结构体的方法

示例代码如下:

```go
import (
    "fmt"
)

type N int
type S struct{
    N
}

func (n N) value() {
	n++
	fmt.Printf("%p, %v\n", &n, n)
}
func (n *N) pointer() {
	(*n)++
	fmt.Printf("%p, %v\n", n, *n)
}

func main() {
	var n N = 20
	n.value()                     // 0xc00000a0d0, 21
	fmt.Printf("%p, %v\n", &n, n) // 0xc00000a0b8 ,20
	n.pointer()                   // 0xc00000a0b8,21
	fmt.Printf("%p, %v\n", &n, n) // 0xc00000a0b8, 21
    
	s := S{23000}
	s.value()                     // 0xc00000a120, 23001
	fmt.Printf("%p, %v\n", &s, s) // 0xc00000a118, {23000}
	s.pointer()                   // 0xc00000a118, 23001
	fmt.Printf("%p, %v\n", &s, s) // 0xc00000a118, {23001}
}
```

> 面向对象的三大特征 "封装", "继承" "多态", Go 仅实现了部分特征, 它更倾向于 "组合优于继承" 这种思想. 将模块分解成相互独立的更小单元, 分别处理不同方面的需求, 最后以匿名嵌入方式组合到一起, 共同实现对外接口. 而其简短一致的调用方式, 更是隐藏了内部实现细节.

## 方法转化为表达式

方法和函数一样, 除直接调用外, 还可以作为参数传递, 依照具体引用方式不同, 可以分为 expression 和 value 两种状态.

## 表达式类型

通过类型引用的 method expression 会被还原为普通函数样式, receiver 是第一参数, 需要显示传参. 至于类型, 可以是 T 或 *T.

```go
import (
    "fmt"
)

type N int

func (n N) test() {
    fmt.Printf("test.n: %p, %v\n", &n, n)
}

func main() {
    var n N = 25
    fmt.Printf("main.n: %p, %v\n", &n, n)
    f1 := N.test        // 也可以直接调用 N.test(n)
    f1(n)
    f2 := (*N).test     // 也可以直接调用 (*N).test(&n)
    f2(&n)
}
// 输出结果
// main.n: 0xc00000a0b8, 25
// test.n: 0xc00000a0f0, 25
// test.n: 0xc00000a100, 25
```
## 变量类型

- 基于实例或指针引用的 method value, 依旧按照正常方式调用. 但当 method value 被赋值给变量或作为参数传递时, 会立即计算并**复制**该方法执行所需的 receiver 对象, 并与其绑定. 在后续调用时, 能够隐式传入 receiver 对象
- 如果目标方法的 receiver 是指针类型, 那么被复制的仅是指针. 会在调用时寻找该指针指向的对象, 所以传入的对象参数为调用时的上下文中的对象

```go
import (
    "fmt"
)

type N int

func (n N) value() {
	fmt.Printf("test.n: %p, %v\n", &n, n)
}
func (n *n) pointer(){
    fmt.Printf("test.n: %p, %v\n", &n, n)
}

func main() {
	var n N = 100
	p := &n
	n++  // 101
	f1 := n.value       // 记录此时上下文中变量 n 的状态,并以此状态复制后一个新的对象, 将该对象隐式传入到 test 函数
    f3 := n.pointer     // 记录并复制此时上下文中变量 n 的指针 &n
	n++  // 102
	f2 := p.value       // 记录此时上下文中指针 p 指向的内存地址对象, 并以此地址状态复制后一个新的内存地址对象, 将该对象隐式传入到 test 函数
    f4 := m.pointer     // 记录并复制此时上下文中变量 n 的指针 &n
	n++  // 103
	fmt.Printf("main.n: %p, %v\n", &n, n)
	f1()
    f3()    // 达到延迟调用的效果 *n = 103
	f2()
    f4()    // 达到延迟调用的效果 *n = 103
}
// 输出
// main.n: 0xc00000a0b8, 103
// test.n: 0xc00000a100, 101
// test.n: 0xc00000a0b8, 103
// test.n: 0xc00000a118, 102
// test.n: 0xc00000a0b8, 103
```