---
title: K8S 实战 - 04 Pod 管理
date: 2021/03/18 04:00:00
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第四个章节.主要了解 Pod 的管理方式,生命周期,状态检测,资源限制等相关知识.最后给出了 Pod 的资源清单文件,在创建 Pod 时用于参考
---

## 容器与 Pod 资源对象

现代的容器技术被设计用来运行单个进程,该进程在容器中 PID 名称空间进程号为1,可直接接受并处理信号.在此进程终止时,容器即终止退出.它将日志信息直接输出至标准输出,支持用户使用命令(`kubectl logs`)进行获取.这也是 Docker 及 Kubernetes 使用容器的标准方式

Pod 对象是一组容器的集合,这些容器共享 Network,UTS,IPC 名称空间,因此具有相同的域名,主机名和网络接口,并可通过 IPC 直接通信.

## 管理 Pod 对象的容器

一个 Pod 对象至少存在一个容器,`containers` 字段是定义 Pod 时嵌套字段 `pod.spec` 的必选项,用于为 Pod 指定要创建的容器列表.

### 镜像及其获取策略

容器的 `pod.spec.containers.imagePullPolicy` 字段用于为其指定镜像获取策略,它的可用值包括

- Always: 镜像标签为 "latest" 或镜像不存在时总是从仓库中获取镜像
- IfNotPresent: 仅当本地镜像缺失时才从目标仓库下载镜像
- Never: 禁止从仓库下载镜像,仅使用本地镜像

需要注意的是,使用私有仓库中的镜像通常需要由 Registry 服务器完成认证后才能进行.认证过程要么在相关节点上交互式执行 `docker login` 登录,要么就是将认证信息定义为专属的 Secret 资源,并配置 Pod 通过 `imagePullSecretes` 字段调用此认证信息完成.其中,`imagePullSecretes` 字段支持配置在 `pod.spec.containers` 字段中,也可以配置在 `serviceaccount.imagePullSecretes` 字段中,后者用于通过 sa 认证拉取镜像.

### 暴露端口

Kubernetes 系统的网络模型中,各 Pod 的 IP 地址处于同一网络平面,无论是否容器指定了要暴露的端口,都不会影响集群中其它节点之上的 Pod 客户端对其进行访问,任何监听在非 lo 接口上的端口都可以通过 Pod 网络直接被请求.

容器的 `pod.spec.containers.ports` 字段是一个列表,由一到多个端口对象组成,它的常用嵌套字段包括:

- `containerPort <integer>`: 必选字段,指定 Pod 对象暴露的端口
- `name <string>`: 指定端口名称,且在当前 Pod 中是唯一的,此端口名可被 Service 资源调用
- `protocol`: 端口相关协议,其值仅为 TCP 或 UDP,默认为 TCP

然而 Pod 对象的 IP 地址仅在当前集群内可达,无法接受来自集群外部客户端的请求流量.解决方案是通过其所在的工作节点的 IP 地址和端口将其暴露到集群外部.需要做如下配置:

- `hostPort <integer>`: 主机端口,它将接收的请求通过 NAT 机制转发至 `containerPort` 字段指定的容器端口
- `hostIP <string>`: 主机端口绑定的主机 IP,默认为 0.0.0.0

需要注意的是,`hostPort` 与 `NodePort` 类型的 Service 对象暴露端口的方式不同,`NodePort` 是通过所有节点暴露容器服务,而 `hostPort` 则只是 Pod 对象所在节点暴露容器服务.

### 自定义运行的容器化应用

`pod.spec.containers` 中的 `command` 字段能够指定不同于镜像默认运行的应用程序,并可以使用 `args` 字段进行参数传递,它们将覆盖镜像中的默认定义.

如果仅为容器定义了 `command` 字段,它将覆盖镜像中定义的程序及参数,并以无参数方式运行应用程序;如果仅为容器定义了 `args` 字段,它将作为参数传递给镜像中默认指定运行的应用程序.

### 环境变量

向 Pod 对象中的容器传递环境变量参数的方法有两种:

- `env`: 该字段嵌套在 `spec.contianers` 中,它是由容器变量构成的列表.通常由 `name` 和 `value` 指定环境变量的名称和值.该字段还可以使用 `valueFrom` 从 `configMap`,`secret`,`field`,`resourceField` 中获取变量的值
- `envFrom`: 该字段可以从 `ConfigMap` 和 `Secret` 资源中获取

### 共享节点的网络名称空间

多数情况下,同一个 Pod 的各个容器均运行于一个独立的,隔离的名称空间中,容器间彼此共享网络协议栈及相关网络设备.然而,有一些 Pod 对象需要运行于所在节点的名称空间中,执行系统级别的任务.Kubernetes 提供了如下配置来共享节点的名称空间

- `pod.spec.hostNetwork`: 是否共享主机节点网络名称空间
- `pod.spec.hostIPC`: 是否共享主机节点的 IPC 名称空间,可用于进程间通信
- `pod.spec.hostPID`: 是否共享主机节点的 PID 名称空间,共享进程 ID

### 设置 Pod 的安全上下文

Pod 对象的安全上下文用于设定 Pod 或容器的权限和访问控制功能,常用属性包含以下几个方面:

- 基于用户 id 和组 id 的访问控制对象时的权限
- 以 root 或非 root 方式运行
- 是否能够 sudo

Pod 的安全上下文定义在 `pod.spec.securityContext` 字段中,容器的安全上下文定义在 `pod.spec.containers.securityContext` 字段中.下面列出几个常用的字段:

- `allowPrivilegeEscalation`: 是否允许权限升级,即 sudo.仅支持 containers
- `privileged`: 是否支持特权运行容器,等同于 root.仅支持 containers
- `runAsGroup`: 以 GID 身份运行程序.两者都支持
- `runAsNonRoot`: 容器是否必须以非 root 用户运行.两者都支持
- `runAsUser`: 以 UID 身份运行程序.两者都支持

## 标签与标签选择器

标签是 Kubernetes 极具特色的功能之一,它能够附加于 Kubernetes 任何资源对象之上,而后即可由标签选择器进行匹配度检查从而完成资源挑选.

### 管理资源标签

创建资源时,可以在其 `pod.metadata.labels` 字段定义要附加的标签项.可在 `kubectl get pods` 命令中使用 `--show-labels` 选项,以额外显示对象的标签信息.

`kubectl label` 命令可以直接管理活动对象的标签,按需进行添加或修改等操作.

### 标签选择器

标签选择器用于表达式标签的查询条件或选择标准,Kubernetes API 目前支持两个选择器: 基于等值关系和基于集合关系.基于等值关系的标签选择器可用操作符有`"=","==","!="`;基于集合关系的标签选择器支持 `"in","notin","exists"` 三种操作符

使用标签选择器遵循如下逻辑:

- 多个选择器之间的逻辑关系为"与"操作
- 使用空值的标签选择器意味着每个资源对象都将被选中
- 空的标签选择器将无法选出任何资源

标签选择器一般在 Pod 控制器或 Service 资源对象中使用,它们在 `pod.spec.selector` 字段,通过 `matchLabels` 或 `matchExpressions` 来指定标签选择器.

### 节点选择器 nodeSelector

节点选择器是标签及标签选择器的一种应用,它能够让 Pod 对象基于集群中工作节点的标签来挑选倾向运行的目标节点

Pod 对象的 `pod.spec.nodeSelector` 可用于定义节点标签选择器,用户事先为特定部分的 Node 资源对象设定好标签,而后配置 Pod 对象通过节点标签选择器进行匹配检测,从而完成节点亲和性调度

## Pod 的生命周期

Pod 对象从其创建开始至其终止退出的时间范围称为生命周期.其中包括

- 初始化容器
- 容器启动后钩子
- 容器存活性检测,就绪性检测
- 容器终止前钩子

![Pod 的生命周期](/images/pod-lifecyle.png)

### Pod 的相位

Pod 对象总是应该处于以下几个状态之一:

- `Pending`: API Server 创建了 Pod 资源对象并已存入 etcd 中,但尚未被调度完成或仍处于从仓库下载镜像的过程
- `Running`: Pod 已经被调度至某节点,所有容器都已经被 kubelet 创建完成
- `Succeeded`: Pod 中所有容器都已经调度成功且不会被重启
- `Failed`: 所有容器已终止,但至少有一个容器终止失败,容器返回了非 0 值的退出状态或已被系统终止
- `Unknown`: API Server 无法正常获取到 Pod 对象的状态信息,通常是其无法与节点的 kubelet 通信所致

### Pod 生命周期中的重要行为

#### 初始化容器

初始化容器即应用程序的主容器启动之前要运行的容器,常用于为主容器执行一些预置操作,有如下典型特性:

- 初始化容器必须运行完成直到结束,若初始化容器失败,Kubernetes 需要重启它直到成功完成
- 每个初始化容器都必须按定义的顺序串行运行

初始化容器的典型应用需求具体如下:

- 运行特定的工具程序
- 提供主容器中不具备的工具程序或代码
- 为容器镜像的构建和部署人员提供分离,独立的工作途径

Pod 资源的 `pod.spec.initContainers` 字段以列表形式定义可用的初始化容器

#### 生命周期钩子函数

Kubernetes 为容器提供了两种生命周期钩子函数:

- `postStart`: 容器创建后立即运行的钩子处理器
- `preStop`: 于容器终止操作之前立即运行的钩子处理器.它以同步的方式调用,在其完成之前会阻塞删除容器操作的调用

以上两种处理器定义在 `pod.spec.containers.lifecycle` 字段中

钩子处理器的实现方式有 `Exec` 和 `HTTP` 两种,分别用于直接执行命令和向 URL 发起 HTTP 请求.

### 容器状态探测

Kubelet 可以执行对容器进行两种类型的检测:

- `livenessProbe`: 存活性检测.判断容器是否处于运行状态,未通过检测的容器,则将其杀死并根据 `pod.spec.restartPolicy` 决定是否将其重启
- `redinessProbe`: 就绪性检测.判断容器是否准备就绪并可对外提供服务,未通过检测的容器,端点控制器(Service等)会将其 IP 从所有匹配到此 Pod 对象的 Service 对象端点列表中移除.检测通过后,再添加到端点列表中

Kubernetes 的容器支持的存活性/就绪性检测探测方法包含如下三种:

- `ExecAction`: 在容器中执行一个命令,根据返回状态码进行诊断
- `TCPSocketAction`: 通过与容器的某个 TCP 端口尝试建立连接进行诊断
- `HTTPGetAction`: 向容器 IP 地址指定端口指定路径发送 HTTP Get 请求

> 设置 exec 探针

exec 类型的探针通过在目标容器中执行有用户自定义命令判断容器状态.`pod.spec.containers.livenessProbe.exec` 字段用于此类检测,它只有一个可用属性 `command`,用于执行要执行的命令

> 设置 HTTP 探针

基于 HTTP 的探测向目标容器发起一个 HTTP 请求,根据其响应码进行结果判定.`pod.spec.containers.livenessProbe.httpGet` 字段用于定义此类检测.它的可用配置字段包括如下几个:

- `host`: 请求的主机地址,默认为 Pod IP
- `port`: 请求端口,必选字段
- `httpHeaders`: 自定义请求报文首部
- `path`: 请求的 HTTP 资源路径
- `scheme`: 建立连接时使用的协议,仅可为 HTTP 或 HTTPS,默认 HTTP

> 设置 TCP 探针

基于 TCP 的存活性探测用于向容器的特定端口发起 TCP 请求并尝试连接并进行结果判定.`pod.spec.containers.livenessProbe.tcpSocket` 字段用于定义此类检测.包含以下两个属性:

- `host`: 请求连接的目标 IP 地址,默认为 Pod IP
- `port`: 请求连接的目标端口,必须字段

> 探测行为属性

`pod.spec.containers.livenessProbe` 字段通过如下字段来配置相关探测属性

字段 | 显示属性 | 含义 | 默认值
:---: | :---: | :---: | :---:
`initialDelaySeconds` | delay | 容器启动多久后开始第一次探测 | 0(s)
`timeoutSeconds` | timeout | 探测的超时时长 | 1(s)
`periodsSeconds` | period | 探测的频度 | 1(s)
`successThreshold` | #success |处于失败状态时,至少多少次成功才认为通过检测 | 1
`failureThreshold` | #failure | 处于成功状态时,至少多少次失败才认为通过检测 | 3

### 容器的重启策略

Pod 对象终止后是否应该被重建取决于重启策略 `pod.spec.restartPolicy` 属性的定义

- Always: 但凡 Pod 对象终止就重启,默认配置
- OnFailure: 仅在 Pod 对象出现错误时重启
- Never: 从不重启

## 资源需求及资源限制

在 Kubernetes 上,可由容器或 Pod 消耗的计算资源是指 CPU 和内存.CPU 是可压缩型资源,资源额度可按需收缩;内存是不可压缩资源,资源不足可能产生问题

目前来说,资源隔离尚属于容器级别,CPU 和内存资源的配置需要在 Pod 中的容器上进行,可由 `pod.spec.containers.resources.requests` 确保资源的可用值,`pod.spec.containers.resources.limits` 限制资源可能的最大值.

根据 Pod 对象的 `requests` 和 `limits` 属性,Kubernetes 将 Pod 对象归类到 `BestEffort`,`Burstable` 和 `Guaranteed` 三个服务质量(QoS)类别下:

- `Guaranteed`: 每个容器都为 CPU 和内存设置了相同值的 requests 和 limits 属性,这类 Pod 具有最高优先级
- `Burstable`: 至少设置了 CPU 或内存资源的 requests 属性,但不满足 Guaranteed 条件的 Pod 资源,具有中等优先级
- `BestEffort`: 没有为任何一个容器设置 requests 或 limits 属性的 Pod 资源,具有最低优先级.

不同 QoS 的 Pod 有以下不同：

- CPU 调度按照 request 资源划分权重，`Guaranteed` 类型的 Pod 会绑定 CPU 核心，`Burstable` 与`BestEffort` 类型的 Pod 共享该节点上剩余的 CPU 资源核心。
- Memory 按照 QoS 划分 OOMScore，`Guaranteed` 类型的 Pod 为 -998，`BestEffort` 类型的 Pod 为 1000，`Burstable` 类型的 Pod 会根据内存申请的大小和节点内存的关系分配 2-999 的 OOMScore。因此在内存资源紧缺时,`BestEffort` 类别的容器首先被终止，因为系统不为其提供任何资源级别的保证。`Guaranteed` 类别最后被终止。同时，同等级别的 Pod 资源，与自身的 requests 属性相比，内存使用率越高的对象越先被杀死。

## Pod 资源定义清单文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: 
  namespace: 
  labels:
    <label_key>: <label_value>
  annotations:
    <anno_key>: <anno_value>
spec:
  shareProcessNamespace: false # pod 内容器是否共享 pid 名称空间。若共享，pause 进程的 pid 为 1
  hostNetwork: false # 是否与主机共享使用网络名称空间
  hostIPC: false # 是否与主机共享使用 IPC
  hostPID: false # 是否与主机共享使用 PID
  priorityClassName: 0 # 指定 pod 的优先级.优先级高的优先被调度.Kubernetes 内置了 2 个 `PriorityClass: `system-cluster-critical`(10^9) 和 `system-node-critical`(10^9+1000)
  # K8S 中内置了 system-node-critical 和 system-cluster-critical.可通过创建 priorityClass 自定义优先级
  serviceAccountName: "default" # 指定要使用的 service account 认证,默认为当前名称空间下的 "default" sa 对象
  restartPolicy: # 重启策略,Always,Never,OnFailure,默认 Always
  nodeSelector:
    <label_key>: <label_value> # 通过标签选择器进行节点筛选
  hostAliases:  # 为此类 Pod 添加 hosts 配置
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
  
  ##### 镜像相关 #####
  initContainers: # 与 containers 支持的字段相同
  # ...
  containers:
  - name: 
    image: 
    imagePullPolicy: # 拉取镜像策略,Always,IfNotPresent,Never
    imagePullSecret: # 拉取景象的 Secret 对象
    
    ports:  # 暴露端口
    - name: # 暴露端口名称
      containerPort: # 容器内暴露端口
    
    env: # 设置变量
    - name: # 变量名
      value: # 变量值
      valueFrom: # 从 configMap,默认传入的 field 字段,resources 设置,secrete 中获取变量值
        configMapKeyRef:
          key: 
          name: 
          options: # 指定 ConfigMap 或指定 key 是否必需存在
        fieldRef: 
          fieldPath: # 支持 metadata.name, metadata.namespace,metadata.labels[<KEY>], metadata.annotations[<KEY>], spec.nodeName,spec.serviceAccountName, status.hostIP, status.podIP
          apiVerion:
        resourceFieldRef:
          resource: # 支持 limits.cpu, limits.memory, requests.cpu,requests.memory
          containerName: # 镜像名称
        secretKeyRef:
          key: 
    
    envFrom:    # 从 Secret,configMap 中获取变量
    - configMapRef:
        name:   # 配置 configMap 的名称
      prefix:  # 指定前缀,避免多个 ConfigMap 键值产生冲突,引用时需要加上此前缀,可选参数
    - secretRef:
        name:   # 配置 Secret 的名称
    
    volumeMounts:
    - name:  # 存储卷名称
      mountPath:  # 挂载容器内路径
      readOnly: false # 是否挂载为只读卷
      subPath:  # 挂载存储卷时使用的子路径,在 mountPath 指定路径下使用一个子路径作为其挂载点 

    resources: # pod资源相关
      requests:
        cpu: "1"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "512Mi"
    
    livenessProbe/ReadinessProbe: # 存活/就绪状态监测策略
      exec | httpGet | tcpSocket: # 执行命令 | 发送 http get 请求 | 端口检测
      initialDelaySeconds: 第一次探测的延时时间
      periodSeconds: 探测时间间隔
      failureThreshold: 失败多少次后判断失败

    lifecycle:  # 启动后,停止前的钩子策略
      postStart/preStop: 启动后,停止前
        exec | httpGet | tcpSocket: # 执行命令 | 发送 http get 请求 | 端口检测
  
  ##### 存储卷相关 #####
  volumes: # 存储卷相关
  - name:  # 要挂载卷的唯一名称
    <type>: # 常用的包括 emptyDir,hostPath,configMap,secret,nfs,rbd,glusterfs,persistentVolumeClaim
    
    # <type> = emptyDir # 多用于同一 Pod 中容器间共享路径
    emptyDir: {}
      
    # <type> = hostPath # 多用于测试
    hostPath:
      path:  # 挂载的本地路径
      type:  # 本地路径的体现方式,详见 https://kubernetes.io/docs/concepts/storage/volumes#hostpath
    
    # <type> = configMap # 挂载 configMap 作为存储卷
    configMap:
      name:  # 指定 configMap 名称
      items: # 挂载 configMap 对象中部分 items,可选参数
      - key:  # configMap 中的键(文件名)
        path:  # 挂载后的相对路径
        mode:  # 挂载的文件访问权限,如 0644
      defaultMode: 0644 # 默认的文件访问权限
      optional: false # configMap 对象是否必须被定义
        
    # <type> = secret # 挂载 secret 作为存储卷,用法基本与 ConfigMap 对象挂载为存储卷相同
    secret:
      secretName:  # 指定 secret 名称
    
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
        
    # <type> = persistentVolumeClaim
    persistentVolumeClaim:
      claimName:  # PVC 名称
      readOnly: false

  ##### Pod/节点亲和相关 #####
  affinity: 
    nodeAffinity: # 节点亲和
      preferredDuringSchedulingIgnoredDuringExecution: # 尽量满足
      - preference:
          matchFields:
          - {key: , operator: , values: []}
        weight:  # 设置权重
      requiredDuringSchedulingIgnoredDuringExecution: # 必须满足
        nodeSelectorTerms: # 节点选择条件列表
        - matchFields:
          - {key: , operator: , values: []}
        - matchExpressions:
          - {key: , operator: , values: []}

    podAffinity: # pod 亲和
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            <key>: <value>
        topologyKey: # 指定键的值相同的 Pod 在同一位置
      
      preferredDuringSchedulingIgnoredDuringExecution:
      - podAffinityTerm:
          labelSelector:
            matchLabels:
              <key>: <value>
          topologyKey: # 指定键的值相同的 Pod 最好在同一位置
        weight:   # 设置权重
    
    podAntiAffinity: # Pod 反亲和,定义方式同 podAffinity
    # ...
  
  ##### 污点容忍相关 #####
  tolerations:
  - key: # 能够容忍的污点的key
    value: # 能够容忍的污点的value
    effect: # 能够容忍的污点的程度,后面会有介绍.可选值为 NoSchedule,NoExecute 或 PreferNoSchedule
    operator: # 对能够容忍的污点和 node 节点上的污点做比较.可选值为 Exists 或 Equal,存在或相等
    tolerationSeconds: # 容忍时间,要求 effect 必须不能设置为 NoExecute
```
