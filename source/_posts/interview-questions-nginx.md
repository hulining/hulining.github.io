---
title: 运维面试题之 Nginx
date: 2020/06/06
tags:
  - Nginx
  - 面试
categories:
  - Nginx
abbrlink: 
description: 总结整理常见 Nginx 面试题,以作备忘
---

## Nginx 支持的负载均衡算法有哪些?分别有什么应用场景

- round robin 轮询

依次将请求分配到各个后端服务器中,是 Nginx 默认的负载均衡方式.适用于后端服务器性能一致的情况.

```conf
upstream bakend {
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
```

- weight 加权轮询

加权轮询.根据权重比率分发请求到各个后端服务器中.使用 `weight` 指定权重.适用于后端服务器性能不一致的情况.

```conf
upstream bakend {
    server 172.16.1.2:8080 weight=10;
    server 172.16.1.3:8080 weight=20;
}
```

- ip_hash 客户端 IP 哈希

根据客户端 IP 地址的 hash 值将请求发送到后端服务器.可保证来自同一个客户端的请求被转发到固定的后端服务器中,可解决 session 问题.

```conf
upstream bakend {
    ip_hash;
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
```

- least_conn 最小连接数

根据后端服务器的连接数进行分发,同时考虑后端服务器的权重.

```conf
upstream bakend {
    least_conn;
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
```

- least_time 最小平均响应时间

根据后端服务器的平均响应时间及活动连接数进行分发,同时考虑后端服务器的权重.

```conf
upstream bakend {
    least_time header | last_byte [inflight];
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
# 如果指定 `header`,则使用 `$upstream_header_time`(请求头的响应时间)作为响应时间的判断依据
# 如果指定 `last_byte`,则使用 `$upstream_response_time`(请求响应时间)作为响应时间的判断依据.如果同时指定了 `inflight` 参数,则不完整的请求也会计算在内.
```

- `hash key` 自定义 hash 键轮询

根据自定义 `key` 的值进行哈希从而选择后端服务器.`key` 可以包含文本,变量及其组合.

```conf
upstream bakend {
    hash <key> [consistent];
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
# 如果指定了 `consistent` 参数,则将使用 ketama 一致性哈希算法.
# 该算法可以确保向组中添加服务器或从组中删除时,只有很少的键被重新映射到与原来不同的服务器.有助于缓存服务器获得更高的缓存命中率
```

- random 完全随机

> nginx version >= 1.15.1

将请求随机转发到后端服务器,同时考虑后端服务器权重.

```conf
upstream bakend {
    random [two [method]];
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
# 可选的 `two` 参数随机选择两个后端服务器,然后使用指定的 `method` 选择一个后端服务器.默认方法是 `least_conn`,将请求传递给活动连接数最少的服务器
```

## Nginx location 匹配的顺序是什么

Nginx location 按照如下优先级顺序进行匹配

- 精确匹配: `location = /path`
- 以某个常规字符串开头: `location ^~ /prefix_path`,且 `prefix_path` 路径长者优先匹配
- 以正则表达式进行匹配: `location ~ regex_path` 或 `location ~* regex_path`(`~*` 表示忽略大小写).不管正则表达式如何进行锚定匹配,首先按照定义的顺序进行匹配.见如下示例

```conf
# 如下情况 uri=/image/.jpg 返回 /image
location ~ /image {
    return 200 '/image';
}
location ~ \.jpg {
    return 200 '.jpg';
}

# 如下情况 uri=/image/.jpg 返回 .jpg
location ~ \.jpg {
    return 200 '.jpg';
}
location ~ /image {
    return 200 '/image';
}
```

- 通用匹配: `location /path`,且 path 路径长者优先匹配

## Nginx servername 匹配的顺序是什么

1. 精确 server_name 匹配.
2. 以 * 通配符开始的泛域名
3. 以 * 通配符结束的泛域名
4. 匹配正则表达式
5. listen 指定的 default server_name
6. http 模块下的第一个server配置块中的server_name

## nginx 重新加载配置文件的流程是什么

1. 向 master 发送 reload信号
2. master 校验配置文件语法
3. master 进程打开新的监听端口
4. master 进程用新配置启动新的 worker 子进程
5. master 进程向 worker 子进程发送 quit 信号
6. 旧 worker 进程关闭监听句柄,处理完当前连接后后结束进程

## Nginx 调优

### 合理设置 Nginx worker 进程数,并绑定到不同 CPU

一般情况下,应将 worker 进程数设置为 CPU 核数,或设置为 `auto` 根据系统 CPU 自动配置.

可以通过 `worker_cpu_affinity` 将 Nginx worker 进程绑定到不同的 CPU 上,避免多个进程竞争同一个 CPU 资源,充分利用 CPU 多核资源.

```conf
worker_processes  4;
worker_cpu_affinity 0001 0010 0100 1000;
```

### 使用 epoll 事件驱动模型,合理设置单个进程的最大连接数及最大打开文件数量

> 进程最大连接数受系统最大打开文件数限制,因此需要使用 `ulimit` 设置打开的文件数.或在 `/etc/security/limits.conf`, `/etc/security/limits.d/nginx.conf` 进行配置.配置如下

```bash
ulimit -n $max_open_files # 设置系统最大打开文件数
cat /etc/security/limits.d/nginx.conf
*       soft    nproc   131072
*       hard    nproc   131072
*       soft    nofile  131072
*       hard    nofile  131072
```

```conf
events {
    use epoll;
    worker_connections 15000;
    worker_rlimit_nofile 65535;
}
```

### 开启 access_log buffer 缓冲区

在高并发场景下,nginx 日志写入磁盘可能会极大的影响 nginx 的性能.可以通过设置 `access_log ... buffer=size flush=time` 来开启 nginx access_log 日志的缓冲区大小,并设置同步到磁盘的时间.

```conf
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
# 或禁用某些不必要的日志.如静态文件的访问日志.
access_log off;
```

### 开启高效文件传输模式,开启或 gzip 压缩

- 配置段 `sendfile` 参数用于开启文件高效传输模式
- 配置段`gzip` 参数可用于开启压缩功能

```conf
http {
    gzip  on;
    sendfile on;
}
```

### 优化 Nginx 连接超时时间

- 配置段 `keepalive_timeout` 参数用于设置客户端连接保持会话的超时时间,超过这个时间服务器会关闭该连接
- 配置段 `client_header_timeout` 参数用于设置读取客户端请求头数据的超时时间.如果读取请求头超时,服务器将返回 "Request time out (408)" 错误.
- 配置段 `client_body_timeout` 参数用于设置读取客户端请求主体数据的超时时间.如果读取请求体超时,服务器将返回 "Request time out (408)" 错误.
- 配置段 `send_timeout` 参数用于指定响应客户端的超时时间,如果超时,Nginx 将会关闭连接.

```conf
http {
    # 在大并发时,需要合理调小如下参数
    keepalive_timeout  65;
    client_header_timeout 15;
    client_body_timeout 15;
    send_timeout 25;
}
```

### Nginx 限制连接与限制请求速率

- `limit_conn_zone` 参数用于为共享内存区域设置参数
- `limit_conn` 参数用于设置指定内存区域的最大连接数
- `limit_conn_status` 参数用于设置超过最大连接数的请求状态码

```conf
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m; # 设置 $binary_remote_addr 客户端地址的地址区域为 10m
    limit_conn ${zone} ${num}; # 设置指定 zone 的最大连接数
    limit_conn_status ${http_code}; # 设置超过最大连接数的状态码,默认 503
}
```

- `limit_req_zone` 参数用于为共享内存区域设置参数
- `limit_req` 参数用于设置指定内存区域的最大连接数
- `limit_req_status` 参数用于设置超过最大连接数的请求状态码

```conf
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s; 
    # 设置 $binary_remote_addr 客户端地址的地址区域为 10m,平均请求速率最大为 1r/s
    limit_req zone=${zone} burst=${num}; # 设置指定 zone 的最大并发请求
    limit_req_status ${http_code}; # 设置超过最大请求数的状态码,默认 503
}
```

### 合理配置 Nginx expires 缓存

- `expires` 参数用于设置用户访问内容的缓存时间.用户会在本地浏览器中缓存这些内容,直到超过缓存时间.多用于配置静态内容

```conf
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css)$ {
    expires     3650d;
}
```

### 限制上传文件大小

- `http` 配置段 `send_timeout` 参数用于设置客户端最大请求体大小.如果超过出,客户端会收到 413 错误,即请求体过大

```conf
http {
    client_max_body_size 8m;    # 设置客户端最大请求体大小为8M
}
```

### 优化服务器域名的散列表大小

如果在 server_name 中配置了长域名,可能会出现如下错误.

```text
nginx: [emerg] could not build the server_names_hash, you should increase server_names_hash_bucket_size: 64
nginx: configuration file /usr/local/nginx/conf/nginx.conf test failed
```

此时需要对 `http` 配置段 `server_names_hash_bucket_size` 进行调整,如下:

```conf
# ngx_http_core_module
http {
    server_names_hash_bucket_size  512;
}
```

## Nginx 安全

### 隐藏/修改 Nginx 响应头 Server 信息

在配置文件 `http` 配置段中添加 `server_tokens off;` 配置项,即可隐藏 Nginx 版本号

要修改 Nginx 响应头部 Server 信息,需要对 `src/core/nginx.h` 源码文件进行修改,再按照需求进行编译.慎重!

```c
// src/core/nginx.h
#define nginx_version      1018000
#define NGINX_VERSION      "version"
#define NGINX_VER          "server/" NGINX_VERSION
```

修改完成后,响应头部如下:

```text
HTTP/1.1 200 OK
Server: server/version
...
```

### 只允许指定客户端访问

- `allow`,`deny` 参数用于配置可以/禁止访问 Nginx 指定内容的客户端.可配置在 `http, server, location, limit_except` 配置段中

```conf
location / {
    allow 192.168.1.1/24;
    deny all;
}
```

### 配置 Nginx 防盗链

防盗链：简单地说，就是某些不法网站未经许可，通过在其自身网站程序里非法调用其他网站的资源，然后在自己的网站上显示这些调用的资源，使得被盗链的那一端消耗带宽资源

- 根据 HTTP `referer` 字段实现防盗链: referer 是 HTTP的一个首部字段,用于指明用户请求的 URL 是从哪个页面通过链接跳转过来的
- 根据 cookie 实现防盗链: cookie 是服务器贴在客户端身上的 "标签",服务器用它来识别客户端

```conf
location ~ .*\.(gif|jpg|jpeg|png|bm|swf|flv|rar|zip|gz|bz2)$ {
    valid_referers  none  blocked  *.test.com  *.abc.com; # 表示这些地址可以访问上面的媒体资源
    if ($invalid_referer) {
        # 如果不是从以上域名访问,则返回 403
        return 403
    }
```

### Nginx 错误页面优雅显示

- 使用 `error_page` 参数可对指定错误码返回的页面

```conf
location / {
    root   html/www;
    index  index.html index.htm;
    error_page 400 401 402 403 404 405 408 410 412 413 414 415 500 501 502 503 506 = http://www.xxxx.com/error.html;
}
```

## Nginx 常用信号

kill 信号 | 解释 | nginx 命令
:--- | :--- | :---
TERM(15), INT(2) | 快速停止 | nginx -s stop
QUIT(3) | 优雅的停止 | nginx -s quit
HUP(1) | 优雅的启动新的进程重新加载配置文件 | nginx -s reload
USR1(10) | 日志文件滚动 | nginx -s reopen
USR2(12) | 升级二进制可执行文件 | -
WINCH(28) | 优雅的关闭 worker 进程

### Nginx 平滑升级与回退

假设 nginx 版本升级前进程 id 为 `30818`,则平滑升级与版本回退过程如下:

```bash
# 向原 master 进程发送 USR2 信号,使用新版本的 nginx 启动 master 进程,此时老版本的 nginx 仍在处理请求
kill -USR2 30818
# 向原 master 进程发送 WINCH 信号,优雅停止旧版本的 nginx worker,此时切换为新版本的 nginx 处理请求
kill -WINCH 30818
# 向原 master 进程发送 QUIT 信号,优雅停止旧版本的 nginx master
kill -QUIT 30818
```

---

参考

- [Nginx 调优](https://www.cnblogs.com/zhichaoma/p/7989655.html)
