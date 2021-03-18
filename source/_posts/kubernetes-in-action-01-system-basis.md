---
title: K8S 实战 - 01 系统基础
date: 2020/03/18
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第一个章节.主要了解 K8S 集群的核心概念,相关组件以及基础网络模型.对 K8S 有个初步的认识.
---

## 概述

Kubernetes 是 Google 开源的一个容器编排引擎,它是 CNCF(Cloud Native Computing Founddation,云原生计算基金会)的第一个开源项目.

Kubernetes 有如下重要特性:

- 自动装箱

构建于容器之上,基于资源依赖及其他约束自动完成容器部署且不影响其可用性,并通过调度机制以提升资源利用率.

- 自我修复

支持容器故障后自动重启,节点故障后重新调度容器,容器健康检查失败后重新创建等自我修复机制.

- 水平扩展

支持通过简单命令或 UI 进行手动水平扩展,目前支持 CPU 资源负载率的自动水平扩展.

- 服务发现和负载均衡

Kubernetes 通过 KubeDNS(或 CoreDNS)组件为系统内置了服务发现功能,并为每个 Service 配置 DNS 名称.Service 通过 iptables 或 ipvs 内建了负载均衡机制.

- 自动发布和回滚

Kubernetes 支持灰度更新应用程序或其配置信息,并会监控更新过程中程序的健康状态,一旦有故障发生,就会自动执行回滚

- 密钥和配置管理

Kubernetes 的 ConfigMap 实现了配置与镜像解耦,使用 Secret 对象保存敏感数据

- 存储编排

Kubernetes 支持 Pod 对象按需自动挂载不同类型的存储系统.

- 批量处理执行

Kubernetes 支持批处理作业及 CI(持续集成)

## 概念和术语

从抽象的视角来讲,Kubernetes 包含如下抽象对象:

- Pod

Pod 是 Kubernetes 的最小调度单元,它可以封装一个或多个容器.同一 Pod 中的容器共享网络名称空间和存储资源,但彼此在 User,PID,mount(根文件系统) 等名称空间上保持隔离.

- 资源标签(Label)

标签是将资源进行分类的标识符,其实就是一个键值型数据,用来指定对象辨识的属性

- 标签选择器(Label Selector)

它是一种根据 Label 来过滤符合条件的资源对象的机制

- Pod 控制器(Controller)

用户通常借助于 Pod 控制器部署及管理 Pod 对象.包括 `ReplicationController,ReplicaSet,Deployment,StatefulSet,Job` 等.

- 服务资源(Service)

Service 是建立在一组 Pod 对象之上的资源抽象,它通过标签选择器选定一组 Pod 对象,并为这组 Pod 对象定义一个统一的固定访问入口.若 Kubernetes 存在 DNS 附件,它就会在 Service 创建时配置一个 DNS 名称以便客户端进行服务发现.到达 Service 的请求将被负载均衡至其后的各个 Pod 之上,因此 Service 本质上来讲是一个四层代理服务.Service 还可以将集群外部流量引入到集群中来

- 存储卷(Volume)

存储卷是独立于容器文件系统之外的存储空间,常用于扩展容器的存储空间并为它提供持久存储能力.

- 名称空间(Namespace)

名称空间通常用于实现租户或项目的隔离,从而形成逻辑分组

- 注解(Annotation)

注解是另一种附加在对象之上的键值型数据,它拥有更大的容量,但不能用于标识和选择对象,其主要目的是方便工具或用户的阅读及查找等

- Ingress

Service 是一种四层代理服务,而 Ingress 是在应用层实现的 HTTP 负载均衡机制,是一种七层代理服务.它仅仅是一组路由规则的集合.这些规则通过 Ingress Controller 发挥作用

## 集群组件

### Master 组件

- api-server

负责输出 RESTful 风格的 API,是发往集群所有 REST 命令的接入点,负责接收,校验,响应所有的 REST 请求.是整个集群的网关

- 集群状态存储

Kubernetes 集群的所有状态信息都需要持久存储于存储系统 etcd 中,etcd 是独立的服务组件,并不隶属于 Kubernetes 集群自身.生产环境应该以 etcd 集群的方式确保其服务可用性

- 控制器管理器(controller-manager)

集群中大多数功能由几个被称为控制器的进程执行实现的,这几个进程被集成于 kube-controller-manager 守护进程中.

- 调度器(Scheduler)

API Server 确认 Pod 对象的创建请求之后,需要由调度器根据集群各节点的可用资源状态及运行容器的资源需求做出调度决策.

### Node 组件

- kubelet

kubelet 从 API Server 接收关于 Pod 对象的配置信息并确保他们处于目标状态.kubelet 会在 API Server 上注册当前工作节点,定期向 Master 汇报节点资源使用情况,并通过 cAdvisor 监控容器和节点的资源占用情况

- 容器运行时环境(Container Runtime)

每个 Node 都需要提供一个容器运行时环境,它负责下载镜像并运行容器.Kubernetes 主要使用的容器运行时环境为 Docker

- kube-proxy

每个工作节点都需要运行一个 kube-proxy 守护进程,它能够按需为 Service 资源对象生成 iptables 或 ipvs 规则

### 其它核心附件

- CNI(Container Network Interface): 用于网路规划及相关策略的分配
- KubeDNS: 提供集群内 DNS 服务
- Kubernetes Dashboard: 管理集群应用或集群自身的 WebUI
- heapster: 容器和节点的性能监控与分析系统,能够收集并解析多种指标.后续使用 Prometheus
- Ingress Controller: 提供应用层负载均衡机制.目前项目有 nginx-controller,traefik 等

### 网络模型

Kubernetes 要求网络模型符合如下需求:

- 所有 Pod 间均可不经 NAT 机制而直接通信
- 所有节点均可不经 NAT 机制而直接与所有容器通信
- 所有 Pod 对象都位于同一平面网络,而且可使用 Pod 自身的地址直接通信

Kubernetes 中的网络模型主要存在四种类型的网络:

- 同一 Pod 内的容器间通信: 直接通过本地的回环地址通信
- 相同节点各 Pod 彼此之间的通信: 通过 Pod 网络(docker0 网络实现)进行通信
- 不同节点各 Pod 彼此之间的通信: 通过不同节点上的 Pod 网络(docker0 网桥实现)进行通信
- Pod 与 Service 间的通信:  通过 keub-porxy 借助 iptables 规则或 ipvs 规则重定向
- 集群外部流量同 Service 之间的通信: 通过 keub-porxy 借助 iptables 规则或 ipvs 规则重定向

Kubernetes 为 Pod 和 Service 资源对象分别使用了各自的专用网络,Pod 网络由 Kubernetes 的网络插件实现,Service 的网络由 Kubernetes 集群指定.
