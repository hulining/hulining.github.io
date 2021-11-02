---
title: logrus 快速开始
date: 2021-11-02 11:11:03
tags:
  - go
  - 3rd
categories:
  - go
description: logrus 日志库快速开始,了解该第三方库的常用使用方式
---

我们日常使用 go 提供的 `log` 标准库可能过于简单了.因此有了很多基于标准库二次开发的第三方日志库.[`logrus`](https://github.com/sirupsen/logrus) 就是其中一个.`logrus` 完全兼容标准的`log` 库,还支持文本、JSON 两种日志输出格式.很多知名的开源项目都使用了这个库,如 [`docker`](https://github.com/moby/moby), [`etcd`](https://github.com/etcd-io/etcd) 等.

## 概述

`logrus` 的使用非常简单,与标准库log类似.但 `logrus` 支持更多的日志级别: `Trace`,`Debug`,`Info`,`Warn`,`Error`,`Fatal`,`Panic`.默认为`Info`. 

```go
package main

import (
	"github.com/sirupsen/logrus"
)

func main() {
	logrus.Debug("debug msg")
	logrus.Info("info msg")
	logrus.Warn("warn msg")
	logrus.Error("error msg")
}

/** 输出如下:
INFO[0000] info msg
WARN[0000] warn msg
ERRO[0000] error msg
FATA[0000] fatal msg
exit status 1
 */
```
可以看到输出内容很简陋.因此我们可以对 `logrus`进行定制.

## 常用函数

`logrus` 提供了众多函数对日志的输出格式进行定制.如:

- 日志的输出级别
- 日志使用文本格式还是 JSON 格式
- 日志带有时间戳
- 日志输出到指定文件中或直接输出到日志处理工具中

```go
// 我们可以自行调用 New 或 StandardLogger 方法,产生一个默认的 Logger 对象.以下所有包函数都是通过调用此对象的方法来实现的.New 方法见如下实现
func New() *Logger
func StandardLogger() *Logger

// 设置日志格式化输出的格式
func SetFormatter(formatter Formatter)
// 设置日志的默认级别
func SetLevel(level Level)
// 设置日志的输出位置
func SetOutput(out io.Writer)
// 设置日志是否记录调用的位置,此函数会显示日志调用的位置,多用于调试
func SetReportCaller(include bool)

// 支持在日志的前面添加一些字段.返回 Entry 对象,该对象是实际产生日志的对象.
func WithField(key string, value interface{}) *Entry
func WithFields(fields Fields) *Entry
```

以上大多都是直接调用包函数来完成相关配置的.其实在 `logrus` 包中有一个默认的 `logrus.Logger` 对象.所有其他的包函数都是调用其相关方法实现的.其定义方式如下:

```go
// https://github.com/sirupsen/logrus/blob/v1.8.1/logger.go#L84
func New() *Logger {
	return &Logger{
		// 输出到 stderr,日志格式为 TextFormatter,日志级别为 info,不记录调用的位置等
		Out:          os.Stderr,
		Formatter:    new(TextFormatter),
		Hooks:        make(LevelHooks),
		Level:        InfoLevel,
		ExitFunc:     os.Exit,
		ReportCaller: false,
	}
}
```

### 日志格式

`logrus` 提供了 `JSONFormatter` 和 `TextFormatter` 来分别实现日志 JSON 和 文本 格式的输出.它们都实现了 `Formatter` 接口.因此,后续我们也可以自己编写对应的 `Formatter` 对象

`JSONFormatter` 和 `TextFormatter` 中都定义了日志输出的各种参数.可以通过配置这些成员变量来配置日志的格式.如下:

```go
TextFormatter{
	// 是否显示日志时间
	FullTimestamp: true,
	// 日志时间格式
	TimestampFormat: "2006-01-02 15:04:05",
}
```

### 日志输出

同时,对于输出位置来说,我们可以使用  `SetOutput` 函数进行修改,将日志输出到文件中.如下

```
import (
  "bytes"
  "io"
  "log"
  "os"

  "github.com/sirupsen/logrus"
)

func main() {
  stdout := os.Stdout
  logfile, err := os.OpenFile("log.txt", os.O_WRONLY|os.O_CREATE, 0755)
  if err != nil {
    log.Fatalf("create file log.txt failed: %v", err)
  }

  logrus.SetOutput(io.MultiWriter(stdout, logfile))
  logrus.Info("info msg")
}
```

### 添加字段

有时,我们会添加一字段到日志中

```
import (
  "github.com/sirupsen/logrus"
)

func main() {
  logrus.WithFields(logrus.Fields{
    "name": "dj",
    "age": 18,
  }).Info("info msg")
}
// time="2021-11-02T15:52:03+08:00" level=info msg="info msg" age=18 name=dj
```

## 自定义 Formatter

除了 `logrus` 包中定义的两种 `Formatter` 外,我们可以自定义 `Formatter`.只需要实现如下接口即可.

```go
type Formatter interface {
	Format(*Entry) ([]byte, error)
}
```

如下是[文档](https://pkg.go.dev/github.com/sirupsen/logrus)中提供的一种自定义的 `Formatter`.

```go
type MyJSONFormatter struct {
}

func (f *MyJSONFormatter) Format(entry *logrus.Entry) ([]byte, error) {
  serialized, err := json.Marshal(entry.Data)
    if err != nil {
      return nil, fmt.Errorf("Failed to marshal fields to JSON, %w", err)
    }
  return append(serialized, '\n'), nil
}

func main(){
	logrus.SetFormatter(new(MyJSONFormatter))
	logrus.WithFields(logrus.Fields{
		"name": "MyJSONFormatter",
		"msg":  "message",
	}).Info("my json format")
}
// {"msg":"message","name":"MyJSONFormatter"}
```

从上面的示例可以看到, `Format()` 接口方法中包含 `Entry` 对象.其定义如下

```go
type Entry struct {
	Logger *Logger
	Data Fields  // 用户自定义的字段数据
	Time time.Time  // 日志 entry 被创建的日期
	Level Level  // 日志 entry 的级别
	Caller *runtime.Frame  // 产生日志的位置.包含包名
	Message string // 日志信息
	Buffer *bytes.Buffer
	Context context.Context // 上下文
	// 其他被过滤的字段
}
```

## hooks 

有这么一种需求,对不同的级别的日志,我希望能够输出到不同的位置.该怎么实现呢? `logrus` 提供了对日志级别的 `hooks` 机制.它可以对不同级别日志添加不同的 `hooks`.从而实现不同的需求.

`logrus` 提供了两种 hooks. `writer` 和 `SyslogHook`.可以实现不同的功能

### writer

参见 [github](https://github.com/sirupsen/logrus/tree/v1.8.1/hooks/writer) 的一个示例,将不同级别的日志写入到不同的日志文件中

```go
package main

import (
	log "github.com/sirupsen/logrus"
	"github.com/sirupsen/logrus/hooks/writer"
	"io/ioutil"
	"os"
)

func main() {
	log.SetOutput(ioutil.Discard)
	log.SetFormatter(&log.TextFormatter{
		TimestampFormat: "2006-01-02 15:03:04",
		FullTimestamp:   true,
	})

	errlog, err := os.OpenFile("err.log", os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		log.Error(err)
	}
	defer errlog.Close()
	infolog, err := os.OpenFile("info.log", os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		log.Error(err)
	}
	defer infolog.Close()

	log.AddHook(&writer.Hook{ // Send logs with level higher than warning to errlog
		Writer: errlog,
		LogLevels: []log.Level{
			log.PanicLevel,
			log.FatalLevel,
			log.ErrorLevel,
			log.WarnLevel,
		},
	})
	log.AddHook(&writer.Hook{ // Send info and debug logs to infolog
		Writer: infolog,
		LogLevels: []log.Level{
			log.InfoLevel,
			log.DebugLevel,
		},
	})
	log.Info("This will go to info.log")
	log.Warn("This will go to err.log")
}
```

### syslogHook

参见 [github](https://github.com/sirupsen/logrus/tree/v1.8.1/hooks/syslog) 将日志输出到 `syslog` 第三方日志处理工具中.

```
import (
  "log/syslog"
  "github.com/sirupsen/logrus"
  lSyslog "github.com/sirupsen/logrus/hooks/syslog"
)

func main() {
  log       := logrus.New()
  hook, err := lSyslog.NewSyslogHook("udp", "localhost:514", syslog.LOG_INFO, "")

  if err == nil {
    log.Hooks.Add(hook)
  }
}
```


其它类似的 `hooks` 还有

- [logrus-logstash-hook](https://github.com/bshuster-repo/logrus-logstash-hook)
- [logrus_fluent](https://github.com/evalphobia/logrus_fluent)
- [logrus_influxdb](https://github.com/abramovic/logrus_influxdb)
- [logrus-postgresql-hook](https://github.com/gemnasium/logrus-postgresql-hook)
- [logrus-kafka-hook](https://github.com/gfremex/logrus-kafka-hook)
- [logrus-redis-hook](https://github.com/rogierlommers/logrus-redis-hook)

---

参考:

- [github - logrus](https://github.com/sirupsen/logrus)
- [logrus package - github.com/sirupsen/logrus - pkg.go.dev](https://pkg.go.dev/github.com/sirupsen/logrus)
- [Go 每日一库之 logrus - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/105759117)
