---
title: Keepalived
date: 2020/06/04
tags:
  - Linux
  - Keepalived
categories:
  - Linux
abbrlink: 
description: 本文章主要介绍 keepalived 的原理及基本使用方式.以作备忘.
---

## 原理

keepalived 主要包含如下三个模块:

- core: keepalived 的核心莫模块,负责主进程的启动和维护,全局配置文件的加载解析等
- check: 负责对 real server 的健康检查,包括了各种健康检查方式,以及对应的配置的解析
- vrrp: 维护了 VIP 与 real server 的映射关系,根据优先级来选择主备节点.它基于 vrrp(虚拟路由冗余)协议,实现了当主节点挂掉时,使得 VIP 和对应的 MAC 地址漂移到备机上,通知其它主机刷新 arp 列表的功能.

## 配置文件

keepalived 常用配置文件示例如下:

```conf
# /etc/keepalived/keepalived.conf
global_defs {
    # 尽量不使用 root 运行脚本
    enable_script_security
}

# 定义定期执行脚本的相关属性
vrrp_script haproxy-check {
    user root
    script "/bin/bash /etc/keepalived/check_haproxy.sh"
    interval 3
    weight -2
    fall 10
    rise 2
}

# 定义 vrrp_instance 示例
vrrp_instance haproxy-vip {
    # 主机状态 MASTER 或 BACKUP
    state MASTER
    # 定义节点优先级,MASTER 比其他节点多 50
    priority 100
    interface ens33
    # 0-255 的任意数字
    virtual_router_id 1
    # 指定发送 VRRP 通告的间隔,单位是秒
    advert_int 3
    # 使用单播将 vrrp 报文发送到以下地址列表.可以理解为 real server 地址列表
    unicast_peer {
        192.168.1.3
        192.168.1.4
        192.168.1.5
    }
    # 虚拟 IP 地址
    virtual_ipaddress {
        192.168.1.100
    }
    # 添加追踪脚本,用于监测 real server 的脚本
    track_script {
        haproxy-check
    }
}
```

如下是配置文件中用到的 `/etc/keepalived/check_haproxy.sh` 脚本文件,用于监测 VIP 的指定端口是否存活

```bash
#!/bin/bash
VIRTUAL_IP=192.168.1.100

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

if ip addr | grep -q $VIRTUAL_IP ; then
    curl -s --max-time 2 --insecure https://${VIRTUAL_IP}:8443/ -o /dev/null || errorExit "Error GET https://${VIRTUAL_IP}:8443/"
fi
```
