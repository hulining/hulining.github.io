---
title: istio可扩展的策略和遥测
tags:
  - istio
  - 读书笔记
categories:
  - 云原生应用
  - istio
abbrlink: 63298
description: 本文章为《云原生服务网格Istio：原理、实践、架构与源码解析》第 4 章读书笔记.
---

> 应用场景

对服务进行全面管理,除了需要具备服务治理功能,还需要知道服务到底运行得怎么样,有没有问题,哪里有问题,这一般是 APM(Application Performance Management,应用性能管理) 的职能,其中涉及采集数据,存储数据和检索数据.

> 实现方式

Istio 将 Envoy 的遥测和策略功能提取出来,放到一个服务端组件 Mixer 上,在逻辑上将 Envoy 和各种遥测数据的收集解耦,并将 Envoy 和真正的遥测后端解耦.Envoy 和控制面组件 Mixer 的单条连接

基于 Mixer Adapter 提供的扩展机制,可以做到在遥测和策略执行时对业务代码的无侵入,解耦数据面 Envoy 和遥测与策略执行的后端服务,并开发自己的 Adapter,提供扩展和定制的能力,提供满足用户特定场景的服务运行监控和控制

# 原理

## 工作流程
简单来说,该流程主要有两步:
1. Envoy 生成数据并将数据上报给 Mixer
2. Mixer 调用对应的服务后端处理收到的数据

每个经过 Envoy 的请求都会调用 Mixer上报数据,Mixer将上报的这些数据作为策略和遥测报告的一部
分发送出来,并转换为对后端服务的调用

## 属性

> 属性定义

Envoy 上报的数据在 Istio 中被称为属性(Attribute).严格来讲,在以上 Mixer 处理流程的两个阶段,从Envoy到Mixer及从Mixer到后端服务,处理的
对象都是属性

> 属性表达式

```
destination_service: destination.service
response_code: response.code
destination_version: destination.labels["version"] | "unknown"
```
更多属性表达式见 <云原生服务网格 istio> 表 4-1

> Mixer的配置模型

Istio 主要通过 `Handler`(业务处理),`Instance`(数据定义)和 `Rule`(关联规则)这三个资源对象来描述对 Adapter 的配置

- `Handler`

Handler 描述定义的 Adapters 及其配置,不同的 Adapter 有不同的配置.Handler 是 Adapter 定义的模板的实现,通过给模板上的参数赋值来进行实例化

示例如下
```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: stdio
metadata:
  name: handler
  namespace: istio-system
spec:
  outputAsJson: true
---
apiVersion: "config.istio.io/v1alpha2"
kind: prometheus
metadata:
  name: handler
  namespace: istio-system
spec:
  metrics:
  - name: requests_total
    instance_name: requestcount.metric.istio-system
    kind: COUNTER
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - request_protocol
    - response_code
    - connection_security_policy
  - name: request_duration_seconds
    instance_name: requestduration.metric.istio-system
    kind: DISTRIBUTION
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - request_protocol
    - response_code
    - connection_security_policy
    buckets:
      explicit_buckets:
        bounds: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
```

- `Instance`

Instance 定义了 Adapter 要处理的数据对象,通过模板为 Adapter 提供对元数据的定义.Mixer 通过 Instance 把来自代理的属性拆分并分发给不通的适配器

示例如下:
```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: logentry
metadata:
  name: accesslog
  namespace: istio-system
spec:
  severity: '"Info"'
  timestamp: request.time
  variables:
    sourceIp: source.ip | ip("0.0.0.0")
    sourceApp: source.labels["app"] | ""
    sourcePrincipal: source.principal | ""
    sourceName: source.name | ""
    sourceWorkload: source.workload.name | ""
    sourceNamespace: source.namespace | ""
    sourceOwner: source.owner | ""
    destinationApp: destination.labels["app"] | ""
    destinationIp: destination.ip | ip("0.0.0.0")
    destinationServiceHost: destination.service.host | ""
    destinationWorkload: destination.workload.name | ""
    destinationName: destination.name | ""
    destinationNamespace: destination.namespace | ""
    destinationOwner: destination.owner | ""
    destinationPrincipal: destination.principal | ""
    apiClaims: request.auth.raw_claims | ""
    apiKey: request.api_key | request.headers["x-api-key"] | ""
    protocol: request.scheme | context.protocol | "http"
    method: request.method | ""
    url: request.path | ""
    responseCode: response.code | 0
    responseSize: response.size | 0
    requestSize: request.size | 0
    requestId: request.headers["x-request-id"] | ""
    clientTraceId: request.headers["x-client-trace-id"] | ""
    latency: response.duration | "0ms"
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
    requestedServerName: connection.requested_server_name | ""
    userAgent: request.useragent | ""
    responseTimestamp: response.time
    receivedBytes: request.total_size | 0
    sentBytes: response.total_size | 0
    referer: request.referer | ""
    httpAuthority: request.headers[":authority"] | request.host | ""
    xForwardedFor: request.headers["x-forwarded-for"] | "0.0.0.0"
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
  monitored_resource_type: '"global"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: requestcount
  namespace: istio-system
spec:
  value: "1"
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["app"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["app"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.host | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    request_protocol: api.protocol | context.protocol | "unknown"
    response_code: response.code | 200
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
```

- `Rule`

Rule 配置了一组规则,告诉 Mixer 有哪个 Instance 在什么时候被发送给哪个 Handler 来处理,一般包括一个匹配的表达式和执行动作(action)匹配表达式控制在什么时候调用 Adapter,在 Action 里配置 Adapter 和 Instance 的名称

字段如下

字段 | 是否必选 | 解释
:---: | :---: | :---:
match | 是 | 表示匹配条件,如果未定义条件,则判定为总是匹配
actions | 是 | 表示满足条件后执行的动作,是一个数组.包含 handler 和 instance 两个字段,用于指定 handler 和 instance 的名称(必须是全名,格式一般为 `<name.kind>`)

示例如下
```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: stdio
  namespace: istio-system
spec:
  match: context.protocol == "http" || context.protocol == "grpc"
  actions:
  - handler: handler.stdio
    instances:
    - accesslog.logentry
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: promhttp
  namespace: istio-system
spec:
  match: context.protocol == "http" || context.protocol == "grpc"
  actions:
  - handler: handler.prometheus
    instances:
    - requestcount.metric
    - requestduration.metric
    - requestsize.metric
    - responsesize.metric
```

# 遥测适配器配置

## Prometheus 适配器

> 工作流程

1. Envoy 通过 Report 接口上报数据给 Mixer
2. Mixer 根据配置将请求分发给 Prometheus Adapter
3. Prometheus Adapter 通过 HTTP 接口发 布Metric 数据
4. Prometheus 服务作为 Addon 在集群中进行安装,拉取并存储 Metric 数据,提供 Query 接口进行检索
5. 集群内的 Dashboard(如Grafana)通过 Prometheus 的检索 API 访问 Metric 数据

> handler 配置定义

handler 配置中最主要的字段是 `metrics`,用于在 Prometheus 中定义 metrics.它是一个数组,每个元素都具有如下属性

- `name`: metric 的名称
- `instance_name`: instance 的全名称,格式为 `<instance_name.kind.namespace>`
- `kind`: 定义指标类型,请求计数类型为 `COUNTER`,请求耗时类型为 `DISTRIBUTION`.DISTRIBUTION 类型的指标可以定义其 buckets
- `label_names`: 定义指标的标签,一般与 instance 中维度相同

> instance 配置定义

instance 配置中最主要的字段是 `dimensions` 和 `value`,分别用于记录数据的维度及对应的值.这两个字段均支持属性表达式.其中维度中的 key 多用于 prometheus-metrics 的标签

> rule 配置定义

Rule 可以将 Handler 和 Instance建立关系,最主要的字段是 `match` 和 `actions`,分别用于匹配规则及匹配后的动作