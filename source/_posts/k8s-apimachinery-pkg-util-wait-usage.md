---
title: k8s.io/apimachinery/pkg/util/wait 包使用
date: 2023-02-23 11:10:14
tags:
  - go
  - Kubernetes
  - tools
categories:
  - Kubernetes
description: 参考 https://pkg.go.dev/k8s.io/apimachinery/pkg/util/wait 整理 wait 包的使用
---

阅读 k8s 源码过程发现总会有 `wait.Poll(param)` 之类的函数调用，因此参考 [pkg.go.dev](https://pkg.go.dev/k8s.io/apimachinery/pkg/util/wait ) 与 [github.com](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/wait/wait.go) 对 `wait` 包中提供的常用函数做了汇总，提升阅读效率。

```golang
type ConditionFunc func() (done bool, err error)
```

```golang
// 等待 interval 后执行一次 ConditionFunc,之后每隔 interval 时间执行一次 ConditionFunc,直到它返回 true,err
func Poll(interval time.Duration, condition ConditionFunc) error

// 等待 interval 后执行一次 ConditionFunc,之后每隔 interval 时间执行一次 ConditionFunc,直到它返回 true,err或timeout
func PollInfinite(interval time.Duration, condition ConditionFunc) error

// 等待 interval 后执行一次 ConditionFunc,之后每隔 interval 时间执行一次 ConditionFunc,直到它返回 true,err 或 stopCh 被关闭
func PollUntil(interval time.Duration, condition ConditionFunc, stopCh <-chan struct{}) error
```

```golang
// 立即执行一次 ConditionFunc,之后每隔 interval 时间执行一次 ConditionFunc,直到它返回 true,err或timeout.共执行 timeout/interval 次
func PollImmediate(interval, timeout time.Duration, condition ConditionFunc) error

// 立即执行一次 ConditionFunc,之后每隔 interval 时间执行一次 ConditionFunc,直到它返回 true或err
func PollImmediateInfinite(interval time.Duration, condition ConditionFunc) error

// 立即执行一次 ConditionFunc,之后每隔 interval 时间执行一次 ConditionFunc,直到它返回 true,err或 stopCh 被关闭
func PollImmediateUntil(interval time.Duration, condition ConditionFunc, stopCh <-chan struct{}) error
```

```golang
// 循环执行 f,直到 stopCh 通道关闭.可以传入 `NeverStop` 参数
// 如果 jitterFactor > 0,则等待周期会在 (period, period+jitterFactor*period) 内抖动.否则,
// 如果 sliding 为 true,则函数执行完成后才计算开始等待周期;否则,等待周期将包含函数的执行时间
func JitterUntil(f func(), period time.Duration, jitterFactor float64, sliding bool, stopCh <-chan struct{})

// 直到 stopCh 关闭,每隔 period 周期运行 f
// 实际是调用了 `JitterUntil(f, period, 0.0, true, stopCh)`.也就是等函数执行完成后再等待 period,运行 f
func Until(f func(), period time.Duration, stopCh <-chan struct{})
// 实际是调用了 `JitterUntil(f, period, 0.0, false, stopCh)`.也就是每隔 period 就周期执行 f,不考虑 f 的执行时间
func NonSlidingUntil(f func(), period time.Duration, stopCh <-chan struct{})
```

以上函数均支持传入`ctx context.Context` 参数,形成带上下文的函数.
如 `func PollImmediateWithContext(ctx context.Context, interval, timeout time.Duration, condition ConditionWithContextFunc) error` 等

## 参考

- [pkg.go.dev](https://pkg.go.dev/k8s.io/apimachinery/pkg/util/wait)
- [github.com](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/wait/wait.go)