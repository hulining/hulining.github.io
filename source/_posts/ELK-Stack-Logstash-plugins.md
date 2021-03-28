---
title: ELK 之 Logstash 插件介绍
date: 2020/05/22
tags:
  - ELK
  - 日志处理
categories:
  - ELK
abbrlink: 53890
description: '本文章主要包含 ELK Logstash 常用的输入,过滤,输出插件及其相关字段的解释以及一些插件的简单使用示例.'
---

Logstash 是高度插件化的日志数据收集工具,常用插件包括 `input`, `filter`, `output`

Logstash 的工作流程 `input > filter > output`.如无需要对数据进行额外处理,则 `filter` 可省略

## `input` 插件

[`input`](https://www.elastic.co/guide/en/logstash/current/input-plugins.html) 插件用于设置 Logstash 获取数据的数据源.

它包含一些通用的字段,用于配置进入 Logstash 数据的属性,用于后续的处理过程.常用字段如下:

字段 | 描述
:---:| :---:
add_field | 为数据内容添加字段键值对映射
id | 设置 input 插件的唯一标识
tags | 设置数据属性标签数组
type | 设置数据类型

### `beats`

[`beats`](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-beats.html) 输入插件用于从 Beats 中接收数据

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
port | Logstash 监听的端口|  是 | -
host | Logstash 监听的地址 | 否 | "0.0.0.0"
ssl | 是否启用 ssl 加密相关 | 否 | false
ssl_certificate | ssl 证书 | 否 | ""
ssl_key | ssl 密钥文件 | 否 | ""

> 示例

```ruby
input {
  beats {
    host => "0.0.0.0"
    port => 5044
    ssl => true
    ssl_certificate => "/path/to/ssl/cert"
    ssl_key => "/path/to/ssl/key"
  }
}
```

### `file`

[`file`](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-file.html) 输入插件用于读取文件数据.它使用一个名为 sincedb 的单独文件中来跟踪每个文件中的当前位置,这样就可以停止或重新启动 Logstash,并使它从中断处开始读取,而不会丢失 Logstash 停止时读取数据的位置.

- 支持多文件读取
- 支持 `tail` 模式(没有 EOF)或直接读取 `read` 模式(有EOF)

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
path | 指定要读取的文件列表或模式匹配 | 是 | -
close_older | 指定多少秒无新数据则关闭文件,检测到新数后重新打开 | 否 | "1 hour"
delimiter | 换行符 | 否 | "\n"
exclude | 当 path 为模式匹配时,排除读取指定文件 | 否 | []
file_chunk_count | 读取文件块数量 | 否 | 4611686018427387903
file_chunk_size | 读取文件块大小 | 否 | 32KB
file_completed_action | 读取文件 EOF 后的操作,delete, log, log_and_delete | 否 | "delete"
file_completed_log_path | 读取文件 EOF 后记录的文件位置 | 否 | ""
file_sort_direction | 排序方式,asc,desc | 否 | "asc"
ignore_older | 忽略指定修改时间前的文件 | 否 | ""
mode | 指定文件的读取模式, tail, read | 否 | "tail"
sincedb_path | 指定保存 sincedb 文件的目录,默认会在 `<path.data>/plugins/inputs/file/` | 否 | ""
start_position | 指定从何处开始读取,start, end | 否 | "end"
stat_interval | 指定读取时间间隔 | 否 | "1 second"

> 示例

- 以 `read` 模式读取昨天新产生的文件,而不读取 "*.gz" 文件

```ruby
input {
  file {
    path => ["/path/to/nginx-80-*.log","/path/to/nginx-443-*.log"]
    exclude => "*.gz"
    mode => "read"
    file_sort_direction => "desc"
    ignore_older => "1d"
    start_position => "beginning"
  }
}
```

- 以 `tail` 模式实时读取文件

```ruby
input {
  file {
    path => ["/path/to/nginx-80.log","/path/to/nginx-443.log"]
    mode => "tail"
    stat_interval => "5 second"
    close_older => "5"
    start_position => "end"
  }
}
```

### `kafka`

[kafka](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html) 输入插件用于从 Kafka 集群中读取事件数据.

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
bootstrap_servers | kafka 实例的列表 | 否 | "localhost:9092"
client_id | 客户端标识 | 否 | "logstash"
connections_max_idle_ms | 空闲连接释放的超时时间 | 否 | -
consumer_threads | 消费者线程数,一般配置为 kafka 的分区数 | 否 | 1
decorate_events | 是否在事件中添加 kafka 元数据,如 topic,消息大小 | 否 | false
group_id | kafka 消费者所属组的 id | 否 | "logstash"
topics | 订阅的 topic | 否 | ["logstash"]
topics_pattern | 以正则表达式方式订阅 topic | 否 | -

> 示例

```ruby
input {
  kafka {
    bootstrap_servers => ["10.237.64.46:9094"]
    group_id => "es-transfer"
    topics => ["JSON_PRODUCE_TOPIC"]
    consumer_threads => 5
    decorate_events => true
    codec => "json"
  }
}
```

### `redis`

[`redis`](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-redis.html) 输入插件用于从 Redis 实例中读取事件数据,支持 Redis channels 和 lists.

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
host | Redis 数据库主机地址 | 否 | "127.0.0.1"
port | Redis 数据库监听端口 | 否 | 6379
passwd | Redis 数据库连接密码 | 否 | ""
db | Redis 的数据库 | 否 | 0
data_type | 指定从 Redis 指定对象中读取数据, list, channel, pattern_channel | 是 | -
key | Redis list 或 channel 的名称 | 是 | -
batch_count | 从 Redis 返回的事件数 | 否 | 125

> 示例

```ruby
input {
  redis {
    host => "192.168.1.2"
    port => 6379
    passwd => "passwd"
    db => 0
    data_type => "list"
    key => "logstash"
  }
}
```

### `stdin`

[`stdin`](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-stdin.html) 输入插件从标准输入读取数据,多用于测试

## `filter` 插件

[`filter`](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html) 插件用于过滤或处理进入 Logstash 的数据.

它包含一些通用的字段,用于配置从 Logstash 输出数据的属性,用于后续的处理过程.常用字段如下

字段 | 描述
:---: | :---:
add_field | 为数据内容添加字段键值对映射
add_tag | 为数据属性添加标签属性
id | 设置 filter 插件的唯一标识
remove_field | 删除数据内容中指定字段
remove_tag | 删除数据属性中指定标签

### `date`

[`date`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html) 过滤插件用于解析字段中的日期,然后使用该日期或时间戳作为时间戳

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
match | 指定字段及日期格式 | 否 | []
tag_on_failure | 当解析失败时添加列表中标签 | 否 | ["_dateparsefailure"]
target | 解析成功后保存的字段名 | 否 | "@timestamp"

> 示例

```ruby
filter {
  date {
    # log_time 变量包含 dd/MMM/yyyy:HH:mm:ss Z 格式的日期
    match => ["log_time","dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@log_time"
    tag_on_failure => ["_dateparsefailure"]
  }
}
```

### `drop`

[`drop`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-drop.html) 过滤插件用于删除事件数据

```ruby
filter {
  if [loglevel] == "debug" {
    drop { }
  }
}
```

### `geoip`

[`geoip`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-geoip.html) 过滤插件用于根据来自 Maxmind GeoLite2 数据库的数据添加有关 IP 地址地理位置的信息

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
database | 指定 Maxmind GeoLite2 数据库地址 | 否 | ""
source | 指定要解析的 IP 地址或主机名 | 是 | -
target | 指定解析后数据存储到的字段名称 | 否 | "geoip"
fields | 指定解析后数据要保留的字段列表 | 否 | []
tag_on_failure | 当解析失败时添加列表中标签 | 否 | ["_geoip_lookup_failure"]

> 示例

```ruby
filter {
  geoip {
    # remote_addr 变量表示 IP 地址,用于对
    source => "remote_addr"
    target => "geoip"
    database => "/home/elk/logstash-6.4.2/config/GeoLite2-City.mmdb"
    fields => ["city_name","country_name","region_name","location"]
    tag_on_failure => ["_geoip_lookup_failure"]
  }
}
```

### `grok`

[`grok`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) 过滤插件基于正则表达式解析任意文本并将其结构化,是将非结构化日志数据解析为结构化和可查询内容的好方法.

`grok` 默认包含 120 种的模式,您可以在 [Github](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns) 或 `${LOGSTASH_HOME}/vendor/bundle/jruby/2.3.0/gems/logstash-patterns-core-4.1.2/patterns/` 找到对应的正则表达式

`grok` 的基本语法为 `%{SYNTAX:SEMANTIC}`.`SYNTAX` 是指定用于匹配文本的模式名称,正则表达式的内容可以在如上位置找到.`SEMANTIC` 为匹配到的文本提供标识符,将匹配到的文本写入到变量中,后续可以直接使用此标识符引用文本.

另外,我们可以按照 `PATTERN_NAME PATTERN` 的方式自定义匹配模式,并在 `pattern_definitions` 选项字段中进行引用.或将匹配模式定义写入到文件中,通过 `pattern_dir` 和 `patterns_files_glob` 字段对该文件进行引用.

可以在 [http://grokdebug.herokuapp.com/](http://grokdebug.herokuapp.com/) 查看文本与模式是否匹配.

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
match | 定义了待匹配的文本和匹配模式的映射,匹配模式可以以列表形式定义多个 | 否 | {}
overwrite | 指定模式匹配后的字段覆盖列表中的字段 | 否 | []
pattern_definitions | 自定义模式的名称和内容的映射 | 否 | {}
patterns_dir | 自定义模式文件的目录 | 否 | []
patterns_files_glob | 使用 glob 匹配 patterns_dir 目录下文件用于自定义模式匹配 | 否 | "*"
tag_on_failure | 匹配失败后添加的标签 | 否 | ["_grokparsefailure"]
tag_on_timeout | 匹配超时时添加的标签 | 否 | "_groktimeout"
target | 定义匹配到的文本保存的字段名称,类似于 `geoip.target` | 否 | ""
break_on_match | 是否在首次匹配成功后则跳出匹配,如果想让尝试所有的模式匹配,则设置为 false | 否 | true

> 示例

```ruby
filter {
  grok {
    # "message" 为 nginx 数据
    match => {
      "message" => "\[%{HTTPDATE:log_time}\] - %{NUMBER:timestamp} - (%{IP_X:remote_addr}|-) - %{USER:remote_user} \"%{WORD:request_method} %{URIHOST:server_host}(%{URIPATH:request_uri}|-)(%{NOTSPACE:request_param})? HTTP\/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:response_body_bytes} - %{DATA:ssl_protocol} %{NUMBER:request_time} - (%{HOSTPORT:upstream_addr}|-) (%{NUMBER:upstream_response_time}|-) %{GREEDYDATA:other}"
    }
    remove_field => [ "@version", "host", "path", "type", "message", "remote_user", "request_method", "http_version", "request_param", "ssl_protocol", "request_time", "upstream_addr", "upstream_response_time", "other"]
  }
}
```

```text
[19/May/2020:16:40:04 +0800] - 1589877604.781 - 42.236.82.156 - - "GET reg.cntv.cn/ HTTP/1.1" 302 0 - - 0.003 - 192.168.2.3:8082 0.003 - 100148273 - 1 - Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36 "-"
```

### `json`

[`json`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-json.html) 过滤插件将 JSON 格式的数据解析为 Logstash 内部格式的数据

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
source | 指定要解析的字段 | 是 | -
target | 解析后数据存储到的字段名称 | 否 | ""
tag_on_failure | 解码失败后添加的标签 | 否 | ["_jsonparsefailure"]
skip_on_invalid_json | 是否跳过无效的 JSON  | 否 | false

> 示例

```ruby
filter {
  json {
    # "message 变量为 json 格式数据
    source => "message"
    skip_on_invalid_json => "true"
    target => "doc"
  }
}
```

```text
{"method":["user.getName"],"client":["client"],"snap":["120x120"],"userid":["12345678"]}
解析为

{
  "doc" => {
    "snap" => [
      [0] "120x120"
    ],
    "client" => [
      [0] "client"
    ],
    "userid" => [
      [0] "12345678"
    ],
    "method" => [
      [0] "user.getName"
    ]
}
```

### `urldecode`

[`urldecode`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-urldecode.html) 过滤插件用于对 urlencoded 的字段解码

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
field | 指定要解码的字段 | 否 | "message"
charset | 指定字符编码 | 否 | "UTF-8"
tag_on_failure | 解码失败后添加的标签 | 否 | ["_urldecodefailure"]
all_field | 是否对所有字段进行解码 | 否 | false

> 示例

```ruby
filter {
  urldecode {
    # request_url 包含已经编码或未编码的信息
    field => "request_url"
    charset => "UTF-8"
    tag_on_failure => ["_urldecodefailure"]
  }
}
```

```text
"client=6&data=%7B%22uid%22%3A73023332%2C%22vid%22%3A%2264d68733bd5343d2bc69334e804ce036%22%2C%22position%22%3A%220%22%7D&method=videoformobile.setVideoPosition"

解析为

"client=6&data={\"uid\":73023332,\"vid\":\"64d68733bd5343d2bc69334e804ce036\",\"position\":\"0\"}&method=videoformobile.setVideoPosition\""
```

## `output` 插件

[`output`](https://www.elastic.co/guide/en/logstash/current/output-plugins.html)插件将事件数据发送到指定的目的地,是事件管道中的最后阶段.

它包含一些通用的字段,用于配置 Logstash 输出数据的属性.常用字段如下:

字段 | 描述
:---: | :---:
codec | 指定输出数据的解码器.默认值为 "rubydebug",可选做 "json"
id | 设置 output 插件的唯一标识

### `elasticsearch`

[`elasticsearch`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html) 输出插件将 Logstash 数据导入到 Elasticsearch.

此插件尝试通过单个请求发送批量事件数据.如果一个请求超过 `20MB`,我们会将其分解为多个批处理请求.如果单个文档超过`20MB`,它将作为单个请求发送.

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
hosts | 指定 Elasticsearch 服务的地址,支持指定多个协议和端口 | 否 | ["127.0.0.1:9200"]
user, password | 指定 Elasticsearch 服务的用户名密码 | 否 | ""
index | 指定索引 | 否 | "logstash-%{+yyyy.MM.dd}"
action | Elasticsearch 内部执行的操作,index,delete,create,update | 否 | "index"
document_id | 指定文档索引,会自动生成 | 否 ""
ssl | 是否使用 ssl | 否 | false
cacert | 证书文件路径 | 否 | ""

### `mongodb`

[`mongodb`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-mongodb.html) 输出插件将事件数据写入到 MongoDB.

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
bulk | 是否启用批量插入 | 否 | false
bulk_interval | 批量插入的时间间隔 | 否 | 2
bulk_size | 批量插入的事件数据数 | 否 | 900
collection | 数据写入 MongoDB 的集合名称,支持动态选择 | 是 | -
database | 数据写入 MongoDB 的数据库名称 | 是 | -
uri | 指定连接 MongoDB 的地址 | 是 | -
generateId | 是否生成 "_id" 字段插入到 MongoDB 文档中.如果设置为 true,则使用事件数据的时间戳,且覆盖现有 "_id" 字段 | 否 | false
retry_delay | 失败后重试的等待时间 | 否 | 3
isodate | 是否将 @timestamp 作为 ISODate 类型保存在 MongoDB 中 | 否 | false

### `redis`

[`redis`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-redis.html) 输出插件将事件数据写入到 Redis.

> 常用配置字段

字段 | 描述 | 是否必需 | 默认值
:---: | :---: | :---: | :---:
host | Redis 数据库主机地址 | 否 | ["127.0.0.1"]
port | Redis 数据库监听端口 | 否 | 6379
passwd | Redis 数据库连接密码 | 否 | ""
db | Redis 的数据库 | 否 | 0
data_type | 指定向 Redis 写入数据的对象, list, channel | 否 | ""
key | Redis list 或 channel 的名称,可以使用动态名称 | 否 | ""
batch | 是否使用 RPUSH 向 Redis list 中批量写入数据 | 否 | false
batch_events | RPUSH 默认向 Redis list 中写入的数据量 | 否 | 50
batch_timeout | RPUSH 的超时时间 | 否 | 5
reconnect_interval | 重试连接的时间间隔 | 否 | 1
timeout | 连接的超时时间 | 否 | 5

> 示例

```ruby

```

### `stdout`

[`stdout`](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-stdout.html) 输出插件将事件数据打印到标准输出,多用于测试.
