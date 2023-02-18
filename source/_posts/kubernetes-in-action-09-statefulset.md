---
title: K8S 实战 - 09 StatefulSet 控制器
date: 2021/03/19 09:00:00
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第九个章节.主要了解 K8S 集群中 StatefulSet 控制器相关概念及其使用方式
---

## 概述

StatefulSet 是 Pod 资源控制器的一种实现,用于部署和扩展有状态应用的 Pod 资源,确保他们的运行顺序及每个 Pod 资源的唯一性,StatefulSet 需要为每个 Pod 维持一个唯一且固定的标识符,为其创建专有的存储卷.StatefulSet 主要有如下特性

- 稳定且唯一的网络标识符
- 稳定且持久的存储
- 有序优雅的部署和扩展,删除和终止
- 有序而自动的滚动更新

一般来说,完整可用的 StatefulSet 通常由以下三个组件组成:

- `Headless Service`: 用于为资源标识符生成可解析的 DNS 资源记录
- `StatefulSet`: 用于管控 Pod 资源
- `VolumeClaimTemplates`: 基于静态或动态 PV 为 Pod 资源提供专有且稳定的存储

对于一个拥有 N 个副本的 StatefulSet 来说,其 Pod 对象会被有序的创建,顺序依次是 `{0...N-1}`,删除则以相反的方式进行.Pod 资源的名称格式为 `$(statefulset_name)-$(ordinal)`,其域名后缀由相关 Headless 类型的 Service 资源给出,格式为 `$(service_name).$(namespace).svc.cluster.local`.

StatefulSet 控制器会为每个 VolumeClaim 模版创建一个专用的 PV,它会从模版中指定 StorageClass 中为每个 PVC 创建 PV,未指定 Storage 将使用默认的 StorageClass 资源.如果不支持 PV 的动态供给,就需要事先创建好满足需求的所有 PV.

## StatefulSet 应用

### 资源清单

一个完整的 StatefulSet 控制器需要由 Headless Service, StatefulSet 和 volumeClaimTemplate 组成.一般来说定义如下:

```yaml
# 定义 Headless Service,暴露后端 Pod IP
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: default
  labels:
    app: prometheus
spec:
  clusterIP: None # clusterIP 为 None
  selector:
    app: prometheus
  ports: 9090
  - port: prometheus

---
# 创建 sts 对象
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: default
  lables:
    app: prometheus
spec:
  podManagementPolicy: "Parallel" # 指定创建/删除 Pod 时的策略,可选值为 "OrderedReady"(默认),"Parallel".分别表示串行和并行(无需等待)创建/删除 Pod 
  revisionHistoryLimit: 10 # 保留历史 ControllerRevision 的数量
  updateStrategy:
    type: "RollingUpdate" # 指定升级策略,可选值为 RollingUpdate 与 OnDelete(用户必须手动删除Pod)
    rollingUpdate:
      partition:  # RollingUpdate 时,保留旧版本 Pod 的数量
  
  serviceName: "prometheus" # 指定 headless Service 服务名称
  replicas: 1 # 指定 Pod 副本数
  selector:
    matchLabels:
      app: prometheus-pod
  
  # Pod 模版相关内容
  template:
    # <Pod 相关内容>
    metadata:
      labels:
        app: prometheus-pod
    spec:
      
      # ...

  # PVC 模版相关内容
  volumeClaimTemplates:
    # <PVC 相关内容>
  - metadata:
      name:  # PVC 名称,用于 Pod 内部挂载
    spec:
      storageClassName:  # 指定 storageClass 名称
      accessModes:  # 指定访问模式
      - ReadWriteOnce
      resources:  # 指定资源,一般为存储大小
        requests:
          storage: "500Mi"
```

可以看出,StatefulSet 除了需要在 spec 字段中定义 `serviceName`,`volumeClaimTemplates` 外,其它常用字段基本与 Deployment 相同

### StatefulSet 中 Pod 资源的标识符

StatefulSet 控制器创建的 Pod 对象拥有固定且唯一的标识符,他们基于唯一索引,即相关 StatefulSet 名称而生成,名称格式为 `<statefulset_name>-<ordinal_index>`(index 从 0 开始).

这些名称会由 StatefulSet 资源相关的 headless 资源创建为 DNS 资源记录,域名格式为 `$(service_name).$(namespace).svc.cluster.local`.Pod 资源记录为 `$(pod_name).$(service_name).$(namespace).svc.cluster.local`.

Headless Service 资源借助 SRV 记录来引用真正提供服务的后端 Pod 资源的主机名称,进行指向包含 Pod IP 地址的记录条目.StatefulSet 控制器管理的 Pod 在重建后 IP 可能会变化,但它的名称标识在重建后保持不变.

## StatefulSet 中 Pod 资源的存储卷

StatefulSet 控制器通过 `volumeClaimTemplates` 为每个 Pod 副本自动创建并关联一个 PVC 对象,它们分别绑定了一个动态供给的 PV 对象.

删除 StatefulSet 控制器的 Pod 资源,其存储卷并不会被删除,除非用户或管理员手动删除.因此,经由 StatefulSet 重建后的 Pod,会自动关联之前的 PVC 存储卷,且之前数据依旧可用,实现数据持久化.

## 案例: etcd 集群

- Service 资源清单

```yaml
apiVerision: v1
kind: Service
metadata:
  name: etcd
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  clusterIP: None
  selector:
    app: etcd-member
  ports:
  - port: 2379
    name: client
  - port: 2380
    name: peer
---
apiVersion: v1
kind: Service
metadata:
  name: etcd-client
spec:
  type: NodePort
  selector: 
    app: etcd-member
  ports:
  - name: etcd-client
    port: 2379
    protocol: TCP
    targetPort: 2379
```

- StatefulSet 资源清单

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
  labels: 
    app: etcd
spec:
  nameService: etcd
  replicas: 3
  selector:
    app: etcd-member
  template:
    metadata:
      name: etcd
      labels:
        app: etcd-member
    spec:
      containers:
      - name: etcd
        image: "quay.io/coreos/etcd:v3.2.16"
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
        env:
        - name: CLUSTER_SIZE
          value: "3"
        - name: SET_NAME
          value: "etcd"
        volumeMounts:
        - name: data
          mountPath: /var/run/etcd
        command:
        - "/bin/sh"
        - "-ecx"
        - |
          IP=$(hostname -i)
          PEERS=""
          for i in $(seq 0 $((${CLUSTER_SIZE} -1))); do
            ...
          done
          exec etcd --name ${HOSTNAME} \
            --listen-peer-urls http://${IP}:2380 \ 
            --listen-client-urls http://${IP}:2379,http://127.0.0.1:2379 \ 
            --advertise-client-urls http://${HOSTNAME}.${SET_NAME}:2379 \ 
            --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}:2380 \ 
            --initial-cluster-token etcd-cluster-1 \
            --initial-cluster ${PEERS} \ 
            --initial-cluster-state new \
            --data-dir /var/run/etcd/default.etcd

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: nfs-storageclass
      accessModes:
      - "ReadWriteOnce"
      resources:
        requests:
          storage: 1Gi
```
