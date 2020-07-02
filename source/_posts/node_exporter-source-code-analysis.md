---
title: node_exporter 源码解析
date: 2020/07/01
tags:
  - go
  - node_exporter
  - Prometheus
categories:
  - node_exporter
abbrlink: 
description: 一直想使用 Golang 写个项目,但不知道怎么入手,所以就看了看 node_exporter 源码,发现人家写的代码真的很棒.这里将阅读 node_exporter 源码做一下记录,以作备忘.
---

## 代码组织

[`node_exporter`](https://github.com/prometheus/node_exporter) 是使用 Go 语言编写的 Prometheus exporter,多用于收集 *NIX 内核公开的硬件或操作系统指标.

当前版本为 [`v1.0.1`](https://github.com/prometheus/node_exporter/tree/v1.0.1)其代码组织结构很简单:

```text
- collector/: 该包下主要定义 Collector,并对定义的 Collector 进行注册
- https/: v1.0.0 新增内容,主要为支持 https 请求
- vendor/: node_exporter 的依赖包
- go.mod: module 包管理的信息
- go.sum: module 包管理的信息
- node_exporter.go: node_exporter 主程序文件
```

在使用过程中可能用到的目录或文件:

```text
- docs/: 一些 prometheus 规则和与 kube-prometheus 中 node_exporter 相关的 libsonnet 文件
- examples/: 一些服务启动脚本
- Dockerfile: 构建 docker 镜像
VERSION: 记录版本信息
```

## `node_exporter.go`

首先来看 node_exporter 主要程序文件 `node_exporter.go`.

### 导包

```go
import (
    "fmt"
    "net/http"
    _ "net/http/pprof"
    "os"
    "sort"  
    "github.com/prometheus/common/promlog"
    "github.com/prometheus/common/promlog/flag"
    "github.com/go-kit/kit/log" // v1.0.0 新增日志输出
    "github.com/go-kit/kit/log/level" // 日志级别相关
    "github.com/prometheus/client_golang/prometheus" // prometheus 客户端库
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "github.com/prometheus/common/version"
    "github.com/prometheus/node_exporter/collector" // node_exporter 中定义的 Collector
    "github.com/prometheus/node_exporter/https" // v1.0.0 新增支持 https 请求
    kingpin "gopkg.in/alecthomas/kingpin.v2" // 用于定义标志.如 `--help`, `--log.level` 等标志
)
```

首先我们要知道 Go 导包过程中做了哪些事情,参见 {% go-study-notes-package %} 导入包小节.这里再赘述一遍.

> 在执行 `main` 包的 `main` 函数之前,Go 程序先对整个程序的包进行初始化.包内的源码文件都可以定义一到多个初始化函数,编译器首先确保完成所有全局变量初始化,然后开始执行 `init()` 初始化函数,直到这些全部结束后,运行时才进入 `main.main` 入口函数.

因此以上包及其引用包中的常量,全局变量会依次被初始化,`init()` 函数会被执行.

这里需要注意的是 `"github.com/prometheus/client_golang/prometheus"` 包中几乎每个文件的 `init()` 函数都会调用 `registerCollector(collector string, isDefaultEnabled bool, factory func() (Collector, error))` 对该文件中定义的 `Collector` 进行注册.

```go
// collector/collector.go
const (
    defaultEnabled  = true
    defaultDisabled = false
)

var (
    // key: Collector 名称,value: 对应 Collector 工厂函数
    factories        = make(map[string]func(logger log.Logger) (Collector, error))
    // key: Collector 名称,value: 对应 Collector 是否启用
    collectorState   = make(map[string]*bool)
    // key: Collector 名称,value: 对应 Collector 是否明确启用或禁用
    forcedCollectors = map[string]bool{} // collectors which have been explicitly enabled or disabled
)

// collector/collector.go#L58
func registerCollector(collector string, isDefaultEnabled bool, factory func(logger log.Logger) (Collector, error)) {
    var helpDefaultState string
    if isDefaultEnabled {
       helpDefaultState = "enabled"
    } else {
       helpDefaultState = "disabled"
    }

    // 注册 Collector 的标志名称,以 `collector.` 为前缀
    flagName := fmt.Sprintf("collector.%s", collector)
    // 注册 Collector 标志的帮助信息
    flagHelp := fmt.Sprintf("Enable the %s collector (default: %s).", collector, helpDefaultState)
    // 注册标志的默认值
    defaultValue := fmt.Sprintf("%v", isDefaultEnabled)

    // 是否启用 `--collector.<collector>` 标志
    flag := kingpin.Flag(flagName, flagHelp).Default(defaultValue).Action(collectorFlagAction(collector)).Bool()

    // collectorState 保存 Collector 启用/禁用状态
    collectorState[collector] = flag

    // factories 保存 Collector 对应的工厂函数
    factories[collector] = factory
}
```

```go
// collector/meminfo.go
import (
    "fmt"
    "strings"

    "github.com/go-kit/kit/log"
    "github.com/go-kit/kit/log/level"
    "github.com/prometheus/client_golang/prometheus"
)

const (
    memInfoSubsystem = "memory"
)

type meminfoCollector struct {
    logger log.Logger
}
func init() {
    // 将定义的 Collector 进行注册/启用
    registerCollector("meminfo", defaultEnabled, NewMeminfoCollector)
}

func NewMeminfoCollector(logger log.Logger) (Collector, error) {
    return &meminfoCollector{logger}, nil
}

// meminfoCollector 实现了 Collector 接口,该函数应该在数据指标更新时调动.
func (c *meminfoCollector) Update(ch chan<- prometheus.Metric) error {
    var metricType prometheus.ValueType // prometheus 的数据类型,可选值有 CounterValue(1),GaugeValue(2),UntypedValue(3)
    memInfo, err := c.getMemInfo() // 获取 内存信息,不再展开
    if err != nil {
       return fmt.Errorf("couldn't get meminfo: %s", err)
    }
    level.Debug(c.logger).Log("msg", "Set node_mem", "memInfo", memInfo)
    // 对其中数据进行遍历
    for k, v := range memInfo {
        // 判断数据类型,并将对应的数据与数据类型组成的 `constMetric` 发送到 `prometheus.Metric` 类型的管道中
        if strings.HasSuffix(k, "_total") {
            metricType = prometheus.CounterValue
        } else {
            metricType = prometheus.GaugeValue
        }
        // `prometheus.MustNewConstMetric()` 返回 `prometheus.constMetric` 对象,由描述信息,指标类型,指标值构成.如下指标
        // # HELP go_info Information about the Go environment.
        // # TYPE go_info gauge
        // go_info{version="go1.14.4"} 1
        ch <- prometheus.MustNewConstMetric(
            prometheus.NewDesc(
                // 数据指标的描述信息由数据指标的名称,帮助信息(HELP xxx),指标变量标签(version),指标常量标签构成
                prometheus.BuildFQName(namespace, memInfoSubsystem, k),
                fmt.Sprintf("Memory information field %s.", k),
                nil, nil,
            ),
            metricType, v,
        )
    }
    return nil
}
```

### `main()` 入口函数

```go
func main(){
    // 使用 kingpin 定义了众多标志
    var (
        listenAddress = kingpin.Flag(
            "web.listen-address",
            "Address on which to expose metrics and web interface.",
        ).Default(":9100").String()
        metricsPath = kingpin.Flag(
            "web.telemetry-path",
            "Path under which to expose metrics.",
        ).Default("/metrics").String()
        disableExporterMetrics = kingpin.Flag(
            "web.disable-exporter-metrics",
            "Exclude metrics about the exporter itself (promhttp_*, process_*, go_*).",
        ).Bool()
        maxRequests = kingpin.Flag(
            "web.max-requests",
            "Maximum number of parallel scrape requests. Use 0 to disable.",
        ).Default("40").Int()
        disableDefaultCollectors = kingpin.Flag(
            "collector.disable-defaults",
            "Set all collectors to disabled by default.",
        ).Default("false").Bool()
        configFile = kingpin.Flag(
            "web.config",
            "[EXPERIMENTAL] Path to config yaml file that can enable TLS or authentication.",
        ).Default("").String()
    )
    promlogConfig := &promlog.Config{}
    flag.AddFlags(kingpin.CommandLine, promlogConfig)

    // 定义 Version 的输出信息,HelpFlag 的短字段
    kingpin.Version(version.Print("node_exporter"))
    kingpin.HelpFlag.Short('h')
    kingpin.Parse()

    // 定义日志记录器
    logger := promlog.New(promlogConfig)
    // 如果指定了 `collector.disable-defaults` 选项,则调用 collector.DisableDefaultCollectors() 函数
    if *disableDefaultCollectors {
        collector.DisableDefaultCollectors()
    }

    level.Info(logger).Log("msg", "Starting node_exporter", "version", version.Info())
    level.Info(logger).Log("msg", "Build context", "build_context", version.BuildContext())

    // 处理 `metricsPath` 请求,默认为 `/metrics`.后续着重分析一下 newHandler() 函数
    http.Handle(*metricsPath, newHandler(!*disableExporterMetrics, *maxRequests, logger))
    // 处理 `/` 请求
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`<html>
            <head><title>Node Exporter</title></head>
            <body>
            <h1>Node Exporter</h1>
            <p><a href="` + *metricsPath + `">Metrics</a></p>
            </body>
            </html>`))
    })

    level.Info(logger).Log("msg", "Listening on", "address", *listenAddress)
    // 定义 http server 及其监听端口.
    server := &http.Server{Addr: *listenAddress}
    // 调用使用 https.Listen() 启动服务.该方法会根据是否指定 `--web.config` 判断是否启用 https 服务
    if err := https.Listen(server, *configFile, logger); err != nil {
        level.Error(logger).Log("err", err)
        os.Exit(1)
    }
```

```go
// collector/collector.go#L84
// 该函数对 collectorState 中保存的 Collector 进行遍历,将不在 forcedCollectors 中的 Collector 设置为 false.(不启用)
// 仅保留一些必须启用的 Collector
func DisableDefaultCollectors() {
    for c := range collectorState {
        if _, ok := forcedCollectors[c]; !ok {
            *collectorState[c] = false
        }
    }
}
```

```go
// https/tls_config.go#L168
func Listen(server *http.Server, tlsConfigPath string, logger log.Logger) error {
    if tlsConfigPath == "" {
        level.Info(logger).Log("msg", "TLS is disabled and it cannot be enabled on the fly.", "http2", false)
        // 调用了 net/http 包的 ListenAndServe() 函数(net/http/server.go#L2813)
        return server.ListenAndServe()
    }
    // ....
    }


}
```

### `newHandler()` 函数

```go
// node.exporter.go#L49
// 返回了处理 http 请求的 handler,当请求到来时,调用 handler.ServeHTTP(ResponseWriter, *Request) 处理请求,并写入响应
func newHandler(includeExporterMetrics bool, maxRequests int, logger log.Logger) *handler {
    h := &handler{
       exporterMetricsRegistry: prometheus.NewRegistry(),
       includeExporterMetrics:  includeExporterMetrics,
       maxRequests:             maxRequests,
       logger:                  logger,
    }
    // 如果没有指定 `--web.disable-exporter-metrics` 选项,则传入 `includeExporterMetrics` 参数为 true
    if h.includeExporterMetrics {
        h.exporterMetricsRegistry.MustRegister(
            // 注册了 processCollector,goCollector.在查看 `/metrics` 时也会看到相关数据指标.如
            // process_cpu_seconds_total,process_max_fds
            // go_goroutines,go_info 等
            prometheus.NewProcessCollector(prometheus.ProcessCollectorOpts{}),
            prometheus.NewGoCollector(),
       )
    }
    // 调用 innerHandler 创建 innerHandler
    if innerHandler, err := h.innerHandler(); err != nil {
        panic(fmt.Sprintf("Couldn't create metrics handler: %s", err))
    } else {
        h.unfilteredHandler = innerHandler
    }
    return h
}
```

```go
// ServeHTTP implements http.Handler.
// 处理请求的方法
func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 开启 debug 日志级别后发现,filters 为 "unsupported value type"
    filters := r.URL.Query()["collect[]"]
    level.Debug(h.logger).Log("msg", "collect query:", "filters", filters)

    if len(filters) == 0 {
       // No filters, use the prepared unfiltered handler.
       h.unfilteredHandler.ServeHTTP(w, r)
       return
    }
    // To serve filtered metrics, we create a filtering handler on the fly.
    // 动态创建 filteredHandler
    filteredHandler, err := h.innerHandler(filters...)
    if err != nil {
       level.Warn(h.logger).Log("msg", "Couldn't create filtered metrics handler:", "err", err)
       w.WriteHeader(http.StatusBadRequest)
       w.Write([]byte(fmt.Sprintf("Couldn't create filtered metrics handler: %s", err)))
       return
    }
    filteredHandler.ServeHTTP(w, r)
}

// innerHandler is used to create both the one unfiltered http.Handler to be
// wrapped by the outer handler and also the filtered handlers created on the
// fly. The former is accomplished by calling innerHandler without any arguments
// (in which case it will log all the collectors enabled via command-line
// flags).
func (h *handler) innerHandler(filters ...string) (http.Handler, error) {
    // 根据 filters 指定字符串 Collector,用于创建 `collector.NodeCollector`
    nc, err := collector.NewNodeCollector(h.logger, filters...)
    if err != nil {
        return nil, fmt.Errorf("couldn't create collector: %s", err)
    }

    // Only log the creation of an unfiltered handler, which should happen
    // only once upon startup.
    // 这里将在 node_exporter 启动时打印启用的 Collector 相关日志
    if len(filters) == 0 {
        level.Info(h.logger).Log("msg", "Enabled collectors")
        collectors := []string{}
        for n := range nc.Collectors {
            collectors = append(collectors, n)
        }
        sort.Strings(collectors)
        for _, c := range collectors {
            level.Info(h.logger).Log("collector", c)
        }
    }

    r := prometheus.NewRegistry()
    r.MustRegister(version.NewCollector("node_exporter"))
    if err := r.Register(nc); err != nil {
       return nil, fmt.Errorf("couldn't register node collector: %s", err)
    }
    // 这里应该是收集指标数据的过程
    handler := promhttp.HandlerFor(
        // 传入的 Gatherers 内部保存了实现 Gatherer 接口的 Registry,将来 Registry 会在 Gather() 方法中通过其中保存的 Collector 对象调用 Collect() 方法收集数据指标.
        // 同时 Collector 对象的 Collect() 方法内部会调用 Collector.Update() 方法获取数据指标相关信息
        prometheus.Gatherers{h.exporterMetricsRegistry, r},
        promhttp.HandlerOpts{
            ErrorHandling:       promhttp.ContinueOnError,
            MaxRequestsInFlight: h.maxRequests,
            Registry:            h.exporterMetricsRegistry,
        },
    )
    // 如果没有指定 --web.disable-exporter-metrics` 选项,则 `h.includeExporterMetrics = true`
    if h.includeExporterMetrics {
        // Note that we have to use h.exporterMetricsRegistry here to
        // use the same promhttp metrics for all expositions.
        // 调用 InstrumentMetricHandler 方法,对 promhttp_metric_handler_requests_total,promhttp_metric_handler_requests_in_flight 值进行设置
        handler = promhttp.InstrumentMetricHandler(
            h.exporterMetricsRegistry, handler,
        )
    }
    return handler, nil
}
```

## 自定义 metrics

以上,我们只需要在 `collector` 包中创建如下内容,再通过 `node_exporter` 帮我们实现自定义 metrics.

1. 自定义 `customCollector` 结构体,实现 Collector 接口,也就是实现 `Update(ch chan<- prometheus.Metric)` 方法.在 `Update()` 方法中,要借助 `prometheus.MustNewConst<Histogram,Metric,Summary>` 等方法创建 `prometheus.Metric` 接口的实现对象,并将其传入管道 `ch`.
2. 在 `init()` 函数中调用 `registerCollector()` 函数注册自定义 customCollector 结构体.

代码示例如下:

```go
package collector

import (
    "fmt"

    "github.com/go-kit/kit/log"
    "github.com/go-kit/kit/log/level"
    "github.com/prometheus/client_golang/prometheus"

    kingpin "gopkg.in/alecthomas/kingpin.v2"
)

const (
    // 定义自定义数据指标的子系统名称
    customMetricsSubsystem = "metrics"
)

// 定义 customMetricsCollector 结构体
type customMetricsCollector struct {
    logger log.Logger
    //...
}

// 定义 customMetricsCollector 的工厂函数,后续传入 registerCollector() 函数中,以便创建 customMetricsCollector 对象
func NewCustomMetricsCollector(logger log.Logger) (Collector, error) {
    return &customMetricsCollector{
        logger: logger,
    }, nil
}

// 实现 Update() 函数,以便在处理请求时被 Collector.Collect() 调用
func (c *customMetricsCollector) Update(ch chan<- prometheus.Metric) error {
    var metricType prometheus.ValueType
    var value = 1.1
    metricType = prometheus.CounterValue
    level.Debug(c.logger).Log("msg", "Set custom_metrics", "metrics", value)

    // 通过 `prometheus.MustNewConstMetric` 创建自定义 `prometheus.Metric` 接口对象 `prometheus.constMetric`,并将其传入`prometheus.Metric` 管道
    ch <- prometheus.MustNewConstMetric(
        // 需要传入 Metric 实现对象的描述信息,对象数据类型,值
        prometheus.NewDesc(
            // 描述信息包括 数据指标名称(由 `BuildFQName()`函数组合而成),帮助信息,变量标签,常量标签
            prometheus.BuildFQName(namespace, customMetricsSubsystem, "custom_metrics"),
            fmt.Sprintf("Custom metrics field %s.", "custom_metrics"),
            nil, nil,
        ),
        metricType, value,
    )
    return nil
}

func init() {
    // 在该函数中调用 registerCollector() 函数,注册自定义 customMetricsCollector
    registerCollector("custom_metrics", defaultEnabled, NewCustomMetricsCollector)
}
```

编译后,执行 `./node_exporter` 发现 `/metrics` 中包含自定义数据指标.

```text
# HELP node_metrics_custom_metrics Custom metrics field custom_metrics.
# TYPE node_metrics_custom_metrics counter
node_metrics_custom_metrics 1.1
```
