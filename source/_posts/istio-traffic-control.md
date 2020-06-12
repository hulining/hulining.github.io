---
title: istio非侵入的流量治理
date: 2020/04/03
tags:
  - 读书笔记
  - istio
categories:
  - istio
abbrlink: 6366
description: 本文章为《云原生服务网格Istio：原理、实践、架构与源码解析》第 3 章读书笔记.
---

## 原理

### 目标

以基础设施的方式提供给用户非侵入的流量治理能力,用户只需关注自己的业务逻辑开发,无须关注服务访问管理

### 流程

1. 管理员通过命令行或 API 创建流量规则
2. Pilot 将流量规则转换为 Envoy 的标准格式,并下发给 Envoy(转向数据面)
3. Envoy 拦截 Pod 上本地容器的 Inbound 流量和 Outbound 流量
4. 在流量经过 Envoy 时执行对应的流量规则,对流量进行治理

### 应用场景和功能

- 负载均衡

服务注册: 各服务将服务名和服务实例的对应信息注册到服务注册中心
服务发现: 在客户端发起服务访问时,以同步或者异步的方式从服务注册中心获取服务对应的实例列表
负载均衡: 据配置的负载均衡算法从实例列表中选择一个服务实例.目前支持的负载均衡算法有轮询,随机和最小连接数算法

- 服务熔断

故障检测和处理逻辑,防止临时故障或意外导致系统整体不可用.最典型的场景是防止网络和服务调用故障级联发生,限制故障的影响范围,防止故障蔓延导致系统整体性能下降或雪崩

- 故障注入

主要用于测试其健壮性和应对故障的能力,例如异常处理,故障恢复等Istio 的故障注入是在网格中对特定的应用层协议进行故障注入,可以模拟出应用的故障场景.

如注入 HTTP Code 503(服务端异常),请求延时(模拟响应慢)

- 灰度发布

新老版本同时在线,新版本只切分少量流量出来,在确认新版本没有问题后,再逐步加大流量比例

其中灰度发布主要有金丝雀发布,蓝绿发布,AB 测试 3 种方式

- 服务访问入口

Istio 中通过 Ingress Gateway 访问网格内的服务,做四层到六层的端口,TLS配置等基本功能,VirtualService则定义七层路由等丰富内容

- 外部接入服务治理

Istio 通过 ServiceEntry 资源对象将网格外的服务注册到网格上,然后像对网格内的普通服务一样对网格外的服务访问进行治理.有时需
要有一个专门的 Egress Gateway 来提供统一的出口网关

## VirtualService 路由规则配置

VirtualService 定义了对特定目标服务的一组流量规则,它将满足条件的流量都转发到对应的服务后端,这个服务后端可以是一个服务,也可以是在 DestinationRule 中定义的服务的子集

### HTTP路由(HTTPRoute)

满足 `HTTPMatchRequest` 条件的流量都被路由到 `HTTPRouteDestination`,执行重定向 `HTTPRedirect`,重写 `HTTPRewrite`,重试 `HTTPRetry`,故障注入 `HTTPFaultInjection`,跨站 `CorsPolicy` 等策略

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
spec:
  hosts:
  - forecast
  http:
  - match:
    - headers:
        location:
          exact: north
    route:
    - destination:
        host: forecast
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

#### HTTPMatchRequest 定义匹配规则

关键字: `match`,支持如下定义方式

- `uri,scheme,method,authority,headers`: 支持 `exact(精确),prefix(前缀),regex(正则)` 方式的匹配
- `port`: 端口
- `sourceLabels`: 支持以特定的标签进行匹配,匹配满足该标签的 Pod 发送出来的流量

#### HTTPRouteDestination 定义路由目标

关键字: `route`,支持如下定义方式

- `destination`: 表示请求的目标,最终的流量要被送到这个目标上.通过 `host,subnet,port` 三个属性来描述
  - `host`: 必选字段,表示 istio 中注册的服务名.建议写全服务的全名 `<hostname>.<namspace>.svc.cluster.local`
  - `subset`: 表示 host 上定义的一个子集,用于表示不同版本的服务
- `weight`: 表示请求流量分配的比例,当指定多个 destination 时需要配置,且weight 总和要求是 100
- `headers`: 对请求和响应的请求头进行修改

#### HTTPRedirect 定义重定向规则

关键字: `redirect`,需要设置重定向的目标 `uri`.这里需要注意的是重定向的 `uri` 会替换原请求中完整的 uri 路径

#### HTTPRewrite 定义 url 重写规则

关键字: `rewrite`,需要设置重写的 `uri`.这里需要注意,与重定向不同,重写的 `uri` 只会会替换原请求中匹配部分的 uri

#### HTTPRetry 定义请求失败时重试策略

关键字 `retries`,需要设置重试次数(attempts),超时(perTryTimeout),重试条件(retryOn)等.

- `attempts`: 必选字段,定义重试次数
- `perTryTimeout`: 每次重试的超时时间,单位可以是 ms,s,m,h
- `retryOn`: 进行重试的条件,以逗号分割.重试条件包含如下:
  - 5xx: 上游服务返回5xx应答,或没有返回时
  - gateway-error: 只对502,503和504应答码进行重试
  - connect-failure：在连接上游服务失败时重试
  - retriable-4xx：在上游服务返回可重试的4xx应答码时执行重试
  - refused-stream：在上游服务使用REFUSED_STREAM错误码重置时执行重试
  - cancelled：在gRPC应答的Header中状态码是cancelled时执行重试
  - deadline-exceeded：在gRPC应答的Header中状态码是deadline-exceeded时执行重试
  - internal：在gRPC应答的Header中状态码是internal时执行重试
  - resource-exhausted：在gRPC应答的Header中状态码是resource-exhausted时执行重试
  - unavailable：在gRPC应答的Header中状态码是unavailable时执行重试

#### Mirror 流量镜像

关键字 `mirror`,指在将流量转发到原目标地址的同时将流量给另外一个目标地址镜像一份,用于真实流量请求

#### HTTPFaultInjection 故障注入

关键字 `fault`,支持 `delay` 和 `abort` 两个字段配置延时和中止两种故障

- `delay`: 用于延迟故障注入,主要设置如下两个字段
  - `fixedDelay`: 必选字段,表示延迟时间,单位可以是 ms,s,m,h
  - `percentage`: 延迟故障作用的请求比例
- `abort`: 用于中止故障注入,主要设置如下两个字段
  - `httpStatus`: 必选字段,中止的 HTTP 状态码
  - `percentage`: 延迟故障作用的请求比例

#### CorsPolicy 跨域资源共享

关键字 `corsPolicy`,用于通过跨域资源共享(Cross
Origin Resource Sharing)机制可允许 Web 应用服务器进行跨域访问控制,使跨域数据传输安全进行

- `allowMethods`: 允许访问资源的 HTTP 方法列表,内容被序列化到 Access-Control-Allow-Methods 的
Header上
- `allowHeaders`: 请求资源的 HTTP Header 列表,内容被序列化到 Access-Control-Allow-Headers 的 Header 上
- `exposeHeaders`: 浏览器允许访问的 HTTP Header 的白名单,内容被序列化到 Access-Control-Expose-Headers 的 Header 上
- `maxAge`: 请求缓存的时长,被转化为 Access-Control-Max-Age 的 Header
- `allowCredentials`: 是否允许服务调用方使用凭据发起实际请求,被转化为 Access-Control-Allow-Credentials 的 Header

### TLS路由(TLSRoute)

满足 `TLSMatchAttributes` 条件的 TLS 和 HTTPS 流量都被路由到 `RouteDestination`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: total-weather-tls
spec:
  hosts:
  - "*.weather.com"
  gateways:
  - "ingress-gateway"
  tls:
  - match:
    - port: 443
      sniHosts:
      - frontend.weather.com
    route:
    - destination:
        host: frontend
        subset: v2
  - match:
    - port: 443
      sniHosts:
      - recommendation.weather.com
    route:
    - destination:
        host: recommendation
```

#### TLSMatchAttributes TLS匹配规则

关键字 `match`,表示 TLS的匹配条件,支持如下定义方式:

- `sniHosts`: 必选字段，用来匹配 TLS 请求的 SNI ,SNI 的值必须是 VirtualService 的 hosts 的子集
- `port`: 访问的目标端口
- `destinationSubnets`: 目标IP地址匹配的IP子网
- `sourceLabels`: 匹配来源负载的标签

一般用法是匹配 `sniHosts` 和 `port`

#### RouteDestination 定义路由目标

关键字: `route`,包含如下两个属性,用法和约束同 `HTTPRouteDestination` 的对应字段

- `destination`: 表示请求的目标,最终的流量要被送到这个目标上.通过 `host,subnet,port` 三个属性来描述
  - `host`: 必选字段,表示 istio 中注册的服务名.建议写全服务的全名 `<hostname>.<namspace>.svc.cluster.local`
  - `subset`: 表示 host 上定义的一个子集,用于表示不同版本的服务
- `weight`: 表示请求流量分配的比例,当指定多个 destination 时需要配置,且weight 总和要求是 100

### TCP路由(TCPRoute)

所有不满足以上HTTP和TLS条件的流量都会应用TCP流量规则

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast
spec:
  hosts:
  - forecast
  tcp:
  - match:
    port: 23003
    route:
    - destination:
        host: inner-forecast
        port:
          number: 3003
```

> 满足 TCP 特有的 4 层匹配规则 `L4MatchAttribute` 条件的流量转发到对应的目标后端

#### L4MatchAttribute 四层匹配规则

关键字 `match`,支持以下匹配属性

- `destinationSubnets`: 目标IP地址匹配的 IP 子网
- `port`: 访问的目标端口
- `sourceLabels`：源工作负载标签

#### RouteDestination 目标后端 与 TLS 定义方式相同

### 三种协议路由规则汇总

协议 | 路由规则 | 支持的流量匹配条件 | 条件属性 | 支持的流量操作 | 目标路由定义 | 目标路由属性
:---: | :---: | :---: | :---: | :---: | :---: | :---:
HTTP | HTTPRoute | HTTPMatchRequest |  uri,scheme,method,port,sourceLabels | route,redirect,rewrite,retry,timeout,faultInjection,corsPolicy | HTTPRouteDestination | destination,weight,headers
TLS | TLSRoute | TLSMatchAttribute | sniHost,port,destinationSubnets,sourceLabels | route | RouteDestination | destination,weight
TCP | TCPRoute | L4MatchAttribute | destinationSubnets,port,sourceLabels | route | RouteDestination | destination,weight

## DestinationRule 目标规则配置

DestinationRule 定义了满足路由规则的流量到达后端后的访问策略.在 Istio 中可以配置目标服务的负载均衡策略,连接池大小,异常实例驱除规则等功能.

```yaml
apiVersion: networking.istio.io/v1alph3
kind: DestinationRule
metadata:
  name: forecast
spec:
  host: forecast
  subnet:
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      loadBalancer:
        simple: RANDON
```

DestinationRule 重要属性如下:

- `host`: 必选字段,表示使用规则的对象,取值是在服务注册中心注册的服务名.host如果取短域名,则会根据规则所在的命名空间进行解析.所以尽量填写全名
- `trafficPolicy`: 规则内容的定义,包括负载均衡,连接池策略,异常点检查等
- `PortTrafficPolicy`: 需设定关键字 port 表示流量策略要应用的服务端口,其余和 TrafficPolicy 没有很大差别
- `subsets`: 定义服务的一个子集,经常用来定义一个服务版本.包含如下重要属性
  - `name`: 必选字段,subnet 名称
  - `labels`: Subset 上的标签,通过一组标签定义了属于这个 Subset 的服务实例
- `exportTo`: 控制 DestinationRule 跨名称空间的可见性.可选值为 "." 或 "*",表示当前名称空间或所有名称空间

### TrafficPolicy 流量策略

流量策略包含以下 4 种重要配置

关键字 | 类型 | 描述
:---: | :---: | :---:
loadBalancer | LoadBalancerSettings | 描述服务的负载均衡算法
connectionPool | ConnectionPoolSettings | 描述服务的连接池配置
outlierDetection | OutlierDetection | 描述服务的异常点检查
tls | TLSSettings | 描述服务的TLS连接设置

#### loadBalancer 负载均衡设置

负载均衡设置支持 `simple(简单)` 和 `consistentHash(一致性 hash)` 两个字段.其中,一致性哈希是一种高级的负载均衡策略,只对 HTTP 有效

- `simple` 字段定义了 `ROUND_ROBIN(轮询,默认)`,`LEAST_CONN(最小连接)`,`RANDOM(随机)`,`PASSTHROUGH(直接转发)` 等几种负载均衡算法
- `consistentHash` 定义了 `httpHeaderName(请求头)`,`httpCookie(cookie)`,`useSourceIp(源 IP)` 等几种负载均衡算法.还提供 `minimumRingSize(哈希环上虚拟节点数的最小值)` 属性,用于增加 hash 节点数量(节点数越多则负载均衡越精细)

#### connectionPool 连接池设置

Istio 连接池管理支持 `tcp(TCP流量)` 和 `http(HTTP流量)` 两个字段.

`tcp` 字段定义了如下属性:

- `maxConnections`: 表示为上游服务的所有实例建立的最大连接数,默认1024
- `connectTimeout`: TCP连接超时,表示主机网络连接超时
- `tcpKeepalive`: 设置TCP keepalives(TCP keepalive).它包含三个字段,如下
  - `probes` 表示多少次探测没有应答则断开,默认是9
  - `time` 表示在发送探测前连接空闲了多长时间,默认2h
  - `interval` 表示探测间隔,默认 75s

`http` 字段定义了如下属性:

- `http1MaxPendingRequests`: 最大等待 HTTP 请求数,默认1024
- `http2MaxRequests`: 最大请求数,默认1024
- `maxRequestsPerConnection`: 每个连接的最大请求数
- `maxRetries`: 最大重试次数,默认3
- `idleTimeout`: 空闲超时,在多长时间内没有活动请求则关闭连接

#### outlierDetection 异常实例检查

异常点检查就是定期考察被访问的服务实例的工作情况,如果连续出现访问异常,则将服务实例标记为异常并进行隔离,在一段时间内不为其分配流量,待恢复后重新恢复流量.可用于熔断模型

异常实例检查可通过如下字段来控制检查驱逐的逻辑:

- `consecutiveErrors`: 实例被驱逐前的连续错误次数,默认5
- `interval`: 驱逐的时间间隔,默认 10s
- `baseEjectionTime`: 最小驱逐时间,默认30s.实例被驱逐的时间等于这个最小驱逐时间乘以驱逐的次数
- `maxEjectionPercent`: 负载均衡池中可以被驱逐的故障实例的最大比例,默认是10%.避免太多实例被驱逐而导致整体服务能力下降
- `minHealthPercent`: 最小健康比例,当可用实例数的比例小于这个比例时,异常点检查功能将被禁用.默认 50%

#### tls TLS连接设置

istio TLS 连接设置支持以下字段属性:

- `mode`: tls 认证方式.支持如下4种模式
  - `Disable`: 不使用 TLS 认证
  - `SIMPLE`: 单向认证
  - `MUTUAL`: 双向认证.需要应用程序提供客户端证书,在配置中指定证书文件路径
  - `ISTIO_MUTUAL`: 双向认证.证书由Istio自动生成,不用指定证书路径
- `privateKey`: 客户端私钥路径
- `clientCertificate`: 客户端证书路径
- `caCertificates`: 验证服务端证书的 CA 文件路径

## Gateway 服务网关配置

Gateway 在网格边缘接收外部访问,并将流量转发到网格内的服务.Istio 通过 Gateway 将网格内的服务发布成外部可访问的服务,还可以通过 Gateway 配置外部访问的端口,协议及与内部服务的映射关系

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      name: http
      number: 80
      protocol: HTTP
    hosts:
    - weather.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend
spec:
  hosts:
  - frontend
  - weather.com
  gateways:
  - istio.gateway.istio-system.svc.cluster.local
  - mesh
  http:
  - match:
    - headers:
      location:
        exact: north
    route:
    - destionation:
        host: frontend
        subset: v2
  - route:  
    - destionation:
        host: frontend
        subset: v1
```

Gateway 定义了 `selector(标签选择)` 和 `servers(开放的服务列表)` 两个关键必选字段.

### server 后端服务

server 真正定义了服务的访问入口,可通过如下字段来配置多个 server

- `port`: 必选字段,描述服务在哪个端口开放,是对外监听的端口
- `hosts`: 为 gateway 发布的服务地址,是一个 FQDN 域名
- `defaultEndpoint`: 表示流量默认转发的后端
- `tls`: 安全服务接口相关内容.可通过如下字段进行配置
  - `httpsRedirect`: 是否要做 HTTPS 重定向
  - `mode`: TLS 模式,`PASSTHROUGH(不做处理,直接转发,TLS证书相关设置在后端负载),SIMPLE(单向认证),MUTUAL(双向认证),AUTO_PASSTHROUGH`
  - `serverCertificate`: 服务端证书路径,在 SIMPLE,MUTUAL 时必须配置
  - `privateKey`: 服务端密钥路径,在 SIMPLE,MUTUAL 时必须配置
  - `caCertificates`: CA证书路径,在 MUTUAL 时必须配置
  - `credentialName`: Gateway 使用 `credentialName` 从远端的凭据存储(Kubernetes-Secrets)中获取证书和密钥,而不是挂载
  - `subjectAltNames`: SAN 列表
  - `minProtocolVersion`,`maxProtocolVersion`: TLS 协议最小,最大版本
  - `cipherSuites`: 指定加密套件,默认使用 Envoy 支持的加密套件

### Gateway 典型应用

#### 将网格内的 HTTPS 服务发布为 HTTPS 外部访问

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      name: https
      number: 443
      protocol: HTTPS
    hosts:
    - weather.com
    tls:
      mode: PASSTHROUGH
```

这种场景下,后端服务需要配置证书,协议为 HTTPS.istio 只是提供了通道和机制,通过 Gateway 将一个内部的 HTTPS 服务发布出去.

#### 将网格内的 HTTP 服务发布为 HTTPS 外部访问

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      name: https
      number: 443
      protocol: HTTPS
    hosts:
    - weather.com
    tls:
      mode: SIMPLE
      # 指定服务端证书路径,需要将证书文件挂载在 ingressgateway pod 中.可以通过证书文件创建 k8s-secrets 后进行挂载
      serverCertificate: /etc/istio/gateway-weather-certs/tls.crt
      privateKey: /etc/istio/gateway-weather-certs/tls.key
      # 直接指定 k8s-secrets 名称,istio 会自动配置对应的证书及密钥(推荐)
      # credentialName: weather-com-secrets
```

这种场景下,后端服务不需要配置证书,协议为 HTTP.istio 将证书配置在 Gateway 上,并通过 Gateway 将一个内部的 HTTP 服务以 HTTPS 方式发布出去.

#### 将网格内的 HTTP 服务发布为双向 HTTPS 外部访问

在某些场景下,比如调用入口服务的是另一个服务,在服务端需要对客户端进行身份校验,这就需要用到 TLS 的双向认证.应用场景如银行 APP

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      name: https
      number: 443
      protocol: HTTPS
    hosts:
    - weather.com
    tls:
      mode: MUTUAL
      serverCertificate: /etc/istio/gateway-weather-certs/tls.crt
      privateKey: /etc/istio/gateway-weather-certs/tls.key
      # 用于验证客户端证书
      caCertificates: /etc/istio/gateway-weather-certs/ca.crt
```

#### 将网格内的 HTTP 服务发布为 HTTPS 外部访问 和HTTPS 内部访问

此场景与场景2 中 Gateway 的 Manifest 完全相同,我们只需知道 istio 可以透明地给网格内的服务启用双向 TLS,并且自动维护证书和密钥.

网关 Gateway 服务和后端 frontend 在这种场景下的工作机制如下

- frontend 服务自身还是HTTP,不涉及证书密钥的事情
- 流量进入网格前,Gateway 作为 frontend 服务的入口网关,对外提供 HTTPS 的访问.外部访问到的是在 Gateway 上发布的 HTTPS 服务,使用 Gateway 上的配置提供服务端证书和密钥
- 流量进入网格后,Gateway 作为客户端访问 frontend 服务的代理(Envoy,Sidecar),对 frontend 服务发起另一个 HTTPS 请求,使用的是 Citadel 分发和维护的客户端证书和密钥,与 frontend 服务代理(Envoy,Sidecar)的服务端证书和密钥进行双向 TLS 认证和通信.(该过程称为 mTLS 双向认证)

## ServiceEntry 外部服务配置

ServiceEntry 用于将网格外的服务加入网格中,并像网格内的服务一样进行管理,在实现上就是把外部服务加入 istio 的服务发现

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: weather-external
spec:
  hosts:
  - www.weatherdb.com
  ports:
  - name: http
    number: 80
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

ServiceEntry 主要包含如下几个字段

- `hosts`: 必选字段,表示与 ServiceEntry 相关的主机名.当 resolution 被设置为 DN S类型并且没有指定 endpoints 时,这个字段将用作后端的域名来进行路由
- `ports`: 必选字段,表示与外部服务关联的端口
- `resolution`: 必选字段,表示服务发现的模式,有以下3种模式可选
  - `NONE`: 用于当连接的目标地址已经是一个明确 IP 的场景
  - `STATIC`: 用在已经用 endpoints 设置了服务实例的地址场景中,即不用解析
  - `DNS`: 表示用查询环境中的 DNS 进行解析.如果没有设置 endpoints,代理就会使用在`hosts`中指定的 DNS 地址进行解析
- `addresses`: 与服务关联的虚拟IP地址
- `location`: `MESH_EXTERNAL`(网格外部) 或 `MESH_INTERNAL`(网格内部).当和网格外部服务通信时,mTLS 双向认证将被禁用,并且策略只能在客户端执行,不能在服务端执行.对于外部服务,我们不可能注入一个 Sidecar 来进行双向认证等操作
- `endpoints`:表示与网格服务关联的网络地址,可以是一个IP,也可以是一个主机名.包含如下字段
  - `address`: 必选字段,表示网络后端的地址.当 resolution 设置为 DNS 时,可以使用域名
  - `ports`: 端口列表
  - `labels`: 后端标签
  - `weight`: 权重

### ServiceEntry 典型应用

#### 通过配置 ServiceEntry + Egressgateway + VirtualService 访问外部服务

```yaml
# 定义 www.weatherdb.com 外部服务
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: weather-external
spec:
  hosts:
  - www.weatherdb.com
  ports:
  - name: http
    number: 80
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
# 定义 www.weatherdb.com 服务的 egress-gateway 网关出口
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-system
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      name: http
      number: 80
      protocol: HTTP
    hosts:
    - www.weatherdb.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: egress-wearthdb
spec:
  hosts:
  - www.weatherdb.com
  gateways:
  - egress-gateway.istio-system.svc.cluster.local
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: egress-gateway.istio-system.svc.cluster.local
  - match:
    - gateways:
      - egress-gateway.istio-system.svc.cluster.local
      port: 80
    route:
    - destination:
        host: www.weatherdb.com
```

在此场景中,定义了两个 route

- 网格内流量: 这个 route 的 gateways 是 mesh,表示来自网格内的流量在访问 www.weatherdb.com 这个外部地址时,将被路由到 egressgateway 网关上
- 网格外流量: 这个 route 的 gateways 是 egress,表示来自 egress 的流量在访问 www.weatherdb.com 这个外部地址时,将被路由到外部服务 www.weatherdb.com 上

## Sidecar 代理规则配置

用于对 istio 数据面的行为进行更精细的控制

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: SideCar
metadata:
  name: default
  namespace: weather
spec:
  egress:
  - hosts:
    - "istio-system/*"
    - "news/*"
```

在Sidecar上主要通过三个字段来描述规则

- `workloadSelector`: 表示工作负载的选择器(pod 标签选择器),如果为空,则应用到整个命名空间
- `egress`: 配置 sidecar 对网格内其它服务的访问.如果没有配置,则只要名称空间可见,服务就可见
  - `hosts`: 监听的服务,为 `namespace/dnsName` 格式,
  - `port`: 监听器关联的端口
  - `bind`: 监听器绑定的地址
  - `captureMode`: 配置如何捕获监听器的流量.有 `DEFAULT(默认),IPTABLES(基于 iptables 的流量拦截),NONE(没有流量拦截)`.
- `ingress`: 配置 Sidecar 对应工作负载的 Inbound流量.
  - `port`: 必选字段,监听器关联的端口
  - `defaultEndpoint`: 必选字段,为流量转发的目标地址
  - `bind`: 监听器绑定的地址
  - `captureMode`: 配置如何捕获监听器的流量
