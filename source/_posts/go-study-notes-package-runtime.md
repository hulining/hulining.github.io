---
title: go 学习笔记之 runtime 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 17440
description: >-
  本文章主要包含 Go runtime 包及其内置类型和方法的使用.runtime 包主要包含与 Go 运行时系统交互的操作,如控制 goroutine
  的函数.
---

`runtime` 包提供了与 Go 运行时系统交互操作的函数.导入方式为 `import "runtime"`

# 环境变量

下面的主机上的环境变量控制 Go 程序的运行时行为,其用法和含义可能在各个版本间改变

## `GOGC`

`GOGC` 设置初始垃圾回收目标百分比.当新分配的数据与上一次垃圾回收之后的剩余数据之比达到此变量值时,将触发垃圾回收.默认 `GOGC=100`. `GOGC=off` 将完全禁用垃圾回收.`runtime/debug` 包的 `SetGCPercent` 函数可以在 Go 程序运行时修改此百分比.参见 [https://golang.org/pkg/runtime/debug/#SetGCPercent](https://golang.org/pkg/runtime/debug/#SetGCPercent)

## `GODEBUG`

`GODEBUG` 环境变量控制运行时中的调试变量,它是一个用逗号分割的 `name=val` 列表,用于设置以下变量

```
allocfreetrace: setting allocfreetrace=1 causes every allocation to be profiled and a stack trace printed on each object's allocation and free

clobberfree: setting clobberfree=1 causes the garbage collector to clobber the memory content of an object with bad content when it freesthe object.

cgocheck: setting cgocheck=0 disables all checks for packages using cgo to incorrectly pass Go pointers to non-Go code.
Setting cgocheck=1 (the default) enables relatively cheap checks that may miss some errors.
Setting cgocheck=2 enables expensive checks that should not miss any errors, but will cause your program to run slower.

efence: setting efence=1 causes the allocator to run in a mode where each object is allocated on a unique page and addresses are never recycled.
```

- allocfreetrace: 设置 allocfreetrace=1 使每次分配内存都进行分析,并在每个对象的内存分配及释放时打印堆栈跟踪信息
- clobberfree: 设置 clobberfree=1 使垃圾回收器在释放对象时用不好的内容破坏对象的内存内容
- cgocheck：设置 cgocheck=0 将禁用所有使用 cgo 将包指针错误传递给非 Go 代码的包检查;cgocheck=1(默认) 将启用相对低耗的检查,可能会丢失一些错误;cgocheck=2 将启用高消耗的检查,不会遗漏任何错误,但会导致程序运行缓慢
- efence：设置 efence=1 分配器以每个对象都分配唯一页面,并且地址从不回收的方式运行

## `GOMAXPROCS`

`GOMAXPROCS` 环境变量限制了可同时执行用户级 Go 代码的操作系统线程的数量.对 Go 代码在系统调用中被阻塞的线程数量没有限制.那些不计入 `GOMAXPROCS` 限制. `runtime.GOMAXPROCS()` 函数查询并修改限制.

## `GORACE`

`GORACE` 环境变量为使用 `-race` 标志的构建程序配置了竞争检测器.详情参见 [https://golang.org/doc/articles/race_detector.html](https://golang.org/doc/articles/race_detector.html )

## `GOTRACEBACK`

`GOTRACEBACK` 环境变量控制在 Go 程序由于未处理的异常或意外的运行时条件而失败时产生的输出量.默认情况下,产生错误时将打印当前 goroutine 的堆栈跟踪,忽略运行时系统的内部函数,然后以退出代码 2 退出.如果没有当前 goroutine 或故障是运行时内部的,则产生错误时会打印所有 goroutine 的堆栈跟踪信息.

- `GOTRACEBACK=none` 表示完全省略 goroutine 堆栈跟踪信息
- `GOTRACEBACK=single`(默认)的行为如上所述
- `GOTRACEBACK=all` 表示为所有的用户创建的 goroutine 添加堆栈跟踪信息
- `GOTRACEBACK=system` 与 `all`类似,但为运行时函数添加了堆栈跟踪信息,并显示了在运行时内部创建的 goroutine
- `GOTRACEBACK=crash` 类似于 `system`,但多用于操作系统崩溃而不是程序退出

GOTRACEBACK 设置 0,1 和 2 分别是 none,all 和 system 的同义词.`runtime/debug` 包的`SetTraceback` 函数允许在运行时增加输出量,但不能将其减少到环境变量指定的量以下.请参阅 [https://golang.org/pkg/runtime/debug/#SetTraceback](https://golang.org/pkg/runtime/debug/#SetTraceback)

## `GOARCH`,`GOOS`,`GOPATH`,`GOROOT`

`GOARCH`,`GOOS`,`GOPATH`,`GOROOT` 环境变量完善了 Go 环境变量集.它们会影响 Go 程序的构建(请参阅 [https://golang.org/cmd/go](https://golang.org/cmd/go) 和 [https://golang.org/pkg/go/build](https://golang.org/pkg/go/build)).`GOARCH`,`GOOS`,`GOROOT` 会在编译时记录下来,并可以通过此包中的常量或函数来使用,但它们不会影响运行时系统的执行.

# 常用类型定义

```go
type Func struct {
    // 内含隐藏或非导出字段
}
```

# 常用内置常量及变量

```go
const Compiler = "gc"  // 编译器
const GOOS string = sys.GOOS // 操作系统,windows,linux,freebsd,darwin等
const GOARCH string = sys.GOARCH // 平台,386,amd64 或 arm
var MemProfileRate int = 512 * 1024
```

# 常用函数

## `runtime` 包函数

```go
// 返回 Go 的根目录.如果存在 GOROOT 环境变量,返回该变量的值
func GOROOT() string
// 返回 Go 的版本字符串表示
func Version() string
// 返回机器的逻辑 CPU 个数
func NumCPU() int
// 设置可同时执行(并行)的最大 CPU 数.返回先前的设置.如果 n<1,则不会更改当前设置.
func GOMAXPROCS(n int) int
// 设置 CPU profile记录的频率为每秒 hz 次.如果 hz<=0,会关闭 profile 记录
func SetCPUProfileRate(hz int)
// 显式执行一次垃圾回收
func GC()
// 终止调用它的 goroutine.其它 goroutine 不受影响.
// 在终止 goroutine 之前会运行所有的延迟调用,且函数中的 recover 调用都将返回 nil. 
// 在主 goroutine 调用该函数会终止该 goroutine,主函数不会返回.程序将执行其它 goroutine.如果所有其它 goroutine 退出,程序将崩溃
func Goexit()
// 使当前 goroutine 释放处理器,允许其它 goroutine 运行.他不会挂起当前 goroutine,因此当前 goroutine 会自动恢复执行 
func Gosched()
// 绑定当前 goroutine 到它所在的操作系统线程.当前 goroutine将总是在该线程中执行,其它 goroutine 则不能进入该线程,除非调用相同次数的 UnlockOSThread.
// 如果调用的 goroutine 在没有解锁线程的情况下退出,则该线程将终止.
func LockOSThread()
// 解除当前 goroutine 与操作系统线程的绑定关系.若 goroutine 未调用 LockOSThread,则不做操作
func UnlockOSThread()
// 返回当前进程对 cgo 调用的数量
func NumCgoCall() int64
// 返回当前存在的 goroutine 数量
func NumGoroutine() int
// 返回 *Func 用于描述给定指针地址的函数或 nil.如果 pc 由于内联表示多个函数,则将返回描述最内部的函数 *Func,但带有外部函数成员地址
func FuncForPC(pc uintptr) *Func
```

## `Func` 结构体方法

```go
// 返回外部函数的地址
func (f *Func) Entry() uintptr
// 返回函数名称
func (f *Func) Name() string
```