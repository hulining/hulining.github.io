---
title: 运维面试题之 Nginx
date: 2020/06/06
tags:
  - Nginx
categories:
  - 面试题
  - Nginx
abbrlink: 
description: 总结整理常见 Nginx 面试题,以作备忘
---

## Nginx 支持的负载均衡算法有哪些?分别有什么应用场景?

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
// 如果指定 `header`,则使用 `$upstream_header_time`(请求头的响应时间)作为响应时间的判断依据
// 如果指定 `last_byte`,则使用 `$upstream_response_time`(请求响应时间)作为响应时间的判断依据.如果同时指定了 `inflight` 参数,则不完整的请求也会计算在内.
```

- `hash key` 自定义 hash 键轮询

根据自定义 `key` 的值进行哈希从而选择后端服务器.`key` 可以包含文本,变量及其组合.

```conf
upstream bakend {
    hash <key> [consistent];
    server 172.16.1.2:8080;
    server 172.16.1.3:8080;
}
// 如果指定了 `consistent` 参数,则将使用 ketama 一致性哈希算法.
// 该算法可以确保向组中添加服务器或从组中删除时,只有很少的键被重新映射到与原来不同的服务器.有助于缓存服务器获得更高的缓存命中率
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
// 可选的 `two` 参数随机选择两个后端服务器,然后使用指定的 `method` 选择一个后端服务器.默认方法是 `least_conn`,将请求传递给活动连接数最少的服务器
```

## Nginx location 匹配的顺序是什么?

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
