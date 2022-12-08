---
title: 运维面试题之 Kubernetes
date: 2020/06/04
tags:
  - Kubernetes
  - 面试
categories:
  - Kubernetes
abbrlink: 
description: 总结整理常见 Kubernetes 面试题,以作备忘
---

## 原理

### Kubernetes 组件

- etcd: 提供数据库服务,保存了整个集群的状态
- kube-apiserver: 提供了资源操作的唯一入口,并提供认证,授权,访问控制,API注册和发现等机制
- kube-controller-manager: 负责维护集群的状态,比如故障检测,自动扩展,滚动更新等
- cloud-controller-manager: 是与底层云计算服务商交互的控制器
- kub-scheduler: 负责资源的调度,按照预定的调度策略将 Pod 调度到相应的机器上
- kubelet: 负责维护 Pod 的生命周期,同时也负责 Volume 和网络的管理
- kube-proxy: 负责为 Service 提供内部的服务发现和负载均衡,并维护网络规则
- container-runtime: 是负责管理运行容器的软件,比如 docker

> master

- kube-apiserver: 暴露 kubernetes 所有API,为 api 对象验证并配置数据.API Server 提供 REST 操作和到集群共享状态的前端,所有其他组件通过它进行交互
- kube-controller-manager: 通过 apiserver 监控整个集群的状态，并确保集群处于预期的工作状态
- kube-scheduler: 负责工作节点上工作负载的分配和管理

> node

- kubelet: 定时汇报当前节点的状态给 apiserver;获取 pod 的期望状态,调用对应的容器平台接口达到这个状态;镜像和容器的清理工作,保证节点上镜像不会占满磁盘空间,退出的容器不会占用太多资源
- kube-proxy: 该进程可以看做是 service 的透明代理和负载均衡器.其核心功能是将某个 service 的访问请求转发到后端的某个 Pod 上. 有 Userspace,Iptables,IPVS 三种实现方式,默认 IPVS

### 简述 Kubernetes 的工作流程

Kubernetes 各个组件中的通信都是以 HTTPS 方式进程的.Kubernetes 各个组件均使用 watch 机制跟踪检查 API Server 上相关变动

1. kubectl 客户端工具会校验请求资源合法性,并将相关资源对象封装成 HTTP 请求,以加密方式发送给 kube-apiserver
2. kube-apiserver 接收到 HTTPS 请求后,对请求来源进行认证,然后进行授权及准入控制的校验.校验通过后,将修改写入 etcd.响应客户端的同时,调用 kube-controller-manager 进行处理
3. kube-controller-manager 对集群中资源对象副本 ReplicaSet 进行检查.如果已经达到预期状态,则不作调整;否则调用 kube-schedule 进行处理
4. kube-scheduler 收到信号后进行调度,包括预选/优选调度,并将结果返回给 API Server,后写入 etcd.如果资源不够,资源对象会进入 Pending 等待状态
5. kubelet 根据调度结果调用 CRI(Container Runtime Interface)执行 Pod 资源创建/回收操作.

![what-happens-when-k8s](/images/what-happens-when-k8s.svg)

### Kubernetes 容器间通信方式

- 相同 Pod 中的容器间可使用 localhost 直接通信
- 相同 Node,不同 Pod 间可通过 docker0 网桥直接通信
- 不同 Node,不同 Pod 间可通过 Node 上的 flannel0 虚拟网卡进行路由转发,从 docker0 转发到 flannel0
- Pod 访问 Service 通过 kube-proxy 进程创建的 ipvs 规则进行通信

### Kubernetes 中的服务类型

- `ClusterIP`: 通过集群内部 IP 地址暴露服务,此地址仅在集群内部可达,而无法被集群外部客户端访问
- `NodePort`: 这种类型建立在 ClusterIP 之上,其在每个节点的 IP 地址的静态端口用于将集群外部的用户请求转发至目标 Service 的 ClusterIP 和 Port.这种类型的 Service 既可以通过 `<ClusterIP:ServicePort>` 进行访问,又可以通过 `<NodeIP>:<NodePort>` 进行访问
- `LoadBalancer`: 这种类型构建在 NodePort 类型之上,其通过云厂商提供的负载均衡器将服务暴露到集群外部.此类型的 Service 会指向关联至 Kubernetes 集群外部的切实存在的某个负载均衡设备,该设备通过工作节点上的 NodePort 向集群内部发送请求流量.此类型优势在于负载均衡设备能够避免客户端指定节点故障而导致服务不可用
- `ExternalName`: 通过 Service 映射至 externalName 字段内容指定的主机名来暴露服务,此主机名需要被 DNS 服务器解析至 CNAME 类型的记录.主要用于将集群外部的服务以 DNS CNAME 的方式映射到集群中,从而让集群内的 Pod 资源能够访问外部 Service 的一种实现方式.

### Kubernetes 负载均衡

通过 service/kube-proxy 实现四层负载均衡,通过 ingress 实现七层负载均衡,常用控制工具有 ingress-controller,traefik

### kube-proxy 原理

kube-proxy 部署在每个 Node 节点上,通过监听集群状态变更,并对本机 iptables/ipvs 做修改,从而实现网络路由.而其中的负载均衡,也是通过 iptables/ipvs 的特性实现的

### kubernetes 中 pause 容器是做什么用的

[参考](https://www.ianlewis.org/en/almighty-pause-container)

- 作为 Pod 共享名称空间的基础容器
- 启动 init 进程,并共享 PID 名称空间,接收信号并作出处理,完成 Pod 的生命周期

### Pod的生命周期

Pod 状态始终处于一下几个状态之一:

- Pending: 部署 Pod 事务已被集群受理,但当前容器镜像还未下载完或现有资源无法满足 Pod 的资源需求
- Running: 所有容器已被创建,并被部署到节点上
- Successed: Pod成功退出,并不会被重启
- Failed: Pod中有容器被终止
- Unknown: 未知原因,如 kube-apiserver 无法与Pod进行通讯

```text
init 初始化容器
主容器
    post start hook 容器创建完成后立即运行的钩子函数
    livenessProbe 存活状态检测
    readinessProbe 就绪状态检测
    pre stop hook 容器删除之前的钩子函数
```

### Kubernetes 中容器的健康状态检测

kubernetes 在创建 Pod 时,可以为 Pod 指定 `LivenessProbe` 相关参数即可进行存活状态检测,指定 `ReadinessProbe` 即可完成就绪状态检测

检测方式包括

- `ExecAction`: 执行命令.如果状态码为 0,表示健康/就绪
- `HTTPGetAction`: 向指定 url 发送 http 请求.如果 http code 为 200-400,则表示健康/就绪
- `TCPSocketAction`: 对指定端口进行 TCP 检查.如果能建立连接,则表示健康/就绪

相关检查会反馈到 `kubectl describe <pod>` 的 status.conditions 字段中

## 实践

### 如何在 Kubernetes 集群中自定义 hosts

- 在集群中添加自定义 hosts

```yaml
# coredns-configmap.yaml 支持 hosts 插件,可以自定义解析
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        hosts {
            1.1.1.1 cache.redis
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

- 在某类 Pod 中添加自定义 hosts

```yaml
# 在 Pod 的 spec 字段中定义 hostAliases 字段
spec:
  hostAliases:
  - ip: "x.x.x.x"
    hostnames:
    - "hostname1_for_x.x.x.x"
    - "hostname2_for_x.x.x.x"
```

### Pod 时间同步

将物理机的时区文件以 hostspath 方式只读挂载,只需要保证系统时间是正确的即可

```yaml
spec:
  containers:
    volumeMounts:
    - name: vol-localtime
      mountPath: /etc/localtime
      readOnly: true
  volumes:
  - name: vol-localtime
    hostPath:
      path: /etc/localtime
```

### 集群内外网络互通问题

- Kubernetes 集群内部 node 上的 Pod 网络通信是通过 cni 网络插件实现的,路由信息保存在 etcd 中,并通过 node 节点上 iptables/ipvs  实现
- 集群外部的主机需要手动添加路由,将 Pod 网络的下一跳地址指向响应的 node 节点即可

```bash
route add -net 10.244.0.0 netmask 255.255.0.0 gw <nodeIP>
```

### 拉取私有镜像

kubernetes 提供了 `imagePullSecret` 配置从私有镜像仓库中拉取镜像.如下:

```bash
kubectl create secret docker-registry NAME --docker-username=<username> --docker-password=<password> --docker-email=<email> --docker-server=<docker-registry-url> [-n <namespace>]

kubectl patch sa <sa_name> -p '{"imagePullSecrets": [{"name": "$NAME"}]}'
```

需要注意的是如上过程并不能拉取私有镜像仓库中的 pause 镜像.
