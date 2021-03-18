---
title: K8S 实战 - 02 快速开始
date: 2020/06/24
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第二个章节.主要了解 K8S 集群的核心对象,并使用 kubeadm 简单部署集群,了解 K8S 命令行管理工具 kubectl 的使用
---

## Kubernetes 的核心对象

### Pod 资源对象

Pod 资源对象是一种集合了一个到多个应用容器,存储资源,专用 IP 及支撑容器运行的其它选项的逻辑组件,是 Kubernetes 的部署单元及原子运行单元.它通常由共享资源紧密的一个或多个应用容器组成.

Kubernetes 的网络模型要求各 Pod 对象的 IP 地址位于同一网络平面中,各 Pod 之间使用其 IP 地址直接进行通信,无论它们运行于集群内的哪个工作节点上,这些 Pod 对象都像是运行于同一局域网中的多个主机

Pod 对象中的各进程均运行于彼此隔离的容器中,并于各容器间共享两种关键资源: 网络和存储卷

- 网络

每个 Pod 对象都会被分配一个集群内专用的 IP 地址,称为 Pod IP.同一 Pod 内部的所有容器共享 Pod 对象的 Network 和 UTS 名称空间,包括主机名,IP 地址和端口等.因此,Pod 内容器间通信可基于本地回环接口进行,Pod 外的其它组件通信则需要使用 Service 资源对象的 ClusterIP 及其相应端口完成

- 存储卷
  
存储卷资源可以共享给 Pod 内部的所有容器使用,从而完成容器间数据的共享.存储卷还可以确保在容器终止后被重启/删除后数据不会丢失,从而保证了声明周期内的 Pod 对象数据的持久化存储.

一个 Pod 对象代表某个应用程序的一个特定实例,如果需要扩展应用程序,则需要为此应用程序同时创建多个 Pod 实例,每个实例均代表应用程序的一个运行的副本.这些副本化的 Pod 对象的创建和管理通常被称为"控制器".

创建 Pod 时,可以使用 Pod Preset 对象为 Pod 注入特定的信息,如 ConfigMap,Secret,存储卷和环境变量等.

基于期望的目标状态和各节点的资源可用性,Master 会将 Pod 对象调度至某选定的工作节点运行,工作节点从指定的镜像仓库下载镜像,并于本地的容器运行时环境中启动容器.Master 会将整个集群的状态保存于 etcd 中,并通过 API Server 共享给集群中的各组件及客户端.

### Controller

Kubernetes 使用控制器实现对 Pod 对象的管理操作,从而实现 Pod 对象的扩缩容,滚动更新和自愈能力等

控制器本身也是一种资源类型,其中与工作负载相关的实现如 `ReplicationController,Deployment,StatefulSet,DaemonSet,Jobs`等,可以统称它们为 Pod 控制器

Pod 控制器的定义通常由期望的 Pod 副本数量,Pod 模版和标签选择器组成.Pod 控制器会根据标签选择器对 Pod 对象的标签进行匹配检查,所有满足选择条件的 Pod 对象都将受控于当前控制器并计入其副本总数,并确保此数目能够精确反映期望的副本数.

在实际应用场景中,用户需要手动修改 Pod 控制器中期望副本数量以实现应用规模的扩缩容.若集群中部署了 HeapSter 或 Prometheus 一类的资源指标监控附件时,用户可以使用 "HorizontalPodAutoscaler(HPA)" 计算出合适的 Pod 副本数量,并自动修改 Pod 控制器中期望的副本数以实现应用规模的动态伸缩,提高资源利用率

Kubernetes 集群中每个节点通过 cAdvisor 收集容器及节点的 CPU,内存,磁盘等资源的利用率指标数据,HPA 基于这些统计数据监控容器健康状态并做出扩展决策.

### Service

尽管 Pod 对象拥有 IP 地址,但此地址无法确保在 Pod 对象重启或重建后保持不变.Service 资源用于被访问的 Pod 对象中添加一个有着固定 IP 地址的中间层,客户端地址发起请求后由相关的 Service 资源调度并代理至后端的 Pod 对象

Service 是微服务的一种实现,实际上是通过规则定义出由多个 Pod 对象组合而成的逻辑集合,并附带访问这组 Pod 对象的策略.Service 对象挑选,关联 Pod 对象的方式同 Pod 控制器一样,都需要基于标签选择器进行定义.

Service 主要有三种常用类型:

- 仅用于集群内部通信的 `ClusterIP`
- 接入集群外部请求的 `NodePort` 类型,它通过端口映射来实现,工作于每个节点的主机 IP 之上
- `LoadBalancer` 类型,将外部请求负载均衡至多个节点的 NodePort 之上.它主要用于云厂商环境中,使用云厂商提供的 LoadBalancer 负载均衡提供服务.

## 部署应用程序的主要过程

用户只需要向 API Server 请求创建一个 Pod 控制器,由控制器根据镜像等信息向 API Server 请求创建出一定数量的 Pod 对象,并由 Master 之上的调度器指派至选定的工作节点以运行容器化应用.

用户还需要创建一个集体的 Service 对象以便为这些 Pod 对象建立起一个固定的访问入口,从而使客户端能够通过服务名称或 ClusterIP 进行访问

## 部署 Kubernetes 集群

## kubeadm 部署工具

kubeadm 仅关心如何初始化并启动集群,余下的其它操作则不在其考虑范围之内.

kubeadm 集成了众多管理 Kubernetes 集群的命令行工具,其中

- `kubeadm init` 用于集群的快速初始化,其核心功能是部署 Master 节点的各个组件.
- `kubeadm join` 用于将节点快速加入到指定集群中
- `kubeadm token` 可在集群构建后管理用于加入集群时使用的认证令牌(token)
- `kubeadm reset` 删除构建集群过程中生成的文件并重置回初始状态
- `kubeadm config` 管理 kubeadm 集群配置,该配置保留在集群的 ConfigMap 中.常用如下
  - `kubeadm config images list/pull` 列出/拉取 kubeadm 构建需要的镜像
  - `kubeadm config print init-defaults/join-defaults` 打印初始化或加入节点的默认配置
  - `kubeadm config view` 查看当前 kubeadm 集群的配置
- `kubeadm upgrade` 用于平滑的升级 K8S 集群

```bash
# 集群初始化
kubeadm init \
  --config=/etc/kubernetes/kubeadm.yml  \ # 指定初始化集群的相关配置文件
  # api-server 相关
  --control-plane-endpoint=${vip:port} \ # 指定用于 api-server 高可用的代理地址,VIP:port
  --apiserver-advertise-address=${ip} \ # 指定 api-server 的 IP 地址
  --apiserver-bind-port=6443 \ # 指定 api-server 监听端口

  # 证书相关
  --cert-dir=/etc/kubernetes/pki \ # 指定初始化过程中生成的证书地址
  --cri-socket=/var/run/dockershim.sock \ # 默认为空,docker 默认使用 /var/run/dockershim.sock 作为 cri-socket
  --image-repository=${image-repo} \ # 指定拉取镜像的镜像仓库
  --kubernetes-version=${k8s-version} \ # 指定 k8s 集群的版本,默认为 kubeadm 版本
  --service-cidr=10.96.0.0/16 \ # 指定 service VIP 地址段
  --pod-network-cidr=10.244.0.0/16 \ # 指定 pod 网络地址段
  --node-name=${HOSTNAME}
  --token=abcdef.0123456789abcdef \ # 指定节点与集群间建立信任的 token
  --upload-certs \ # 将证书信息上传到 k8s 集群 kubeadm-certs Secret 对象
  --ignore-preflight-errors=Swap,IsPrivilegedUser,all \ # 忽略检查过程中产生的报错,而使用警告方式通知

```

集群初始化示例如下:

```bash
kubeadm init --kubernetes-version=v1.15.0 --service-cidr=10.96.0.0/16 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.2.3 --image-repository=registry.cn-hangzhou.aliyuncs.com/k8sxio
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 添加节点
kubeadm join 192.168.2.3:6443 --token 1cvajd.j94i1p8vxzc5kim9 \
--discovery-token-ca-cert-hash sha256:27a53f3a71536eb9c78c380f302866ed6801b221fbbcc4ad71d632a19a9cfcda
```

其中,另外还可以使用配置文件方式初始化 kubernetes 集群,使用 `kubeadm config print init-defaults` 打印出的默认配置文件如下,可以在此基础上稍作修改

```yml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef  # 可使用 kubeadm token generate 生成随机 token
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
# 高可用 api-server 
# controlPlaneEndpoint: "k8s-lb:16443"
# 单机 api-server 
localAPIEndpoint:
  advertiseAddress: 1.2.3.4   # 单机 api-server 地址
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-m3
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
  # 支持使用自建的 etcd
  # external:
    # endpoints:
    # - https://192.168.2.2:2379
    # - https://192.168.2.3:2379
    # - https://192.168.2.4:2379
    # caFile: /etc/kubernetes/pki/etcd/ca.pem  #搭建 etcd 集群时生成的ca证书
    # certFile: /etc/kubernetes/pki/apiserver-etcd-client.pem   # 访问 etcd 的客户端证书
    # keyFile: /etc/kubernetes/pki/apiserver-etcd-client-key.pem  # 访问 etcd 的客户端证书密钥文件

imageRepository: k8s.gcr.io # 默认的镜像仓库
kind: ClusterConfiguration
kubernetesVersion: v1.19.0 # 默认的 k8s 版本
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

### 2.2.2 集群运行模式

- 独立组件模式: 系统组件直接以守护进程方式运行于节点之上,各个组件之间相互协作构成集群.期间需要用到的证书及 Token 等认证信息也需要手动生成,过程较为繁琐.
- 静态 Pod 模式: 除 kubelet 和 Docker 之外的其它组件都是都是以静态 Pod 对象运行于 Master 主机上

## 2.3 kubectl 命令详解

详见 [kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands) 命令行官网

```bash
get: 显示一个或多个资源
    pods(pods信息) | cs/componentstatus(运行状态信息) | nodes(节点) | namespace(名称空间) | deployment(资源部署) | svc(pods的service): 相关资源可通过 `kubectl api-resources` 查看
 -o --output wide | json | yaml: 指定输出格式
 -n default | kube-system | kube-public : 指定名称空间
 -l --selector= key [=value] : 指定带有指定标签或标签值的资源
 -L --label-column= k1,k2: 输出各个指定标签的标签值
    --show-labels : 输出 label 信息

create: 根据命令行临时创建一个资源,可能主要用于测试
    # 创建用于 imagePullSecret 指定拉取镜像的认证信息
    kubectl create secret docker-registry <NAME> 
      --docker-username=user --docker-password=password
      --docker-email=email --docker-server=	https://index.docker.io/v1/

run: 在集群上运行镜像
    kubectl run <NAME> --image=<IMAGE> : 运行镜像,并指定pod名称<NAME>
    --port=<PORT>: 指定被暴露的端口
    --env="<KEY>=<VALUE>": 环境变量,可以指定多次,但每次只能指定一个变量
 -l --labels="<K1>=<V1,K2=V2>": 指定标签,可以一次指定多个
 -r --replicas=1: 指定运行的实例个数
    --dry-run= false | true: 如果为true,则打印相应的API对象而不创建
    --restart= Always | OnFailure | Never: pod的重启策略
    -- <COMMAND> <arg1> <arg2> ... <argN>: 运行command及其参数,而不运行镜像默认程序
    --schedule="0/5 * * * ?": 开启周期调度任务
    --expose= false | true : 如果为true,则为运行的容器创建公共外部服务,相当于 `kubectl expose`
    --limits='': 资源限定,如 cpu=200m,memory=512Mi
 -o --output wide | json | yaml: 指定输出格式
 -it: 交互式启动
 -w: 监控,类似于 watch 命令

apply: 将指定文件中的配置用于创建资源
 -f --filename: 指定yaml文件创建资源

config: kubectl的配置相关
    current-context: 打印当前使用的上下文
    set-cluster NAME --server=<serverIP:PORT> --certificate-authority=<ca.crt> --embed-certs=true: 添加集群
    set-credentials NAME --client-certificate=<crt_file> --client-key=<key_file> --embed-certs=true: 添加用户
    set-context NAME --cluster=<cluster_name> --user=<user_name>: 添加/设置上下文
    delete-cluster NAME: 删除指定集群
    use-context CONTEXT_NAME: 使用指定上下文
    view: 打印默认的 kubeconfig

describe: 资源或资源组的详细信
    pod | svc | node | deployment [name]
 -l --selector='': 通过指定的标签筛选查看,如 key=value  

edit: 对资源配置进行编辑,默认使用vi编辑器
    RESOURCE NAME: 如 `kubectl edit svc <svc_name>`

explain: 列出资源支持的字段.可用作构建yaml文件时的参考文档
    kubectl explain RESOURCE: RESOURCE 支持多级嵌套

expose: 暴露出一个资源作为 kubenetes 公共外部服务,可以理解为 k8s 中的 pods 的 service
    deployment <pod名称>: 创建service服务
    --name=<deployment_name>: 指定公共服务名称
    --port=<PORT>: 指定映射pod中的端口
    --target-port=<PORT>: 指定映射 service 中的端口
    --protocol= TCP | UDP : 指定协议
    --type=
        ClusterIP: 只有 k8s 集群才可以访问
        NodePort: 将端口映射到本地 kube-proxy 进程,端口随机.可通过外部服务进行访问

label: 为已经创建/存在的资源添加标签
    kubectl label TYPE NAME KEY=VALUE
    --overwrite: 是否进行覆盖,如果不指定,且已经有该标签,则报错
    --all: 更新所有名称空间内的资源标签
    # 为 k8s 集群节点添加/删除 role
    # kubectl label node k8s-m1 node-role.kubernetes.io/master=
    # kubectl label node k8s-m1 node-role.kubernetes.io/master-

patch: 使用指定的 PATCH 合并修补程序, PATCH 格式为 JSON 格式
    kubectl patch TYPE NAME -p PATCH 

rollout: 管理资源的部署
    history: 查看 deployment 的更新历史, 如, `kubectl rollout history deployment deploy-demo`
    undo: 对之前的设定做回滚,如 `kubectl rollout undo deployment nginx-deploy`
        --to-revision: 指定回滚到哪个版本,默认回滚到上一个版本

scale: 改变 Deployment,ReplicaSet 等的对象的数量
    --replicas=<num> TYPE NAME

set: 设定对象的特定功能
    env(环境变量)
    image(镜像):
        TYPE NAME CONTAINER_NAME=CONTAINER_IMAGE: 更换镜像,如 `kubectl set image deployment nginx-deploy nginx-deploy=<image_name>` 

taint: 管理污点
    kubectl taint <node> <污点名> key=value
    # 为 k8s 节点添加/删除 NoSchedule
    # kubectl taint nodes k8s-master node-role.kubernetes.io/master:NoSchedule
    # kubectl taint nodes k8s-master node-role.kubernetes.io/master:NoSchedule-

token: 管理 token
    create: 创建token
      --print-join-command: 创建加入节点的token
```
