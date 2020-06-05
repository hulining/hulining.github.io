---
title: LVS Nginx HaProxy 负载均衡器对比分析
date: 2020/06/04
tags:
  - Linux
  - LVS
  - Nginx
  - Haproxy
categories:
  - Linux
abbrlink: 
description: 本文章主要介绍目前常用的三种负载均衡器 LVS Nginx HaProxy 的特性及区别
---

> LVS

- LVS 工作在 OSI 模型第 4 层,基于内核实现,没有额外流量的产生,性能最好
- 配置性低,无需配置文件,只是提供了 `ipadm` 命令行工具对 ipvs 规则进行设置与配置

> Nginx

- Nginx 工作在 OSI 模型第七层
- 安装配置较为简单,支持正则表达式匹配,能够针对 http 应用层做一些分流策.
- 高性能,可承担高并发压力
- 同时也是功能强大的 Web 应用服务器

> HaProxy

- 专业级负载均衡软件,工作在 OSI 模型第 4/7 层
- 相较于 Nginx 有更出色的负载均衡速度,在并发处理上也优于 Nginx.但在正则表达式支持上较弱
