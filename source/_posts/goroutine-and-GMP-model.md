---
title: goroutine 与 GMP 模型介绍
date: 2021/03/09
tags:
  - go
  - goroutine
  - GMP
categories:
  - go
abbrlink: 
description: 该文章主要介绍 goroutine 调度器过程及原理,可以对 goroutine 中 GMP 模型有一个简单的认识
---

## goroutine

Go 为了提供更容易使用的并发方法,使用了 goroutine 和 channel.goroutine 来自协程的概念,让一组可复用的函数运行在一组线程之上,即使有协程阻塞,该线程的其他协程也可以被 `runtime` 调度,转移到其他可运行的线程上.

Go 中,协程被称为 goroutine,它非常轻量,一个 goroutine 只占几 KB,这就能在有限的内存空间内支持大量的 goroutine,支持了更多的并发.虽然一个 goroutine 的栈只有几 KB,但实际上是可伸缩的,如果需要更多资源,`runtime` 会自动为 goroutine 分配.

goroutine 具有占用内存小,调度灵活的特点:

- 占用内存更小(几 kb)
- 调度更灵活(`runtime` 调度)

## GMP 模型

- G: goroutine 协程
- M: thread 线程
- P: Processor,包含了运行 goroutine 的资源.包含了可运行的 G 队列.如果线程想要运行 goroutine,则必须先获取 P.

![GMP 模型](/images/gmp-model.png)

GMP 模型中包含以下概念:

- 全局队列(Global Queue):存放等待运行的 G.
- P 的本地队列: 同全局队列类似,存放的也是等待运行的 G,存的数量有限,一般不超过256个.新建 G' 时,G' 优先加入到 P 的本地队列.如果队列满了,则会把本地队列中一半的 G 移动到全局队列.
- P 的列表: 所有的 P 都在程序启动时创建,并保存在数组中,最多有 `GOMAXPROCS` 个.可通过环境变量 `GOMAXPROCS` 或 `runtime.GOMAXPROCS()` 进行修改.
- M: 线程想运行任务就得获取 P,从 P 的本地队列中获取 G.当 P 的本地队列为空时,M 也会尝试从全局队列拿一批 G 放到 P 的本地队列,或从其他 P 的本地队列"偷"一半放到自己 P 的本地队列.M 运行 G,G 执行之后,M 会从 P 获取下一个 G,不断重复下去.

### P 和 M 的数量与创建时机

#### P

> P 的数量

由环境变量 `GOMAXPROCS` 或 `runtime.GOMAXPROCS()` 进行指定.

> 创建时机

在确定了 P 的最大数量 n 后,运行时系统会根据这个数量创建 n 个 P.

#### M

> M 的数量

- 默认最大限制为 1000,但内核很难支持这么多线程,因此此限制可忽略.
- 该数量可通过 `runtime.SetMaxThreads()` 方法设置 M 的最大数量.
- 倘若一个 M 阻塞了,会创建新的 M.

> 创建时机

没有足够的 M 来关联 P 并运行其中的可运行的 G 时会创建 M.

## go func() 调度流程

![go func() 调度流程](/images/go-func-process.jpeg)

如下是 go func() 的调度流程:

1. 首先通过 go func() 来创建一个 goroutine
2. 有两个存储 G 的队列,一个是局部调度器 P 的本地队列,一个是全局 G 队列.新创建的 G 会先保存在 P 的本地队列中,如果 P 的本地队列已经满了就会保存在全局的队列中
3. M 与 P 绑定后,会从 P 的本地队列弹出一个可执行状态的 G 来执行.如果 P 的本地队列为空,就会尝试从其他的 MP组合"偷"取一个可执行的 G 来执行
4. 当 M 执行某一个 G 时候如果发生了系统调用或则其余阻塞操作,M 会阻塞,如果当前有一些 G 在执行,`runtime` 会把这个线程 M 与 P 解绑(detach),然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)来服务于这个 P
5. 当 M 系统调用结束时候,这个 M 会尝试获取一个空闲的 P 执行,并放入到这个 P 的本地队列.如果获取不到 P,那么这个线程 M 变成休眠状态,加入到空闲线程中,进入休眠队列中.
6. 重复上述过程,直到 P 的本地队列与全局队列中 G 为空

调度器的生命周期可表示如下:

![调度器的生命周期](/images/life-cycle-of-scheduler.png)

---

参考

- [[典藏版]Golang调度器GPM原理与调度全分析](https://www.jianshu.com/p/fa696563c38a)
