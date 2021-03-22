---
title: K8S 实战 - 03 资源管理
date: 2021/03/18 03:00:00
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第三个章节.主要了解 K8S 集群的核心对象,并使用 kubeadm 简单部署集群,了解 K8S 命令行管理工具 kubectl 的使用
---

## 资源对象与 API 群组

Kubernetes 将一切事物都抽象为 API 资源,其遵循 REST 风格组织并管理这些资源及对象,同时支持通过标准的 HTTP 方法对资源进行增删改查等管理操作

### 资源对象

依据资源的主要功能作为分类标准,Kubernetes 的 API 对象大体可分为

- 工作负载(Workload)
- 发现和负载均衡(Discovery LB)
- 配置和存储(Config,Storage)
- 集群(Cluster)
- 元数据(Metadata)

#### 工作负载型资源

Pod 是工作负载型资源中的基础资源,它负责运行容器,并向容器解决环境性的依赖.但 Pod 可能会因为资源超限或节点故障等原因而终止,这些 Pod 资源需要被重建,而这类工作通常需要 Pod 控制器来完成

下面是各 Pod 控制器的说明

- `ReplicationController`: 用于确保每个 Pod 副本在任意时刻均能满足目标数量.建议使用 `Deployment` 代替它
- `ReplicaSet`: 新一代的 `ReplicationController`,`ReplicationController` 只支持等值选择,`ReplicaSet` 还支持基于集合的选择器
- `Deployment`: 管理无状态的持久化应用,为 Pod 和 `ReplicaSet` 提供声明式更新,是构建在 `ReplicaSet` 上更为高级的控制器
- `StatefulSet`: 管理有状态的持久化应用,其与 `Deployment` 的不同之处在于 `StatefulSet` 会为每个 Pod 创建一个独有的持久性标识符,并会确保各 Pod 间的顺序性
- `DaemonSet`: 确保节点都运行了某 Pod 的一个副本,新增的节点一样会被添加此 Pod;节点移除时,Pod 会被回收.常用于运行集群存储守护进程,日志收集进程,监控进程等.
- `Job`: 管理运行完成后即可终止的应用.如批处理作业任务.
- `CronJob`: 定时作业任务,会创建多个 `Job` 任务

#### 发现和负载均衡

Kubernetes 使用 `Service` 资源和 `Endpoint` 资源为工作负载添加发现机制及负载均衡功能,使用 `Ingress` 通过七层代理实现请求流量负载均衡.

#### 配置与存储

- `Volume`: 通过引入挂载外部存储卷的方式来满足内部存储数据持久化的需要,它支持众多类型的存储设备或存储系统.
- `ConfigMap`: 能够以环境变量或存储卷的方式将配置文或配置文件接入到 Pod 资源的容器中,并且可被多个同类的 Pod 共享引用,从而实现"一次修改,多次生效"的功能
- `Secret`: 与 `ConfigMap` 类似,对于敏感类数据

#### 集群级资源

Pod,Deployment,Service,ConfigMap 等资源属于名称空间资源,而 Kubernetes 中还存在一些集群级别的资源,用于定义集群自身配置信息的对象,包含如下资源

- `Namespace`: 资源对象名称的作用范围.
- `Node`: Kubernetes 集群的工作节点,其标识符在当前集群中必须是唯一的
- `Role`: 名称空间级别的由规则组成的权限集合
- `ClusterRole`: Cluster 级别的由规则组成的权限集合
- `RoleBinding`: 将 Role 中的权限绑定在一个或一组用户上,它隶属于且仅能作用于一个名称空间
- `ClusterRoleBinding`: 将 ClusterRole 中的权限绑定在一个或一组用户上,它可作用于整个集群

### 资源 API 的组织形式

Kubernetes 将 API 分割为多个逻辑组合,称为 API 群组.当前系统的 API Server 上的相关信息可通过 `kubectl api-versions` 命令获取.每个 API 群组表现为一个以 `/apis` 为根路径的 REST 路径

```text
/api/v1/namespaces/<namespace>/<resource_types>/<resource_name>
/apis/<group>/<version>/namespaces/<namespace>/<resource_types>/<resource_name>
```
