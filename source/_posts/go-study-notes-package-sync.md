---
title: go 学习笔记之 sync 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 41417
description: >-
  本文章主要包含 Go sync 包及其内置类型和方法的使用.sync 包提供基本的同步原语,如互斥锁.除 Once 和 WaitGroup
  类型外,大多数都共低级库使用.更高级的同步最好通过管道和通信来实现
---

`sync` 包提供了基础的同步方法和锁机制.导入方式为 `import "sync"`

# 常用类型定义

```go
// Locker 接口,表示可以解锁和加锁的对象
type Locker interface {
    Lock()
    Unlock()
}

// 互斥锁
// 互斥锁与 goroutine 没有关联,允许一个 goroutine 添加锁,另一个 goroutine 释放锁
type Mutex struct {
    // 内含隐藏或非导出字段
}

// 仅执行一次动作
type Once struct {
    // 内含隐藏或非导出字段
}

// 资源池对象
// 池中的任何对象都可能被随时删除,并且没有通知.如果池中的唯一对象被删除,则该资源池对象也会被释放
// 资源池对象中的实例可被多个 goroutine 安全使用
type Pool struct {

    // 可以指定一个函数,用于为 Get 生成一个对象.否则 Get 返回 nil
    New func() interface{}
    // 内含隐藏或非导出字段
}

// 等待 goroutine 运行结束
// 主线程调用 Add 方法设置要等待的 goroutine 数量,每个 goroutine 在运行完成后调用 Done.同时,使用 Wait 阻塞主函数,直到所有 goroutine 完成
type WaitGroup struct {
    // contains filtered or unexported fields
}
```

# 常用函数

## `Mutex` 结构体方法

互斥锁与 goroutine 没有关联,允许一个 goroutine 添加锁,另一个 goroutine 释放锁

```go
// 实例声明方式 `var m sync.Mutex`

// 加锁.如果锁已在使用中,则阻塞,直到互斥锁可用
func (m *Mutex) Lock()
// 释放锁.如果未加锁,则会出现运行时错误
func (m *Mutex) Unlock()
```

## `Once` 结构体方法

```go
// 实例声明 `var o sync.Once`

// 当且仅当 o 第一次调用 Do 方法时,调用函数 f.
func (o *Once) Do(f func())
```

## `Pool` 结构体方法

```go
// 实例声明方式 `var p sync.Pool` 或 `var bufPool sync.Pool = sync.Pool{New: funName}`

// 从池中任意取出一个实例,并返回.如果资源池为空,且 p.New 不为空时,Get 返回调用 p.New 的结果
func (p *Pool) Get() interface{}
// 将 x 放入 p 资源池中
func (p *Pool) Put(x interface{})
```

## `WaitGroup` 结构体方法

```go
// 实例声明 `var wg sync.WaitGroup`

// 增加可能为负数的 delta 到计数器中.
// 如果计数器为 0,则释放调用 Wait 处于阻塞状态的所有 goroutine
// 如果计数器变为负数,发生错误
func (wg *WaitGroup) Add(delta int)
// 使用 wg 计数器减 1
func (wg *WaitGroup) Done()
// 阻塞当前 goroutine 直到 wg 计数器为 0
func (wg *WaitGroup) Wait()
```

# 示例

## `Once` 使用示例

```go
import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    for i := 0; i < 5; i++ {
        once.Do(func() {
            fmt.Printf("在 i=%v 时被调用\n", i)
        })
    }
}
// 输出如下. 可看到匿名函数仅在第一次调用 `once.Do()` 时执行一次,以后的循环不再执行
// 在 i=0 时被调用
```

## `Pool` 使用示例

- 使用 `Pool` 返回 5 个随机数

```go
import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    var p sync.Pool = sync.Pool{
        New: func() interface{} {
            return rand.Intn(100)
        },
    }
    for i := 0; i < 5; i++ {
        fmt.Println(p.Get())
    }
}
```

## `WaitGroup` 使用示例

- 使用 `WaitGroup` 等待所有 goroutine 执行结束

```go
import (
    "fmt"
    "sync"
)

func main() {

    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            fmt.Printf("goroutine %v done\n", i)
        }(i)
    }
    wg.Wait()
    fmt.Println("main goroutine done")
}
```