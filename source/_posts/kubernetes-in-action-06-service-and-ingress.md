---
title: K8S 实战 - 06 Service 和 Ingress
date: 2021/03/18
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第六个章节.主要了解 K8S 集群中 Service 和 Ingress 的相关定义与用法
---

Kubernetes 提供了两种内建的云端负载均衡机制用于发布公共应用,一种是工作于传输层的 Service 资源,另一种是工作与应用层的 Ingress 资源

## Service 资源及其实现

### 概述

Service 是 Kubernetes 的核心资源之一,它是一种抽象: 通过规则定义出由多个 Pod 对象组合而成的逻辑组合以及访问这组 Pod 的策略. Service 关联 Pod 资源的规则要借助于标签选择器来完成,

Service 基于标签选择器,将一组 Pod 定义成一个逻辑组合,并通过自己的 IP 地址和端口调度代理至组内的 Pod 对象之上,它向客户端隐藏了真实的,处理用户请求的 Pod 资源.

Service 对象的 IP 也被称为 Cluster IP,它在 Service 对象创建后即保持不变,并能被同一集群中的资源所访问.Service 端口用于接收客户端请求,并将其转发至其后端 Pod 的相应端口上,这种代理机制被称为端口代理或四层代理,它工作在 TCP/IP 协议栈的传输层.

Service 资源会通过 API Server 持续监控标签选择器匹配到后端 Pod 对象,并跟踪各对象的变动.不过,Service 并不直接链接至 Pod 对象,他们之间还有 Endpoints 资源对象,它是一个由 IP 地址和端口组成的列表,这些 IP 地址和端口来自于 Service 标签选择器匹配到的 Pod 资源,创建 Service 资源对象时,其关联的 Endpoints 对象也会自动创建.

### 工作原理

简单来讲, Service 对象就是工作节点上一些 iptables 或 ipvs 规则,用于将到达 Service 对象 IP 地址的流量调度转发至相应的 Endpoints 对象指向的 IP 地址和端口之上.工作于每个节点的 kube-proxy 组件通过 API Server 持续监控各个 Service 及其关联的 Pod 对象,并将其变动反映到 iptables 或 ipvs 规则上.

Service IP 是用于生成 iptables 或 ipvs 规则时使用的 IP 地址,是虚拟 IP,kube-proxy 将请求代理至相应端点的方式有3种,`userspace(用户空间)`,`iptables` 和 `ipvs`

- userspace 代理模型

kube-proxy 负责跟踪 API Server 上 Service 和 Endpoints 对象的变动,并据此调整 Service 资源的定义.对于每个 Service 对象,它会随机打开一个本地端口,任何到达此代理端口的连接请求都将被代理至 Pod 对象上.默认调度算法是轮询

在这种代理模型中,请求流量到达内核空间后经由套接字送往用户空间的 kube-proxy,而后再送往内核空间,调度至后端 Pod.请求在内核空间和用户空间中来回转发效率不高.

- iptables 代理模型

与 userspace 基本类似,kube-proxy 负责跟踪 API Server 上 Service 和 Endpoints 对象的变动,并据此调整 Service 资源的定义.对于每个 Service 对象,它都会创建 iptables 规则直接捕获到达 Cluster IP 和 Port 的流量,并将其重定向(DNAT,目标地址转换)至当前 Service 的后端.默认算法是随机调度.

相对于用户空间模型,iptables 无需将流量在用户空间和内核空间切换,效率更高;但后端 Pod 无响应时不会自动进行重定向

- ipvs 代理模型

kube-proxy 负责跟踪 API Server 上 Service 和 Endpoints 对象的变动,据此来调用 netlink 接口创建 ipvs 规则,并确保与 API Server 中变动保持同步.其请求流量的调度功能由 ipvs 实现,余下功能仍由 iptables 完成.

类似于 iptables 模型,ipvs 构建于 netfilter 钩子函数上,它使用 hash 表作为底层数据结构并工作于内核空间,流量转发速度快,规则同步性能好的特性

### Service 清单文件

一般定义如下:

```yaml
apiVerion: v1
kind: Service
metadata:
  name: 
  namespace:
  labels:
  
spec:
  type:  # Service 类型,默认为 ClusterIP.可选值为 ClusterIP,NodePort,LoadBalacer,ExternalName(用于接入集群外部CNAME)
  selector:  # Service 的标签选择器,用于关联后端 Pod
  clusterIP:  # Service 的 IP地址,可选值为 "None","" 或指定 IP 地址."None" 用于构建 Headless Service.
  ports:
  - name:  # 端口名称
    port:  # Service 暴露出来的端口,如果为空,默认与 targetPort 一致,
    targetPort:  # 转发到后端的 Pod 端口
    protocol: TCP  # 端口协议,TCP 或 UDP
    nodePort:  # Service 端口在工作节点进行端口映射暴露出来的端口,默认范围为 30000-32767,仅当 type 为 NodePort 时有效.
  
  externalName:  # 接入集群外部 CNAME,一般为域名,但是不可以指定端口,仅用于 Service 类型为 ExternalName 时
  externalIPs:  # 接入集群外部服务,外部服务的 IP 列表
  loadBalancerIP:  # 指定创建负载均衡器使用的 IP 地址,需要云厂商支持
  loadBalancerSourceRanges:  # 指定负载均衡器允许的客户端来源的地址范围
  sessionAffinity: None # 会话粘滞性,可选值有 ClientIP,None
```

## 服务发现

Kubernetes 内部提供了服务发现机制.它通过注入环境变量和域名解析的方式实现服务发现.

### 服务发现方式: 环境变量

创建 Pod 资源时,kubelet会将其所属名称空间内的每个活动的 Service 对象以一系列环境变量的形式注入其中.

> Kubernetes Service 环境变量

Kubernetes 为每个 Service 资源生成包括以下形式的环境变量在内的一系列环境变量,在同一名称空间中创建的 Pod 对象都会自动拥有这些变量

- `{SVCNAME}_SERVICE_HOST`
- `{SVCNAME}_SERVICE_PORT`

> Docker link 形式的环境变量

在创建 Pod 对象时,Kubernetes 会将一系列环境变量注入到 Pod 对象中.如下示例:

```bash
/ # printenv | grep MYAPP
MYAPP_SVC_PORT_80_TCP_ADDR=10.107.208.93
MYAPP_SVC_PORT_80_TCP_PORT=80
MYAPP_SVC_PORT_80_TCP_PROTO=TCP
MYAPP_SVC_PORT_80_TCP=tcp://10.107.208.93:80
MYAPP_SVC_SERVICE_HOST=10.107.208.93
MYAPP_SVC_SERVICE_PORT=80
MYAPP_SVC_PORT=tcp://10.107.208.93:80
```

### 服务发现方式: DNS

创建 Service 资源对象时,集群内的 DNS 服务会为它们自动创建资源记录用于名称解析和服务注册.Pod 可以直接使用标准的 DNS 名称来访问这些 Service 资源.每个 Service 对象相关的 DNS 记录包含如下两个.

- `{SVCNAME}.{NAMESPACE}.{CLUSTER_DOMAIN}`
- `{SVCNAME}.{NAMESPACE}.svc.{CLUSTER_DOMAIN}`

而 Kubernetes 中 Pod 的 DNS 配置信息会自动注入到它的 `/etc/resolv.conf` 文件中.其文件内容如下:

```text
nameserver 10.96.0.10
search {NAMESPACE}.svc.{CLUSTER_DOMAIN} svc.{CLUSTER_DOMAIN} {CLUSTER_DOMAIN}
```

## 服务暴露

Service 的 IP 地址仅在集群内可达,但总会有些服务需要暴露到外部网络中接收各类客户端的访问.此时,需要在集群的边缘为其添加一层转发机制,以实现将外部请求流量接入到集群的 Service 资源上.

### Service 类型

Kubernetes 的 Service 有以下 4 种类型,详见[官方文档](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

- `ClusterIP`: 通过集群内部 IP 地址暴露服务,此地址仅在集群内部可达,而无法被集群外部客户端访问
- `NodePort`: 这种类型建立在 ClusterIP 之上,其在每个节点的 IP 地址的静态端口用于将集群外部的用户请求转发至目标 Service 的 ClusterIP 和 Port.这种类型的 Service 既可以通过 `<ClusterIP:ServicePort>` 进行访问,又可以通过 `<NodeIP>:<NodePort>` 进行访问
- `LoadBalancer`: 这种类型构建在 NodePort 类型之上,其通过云厂商提供的负载均衡器将服务暴露到集群外部.此类型的 Service 会指向关联至 Kubernetes 集群外部的切实存在的某个负载均衡设备,该设备通过工作节点上的 NodePort 向集群内部发送请求流量.此类型优势在于,负载均衡设备能够避免客户端指定节点故障而导致服务不可用
- `ExternalName`: 通过 Service 映射至 externalName 字段内容指定的主机名来暴露服务,此主机名需要被 DNS 服务器解析至 CNAME 类型的记录.主要用于将集群外部的服务以 DNS CNAME 的方式映射到集群中,从而让集群内的 Pod 资源能够访问外部 Service 的一种实现方式.

### HeadLess 类型的 Service 资源

如果客户端需要直接访问 Service 资源后端的所有 Pod 资源,这时就应该想客户端暴露每个 Pod 资源的 IP 地址,而不再是中间层 Service 对象的 ClusterIP.这类型的 Service 资源便称为 Headless Service.

正常的 Service 需要通过 ipvs 规则转发到实际的 Pod 上.而 Headless Services 不会分配 ClusterIP,而是将 Endpoints 返回,也就将服务端的所有节点地址返回,让客户端自行要通过负载策略完成负载均衡.

其定义方式与 ClusterIP 定义方式基本相同,区别在于 `spec.clusterIP` 的值为 "None",如下:

```yaml
apiVerison: v1
kind: Service
metadata:
  name: 
spec:
  clusterIP: None
  selector:
  
  # ...
```

## Ingress 资源

Ingress 是 Kubernetes API 标准类型之一,它是基于 DNS 名称或 URL 路径把请求转发至指定的 Service 资源的规则,用于将集群外部的请求流量转发至集群内部完成服务发布.

Ingress 资源本身并不能进行流量从穿透,它仅仅是一组路由规则的组合,这些规则要想真正发挥作用还需要其它功能的辅助,然后根据这些规则的匹配机制路由请求流量.这种能够为 Ingress 资源监听套接字并转发流量的组件被称为 Ingress 控制器.

Ingress 控制器可以由任何具有反向代理功能的服务程序实现,如 Nginx,[Traefik](https://traefik.io/)

使用 Ingress 资源进行流量转发时,Ingress 控制器可基于某 Ingress 资源定义的规则将客户端流量直接转发至与 Service 对应的后端 Pod 资源上,这种转发机制会绕过 Service 资源,从而省去不必要的开销.

### 创建 Ingress 资源

Ingress 资源是基于 HTTP 虚拟主机或 URL 的转发规则,它在配置清单的 spec 字段提供了 `rules,backend,tls` 等字段进行定义.一般如下:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name:
  namespace:
spec:
  rules: # 定义负载均衡的规则
  - host: # 指定主机名
    http:
      paths:
      - path: # 指定主机名下的指定指定路径会转发到指定后端,默认为"/"
        backend: # 指定后端服务相关参数
          serviceName:
          servicePort: 
  - host: # 支持指定多个主机名,根据不同主机名分发到不同的后端服务
    http:
      paths:
      - path:
        backend:
          serviceName:
          servicePort: 
  
  backend: # 默认提供后端负载均衡服务的配置
    serviceName:  # 指定后端负载均衡的服务名称 
    servicePort:  # 指定后端负载均衡的服务端口
  
  tls: # TLS 相关配置,目前仅支持默认 443 端口
  - secretName:  # 用于 TLS 加密通信的密钥名称
    hosts: # 用于 TLS 加密通信的主机列表
```

### Ingress 资源类型

ingress 根据 rules 规则定义参数/方式的不同,大致分为 3 种类型

- 单 Service 资源型 Ingress: 只指定 `spec.rules.backend` 字段,将所有请求转发至指定 Service 资源.
- 基于 URL 路径进行流量分发: 在 `spec.rules` 字段中指定单个主机名 `host`,并在该主机名下指定的多个不同的 `path`,各个 `path` 的请求都会转发至对应的 Service 资源.
- 基于主机名的虚拟主机: 在 `spec.rules` 下指定多个主机名 `host`,并根据请求中 `Host` 请求头将请求转发到对应 `host` 的后端 Service 资源.

### 部署 Ingress 控制器

Ingress 控制器自身是运行于 Pod 中的容器应用,一般是 Nginx 一类具有代理及负载均衡能力的守护进程.它监视着来自于 API Server 的 Ingress 对象状态,并以其规则生成相应的应用程序专有格式的配置文件并通过重载或重启守护进程而使新配置生效.

同样运行为 Pod 资源的 Ingress 控制器有该如何接入外部请求的流量呢?常用解决方案有如下两种

- 以 Deployment 控制器管理 Ingress 控制器的 Pod 资源,并通过 NodePort 或 LoadBalancer 类型的 Service 对象为其接入集群外部的请求流量.这就意味着,定义一个 Ingress 控制器时,需要在其前端定义一个专用的 Service 资源
- 借助于 DaemonSet 控制器,将 Ingress 控制器的 Pod 资源各自以单一实例的方式运行于集群的所有部分工作节点之上,并配置这类 Pod 对象以 hostPort 的方式在当前节点接入流量.好像都需要在其前端定义一个专用的 Service 资源

常用的 Ingress-controller 资源有:

- 以 Nginx 为例

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/cloud/deploy.yaml
```

- 以 Traefik:v1.7 为例

```bash
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-deployment.yaml
# 或使用 DaemonSet
# kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-ds.yaml
# 使用 Traefik 内置的 WebUI
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/ui.yaml
```
