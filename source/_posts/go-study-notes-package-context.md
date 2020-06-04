---
title: go 学习笔记之 context 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 2915
description: >-
  本文章主要包含 Go context 包及其内置类型和方法的使用.context 包定义了 Context 类型,该类型在 API
  边界之间及进程基建传递截止日期,取消信号和其它请求范围的信息.
---

context 包定义了 Context 类型,该类型在 API 边界之间及进程基建传递截止日期,取消信号和其它请求范围的信息.导入方式为 `import context`

向服务发送请求应该创建一个 `Context` 上下文对象,而对服务的传出调用应该接受一个上下文对象.它们之间的函数调用链必须传播 `Context`,可以使用 `WithCancel`, `WithDeadline`,`WithTimeout` 或 `WithValue` 创建派生上下文作为替换.取消上下文后,从该上下文派生的所有上下文也会被取消

> 应用场景：在 Go http 包的 Server 中,每一个请求在都有一个对应的 goroutine 去处理.请求处理函数通常会启动额外的 goroutine 用来访问后端服务,比如数据库和 RPC 服务.用来处理一个请求的 goroutine 通常需要访问一些与请求特定的数据,比如终端用户的身份认证信息,验证相关的 token,请求的截止时间等.当一个请求被取消或超时时,所有用来处理该请求的 goroutine 都应该迅速退出,然后系统才能释放这些 goroutine 占用的资源

`WithCancel`,`WithDeadline` 和 `WithTimeout` 函数接受 Context(parent)对象并返回派生的 `Context`(child)对象和 `CancelFunc`.调用 `CancelFunc` 会取消该子上下文及其子上下文,删除 parent 上下文对 child 上下文的引用,并停止所有关联计时器.

# 遵循规则

使用 Contexts 上下文的程序都应该遵循以下规则,使各个包之间的接口保持一致:

- 不要将上下文存储在结构体类型,而是将上下文显式传递给需要它的函数.且 Context 应该是第一个参数,通常命名为 `ctx`:
```go
func DoSomething(ctx context.Context, arg Arg) error {
    // ... use ctx ...
}
```
- 即使函数支持,也不要传递 nil 上下文对象.如果不确定使用哪个上下文,请使用 `context.TODO`
- 仅将上下文中的值用于传递过程和 API 的请求范围的数据,而不是作为可选参数传递给函数
- 可以将相同的上下文传递给在不同 goroutine 中运行的函数.上下文对于由多个 goroutine 同时使用是安全的


# 类型定义

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

# 常用函数

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

