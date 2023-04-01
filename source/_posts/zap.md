---
title: 高性能日志库 zap
date: 2021-11-17 14:59:44
tags:
  - go
  - 3rd
categories:
  - go
description: zap 是 uber 开源的高性能日志库.本文主要介绍 zap 库的简单使用,并简单了解 zap 库高性能的实现方式
---

[`zap`](https://github.com/uber-go/zap) 是 uber 开源的高性能日志库,提供了快速的、结构化的、可分级的日志记录.

目前常见的 log 库中,大多使用 `json.Marshal` 和 `fmt.Fprintf` 来记录大量 `interface{}`,这种基于反射的序列化和字符串格式化会占用大量 CPU 资源并进行许多小的内存的分配,这会使得日志记录变慢,从而影响整体应用程序.

而 `zap` 采用了上述不同的方法.它包含一个无反射、零分配的 JSON 编码器,并提供了基础 `Logger` 力求尽可能避免序列化开销和内存分配.

## 快速开始

`zap` 库的使用与其他的日志库非常相似.先创建一个 `logger`,然后调用各个级别的方法记录日志(Debug/Info/Error/Warn).我们可以通过[文档](https://pkg.go.dev/go.uber.org/zap) 中提供的一些示例来快速了解该库的使用方式.

```go
package main

import (
 "time"
 "go.uber.org/zap"
)

func main() {
 sugar := zap.NewExample().Sugar()
 defer sugar.Sync()
 sugar.Infow("failed to fetch URL",
  "url", "http://example.com",
  "attempt", 3,
  "backoff", time.Second,
 )
 sugar.Infof("failed to fetch URL: %s", "http://example.com")

 logger := zap.NewExample()
 defer logger.Sync()
 logger.Info("failed to fetch URL",
   // 需要使用 zap 内置的 Field 对象来强制定义字段的类型
  zap.String("url", "http://example.com"),
  zap.Int("attempt", 3),
  zap.Duration("backoff", time.Second),
 )
}
```

如上示例中,使用 `zap` 包提供的 `NewExample()` 方法创建 `Logger` 对象,并打印日志.相比于 `sugar`,`logger` 对象更快,分配的资源也少的多,但是它只支持强类型,结构化的日志记录.

另外,`zap` 提供了几个快速创建 `logger` 的方法: `zap.NewExample()`,`zap.NewDevelopment()`,`zap.NewProduction()` 以及高度定制化的 `zap.New()`.在使用前 3 个 logger 时, zap 会使用一些预定义的配置.详情可通过 [Github](https://github.com/uber-go/zap/blob/v1.19.1/logger.go#L98-L125) 查看.

### option

在通过以上方法创建 `logger` 对象时,zap 支持传入一些 `Option` 对象来调整 logger 对象的内部属性,从而定制 Logger 对象的行为.`zap` 库提供了丰富的构建 `Option` 对象的方法供我们选择.常用的如

#### 打印日志的行号

可以通过添加 `zap.AddCaller()` 返回的 `Options` 对象对日志添加行号.该 `Options` 不适用于 `zap.NewExample()` 返回的 `logger` 对象(因为该对象的 `Config` 没有 `CallerKey` 属性)

```go
func main() {
  logger, _ := zap.NewProduction(zap.AddCaller())
  defer logger.Sync()

  logger.Info("hello world")
}
// 输出
// {"level":"info","ts":1637302341.770313,"caller":"zap/main.go:11","msg":"hello world"}
```

再看下面,我们对日志进行了一些封装.

```go
func log(logger *zap.Logger, msg string) () {
 logger.Info(msg)
}

func main() {
 logger, _ := zap.NewProduction(zap.AddCaller())
 defer logger.Sync()
 log(logger, "hello world")
}
// 输出
// {"level":"info","ts":1637302670.685756,"caller":"zap/main.go:8","msg":"hello world"}
```

此时再看输出的行号就不准确了,因此需要跳过一些调用.`zap` 包提供了 `AddCallerSkip(skip int)` 函数跳过一些调用.如下:

```go
func log(logger *zap.Logger, msg string) () {
 logger.Info(msg)
}

func main() {
 logger, _ := zap.NewProduction(zap.AddCaller(),zap.AddCallerSkip(1))
 defer logger.Sync()
 log(logger, "hello world")
}
// 输出
// {"level":"info","ts":1637302912.6387358,"caller":"zap/main.go:14","msg":"hello world"}
```

此时再看输出日志的调用就准确了.

#### 添加调用栈

有时在出现错误时,我们希望输出代码的调用栈信息.`zap` 包提供了 `AddStackTrace(lvl zapcore.LevelEnabler)` 实现这个功能.如下:

```go
func log(logger *zap.Logger, msg string) () {
 logger.Error(msg)
}

func main() {
 logger, _ := zap.NewProduction(zap.AddCallerSkip(1), zap.AddStacktrace(zapcore.ErrorLevel))
 defer logger.Sync()
 log(logger, "hello world")
}
// 输出了调用栈信息
// {"level":"error","ts":1637303198.4552681,"caller":"zap/main.go:15","msg":"hello world","stacktrace":"main.main\n\t//code/go/src/go-notes/zap/main.go:15\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:255"}
```

### Filed

`logger` 对象在记录日志时,需要将 zap 提供的格式化的 `Filed` 对象来强制定义字段的类型,避免了 `interface{}` 反射所造成的性能损失.`zap` 包中提供了很多构造各种类型 `Filed` 对象的方法.如 `Bool`,`String` 等.zap 还支持形如 `Boolp`,`Bools` 的方法,用于传入该对象的指针与切片对象.如下

```go
func Bool(key string, val bool) Field
func Boolp(key string, val *bool) Field
func Bools(key string, bs []bool) Field
```

## 定制 Logger

我们可以通过调用 `NexExample()/NewDevelopment()/NewProduction()` 方法创建 `zap` 内置的默认的 `Logger` 对象.我们也可以通过自定义 `Config` 对象,并通过其 `Build` 方法构建自定义的 `Logger` 对象.其中,`Config` 结构体定义如下:

```go
type Config struct {
 // 日志级别
 Level AtomicLevel `json:"level" yaml:"level"`
 // 是否是开发环境
 Development bool `json:"development" yaml:"development"`
 // 是否禁用 caller 记录
 DisableCaller bool `json:"disableCaller" yaml:"disableCaller"`
 // 是否禁用调用栈
 DisableStacktrace bool `json:"disableStacktrace" yaml:"disableStacktrace"`
 // 日志输出格式
 Encoding string `json:"encoding" yaml:"encoding"`
 // 编码配置
 EncoderConfig zapcore.EncoderConfig `json:"encoderConfig" yaml:"encoderConfig"`
 // 日志输出位置,可以定义多个,多用于文件路径和 stdout
 OutputPaths []string `json:"outputPaths" yaml:"outputPaths"`
 // 错误输出位置
 ErrorOutputPaths []string `json:"errorOutputPaths" yaml:"errorOutputPaths"`
 // 初始化字段,每条日志中都会包含这个字段
 InitialFields map[string]interface{} `json:"initialFields" yaml:"initialFields"`
}
```

可通过定义 `Config` 后进行构建 `Logger` 对象

```go
func main() {
 rawJSON := []byte(`{
    "level":"debug",
    "encoding":"json",
    "outputPaths": ["stdout", "server.log"],
    "errorOutputPaths": ["stderr"],
    "initialFields":{"name":"dj"},
    "encoderConfig": {
      "messageKey": "message",
      "levelKey": "level",
      "levelEncoder": "lowercase"
    }
  }`)

 var cfg zap.Config
 if err := json.Unmarshal(rawJSON, &cfg); err != nil {
  panic(err)
 }
 logger, err := cfg.Build()
 if err != nil {
  panic(err)
 }
 defer logger.Sync()

 logger.Info("server start work successfully!")
}
```

可以看到标准输出 std 和 server.log 的日志

```text
{"level":"info","message":"server start work successfully!","name":"dj"}
```

## 搭配标准日志库

如果项目一开始使用的是标准日志库 `log`,后面想转为 `zap`.我们可以调用 `zap.NewStdLog(l *Logger) *log.Logger`返回一个标准的 `log.Logger`,内部实际上写入的还是我们之前创建的 `zap.Logger`.如下:

```go
func main() {
 logger := zap.NewExample()
 defer logger.Sync()
 
 // NewStdLog 返回包装后的 *log.Logger 对象
 std := zap.NewStdLog(logger)
 std.Print("standard logger wrapper")
}
```

## 常用函数及方法

```go
// 返回标准库 *log.Logger 对象,实现无缝替换
func NewStdLog(l *Logger) *log.Logger

// 创建 info 级别的 AtomicLevel 对象
func NewAtomicLevel() AtomicLevel
// 可以使用上述 AtomicLevel 对象修改级别.包含 DebugLevel,InfoLevel,WarnLevel,ErrorLevel,DPanicLevel,PanicLevel,FatalLevel
func (lvl AtomicLevel) SetLevel(l zapcore.Level)

// 自定义 Logger 对象
func New(core zapcore.Core, options ...Option) *Logger

// 创建 开发,示例,生产环境的 Logger 对象
func NewDevelopment(options ...Option) (*Logger, error)
func NewExample(options ...Option) *Logger
func NewProduction(options ...Option) (*Logger, error)
// logger 对象完成工作后,一般需要使用 defer logger.Sync() 同步
func (log *Logger) Sync() error

// 添加字段或添加一些 options
func (log *Logger) With(fields ...Field) *Logger
func (log *Logger) WithOptions(opts ...Option) *Logger


// 由强类型的 Logger 对象转化为弱类型的 SugaredLogger 对象
func (log *Logger) Sugar() *SugaredLogger

// 不同级别的记录日志的方法
func (log *Logger) Info(msg string, fields ...Field)


// 创建 option 对象
// 添加调用行,调用栈
func AddCaller() Option
func WithCaller(enabled bool) Option
func AddCallerSkip(skip int) Option
func AddStacktrace(lvl zapcore.LevelEnabler) Option


// 创建 Field 对象
// 3种构建指定类型的 Field 字段,可以传入到日志记录中,实现快速记录
func Bool(key string, val bool) Field
func Boolp(key string, val *bool) Field
func Bools(key string, bs []bool) Field
```

## zap 高性能的秘诀

- 避免 GC: 对象复用
- 内建的 Encoder: 避免反射
- 使用写时复制机制,避免竞态

具体可参见 [深度 | 从Go高性能日志库zap看如何实现高性能Go组件](https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw)

---

参考:

- [uber-go/zap (github.com)](https://github.com/uber-go/zap)
- [zap - pkg.go.dev](https://pkg.go.dev/go.uber.org/zap)
- [Go 每日一库之 zap](https://segmentfault.com/a/1190000022461706)
- [深度 | 从Go高性能日志库zap看如何实现高性能Go组件](https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw)
