---
title: 运维面试题之 docker
date: 2020/06/04
tags:
  - docker
categories:
  - 面试
  - docker
abbrlink: 
description: 总结整理常见 docker 面试题,以作备忘
---

## docker 配置

docker 默认配置见官方文档 [daemon.json](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file) 示例.且提供了众多[命令行参数](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon).下面介绍一些常用配置:

```text
# 进程相关
--config-file string: 指定配置文件,默认是 `/etc/docker/daemon.json`
-p, --pidfile string: 指定 PID 文件.默认是 `/var/run/docker.pid`
--containerd string: 指定 grpc 地址
--data-root string: 指定 docker 镜像和容器相关文件保存位置.默认是 `/var/lib/docker`
--log-driver string: 指定日志驱动.默认为 `json-file`
--log-level string: 指定日志级别.可选值为 debug,info,warn,error,fatal.默认 `info`

log-opts: 指定日志选项,只能用于配置文件中.`max-size` 指定日志文件大小,`max-file` 指定日志文件个数

# 镜像相关
--insecure-registry list: 指定不安全的镜像仓库地址
--registry-mirror list: 指定安全的镜像仓库.配置文件中为 `registry-mirrors`

# 网络相关
--bip string: 指定 docker0 网桥 IP 地址
--default-gateway string: 子网 IPv4 默认网关
--default-gateway-v6 string: 子网 IPv6 默认网关
--fixed-cidr string: 指定子网 IPv4 地址
--fixed-cidr-v6 string: 指定子网 IPv6 地址
--mtu int: 指定容器最大传输单元
--dns list: 指定容器使用的 DNS 地址

# 调试
-D, --debug: 开启 debug 模式,多用于调试
```

## docker 底层原理

docker 使用 Go 语言编写,并利用 Linux 内核的多个功能来实现其功能.

### Namespaces 名称空间

docker 使用一种称为 `namespaces` 的技术来为容器提供运行环境的隔离.运行容器时,docker 引擎会为该容器创建一组如下的名称空间

名称空间 | 描述
:---: | :---
PID | 提供进程隔离.每个容器中都有独立的进程号
Network | 提供网络资源隔离.每个容器都有独立的网络设备接口, IPv4, IPv6 协议栈,路由表,防火墙等
IPC | 提供进程间间通信资源隔离.容器中进程间通信仍然使用 Linux 进程间通信方法,信号量,消息队列,共享内存等
Mount | 提供文件系统隔离.每个容器都有独立的 `/` 根文件系统
UTS | 提供容器主机名和域名隔离.每个容器都有独立的主机名和域名,其主机名一般为容器 ID
User | 提供用户及用户组隔离.每个容器都有独立的用户,用户组及其相关访问权限

### Control groups

控制组(cgroups)以一组进程为目标进行系统资源分配和控制,它提供了如下功能:

- Resource limitation,资源限制,如内存,CPU 等硬件资源
- Prioritization,优先级控制
- Accounting,审计或统计
- Controll,进程控制,如进程挂起与恢复

系统管理员可更具体地控制对系统资源的分配,优先顺序,拒绝,管理和监控.可更好地根据任务和用户分配硬件资源,提高总体效率.在实践中,系统管理员一般会利用 CGroup 做下面这些事:

- 隔离进程集合,并限制他们所消耗的资源
- 为这组进程分配其足够使用的内存,网络带宽和磁盘存储限制
- 限制访问某些设备

### Union file systems

Union file systems(联合文件系统)是通过创建层级进行操作的文件系统,它将对文件系统的修改作为一次提交来一层层的叠加.常用的包含 overlay2 aufs

overlay2 采用三层结构:

- lowerdir: 只读层,镜像层
- uperdir: 读写层.创建容器时创建,所有对容器的改动发生在这里
- merged: 容器挂载点,将以上两层进行合并后看到的内容

## docker 几种网络模型

可以使用 `docker run --net=xxx` 指定容器使用的网络类型

- `bridge`: 桥接,默认的网络模型.为主机上的容器分配单独的网络名称空间,IP 等,并将容器中的网络接口连接到虚拟网桥上(docker0)
- `host`: 与宿主机共用网络名称空间,容器使用宿主机的 IP 和端口
- `overlay`: 与其它容器共用网络名称空间,使容器间能够通过 lo 进行通信
- `none`: 容器有独立的网络名称空间,但不进行任何网络配置,只有本地地址
- `macvlan`: 为容器分配 MAC 地址,使其在网络上显示为物理设备.可通过 MAC 地址直接将流量路由到容器.

## docker 开发最佳实践

### 保持镜像尽可能的小

- 尽量使用 `ENV` 和 `ARG` 让人不改或者少改 Dockerfile 即可做构建对应版本的镜像

- 尽量减少 Dockerfile 中单独的 RUN 命令的数量来减少镜像的层数

```Dockerfile
RUN mkdir /data
RUN touch /data/index.html
```

```Dockerfile
RUN mkdir /data && touch /data/index.html
```

- 从适当的基础镜像开始.例如,如果您需要 JDK,请考虑基于正式的 openjdk 镜像,而不是基于 ubuntu 镜像开始,再将 openjdk 的安装作为 Dockerfile 的一部分
- 可以在官方 Dockerfile 里添加一些常见的排错命令,也可以将二进制及其依赖库添加到镜像中,参见[为容器镜像定制安装Linux工具](https://docs.lvrui.io/2018/10/19/%E4%B8%BA%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F%E5%AE%9A%E5%88%B6%E5%AE%89%E8%A3%85Linux%E5%B7%A5%E5%85%B7/).如

```Dockerfile
# 本示例仅作为示例演示,并没有实际意义
# 系统环境 CentOS 7.5.1804,发现 centos:centos7.5.1804 没有 lsof 工具.添加一下
# 首先在宿主机中安装 lsof 工具,并通过 ldd 查看其依赖库 `ldd $(which lsof)`
# 在 centos:centos7.5.1804 镜像启动的容器中查找 lsof 的依赖库,可看到都是存在的.因此直接将二进制文件复制进入即可
FROM centos:centos7.5.1804
ADD lsof /usr/sbin/
```

- 使用多阶段构建.例如,您可以使用 maven 镜像构建 Java 应用程序,然后使用 tomcat 镜像并将构建的 Java 程序复制到正确的位置.这意味着您最终构建的镜像不包括构建所引入的所有库和依赖项.见如下示例

```Dockerfile
FROM golang:1.13.6-alpine3.10 as builder
WORKDIR $GOPATH/src/demo
COPY . $GOPATH/src/demo
RUN CGO_ENABLED=0 GOOS=linux go build -o /demo
EXPOSE 8080
ENTRYPOINT ["./demo"]
# 此时我们构建的镜像包括 go 的运行环境,相关源码或依赖文件,二进制可执行文件.镜像较大
# 其中运行环境与源码或依赖对于容器的运行来说都是多余的.
```

```Dockerfile
FROM golang:1.13.6-alpine3.10 as builder
WORKDIR $GOPATH/src/demo
COPY . $GOPATH/src/demo
RUN CGO_ENABLED=0 GOOS=linux go build -o /demo

FROM alpine:3.6
COPY --from=builder /demo .
EXPOSE 8080
ENTRYPOINT ["./demo"]
# 此时我们构建镜像仅包含二进制可执行文件.可以理解为 builder 构建完成后就将其丢弃了.镜像较小
```

- 尝试将共享的运行环境或依赖构建为独立的镜像,然后在此基础上构建其它镜像.Docker 只需要加载一次公共层,然后将它们缓存,可以更快的构建.如多个应用都需要自定义的 Tomcat 环境,可以将自定义 Tomcat 构建为单独镜像,而不需要每次构建应用时从最初始自定义 Tomcat 环境开始构建
- 容器时区问题可以在构建镜像的时候安装 `tzdate` 包，然后声明变量 `TZ` 即可声明容器运行的时区,或者构建的时候复制宿主机的`/etc/localtime`或者运行的时候挂载宿主机的`/etc/localtime`

### 在何处以及如何保留数据

- 避免将数据存储在容器的可写层中,这会增加容器大小,且效率不如使用 volumes 或 bind 挂载
- bind 挂载多用于开发或测试过程中.对于生产环境,请使用 volumes

## Dockerfile

```bash
docker build [OPTIONS] PATH | URL | -`
```

`docker build` 命令从 `Dockerfile` 及上下文构建镜像,构建的上下文是位于 `PATH` 或 `URL` 指定的位置的文件集合.`PATH` 是本地文件系统上的目录,`URL`是一个 Git 仓库位置.

上下文是递归处理的.因此 `PATH` 包括任何子目录,`URL` 包括仓库及其子模块.

构建过程是 Docker 守护进程进行的.构建的第一件事就是将整个上下文目录及其子目录发送到守护进程中.因此,最好以空目录作为上下文,仅包含 Dockerfile 及 Dockerfile 构建过程中所需要的文件.

### .dockerignore

通过在上下文中添加 `.dockerignore` 文件可以排除构建上下文中包含的文件或目录.

`.dockerignore` 文件使用 `#` 作为注释,每行包含一个忽略的文件或目录,支持使用 `*`,`?`,`**` 作为通配符匹配.分别表示所有文件,单个字符,目录递归.

在使用 `*` 忽略所有文件后,可以使用 `!` 向上下文中添加被忽略的文件

### 指令详解

#### ARG

`ARG` 定义一个变量,用户也可以在构建时使用 `--build-arg <varname>=<value>` 传入构建参数.如果该参数没有在 Dockerfile 中定义,则输出警告信息.

```Dockerfile
ARG <name>[=<default value>]
```

#### FROM

`FROM` 指定构建过程的基础镜像,一个有效的 Dockerfile 必须以 `FROM` 指令启动.它支持使用 `ARG` 定义的变量

```Dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

#### RUN

`RUN` 指令从当前镜像最新层执行命令并提交结果.生成的镜像层用于 Dockerfile 中下一步.

```Dockerfile
RUN COMMAND
```

#### LABEL

`LABEL` 指令为镜像打标签

```Dockerfile
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

#### EXPOSE

`EXPOSE` 指令暴露 Docker 容器监听的端口

```Dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

#### ENV

`ENV` 指令定义环境变量

```Dockerfile
ENV <key> <value>
ENV <key>=<value> <key>=<value>
```

#### WORKDIR

`WORKDIR` 指令定义工作目录,相当于 `cd`

```Dockerfile
WORKDIR /path/to/workdir
```

#### USER

`USER` 指令指定运行镜像时使用的用户名及可选组

```Dockerfile
USER <user>[:<group>]
USER <UID>[:<GID>]
```

#### VOLUME

`VOLUME` 指定挂载点

```Dockerfile
VOLUME ["/path"]
```

#### ADD

- `ADD` 指令支持拷贝压缩文件到镜像中,并自动解压
- `ADD` 指令支持从源文件来自指定URL,构建容器时会自动下载到指定目录

```Dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

#### COPY

```Dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

#### ENTRYPOINT

`ENTRYPOINT` 指令设置容器启动后要执行的命令.使用 `--entrypoint` 指令进行替换.它有两种形式

`--entrypoint` 指令指定的命令会覆盖原有所有命令及参数

```Dockerfile
# exec 形式,是推荐的形式
ENTRYPOINT ["executable", "param1", "param2"]

# shell 形式. 容器启动后默认执行 `/bin/sh -c command param1 param2`,这种方式启动的容器会自动回收孤儿进程与僵尸进程
ENTRYPOINT command param1 param2
```

#### CMD

`CMD` 指令一般用来设置容器启动后执行命令的默认参数.如果是参数,则必须指定 `ENTRYPOINT` 指令.如果包含多个 `CMD` 指令,以最后一个为准.

`docker run [command]` 时会覆盖 `CMD` 指令内容,对 `ENTRYPOINT` 无影响

`CMD` 指令有 3 种形式

```Dockerfile
# exec 形式,是推荐的形式.该形式不会支持管道或变量替换.容器中 `executable` 进程 ID 为 1
CMD ["executable","param1","param2"]

# shell 形式. 容器会默认使用 `/bin/sh -c command param1 param2` 启动容器,这种方式启动的容器会自动回收孤儿进程与僵尸进程
CMD command param1 param2

# 作为 ENTRYPOINT 的默认参数,此时 ENTRYPOINT 必须使用 exec 形式
CMD ["param1","param2"]
```

下表列出了不同 `ENTRYPOINT` 与 `CMD` 指令在容器启动时运行的命令:

|  | No ENTRYPOINT | `ENTRYPOINT exec_entry p_entry` | `ENTRYPOINT ["exec_entry", "p_entry"]`
:---: | :---: | :---: | :---:
No CMD | error | `/bin/sh -c exec_entry p_entry` | `exec_entry p_entry`
`CMD ["exec_cmd p_cmd"]` | `exec_cmd p_c` | `/bin/sh -c exec_entry p_entry` | `exec_entry p_entry exec_cmd p_cmd`
`CMD ["p1_cmd", "p2_cmd"]` | `p1_cmd p2_cmd` | `/bin/sh -c exec_entry p_entry` | `exec_entry p_entry p1_cmd p2_cmd`
`CMD exec_cmd p_cmd` | `exec_cmd p_cmd` | `/bin/sh -c exec_entry p_entry` | `exec_entry p_entry /bin/sh -c exec_cmd _cmd`

可以在 [GitHub](https://github.com/docker-library/) 上查看官方镜像的 Dockerfile
