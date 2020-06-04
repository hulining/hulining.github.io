---
title: istio 可插拔的服务安全
tags:
  - istio
  - 读书笔记
categories:
  - 云原生应用
  - istio
abbrlink: 11402
description: 本文章为《云原生服务网格Istio：原理、实践、架构与源码解析》第 5 章读书笔记.
---

# 原理

> 安全原理

- 安全核心组件 Citadel 用于密钥和证书管理
- Envoy 作为数据面组件代理服务间的安全通信,包括认证,通道加密等
- Pilot 作为配置管理服务,在安全场景下将安全相关的配置分发 Envoy
- Mixer 可以通过配置 Adapter 来做授权和访问审计

从控制面到数据面的配置流程:
- Citadel 监听Kube-apiserver,为每个 Service 都生成密钥和证书,并保存为 Kubernetes Secrets.当创建Pod时,Kubernetes 将包含密钥和证书的 Secret 挂载到对应的Pod中
- Citadel会维护证书的生命周期,并根据配置定期重建 Kubernetes Secrets 以自动更新证书
- Pilot生成配置信息,定义哪个 ServiceAccount 可以运行哪个服务，并将这个配置下发给 Envoy

数据面主要流程如下:
- 客户端的 Envoy 拦截到服务的 Outbound 流量
- 客户端的 Envoy 和服务端的 Envoy 进行双向 TLS 握手
- 在双向TLS建立后,请求到达服务端 Envoy,服务端 Envoy 将请求转发给本地服务

Istio 提供的安全功能主要有认证和授权

> 认证方式

istio 中提供了两种认证方式:
- 传输认证: 又称为从服务到服务的认证.Istio 基于双向 TLS 来实现传输认证,包括双向认证,通道安全和证书自动维护.基于双向TLS可以保护从服务到服务的通信
- 来源认证: 又称为最终用户认证,用于认证请求的最终用户或设备

> 授权

istio 中的授权是基于角色的访问控制(RBAC)

> 密钥证书管理

Citadel 服务主要做 4 个操作
- 给每个 Service Account 都生成 SPIFFE 密钥证书对
- 根据Service Account 给对应的 Pod 分发密钥和证书对
- 定期替换密钥证书
- 根据需要撤销证书

# 服务认证配置

示例
```yaml
# 为 forecast 开启双向认证
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: forecast-weather-mtls
  namespace: istio-system
spec:
  targets:
  - name: forecast
  peers:
  - mtls: {}
```

## 认证策略定义

认证策略定义过程中重要字段如下:
- `targets`: 表示策略作用的目标对象,如果为空,则对策略作用范围内的所有服务都生效.该字段包含 `name` 和 `ports` 分别表示服务名称及端口
- `peers`: 描述传输认证的配置.一般被赋值为 mtls,表示启用双向认证.若不启用,则不用赋值
- `orgins`: 描述访问来源认证的配置.

## 认证策略典型应用

> 全网格服务启用双向 TLS 认证

```yaml
# 作用范围: 全网格,只能存在一个
apiVersion: authentication.istio.io/v1alpha1
kind: MeshPolicy
metadata:
  name: default
spec:
  peers:
  - mtls: {}
```

> 全名称空间服务启用双向 TLS 认证()

```yaml
# 作用范围: 名称空间,名称空间内只能存在一个
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: default
  namespace: default
spec:
  peers:
  - mtls: {}
```

> 对特定服务不启用双向 TLS 认证

```yaml
---
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: enable_weather_tls
  namespace: weather
spec:
  peers:
  - mtls:
---
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: disable_adv_tls
  namespace: weather
spec:
  targets:
  - name: advertisement
```

# 服务授权配置

## 授权启用配置

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ClusterRbacConfig
metadata: ClusterRbacConfig
  name: default
spec:
  mode: "ON_WITH_INCLUSION"
  # 支持 4 种模式,OFF(所有关闭),ON(所有开启),ON_WITH_INCLUSION(只对 inclusion 启用授权),ON_WITH_EXCLUSION(只对 exclusion 禁用授权)
  inclusion:
    namespaces:
    - weather
    # services:
    # - weather_service
```
## 授权策略配置

```yaml
---
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: advertisement-reader
  namespace: weather
spec:
  rules:
  - services: ["advertisement.weather.svc.cluster.local"]
    methods: ["GET"]
---
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: binding-advertisement-reader
  namespace: weather
spec:
  subjects:
  - properties:
      source.namespace: "terminal"
  roleRef:
    kind: ServiceRole
    name: advertisement-reader
```

> serviceRole 定义

关键字 `rules`,表示权限的许可,元素 rule 通过如下字段来定义
- `servces`: 必选字段,表示作用的服务的集合
- `paths`: 配置对应服务的接口列表.未指定时表示所有 path
- `methods`: 指定接口的方法,如 GET,POST 等
- `constraints`: 可选字段,可以理解为服务的扩展字段.常用扩展字段如下
    - `destination.labels`: 目标服务上的标签,如 destination.labels[version] 是 [v1] 来描述对特定版本的权限
    - `request.headers`: 通过 Header 上取值 来描述规则
    - `destination.ip`: 目标服务实例的 IP 地址
    - `destination.port`: 目标服务端口
    - `destination.namespace`: 目标服务名称空间
    - `destination.user`: 目标服务负载上取到的标识

> serviceRoleBinding 定义

serviceRoleBinding 作为绑定的定义,主要绑定两个对象,一个是定义的角色 `roleRef`,另一个是角色分配的目标对象 `subjects`

- `roleRef`: 必选字段,表示要绑定的角色.包含 `kind(角色类型,ServiceRole)` 和 `name(角色名称)` 两个字段
- `subjects`: 必选字段,表示角色分配的目标,是一个列表,可将角色分配到多个对象上.该字段有两个属性分类
    - `user`: 对应用户名或 ID. "*" 表示授权给所有用户和服务
    - `properties`: 扩展属性.支持以下属性
        - `source.ip`: 源服务实例的 IP 地址
        - `source.namespace`: 源服务实例的名称空间
        - `source.principal`: 源服务实例标识
        - `request.headers`: 请求 HTTP header
        - `request.auth.principal`: 请求认证主体
        - `request.auth.audiences`: 认证信息的目标受众
        - `request.auth.presenter`: 授权的凭证提供者
        - `request.auth.claims`: 来源JWT声明

## 授权策略典型应用

> 特定名称空间授权

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: advertisement-reader
  namespace: weather
spec:
  rules:
  - services: ["*"]
    methods: ["*"]
---
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: binding-advertisement-reader
  namespace: weather
spec:
  subjects:
  - properties:
      source.namespace: "client"
  roleRef:
    kind: ServiceRole
    name: advertisement-reader
```

> 特定服务授权

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: advertisement-reader
  namespace: weather
spec:
  rules:
  - services: ["advertisement.weather.svc.cluster.local"]
    methods: ["GET"]
```

> 特定接口授权

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: advertisement-reader
  namespace: weather
spec:
  rules:
  - services: ["advertisement.weather.svc.cluster.local"]
    paths: ["v2/weatherdata"]
    methods: ["GET"]
```

> 特定版本授权

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: advertisement-reader
  namespace: weather
spec:
  rules:
  - services: ["advertisement.weather.svc.cluster.local"]
    methods: ["GET"]
    constraints:
    - key: "destination.labels[version]"
      value: ["v2"]
```

> 特定源授权

```
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: binding-advertisement-reader
  namespace: weather
spec:
  subjects:
  - properties:
      source.ip: "1.1.0.0/16"
  roleRef:
    kind: ServiceRole
    name: advertisement-reader
```