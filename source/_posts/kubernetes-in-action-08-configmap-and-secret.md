---
title: K8S 实战 - 08 ConfigMap 和 Secret
date: 2021/03/19 08:00:00
tags:
  - Kubernetes
categories:
  - Kubernetes
description: 本文为 K8S 实战的第八个章节.主要了解 K8S 集群中 ConfigMap 和 Secret 相关概念及使用方式.
---

ConfigMap 和 Secret 是 Kubernetes 系统上两种特殊类型的存储卷,ConfigMap 对象用于为容器中应用提供配置数据,Secret 用于保存敏感的配置信息,如密钥,证书等.它们将相应的配置信息保存至对象中,而后在 Pod 资源上以存储卷的形式将其挂载并获取相关配置.

## 容器化应用的配置方式

为容器中应用提供配置信息主要有如下几种方式:

- 通过命令行传递参数

在制作 Docker 镜像时,Dockerfile 中的 `ENTRYPOINT` 和 `CMD` 指令可用于指定容器启动时要运行的程序及相关参数.

在基于某镜像使用 `docker` 相关命令创建容器时,可以使用 `docker run IMAGE [COMMAND] [ARGS]` 在命令行向容器传入自定义参数

在 Kubernetes 系统创建 Pod 资源时,可以使用 `spec.containers` 下的 `command` 和 `args` 向容器传入命令和参数

- 将配置文件嵌入镜像文件

在 Dockerfile 中使用 `COPY` 指令将定义好的配置文件复制到镜像文件系统上的目标位置.但配置文件相关的任何修改都不得不重新构建镜像文件来实现

- 通过环境变量向容器注入配置信息

用户在使用 `docker run` 命令启动容器时,通过 `-e` 选项可以像容器传递环境变量.一般来说容器的 `ENTRYPOINT` 脚本会为这些环境变量提供默认值,保证用户在未传入环境变量时能够基于默认值启动容器

Kubernetes 创建 Pod 资源时,可以使用 `spec.containers.env` 和 `spec.containers.envFrom` 为容器的环境变量传值.

- 通过存储卷向容器注入配置信息

Docker 存储卷能够将宿主机上的任何文件或目录映射到容器文件系统上,在启动容器时进行加载.

Kubernetes 中的 `ConfigMap` 和 `Secret` 资源对象,要么被 Pod 资源以存储卷的形式加载,要么由容器通过 `envFrom` 字段以变量形式加载.

### 通过命令行参数配置容器应用

创建 Pod 资源时,可以在容器定义中自定义要运行的命令以及为其传递的选项和参数.在容器的配置上下文中,使用 `command` 字段指定要运行的程序,`args` 字段指定传递给程序的选项及参数.这类程序会被直接执行,而不会由 shell 解析器运行,因此不支持命令展开 `{}`,重定向等操作.

可以简单理解为,Kubernetes 配置清单文件中的 `command` 对应于 Dockerfile 中 `ENTRYPOINT`,`args` 对应于 `CMD`.

### 利用环境变量配置容器应用

在 Kubernetes 中启动创建 Pod 资源时,可以使用 `spec.containers.env` 参数来定义所使用的环境变量列表.

环境变量通常由 `name` 和 `value(或 valueFrom)` 字段构成.定义方式详见 {% post_link kubernetes-in-action-04-manage-pod 'K8S 实战 - 04 Pod 管理' %} 最后.

## 应用程序配置管理及 ConfigMap 资源

Kubernetes 基于 ConfigMap 对象实现了将配置文件从容器镜像中解耦,从而增强了容器应用的可移植性.ConfigMap 对象将配置数据以键值对的形式进行存储,用户可以在不同环境中创建名称相同但内容不同的 ConfigMap 对象,从而为不同环境中同一功能的 Pod 资源提供不同的配置信息,实现应用与配置的灵活勾兑.

### 创建 ConfigMap 对象

ConfigMap 资源对象可以使用清单创建,也可以通过 `kubectl create configmap` 命令创建,命令语法如下:

```bash
kubectl create configmap <map-name> <datasource>
```

其中数据源可以通过目录,文件或直接值来获取,并转换为 key-value 后创建 ConfigMap 对象

#### 使用键值对直接创建

为 `kubectl create configmap` 命令使用 `--from-literal` 选项可以在命令行直接给出键值对创建 ConfigMap 对象.该选项可以重复使用以传递多个键值对.

#### 基于配置文件创建

为 `kubectl create configmap` 命令使用 `--from-file` 选项可以基于文件内容创建 ConfigMap 对象.该选项可以重复使用以传递多个文件.

这种方式创建的 ConfigMap 对象,其存储的数据键为文件名,值为文件内容

#### 基于目录创建

与基于文件创建方式一样,`--from-file` 支持传入路径,这种方式创建的 ConfigMap 对象,会将该路径下所有文件一同创建于同一 ConfigMap 资源中.其存储的数据键为指定目录下文件名,值为对应文件内容

#### 使用清单创建

一般创建方式如下:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name:
  namespace:
data:
  # 存储键值对数据
```

### 在 Pod 中使用 ConfigMap 资源对象

- 通过在 Pod 清单文件中使用 `spec.containers.env.valueFrom.configMapKeyRef` 将 ConfigMap 对象中键值数据传递到 Pod 中作为环境变量,Pod 中环境变量的引用方式为 `$(VAR_NAME)`
- 通过在 Pod 清单文件中使用 `spec.volumes.configMap` 将 configMap 作为存储卷挂载到 Pod 中,Pod 中挂载存储卷的字段为 `spec.containers.volumeMounts`

清单文件详见 {% post_link kubernetes-in-action-04-manage-pod 'K8S 实战 - 04 Pod 管理' %} 最后.

### 注意事项

ConfigMap 资源支持容器应用动态更新配置,用户直接更新 ConfigMap 对象,而后由容器应用重载其配置文件即可

在 Pod 资源中调用 ConfigMap 对象时需要注意:

- 以存储卷方式引用的 ConfigMap 对象必须先于 Pod 存在,否则将会导致 Pod 无法正常启动的错误
- 当以环境变量方式注入 ConfigMap 中键不存在时会被忽略,Pod 会正常启动,但日志会记录 "InvalidVariableNames" 错误引用信息
- ConfigMap 是名称空间级别的资源,引用它的 Pod 必须处于同一名称空间中

## Secret 资源

Secret 资源的功能类似于 ConfigMap,以键值方式存储数据,在 Pod 资源中通过环境变量或存储卷进行数据访问.不同的是,Secret 对象仅会被分发至调用了此对象的 Pod 资源所在的工作节点,且只能由节点将其存储在内存中;Secret 对象的数据存储及打印格式为 Base64 编码的字符串(可通过 `base64 --decode` 进行解码),因此用户在创建 Secret 数据时也要提供此种编码格式的数据.不过,在容器中以环境变量或存储卷的方式进行访问时,它们会被自动解码为明文格式

Secret 资源主要分为4种,如下:

- `opaque`: 自定义数据内容,base64 编码,存储密码,密钥,证书等数据.类型标识为 `generic`
- `kubernetes.io/service-account-token`: Service Accoount 的认证信息,在创建 Service Account 时自动创建
- `kubernetes.io/dockerconfigjson`: 存储 Docker镜像仓库的认证信息,类型标识为 `docker-registry`
- `kubernetes.io/tls`: 用于为 SSL 通信模式存储证书和私钥文件,类型标识为 `tls`

## 创建 Secret 资源

### 创建 `generic` 类型的 Secret 资源

#### 基于直接值创建

为 `kubectl create secret generic` 命令使用 `--from-literal` 选项可以在命令行直接给出键值对创建键值对形式的 Secret 对象.该选项可以重复使用以传递多个键值对.

#### 基于文件创建

为 `kubectl create secret generic` 命令使用 `--from-file` 选项可以基于文件内容创建 Secret 对象.该选项可以重复使用以传递多个文件.

### 创建 `tls` 类型的 Secret 资源

### 基于私钥和证书文件创建用于 SSL/TLS 通信的 Secret 对象

使用如下命令可以基于私钥文件和证书文件创建用于 SSL/TLS 通信的 Secret 对象

```bash
kubectl create secret tls <secret_name> --key=<key_file> --cert=<crt_file>
```

示例如下:

```bash
(umask 077;openssl genrsa -out nginx.key 2048)
openssl req -new -x509 -key nginx.key -out nginx.crt -subj /C=CN/ST=BeiJing/L=BeiJing/O=DevOps/CN=<hostname.com>
kubectl create secret tls <sceret_name> --key=nginx.key --cert=nginx.crt
```

### 创建 `docker-registry` 类型的 Secret 资源

使用如下命令可以创建 `docker-registry` 类型的 Secret 对象,该对象可以用作 `spec.containers.imagePullSecret` 的值

```bash
kubectl create secret docker registry <secret_name> \
  --docker-user=<username> --docker-password=<password> \
  ---docker-server=https://index.docker.io/v1/
```
