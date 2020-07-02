---
title: cobra 快速开始
date: 2020/06/23
tags:
  - go
  - cobra
categories:
  - go
description: 在学习 Go 语言过程中发现很多项目都导入了 cobra 库.因此,根据 GitHub 上 cobra 项目的 README.md 对 cobra 进行系统的学习.
---

[`Cobra`](https://github.com/spf13/cobra) 是一个可以创建功能强大的现代 CLI 应用程序的库,又是一个可以用于生成应用程序和命令文件的程序.

Cobra 被应用于很多项目中,如 [Kubernetes](http://kubernetes.io/), [Hugo](https://gohugo.io), 和 [Github CLI](https://github.com/cli/cli) 等.[此列表](https://github.com/spf13/cobra/blob/master/projects_using_cobra.md) 包含了使用 Cobra 项目的列表.

## 概述

Cobra 是一个库,它提供了简单易用的接口,用于创建功能强大的现代 CLI 接口,如 git & go tool.

Cobra 也是一个程序(工具),它可以生成基于 Cobra 的应用程序的代码结构以便于快速开发.

Cobra 提供了以下功能:

- 简单的子命令行模式: `app server`, `app fetch` 等
- 完全兼容 POSIX 标志(长短版本均支持)
- 嵌套子命令
- 全局,局部和级联标志
- 使用 `cobra init appname` 或 `cobra add cmdname` 轻松生成应用程序或命令
- 智能建议(如, `app srver`... 您是指 `app server`)
- 自动生成命令和标志的帮助
- 自动帮助标志识别,如 `-h`, `--help` 等
- 自动为您的应用生成 bash 自动完成
- 自动为您的应用生成帮助手册
- 支持命令别名
- 灵活地自定义帮助,用法等.
- 可选的与 viper 紧密集成,可用于 12-factor 应用.

## 概念

Cobra 建立在 commands(命令), arguments(参数), flags(标志) 之上.

**Commands** 表示要执行的操作, **Args** 和 **Flags** 是这些操作的修饰符.

最好的应用程序在使用时就像读句子.用户知道如何使用该应用程序,因为他们可以自然地理解如何使用它.

应用程序遵循的命令行模式应该是 `APPNAME VERB NOUN --ADJECTIVE` 或 `APPNAME COMMAND ARG --FLAG`.

一些示例可以更好的说明这一点.

如下示例中, `server` 是一个命令,`port` 是一个标志:

```bash
hugo server --port=1313
```

在如下命令中,我们告诉 git 只是克隆 URL 内容,而不创建 `.git` 相关文件.

```bash
git clone URL --bare
```

### Commands 命令

命令是应用程序的核心.应用程序支持的每个交互动作都将包含在命令中.命令可以有子命令,还可以执行操作.

在如上示例中,`server` 是一个命令.

有关命令的更多内容,请参见 [cobra.Command](https://godoc.org/github.com/spf13/cobra#Command).

### Flags

标志是一种改变命令行为的方法.Cobra 支持 POSIX 风格的标志以及 Go [flag](https://golang.org/pkg/flag/).Cobra 命令可以定义子命令同样适用的标志和仅用于该命令的标志.

在上面的示例中,`port` 是一个标志

标志相关功能由 [`pflag`](https://github.com/spf13/pflag) 库提供,该库是 flag 标准库的一个分支,它包含相同的接口,并添加了 POSIX 风格标志的兼容性.

## 安装

Cobra 的使用非常简单.首先,使用 `go get` 安装最新版的库.如下命令将安装 `cobra` 及其依赖项并生成可执行文件:

```bash
go get -u github.com/spf13/cobra/cobra
```

然后,在你的应用中导入 Cobra

```go
import "github.com/spf13/cobra"
```

## 快速开始

基于 Cobra 的应用程序通常会遵循以下组织结构,当然您也可以遵循自己的组织结构:

```text
  ▾ appName/
    ▾ cmd/
        add.go
        your.go
        commands.go
        here.go
      main.go
```

在 Cobra 应用程序中,通常 main.go 文件非常简单,它只有一个目的: 初识化 Cobra.

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

### 使用 Cobra 生成

Cobra 提供了自己的程序,该程序将创建您的应用程序并添加所需的任何命令.这是将 Cobra 集成到您的应用程序中的最简单方法.

在[这里](https://github.com/spf13/cobra/blob/master/cobra/README.md)您可以找到有关它的更多信息

### 使用 Cobra 库

要手动使用 Cobra,您需要创建创建一个简单的 main.go 文件和一个 rootCmd 文件.您可以可选地提供其它合适的命令.

#### 创建 rootCmd

Cobra 不需要任何特殊的构造函数.只需要创建您的命令即可.

理想情况下,你可以将如下代码写入 `app/cmd/root.go` 中:

```go
var rootCmd = &cobra.Command{
  Use:   "hugo",
  Short: "Hugo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at http://hugo.spf13.com`,
  Run: func(cmd *cobra.Command, args []string) {
    // Do Stuff Here
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}
```

您可以在 `init()` 函数中定义标志或解析配置.

例如, `cmd/root.go` 文件:

```go
package cmd

import (
    "fmt"
    "os"
    homedir "github.com/mitchellh/go-homedir"
    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var (
    // Used for flags.
    cfgFile     string
    userLicense string
    rootCmd = &cobra.Command{
        Use:   "cobra",
        Short: "A generator for Cobra based Applications",
        Long: `Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
    }
)

// Execute executes the root command.
func Execute() error {
    return rootCmd.Execute()
}

func init() {
    cobra.OnInitialize(initConfig)

    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra.yaml)")
    rootCmd.PersistentFlags().StringP("author", "a", "YOUR NAME", "author name for copyright attribution")
    rootCmd.PersistentFlags().StringVarP(&userLicense, "license", "l", "", "name of license for the project")
    rootCmd.PersistentFlags().Bool("viper", true, "use Viper for configuration")
    viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
    viper.BindPFlag("useViper", rootCmd.PersistentFlags().Lookup("viper"))
    viper.SetDefault("author", "NAME HERE <EMAIL ADDRESS>")
    viper.SetDefault("license", "apache")

    rootCmd.AddCommand(addCmd)
    rootCmd.AddCommand(initCmd)
}

func er(msg interface{}) {
    fmt.Println("Error:", msg)
    os.Exit(1)
}

func initConfig() {
    if cfgFile != "" {
        // Use config file from the flag.
        viper.SetConfigFile(cfgFile)
    } else {
        // Find home directory.
        home, err := homedir.Dir()
        if err != nil {
            er(err)
        }

        // Search config in home directory with name ".cobra" (without extension).
        viper.AddConfigPath(home)
        viper.SetConfigName(".cobra")
    }

    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err == nil {
        fmt.Println("Using config file:", viper.ConfigFileUsed())
    }
}
```

#### 创建 main.go 文件

创建 `rootCmd` 后,您需要让您程序的 `mian()` 函数执行它.为了清晰起见,`Execute()` 函数应该在入口函数中调用,尽管命令都可以调用它.

在 Cobra 应用程序中,通常 main.go 文件非常简单,它只有一个目的: 初识化 Cobra.

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

#### 创建其它命令

可以定义其它命令,且通常在 `cmd/` 目录中为每个命令提供自己的文件.

如果要创建 `version` 命令,则可以创建 `cmd/version.go`,内容如下:

```go
package cmd

import (
  "fmt"

  "github.com/spf13/cobra"
)

func init() {
  rootCmd.AddCommand(versionCmd)
}

var versionCmd = &cobra.Command{
  Use:   "version",
  Short: "Print the version number of Hugo",
  Long:  `All software has versions. This is Hugo's`,
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
  },
}
```

### 使用 Flags

Flags 提供修饰符用于控制命令操作的方式.

#### 将 flags 指配给 command

由于标志时在不通位置定义和使用的,因此我们需要在外部定义一个具有正确作用域的变量以分配要使用的标志.

```go
var Verbose bool
var Source string
```

标志有两种不同的指配方式.

#### 持久的标志

一个标志是 '持久的',意味着该标志可用于指配给它的命令及该命令下的每个子命令.对于全局标志,将标志指配为 `rootcmd` 根命令上的持久标志.

```go
rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")
```

#### 本地标志

标志也可以被指定为本地的,这类标志仅适用于这个指定的命令:

```go
localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
```

#### 父命令上的本地标志

默认情况下,Cobra 仅解析目标命令上的本地标志,而忽略父命令上的任何本地标志.通过启用 `Command.TraverseChildren`,Cobra 将在执行目标命令之前解析每个命令上的本地标志.

```go
command := cobra.Command{
    Use: "print [OPTIONS] [COMMANDS]",
    TraverseChildren: true,
}
```

#### 标志与配置绑定

您还可以通过 [`viper`](https://github.com/spf13/viper) 绑定标志:

```go
var author string

func init() {
    rootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution")
    viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
}
```

在如上示例中,持久标志 `author` 被 `viper` 绑定.注意,当用户没有指定 `--author` 标志时,`author` 变量不会被赋值.

#### 必须的标志

标志默认是可选的.如果您希望命令在未设置标志时报错,请将其标记为 `required`:

```go
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
```

### 自定义参数

可以使用 `Command` 的 `Args` 字段指定参数的校验规则.

Cobra 内置了如下校验规则:

- `NoArgs` - 如果有任何参数,该命令将报错
- `ArbitraryArgs` - 该命令接受任何参数
- `OnlyValidArgs` - 如果有任何参数不在 `Command` 的 `ValidArgs` 字段中,则该命令将报错
- `MinimumNArgs(int)` - 至少需要 N 个参数,否则将报错
- `MaximumNArgs(int)` - 最多传入 N 个参数,否则将报错
- `ExactArgs(int)` - 传入参数个数为 N,否则将报错
- `ExactValidArgs(int)` - 传入参数个数为 N,且每个参数都在 `Command` 的 `ValidArgs` 字段中,否则将报错
- `RangeArgs(min, max)` - 参数的数量在 min 和 max 之间,否则将报错

如下是自定义参数验证的示例:

```go
var cmd = &cobra.Command{
  Short: "hello",
  Args: func(cmd *cobra.Command, args []string) error {
    if len(args) < 1 {
      return errors.New("requires a color argument")
    }
    if myapp.IsValidColor(args[0]) {
      return nil
    }
    return fmt.Errorf("invalid color specified: %s", args[0])
  },
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}
```

### 示例

在如下示例中,我们定义了三个命令.两个是顶级命令,一个(cmdTimes)是其中一个顶级命令的字命令.在此示例中,根命令不可执行,因此需要一个子命令.这通过不为 `rootCmd` 提供 `Run` 字段来实现

```go
package main

import (
  "fmt"
  "strings"

  "github.com/spf13/cobra"
)

func main() {
  var echoTimes int

  var cmdPrint = &cobra.Command{
    Use:   "print [string to print]",
    Short: "Print anything to the screen",
    Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Println("Print: " + strings.Join(args, " "))
    },
  }

  var cmdEcho = &cobra.Command{
    Use:   "echo [string to echo]",
    Short: "Echo anything to the screen",
    Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Println("Echo: " + strings.Join(args, " "))
    },
  }

  var cmdTimes = &cobra.Command{
    Use:   "times [string to echo]",
    Short: "Echo anything to the screen more times",
    Long: `echo things multiple times back to the user by providing
a count and a string.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      for i := 0; i < echoTimes; i++ {
        fmt.Println("Echo: " + strings.Join(args, " "))
      }
    },
  }

  cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")

  var rootCmd = &cobra.Command{Use: "app"}
  rootCmd.AddCommand(cmdPrint, cmdEcho)
  cmdEcho.AddCommand(cmdTimes)
  rootCmd.Execute()
}
```

有关大型应用程序的更完整示例,请参考 [Hugo](http://gohugo.io/).

### Help Command

当有子命令时,Cobra 自动向您的应用程序添加一个帮助命令.当用户运行 `app help` 时,将调用此方法.此外,help 命令还支持所有其它命令作为输入.如,假设您有一个名为 'create' 的命令,而没有任何其它配置.当调用 `app help create` 时,Cobra 将自动工作.每个命令都会自动添加 `--help` 标志.

#### 示例

以下输出由 Cobra 自动生成.除命令和标志定义外,什么都不需要做.

```text
$ cobra help

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cobra [command]

Available Commands:
  add         Add a command to a Cobra Application
  help        Help about any command
  init        Initialize a Cobra Application

Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")
      --config string    config file (default is $HOME/.cobra.yaml)
  -h, --help             help for cobra
  -l, --license string   name of license for the project
      --viper            use Viper for configuration (default true)

Use "cobra [command] --help" for more information about a command.
```

帮助命令就像其它命令一样.没有特殊的逻辑或行为.实际上,您可以根据需要自行修改.

#### 自定义帮助信息

您可以使用以下函数为默认命令提供自己的帮助命令或模版:

```go
cmd.SetHelpCommand(cmd *Command)
cmd.SetHelpFunc(f func(*Command, []string))
cmd.SetHelpTemplate(s string)
```

### Usage Message

当用户提供无效标志或无效命令时,Cobra 会通过向用户显示 'usage' (用法) 作为响应.

#### 示例

```text
$ cobra --invalid
Error: unknown flag: --invalid
Usage:
  cobra [command]

Available Commands:
  add         Add a command to a Cobra Application
  help        Help about any command
  init        Initialize a Cobra Application

Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")
      --config string    config file (default is $HOME/.cobra.yaml)
  -h, --help             help for cobra
  -l, --license string   name of license for the project
      --viper            use Viper for configuration (default true)

Use "cobra [command] --help" for more information about a command.
```

#### 自定义用法信息

您可以通过自定义自己的使用函数或模版供 Cobra 使用.类似于 help 命令,函数与模版可以被公共方法覆盖.

```go
cmd.SetUsageFunc(f func(*Command) error)
cmd.SetUsageTemplate(s string)
```

### Version Flag

如果在根命令上设置了 `Version` 字段,Cobra 将自动自动添加一个顶级的 `--version` 标志.使用 `--version` 运行应用程序将使用版本模版将版本打印到标准输出.可以使用 `cmd.setVersionTemplate(s string)` 函数自定义模版.

### PreRun and PostRun Hooks

可以在根命令的 `Run` 函数之前或之后运行函数.`PersistentPreRun` 和 `PreRun` 函数将在 `Run` 函数之前执行.`PersistentPostRun` 和 `PostRun` 将在 `Run` 函数之后执行.如果子命令没有定义 `Persistent*Run` 函数,它们将继承父命令的.这些函数按照以下顺序运行:

- `PersistentPreRun`
- `PreRun`
- `Run`
- `PostRun`
- `PersistentPostRun`

下面是使用所有这些函数的两个命令的示例.子命令执行后,它将运行根命令的 `PersistentPreRun`,而不运行根命令的 `PersistentPostRun`:

```go
package main

import (
  "fmt"

  "github.com/spf13/cobra"
)

func main() {

  var rootCmd = &cobra.Command{
    Use:   "root [sub]",
    Short: "My root command",
    PersistentPreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPreRun with args: %v\n", args)
    },
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPostRun with args: %v\n", args)
    },
  }

  var subCmd = &cobra.Command{
    Use:   "sub [no options!]",
    Short: "My subcommand",
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PersistentPostRun with args: %v\n", args)
    },
  }

  rootCmd.AddCommand(subCmd)

  rootCmd.SetArgs([]string{""})
  rootCmd.Execute()
  fmt.Println()
  rootCmd.SetArgs([]string{"sub", "arg1", "arg2"})
  rootCmd.Execute()
}
```

输出:

```text
Inside rootCmd PersistentPreRun with args: []
Inside rootCmd PreRun with args: []
Inside rootCmd Run with args: []
Inside rootCmd PostRun with args: []
Inside rootCmd PersistentPostRun with args: []

Inside rootCmd PersistentPreRun with args: [arg1 arg2]
Inside subCmd PreRun with args: [arg1 arg2]
Inside subCmd Run with args: [arg1 arg2]
Inside subCmd PostRun with args: [arg1 arg2]
Inside subCmd PersistentPostRun with args: [arg1 arg2]
```

### Suggestions when "unknown command" happens

出现 "unknown command" 错误时,Cobra 将打印建议.发生错字时,Cobra 的行为类似于 git 命令.例如:

```text
$ hugo srever
Error: unknown command "srever" for "hugo"

Did you mean this?
        server

Run 'hugo --help' for usage.
```

根据注册的每个子命令,建议都是自动生成的,并使用 [Levenshtein 距离](http://en.wikipedia.org/wiki/Levenshtein_distance)实现.每个匹配最小距离 2(忽略大小写)的已注册命令都将显示建议.

如果要禁用建议或在命令中调整字符串距离,请使用:

```go
command.DisableSuggestions = true
# 或
command.SuggestionsMinimumDistance = 1
```

您还可以使用 `SuggestFor` 属性来显式设置为给定命令建议的名称,这样就可以针对字符串提出建议,对于您的命令集和不希望使用别名的命令都有意义.如:

```text
$ kubectl remove
Error: unknown command "remove" for "kubectl"

Did you mean this?
        delete

Run 'kubectl help' for usage.
```
