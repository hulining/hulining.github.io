---
title: Linux 防火墙及 iptables 详解
date: 2020/06/11
tags:
  - Linux
  - iptables
  - 面试
categories:
  - Linux
abbrlink: 
description: 详细介绍 Linux 防火墙及 iptables 命令的使用,包含一些常用的 iptables 规则.
---

## Linux 防火墙

Linux 防火墙基于内核实现,它使用 Netfilter 包过滤引擎,根据用户通过 `iptables` 命令定义的规则实现对数据包的过滤或转发.

### 四表五链

Linux 防火墙内置了 4 个表分别提供不同的功能.表及其功能如下:

表 | 提供功能 | 内置规则链
:---: | :---: | :---:
`filter` |  包过滤 | INPUT,FORWARD,OUTPUT
`nat` | 网络地址转换 | PREROUTING,POSTROUTING,OUTPUT
`mangle` | 包重构 | PREROUTING,POSTROUTING,INPUT,OUTPUT,FORWARD
`raw` | 数据跟踪处理,提供连接跟踪机制 | OUTPUT,PREROUTING

规则表之间的优先顺序为 `raw -> mangle -> nat -> filter`.

![四表五链](/images/firewalld-tables-and-chains.png)

Linux 防火墙还内置了 5 条规则链,用于表示数据包传输的路径,每一条链其实就是用户定义的规则清单,其中包含一条或多条规则.规则链及其生效时间/功能如下:

链 | 生效时间 | 功能
:---: | :---: | :---:
`PREROUTING` | 数据包刚进入网络接口,路由之前 | 对刚进如网络接口数据包进行路由选择,是流入本机还是由本机转发
`INPUT` | 数据包从内核流入用户空间 | 对流入用户空间的包应用规则,是拒绝,丢弃还是接受
`FORWARD` | 数据包在内核空间中,从一个网络接口进入,另一个网络接口出去 | 对由本机转发的数据包应用规则
`OUTPUT` | 数据包从用户空间流出到内核空间 | 对从用户空间流出到内核空间的数据包应用规则
`POSTROUTING` | 数据包离开网络接口前 | 对离开本机前的数据包进行路由选择

### 数据包的传输过程

1. 当一个数据包流入本机网卡时,它首先进入 `PREROUTING` 链,内核根据数据包目的 IP 判断是否需要转发出去
2. 如果数据包就是进入本机的,它将会到达 `INPUT` 链.`INPUT` 链规则接受该数据包后,相关进程就会收到它,进行处理后发送响应数据包,响应数据包会经过 `OUTPUT` 链,然后到 `POSTROUTING` 链后流出本机.
3. 如果数据包是要转发出去的,且内核允许转发(通过 `echo 1 > /proc/sys/net/ipv4/ip_forward` 开启).数据包就会经过 `FORWARD` 链,到达 `POSTROUTING` 链后流出本机

![数据包流向](/images/packet-flow.svg)

![数据包流向](/images/packet-flow.png)

## 使用 iptables 管理防火墙规则

iptables 仅作为 Linux 防火墙管理工具,让人们更加便捷,直观的管理 Linux 主机防火墙.

iptables 设置规则的基本语法格式为

```text
iptables [-t 表名] 命令选项 [链名] [条件匹配] [-j 目标动作或跳转]

Usage: iptables -[ACD] chain rule-specification [options]
       iptables -I chain [rulenum] rule-specification [options]
       iptables -R chain rulenum rule-specification [options]
       iptables -D chain rulenum [options]
       iptables -[LS] [chain [rulenum]] [options]
       iptables -[FZ] [chain] [options]
       iptables -[NX] chain
       iptables -E old-chain-name new-chain-name
       iptables -P chain target [options]
       iptables -h (print this help information)

```

```text
iptables --help


# 常用选项
-n, --number: 显示防火墙规则链的序号
-t <TABLE>: 指定表,默认表为 filter
-L, --list: 列出表下的防火墙规则链
-F, --flush: 清空防火墙规则链中所有条目


# 命令选项
-P <chain> <target>: 指定默认的
-N <chain>: 创建新规则链
-E <old_chain> <new_chain>: 重命名规则链

-A <chain> <rule>: 向规则链中追加规则
-A <chain> <rule_num>: 删除规则链中指定规则
-I <chain> [rule_num] <rule>: 向规则链中插入规则,默认在最开始插入规则
-D <chain> [rule_num]: 删除指定规则,默认删除 num=1 的规则
-R <chain> <rule_num> <rule>: 将规则链中指定规则

-L [chain [rule_num]]: 列出指定链的防火墙规则.默认为表中所有
-S [chain [rule_num]]: 列出指定链的防火墙规则.默认为表中所有


# 条件匹配
--- 基本匹配 ---
[!] -s, --src, --source: 检查源 IP
[!] -d, --dst, --destination: 检查目标 IP
[!] -p, --protocol {tcp|udp|icmp}: 检查报文协议
-i, --in-interface IFACE: 数据报文的流入接口.仅能用于 PREROUTING,INPUT 及 FORWARD 链上
-o, --out-interface IFACE: 数据报文的流出接口.仅能用于 FORWARD,OUTPUT 及 POSTROUTING 链上

--- 隐式扩展匹配 ---
-p {tcp|udp}
    --dport PORT[-PORT]: 检查目标端口
    --sport PORT[-PORT]: 检查源端口
-p icmp
    --icmp-type: icmp 类型,0-8

--- 显示扩展匹配,需要显示使用 -m 显示指定 ---
-m multiport: 多端口扩展
    [!] --source-ports,--sports port[,port|,port:port]...：指明多个源端口
    [!] --destination-ports,--dports port[,port|,port:port]...：指明多个离散的目标端口
    [!] --ports port[,port|,port:port]...

-m iprange: 指定 ip 地址范围
    [!] --src-range from[-to]：指明连续的源 IP 地址范围
    [!] --dst-range from[-to]：指明连续的目标 IP 地址范围

-m string: 指定报文内容出现的字符串
    [!] --string pattern: 指定字符正则表达式

-m time扩展: 指定时间范围
    --datestart
    --datestop
    --timestart
    --timestop
    --monthdays
    --weekdays

-m connlimit: 指定并发连接数
    --connlimit-above n：连接的数量大于 n
    --connlimit-upto n: 连接的数量小于等于 n

-m limit: 报文收发速率
    --limit rate[/second|/minute|/hour|/day]: 限制速率
    --limit-burst number: 限制并发量

-m state: 根据连接追踪机制检查连接的状态
    --state {NEW,ESTABLISHED,RELATED,INVALIED}: 指定连接状态,一个或多个

# 目标或动作
-j TARGET: 跳转至指定的 TARGET
   ACCEPT: 接受
   DROP: 丢弃,不回应任何信息
   REJECT: 拒绝,回应拒绝信息
   RETURN: 返回调用链
   REDIRECT: 端口重定向
   LOG: 在 /var/log/messages 文件中记录日志信息,然后将数据包传递给下一条规则
   MARK: 做防火墙标记
   DNAT --to: 目标地址转换
   SNAT: 源地址转换
   MASQUERADE: 地址伪装
   ...
   自定义链：由自定义链上的规则进行匹配检查
```

## 常用 iptables 规则链

- 允许 ssh 连接,仅开放 22 端口

```bash
iptables -A INPUT -i eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT

# 仅对指定网段开放 22 端口
iptables -A INPUT -i eth0 -p tcp -s 192.168.200.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT

# 一次性开放多端口 22,80,443
iptables -A INPUT -i eth0 -p tcp -m multiport --dports 22,80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp -m multiport --sports 22,80,443 -m state --state ESTABLISHED -j ACCEPT
```

- 允许 ping

```bash
# 从外部向内部 ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT

# 从内部向外部 ping
iptables -A OUTPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT
```

- 防止 DDOS 攻击

```bash
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
```

- 设置端口转发

```bash
iptables -t nat -A PREROUTING -p tcp -d 192.168.102.37 --dport 422 -j DNAT --to 192.168.102.37:22
iptables -A INPUT -i eth0 -p tcp --dport 422 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 422 -m state --state ESTABLISHED -j ACCEPT
```

- 记录丢弃的包

```bash
iptables -N LOGGING
iptables -A LOGGING -m limit --limit 2/min -j LOG --log-prefix "IPTables Packet Dropped: " --log-level 7
iptables -A LOGGING -j DROP
iptables -A INPUT -j LOGGING
```

## iptables 性能优化技巧

合理的配置 iptables 规则可以保护我们的内部主机.但是当 iptables 规则设置不合理时,可能会造成服务不可访问或服务性能下降.因此在使用 iptables 配置防火墙规则时,应注意如下几点:

- 安全放行所有 ESTABLISHED 状态连接,建议放在第一条,效率更高

```bash
iptables -I INPUT -m state --state ESTABLISHED -j ACCEPT
iptables -I OUTPUT -m state --state ESTABLISHED -j ACCEPT
```

- 尽可能将可由一条规则能够描述的多个规则合并为一条规则,如添加多个端口/IP 访问

```bash
iptables -A INPUT -p tcp -m multiport --dports 22,80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp -m multiport --sports 22,80,443 -m state --state ESTABLISHED -j ACCEPT

iptables -A INPUT -m iprange --src-range 192.168.0.10-192.168.0.20 -j ACCEPT
iptables -A OUTPUT -m iprange --src-range 192.168.0.10-192.168.0.20 -j ACCEPT
```

- 有特殊目的限制访问功能要在放行规则之前加以拒绝

```bash
# 对服务器限制发送带有 *iqiyi.com 的报文
iptables -I OUTPUT 2 -m string --string "*iqiyi.com" --algo kmp -j REJECT
```

- 谨慎放行入站的新请求
- 在规则最后添加默认策略

```bash
iptables -A INPUT -P DROP
```
