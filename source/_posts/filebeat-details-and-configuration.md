---
title: Filebeat 原理及使用
date: 2021-04-03 17:25:23
tags:
  - Filebeat
  - ELK
  - 日志处理
categories:
  - ELK
abbrlink:
description: '本文章主要包含 filebeat 工作原理及常用的使用方式及配置'
---

## Filebeat 工作原理

如[官方文档](https://www.elastic.co/guide/en/beats/filebeat/current/how-filebeat-works.html)所描述的那样,filebeat 包含两个主要的组件,`inputs` 和`harvesters`.

### harvesters 组件

`harvester` 负责读取单个文件的内容.它逐行读取每个文件,将内容发送到 `output`.对于每个文件,filebeat 都会启动一个 harvester,负责打开和关闭文件.默认情况下 filebeat 会使文件处于打开状态,直到达到 `close_inactive` 配置的未读取到内容的超时时长.

> 需要注意的是: 如果 harvester 在读取文件过程中,对该文件删除或重命名,filebeat 会继续读取该文件,直到 harvester 关闭为止.

### inputs 组件

`input` 负责管理 harvesters 并查找所有可读取的输入源文件.

如果 input 类型为 `log`,则 input 将查找与定义的 glob 路径匹配的所有文件,并为每个文件启动 harvester.每个 intput 都在其自己的 goroutine 中运行.示例如下:

```yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/*.log
    - /var/path2/*.log
```

input 支持多种类型,详见[input types](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-filebeat-options.html#filebeat-input-types)

### filebeat 如何保存文件状态

filebeat 会保存每个文件的读取状态,并将相关状态刷新到磁盘中的 `registry file`(注册表文件)中,用于记住 harvester 正在读取的文件的偏移量,并确保发送了所有的日志内容.重新启动 filebeat 时,会使用注册表文件中的数据来重建读取状态,并且 harvester 会在上一次读取到的位置继续进行读取.

由于在 harvester 读取文件过程中,可以重命名或移动文件,因此文件名和路径不足以标识文件.对于每个文件,filebeat 都存储唯一的标识符已检测文件是否被读取过

如果每天创建大量新文件,则可能会发现注册表文件变得太大.有关可以设置为解决此问题的配置选项的详细信息,请参阅[注册表文件太大](https://www.elastic.co/guide/en/beats/filebeat/current/reduce-registry-size.html)相关内容.主要是使用 `clean_removed` 和`clean_inactive` 配置,分别对被忽略的旧文件和删除旧文件的注册表信息进行删除.

### filebeat 如何保证事件至少一次被传递

filebeat 之所以能够实现此行为,是因为它在注册表文件中存储了每个事件的传递状态.在定义的 output 无法访问且尚未确认所有事件的情况下,filebeat 将继续尝试发送事件,直到 output 确认它已经收到事件为止.

重新启动 filebeat 时,将再次发送之前发送到 output 但在未确认的所有事件.这样可以确保每个事件至少发送一次,但是最终可能会将重复的事件发送到 output.您可以通过 `shutdown_timeout` 配置项设置 filebeat 在关闭之前等待的时间.

## filebeat 配置

filebeat 配置文件默认为 `filebeat.yml`.官方给出了不同平台下 filebeat 配置文件的位置.详见[官方文档](https://www.elastic.co/guide/en/beats/filebeat/current/directory-layout.html)

filebeat 支持对 input 等配置进行拆分,并以文件形式包含到主配置文件中

- 可以使用 `filebeat.config.inputs` 将 inputs 相关配置拆分后,包含到主配置文件中.

```yaml
filebeat.config.inputs:
  enabled: true
  path: inputs.d/*.yml
```

- 可以使用 `filebeat.config.modules` 将 modules 相关配置拆分后,包含到主配置文件中.

```yaml
filebeat.config.modules:
  enabled: true
  path: ${path.config}/modules.d/*.yml
```

### inputs

inputs 指定了 filebeat 如何查找和处理输入的数据.配置块为 `filebeat.inputs`.

inputs 支持多种输入类型,常用的主要如下:

- `log`: 从日志文件中读取内容

inputs 包含一些通用的配置,主要如下:

通用配置 | 描述 | 默认值
:--- | :--- | :---
enabled | 是否启用该 input | true
tags | filebeat 为每个事件添加的标签字段列表,主要用于过滤操作 | -
fields | 添加额外的字段列表到 output.添加的额外字段默认会在 `fields` 字段下 | -
fields_under_root | 将额外添加的字段直接添加到 output 中,而不在 `fields` 字段下 | false
processors | 应用于输入数据的处理器列表 | -
keep_null | 是否将值为 null 字段在 output 文档中发布 | false
index | 设置来自此输入事件的索引 | 示例: `%{[agent.name]}-myindex-%{+ yyyy.MM.dd}`替换为 `filebeat-myindex-2019.11.01`

#### log

`log` 类型的 input 用于从日志文件中读取行,它包含如下参数:

- `path`: input 读取文件的路径列表
- `recursive_glob.enabled`: 是否将 `**` 扩展为递归 glob 匹配.最多支持 8 级.默认为 true

> 以下为 `log`,`container` 类型共有的参数

- `encoding`: 用于读取字符数据的文件编码.如: `plain`,`utf-8`,`gbk`.
- `exclude_lines`: 删除与列表中正则表达式匹配的所有行
- `include_lines`: 仅保留与列表中正则表达式匹配的所有行
- `harvester_buffer_size`: harvester 读取文件时的缓冲区大小,单位字节.默认为 16384(16KB)
- `json`: 解码结构为 JSON 消息的日志.
- `exclude_files`: 忽略列表中正则表达式匹配的文件
- `ingore_older`: 忽略在指定时间之前修改的文件
- `close_inactive`: 若在指定时间内文件内容未更新,则关闭文件句柄.一般设置为比日志文件更新频率大的值.默认值为 5m.
- `close_renamed`: 是否在重命名文件后,关闭文件处理程序.默认为 false
- `close_removed`: 是否在文件删除后,关闭文件处理程序.默认为 false
- `close_eof`: 是否在达到文件末尾时,立即关闭文件.默认为 false
- `close_timeout`: 在指定的超时时间后,harvester 关闭文件.若文件仍在更新,可以在 `scan_frequency` 后在次启动新的 harvester.对于释放从磁盘上删除的文件特别有用.默认为 0,表示禁用
- `clean_inactive`: 若在指定时间内文件内容未更新,则删除注册表中文件的状态信息
- `clean_removed`: 是否在重命名文件后,则删除注册表中文件的状态信息.默认为 true.若文件在段时间内消息并再次出现,则将从头开始读取文件内容
- `scan_frequency`: 指定每隔多久扫描一次 path 路径中新文件的频率.默认为 10s
- `scan.sort`: 对扫描到的文件按指定字段进行排序.可选值为使用 `modtime` 修改文件和 `filename` 文件名称进行排序.默认为空值
- `scan.order`: 对扫描到的文件进行升序或降序排序.可选值为 `asc` 升序和 `desc` 降序.
- `tail_files`: 是否从文件末尾开始进行读取文件,默认值为 false
- `symlinks`: 是否除常规文件外还读取符号链接文件.默认为 false.如果符号链接与源文件都在 path 路径下,则可能会发送重复的数据.在处理 Kubernetes 日志文件时很有用.
- `harvester_limit`: 限制一个 input 并行启动 harvester 的数量.默认为 0,表示没有限制.

#### container

`container` 类型的 input 用于从容器日志文件中读取日志文件,它包含如下参数:

- `stream`: 仅从指定的数据流中读取,可选值为 `all`,`stdout` 或 `stderr`.默认值为 `all`.
- `format`: 使用给定格式读取日志文件.`auto`,`docker` 或 `cri`.默认为 `auto`,自动检测.

> 其余参数见 [log](#log)

#### kafka

`kafka` 类型的 input 用于从 Kafka 集群中读取日志文件,主要支持的 kafka 集群版本为 `0.11 - 2.1.0`.它包含如下参数:

- `hosts`: kafka 集群主机列表
- `topics`: 从指定 kafka 主题中读取消息
- `group_id`: kafka 消费者组的 id
- `client_id`: kafka 客户端 id.可选参数
- `version`: kafka 协议版本.默认为 `1.0.0`
- `initial_offset`: 从主题读取的初始位置.可选值为 `oldest` 或 `newest`.默认为 `oldest`.
- `connect_backoff`: 出现错误后,重新连接 kafka 集群的等待时间.默认为 30s.
- `consume_backoff`: 读取失败后,重新读取数据的等待时间.默认为 2s.
- `max_wait_time`: 读取时等待最小输入字节数的时间.默认值为 250ms.
- `wait_close`: 关闭时,等待传输消息并确认的时间.
- `isolation_level`: 设置 kafka 的组隔离级别.默认为 `read_uncommitted`
  - `read_uncommitted`: 返回消息通道中的所有消息
  - `read_committed`: 隐藏作为中止的事务的一部分的消息
- `fatch`: 从 kafka 获取数据的相关设置.`min`,`default`,`max` 分别设置获取数据的最小,默认,最大 bytes 数.默认分别为`1,1MB,0.`
- `rebalance`: kafka 负载均衡设置
  - `strategy`: 负载均衡策略.`range` 遍历或 `roundrobin` 轮询.默认为 `range`
  - `timeout`: 负载均衡的超时时间.默认为 60s
  - `max_retries`: 负载均衡失败后重启多少次.默认为 4
  - `retry_backoff`: 负载均衡失败后要等待多长时间.默认为 2s.

#### redis

`redis`  类型的 input 用于从 redis 慢日志中读取数据.它包含如下参数:

- `hosts`: redis 服务列表
- `password`: 连接 redis 的密码
- `scan_frequency`: 从 redis 慢日志中读取数据的频率.默认为 10s.
- `timeout`: 等待 redis 响应的超时时间.默认为 1s.
- `network`: redis 连接的网络类型.可选值为 `tcp`,`tcp4`,`tcp6`,`unix`.默认值为 `tcp`.
- `maxconn`: redis 客户端的最大连接数.默认为 10.

#### syslog

`syslog` 类型的 input 用于从 TCP,UDP,Unix 套接字中读取事件.它包含如下参数:

参数 | 描述 | `protocol.udp` | `protocol.tcp` | `protocol.unix`
:--- | :--- | :--- | :--- | :---
`max_message_size` | 接收数据的最大大小 | 10KiB | 20MiB | 20MiB
`host` | 监听事件流的主机和端口 | - | - | 不支持
`path` | unix 套节点地址 | 不支持 | 不支持 | -
`socket_type` | unix 套接字类型 | 不支持 | 不支持 | `stream` 或 `datagram`,默认为 `stream`
`group` | unix 套接字的属组 | 不支持 | 不支持 | 运行 filebeat 程序的用户属组
`mode` | unix 套接字的文件模式 | 不支持 | 不支持 | 默认为 `0755`
`read_buffer` | 套接字上读取缓冲区的大小 | - | - | -
`timeout`| 套接字的读写超时时间 | - | - | -
`frame` | 指定用于拆分传入事件的帧 | 不支持 |  - | -
`line_delimiter` | 指定分割传入事件的字符 | 不支持 | \n | \n
`max_connections` | 指定任意时间点的最大连接数 | 不支持 | - | -
`timeout` | 不活动的远程连接关闭之前的秒数 | 不支持 | 300s | 300s
`ssl` | ssl 参数配置 | 不支持 | - | -

#### stdin

`redis`  类型的 input 用于从 stdin 中读取事件,它主要用于测试.

### output

您可以通过设置 `filebeat.yml` 配置文件的 `output` 部分中的参数,将filebeat 中的事件写入到指定 output.filebeat 只能定义一个 output.

#### elasticsearch

`elasticsearch` 类型的 output 将事件输出到 elasticsearch.它包含以下参数:

- `api_key`: 使用 api_key 保护与 elasticsearch 的通信.
- `username`: elasticsearch 用户名
- `password`: elasticsearch 密码
- `parameters`: HTTP 参数字典
- `protocol`: elasticsearch 协议名称,`http` 或 `https`.默认为 `http`
- `path`: HTTP API 调用之前的 HTTP 路径前缀.
- `headers`: 自定义 HTTP 请求头

> 以下为 `elasticsearch`,`logstash` 类型共有的参数

- `enabled`: 是否启用此输出,默认为 `true`
- `hosts`: elasticsearch 的主机列表.
- `compression_level`: 事件的 gzip 压缩级别,`1-9` 压缩级别逐级递增.默认为 0,表示不压缩
- `escape_html`: 是否转义 html 为字符串.默认为 false
- `worker`: 向每个 elasticsearch 实例发送事件的工作程序数,最好在负载均衡模式下使用.默认为 1.
- `proxy_url`: 连接 elasticsearch 的代理 URL
- `index`: 写入事件事的索引名称.默认值为 `filebeat-%{[agent.version]}-%{+yyyy.MM.dd}`
- `indices`: 索引选择器规则的数组.支持条件判断等复杂条件
  - `index`: 索引名称
  - `mappings`: 一个字典,使用 index 返回的值,并将其映射到新名称
  - `default`: 如果 mapping 没有找到匹配项,则使用默认的字符串
  - `when`: 在何时使用此索引.见如下示例

```yaml
# when 条件判断示例
output.elasticsearch:
  hosts: ["http://localhost:9200"]
  indices:
    - index: "warning-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        message: "WARN"
    - index: "error-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        message: "ERR"
# mapping,default 示例
output.elasticsearch:
  hosts: ["http://localhost:9200"]
  indices:
    - index: "%{[fields.log_type]}"
      mappings:
        critical: "sev1"
        normal: "sev2"
      default: "sev3"
```

- `bulk_max_size`: 单个 elasticsearch 批量 API 索引请求中要批量处理的最大事件数.默认为 50.
- `backoff.init`: 网络错误后尝试重新连接到 elasticsearch 的等待时长.
- `backoff.max`: 网络错误后尝试重新连接到 elasticsearch 的最大等待时长.默认为 60s
- `timeout`: elasticsearch 请求的超时时长,默认为 90s.
- `ssl`: SSL 配置相关

#### logstash

`logstash` 类型的 output 将事件输出到 logstash.它包含以下参数:

> 其余参数见 [elasticsearch](#elasticsearch)

- `loadbalance`: 是否启用多个 logstash 的负载均衡.默认为 false,随机分发到主机上
- `ttl`: filebeat 与 logstash 连接的生存时间,之后将重新建立连接.默认为 0,表示禁用此功能

#### kafka

`kafka` 类型的 output 将事件输出到 kafka.它包含以下参数:

- `enabled`: 是否启用此 output
- `hosts`: kafka 集群主机列表
- `topic`: 从指定 kafka 主题中读取消息
- `topics`: topic 选择器规则的数组.支持条件判断等复杂条件
  - `topic`: topic 名称
  - `mappings`: 一个字典,使用 topic 返回的值,并将其映射到新名称
  - `default`: 如果 mapping 没有找到匹配项,则使用默认的字符串
  - `when`: 在何时使用此 topic.见如下示例

```yaml
output.kafka:
  hosts: ["localhost:9092"]
  topic: "logs-%{[agent.version]}"
  topics:
    - topic: "critical-%{[agent.version]}"
      when.contains:
        message: "CRITICAL"
    - topic: "error-%{[agent.version]}"
      when.contains:
        message: "ERR"
```

- `version`: kafka 协议版本.默认为 `1.0.0`
- `username`: 连接 kafka 的用户名
- `password`: 连接 kafka 的密码
- `client_id`: kafka 客户端 id.可选参数
- `worker`: 向每个 kafka 实例发送事件的工作程序数,最好在负载均衡模式下使用.默认为 1.
- `codec`: 编码格式.默认为 json
- `metadata`: kafka 元数据信息设置.
- `bulk_max_size`: 单个 kafka 请求中要批量处理的最大事件数.默认值为 2048.
- `bulk_flush_frequency`: 向 kafka 发送请求之前需要等待的时间.默认为 0.表示没有延迟.
- `timeout`: kafka 响应超时时间.默认值为 30s
- `channel_buffer_size`: kafka 在 output 管道中缓冲的消息数.
- `keep_alive`: 活动连接的保持活动时长.默认为 0.表示禁用连接.
- `compression`: 压缩编码方式.可选值为 `none`,`snappy`,`lz4` 或 `gzip`.默认值为 `gzip`.
- `compression_level`: 压缩级别,0-9 依次升高.默认为 4.
- `max_message_bytes`: json 消息的最大大小.默认为 100 万字节

#### redis

`redis`  类型的 output 用于将事件输出到 Redis list 或 channel 中.此插件与 logstash  的 redis 输入插件兼容.它包含如下参数:

- `enabled`: 是否启用此插件
- `hosts`: redis 服务列表
- `password`: 连接 redis 的密码
- `db`: 连接 redis 的数据库.默认为 0.
- `key`: 发送事件到 redis 的 list 或 channel 的 key 的名称.若为空,则使用 `index` 设置的值.
- `index`: 添加到事件元数据中的索引名称,以供 logstash 使用.默认值为 `filebeat`.
- `keys`: key 选择器的规则的数组.支持条件判断等复杂条件
  - `key`: key 名称
  - `mappings`: 一个字典,使用 key 返回的值,并将其映射到新名称
  - `default`: 如果 mapping 没有找到匹配项,则使用默认的字符串
  - `when`: 在何时使用此 key.见如下示例

```yaml
output.redis:
  hosts: ["localhost"]
  key: "default_list"
  keys:
    - key: "info_list"   # send to info_list if `message` field contains INFO
      when.contains:
        message: "INFO"
    - key: "debug_list"  # send to debug_list if `message` field contains DEBUG
      when.contains:
        message: "DEBUG"
    - key: "%{[fields.list]}"
      mappings:
        http: "frontend_list"
        nginx: "frontend_list"
        mysql: "backend_list"
```

- `datatype`: 用于发布事件的 redis 数据类型.可选值为 `list`(使用 `rpush` 命令)或 `channel`(使用 `publish` 命令).默认为 `list`.
- `codec`: 发送到 redis 数据的编码格式.默认为 `json`
- `worker`: redis 连接的网络类型.可选值为 `tcp`,`tcp4`,`tcp6`,`unix`.默认值为 `tcp`.
- `loadbalance`: redis 客户端的最大连接数.默认为 10.
- `timeout`: 连接 redis 的超时时间.默认为 5s.
- `backoff.init`: 网络错误后尝试重新连接到 redis 的等待时长.默认为 1s.
- `backoff.max`: 网络错误后尝试重新连接到 redis 的最大等待时长.默认为 60s.
- `max_retries`: 设置重试次数
- `bulk_max_size`: 单个 redis 请求或 pipeline 批量发送数据最大数据大小.默认为 2048.

#### file

`file`  类型的 output 用于将事件输出到文件中.每个事件采用 json 格式,主要用于测试.它包含如下参数:

- `enabled`: 是否启用此插件.默认为 true
- `path`: 文件位置.必选参数
- `filename`: 生成文件名称.默认设置为 Beat 的名称.如 `filebeat` 生成文件将是 `filebeat.1`
- `rotate_every_kb`: 每个文件的最大大小,以 kb 为单位.超过此大小后,文件会自动滚动.默认值为 1024 kb.
- `number_of_files`: `path` 下保存的最大文件数.最早的文件将被删除.默认为 7.
- `permissions`: 指定创建文件的权限.默认值为 `0600`.
- `codec`: 输出事件的编码格式.默认为 json.

#### console

`console` 类型的 output 用于将事件输出到标准输出.每个事件采用 json 格式,主要用于测试.它包含如下参数:

- `pretty`: 是否以漂亮格式输出.默认为 false
- `enabled`: 是否启用此插件.默认为 true
- `bulk_max_size`: 输出缓冲区的最大事件数.默认为 1024.
- `codec`: 输出的编码格式

### Logging

配置文件 `filebeat.yml` 的 `logging` 部分包含用于配置日志输出的选项.日志系统可将日志写入 syslog 或滚动日志文件中.如果未显式配置,则使用滚动文件输出日志.它主要包含如下配置项:

- `logging.to_stderr`: 如果为 true,则将所有日志写入到标准输出中.与使用 `-e` 选项相同.
- `logging.to_syslog`: 如果为 true,则将所有日志写入到 syslog 中.
- `logging.to_files`: 如果为 true,则将所有日志写入到滚动日志文件中,在日志文件大小到达一定大小后进行滚动
- `logging.level`: 设置日志级别.可选值为 `debug`,`info`,`warning`,`error`.默认值为 `info`
- `logging.metrics.enabled`: 是否定期记录其内部数据指标.
- `logging.metrics.period`: 日志记录内部指标的时间间隔.默认为 30s.
- `logging.files.path`: 日志文件写入的目录.默认值为 `logs`
- `logging.files.name`: 日志的文件名,默认值为 `filebeat`.
- `logging.files.rotateeverybytes`: 日志文件的最大大小.默认值为 10485760 (10 MB).
- `logging.files.keepfiles`: 日志保存的文件个数.默认值为 7.
- `logging.files.permissions`: 滚动日志时要使用的权限掩码.默认值为 0600.
- `logging.files.interval`: 除了基于大小的滚动外,filebeat 日志还支持基于日期的文件滚动.默认是禁用的
- `logging.files.rotateonstartup`: 如果日志文件启动时已存在,则立即滚动并写入新的文件中.
- `logging.json`: 是否以 json 格式写入文件.默认为 false.

### HTTP endpoint

- `http.enabled`: 是否启用 HTTP endpoint.默认为 false
- `http.host`: 绑定地址,支持主机名,IP 地址, Unix 套接字.默认为 `localhost`
- `http.port`: http endpoint 监听端口.默认为 `5066`

```bash
# /stats 暴露了内部指标,如下
curl -XGET 'localhost:5066/stats?pretty'

{
  "beat": {
    # ...
  }
}
```

### filebeat.reference.yml

参见[filebeat.reference.yml](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-reference-yml.html)
