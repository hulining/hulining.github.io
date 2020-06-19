---
title: Kubernetes 相关错误及其解决
date: 2020/04/03
tags:
  - Kubernetes
  - etcd
categories:
  - Kubernetes
abbrlink: 57264
description: 本文章为学习 Kubernetes 及其相关组件中遇到的问题及解决办法
---

## etcd

> - [x] [执行命令时间太长](https://github.com/etcd-io/etcd/issues/10174)

日志

```log
2019-07-22 03:33:16.621867 W | etcdserver: read-only range request "key:\"/registry/namespaces/kube-system\" " with result "range_response_count:1 size:177" took too long (265.312097ms) to execute
2019-07-22 03:33:16.622103 W | etcdserver: read-only range request "key:\"/registry/health\" " with result "range_response_count:0 size:4" took too long (255.015614ms) to execute
2019-07-22 03:33:16.622848 W | etcdserver: read-only range request "key:\"/registry/clusterroles/\" range_end:\"/registry/clusterroles0\" " with result "range_response_count:0 size:4" took too long (266.165829ms) to execute
2019-07-22 03:33:16.625493 W | etcdserver: read-only range request "key:\"/registry/health\" " with result "range_response_count:0 size:4" took too long (231.260569ms) to execute
```

原因

- [磁盘IO瓶颈](https://github.com/etcd-io/etcd/blob/master/Documentation/faq.md#what-does-the-etcd-warning-apply-entries-took-too-long-mean)

> - [x] [负载过高](https://github.com/etcd-io/etcd/issues/5154)

环境

```bash
# etcd 二进制启动
etcd --name vm1 --initial-advertise-peer-urls http://192.168.2.3:12380 \
  --listen-peer-urls http://192.168.2.3:12380 \
  --listen-client-urls http://192.168.2.3:12379,http://127.0.0.1:12379 \
  --advertise-client-urls http://192.168.2.3:12379 \
  --initial-cluster-token etcd-cluster-bin \
  --initial-cluster vm1=http://192.168.2.3:12380,vm2=http://192.168.2.4:12380 \
  --initial-cluster-state new
```

日志

```log
2019-08-20 16:58:45.366953 W | etcdserver: failed to send out heartbeat on time (exceeded the 100ms timeout for 154.491083ms, to 7ceb206ff2a779a3)
2019-08-20 16:58:45.366979 W | etcdserver: server is likely overloaded
2019-08-20 16:59:14.442020 W | etcdserver: failed to send out heartbeat on time (exceeded the 100ms timeout for 44.374276ms, to 7ceb206ff2a779a3)
2019-08-20 16:59:14.442049 W | etcdserver: server is likely overloaded
2019-08-20 16:59:38.078540 W | etcdserver: failed to send out heartbeat on time (exceeded the 100ms timeout for 380.617868ms, to 7ceb206ff2a779a3)
```

解决: 添加`--heartbeat-interval 250 --election-timeout 1250`启动参数

## kube-apiserver

> - [ ] [unknow](https://github.com/kubernetes/kubernetes/issues/76956)

```log
E0722 03:32:55.266253       1 prometheus.go:55] failed to register depth metric admission_quota_controller: duplicate metrics collector registration attempted
E0722 03:32:55.266276       1 prometheus.go:68] failed to register adds metric admission_quota_controller: duplicate metrics collector registration attempted
E0722 03:32:55.266294       1 prometheus.go:82] failed to register latency metric admission_quota_controller: duplicate metrics collector registration attempted
E0722 03:32:55.266308       1 prometheus.go:96] failed to register workDuration metric admission_quota_controller: duplicate metrics collector registration attempted
E0722 03:32:55.266330       1 prometheus.go:112] failed to register unfinished metric admission_quota_controller: duplicate metrics collector registration attempted
E0722 03:32:55.266346       1 prometheus.go:126] failed to register unfinished metric admission_quota_controller: duplicate metrics collector registration attempted
```

## metrics-server

> - [x] [metrics 不可用](https://github.com/kubernetes-incubator/metrics-server/issues/247)

```log
E0821 01:20:44.192985       1 reststorage.go:128] unable to fetch node metrics for node "vm1": no metrics known for node
E0821 01:20:44.193003       1 reststorage.go:128] unable to fetch node metrics for node "vm2": no metrics known for node
E0821 01:25:06.254119       1 reststorage.go:147] unable to fetch pod metrics for pod default/bbox: no metrics known for pod
E0821 01:25:06.254143       1 reststorage.go:147] unable to fetch pod metrics for pod default/nginx: no metrics known for pod
E0821 01:25:43.654288       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:vm2: unable to fetch metrics from Kubelet vm2 (vm2): Get https://vm2:10250/stats/summary/: x509: certificate signed by unknown authority, unable to fully scrape metrics from source kubelet_summary:vm1: unable to fetch metrics from Kubelet vm1 (vm1): Get https://vm1:10250/stats/summary/: x509: certificate signed by unknown authority]
```

解决: 在`metrics-server-deployment.yaml`文件中的容器参数中添加`args: [ "--kubelet-insecure-tls" ]`

> - [x] [权限问题](https://www.cnblogs.com/vincenshen/p/9638162.html)

```log
E1218 10:26:36.104471       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:master: unable to fetch metrics from Kubelet master (10.168.67.6): request failed - "403 Forbidden", response: "Forbidden (user=system:serviceaccount:kube-system:metrics-server, verb=get, resource=nodes, subresource=stats)"]
```

原因

- clusterrole`system:metrics-server`对`nodes/stats`资源没有相关权限

解决: 在资源中添加对`nodes/stats`资源的 get 权限

```bash
kubectl get clusterrole system:metrics-server -o yaml > metrics-server-clusterrole.yaml
kubectl apply -f metrics-server-clusterrole.yaml
```

## Kubelet

> - [x] [已被删除的容器还在磁盘上](https://github.com/kubernetes/kubernetes/issues/60987#issuecomment-529107444)

```log
# /var/log/kubernetes/kubelet.log
E0214 17:00:33.187665    1547 kubelet_volumes.go:154] Orphaned pod "8d73e524-fb7c-4b25-a907-3c18b78f587d" found, but volume paths are still present on disk : There were a total of 1 errors similar to this. Turn up verbosity to see them.
E0214 17:00:35.188258    1547 kubelet_volumes.go:154] Orphaned pod "8d73e524-fb7c-4b25-a907-3c18b78f587d" found, but volume paths are still present on disk : There were a total of 1 errors similar to this. Turn up verbosity to see them.
```

解决: 找到出现问题的Pod的id后,删除该目录即可

```bash
journalctl -fu kubelet | awk -F'"' '{ print $2}'
cd /var/lib/kubelet/pods
rm -rf <pod id#>
```

## helm

> - [x] [helm 连接 tiller 报错](https://www.orchome.com/1977)

环境

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
EOF
helm init --service-account tiller
helm version
```

报错

```log
Client: &version.Version{SemVer:"v2.12.0", GitCommit:"d325d2a9c179b33af1a024cdb5a4472b6288016a", GitTreeState:"clean"}
E0311 14:30:57.739137    2268 portforward.go:331] an error occurred forwarding 37108 -> 44134: error forwarding port 44134 to pod 656c9a0a94968a3ef99a17bba846c4617d8afd1b6f9a375890a86b965cd9fa5f, uid : unable to do port forwarding: socat not found
Error: cannot connect to Tiller
```

解决: 所有设备上安装`socat`包

## 其它问题

> - [x] 虚拟机关机重启后,不同节点间的 Pod 不能互 ping

环境: 检查发现 node 节点的 flannel.1 网卡 down,与该[博客](https://www.oschina.net/question/2344660_2286913)描述基本相符

解决: 删除 cni0,flannel.1 网桥后重新启动服务

```bash
systemctl stop kubelet
systemctl stop docker
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
systemctl start kubelet
```

> - [x] kubernetes无法删除Terminating状态的namespace

解决: 删除对应namespace json文件的`spec`字段后,重新发起请求

```bash
kubectl get namespace <namespace-name> -o json > tmp.json
# 编辑 tmp.json,删除 "spec" 字段部分
curl -k -XPUT -H "Content-Type: application/json" --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/istio-system/finalize
```
