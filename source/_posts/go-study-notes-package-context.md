---
title: go 学习笔记之 context 包
date: 2020/05/05
tags:
  - go
categories:
  - go
abbrlink: 2915
description: >-
  本文章主要包含 Go context 包及其内置类型和方法的使用.context 包定义了 Context 类型,该类型在 API
  边界之间及进程基建传递截止日期,取消信号和其它请求范围的信息.
---


对服务的传入请求应创建一个上下文,而对服务器的传出调用应接受一个上下文.它们之间的函数调用链必须传播`Context` 上下文对象.

context 包定义了 `Context` 上下文类型,并可以选择使用 `WithCancel`,`WithDeadline`,`WithTimeout` 和 `WithValue` 等方法创建派生 `Context` 上下文对象.如下:

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

`WithCancel`,`WithDeadline` 和 `WithTimeout` 函数传入父级 `Context` 对象,并返回派生的子级 `Context` 和`CancelFunc`.调用 `CancelFunc` 会取消该子级 `Context` 对象,删除父级对子级的引用,并停止所有关联的计时器.没有调用 `CancelFunc` 会使子级及其子级上下文对象泄漏,直到父代被取消或计时器触发为止.

上下文的使用遵循如下规则:

- 不要将上下文存储在结构体中,而是将其作为参数传递给需要它的函数
- 不要传递 nil 上下文.如果不确定使用哪个上下文,则使用 `context.TODO`
- 可以将相同的上下文传递给在不同 goroutine 中运行的函数.上下文对于由多个 goroutine 同时使用是安全的.

## 类型及函数定义

- 常见类型

```go
// 取消函数,会取消其上下文.多个 goroutine 可同时调用 CancelFunc,在第一个调用之后,随后对其调用将什么也不做
type CancelFunc func()

// 上下文接口
type Context interface {
    // 返回取消该上下文的时间.如果未设置截止日期,则 ok 返回 false
    Deadline() (deadline time.Time, ok bool)
    // 返回一个取消此上下文或超时的通道
    Done() <-chan struct{}
    // 如果 Done() 返回的通道尚未关闭,返回 nil.否则返回非 nil 错误,用于解释原因
    Err() error
    // 返回指定上下文中指定 key 的值.如果没有则返回 nil
    Value(key interface{}) interface{}
}
```

- 常用函数

```go
// 返回具有新 Done 通道的 parent 上下文副本.
// 当调用返回的 cancel 函数或关闭 parent 上下文的 Done 通道时,关闭返回的上下文的 Done 通道
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// 返回带有取消时间的 parent 上下文的副本
// 如果 parent 上下文的截止时间早于 d,则该函数返回的上下文在语义上等同于 parent.
// 当截止时间到期,调用返回的 cancel 函数或关闭 parent 上下文的 Done 通道时,将关闭返回的上下文的 Done 通道
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)  

// 返回 WithDeadline(parent, time.Now().Add(timeout)),返回带有超时时间的 parent 上下文副本
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// 返回一个非空 Context 上下文,它永远不会被取消.它通常用于主函数,初始化和测试,用作传入请求的顶级上下文
func Background() Context  

// 返回一个非空 Context 上下文,当不清楚要使用哪个上下文或尚不可用时,应使用 `context.TODO`
func func TODO() Context  

// 返回与 parent 上下文关联的副本,其中包含 key=val 变量
func WithValue(parent Context, key, val interface{}) Context  
```

## Context

`Context` 定义如下,包含了过期日期,取消信号和键值对等 API.

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    // 返回代表此山下文工作完成时关闭的 channel.
    // 此 channel 在 WithCancel 在调用 cancel 函数时关闭,在 WithDeadline 在 deadline 之前关闭,在 WithTimeout 超时后关闭.
    // 同时可以在 select 语句中使用此函数,以接收上线关闭的信号.
    // select {
    //      case <-ctx.Done():
    //          return ctx.Err()
    //      }
    // 
    Done() <-chan struct{}
    Err() error
    // 返回与此上下文关联的键值
    Value(key interface{}) interface{}
}
```

## WithCancel()

首先看如下示例,创建了只能显式取消的子上下文对象.

```go

import (
    "context"
    "fmt"
    "sync"
)

func ctxCancel() {
    var wg sync.WaitGroup
    ctx, cancel := context.WithCancel(context.Background())
    wg.Add(1)
    go func(ctx context.Context) {
        defer wg.Done()
        select {
        case <-ctx.Done():
            fmt.Println("ctx.done()", ctx.Err())
        case <-time.After(time.Second):
            fmt.Println("time out", ctx.Err())
        }
    }(ctx)
    // Tip: 只能通过调用该函数向 ctx.Done() channel 发送完成的信号
    cancel()
    wg.Wait()
}
```

`WithCancel()` 方法如下,返回 `cancelCtx` 对象与 `cancelFunc` 方法.

```go
// 使用 WithCancel 创建子上下文对象的
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
```

```go
// cancelCtx 结构体如下
type cancelCtx struct {
    Context
    mu       sync.Mutex            // protects following fields
    done     chan struct{}         // created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

// Tip: 显示关闭时调用的方法
// 可以看到该方法关闭了 c.done.取消了 c 的子上下文对象.并删除了其所有孩子节点
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    if c.done == nil {
        c.done = closedchan
    } else {
        close(c.done)
    }
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

## WithTimeout() 与 WithDeadline()

首先看如下示例,创建了带有超时时间与过期时间的子上下文对象.

```go
import (
    "context"
    "fmt"
    "sync"
    "time"
)

func ctxTimeout() {
    var wg sync.WaitGroup
    // Tip: 设置自动超时时间,向 ctx.Done() 发送信号
    ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond)
    defer cancel() // Tip: 若函数退出,则自动调用 cancel() 函数,释放上下文及其子上下文资源
    wg.Add(1)
    go func(ctx context.Context) {
        defer wg.Done()
        select {
        case <-ctx.Done():
            fmt.Println("ctx.done()", ctx.Err())
        case <-time.After(time.Second): // Tip: 这里应该大于 ctx 的超时时间
            fmt.Println("time out", ctx.Err())
        }
    }(ctx)
    wg.Wait()
}

func ctxDeadline() {
    var wg sync.WaitGroup
    // Tip: 设置自动过期时间,向 ctx.Done() 发送信号
    ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(time.Millisecond))
    defer cancel()
    wg.Add(1)
    go func(ctx context.Context) {
        defer wg.Done()
        select {
        case <-ctx.Done():
            fmt.Println("ctx.done()", ctx.Err())
        case <-time.After(time.Second): // Tip: 这里应该大于 ctx 的超时时间
            fmt.Println("time out", ctx.Err())
        }
    }(ctx)
    wg.Wait()
}
```

可以看到 `WithTimeout()` 函数其实是调用了 `WithDeadline()` 函数.返回的上下对象为 `timerCtx`.

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

`timerCtx` 结构体如下.相比于 `cancelCtx`,该结构体包含了计时器 `timer` 与 `deadline` 过期时间.它调用了 `cancelCtx.cancel()`,并停止了计时器.

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.
    deadline time.Time
}

// Tip: 调用了 cancelCtx.cancel() 来取消上下文,并停止了 timer,将其设置为 nil
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

## WithValue()

首先看如下示例,创建了带有父上下文中键值对的子上下文对象.

```go
import (
    "context"
    "fmt"
    "sync"
)

type User struct {
    name string
    age  int
}

func ctxValue() {
    var wg sync.WaitGroup
    // Tip: 创建包含键值对的 ctx,其中 key,value 均为 interface 类型,可以将对象传入
    user := User{
        name: "tom",
        age:  18,
    }
    ctx := context.WithValue(context.Background(), "user", user)
    wg.Add(1)
    go func(ctx context.Context) {
        defer wg.Done()
        // Tip: 可以在携程中通过传入的上下文对象将指定键值取出来
        if v, ok := ctx.Value("user").(User); ok {
            fmt.Printf("user is %v", v)
        }
    }(ctx)
    wg.Wait()
}
```

可以看到 `WithValue()` 函数返回的上下对象为 `valueCtx`,且不提供相关的 `cancelFunc` 方法.

```go
func WithValue(parent Context, key, val interface{}) Context {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

`valueCtx` 结构体如下,其仅在 `Context` 的基础上添加了 `key, val interface{}` 成员对象,用于将键值对传入到子上下文对象中.其中其 `Value()` 方法会首先在当前上下文中查找指定键的值,找到则返回,否则在其父上下文中进行查找.

```go
type valueCtx struct {
    Context
    key, val interface{}
}

// Tip: 用于查找上下文中指定的键,并返回其值
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

需要注意的是:

- `WithValue()` 函数传入的 `key` 必须是可比较的,并且不能为字符串类型或任何其他内置类型,以避免在上下文之间发生冲突.
- 用户应定义自己的 `key` 类型,`key` 通常是具体的的 `struct{}` 结构体.
