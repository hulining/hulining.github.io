---
title: Linux 之 systemd 的工作原理及使用
date: 2020/06/11
tags:
  - Linux
  - systemd
categories:
  - Linux
abbrlink: 
description: 详细介绍 Linux systemd 工作原理及使用方式
---

## 特性

- 兼容 SysVinit 和 LSB init scripts
- 以并行方式启动,启动速度更快
  - 尽可能启动更少的进程
  - 尽可能将更多进程并行启动
- 提供按需启动能力,只有在某个服务被真正请求的时候才启动它
- 采用 Linux的 Cgroup 特性跟踪和管理进程的生命周期.Systemd 只需要简单地遍历指定的 CGroup 即可正确地找到所有的相关进程
- 实现事务性依赖关系管理.系统启动过程是由很多的独立工作共同组成的,这些工作之间可能存在依赖关系.Systemd 维护一个"事务一致性"的概念,保证所有相关的服务都可以正常启动而不会出现互相依赖,以至于死锁的情况.
- 使用 systemd-journald 提供日志服务.有便于管理,零维护,高性能,统一化的特点
- 向后兼容 SysV 的服务脚本

## 基本概念

### unit

系统初始化需要做的事情非常多,如启动后台服务以及做一些配置工作.这个过程中的每一步都被 Systemd 抽象为一个配置单元,即 `unit`.它主要包含如下几种类型

| 名称 | 文件扩展名 | 用途
:--- | :--- | :---
`service unit` | `.service` | 用于定义系统类服务
`target unit` | `.target` | 用于实现模拟"运行级别"
`mount unit` | `.mount` | 用于定义文件系统挂载点
`socket unit` | `.socket` | 用于标识进程间通信的 socket 文件
`snapshot unit` | `.snapshot` | 管理系统快照
`swap unit` | `.swap` | 用于标识 swap 设备
`automount unit` | `.automount` | 文件系统自动挂载设备
`path unit` | `.path` | 用于定义文件系统中的一个文件或目录
`timer unit` | `.timer` | 用来定时触发用户定义的操作

### Target和运行级别的对应关系

sysvinit | systemd target | 备注
:--- | :--- | :---
0 | poweroff.target | 关闭系统
1 | rescue.target | 单用户模式
2 | multi-user.target | 多用户模式,无网络
3 | multi-user.target | 多用户模式
4 |  - | 没有用到
5 | graphical.target | 多用户,图形化界面
6 | reboot.target | 重启

## service unit file

文件通常由三部分组成

### [Unit]

`Unit` 配置块通常是配置文件的第一个配置块,用来定义 Unit 的元数据,以及配置与其他 Unit 的依赖关系.它的主要字段如下:

- `Description`: 简短描述
- `Documentation`: 文档地址
- `Requires`: 当前 Unit 依赖的其他 Unit,如果它们没有运行,当前 Unit 会启动失败
- `Wants`: 与当前 Unit 配合的其他 Unit,如果它们没有运行,当前 Unit 不会启动失败
- `BindsTo`: 与 Requires 类似,它指定的 Unit 如果退出,会导致当前 Unit 停止运行
- `Before`: 当前 Unit 必须在指定的 Unit 之前启动
- `After`: 当前 Unit 必须在指定的 Unit 之后启动
- `Conflicts`: 这里指定的 Unit 不能与当前 Unit 同时运行
- `Condition...`: 当前 Unit 运行必须满足的条件，否则不会运行
- `Assert...`:当前 Unit 运行必须满足的条件，否则会报启动失败

### [Service]

`Service` 配置块用来定义 Service 的属性,只有 Service 类型的 Unit 才有这个配置块.它的主要字段如下:

- `Type`: 定义启动时的进程行为.可选值为
  - `simple`: 默认值,执行 ExecStart 指定的命令,启动主进程
  - `forking`: 以 fork 方式从父进程创建子进程,创建后父进程会立即退出
  - `oneshot`: 一次性进程,Systemd 会等当前服务退出,再继续往下执行
  - `dbus`: 当前服务通过 D-Bus 启动
  - `notify`: 当前服务启动完毕,会通知 Systemd,再继续往下执行
  - `idle`: 若有其他任务执行完毕,当前服务才会运行
- `Environment`: 指定环境变量
- `ExecStart`: 启动当前服务的命令
- `ExecStartPre`: 启动当前服务之前执行的命令
- `Restart`: 定义何种情况 Systemd 会自动重启当前服务.可选值为 `always`,`on-success`,`on-failure`,`on-abnormal`,`on-abort`,`on-watchdog`
- `ExecReload`: 重新加载当前服务时执行的命令
- `ExecStop`: 停止当前服务时执行的命令
- `ExecStartPost`: 启动当前服务之后执行的命令
- `ExecStopPost`: 停止当其服务之后执行的命令
- `RestartSec`: 自动重启当前服务间隔的秒数
- `TimeoutSec`: 定义 Systemd 停止当前服务之前等待的秒数

### [Install]

`Install` 配置块通常在文件最后,用来定义如何启动,以及是否开机启动.它的主要字段如下

- `WantedBy`: 它的值是一个或多个 `Target`,当前 Unit enable 时(服务开机启动时),会在 `/etc/systemd/system` 目录及指定 `${target}.wants` 子目录下创建软链接
- `RequiredBy`: 它的值是一个或多个 `Target`,当前 Unit enable 时(服务开机启动时),会在 `/etc/systemd/system` 目录及指定 `${target}.required` 子目录下创建软链接
`Alias`: 当前 Unit 可用于启动的别名
`Also`: 当前 Unit enable 时(服务开机启动时),同时激活的其他 Unit
