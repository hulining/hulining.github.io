---
title: K8S 实战 - 05 Pod 控制器
date: 2021/03/18
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第五个章节.主要了解 K8S 集群中 Pod 控制器的相关概念及使用方法
---

Pod 控制器由 master 的 kube-controller-manager 组件提供,常见的控制器有 `ReplicationController,ReplicaSet,Deployment,DaemonSet,StatefulSet,Job,CronJob` 等.它们分别以不同的方式管理 Pod 对象.

Pod 控制器资源至少应该包含三个基本组成部分:

- 标签选择器: 匹配并关联 Pod 资源对象,并据此完成受其管控的 Pod 资源计数
- 期望的副本数: 期望在集群中运行着的 Pod 资源的对象数量
- Pod 模版: 用于新建 Pod 资源对象的 Pod 模版资源

Pod 控制器类资源的 spec 字段通常需要内嵌 `replica`,`selector` 和 `template` 字段,其中 `template` 为 Pod 模版定义,Pod 模版的定义基本与 Pod 定义相同,包括 metadata 和 spec 及其内嵌的其它各个字段.

## ReplicaSet 控制器

ReplicaSet(rs) 是 Pod 控制器的一种实现,用于确保由其管控的 Pod 对象副本在任意时刻都能满足期望的数量.

ReplicaSet 控制器资源启动后会查找集群中匹配其标签选择器的 Pod 资源对象,当前活动对象的数量与期望的数量不吻合时,多则删除,少则通过 Pod 模版创建以补足

想对于手动创建 Pod 和管理 Pod 资源来说,ReplicaSet 能够实现如下功能:

- 确保 Pod 资源对象的数量精确反映期望值,多退少补
- 确保 Pod 健康运行,如果节点故障会自动转移
- 弹性伸缩,根据系统负载动态调整相关 Pod 资源对象的数量

ReplicaSet 控制器的定义也由 `kind,apiVerion,metadata,spec` 字段组成.它的 spec 字段一般嵌套使用如下几个属性字段:

- `replicas`: 期望的 Pod 对象副本数
- `selector`: 当前控制器匹配 Pod 对象副本的标签选择器,支持 `matchLabels` 和 `matchExpressions` 两种匹配机制
- `template`: Pod 资源模版

一般来说定义如下:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: 
  namespace: 
  labels:
    
spec:
  replicas: # 副本数
  selector: # 标签选择器
    matchLabels: # 选择器
      # ... 标签
    matchExpressions: 
      # ... 表达式
  
  ##### pod 模版,内嵌 pod yaml 中 matadata 和 spec #####
  template: # 内嵌 pod yaml中的 metadata和spec
    metadata: 
      name: 
      # namespace:  # 一般与控制器在同一个 namespace 中
      labels: # 这里的labels至少要包含所有标签选择器中的标签
        # ... 标签
    spec: 
      containers:
      - name: 
        image: 
        ports: 
        - name: 
          containerPort: 
```

## Deployment 控制器

Deployment 是 Kubernetes 控制器的又一种实现,它构建于 ReplicaSet 控制器之上,可以为 Pod 和 ReplicaSet 提供声明式更新.

与 ReplicaSet 类似,Deployment 同样是为了保证 Pod 资源健康运行,同时添加了如下特性:

- 事件和状态查看: 可以查看 Deployment 对象升级的详细进度和状态
- 回滚: 升级完成后发现问题,使用回滚机制将应用返回到前一个或指定版本上
- 版本记录: 对 Deployment 对象的每次操作都保存下来,以供后续可能执行的回滚操作使用
- 暂停和启动: 对于每次升级,随时暂停和启动
- 多种自动更新方案: 支持重建更新及滚动更新机制

一般来说定义如下:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: 
  namespace: 
  labels:
    
spec:
  replicas: # 副本数
  selector: # 标签选择器
    matchLabels:
  
  ##### 升级策略 #####
  strategy:
    type: RollingUpdate # 更新类型,可选参数为 Recreate 和 RollingUpdate,默认滚动更新
    rollingUpdate:
      maxSurge: 1 # 最多允许同时创建几个 Pod
      maxUnavailable: 0 # 最多允许几个 Pod 同时不可用
      
  resverionHistoryLimit: # 旧版本记录个数,默认为最大值,意味着全部记录
  
  template: # Pod 的模版,详见 Pod yaml
    metadata:
      name: # 这里是 pod 的名称前缀
      labels:
        # ... 标签
    spec:   
      containers:
      - name: # 容器名称
        image: 
        ports: 
        - name: # 容器暴露出来的端口名称
          containerPort: 
```

## DaemonSet 控制器

DaemonSet 是 Pod 控制器的又一种实现,用于集群中的全部节点(包括新加入的节点)同时运行一个指定的 Pod 资源副本,当从集群移除节点时, Pod 对象也将被自动回收.管理员也可以使用节点选择器和节点标签指定仅在部分节点上运行指定 Pod

DaemonSet 有特定的应用场景,通常运行执行系统级操作任务的应用,如:

- 运行集群存储的守护进程,如 glusterfs 或 ceph
- 运行日志守护进程,如 fluentd 和 logstash
- 运行监控系统的代理守护进程,如 Prometheus,Ganglia 等

一般定义如下:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name:
  namespace:
  labels:
    
spec:
  minReadSeconds: 0 # 创建成功后最小准备时间
  selector: # 节点标签选择器
    matchLabels:
    
  ##### 升级策略 #####
  reversionHistoryLimit:  # 旧版本记录个数,默认为最大值,意味着全部记录
  strategy:
    type: RollingUpdate # 更新类型,可选参数为 Recreate 和 RollingUpdate,默认滚动更新
    rollingUpdate:
      maxSurge: 1 # 最多允许多出几个
      maxUnavailable: 0 # 最多允许少几个
  
  template: # Pod 的模版,详见 Pod yaml
    metadata:
      name: # 这里是 pod 的名称前缀
      labels:
        # ... 标签
    spec:   
      containers:
      - name: # 容器名称
        image: 
        ports: 
        - name: # 容器暴露出来的端口名称
          containerPort: 
```

## Job 控制器

Job 控制器用于调配 Pod 对象的一次性任务,容器中的进程在正常运行结束后不会对其进行重启,而是将 Pod 对象置于 "Cpmpleted" 状态.

Job 控制器常用于管理那些运行一段时间便可完成的任务,如计算或备份

一般来说定义如下:

```yaml
apiVerion: batch/v1
kind: Job
metadata:
  name:
  namespace:
spec:
  completions: # 定义 Job 完成期望的数量
  parallelism: # 定义 Job 的并行数量
  
  template: # Pod 模版
    spec:
      containers:
      - name: 
        image:
        command: 
```

## CronJob 控制器

Crontab 控制器用于管理 Job 控制器资源的运行时间.CronJob 类似于 Linux 操作系统的周期性任务作业计划(crontab)的方式控制器运行的时间及重复运行的方式.

CronJob 对象支持使用的时间格式类似于 Crontab,"?" 和 "*" 意义相同,表示任何可用有效值.

一般定义如下:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: 
  namespace:
  labels:
spec:
  schedule:  # Cron 格式的作业调度运行时间点,必须字段
  jobTemplate:  # Job 控制器模版,生成 Job 对象,必须字段
    # ... Job ...
  concurrencyPolicy:  # 并发执行策略,可选值有 "Allow","Forbid","Replace"
  failedJobHistoryLimit: 1 # 失败记录保留历史记录数,默认为1
  successfulJobHistoryLimit: 3 # 成功记录保留历史记录数,默认为3
  startDeadlineSeconds:  # 记录启动的超时时长
  suspend: false # 是否挂起后续的执行任务,默认为 false
```
