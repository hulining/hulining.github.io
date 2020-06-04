---
title: 自定义kube-prometheus添加etcd监控
date: 2020/04/03
tags:
  - kubernetes
  - prometheus
  - etcd
categories:
  - 云原生应用
  - prometheus
abbrlink: 43867
description: kube-prometheus 默认不包含对 etcd 的监控, 本文章使用 kube-prometheus 为 etcd 添加监控
---

# 安装依赖
```bash
yum install -y go
go get github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb # jsonnet 的包管理器
go get github.com/brancz/gojsontoyaml # go 语言编写的 json 转化为 yaml 工具
# 安装好的二进制包在 ~/go/bin/ 目录下
cd ~/go/bin/
cp jb gojsontoyaml /usr/local/bin/
```

# 自定义 jsonnet
```bash
cd my-kube-prometheus # 在 自定义 kube-prometheus
jb init  # 创建初始化空文件 jsonnetfile.json
# 安装 kube-prometheus 依赖.花费时间较长
# 该版本较老,推荐使用 github.com/coreos/kube-prometheus/jsonnet/kube-prometheus@master
jb install github.com/coreos/kube-prometheus/jsonnet/kube-prometheus@release-0.1 
jb update
```
# 编译
使用 `build.sh` 进行编译,以 `etcd.jsonnet` 为例
```bash
#!/bin/bash

set -e
set -x
set -o pipefail

rm -rf manifests
mkdir -p manifests/setup

jsonnet -J vendor -m manifests "${1-etcd.jsonnet}" | xargs -I{} sh -c 'cat {} | gojsontoyaml > {}.yaml; rm -f {}' -- {}
```

```jsonnet
// https://github.com/coreos/kube-prometheus/blob/master/examples/etcd.jsonnet
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') +
           (import 'kube-prometheus/kube-prometheus-static-etcd.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',
    etcd+:: {
      // 配置 etcd 集群的 IP 地址,使用 逗号 隔开
      ips: ['10.168.67.6', '10.168.67.7', '10.168.67.8'],
      clientCA: importstr '/etc/kubernetes/pki/etcd/ca.crt',
      clientCert: importstr '/etc/kubernetes/pki/etcd/healthcheck-client.crt',
      clientKey: importstr '/etc/kubernetes/pki/etcd/healthcheck-client.key',
      // 指定 serverName 或 insecureSkipVerify 其中一个.
      // 若指定 serverName,你应该指定 openssl.cnf 中填写的 etcd 的包含证书的 DNS名称,否则会报 x509 错误.且 serverName 可以被 nslookup 解析
      // 若指定 insecureSkipVerify,会跳过认证
      serverName: 'etcd.kube-system.svc.cluster.local',
      //insecureSkipVerify: true,
    },
  },
};

{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }
```

# 应用 
重新应用编译后产生的 `manifests`
```bash
kubectl apply -f manifests/setup
kubectl apply -f manifests/
```

如果想要在已有 `kube-prometheus` 的基础上作出修改,只需要执行如下命令
```bash
jb init
jb update
build.sh
kubectl apply -f manifests/setup
kubectl apply -f manifests/
```