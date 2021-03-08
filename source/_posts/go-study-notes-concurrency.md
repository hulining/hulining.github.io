---
title: go 学习笔记之并发
date: 2020/04/30
tags:
  - go
categories:
  - go
abbrlink: 9125
description: >-
  记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中并发或 goroutine 协程的实现, 通过 sync.WaitGroup
  等待协程执行完毕, 简单描述了 channel 的使用
---

## 并发与并行

首先理解一下并发(concurrency) 与并行(parallesim)

- 并发: 在一段时间内交替做不同事情的能力, 可以理解为单线程(协程)/多线程运行在单核处理器上, 如果有其中一个任务/线程阻塞, CPU 立即切换, 执行另一个任务/线程的代码逻辑
- 并行: 在同一时刻做不同事情的能力, 可以理解为多线程运行在多核处理器上, 一个线程绑定一个 CPU, 多个 CPU 同时处理代码逻辑

我们通常所说的程序是并发设计的, 允许多个任务同时执行. 但实际上,在单核处理器上, 某一时刻, 一个 CPU 只能处理一个任务. 而多个任务是以切换方式进行的, 只不过切换时间非常短, 我们无法感知.

而并行依赖多核处理器, 让多个任务真正在同一时刻进行. 因此多线程或多进程是并行的基本条件.

## 协程

Go 语言中采用 `goroutine` 来处理并发任务, `goroutine` 是建立在线程之上的轻量级抽象, 它允许我们以非常低的代价在同一地址空间中并行地执行多个函数或方法. 相比于线程, 它的创建和销毁的代价要小很多. `goroutine` 所需要的内存通常只有 2KB, 线程所需要的内存默认 MB 级别

在 Go 中创建一个 `goroutine` 非常简单, 在函数调用前加 `go` 关键字即可创建并发任务.

```go
go func()  // 创建一个 goroutine
```

需要注意的是, `go` 关键字并非执行并发操作, 而是创建一个并发任务单元. 新建的任务被放置在系统队列中, 等待调度器安排合适的系统线程执行. 当前任务流程不会阻塞, 不会等待该任务启动或结束, 且运行时也不保证并发任务的执行次序.

与 `defer` 定义的延迟调用函数一样, `go` 定义的 `goroutine` 函数也会立即计算并记录当时上下文中参数对象的状态, 并复制参数对象用于真正调用时隐式传入.

```go
import (
    "fmt"
    "time"
)

var c int

func counter() int {
    c++
    return c
}

func main() {
    a := 100
    fmt.Printf("main: %p,%v\n", &a, a)
    go func(x, y int) {
        time.Sleep(time.Second)                // 让 goroutine 在 mian 逻辑之后执行
        fmt.Printf("go: %p,%v,%v\n", &x, x, y) // 立即计算并复制参数
    }(a, counter())

    a += 100
    fmt.Printf("main: %p,%v,%v\n", &a, a, counter())
    time.Sleep(time.Second * 3) // 等待 goroutine 结束
}
// 输出
// main: 0xc000062090,100
// main: 0xc000062090,200,2
// go: 0xc00000a038,100,1
```

以上代码只能通过 `time.sleep()` 的方式等待 `goroutine` 执行完毕, 我们不能判断 `goroutine` 中的任务何时执行结束, `main` 函数 `sleep` 的时间也就不能确定.

### 等待

为了解决以上问题,我们可以使用 `channel` 阻塞 `main` 函数,然后在 goroutine 中发出退出信号

```go
import (
    "fmt"
)

func main() {
    exit := make(chan struct{}) // 创建 channel
    go func() {
        time.Sleep(time.Second)
        fmt.Println("goroutine done")
        close(exit) // 关闭 channel, 发出信号
    }()
    fmt.Println("main...")
    val, ok := <-exit // 如果 channel 关闭,则可以解除阻塞
    fmt.Println(val, ok) // 但此时实际上是没有任何数据传入管道的,取出的数据为初始化时的定义类型的默认值,ok 为 false
    fmt.Println("main exit...")
}
```

如果要等待多个任务结束, 推荐使用 `sync.WaitGroup`. 通过设定计数器, 让每个 goroutine 在退出时递减, 直到归 0 时解除阻塞.

 ```go
import (
    "fmt"
    "time"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1) // 每次新创建一个 goroutine, 计数器加 1
        go func(id int) {
            defer wg.Done() // 每个 goroutine 执行完成后, 计数器减 1
            time.Sleep(time.Second)
            fmt.Printf("goroutine %v done\n", id)
        }(i)
    }

    fmt.Println("main...")
    wg.Wait() // 阻塞, 直到计数器归 0
    fmt.Println("main exit...")
}
```

`WaitGroup.Add` 实现了原子操作,但仍然建议在 `goroutine` 外累加计数器, 防止累加(Add)操作尚未执行, 阻塞(Wait)已经退出

## `channel`

### 定义

Go 鼓励使用 CSP(Communicating Sequential Process) channel, 以通信来代替内存共享,实现并发安全.

> Don't communicate by sharing memory, share memory by communicating. 不要以共享内存来进行通信, 而是通过通信来共享内存.

作为 CSP 核心, channel 是显示的, 要求操作双方必须知道数据类型和具体通道,但并不关心另一端操作者身份和数量. 可如果另一端未准备妥当或消息未及时处理时,会阻塞当前端.

channel 定义方式如下:

```go
var channelName = make(chan Type, [cap])  // 初始化缓冲区容量为 cap 的 channel, 用于传递 Type 类型的对象.其中 cap 可省略
```

### channel 注意事项

- 缓冲区大小仅是内部属性, 不属于类型组成部分. channel 本身就是指针, 可用相等操作符判断是否为同一对象或 nil
- 内置函数 `cap()` 和 `len()` 返回缓冲区大小和当前已缓冲数量.而对于同步通道(cap 为 0 的 channel)则都返回 0, 据此可判断通道是同步(0)还是异步(非0).
- 同步模式(cap 为 0 的 channel)必须有成对的发送/接收数据的 goroutine 出现, 否则会一直阻塞
- channel 支持使用 ok-idom 或 for-range 模式处理数据.其中 for-range 模式只能处理已关闭的具有缓冲区的 channel,否则会引发死锁 `fatal error: all goroutines are asleep - deadlock`.若 channel 已关闭,则 ok-idom 返回值为 `defaultValue, false`.可通过 ok 判断 channel 是否已经关闭,取值是否可用
- 向已关闭的 channel 发送数据,会引发 panic.

```go
import (
    "fmt"
    "sync"
)

func putNum(intChan chan int) {
    defer close(intChan)  // 关闭 intChan
    for i := 2; i <= 1000; i++ {
        intChan <- i  // 向 intChan 发送数据
    }
}

func isPrime(value int) bool {
    if value <= 3 {
        return value >= 2
    }
    if value%2 == 0 || value%3 == 0 {
        return false
    }
    for i := 5; i*i <= value; i += 6 {
        if value%i == 0 || value%(i+2) == 0 {
            return false
        }
    }
    return true
}

func getNum(intChan chan int, outChan chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for value := range intChan {  // 通过 for-range 遍历已关闭的 intChan 中的数据
        if isPrime(value) {
            outChan <- value  // 找到符合条件的数据,发送到 outChan
        }
    }
}

func main() {
    var inChannel = make(chan int, 10)  // 定义缓冲区为 10 的 channel
    var outChannel = make(chan int, 1000) // 定义缓冲区为 1000 的 channel
    var wg sync.WaitGroup

    go putNum(inChannel)    // 已经在 putNum 中关闭了 inChannel
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go getNum(inChannel, outChannel, &wg)
    }
    wg.Wait()
    close(outChannel)   // 关闭 outChannel,后面才可以使用 for-range 进行遍历
    for x := range outChannel {
        fmt.Println(x)
    }
}
```

### 有缓冲与无缓冲的通道

无缓冲的通道又称为阻塞通道或同步通道.先看一下下面的代码:

```go
func main() {
    ch := make(chan int)
    ch <- 10    // 在这里发生阻塞,等待通道的另一端进行接收
    fmt.Println("发送成功")
}
```

上面这段代码能够通过编译,但是执行的时候会出现以下错误:

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
...
```

原因是上述代码会阻塞在 `ch <- 10` 这一行阻塞,形成死锁.解决此问题的方式为通过 goroutine 接收值.如:

```go
func recv(c chan int) {
    ret := <-c
    fmt.Println("接收成功", ret)
}

func main() {
    ch := make(chan int)
    go recv(ch) // 启用 goroutine 从通道另一端接收值
    ch <- 10
    fmt.Println("发送成功")
}
```

解决上述问题的还有一种方法是创建有缓冲区的通道.我们可以在使用 `make` 函数初始化通道时指定通道的容量,如:

```go
func main() {
    ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
    ch <- 10
    fmt.Println("发送成功")
}
```

#### 单向 channel

channel 默认是双向的, 并不区分发送和接收端. 但我们可在定义时, 指定其为单向 channel, 且不能在单向 channel 上做逆向操作

```go
var send chan<- int = make(chan int, 10)
var recv <-chan int = make(chan int, 10)
// <-send // 报错, invalid operation: <-send (recevice from send-only type chan<- int), 无效操作
// recv <- 1
```

#### `select` 多路复用

如要同时处理多个 channel, 可选用 `select` 语句.

`select` 语句与 `switch` 语句类似, 它要求每个 case 必须是一个通信操作, 要么发送要么接收. 它会随机执行一个可运行的 case. 如果没有 case 可运行, 则会执行 default 或一直阻塞

```go
import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    wg.Add(2)
    a, b := make(chan int), make(chan int)

    go func() {
        defer wg.Done()
        for {
            var (
                name string
                x    int
                ok   bool
            )
            select {
            case x, ok = <-a:
                name = "a"
            case x, ok = <-b:
                name = "b"
            }
            if !ok {
                return
            }
            fmt.Println(name, x)
        }
    }()
    go func() {
        defer wg.Done()
        defer close(a)
        defer close(b)
        for i := 0; i < 10; i++ {
            select {
            case a <- i:
            case b <- i * 10:
            }
        }
    }()
    wg.Wait()
}
```

default 可用于 处理一些默认逻辑, 如添加新的缓存 channel 等.

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
```

#### channel 应用

channel 本身就是一个并发安全的队列, 可用作 ID 生成器, Pool 等用途

如下是 Pool 的简单实现:

```go
import (
    "fmt"
)

type pool chan []byte

func newPool(cap int) pool {
    return make(chan []byte, cap)
}

func (p pool) get() []byte {
    var bytes []byte
    select {
    case bytes = <-p:
        fmt.Println("获取成功并返回")
    default:
        fmt.Println("获取失败, 返回默认")
        bytes = make([]byte, 10)
    }
    return bytes
}
func (p pool) put(bytes []byte) {
    select {
    case p <- bytes:
        fmt.Println("放回成功")
    default:
        fmt.Println("放回失败")
    }
}
```

## 并发安全和锁

有时候在 Go 代码中可能会存在多个 goroutine 同时操作一个资源,这种情况会发生数据竞争问题.如

```go
var x int64
var wg sync.WaitGroup

func add() {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        x = x + 1
    }
}
func main() {
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go add()
    }
    wg.Wait()
    fmt.Println(x)
}
```

可以发现与预期结果不符,原因是两个 goroutine 在访问和修改 x 变量的时候会存在数据竞争.

未解决以上问题,Go 引入了锁的概念.Go 语言中使用 `sync.Mutex` 类型来实现互斥锁.使用互斥锁来修复上面代码的问题:

```go
var x int64
var wg sync.WaitGroup
var lock sync.Mutex

func add() {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        lock.Lock() // 加锁
        x = x + 1
        lock.Unlock() // 释放锁
    }
}
func main() {
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go add()
    }
    wg.Wait()
    fmt.Println(x)
}
```

加锁虽然保障了数据的准确性,但是加锁是非常消耗性能的一种做法.因此合理使用锁机制是 Go 语言中高性能并发的关键因素之一.

### 原子操作

加锁操作因为涉及内核态的上下文切换会比较耗时.针对基本数据类型我们还可以使用原子操作来保证并发安全.性能比加锁操作更好.

Go 语言中原子操作由内置的标准库 `sync/atomic` 提供.常用的方法如下:

```go
// 对 int32 或 int64 数据修改
func AddInt32(addr *int32, delta int32) (new int32)
func AddInt64(addr *int64, delta int64) (new int64)
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddUint64(addr *uint64, delta uint64) (new uint64)

// 将 int32 或 int64 数据写入到指定地址
func StoreInt32(addr *int32, val int32)
func StoreInt64(addr *int64, val int64)
func StoreUint32(addr *uint32, val uint32)
func StoreUint64(addr *uint64, val uint64)

// 读取指定地址的 int32 或 int64 数据
func LoadInt32(addr *int32) (val int32)
func LoadInt64(addr *int64) (val int64)
func LoadUint32(addr *uint32) (val uint32)
func LoadUint64(addr *uint64) (val uint64)

// 比较并交换.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
// 虽然不清楚什么意思.但是在 sync.waitgroup.go#func (wg *WaitGroup) Wait() 函数有用到.先记录一下
```

使用原子操作对上面变量 x 进行修改.代码如下:

```go
var x int64
var wg sync.WaitGroup
var lock sync.Mutex

func add() {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        atomic.AddInt64(&x, 1)
    }
}
func main() {
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go add()
    }
    wg.Wait()
    fmt.Println(x)
}
```
