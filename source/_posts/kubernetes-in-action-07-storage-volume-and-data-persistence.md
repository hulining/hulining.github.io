---
title: K8S 实战 - 07 存储卷及数据持久化
date: 2021/03/19 07:00:00
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第七个章节.主要了解 K8S 集群中存储卷相关概念,主要讲解了 PV 和 PVC 的使用方式.
---

Kubernetes 提供的存储卷(Volume)属于 Pod 资源级别,共享于 Pod 内所有容器,可用于在容器文件系统之外的存储应用程序的相关数据,甚至还可以独立于 Pod 声明周期之外实现数据持久化.

## 概述

### Kubernetes 支持的存储卷类型

Kubernetes 支持非常丰富的存储卷类型,包括本地存储(节点)和网络存储系统中的诸多机制,甚至支持 Secret 和 ConfigMap 这样的存储资源.

目前,Kubernetes 支持的存储卷包含如下类型:

```text
节点级别: emptyDir,hostPath
网络存储: nfs,Glusterfs,cephfs
特殊存储: ConfigMap,Secret # 用于向 Pod 注入配置或敏感数据
云端存储: awsElaasticBlockStore,gcePersistentDisk
```

上述类型中,`emptyDir,hostPath` 两种类型不具有持久性,要想使用持久类型的存储卷,就得使用网络存储系统.然而,网络存储通常不易使用.

Kubernetes 设计了一种集群级别的资源 `PersistentVolume(PV)` 来用于简易使用网络存储,它由管理员配置存储系统,而后由用户通过 `PersistentVolumeClaim(PVC)` 存储卷直接申请使用的机制大大简化了终端存储用户的配置过程.

### 存储卷的使用方式

在 Pod 中定义使用存储卷的配置由两部分组成:

- 通过 `spec.volumes` 字段定义在 Pod 之上的存储卷列表
- 通过 `spec.containers.volumeMounts` 字段定义在容器上定义的存储卷挂载列表,它只能挂载当前 Pod 资源中定义的具体存储卷

定义方式详见 {% post_link kubernetes-in-action-04-manage-pod 'K8S 实战 - 04 Pod 管理' %} 最后.

## 持久存储卷

PersistentVolume(PV) 是指由集群管理员配置提供的魔偶存储系统上的一段存储空间,它是对底层共享存储的抽象,将共享存储作为一种可由用户申请使用的资源,实现了存储消费机制.

PV 是集群级别的资源,不属于任何名称空间,用户对 PV 资源的使用是需要通过 PersistentVolumeClaim(PVC) 提出使用申请来完成绑定.PVC 是 PV 资源的消费者,它向 PV 申请特定大小空间及访问模式,从而创建出 PVC 存储卷,由 Pod 资源通过 PersistentVolumeClaim 存储卷关联使用

尽管 PVC 使得用户可以以抽象的方式访问存储资源,但是很多时候还是回涉及 PV 的属性.为此,Kubernetes 引入了 StorageClass 资源对象,将存储资源定义为具有显著特性的类别(Class),而不是具体的PV.用户通过 PVC 直接向意向的类别发出申请,这么做避免事先创建 PV 的过程.

### 创建 PV

PersistentVolume 主要支持以下几个通用字段,用于定义 PV 的容量,访问模式和回收策略等

- `capacity`: 当前 PV 的容量
- `accessModes`: 访问模式,可选参数为
  - `ReadWriteOnce`: 仅可被单个节点读写挂载,RWO
  - `ReadOnlyMany`: 可被多个节点只读挂载,ROX
  - `ReadWriteMany`: 可被多个节点读写挂载,RWX
- `persistentVolumeReclaimPolicy`: PV 空间被释放时的处理机制,可选参数如下:
  - `Retain`: 保持不动,由管理员手动回收
  - `Recycle`: 空间回收,删除存储卷目录下所有文件,仅支持 NFS 和 hostPath
  - `Delete`: 删除存储卷,仅支持云端存储系统
- `StorageClass`: 当前 PV 所属的 StorageClass 的名称,默认为空
- `mountOptions`: 挂载选项组成的列表

一般定义如下:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name:
  labels:
  
spec:
  capacity:
    storage:  # 指定 PV 容量,单位为 Mi,Gi
  accessModes: # 设定访问模式列表
  
  persistentVolumeReclaimPolicy:  # 设定回售策略
  mountOptions: # 设定挂载选项列表
  
  <type>: # 与 Pod 中定义 Volumes 大致相同,支持 hostPath,nfs,rbd(ceph),glusterfs
  # <type> = hostPath # 多用于测试
  hostPath:
    path:  # 挂载的本地路径
    type:  # 本地路径的体现方式,详见 https://kubernetes.io/docs/concepts/storage/volumes#hostpath
  
  # <type> = nfs
  nfs:
    path: # 共享的nfs路径
    server:  # nfs 服务器地址
    readOnly: false
  
  # <type> = rbd
  rbd:
    monitors:  # Ceph 存储监控器列表,逗号分隔字符串,必需字段
    image: # rados image 名称
    pool: RBD # rados 存储池名称
    user: admin # rados 用户名
    keyring: "/etc/ceph/keyring" # RBD 用户认证时使用的 keyring 文件路径
    secretRef:  # RBD 用户认证时使用的 Secret 对象,会覆盖 keyring 指定信息
    readOnly: false
    fsType: ext4 # 要挂载的文件系统类型
  
  # <type> = glusterfs
  glusterfs:
    endpoints:  # Endpoints 资源名称,用于提供 Glusterfs 集群部分节点信息作为访问入口,必需字段,可以手动创建
    path:  # 用到的 GlusterFS 集群的卷路径,必需字段
    readOnly: false
  
```

创建完成的 PV 资源可能处于下列四种状态,它们代表着 PV 资源生命周期中的各个阶段:

- `Available`: 可用状态的自由资源,尚未被 PVC 绑定
- `Bound`: 已经绑定至某 PVC
- `Released`: 已经绑定的 PVC 已经被删除,但资源尚未被集群回收
- `Failed`: 故障状态

### 创建 PVC

PersistentVolumeClaim 是存储卷类型的资源,它通过申请占用某个 PersistentVolume 而创建,它与 PV 是一对一关系,用户无须关系底层细节.申请时,用户只需要指定目标空间大小,访问模式,PV 标签选择器和 StorageClass 等相关信息即可.PVC 的 spec 可嵌套字段如下:

- `accessMode`: 当前 PVC 访问模式,可选字段与 PV 相同
- `resources`: 当前 PVC 存储卷需要占用的资源量最小值
- `selector`: 绑定 PV 应用的标签选择器,用于挑选要绑定的 PV
- `storageClassName`: 存储类名称
- `volumeName`: 直接指定要绑定的 PV 卷名

一般定义如下:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name:
  namespace:
  labels:
  
spec:
  accessMode: # 设定访问模式列表
  
  resources:
    requests:
      storage:  # 指定 PVC 需求容量,单位为 Mi,Gi
  selector: # 指定标签选择器
    matchLabels:
      
  storageClassName:  # 限定从指定 storageClass 中选择 PV
```

使用 PVC 的方式见 {% post_link kubernetes-in-action-04-manage-pod 'K8S 实战 - 04 Pod 管理' %} 最后,只需要指定 `spec.volumes.persistentVolumeClaim` 下 `claimName` 和 `readOnly`(可选) 字段即可

### 存储类 StorageClass

存储类(storageClass)是 Kubernetes 资源的一种,它由管理员为管理 PV 方便而按需创建的类别,管理员可以自定义标准进行分类.

存储类的好处之一就是支持 PV 的动态创建,系统按照 PVC 的需求标准动态创建 PV 会为存储类带来极大的灵活性.

StorageClass 中没有 `spec`字段,但包含如下5个字段:

- `provisioner`: 供给方,提供了存储资源的存储系统,存储类要依赖 Provisioner 来判定要使用的存储插件以便适配到目标存储系统
- `parameters`: 存储类使用参数描述要关联到的存储卷,不同的 provisioner 需要提供的参数各不相同
- `reclaimPolicy`: 动态创建 PV 的回收策略,可选值为 Delete(删除,默认) 和 Retain(保留).
- `mountOptions`: 挂载选项列表

一般定义如下:

```yaml
# glusterfs
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: nfs-storageclass     # 暴露出来的 StorageClass 名称
  annotations:
    storageclass.kubernetes.io/is-default-class: true 
provisioner: fuseim.pri/ifs # 需要与 nfs-client-provisioner Deployment 中传入 PROVISIONER_NAME 变量值一致
# parameters: # nfs-storageclass 没有参数
```

### PV 和 PVC 的生命周期

PV 是 Kubernetes 集群的存储资源,PVC 则代表着资源需求.创建 PVC 时对 PV 发起的使用申请,即为绑定.PV 和 PVC 是一一对应关系,可用于响应 PVC 申请的 PV 必须能够容纳 PVC 的请求条件.它们遵循如下生命周期

- 存储供给: 静态供给或动态供给
- 存储绑定: PV 和 PVC 是一一对应关系,如果系统找到符合 PVC 条件的 PV,两者会进行绑定,且绑定后不再与其它 PV 或 PVC 资源进行绑定
- 存储回收: 完成存储卷的使用目标后,即可删除 PVC 对象以便进行资源回收.可用的资源回收策略有3种: `Retained`,`Recycled` 和 `Deleted`
