---
title: 运维面试题之 TCP/IP
date: 2020/06/04
tags:
  - 面试题
  - TCP/IP
categories:
  - TCP/IP
abbrlink: 
description: 总结整理常见 TCP/IP 面试题,以作备忘
---

## OSI 七层模型,TCP/IP 四层模型

OSI从上到下依次是 应用层 -> 表示层 -> 会话层 -> 传输层 -> 网络层 -> 数据链路层 -> 物理层

TCP/IP 从上到下依次是 应用层 -> 传输层 -> IP层 -> 网络接口层

## TCP/IP 首部

![IP 首部](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/interview-questions-tcp-ip/IP-header.png)

![TCP 首部](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/interview-questions-tcp-ip/TCP-header.png)

- 16 位源端口号,16位目的端口号: 用于寻找发送端和接收端的应用进程,加上IP首部的源端 IP 及目的 IP,唯一确认一个 TCP 连接
- 32 位序号: 标识发送的数据字节流,标识在这个报文段中的第一个数据字节,2^32 - 1 后重新从 0 开始.包含该主机选择的连接的ISN(Initial Sequence Number),要发送的第一个数据字节序号为 ISN + 1.
- 32 位确认序号: ACK为 1 时有效,确认号应该是上次成功收到的数据字节序号 + 1.
- 4 位首部长度: 首部中 32bit 的数目,一般为 5,也就是 20 字节
- URG: 紧急指针
- ACK: 确认序号有效
- PSH: 接收方应尽快将此报文段交给应用层
- RST: 重建连接
- SYN: 同步序号,用来发起一个新连接
- FIN: 发端完成发送任务
- 16 位窗口大小: TCP流量控制,字节数，起始于确认序列号指明的值,接收端期望收到的字节,最大为65535.
- 16 位检验和：包括计算 TCP 首部和数据综合的二进制反码和检验和.用于接收端校验
- 16 位紧急指针：URG为 1 时有效,是一个正向的偏移量,和序号字段值相加表示最后一个字节的序号

## TCP 三次握手,四次挥手过程

### TCP 三次握手

![TCP 建立连接](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/interview-questions-tcp-ip/tcp-establish-connection.jpg)

1. 客户端发送连接请求报文(`SYN=1,seq=x`)到服务器,并进入 `SYN_SEND` 状态,等待服务端确认
2. 服务端收到后,回传一个确认报文(`SYN=1,seq=y,ACK=1,ack=x+1`)以示传达确认信息,此时服务器进入 `SYN_RECV` 状态
3. 客户端接收服务端的确认包,向服务器发送确认包(`ACK=1,seq=x+1,ack=y+1`).随后,客户端与服务端建立连接,进入 `ESTABLISHED` 状态,完成三次握手

握手过程中状态码如下

```text
SYN = 1,ACK = 0,seq = x;
SYN = 1,ACK = 1,seq = y,ack = x+1;
SYN = 0,ACK = 1,seq = x+1,ack=y+1
```

### TCP 四次挥手

![TCP 断开连接](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/interview-questions-tcp-ip/tcp-release-connection.jpg)

1. 客户端发送连接释放报文(`FIN=1,seq=u`),并停止发送数据,客户端进入 `FIN_WAIT_1` 状态
2. 服务器收到连接释放报文,发出确认报文(`ACK=1,seq=v,ack=u+1`),服务器进入 `CLOSE_WAIT` 状态.客户端接收到服务器的确认请求后,客户端进入`FIN_WAIT_2` 状态,等待服务器发送连接释放报文.此时客户端不再向服务端发送数据,若服务端发送数据,客户端依然接受.
3. 服务端将最后的数据发送完毕后,向客户端发送释放报文(`FIN=1,ACK=1,seq=w,ack =u+1`),服务端进入 `LAST_ACK` 状态.
4. 客户端收到服务端连接释放报文后,必须回复确认(`ACK=1,seq=u+1,ack=w+1`),客户端进入 `TIME_WAIT` 状态,时长为 2∗MSL(报文最长寿命).

挥手过程中状态码如下

```text
1. FIN = 1,seq = u;
2. ACK = 1,seq = v,ack = u+1;
3. FIN = 1,ACK = 1,seq = w,ack =u+1;
4. ACK = 1,seq = u+1,ack = w+1
```

### 为什么最后要等待 2 * MSL

- 保证客户端发送的最后一个 ACK 报文能够到达服务器,保证可靠的终止 TCP 连接.TCP 内部存在超时重传机制,如果服务端在发出释放报文后,没有收到确认报文,服务端会重新发送一次,客户端就能够在 2 * MSL 内收到重传报文
- 保证新连接中不会出现旧连接的请求报文.客户端发送完最后一个确认报文后,在这个 2 * MSL 时间中,就可以使本连接持续的时间内所产生的所有报文段都从网络中消失.

### 为什么需要三次握手,两次不行吗

主要是为了防止已失效的请求报文段突然又传送到了服务端而产生连接的误判,造成服务端资源/时间的浪费.

### 为什么建立连接是三次握手,而关闭连接却是四次挥手呢

- 建立连接

因为服务端在 LISTEN 状态下,收到建立连接请求的SYN报文后,把ACK和SYN放在一个报文里发送给客户端.

- 关闭连接

当收到对方的 FIN 报文时,仅表示对方不再发送数据但还能接收收据,我们也未必把全部数据都发给了对方,所以我们可以立即关闭,也可以发送一些数据给对方后,再发送 FIN 报文给对方表示同意关闭连接.因此关闭连接的 ACK 和 FIN 一般会分开发送.

### 三次握手有什么缺陷

伪造IP大量的向 server 发送 TCP 连接请求报文包,从而将 server 的半连接队列(即 server 收到连接请求 SYN 之后将 client 加入半连接队列中)占满,从而使得 server 拒绝其他正常的连接请求,即拒绝服务攻击.

### TCP 以什么方式提供数据可靠性

- 将数据切分为最合适发送的数据块
- 计时,收到后确认与超时重传.当 TCP 发出一个段后,它会启动一个定时器,等待目的端确认收到这个报文段.如果不能及时收到一个确认,将重发这个报文
- 首部与数据校验和确保数据在传输过程中的任何变化
- 重新排序机制确保收到的数据以正确的顺序交给应用层
- 丢弃重复数据
- 流量控制,防止较快主机致使较慢主机的缓冲区溢出

### TCP 与 UDP 有什么区别

- TCP 是面向连接的,通信之前需要进行三次握手,建立连接,通信结束后需要四次挥手,断开连接; UDP是无连接的,即发送数据之前不需要建立连接
- TCP 提供可靠的服务,以收到确认,超时重传,重新排序等机制提供数据可靠性; UDP 不保证数据的可靠性
- TCP 面向字节流,实际上是 TCP 把数据看成一连串无结构的字节流; UDP 是面向报文的
- 每一条 TCP 连接只能是点到点的; UDP 支持一对一,一对多,多对一和多对多的交互通信
- TCP 的逻辑通信信道是全双工的可靠信道; UDP 则是不可靠信道
- TCP 相对于 UDP 来说要求系统资源较多

## DNS 域名解析过程

1. 应用首先检查缓存中有没有这个域名对应的解析过的 IP 地址.如果缓存中有,则返回缓存中的 IP 地址.否则进行下一步
2. 查找本机的 host 文件, Linux 为 `/etc/hosts`,Windows 为 `C:\Windows\System32\drivers\etc\hosts`.若查到,则返回.否则进行下一步
3. 发送请求到 DNS 服务器 Linux 为 `/etc/resolv.conf`.DNS 服务器在收到客户机的请求后,首先检查 DNS 服务器的缓存,若查到请求的地址或名字,即向客户机发出应答信息.否则在数据库中查找,若查到请求的地址或名字,即向客户机发出应答信息
4. 若没有找到,则将请求发给根域 DNS 服务器,并依序从根域查找顶级域,由顶级域查找二级域...直至找到要解析的地址或名字,即向客户机所在网络的 DNS 服务器发出应答信息,DNS 服务器收到应答后现在缓存中存储,然后,将解析结果发给客户机
