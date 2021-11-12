---
title: ants 快速开始
date: 2021-11-11 17:06:35
tags:
  - go
  - 3rd
categories:
  - go
description: ants 实现了一个高性能的 goroutine 池,实现了对大规模 goroutine 的调度管理及复用,达到更高效执行任务的效果.
---

[`ants`](https://github.com/panjf2000/ants) 实现了一个高性能的 `goroutine` 池,实现了对大规模 goroutine 的调度管理及复用.它允许使用者在开发并发程序的时候限制 goroutine 数量,复用资源,以便达到更高效执行任务的效果

ants 有以下特性:

- 自动调度海量的 goroutine, 复用 goroutine
- 定期清理过期的 goroutine, 节省资源
- 提供了任务提交、获取运行中的 goroutine 数量、动态调整 Pool 大小、释放 Pool、重启 Pool 等常用 API
- 优雅处理 panic, 防止程序崩溃
- 在大规模批量并发任务场景下比原生 goroutine 并发具有[更高的性能](https://github.com/panjf2000/ants/blob/master/README_ZH.md#-性能小结)
- 非阻塞机制

可通过 [panjf2000/ants(github.com)](https://github.com/panjf2000/ants) 了解它是如何工作的

## 快速开始

我们可以通过 [panjf2000/ants(github.com)](https://github.com/panjf2000/ants)  中提供的一些示例来快速了解该库的使用方式

```go
package main

import (
	"fmt"
	"sync"
	"time"
	
	"github.com/panjf2000/ants/v2"
)

func main() {
	// 执行结束后,需要释放 ants goroutine 池
	defer ants.Release()
	runTimes := 1000
	var wg sync.WaitGroup
	doSomthing := func() {
		defer wg.Done()
		time.Sleep(10 * time.Millisecond)
		fmt.Println("Hello World!")
	}
	for i := 0; i < runTimes; i++ {
		wg.Add(1)
		// Use the default pool.
		_ = ants.Submit(doSomthing)
	}
	wg.Wait()
	fmt.Printf("running goroutines: %d\n", ants.Running())
	fmt.Printf("finish all tasks.\n")
}
```

如上示例中,使用包提供的默认的 goroutine 池提交任务.通过直接调用 `Submit()` 函数,并传入一个自定义 `func()` 作为参数来向 goroutine 池提交任务.

我们也可以通过 `ants.NewPool` 创建自定义大小 goroutine 池,同样通过调用其 `Submit()` 函数来提交任务.其实与上面代码一样

```go
func main() {
	// ... 省略部分代码
	// 创建一个自定义大小的 goroutine 池
	p, _ := ants.NewPool(10)
	defer p.Release()
	for i := 0; i < runTimes; i++ {
		wg.Add(1)
		_ = p.Submit(doSomthing)
	}
	wg.Wait()
	fmt.Printf("running goroutines: %d\n", p.Running())
}
```

假如 `doSomthing` 函数需要传入参数,又该怎么办呢? `ants` 包提供了 `NewPoolWithFunc(size int, pf func(interface{}), options ...Option) (*PoolWithFunc, error)` 来解决这个问题. 示例如下:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	
	"github.com/panjf2000/ants/v2"
)

var sum int32

func doSomthing(i interface{}) {
	n, _ := i.(int32)
	atomic.AddInt32(&sum, n)
	fmt.Printf("run with %d\n", sum)
}

func main() {
	var wg sync.WaitGroup
	runTimes := 1000
	// 创建一个带参数的 goroutine 池
	p, _ := ants.NewPoolWithFunc(10, func(i interface{}) {
		doSomthing(i)
		wg.Done()
	})
	defer p.Release()
	// 提交任务
	for i := 0; i < runTimes; i++ {
		wg.Add(1)
		_ = p.Invoke(int32(i))
	}
	wg.Wait()
	fmt.Printf("running goroutines: %d\n", p.Running())
}
```

同时, `NewPool` 与 `NewPoolWithFunc` 支持传入各种 `Option` 来实现不同的初始化属性.见下面的内容

## 常用函数

下面总结一下,`ants`包及其对象中常用的函数/方法.

### 

```go
// 创建 goroutine 池
// 创建一个指定大小的 goroutine 池
func NewPool(size int, options ...Option) (*Pool, error)
// 创建一个指定大小的 goroutine 池,支持通过 Invoke 传入参数
func NewPoolWithFunc(size int, pf func(interface{}), options ...Option) (*PoolWithFunc, error)


// 下面函数可以创建 Option 对象,传入到创建 goroutine 池的函数中,设置一些初始化属性
// 指定 logger
func WithLogger(logger Logger) Option
// 是否应该为 worker 动态分配内存空间
func WithPanicHandler(panicHandler func(interface{})) Option
// 指定发生 `panic` 时的处理函数
func WithPanicHandler(panicHandler func(interface{})) Option
// 当没有可用 worker 时, pool 返回 nil.非阻塞方式
func WithNonblocking(nonblocking bool) Option
// 设置清理 goroutine 的时间间隔
func WithExpiryDuration(expiryDuration time.Duration) Option

// Pool 对象与 PoolWithFunc 的一些方法,简单介绍
// 总容量,剩余容量,正在运行的 goroutine,是否已经关闭
// Cap(),Free(),Running(),IsClosed()

// 重启,释放,调整大小,
// Reboot(),Release(),Tune(size int)
// 向 Pool 提交无参数的任务
func (p *Pool) Submit(task func()) error
// PoolWithFunc 传入参数,调用其任务
func (p *PoolWithFunc) Invoke(args interface{}) error
```

---

参考:
- [panjf2000/ants (github.com)](https://github.com/panjf2000/ants)
- [ants - pkg.go.dev](https://pkg.go.dev/github.com/panjf2000/ants/v2)
- [GMP 并发调度器深度解析之手撸一个高性能 goroutine pool - Strike Freedom](https://strikefreedom.top/high-performance-implementation-of-goroutine-pool)

