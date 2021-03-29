---
title: Zookeeper 基础
date: 2020-03-26 10:51:44
tags:
  - Zookeeper
categories:
  - Zookeeper
abbrlink: 
description: 对 Zookeeper 的简单介绍,作为备忘
---

## 安装及配置

```bash
cp $ZK_HOME/conf/{zoo_sample.cfg,zoo.cfg}

# zookeeper配置文件简单解释
tickTime=2000 # 心跳基本时间单位,毫秒级
initLimit=10 # 初始化的超时时间,initLimit*tickTime
syncLimit=5 # Leader 与 Follower的超时时间,syncLimit*tickTime
clientPort=2181 # 端口
dataDir=${ZK_HOME}/data # 数据目录
ataLogDir=${ZK_HOME}/logs # 日志目录
# 集群配置
server.1=zk_server_1:2888:3888
server.2=zk_server_2:2888:3888
server.3=zk_server_3:2888:3888

# 这里需要注意的是,在每一个zookeeper server节点的dataDir目录下有一个 myid 的文件,对应 server.1
echo 1 > ${dataDir}/myid
```

## 客户端操作

```bash
connect host:port   # 连接到指定服务
get path [watch]    # 获取指定节点信息(包括内容和状态), watch 表示是否对该参数进行监听
ls path [watch]     # 列出节点下的子节点
ls2 path [watch]    # 列出节点下的子节点,及该节点的信
set path data [version] # 设置节点数据
rmr path            # 删除节点及其子节点数据
create [-s] [-e] path data acl  # 创建节点. -s 表示顺序节点,-e 表示临时节点
stat path [watch]   # 获取节点的元数据信息
listquota path      # 显示节点配额
setAcl path acl     # 设置 ACL 访问控制
getAcl path         # 获取 ACL 访问控制
delete path [version] # 删除节点
```

## znode 节点类型

每个 znode 都有不同的生命周期,而生命周期长短取决于 znode 的节点类型.Zoookeeper 提供了如下4种节点类型:

节点类型 | 特性 | 创建方式
:--- | :--- | :---
持久节点 | 会话关闭后,该节点仍然存在,可以创建子节点 | `create <path> <data>`
持久顺序节点 | 在持久节点特性的基础上.节点名后缀为自增数字 | `create -s <path> <data>`
临时节点 | 会话关闭后,该节点不存在,不可以创建子节点 | `create -e <path> <data>`
临时顺序节点 | 在临时节点特性的基础上,节点名后缀为自增数字 | `create -s -e <path> <data>`

## ACL 访问控制列表

ZooKeeper 提供了一套完善的 ACL 权限控制机制保障数据安全性.其实 ACL 规则由`Scheme:id:permission`格式组成,其中,对于身份认证,提供了以下几种方式:

身份认证方式 | 解释
:--- | :---
world | 默认方式,所有用户都可无条件访问,组合形式为: `world:anyone:[permissions]`
digest | 用户名:密码认证方式,最常用.组合形式为: `digest:username:BASE64(SHA1(password)):[permissions]`
ip | 对指定ip进行限制,组合形式为: `ip:127.0.0.1:[permissions]`
auth | 认证登录形式,需要用户获取权限后才可访问,组合形式为: `auth:user:password:[permissions]`

对于 znode 的 `permissions` 权限,提供了以下 5 种操作权限:

权限 | 简写 | 解释
:--- | :--- | :---
CREATE| C | 允许授权对象在当前节点下创建子节点
DELETE | D |允许授权对象在当前节点下删除子节点
WRITE | W | 允许授权对象在当前节点进行更新操作
READ | R |允许授权对象在当前节点获取节点内容或获取子节点列表
ADMIN | A | 允许授权对象对当前节点进行 ACL 相关的设置操作

## zookeeper 四字命令

```bash
echo conf | nc 192.168.0.68 2181 # 查看状态信息
echo cons | nc 192.168.0.68 2181 # 查看客户端连接信息
echo envi | nc 192.168.0.68 2181 # 查看环境变量信息
echo mntr | nc 192.168.0.68 2181 # 查看 zk 的健康信息
echo ruok | nc 192.168.0.68 2181 # 查看 zk 是否启动
echo stat | nc 192.168.0.68 2181 # 查看状态信息
echo wchs | nc 192.168.0.68 2181 # 查看 watch 信息
```
