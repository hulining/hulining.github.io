---
title: Golang 项目结构
date: 2020/06/24
tags:
  - go
categories:
  - go
description: 在学习 Go 过程中,不知道 GitHub 上 Go 项目的代码结构是如何组织的,在阅读代码过程中会有一定的障碍.现将官方推荐的项目代码组织结构进行整理,以作备忘.
---


[https://github.com/golang-standards/project-layout](https://github.com/golang-standards/project-layout) 项目是常见的 Go 项目代码的布局方式.我们在阅读或开发 Go 项目时,可以遵循此项目中的布局方式进行组织代码.

以下是 Go 项目中常见目录及其简单介绍.

## Go 目录

- `/cmd`: 项目的主程序目录

每个应用程序的目录名称应该与您的可执行文件名称一致.如 `/cmd/myapp`.

不要在此目录中放置很多代码.如果您任务该代码可以被导入并在其它项目中使用,则应将其放在 `/pkg` 目录中.如果代码不可导入或不希望其他人使用它,则应将代码放在 `/internal` 目录中.

此目录下通常有一个简单的 main 函数,并通过导入或调用 `/internal` 和 `/pkg` 目录中的代码,而没有其他的函数.

如,[prometheus/cmd](https://github.com/prometheus/prometheus/tree/master/cmd)

- `/internal`: 私有应用程序和代码库.放在该包中的代码,表明只希望在此项目内部使用,其它项目不能使用.

- `/pkg`: 外部应用程序可以使用的库代码.该目录与 `internal` 对应,是公开的.如 [prometheus/pkg](https://github.com/prometheus/prometheus/tree/master/pkg).

一般来说,放在该目录下的代码应该和具体业务无关,方便本项目或其他项目重用.当你决定将代码放入该包时,你应该对其负责,因为别人很可能使用它.

- `/vendor`: 应用程序依赖项目录.

`go mod vendor` 命令将为您创建 `/vendor` 目录.注意,如果您使用的不是 Go 1.14 版本(1.14 版本默认开启),则可能需要在 `go build` 命令中添加 `-mod=vender` 标志.

注意,从 1.13 开始,Go 还启用了模块代码功能(默认情况下使用 [`https://proxy.golang.org`](https://proxy.golang.org) 作为其模块的代理服务器).如果是这样,那么您根本不需要 `vendor` 目录.

## 服务应用目录

- `/api`: 主要保存 OpenAPI/Swagger 规范, JSON 格式文件和协议定义文件.如 [kubernetes/api](https://github.com/kubernetes/kubernetes/tree/master/api)

## Web 应用目录

- `/web`: Web 应用程序特定的组件,包含静态 Web 资源,服务端模版和 SPA.

## 常见应用目录

- `/configs`: 配置文件模版或默认配置目录.
- `/init`: 系统初始化(systemd, upstart,sysv)和进程管理/监控(runit, supervisord) 配置.
- `/scripts`: 用于执行构建,安装,分析等操作的各种脚本.

这些脚本可使根目录下的 Makefile 变得简单.如 [terraform/Makefile](https://github.com/hashicorp/terraform/blob/master/Makefile), [helm/scripts](https://github.com/helm/helm/tree/master/scripts).

- `/build`: 打包构建与持续集成.

将您的容器,操作系统软件包配置和脚本放在 `/build/package` 目录中.将您的持续集成(travis, circle, drone)配置和脚本放在 `/build/ci` 目录中.注意,有些 CI 工具对于其配置文件的位置非常挑剔.尝试将其放在 `/build/ci` 目录中,并将其链接到工具期望的位置(如果可能).

- `/deployments`: Iaas, Paas 系统和容器编排配置和模版目录.注意,有些代码仓库中,该目录为 `/deploy`.
- `/test`: 其它外部测试应用程序和测试数据目录.

## 其它目录

- `/docs`: 用户帮助文档.
- `/tools`: 此项目的工具.这些工具可以从 `/pkg` 和 `internal` 目录中导入代码
- `/examples`: 应用程序或公共库的示例
- `/third_party`: 第三方工具
- `/assets`: 仓库的其它资源.如,图片, logo 等
- `/website`: 项目的网站数据目录.

## 不应该包含的目录

- `/src`: 不要将 Java 项目的代码结构组织习惯应用到 Go 中
