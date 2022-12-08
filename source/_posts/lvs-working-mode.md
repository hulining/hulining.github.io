---
title: LVS 三种工作模式与负载均衡算法
date: 2020/06/04
tags:
  - Linux
  - LVS
  - 面试
categories:
  - Linux
abbrlink: 
description: 本文章主要介绍 LVS(Linux Virtual Server)的三种工作模式 - NAT,IP 隧道,直接路由(Direct Routing),并简述了三中工作模式的优缺点.后面介绍了 LVS 支持的负载均衡算法
---

## LVS 简介

LVS(Linux Virtual Server) 是由章文嵩博士主导的开源负载均衡项目,在 2.6 的及以后内核版本中已将 LVS 集成于内核模块中.该项目在 Linux 内核中实现了基于 IP 的数据请求负载均衡调度方案,其应用场景如图 1 所示:

![LVS 应用场景](/images/lvs-use-cases.jpg)

互联网用户从外部访问公司的外部负载均衡服务器,终端用户的 Web 请求会发送给 LVS 调度器,调度器根据自己预设的算法决定将该请求发送给后端的某台 Web 服务器,比如,轮询算法可以将外部的请求平均分发给后端的所有服务器,终端用户访问 LVS 调度器虽然会被转发到后端真实的服务器,但如果真实服务器连接的是相同的存储,提供的服务也是相同的服务,最终用户不管是访问哪台真实服务器,得到的服务内容都是一样的,整个集群对用户而言都是透明的.根据 LVS 工作模式的不同,真实服务器会选择不同的方式将用户需要的数据发送到终端用户.LVS 工作模式分为 NAT 模式,IP 隧道(TUN)模式,以及直接路由(DR)模式.

Linux 操作系统提供了 `ipvsadm` 管理工具来管理 LVS 规则.其常用选项及参数有:

```text
-A, --add-service     添加一个新的虚拟服务
-E, --edit-service    编辑虚拟服务
-D, --delete-service  删除虚拟服务
-C, --clear           清除所有的虚拟服务规则
-R, --restore         恢复虚拟服务规则
-a, --add-server      在虚拟服务中添加一个真实服务器
-r, --real-server     指定 Real Server
-e, --edit-server     编辑某个真实服务器
-d, --delete-server   删除某个真实服务器
-L, -l, --list        显示内核中的虚拟服务规则
-n, --numeric         以数字形式显示IP端口
-c, --connection      显示 ipvs 中目前存在的连接,也可以用于分析调度情况
-Z, --zero            将转发消息的统计清零
-p, --persistent      配置持久化时间
--set tcp tcpfin udp  配置超时时间(tcp/tcpfin/udp)
-t | -u               TCP/UDP 协议的虚拟服务
-g | -m | -i          设置 LVS 模式为: DR | NAT | TUN
-w        配置真实服务器的权重
-s        配置负载均衡算法，如:rr, wrr, lc 等
--timeout 显示配置的tcp/tcpfin/udp超时时间
--stats   显示历史转发消息统计（累加值）
--rate    显示转发速率信息(瞬时值)
```

## LVS 三种工作模式

首先了解以下缩写:

- DS：Director Server 前端负载均衡器节点
- RS：Real Server 后端真实的工作服务器
- VIP：DS 上的虚拟 IP 地址,直接面向用户请求
- DIP：DS 内网 IP 地址,主要用于和内部主机通讯
- RIP：Real Server IP,后端服务器的 IP 地址
- CIP：Client IP,访问客户端的IP地址

### NAT 模式

NAT(Network Address Translation)即网络地址转换,包括 SNAT(源地址转换) 与 DNAT(目标地址转换).它通过修改请求报文的目标 IP 地址(同时可能会修改目标端口)挑选出某台 Real Server 的 RIP 地址实现转发.在请求与响应过程中期间,无论是进来的流量,还是出去的流量,都必须经过 LVS 负载均衡器.

1. 客户端将请求发往负载均衡器,请求报文源地址是 CIP,目标地址为 VIP
2. 负载均衡器通过地址转换,将客户端发来的数据包转发至后端 RIP
3. RS 处理完成后响应对负载均衡器返回响应数据
4. 负载均衡器通过地址转换,将 RS 的响应数据响应给客户端

![LVS NAT](/images/lvs-NAT.jpg)

> 注意: RIP 与 DIP 需要在同一网络,且 RIP 网关必须指向 DIP

- 优点: 只需要暴露出一个 VIP 地址即可,对用户来说后端服务器是完全透明的
- 缺点: 当 RS 节点增长过多时,负载均衡器将成为性能瓶颈,速度会变慢

### IP 隧道(TUN)模式

负载均衡器把客户端发来的数据包,封装一个新的 IP 头标记(仅目的IP)发给 RS.RS 收到后,先把数据包的头解开,还原数据包,处理后,直接返回给客户端,不需要再经过负载均衡器

1. 客户端将请求发往负载均衡器,请求报文源地址是 CIP,目标地址为 VIP
2. 负载均衡器将在客户端请求报文的首部再封装一层 IP 报文,将源地址改为DIP,目标地址改为RIP,并通过 IP 隧道技术将报文发给 RS
3. RS 收到后,先把数据包头解开,还原数据包,处理后,直接返回给客户端,不需要再经过负载均衡器

![LVS TUN](/images/lvs-TUN.jpg)

- 优点: 减少负载均衡器压力,负载均衡器不再是系统的瓶颈,能够处理更多的请求流量
- 缺点: RS 节点需要合法 IP,且需要所有服务器支持 `IP Tunneling`(IP Encapsulation)协议,因此服务器可能只局限于部分 Linux 系统上

### 直接路由(DR)模式

负载均衡器和 RS 都使用同一个 IP 对外服务,但只有负载均衡器对 ARP 请求进行响应.负载均衡器收到数据包后根据调度算法,找出对应的RS,把目的 MAC 地址改为 RS 的 MAC(IP一致),并将请求分发给这台RS.RS 收到数据包并处理完成之后,由于 IP 一致,可以直接将数据返给客户端,与直接从客户端收到这个数据包无异,处理后直接返回给客户端

1. 客户端将请求发往前端的负载均衡器,请求报文源地址是 CIP,目标地址为 VIP
2. 负载均衡器将客户端请求报文的源 MAC 地址改为自己的 MAC 地址,目标 MAC 改为了 RS 的 MAC 地址,并将此包发送给 RS
3. 处理完请求报文后,由于 RS 与 负载均衡器有具有同一 VIP,会将响应报文直接发送给客户端

![LVS DR](/images/lvs-DR.jpg)

- 优点: 与隧道模式一样,负载均衡器也只是分发请求,应答包通过单独的路由方法返回给客户端.同时,不需要隧道结构,可以使用大多数服务器作为 RS
- 缺点：要求 VIP 必须与物理网卡在一个物理段上,否则 ARP 不能寻到不同网段的 MAC 地址.也就是说所有 RS 节点和调度器 LB 只能在一个局域网里面

## LVS 负载均衡算法

- RR(Round Robin Scheduling): 轮询调度算法,将请求依次分配给不同的 RS 节点,RS 节点均摊请求.适用于 RS 节点性能差不多的情况
- WRR(Weighted Round-Robin Scheduling): 加权轮询调度算法,按照 RS 的权重将请求分发给不同的 RS 节点, RS 节点按照权重比分摊请求.
- WLC(Weighted Least-Connection Scheduling): 加权最小连接数调度算法, 假设 RS 权重为 Wi,当前连接数为 Ti,则选择 Ti/Wi 最小的 RS 节点作为下一个调度节点
- DH(Destination Hashing Scheduling): 目标地址哈希调度算法,以目标地址为关键字做 hash 获取 RS 调度节点
- SH(Source Hashing Scheduling): 源地址哈希调度算法,以源地址为关键字做 hash 来获取 RS 调度节点
- LC(Least-Connection Scheduling): 最小连接数调度算法,IPVS 表存储了所有活动的连接,负载均衡器会进行比较,将连接请求发送到当前连接最少的 RS
- LBLC(Locality-Based Least Connections Scheduling): 基于地址的最小连接数调度算法,将来自同一个目的地址的请求分配给同一台 RS.如果 RS 满负荷,则将这个请求分配给连接数最小的 RS,并以它作为下一次分配的首先考虑
- LBLCLR(Locality-Based Least Connectio ns with Replication Scheduling):带复制的基于局部最少链接,先根据请求的目标 IP 地址找出该目标 IP 地址对应的服务器组,按最小连接原则从该服务器组中选出一台服务器,若服务器没有超载,将请求发送到该服务器,否则按最小连接原则从整个集群中选出一台服务器,将该服务器加入到服务器组中,将请求发送到该服务器.同时,当该服务器组有一段时间没有被修改,将最忙的服务器从服务器组中删除,以降低复制的程度.
