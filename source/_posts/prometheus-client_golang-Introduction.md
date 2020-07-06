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

## `prometheus` 包

导入方式 `import "github.com/prometheus/client_golang/prometheus"`.

`prometheus` 包是 prometheus/client_golang 的核心包.它为工具代码提供原生数据指标用于监控,并为数据指标对象提供了注册表.`promauto` 为数据指标提供自动注册的构造函数,`promhttp` 子包允许通过 HTTP 公开已注册的数据指标,`push` 子包可以将已注册的数据指标推送到 Pushgateway.

示例如下:

```go
package main

import (
    "log"
    "net/http"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // 通过 NewGauge() 方法创建 Gauge 接口的实现对象
    cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "cpu_temperature_celsius",
        Help: "Current temperature of the CPU.",
    })
    // 通过 NewCounterVec() 方法创建带"标签"的 CounterVec 对象
    hdFailures = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "hd_errors_total",
            Help: "Number of hard-disk errors.",
        },
        []string{"device"},
    )
)

func init() {
    // Metrics 数据指标必须被注册后才会被公开
    prometheus.MustRegister(cpuTemp)
    prometheus.MustRegister(hdFailures)
}

func main() {
    cpuTemp.Set(65.3)
    hdFailures.With(prometheus.Labels{"device":"/dev/sda"}).Inc()

    // Handler 函数提供了默认的处理程序,以便通过 HTTP 服务公开指标.通常使用 "/metrics" 作为入口点.
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 数据指标

`prometheus` 包提供了四种基本的数据指标类型,`Counter`,`Gauge`,`Histogram` 和 `Summary`.可以在[Prometheus docs](https://prometheus.io/docs/concepts/metric_types/) 中找到对这四种度量标准类型的更全面的描述.

除了四种基本的数据指标类型外,Prometheus 数据模型的一个非常重要的部分是沿着称为 "标签" 的维度对数据指标样本进行划分,这就产生了数据指标向量(metric vectors).`prometheus` 分为为四种基本数据指标类型提供了相应的数据指标向量,分别是 `CounterVec`,`GaugeVec`,`HistogramVec` 和 `SummaryVec`.

数据指标向量的方法如下:

```go
Collect(ch chan<- Metric)  // 实现 Collector 接口的 Collect() 方法
CurryWith(labels Labels)  // 返回带有指定标签的向量指标及可能发生的错误.多用于 promhttp 包中的中间件.
Delete(labels Labels)  // 删除带有指定标签的向量指标.如果删除了指标,返回 true
DeleteLabelValues(lvs ...string)  // 删除带有指定标签和标签值的向量指标.如果删除了指标,返回 true
Describe(ch chan<- *Desc)  // 实现 Collector 接口的 Describe() 方法
GetMetricWith(labels Labels)  // 返回带有指定标签的数据指标及可能发生的错误
GetMetricWithLabelValues(lvs ...string)  // 返回带有指定标签和标签值的数据指标及可能发生的错误
MustCurryWith(labels Labels)  // 与 CurryWith 相同,但如果出现错误,则引发 panics
Reset()  // 删除此指标向量中的所有数据指标
With(labels Labels)  // 与 GetMetricWithLabels 相同,但如果出现错误,则引发 panics
WithLabelValues(lvs ...string)  // 与 GetMetricWithLabelValues 相同,但如果出现错误,则引发 panics
```

要创建 `Metrics` 及其向量版本的实例,您需要一个合适的`…Opts`结构,即 `GaugeOpts`,`CounterOpts`,`SummaryOpts` 或 `HistogramOpts`.结构体如下:

```go
// 其中 GaugeOpts, CounterOpts 实际上均为 Opts 的别名
type CounterOpts Opts
type GaugeOpts Opts
type Opts struct {
    // Namespace, Subsystem, and Name  是 Metric 名称的组成部分(通过 "_" 将这些组成部分连接起来),只有 Name 是必需的.
    Namespace string
    Subsystem string
    Name      string

    // Help 提供 Metric 的信息.具有相同名称的 Metric 必须具有相同的 Help 信息
    Help string

    // ConstLabels 用于将固定标签附加到该指标.很少使用.
    ConstLabels Labels
}

type HistogramOpts struct {
    // Namespace, Subsystem, and Name  是 Metric 名称的组成部分(通过 "_" 将这些组成部分连接起来),只有 Name 是必需的.
    Namespace string
    Subsystem string
    Name      string

    //  Help 提供 Metric 的信息.具有相同名称的 Metric 必须具有相同的 Help 信息
    Help string

    // ConstLabels 用于将固定标签附加到该指标.很少使用.
    ConstLabels Labels

    // Buckets 定义了观察值的取值区间.切片中的每个元素值都是区间的上限,元素值必须按升序排序.
    // Buckets 会隐式添加 `+Inf` 值作为取值区间的最大值
    // 默认值是 DefBuckets =  []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10}
    Buckets []float64
}

type SummaryOpts struct {
    // Namespace, Subsystem, and Name  是 Metric 名称的组成部分(通过 "_" 将这些组成部分连接起来),只有 Name 是必需的.
    Namespace string
    Subsystem string
    Name      string

    //  Help 提供 Metric 的信息.具有相同名称的 Metric 必须具有相同的 Help 信息
    Help string

    // ConstLabels 用于将固定标签附加到该指标.很少使用.
    ConstLabels Labels

    // Objectives 定义了分位数等级估计及其各自的绝对误差.如果 Objectives[q] = e，则 q 报告的值将是 [q-e, q + e]之间某个 φ 的 φ 分位数
    // 默认值为空 map,表示没有分位数的摘要
    Objectives map[float64]float64

    // MaxAge 定义观察值与摘要保持相关的持续时间.必须是正数.默认值为 DefMaxAge = 10 * time.Minute
    MaxAge time.Duration

    // AgeBuckets 用于从摘要中排除早于 MaxAge 的观察值的取值区间.默认值为 DefAgeBuckets = 5
    AgeBuckets uint32

    // BufCap 定义默认样本流缓冲区大小.默认值为 DefBufCap = 500.
    BufCap uint32
}
```

这里需要注意的是 `Counter`,`Gauge`,`Histogram`,`Summary` 都继承了 `Metric` 和 `Collector` 接口,其本身是接口类型.而 `CounterVec`,`GaugeVec`,`HistogramVec`, `SummaryVec` 均继承自 `metricVec` 结构体,其本身的是结构体,而它们只实现了 `Collector` 接口.

### 自定义 Collectors 和常量指标

`prometheus` 包提供了 `NewConstMetric()`,`NewConstHistogram()`,`NewConstSummary()` 及其各自的 `Must…` 版本的函数 "动态" 创建 `Metric` 实例.其中 `NewConstMetric()` 函数用于创建仅以 float64 数据作为其值的数据指标类型对象,如 `Counter`,`Gauge` 及 `Untyped` 特殊类型对象.`Metric` 实例的创建在 `Collect()` 方法中进行.

`prometheus` 包提供了 `NewDesc()` 函数创建用于描述以上 `Metric` 实例的 `Desc` 对象,其中主要包含  `Metric` 实例的名称与帮助信息.

`prometheus` 包还提供了 `NewCounterFunc()`,`NewGaugeFunc()` 或 `NewUntypedFunc()` 函数用于创建实现了 `CounterFunc`, `GaugeFunc`, `UntypedFunc` 接口的 `valueFunc` 对象,用于只需要以传入函数的浮点数返回值作为数据指标值创建数据指标的场景.

### Registry 的高级用法

`prometheus` 包提供了 `MustRegister()` 函数用于注册 `Collector`,但如果注册过程中发生错误,程序会引发 panics.而使用 `Register()` 函数可以实现注册 `Collector` 的同时处理可能发生的错误.

`prometheus` 包中所有的注册都是在默认的注册表上进行的,可以在全局变量 `DefaultRegisterer` 中找到该对象.`prometheus` 包提供了 `NewRegistry()` 函数用于创建自定义注册表,甚至可以自己实现 `Registerer` 或 `Gatherer` 接口.

 `prometheus` 通过 `NewGoCollector()` 和 `NewProcessCollector()` 函数创建 Go 运行时数据指标的 `Collector` 和进程数据指标的 `Collector`.而这两个 `Collector` 已在默认的注册表 `DefaultRegisterer` 中注册.使用自定义注册表,您可以控制并自行决定要注册的 `Collector`.

### HTTP 公开数据指标

注册表(`Registry` 结构体)实现了 `Gatherer` 接口,实现了 `Gather()` 方法.`Gather()` 方法的调用者可以以某种方式公开收集的数据指标.通常通过 `/metrics` 入口以 HTTP 方式提供.通过 HTTP 公开数据指标的工具在 `promhttp` 包中.

### 推送数据指标到 Pushgateway

在 `push` 子包中可以找到用于推送到 Pushgateway 的函数.

## `promauto` 包

导入方式: `import "github.com/prometheus/client_golang/prometheus/promauto"`

`promauto` 包提供了 Prometheus 指标的基本数据类型及其 `…Vec` 和 `…Func` 变体数据类型的构造函数.与 `prometheus` 包中提供的构造函数不同的是,`promauto` 包中的构造函数返回已经注册的 `Collector` 对象.

`promauto` 包中包含三组构造函数,`New<Metric>`, `New<Metric>Vec` 与 `New<Metric>Func`.

`promauto` 包中 `NewXXX` 函数,其实都是调用了 `prometheus` 包中对应的 `NewXXX` 函数创建了 `Collector` 对象,并将此 `Collector` 对象在 `prometheus.DefaultRegisterer` 中调用 `MustRegister()` 方法注册.因此如果注册失败,所有构造函数都会引发 `panics`.

以 `promauto.NewCounter()` 为例,源代码如下:

```go
// github.com/prometheus/client_golang@v1.7.1/prometheus/promauto/auto.go#L167
func NewCounter(opts prometheus.CounterOpts) prometheus.Counter {
    return With(prometheus.DefaultRegisterer).NewCounter(opts)

func (f Factory) NewCounter(opts prometheus.CounterOpts) prometheus.Counter {
    c := prometheus.NewCounter(opts) // 调用 prometheus 包中对应方法,创建 Counter 实现对象
    if f.r != nil {
        f.r.MustRegister(c) // 调用 `prometheus.DefaultRegisterer.MustRegister()` 方法对 Counter 进行注册
    }
    return c // 返回已注册的 Counter 实现对象
}
```

示例如下:

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

同时,`promauto` 包还提供了 `With()` 函数.该函数返回创建 `Collector` 的工厂对象.通过该工厂对象创建的 `Collector` 都在传入的 `Registerer` 中进行注册.

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

## `promhttp` 包

`promhttp` 包允许创建 `http.Handler` 实例通过 HTTP 公开 Prometheus 数据指标.

### `Handler()` 与 `HandlerFor()` 函数

`promhttp` 包提供了 `Handler()` 函数使用默认的 `prometheus.DefaultGatherer` 返回一个 `http.Handler`.它将第一个错误报告为 HTTP 错误,没有错误记录.返回的 `http.Handler` 已使用 `InstrumentMetricHandler()` 函数和默认的 `prometheus.DefaultRegisterer` 进行检测.如果调用多个 `Handler()` 函数创建多个 `http.Handler`,则用于检测的数据指标将在它们之间共享,从而提供全局采集计数.如 `promhttp_metric_handler_requests_total` 和 `promhttp_metric_handler_requests_in_flight` 数据指标.

`promhttp` 包提供了`HandlerFor()` 函数,您可以为自定义注册表或实现 `Gatherer` 接口的任何内容创建处理程序,还允许通过传入 `HandlerOpts` 对象自定义错误处理行为或记录错误的对象.

### `InstrumentHandlerX` 包装器函数

`promhttp` 包提供了通过中间件来检测 `http.Handler` 实例的工具.中间件包装器遵循 `InstrumentHandlerX` 命名方案,其中 `X` 描述了中间件的预期用途.有关详细信息,请参见每个函数的文档注释.

`promhttp` 包提供以下中间件包装器:

- `func InstrumentHandlerCounter(counter *prometheus.CounterVec, next http.Handler) http.HandlerFunc`

`InstrumentHandlerCounter` 包装传入的 `http.Handler`,并通过传入的 `prometheus.CounterVec` 记录不同请求方法或响应状态分组的计数结果.

`CounterVec` 可以通过 HTTP 状态码或方法对 `CounterVec` 中带有相应标签实例的数据指标进行分组,其允许的标签名称是 `"code"` 和 `"method"`,否则该函数会引发 panics.对于未分区的计数,可使用不带标签的 `CounterVec`.如果装饰的 `Handler` 未设置状态码,默认为 200.

- `func InstrumentHandlerDuration(obs prometheus.ObserverVec, next http.Handler) http.HandlerFunc`

`InstrumentHandlerDuration` 包装传入的 `http.Handler`,并通过传入的 `prometheus.ObserverVec` 记录不同请求方法或响应状态分组的持续时间.持续时间以秒为单位.

与 `CounterVec` 类似,`ObserverVec` 可以通过 HTTP 状态码或方法对 `ObserverVec` 中带有相应标签实例的数据指标进行分组,其默认允许的标签名称是 `"code"` 和 `"method"`,如果除 `method,code` 外有其它标签,需要在包装器中调用 `CurryWith()` 或 `MustCurryWith()` 传入标签的值,否则该函数会引发 panics.对于未分区的计数,可使用不带标签的 `ObserverVec`.如果装饰的 `Handler` 未设置状态码,默认为 200.

- `func InstrumentHandlerInFlight(g prometheus.Gauge, next http.Handler) http.Handler`

`InstrumentHandlerInFlight` 包装传入的 `http.Handler`,它通过传入的 `prometheus.Gauge` 记录处理的请求数

- `func InstrumentHandlerRequestSize(obs prometheus.ObserverVec, next http.Handler) http.HandlerFunc`

`InstrumentHandlerRequestSize` 包装传入的 `http.Handler`,它通过传入的 `prometheus.ObserverVec` 记录不同请求方法或响应状态分组的请求大小.请求大小以字节为单位.

与 `CounterVec` 类似,`ObserverVec` 可以通过 HTTP 状态码或方法对 `ObserverVec` 中带有相应标签实例的数据指标进行分组,其默认允许的标签名称是 `"code"` 和 `"method"`,如果除 `method,code` 外有其它标签,需要在包装器中调用 `CurryWith()` 或 `MustCurryWith()` 传入标签的值,否则该函数会引发 panics.对于未分区的计数,可使用不带标签的 `ObserverVec`.如果装饰的 `Handler` 未设置状态码,默认为 200.

- `func InstrumentHandlerResponseSize(obs prometheus.ObserverVec, next http.Handler) http.Handler`

`InstrumentHandlerResponseSize` 包装传入的 `http.Handler`,它通过传入的 `prometheus.ObserverVec` 记录不同请求方法或响应状态分组的响应大小.响应大小以字节为单位.

与 `CounterVec` 类似,`ObserverVec` 可以通过 HTTP 状态码或方法对 `ObserverVec` 中带有相应标签实例的数据指标进行分组,其默认允许的标签名称是 `"code"` 和 `"method"`,如果除 `method,code` 外有其它标签,需要在包装器中调用 `CurryWith()` 或 `MustCurryWith()` 传入标签的值,否则该函数会引发 panics.对于未分区的计数,可使用不带标签的 `ObserverVec`.如果装饰的 `Handler` 未设置状态码,默认为 200.

- `func InstrumentHandlerTimeToWriteHeader(obs prometheus.ObserverVec, next http.Handler) http.HandlerFunc`

`InstrumentHandlerResponseSize` 包装传入的 `http.Handler`,它通过传入的 `prometheus.ObserverVec` 记录不同请求方法或响应状态分组的写入响应头部的时间.持续时间以秒为单位.

与 `CounterVec` 类似,`ObserverVec` 可以通过 HTTP 状态码或方法对 `ObserverVec` 中带有相应标签实例的数据指标进行分组,其默认允许的标签名称是 `"code"` 和 `"method"`,如果除 `method,code` 外有其它标签,需要在包装器中调用 `CurryWith()` 或 `MustCurryWith()` 传入标签的值,否则该函数会引发 panics.对于未分区的计数,可使用不带标签的 `ObserverVec`.如果装饰的 `Handler` 未设置状态码,默认为 200.

- `func InstrumentMetricHandler(reg prometheus.Registerer, handler http.Handler) http.Handler`

`InstrumentMetricHandler` 通常与 `HandlerFor` 函数返回的 `http.Handler` 一起使用.它使用两个数据指标为传入的 `http.Handler` 进行包装: `promhttp_metric_handler_requests_total`(`CounterVec` 类型) 对按 HTTP 响应状态码分组的请求进行计数,`promhttp_metric_handler_requests_in_flight`(`Gauge` 类型) 跟踪同时进行的请求数量.

如上两个数据指标对于查看多少数据指标采集请求发送到监控目标上以及它们的重复率(同时有多少个采集请求)非常有用.

### 示例

中间件包装器示例如下:

```go
func main() {
    inFlightGauge := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "in_flight_requests",
        Help: "A gauge of requests currently being served by the wrapped handler.",
    })
    // 带有 "code", "method" 标签的计数器
    counter := prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "api_requests_total",
            Help: "A counter for requests to the wrapped handler.",
        },
        []string{"code", "method"},
    )
    // 带标签的 duration.如果除 `method,code` 外有其它标签,需要在包装器中调用 `CurryWith()` 或 `MustCurryWith()` 传入标签的值.
    duration := prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "request_duration_seconds",
            Help:    "A histogram of latencies for requests.",
            Buckets: []float64{.25, .5, 1, 2.5, 5, 10},
        },
        []string{"handler", "method"},
    )
    // 不带标签的 responseSize
    responseSize := prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "response_size_bytes",
            Help:    "A histogram of response sizes for requests.",
            Buckets: []float64{200, 500, 900, 1500},
        },
        []string{},
    )

    // 创建将被中间件包装的 Handlers
    pushHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Push"))
    })
    pullHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Pull"))
    })

    // 在默认的注册表中注册所有的数据指标
    prometheus.MustRegister(inFlightGauge, counter, duration, responseSize)

    // 按照以上定义的数据指标将 Handler 分组,并通过 `ObserverVec` 接口的 `MustCurryWith()` 方法传入 "handler" 标签
    pushChain := promhttp.InstrumentHandlerInFlight(inFlightGauge,
        promhttp.InstrumentHandlerDuration(duration.MustCurryWith(prometheus.Labels{"handler": "push"}),
            promhttp.InstrumentHandlerCounter(counter,
                promhttp.InstrumentHandlerResponseSize(responseSize, pushHandler),
            ),
        ),
    )
    pullChain := promhttp.InstrumentHandlerInFlight(inFlightGauge,
        promhttp.InstrumentHandlerDuration(duration.MustCurryWith(prometheus.Labels{"handler": "pull"}),
            promhttp.InstrumentHandlerCounter(counter,
                promhttp.InstrumentHandlerResponseSize(responseSize, pullHandler),
            ),
        ),
    )

    http.Handle("/metrics", promhttp.Handler())
    // 对不同的 HTTP 入口端点请求应用到带有不同标签的 Handler 中间件包装器
    http.Handle("/pull", pullChain)
    http.Handle("/push", pushChain)

    if err := http.ListenAndServe(":3000", nil); err != nil {
        log.Fatal(err)
    }
}
```

## `push` 包

`push` 包提供了将数据指标推送到 Pushgateway 的函数,它使用构造其函数 `New()` 创建 `Pusher` 对象,然后使用其实例方法添加各种选项,最后调用 `Add()` 或 `Push()` 方法向 Pushgateway 推送数据指标.

```go
push.New("http://example.org/metrics", "my_job").Gatherer(myRegistry).Push()

// Complex case:
push.New("http://example.org/metrics", "my_job").
    Collector(myCollector1).
    Collector(myCollector2).
    Grouping("zone", "xy").
    Client(&myHTTPClient).
    BasicAuth("top", "secret").
    Add()
```

源码解析如下:

```go
// github.com/prometheus/client_golang/prometheus/push/push.go
// 定义 Pusher 结构体及构造方法
type Pusher struct {
    error error
    url, job string
    grouping map[string]string  
    gatherers  prometheus.Gatherers
    registerer prometheus.Registerer=
    client             HTTPDoer
    useBasicAuth       bool
    username, password string
    expfmt expfmt.Format
}

func New(url, job string) *Pusher {
    var (
        reg = prometheus.NewRegistry()
        err error
    )
    if job == "" {
        err = errJobEmpty
    }
    if !strings.Contains(url, "://") {
        url = "http://" + url
    }
    if strings.HasSuffix(url, "/") {
        url = url[:len(url)-1]
    }
    return &Pusher{
        error:      err,
        url:        url,
        job:        job,
        grouping:   map[string]string{},
        gatherers:  prometheus.Gatherers{reg},
        registerer: reg,
        client:     &http.Client{}, // 使用默认的 HTTP 客户端
        expfmt:     expfmt.FmtProtoDelim, // 默认使用 expfmt.FmtProtoDelim
    }
}

// 添加 Gatherer 对象到 Pusher 中
func (p *Pusher) Gatherer(g prometheus.Gatherer) *Pusher {
    p.gatherers = append(p.gatherers, g)
    return p
}

// 添加 Collector 对象到 Pusher 中
func (p *Pusher) Collector(c prometheus.Collector) *Pusher {
    if p.error == nil {
        p.error = p.registerer.Register(c)
    }
    return p
}

// 对 Pusher 添加键值对分组
func (p *Pusher) Grouping(name, value string) *Pusher {
    if p.error == nil {
        if !model.LabelName(name).IsValid() {
            p.error = fmt.Errorf("grouping label has invalid name: %s", name)
            return p
        }
        p.grouping[name] = value
    }
    return p
}

// 自定义 Pusher 向 Pushgateway 发送请求的 HTTP 客户端
// HTTP 客户端是 `HTTPDoer` 接口的实现类,也就是实现了 `Do(*http.Request)` 方法
// 默认的 http.Client 满足这个要求,因此可以直接使用
func (p *Pusher) Client(c HTTPDoer) *Pusher {
    p.client = c
    return p
}

// BasicAuth 配置 Pusher 使用 HTTP Basic Authentication
func (p *Pusher) BasicAuth(username, password string) *Pusher {
    p.useBasicAuth = true
    p.username = username
    p.password = password
    return p
}

// Format 配置 Pusher 使用 `expfmt.Format` 的编码类型.默认为 `expfmt.FmtProtoDelim`
func (p *Pusher) Format(format expfmt.Format) *Pusher {
    p.expfmt = format
    return p
}

// Pusher 向 Pushgateway 发送 DELETE 请求,删除 URL 下所有数据指标.
func (p *Pusher) Delete() error {
    if p.error != nil {
        return p.error
    }
    req, err := http.NewRequest(http.MethodDelete, p.fullURL(), nil)
    if err != nil {
        return err
    }
    if p.useBasicAuth {
        req.SetBasicAuth(p.username, p.password)
    }
    resp, err := p.client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusAccepted {
        body, _ := ioutil.ReadAll(resp.Body) // Ignore any further error as this is for an error message only.
        return fmt.Errorf("unexpected status code %d while deleting %s: %s", resp.StatusCode, p.fullURL(), body)
    }
    return nil
}

// Push 从添加到此 Pusher 的所有 Collector 和 Gatherer 收集所有指标,然后使用配置的作业名称和分组标签作为分组键,将指标推送到配置的 Pushgateway
// 具有该作业和其他分组标签的所有先前推送的数据指标将被此调用推送的指标替换.一般用于第一次推送(类似于删除原来推送的数据,后重新推送)
// Push 方法向 Pushgateway 发送 PUT 请求
func (p *Pusher) Push() error {
    return p.push(http.MethodPut)
}

// Add 的工作方式类似于 Push,但只会替换具有相同名称(以及相同的作业和其他分组标签)的先前推送的指标.一般用于后续推送(类似于对此次推送的数据做更新或添加)
// Add 方法向 Pushgateway 发送 POST 请求
func (p *Pusher) Add() error {
    return p.push(http.MethodPost)
}

// github.com/prometheus/client_golang/prometheus/push/push.goL236
func (p *Pusher) push(method string) error {
    if p.error != nil {
        return p.error
    }
    // 这里应该是收集各个注册的 Collector 的数据指标
    mfs, err := p.gatherers.Gather()
    if err != nil {
        return err
    }
    buf := &bytes.Buffer{}
    enc := expfmt.NewEncoder(buf, p.expfmt)
    // Check for pre-existing grouping labels:
    for _, mf := range mfs {
        for _, m := range mf.GetMetric() {
            for _, l := range m.GetLabel() {
                if l.GetName() == "job" {
                    return fmt.Errorf("pushed metric %s (%s) already contains a job label", mf.GetName(), m)
                }
                if _, ok := p.grouping[l.GetName()]; ok {
                    return fmt.Errorf(
                        "pushed metric %s (%s) already contains grouping label %s",
                        mf.GetName(), m, l.GetName(),
                    )
                }
            }
        }
        // 这里应该是将 mf 序列化后写入了请求体 buf
        enc.Encode(mf)
    }
    // 使用 http 包创建新请求,请求使用指定的 方法,URL 与 请求体 buf,并配置请求的各种参数
    req, err := http.NewRequest(method, p.fullURL(), buf)
    if err != nil {
        return err
    }
    if p.useBasicAuth {
        req.SetBasicAuth(p.username, p.password)
    }
    req.Header.Set(contentTypeHeader, string(p.expfmt))
    // 发送请求
    resp, err := p.client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    // Depending on version and configuration of the PGW, StatusOK or StatusAccepted may be returned.
    if resp.StatusCode != http.StatusOK && resp.StatusCode != http.StatusAccepted {
        body, _ := ioutil.ReadAll(resp.Body) // Ignore any further error as this is for an error message only.
        return fmt.Errorf("unexpected status code %d while pushing to %s: %s", resp.StatusCode, p.fullURL(), body)
    }
    return nil
}
```
