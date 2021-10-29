---
title: cobra 快速开始
date: 2020/06/23
tags:
  - go
  - 3rd
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

应用程序遵循的命令行模式应该是 `APPNAME COMMAND ARG --FLAG`.

如下示例中, `server` 是一个命令,`port` 是一个标志, `1313` 是参数:

```bash
hugo server --port=1313
```

### Commands 命令

命令是应用程序的核心.应用程序支持的每个交互动作都将包含在命令中.命令可以有子命令,还可以执行操作.

在如上示例中,`server` 是一个命令.

有关命令的更多内容,请参见 [cobra.Command](https://godoc.org/github.com/spf13/cobra#Command).

### Flags 标志

标志是一种改变命令行为的方法.Cobra 支持 POSIX 风格的标志以及 Go [flag](https://golang.org/pkg/flag/).Cobra 命令可以定义子命令同样适用的标志和仅用于该命令的标志.

在上面的示例中,`port` 是一个标志

标志相关功能由 [`pflag`](https://github.com/spf13/pflag) 库提供,该库是 flag 标准库的一个分支,它包含相同的接口,并添加了 POSIX 风格标志的兼容性.

## 安装

Cobra 的使用非常简单.首先,使用 `go get` 安装最新版的库.如下命令将安装 `cobra` 及其依赖项并生成可执行文件:

```bash
#export GO111MODULE=on
#export GOPROXY=https://goproxy.cn
go get -u github.com/spf13/cobra
```

然后,在你的应用中导入 Cobra

```go
import "github.com/spf13/cobra"
```

## 快速开始

### 使用 cobra 命令行工具生成代码

安装 `cobra` 后会在 `$GOPATH/bin` 中生成 `cobra` 可执行文件,可以使用它来生成大体代码

```text
# cobra -h

Usage:
  cobra [command]

Available Commands:
  add         Add a command to a Cobra Application  # 添加子命令
  help        Help about any command    # 帮助信息
  init        Initialize a Cobra Application    # 初始化一个命令

Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")    # 指定作者
      --config string    config file (default is $HOME/.cobra.yaml) # 指定配置文件
  -h, --help             help for cobra
  -l, --license string   name of license for the project    # 指定项目的 license,默认基于 Apache 2.0
      --viper            use Viper for configuration (default true)
```

可以使用 `cobra init <appName>` 自动创建应用程序.如 `cobra init cli`. 基于 Cobra 的应用程序通常会遵循以下组织结构:

```text
▾ cli/
▾ cmd/
    root.go
  main.go
```

在 Cobra 应用程序中,通常 `main.go` 文件非常简单,它只有一个目的: 初识化 Cobra.

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

查看 `cmd/root.go` 可以发现由 `cobra` 自动生成的代码, 其中包含自动生成的帮助信息

```go
// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
    Use:   "cli",
    Short: "A brief description of your application",
    Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application.`,
    // Uncomment the following line if your bare application
    // has an action associated with it:
    // Run: func(cmd *cobra.Command, args []string) { },
}

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
    cobra.CheckErr(rootCmd.Execute())
}
```

可通过 `cobra add <subcommand>` 添加子命令,如 `cobra add app`.

cobra 会自动生成 app 子命令的相关代码.

```go
var appCmd = &cobra.Command{
	Use:   "app",
	Short: "A brief description of your command",
	Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command.`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("app called")
	},
}
// 在 init() 中 rootCmd 添加了 appCmd 子命令. AddCommand 可以在父命令中添加子命令
func init() {
	rootCmd.AddCommand(appCmd)
}
```
下面,我们在 app 子命令下继续添加子命令 remove, 结构如下

```text
cli
|----app
      |----remove
```

而 `cobra add remove` 实际生成的代码将会是如下结构,因此需要手动修改 `remove.go` 里的代码

```text
cli
|----app
|----remove
```

```go
// remove.go
func init() {
	rootCmd.AddCommand(removeCmd)
}
// 需要修改为
func init() {
	appCmd.AddCommand(removeCmd)
}
```

此时,可以看到 remove 子命令在 app 命令下

```text
Usage:
  cli app [flags]
  cli app [command]

Available Commands:
  remove      A brief description of your command

Flags:
  -h, --help   help for app

Global Flags:
      --config string   config file (default is $HOME/.cli.yaml)

Use "cli app [command] --help" for more information about a command.
```

此时,文件目录结构如下:

```text
▾ cli/
▾ cmd/
    app.go
    remove.go
    root.go
  main.go
```

为了规范化,命令及子命令的目录结构,将子命令放在同一包下,如:
```text
▾ cli/
▾ cmd/
    ▾ app1/
        app1.go
        remove.go
    ▾ app2/
        app2.go
        remove.go
    root.go
  main.go
```

则需要进行如下操作

```text
1. 将 app 声明为其他包可访问的变量. var appCmd 变更为 var AppCmd
2. 在 app.go 中的 init() 函数中使用 AppCmd.AddCommand() 函数添加 remove 子命令
3. 在 root.go 中的 init() 函数中使用 rootCmd.AddCommand() 函数添加多个 app 子命令
```

#### Command的常见字段

```go
type Command struct {
    Use       string   // 单行的用法信息,可以理解为为命令名称
    Aliases   []string // 命令别名
    Short     string   // 短帮助信息
    Long      string   // 长帮助信息
    Example   string   // 命令的用法示例
    ValidArgs []string // 命令行中传入非标志位参数的有效列表
    // ValidArgsFunction 用于判断命令行中传入非标志位参数是否有效
    ValidArgsFunction      func(cmd *Command, args []string, toComplete string) ([]string, ShellCompDirective)
    Args                   PositionalArgs // 限定命令行参数,见 Args 小节
    BashCompletionFunction string         // 生成命令行补全命令的函数
    Deprecated             string         // 过期命令执行是打印的内容
    Version                string         // 定义命令版本内容,一般使用 cmd.SetVersionTemplate() 设置版本信息

    // *Run func(cmd *cobra.Command, args []string)
    // The *Run functions are executed in the following order:
    //   * PersistentPreRun()
    //   * PreRun()
    //   * Run()
    //   * PostRun()
    //   * PersistentPostRun()
    // All functions get the same args, the arguments after the command name.

    TraverseChildren   bool // 是否继承父命令的标志位
    Hidden             bool // 是否隐藏此命令
    SilenceErrors      bool // 是否静默错误
    SilenceUsage       bool // 是否在错误发生时静默用法信息,默认在发生错误时会打印用法信息
    DisableSuggestions bool // 是否禁用命令建议
    SuggestionsMinimumDistance  int // 最小 levenshtein 距离以显示建议
}
```

#### Command的常见方法

```go
func (c *Command) AddCommand(cmds ...*Command)  // 添加子命令
func (c *Command) Execute() error // 运行命令树,并解析标志位
func (c *Command) Flags() *flag.FlagSet // 获取命令的标志位
func (c *Command) GenBashCompletion(w io.Writer) error  // 生成 bash 自动完成文件


// 打印帮助,用法,版本等信息
func (c *Command) Help() error
func (c *Command) HelpFunc() func(*Command, []string)
func (c *Command) HelpTemplate() string
func (c *Command) Usage() error
func (c *Command) UsageFunc() (f func(*Command) error)
func (c *Command) UsageTemplate() string
func (c *Command) VersionTemplate() string

// 设置帮助,用法,版本等信息
func (c *Command) SetHelpFunc(f func(*Command, []string))
func (c *Command) SetHelpTemplate(s string)
func (c *Command) SetUsageFunc(f func(*Command) error)
func (c *Command) SetUsageTemplate(s string)
func (c *Command) SetVersionTemplate(s string)

// 验证参数
func (c *Command) ValidateArgs(args []string) error
```

### 使用 Flags

Flags 提供修饰符用于控制命令操作的方式.标志有两种不同的指配方式.

#### 全局标志位与局部标志位

一个标志是 `persistent`,意味着该标志可用于指配给它的命令及该命令下的每个子命令.即全局标志.对于全局标志,将标志指配为 `rootcmd` 根命令上的持久标志.

```go
rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")   // 全局标志位
localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")   // 仅限于该命令使用的
```

#### 标志位参数获取

由于标志位是不同位置定义和使用的,因此我们需要在外部定义一个具有正确作用域的变量以分配要使用的标志位.

```go
var Verbose bool
var Source string

localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
```

其中,添加选项参数都是在 `init()` 函数里使用 `cmd.Flags()` 或者 `cmd.PersistentFlags()` 的方法,且具有以下使用规律

```text
type    typeP
typeVar typeVarP
```

有如下区别:

- `typeP`,`typeVarP` 相对于 `type`, `typeVar` 多了一个短选项,而没有 P 的只能使用类似于 `--long-iotion` 格式的长选项.
- `typeVar`, `typeVarP` 支持传入一个变量地址,从而将标志位传入的值直接传入到该变量中.而不需要使用 `GetType("optionName")`.

```go
String(name string, value string, usage string)
StringP(name, shorthand string, value string, usage string)
StringVar(p *string, name string, value string, usage string)
StringVarP(p *string, name, shorthand string, value string, usage string)

// name: 标志位名称
// value: 默认值
// usage: 帮助信息
// shorthand: 短标志位名称
// p: 标志位获取到的值传入该参数
```

#### 标志位继承

默认情况下,Cobra 仅解析目标命令上的局部标志,而忽略父命令上的任何局部标志.通过启用 `Command.TraverseChildren`,Cobra 将在执行目标命令之前解析每个命令上的局部标志.

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

#### 必须的标志位

标志默认是可选的.如果您希望命令在未设置标志时报错,请将其标记为 `required`:

```go
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
```

### Args 自定义参数

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

### 自定义帮助,用法,版本信息

```go
// 自定义帮助信息
cmd.SetHelpCommand(cmd *Command)
cmd.SetHelpFunc(f func(*Command, []string))
cmd.SetHelpTemplate(s string)

// 自定义用法信息
cmd.SetUsageFunc(f func(*Command) error)
cmd.SetUsageTemplate(s string)

// 自定义版本信息
cmd.setVersionTemplate(s string)
```

### PreRun and PostRun Hooks

可以在根命令的 `Run` 函数之前或之后运行函数.其中, `PersistentPreRun` 和 `PreRun` 函数将在 `Run` 函数之前执行.`PersistentPostRun` 和 `PostRun` 将在 `Run` 函数之后执行.

如果子命令没有定义 `Persistent*Run` 函数,它们将继承父命令的.这些函数按照以下顺序运行:

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
// 或
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

