---
title: K8S 实战 - 11 网络模型与网络策略
date: 2021/03/19
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第十一个章节.主要了解 K8S 集群的网络模型及网络策略,了解相关网络插件的使用
---

在了解 Kubernetes 网络模型之前,我们先来看一下 Docker 的网络模型

## Docker 容器网络模型

Docker 网络模型的原始模型主要有三种:

- `Bridge`: 桥接,借助于虚拟网桥设备为容器建立网络连接.是 Docker 默认的网络模型
- `Host`: 主机,设定容器直接共享使用节点主机的网络名称空间
- `Container`: 容器,多个容器共享同一个网络名称空间,彼此之间能够以本地通信的方式建立连接
- `None`: 不使用网络

因此,在创建 Docker 容器时,默认有四种网络可供选择使用,具体如下:

- Closed container(封闭式容器): 此类容器使用 "None" 网络,它们没有对外通信的网络接口,通常用于不需要网络的后端作业处理场景
- Bridge container(桥接式容器): 此类容器使用 "Bridge" 模型的网络,容器引擎会为每个容器创建一对(两个)虚拟以太网设备,一个配置为容器的网络接口,另一个在节点主机上接入指定的虚拟网桥设备(默认为 docker0)
- Open container(开放式容器): 此类容器使用 "Host" 模型的网络,共享使用 Docker 主机的网络及器接口
- Joined container(联盟式容器): 此类容器共享使用某个已存在的容器的网络名称空间,共享使用指定容器的网络及接口.

Docker 进程首次启动时,它会在当前节点上创建一个名为 `docker0` 的桥设备,并默认配置其使用 `172.17.0.0/16` 的网络(可通过 `bridge` 进行配置),该网络是 Bridge 模型的一种实现,也是创建 Docker 容器时默认使用的网络模型.

跨节点的容器间进行通信时,每个节点上的容器都将从 `172.17.0.0/16` 网络中获取 IP 地址,导致不同的 Docker 主机上的容器可能会使用相同的地址,因此它们无法直接通信.

而解决此问题的通信方式是 NAT.所有发往 Docker 主机外部的流量都会在执行过程中进行源地址转换,接入 Docker 主机外部的流量,则需要通过目标地址转换或端口映射将其暴露于外部网络中.因此,多节点上的 Docker 容器间通信依赖 NAT 机制转发实现.

## Kubernetes 网络模型

Kubernetes 网络模型主要可用于解决四类通信请求:

- 同一 Pod 内容器间通信

Pod 对象内各容器共享同一网络名称空间,彼此间可以通过 lo 接口完成交互

- Pod 间通信

各 Pod 对象运行于同一个平面网络中,每个 Pod 对象拥有一个集群全局唯一的地址,并直接可以与其它 Pod 进行通信.此网络称为 Pod 网络.

运行 Pod 的各节点也会通过桥接设备持有此平面网络的一个 IP 地址,因此 Node 到 Pod 间通信也可以在此网络上直接进行.

- Service 到 Pod 间通信

Service 资源的专用网络也称为集群网络(Cluster Network),需要在启动 kube-apiserver 时通过 `--service-cluster-ip-range` 选项进行指定,如 `10.96.0.0/12`.每个 Service 对象在此网络有拥有一个称为 ClusterIP 的固定地址.

Service 创建完成后,会触发各节点上的 kube-proxy,并在相应节点上创建 iptables 或 ipvs 规则,完成从 Service 的 ClusterIP 与 PodIP 之间的报文转发.

- 集群外部与 Service 间通信

Service 有多种类型,常用的有如下三种: ClusterIP(默认),NodePort,LoadBalancer

ClusterIP 类型的 Service 只会得到虚拟的 ServiceIP 和 ServicePort,只能在 Kubernetes 集群内部被访问.

NodePort 类型的 Service 在 ClusterIP 的基础上构建,它 会得到虚拟的 ServiceIP 和 ServicePort 用于 Kubernetes 集群内部通信,Kubernetes 还会在所有 Node 节点上为其分配端口,用于从集群外部通过 NodeIP:nodePort 访问.分配的端口的值可以通过`spec.ports[*].nodePort` 指定,或由 Kubernetes 自行分配,默认为30000-32767.

LoadBalancer 类型的 Service 在 NodePort 的基础上构建,并为其开通负载均衡,一般需要云厂商设备的支持.

## flannel 网络插件

```bash
# 安装 flannel, yaml 文件地址可通过 https://github.com/flannel-io/flannel 获得
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

flannel 解决了多节点 Docker 主机中容器通信中时的两个问题

- 各 Docker 主机在 docker0 桥上默认使用同一个子网,不同节点的容器可能会得到相同的地址,跨节点的容器间通信会面临地址冲突的问题
- 各个节点上网络中缺乏路由信息,报文无法准确送达

解决办法如下:

- 预留使用一个网络,为每各个节点的 Docker 容器引擎分配一个子网(如 10.244.1.0/24 和 10.244.2.0/24),并将分配信息保存于 etcd 持久存储.
- 支持 VxLAN, host-gw, UDP 网路模型,使用"后端"解决相关问题
  - `VxLAN`: 使用 VxLAN 模块封装报文
  - `host-gw`: Host GateWay,在节点上创建到达目标容器的路由完成报文转发,这种方式要求各节点本身在一个二层网络中.
  - UDP: 使用普通 UDP 报文封装完成隧道转发,性能较低

### flannel 配置参数

默认情况下,flannel 的配置信息保存于 etcd 的键名 `/coreos.com/netwok/config` 之下,它的值是一个 JSON 格式的字典数据,它可以使用的键包含以下几个:

- `Network`: flannel 在全局使用的 CIDR 格式的 IPv4 网络,字符串格式,必选字段
- `SubnetLen`: 将 Network 属性指定的 IPv4 网络基于指定位的掩码切割为供节点使用的子网.当网络的掩码小于24时,默认为24
- `SubnetMin/SubnetMax`: 可用作分配给节点使用的子网范围
- `Backend`: flannel 要使用的后端类型,以及后端的相关配置.包含 type(类型,默认为 VxLAN)和 Port(端口)字段

配置示例如下:

```json
{
    "Network": "10.244.0.0/16",
    "SubnetLen": 24,
    "Backend": {
        "Type": "VxLAN",
        "Port": 8472
    }
}
```

## 网络策略

网络策略(Network Policy)用于控制分组的 Pod 资源之间如何进行通信,它为 Kubernetes 实现了更为精细的流量控制,实现租户隔离控制. Kuberntes 使用资源对象 `NetworkPolicy` 定义网络控制策略.

Pod 网络流量包含 "流入(Ingress)" 和 "流出(Egress)" 两种方向,每种方向的控制包含 "允许" 和 "禁止" 两种.默认情况下,Pod 的流量可以自由来去,一旦有策略应用于 Pod,那么所有为经明确允许的流量都将被拒绝,

网络策略需要使用 Cannal 或 Calico 提供,安装方式如下,详情请见[官方文档](https://docs.projectcalico.org/)

```bash
kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/canal.yaml
```

一般来说,网络策略定义如下:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: 
  namesapce: 
spec:
  podSelector:  # pod 标签选择器,{}表示所有
    matchLabels:
      key: value
  policyTypes:
  - Ingress # 表示入站流量管理策略启用
  - Egress  # 表示出站流量管理策略启用
  ingress/egress: # 定义入站/出站策略.如果不定义此字段,则表示禁止所有入站/出站流量;如果该字段值为{},则表示允许所有入站/出站流量
  - from/to:  # 入站或出站流量的列表对象,可以根据如下方式进行限定
    - ipBlock: # 地址范围
        cidr: 10.244.1.0/24 # 是一个 CIDR 地址,格式为:网段/掩码
        except: 10.244.1.2/32 # 是一个 CIDR 地址,格式为:网段/掩码,表示除了这个
    - namespaceSeletor: # 名称空间选择器
        matchLabels:
          key: value
    - podSelector:  # pod 选择器
        matchLabels:
          key: value
    ports:
      - port: 80
        protocol: TCP/UDP
```

一般来说,我们会将同一名称空间下的 Pod 的入站流量全部禁止,只留下特定应用端口的入站流量,将除 kube-system 外的不同名称空间下的 Pod 进行隔离.故默认定义如下:

```yaml
---
# 禁止所有名称空间中 Pod 的入站出站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-other-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 开放 kube-system 和自身名称空间中 Pod 出站入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-namespace-default
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchExpressions:
        - key:name
          operator: In
          values:
          - kube-system
          - default
  egress:
  - to:
    - namespaceSelector:
        matchExpressions:
        - key:name
          operator: In
          values:
          - kube-system
          - default

---
# 禁止所有 Pod 入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# 定义允许 Pod 的入站流量,开放应用端口的入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-xxx-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from: {}
    ports:
    - protocol : TCP
      port: 80
```
