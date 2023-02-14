---
title: Nginx 易错记录
date: 2022-12-09 10:23:33
tags:
  - Nginx
categories:
  - Nginx
abbrlink: 
description: 总结整理 Nginx 使用过程中的容易出错&踩坑记录

---

## upstream keepalive 配置后不生效问题

### 问题描述

按照如下方式配置 keepalive 之后发现 nginx 与 upstream server 的长连接并没有开启.

```
upstream test {
    keepalive 16;
    ip_hash;
    server 1.1.1.1:80;
    server 2.2.2.2:80;
}

server {
    proxy_set_header Host $host;
    proxy_set_header Connection "";
    
    location / {
        proxy_pass http://test;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Connection "";
    }
}
```

### 原因

参考 [nginx 官方文档 - keepalive](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive) `When using load balancing methods other than the default round-robin method, it is necessary to activate them before the keepalive directive.` 因为在 upstream 配置块里使用了 `ip_hash` 负载均衡算法，导致 `keepalive` 配置不生效.这也是 nginx 里少有的跟配置顺序相关的配置。

### 修正

修正后配置如下

```
upstream test {
    ip_hash;
    server 1.1.1.1:80;
    server 2.2.2.2:80;
    keepalive 16;  # 将 keepalive 配置配置到最后面
}

server {
    proxy_set_header Host $host;
    proxy_set_header Connection "";
    
    location / {
        proxy_pass http://test;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Connection "";
    }
}
```

### 参考

- [top 10 错误配置](https://www.nginx.com/blog/avoiding-top-10-nginx-configuration-mistakes/)
- [nginx 官方文档 - keepalive](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive)

## nginx 惊群效应引发 cpu 使用率高

### 问题描述

从 tengine-2.2.0 升级到 tengine-2.3.3 之后，发现升级之后 sys cpu 使用及平均负载会相较于之前高很多。同时,通过 `top` 可以看到 nginx worker 进程的 cpu 使用率也很高。

```
worker_processes  16;
worker_cpu_affinity auto;
worker_rlimit_nofile 655360;
daemon on;
error_log  var/logs/error.log;
pid        var/run/nginx.pid;
events {
   use epoll;
   worker_connections  655360;
}
http {
    # ...
}
```

使用 `strace -C -T -ttt -p ${nginx_pid}  -o /tmp/strac.log` 后可以观察到如下日志:

```
1670466059.603682 write(116, "{\"@timestamp\": \"2022-12-08T10:20"..., 568) = 568 <0.000014>
1670466059.603716 sendto(43, "<190>Dec  8 10:20:59 bx-slb-10-1"..., 616, 0, NULL, 0) = 616 <0.000018>
1670466059.603766 setsockopt(309, SOL_TCP, TCP_NODELAY, [1], 4) = 0 <0.000021>
1670466059.603824 epoll_wait(131, [{EPOLLIN|EPOLLOUT|EPOLLRDHUP, {u32=3918290177, u64=140707046907137}}], 512, 42) = 1 <0.000027>
1670466059.603878 recvfrom(196, "", 8192, 0, NULL, NULL) = 0 <0.000011>
1670466059.603913 close(196)            = 0 <0.000024>
1670466059.603955 epoll_wait(131, [{EPOLLIN, {u32=3918234496, u64=140707046851456}}], 512, 42) = 1 <0.000152>
1670466059.604138 accept4(12, 0x7ffdb837b1f0, 0x7ffdb837b26c, SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable) <0.000027>
1670466059.604205 epoll_wait(131, [{EPOLLIN, {u32=3918234496, u64=140707046851456}}], 512, 41) = 1 <0.000345>
1670466059.604590 accept4(12, 0x7ffdb837b1f0, 0x7ffdb837b26c, SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable) <0.000021>
1670466059.604640 epoll_wait(131, [{EPOLLIN, {u32=3918234496, u64=140707046851456}}], 512, 41) = 1 <0.000092>
1670466059.604768 accept4(12, 0x7ffdb837b1f0, 0x7ffdb837b26c, SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable) <0.000013>
1670466059.604813 epoll_wait(131, [{EPOLLIN, {u32=3918234496, u64=140707046851456}}], 512, 41) = 1 <0.000189>
1670466059.605043 accept4(12, 0x7ffdb837b1f0, 0x7ffdb837b26c, SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable) <0.000026>
1670466059.605114 epoll_wait(131, [{EPOLLIN|EPOLLOUT, {u32=3918271617, u64=140707046888577}}], 512, 40) = 1 <0.000014>
1670466059.605156 recvfrom(279, "HTTP/1.1 200 OK\r\nServer: nginx\r\n"..., 16384, 0, NULL, NULL) = 377 <0.000013>
1670466059.605203 writev(306, [{"HTTP/1.1 200 OK\r\nServer: Tengine"..., 358}, {"4d\r\n", 4}, {"{\"retcode\":20000000,\"msg\":\"\",\"da"..., 77}, {"\r\n", 2}, {"0\r\n\r\n", 5}], 5) = 446 <0.000021>
1670466059.605279 write(116, "{\"@timestamp\": \"2022-12-08T10:20"..., 525) = 525 <0.000016>
1670466059.605316 sendto(43, "<190>Dec  8 10:20:59 bx-slb-10-1"..., 573, 0, NULL, 0) = 573 <0.000014>
1670466059.605350 epoll_wait(131, [{EPOLLIN|EPOLLOUT|EPOLLRDHUP, {u32=3918243081, u64=140707046860041}}], 512, 40) = 1 <0.000614>
1670466059.605986 recvfrom(169, "", 8192, 0, NULL, NULL) = 0 <0.000012>
1670466059.606015 close(169)            = 0 <0.000024>
1670466059.606056 epoll_wait(131, [{EPOLLIN, {u32=3918234496, u64=140707046851456}}], 512, 39) = 1 <0.000453>
1670466059.606544 accept4(12, 0x7ffdb837b1f0, 0x7ffdb837b26c, SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable) <0.000028>
1670466059.606612 epoll_wait(131, [{EPOLLIN, {u32=3918234496, u64=140707046851456}}], 512, 39) = 1 <0.000946>
```

可以看到会有很多 `epoll_wait` 的 `accept4` 返回了 `Resource temporarily unavailable` 的字段。

使用 `sar -w` 可以看到如下内容，上下文切换要高很多，这也是 worker 进程占用高的原因

```
11:00:01 AM    proc/s   cswch/s
Average:        21.96  132824.27`
```

### 原因

按照 [nginx 官方文档 - accept_mutex](http://nginx.org/en/docs/ngx_core_module.html#accept_mutex) 中描述，nginx-1.11.3 之后 `accept_mutex` 默认为关闭状态，当连接到来时，所有 worker 进程会尝试建立连接，就会导致所有 worker 进程都会被唤醒。开启这个参数后，相当于对连接加了一把锁，每个 worker 进程抢占这把锁,抢到锁的 worker 进程处理请求，没有抢到的继续处于 sleep 状态。避免由于新连接请求过来时所有 worker 都会被惊醒，造成 cpu 上下文切换。

tengine 与 nginx版本对应如下:

| tengine | nginx  | 备注                                                         |
| :------ | :----- | :----------------------------------------------------------- |
| 2.2.0   | 1.8.1  |                                                              |
| 2.2.1   | 1.8.1  |                                                              |
| 2.2.3   | 1.8.1  |                                                              |
| 2.3.0   | 1.15.9 | 废弃 `dso_tool` 工具以及 `dso` 配置指令,可以使用 `load_module` 指令动态加载.参考: [tengine - dso](http://tengine.taobao.org/document_cn/dso_cn.html) |
| 2.3.1   | 1.16.0 |                                                              |
| 2.3.2   | 1.17.3 |                                                              |
| 2.3.3   | 1.18.0 |                                                              |

### 解决

`accept_mutex` 参数配置为 on
```
event {
    accept_mutex on;
}
```

### 思考

开启 `accept_mutex` 参数后，同一时刻只有一个 worker 进程处理新的连接请求。这就导致在高并发的场景下一定会有一些性能损耗。因此可以使用 `reuseport` 参数来开启多个 worker 进程监听同一个 socket 套接字。[Socket Sharding in NGINX Release 1.9.1](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/) 这里看起来开启 reuseport 性能会高不少。

### 参考

- [Nginx惊群效应引起的系统高负载](https://zhuanlan.zhihu.com/p/401910162)
- [Nginx 是如何解决 epoll 惊群的](https://ld246.com/article/1588731832846)
- [accept 与 epoll 惊群](https://pureage.info/2015/12/22/thundering-herd.html)
- [nginx 官方文档 - accept_mutex](http://nginx.org/en/docs/ngx_core_module.html#accept_mutex)
- [知乎 - reuseport](https://www.zhihu.com/question/51618274)
- [Socket Sharding in NGINX Release 1.9.1](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/)

## nginx POST 请求偶发 502

### 问题描述

从 tengine-2.2.0 升级到 tengine-2.3.3 之后，nginx 错误日志中有如下报错，且发现 POST 请求会偶发 502，GET 请求没有 502

```
upstream prematurely closed connection while reading response header from upstream, client: 2.2.2.2, server: xxx, request: "POST /path/to/post/uri HTTP/1.1", upstream: "http://1.1.1.1/path/to/post/uri", host: "xxx"
```

抓包后发现后端 upstream server 在接收到请求后，返回了 FIN、ACK 的 TCP 包，导致 nginx 与 upstream server 连接断开。POST 请求不会重发，GET 请求会复用其他连接重新发到下一个个可用的 upstream server。抓包截图如下：

![502 POST 请求抓包](/images/nginx-upstream-tcpdump-post.png)

![GET 请求抓包](/images/nginx-upstream-tcpdump-get.png)

### 原因

tengine-2.2.0 升级到 tengine-2.3.3 后，对应 nginx 1.18.0，[官方文档 proxy_next_upstream](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) 描述说在  1.9.13 之后的版本里添加了 `non_idempotent` 参数。对于非幂等的 POST, LOCK, PATCH 请求，需要显示打开此参数，nginx 才会帮忙转发。

### 解决

在确认 POST 请求为幂等的情况下，`proxy_next_upstream` 参数显式配置为 `error timeout non_idempotent`

### 参考

- [nginx 官方文档 - proxy_next_upstream](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream)