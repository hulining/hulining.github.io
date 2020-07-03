---
title: prometheus client_golang 简单使用
date: 2020/07/01
tags:
  - go
  - Prometheus
categories:
  - Prometheus
abbrlink: 
description: 根据 Prometheus client_golang 官方文档总结的简单使用方法.记录以作备忘.
---

[Prometheus](https://prometheus.io/) 生态是一款优秀的开源监控解决方案,其中包括如下组件

- [Prometheus](https://github.com/prometheus/prometheus)

通过配置各个采集任务,采集各个 expoter 或 pushgateway 数据,保存到其内部的时间序列数据库(TSDB)中.并根据规则对采集到的数据指标进行计算或重新保存为新的数据指标,判断是否达到阈值并向 Alertmanager 推送告警信息.

- [Alertmanager](https://github.com/prometheus/alertmanager)

接收 Prometheus 推送过来的告警信息,通过告警路由,向集成的组件/工具发送告警信息.

- 各种 [Exporter](https://prometheus.io/docs/instrumenting/exporters/)

收集系统或进程信息,转换为 Prometheus 可以识别的数据指标,以 http 或 https 服务的方式暴露给 Prometheus.

- [Pushgateway]

收集系统或进程信息,转换为 Prometheus 可以识别的数据指标,向 Prometheus 推送数据指标.

本篇文章主要内容为根据 [Prometheus client_golang 官方文档](https://godoc.org/github.com/prometheus/client_golang)总结的 Prometheus client_golang 简单使用方法.

## 包结构

[prometheus/client_golang](https://github.com/prometheus/client_golang) 包结构如下:

包路径 | 简介
:---: | :---:
api | api 包提供了 Prometheus HTTP API
api/prometheus/v1 | v1 包提供了 v1 版本的 Prometheus HTTP API,详见 [http://prometheus.io/docs/querying/api/](http://prometheus.io/docs/querying/api/)
examples/random | 一个简单的示例,将具有不同类型的随机分布(均匀,正态和指数)的虚构 RPC 延迟公开为 Prometheus 数据指标
examples/simple | 一个使用 Prometheus 工具的最小示例
prometheus | prometheus 包是 prometheus/client_golang 的核心包
prometheus/graphite | graphite 包提供了将 Prometheus 数据指标推送到 Graphite 服务的相关代码
prometheus/internal | 内部包
prometheus/promauto | promauto 包提供了 Prometheus 指标的基本数据类型及其 `…Vec` 和 `…Func` 变体数据类型的构造函数
prometheus/promhttp | promhttp 包提供了 HTTP 服务端和客户端相关工具
prometheus/push | push 包提供了将指标推送到 Pushgateway 的函数
prometheus/testutil | testutil 包提供了测试使用 prometheus/client_golang 编写的代码的帮助程序
prometheus/testutil/promlint | promlint 包为 Prometheus 数据指标提供一个参考.

## `promauto` 包

导入方式: `import "github.com/prometheus/client_golang/prometheus/promauto"`

`promauto` 包提供了 Prometheus 指标的基本数据类型及其 `…Vec` 和 `…Func` 变体数据类型的构造函数.与 `prometheus` 包中提供的构造函数不同的是,`promauto` 包中的构造函数返回已经注册的 `Collector` 对象.

`promauto` 包中包含三组构造函数,`New<Metric>`, `New<Metric>Vec` 与 `New<Metric>Func`.如果注册失败,所有构造函数都会引发 `panic`

- `New<Metric>` 返回已在全局注册表 `prometheus.DefaultRegisterer` 中注册的 `Collector`
- `New<Metric>Vec` 返回已在全局注册表 `prometheus.DefaultRegisterer` 中注册的带有标签的 `Collector`.
- `New<Metric>Func` 返回已在 `prometheus.DefaultRegisterer` 构造工厂中注册的 `Collector`,指标值由传入的 `func() float64` 函数对象返回. `New<Metric>Func` 仅支持 `Counter` 与 `Gauge` 类型的数据指标.

`promauto` 包中 `NewXXX` 函数,其实以上方法都是调用了 `prometheus` 包中对应的方法创建了 `Collector` 对象,并使 `Collector` 对象在 `prometheus.DefaultRegisterer` 注册.示例如下:

```go
// 通过 `promauto` 包中方法创建的 Collector 对象 histogramRegistered 已被注册,可直接被公开为数据指标
var histogramRegistered = promauto.NewHistogram(
    prometheus.HistogramOpts{
        Name:    "random_number_registered",
        Help:    "A histogram of normally distributed random numbers.",
        Buckets: prometheus.LinearBuckets(-3, .1, 61),
    },
)

// 通过 `prometheus` 包中方法创建的 Collector 对象 histogramNotRegistered 没有被注册,需要手动进行注册才被公开为数据指标
var histogramNotRegistered = prometheus.NewHistogram(
    prometheus.HistogramOpts{
        Name:    "random_number_not_registered",
        Help:    "A histogram of normally distributed random numbers.",
        Buckets: prometheus.LinearBuckets(-3, .1, 61),
    },
)

// 在 `init()` 函数中对 histogramNotRegistered 对象进行注册
func init() {
    prometheus.Register(histogramNotRegistered)
}
```

同时,`promauto` 包还提供了 `With` 函数.该函数返回创建 `Collector` 的工厂对象.通过该工厂对象创建的 `Collector` 都在传入的 `Registerer` 对象中进行注册

```go
func With(r prometheus.Registerer) Factory { return Factory{r} }
```

示例如下:

```go
var (
    reg           = prometheus.NewRegistry()
    factory       = promauto.With(reg) // 使用 reg 定义工厂对象
    // 创建已在 reg 中注册的 histogram 类型的数据对象,该对象实现了 Histogram 接口与 Metric 接口
    randomNumbers = factory.NewHistogram(prometheus.HistogramOpts{
        Name:    "random_numbers",
        Help:    "A histogram of normally distributed random numbers.",
        Buckets: prometheus.LinearBuckets(-3, .1, 61),
    })
    // 创建已在 reg 中注册的带有标签的 counter 类型的数据对象,该对象实现了 Collector 接口
    requestCount = factory.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests by status code and method.",
        },
        []string{"code", "method"},
    )
)
```
