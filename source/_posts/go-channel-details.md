---
title: go channel 详解
date: 2021/03/11
tags:
  - go
  - channel
  - 面试
  - 源码阅读
categories:
  - go
abbrlink: 
description: 本文章主要包含 go channel 使用的注意事项及 channel 实现的源码阅读.
---

## channel 介绍

channel 通常与 goroutine 一起使用,是 goroutine 之间通信的一种方式

### 创建并初始化

channel 的创建使用内置的 `make` 函数创建并初始化,初始化方式如下:

```go
// 初始化缓冲区容量为 cap 的 channel, 用于传递 Type 类型的对象.其中 cap 可省略,表示没有缓冲区
var chName = make(chan <chanType>, [cap])

// 如创建并初始化 struct{} 类型的 channel
// ch := make(chan struct{}, 1)

var chNil chan <chanType>
// 因为还没有进行初始化,此时创建的 chNil 为 nil.
fmt.Println(chNil == nil) // true
```

### 类型

根据 channel 创建时 `cap` 值是否为 0,channel 可分为不带缓冲区的 channel 和带缓冲区的 channel.

对无缓冲区的 channel 发送/接收数据时,无缓冲区的 channel 会发送阻塞.当程序中只有对该 channel 发送或接收数据操作时,该程序会发生死锁.因此在程序中使用无缓冲区的 channel 时,必须同时存在发送和接收操作(可以在 goroutine 中实现),所以无缓冲区的 channel 又被称为同步 channel.

有缓冲区的 channel 类似于一个队列.当缓冲区未满时,向 channel 中发送数据不会阻塞.当缓冲区满时,发送数据操作将被阻塞,直到有其它 goroutine 从中读取消息.

### 关闭 channel

go 提供了内置的 `close` 函数对 channel 进行关闭操作.

```go
ch := make(chan int)
close(ch)
```

有关 channel 的关闭,有如下注意事项:

- 关闭未初始化的 channel (nil) 会产生 panic
- 关闭已关闭的 channel 会产生 panic
- 向已关闭的 channel 中发送数据会产生 panic
- 从一个已关闭的 channel 中读取消息不会产生 panic 和阻塞.如果已关闭的 channel 的缓冲区还有数据,则可以正常读取,并返回值为 `true` 的 ok-idiom.否则,返回 channel 的默认值和 false 的 ok-idiom.
- 要将已关闭的 channel 中的数据全部读取出来,可以使用 for-range 方式进行读取.遍历结束后,channel 缓冲区数据为空.此时再进行读取,会返回 channel 的默认值和 false 的 ok-idiom.

对已经关闭的 channel 进行遍历如下:

```go
import "fmt"

func main() {
    ch := make(chan int, 1)
    ch <- 1
    close(ch)
    for value := range ch {
        fmt.Println(value)
    }
    val, ok := <-ch
    fmt.Println(val, ok)
}
```

### 用法

#### goroutine 通信

看一个 effective go 中的例子:

```go
c := make(chan int)  // Allocate a channel.

// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()

doSomethingForAWhile()
<-c
```

主 goroutine 会阻塞,直到执行 sort 的 goroutine 完成.

#### 配合 select 形成多路复用

select 可以同时监听多个 channel 的消息状态.

当其中一个 `case` 语句非阻塞,则执行对应 `case` 语句中的内容.若有多个 `case` 语句非阻塞,则随机挑选一个 `case` 语句中的内容执行.

若所有 `case` 语句均处于阻塞状态且定义了 `default` 语句,则执行 `default` 语句中的内容.否则,一直阻塞

```go
import (
    "fmt"
    "sync"
)
func main() {
    var wg sync.WaitGroup
    wg.Add(2)
    ach, bch := make(chan int), make(chan int)

    // 消费者 goroutine
    go func(wg *sync.WaitGroup, a, b <-chan int) {
        defer wg.Done()
        var (
            name string
            x    int
            ok   bool
        )
        for {
            select {
            case x, ok = <-a:
                name = "a"
            case x, ok = <-b:
                name = "b"
            }
            if !ok {
                // 如果没有数据发送,则跳出循环
                return
            }
            fmt.Println(name, x)
        }
    }(&wg, ach, bch)

    // 生产者 goroutine
    go func(wg *sync.WaitGroup, a, b chan<- int) {
        defer wg.Done()
        defer close(a)
        defer close(b)
        for i := 0; i < 10; i++ {
            select {
            case a <- i:
            case b <- i * 10:
            }
        }
    }(&wg, ach, bch)

    wg.Wait()
}
```

上述代码,分别创建了生产者和消费者的 goroutine,生产者会随机从 a b channel 中随机挑选一个发送消息,而消费者使用一个 for 循环来监控 a b channel,当 a b 其中一个接收到数据时,则指定对应内容.如果没有数据,则跳出循环,结束 goroutine.

#### 应用示例

- 设置超时时间

```go
ch := make(chan struct{})

// finish task while send msg to ch
go doTask(ch)

// 程序会在 5 s 内超时自动退出
timeout := time.After(5 * time.Second)
select {
    case <- ch:
        fmt.Println("task finished.")
    case <- timeout:
        fmt.Println("task timeout.")
}
```

- 设置退出信号

```go
msgCh := make(chan struct{})
quitCh := make(chan struct{})
for {
    select {
    case <- msgCh:
        doSomeWork()
    case <- quitCh:
        finish()
        return
}
```

- 按指定顺序在 goroutine 中循环打印内容

如下是启动了 3 个 goroutine 按顺序分别打印不同的内容,每个 goroutine 打印 n 次的代码实现.

```go
import (
    "fmt"
    "sync"
    "sync/atomic"
)

func f(wg *sync.WaitGroup, counter, n int64, in <-chan bool, out chan<- bool, message string) {
    for {
        // if 必须在 <- in 之前进行判断,否则 return 时会发生死锁
        if counter >= n {
            wg.Done()
            return // 别忘了 return
        }
        x := <-in // 使用 x 作为启动函数的标志,信号进入到此函数
        fmt.Println(message)
        atomic.AddInt64(&counter, 1) // 原子操作,协程数据安全
        out <- x                     // 信号离开此函数,并进入到下一个函数
    }
}

func main() {
    var (
        dogCh   = make(chan bool, 1) // 必须定义为缓冲区为 1 的 channel,用于保存启动信号/标志
        catCh   = make(chan bool, 1)
        fishCh  = make(chan bool, 1)
        wg      sync.WaitGroup
        n       int64 = 3
        counter int64 = 0
    )
    dogCh <- true //启动的信号
    wg.Add(3)
    go f(&wg, 0, n, dogCh, catCh, "dog")
    go f(&wg, 0, n, catCh, fishCh, "cat")
    go f(&wg, 0, n, fishCh, dogCh, "fish")
    wg.Wait()
}
```

## channel 源码分析

channel 的实现主要在 `${GOROOT}/src/runtime/chan.go` 中.该文件包含 channel 的整个生命周期,包含 channel 的结构体定义,初始化函数,发送/接收数据函数以及 channel 的关闭函数.

主要可以概括为以下内容:

```text
# 1. 初始化
make(chann interface{}, size)  =>  runtime.makechan(interface{}, size)
make(chann interface{})        =>  runtime.makechan(interface{}, 0)

# 2. 发送数据
ch <- v  =>  runtime.chansend1(ch, &v)

# 3. 接收数据
v <- ch      =>  runtime.chanrecv1(ch, &v)
v, ok <- ch  =>  runtime.chanrecv2(ch, &v)

# 4. 关闭 channel
close(ch)    =>  runtime.closechan(ch)
```

### hchan 结构体

channel 的结构体定义为 `hchan`,其定义如下:

```go
type hchan struct {
    qcount   uint           // total data in the queue 队列中所有数据的总数
    dataqsiz uint           // size of the circular queue 环形队列的大小,由 make 初始化时的 size 决定
    buf      unsafe.Pointer // points to an array of dataqsiz elements 指向大小为 dataqsiz 数组的指针
    elemsize uint16 // 元素大小
    closed   uint32 // 是否关闭
    elemtype *_type // element type 元素数据类型,由 make 初始化时的 元素类型决定
    sendx    uint   // send index 发送索引
    recvx    uint   // receive index 接收索引
    recvq    waitq  // list of recv waiters  recv 等待列表,即 <-ch
    sendq    waitq  // list of send waiters  send 等待列表,即 ch<-

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    // Trans: lock 保护了 hchan 的所有字段,以及在此 channel 上阻塞的 sudog 的一些字段
    // 当持有此锁时不应改变其它 G 的状态,因为它在栈收缩时会发生死锁
    lock mutex  // 锁
}

// 可以理解为由封装了 goroutine 的 sudog 组成的双向环形链表,
type waitq struct {
    first *sudog
    last  *sudog
}

// ${GOROOT}/src/runtime/runtime2.go
// 双向环形链表的元素结构体,内部封装了 goroutine 的指针
type sudog struct {
    // The following fields are protected by the hchan.lock of the
    // channel this sudog is blocking on. shrinkstack depends on
    // this for sudogs involved in channel ops. 
    g *g    // goroutine 的指针
    next *sudog  // 链表的下一个节点
    prev *sudog  // 前一个节点
    elem unsafe.Pointer // 数据元素 data element (may point to stack) 
    
    // The following fields are never accessed concurrently.
    // For channels, waitlink is only accessed by g.
    // For semaphores, all fields (including the ones above)
    // are only accessed when holding a semaRoot lock.  
    acquiretime int64
    releasetime int64
    ticket      uint32  
    // isSelect indicates g is participating in a select, so
    // g.selectDone must be CAS'd to win the wake-up race.
    isSelect bool   
    parent   *sudog // semaRoot binary tree
    waitlink *sudog // g.waiting list or semaRoot
    waittail *sudog // semaRoot
    c        *hchan // channel
}
```

通过以上结构体定义可以了解到 channel 内部的主要实现:

- 一个环形数组实现的队列,用于存储消息元素.其中涉及的属性包括队列指针,队列容量,队列中元素类型,元素大小,队元素个数
- 两个元素为 `sudog` 的双向链表,`recvq` 和 `sendq`,表示接收/发送数据的等待队列.其中 `sudog` 封装了 goroutine 的指针及要传输的数据元素.
- 一个互斥锁,用于 channel 各个属性变动的同步
- 一个用于判断 channel 是否关闭的标志位 closed

![hchan 结构可视化](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/go-channel-details/hchan-structure-visualization.png)

### channel 初始化

```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // 省略部分代码

    // 根据 make 时指定元素类型及容量计算环形队列 buf 所需内存
    mem, overflow := math.MulUintptr(elem.size, uintptr(size))

    var c *hchan
    switch {
    case mem == 0:
        // 环形队列容量大小为 0, make(chan interface{})
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        c.buf = c.raceaddr()
    case elem.ptrdata == 0:
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        // 元素不包含指针时,分配连续的内存
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true)) // 为 channel 和 环形队列分配连续的内存
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // Elements contain pointers.
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

    // 传入相关属性值
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    lockInit(&c.lock, lockRankHchan)

    // 省略部分代码

    return c
}
```

### 发送数据

从以下代码可以看出, send 分为以下 3 种情况:

1. 有 receiver 的 goroutine 阻塞在 channel 的 recvq 队列上,此时 channel 的缓冲队列为空.若有数据发送,则直接将数据发送给 receiver 的 goroutine.Go 在此处做了优化,数据只产生一次复制.
2. 当 channel 缓冲队列仍有剩余空间时,会将数据放到缓冲队列里,等待 receiver 的 goroutine接收.
3. 当 channel 缓存队列已满时,将当前 goroutine 加入 sendq 队列并阻塞.

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 判断 c 是否为空,若为空,则执行 gopark,会进行锁定
    if c == nil {
        if !block {
            return false
        }
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    // fastpath: 快速检测 channel 是否关闭或其元素已满
    if !block && c.closed == 0 && full(c) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    // 锁定
    lock(&c.lock)
    // fastpath: 再次进行检测
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    // case1: 如果 channel 的接收队列中已经有 goroutine 等待接收,则直接发送到 receiver
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // case2: 如果队列未满,则将其移动到队列中,并使 send 索引和队列中元素数加1
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    if !block {
        unlock(&c.lock)
        return false
    }

    // case3: 缓存队列已满,将 goroutine 加入缓存队列

    // Block on the channel. Some receiver will complete our operation for us.
    gp := getg()
    // 获取 sudog.这里点进去会发现,获取 sudog 的方式为 优先从全局拿,数量为本地缓存队列空间的一半.如果没有,则创建新的 sudog
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    // 将 sudog 添加到 队列中
    c.sendq.enqueue(mysg)
    // Signal to anyone trying to shrink our stack that we're about
    // to park on a channel. The window between when this G's status
    // changes and when we set gp.activeStackChans is not safe for
    // stack shrinking.
    atomic.Store8(&gp.parkingOnChan, 1)
    // 阻塞,等待重新被调度后继续从此位置开始执行
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    // Ensure the value being sent is kept alive until the
    // receiver copies it out. The sudog has a pointer to the
    // stack object, but sudogs aren't considered as roots of the
    // stack tracer.
    KeepAlive(ep)

    // someone woke us up.
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    gp.activeStackChans = false
    if gp.param == nil {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    // 释放 sudog
    releaseSudog(mysg)
    return true
}
```

### 接收数据

同样的,我们再来看接收数据.从以下代码可以看出, recv 分为以下 3 种情况:

1. 有 sender 的 goroutine 阻塞在 channel 的 recvq 队列上.
   1. 若 channel 的缓冲队列不存在,则直接从 sender 的 goroutine 接收数据.
   2. 若 channel 的缓冲队列已满,则从 channel 的缓冲区队列头部获取数据,并复制发送者数据到缓冲队列中,并使得 send/recv 索引自增.
   3. 数据接收完成后释放锁,并使得 sender 的 goroutine 处于 ready 状态
2. 当 channel 缓冲队列仍有数据时,会直接将数据接收,并使得 recv 索引自增,qcount 元素个数自减
3. 当 channel 缓存队列没有数据时,将 receiver 的 goroutine 加入 recvq 队列，并阻塞.

```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: don't need to check ep, as it is always on the stack
    // or is new memory allocated by reflect.

    if debugChan {
        print("chanrecv: chan=", c, "\n")
    }

    // fastpath: 快速检测 channel 是否为 nil
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    // Fast path: 快速检查失败的非阻塞操作
    if !block && empty(c) {
        // channel 是否已经被关闭
        if atomic.Load(&c.closed) == 0 {
            return
        }
        // 快速检查是否有数据
        if empty(c) {
            // The channel is irreversibly closed and empty.
            if raceenabled {
                raceacquire(c.raceaddr())
            }
            if ep != nil {
                typedmemclr(c.elemtype, ep)
            }
            return true, false
        }
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    // 加锁
    lock(&c.lock)

    // 再次进行检查
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }

    // case1: 如果 channel 的发送列中已经有 sender 的 goroutine 等待发送,则
    // 当没有缓冲队列时(c.dataqsiz == 0),直接从等待的 goroutine 中接收数据
    // 当缓冲队列已满,则从 channel 的缓冲区队列头部获取数据,并复制发送者数据到缓冲队列中,并使得 send/recv 索引自增.
    // 最终解锁并使得  sender 的 goroutine 处于 ready 状态
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    // case2: channel 缓冲队列不为空,则直接从缓冲队列中获取数据,并使得 recv 索引自增,qcount 元素个数自减
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    if !block {
        unlock(&c.lock)
        return false, false
    }

    // no sender available: block on this channel.
    // case3: 缓存队列为空,将 receiver 的 goroutine 加入 recvq 队列，并阻塞.细节基本与发送数据的 case3 一致
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg)
    // Signal to anyone trying to shrink our stack that we're about
    // to park on a channel. The window between when this G's status
    // changes and when we set gp.activeStackChans is not safe for
    // stack shrinking.
    atomic.Store8(&gp.parkingOnChan, 1)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

    // someone woke us up
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    gp.activeStackChans = false
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, !closed
}
```

### 关闭

从以下代码片段可以看出,关闭 channel 时做了如下事情:

- channel 的 close 标志位置为 1,表示关闭状态
- 创建 glist,用于保存 recvq 和 sendq 中的 goroutine
- 保存完成后,释放锁.(这里先释放锁,避免使 glist 中 goroutine 处于 ready 状态时锁的消耗)
- 使得 glist 中的 goroutine 处于 ready 状态

```go
func closechan(c *hchan) {
    // fastpaht: 关闭 nil 的 channel 引发 panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }
    // 加锁后判断,关闭已关闭 channel 引发 panic
    lock(&c.lock)
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    if raceenabled {
        callerpc := getcallerpc()
        racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
        racerelease(c.raceaddr())
    }

    c.closed = 1

    var glist gList

    // release all readers
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp)
    }

    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, c.raceaddr())
        }
        glist.push(gp)
    }
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```

## 参考

- [由浅入深剖析 go channel](https://www.jianshu.com/p/24ede9e90490)
- [Go 夜读 - #56 channel & select 源码分析](https://www.bilibili.com/video/BV1g4411R7p5)
- [Go 夜读 - 第 56 期 channel & select 源码分析](https://github.com/talkgo/night/issues/450)
