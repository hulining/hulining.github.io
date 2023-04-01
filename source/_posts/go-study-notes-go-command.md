---
title: go 学习笔记之 go 命令
date: 2020/05/02
tags:
  - go
categories:
  - go
abbrlink: 44881
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 提供的命令的常用使用方式, 以及在实际中应用'
---

安装配置好 Go 语言环境之后, 让我们先来看一看 go 为我们提供了哪些命令, 以及这些命令可以做什么吧

查看 go 命令行帮助可以了解到 `go` 是用于管理 Go 代码的命令行工具.

它提供了很多子命令来实现各种功能. 使用方式为 `go <command> [arguments]`, 使用 `go help <command>` 可以查看子命令的帮助信息

以下是 go 的子命令及其简单描述

子命令 | 简单解释 | 详细描述
:---: | :--- | :---
bug | bug/issue 报告 | 它会收集当前系统及 Go 语言信息, 在 `github/golang/go` 代码仓库中帮助你创建一个 issues
build | 编译软件包和依赖 |
clean | 删除旧的对象文件和缓存文件 |
doc | 显示包及其内容或符号的帮助文档 |
env | 打印当前 Go 环境信息 | 包括 `GO111MODULE`, `GOARCH`, `GOBIN`, `GOHOSTARCH`, `GOHOSTOS`, `GOPATH`, `GOPROXY`, `GOROOT`, `GOTOOLDIR`,`GOMOD` 等环境变量
fix | 更新软件包以使用新的 API |
fmt | 用于格式化 go 源代码
generate | 通过预处理源生成 Go 代码文件
get | 将依赖项添加到当前模块并安装它们
install | 编译并安装软件包和依赖
list | 列出软件包或模块
mod | 提供对模块的操作
run | 编译并运行 Go 程序
test | 测试包
tool | 运行指定的 go 工具
version | 打印 Go 版本信息
vet | 报告包中可能的错误 | 一些语法上虽然没有错误, 但是逻辑上可能存在问题的错误, 如死循环

它还提供了一些额外的话题帮助. 使用 `go help <topic>` 可以查看关于该话题的更多帮助.

topic | 简单描述
:--- | :---
buildmode | 构建模式
c | Go 与 C 语言间的调用
cache | 简历并测试缓存
environment | 环境变量
filetype | 文件类型
go.mod | go.mod 文件, 该文件保存了项目依赖的模块, 类似于依赖包的集合(python 中的 requirements.txt 文件)
gopath | `GOPATH` 环境变量
gopath-get | 早期的 go get 的 GOPATH
goproxy | 访问 go module 的代理
importpath | import 导入包路径的语法
modules | 模块, 模块版本等
module-get |  go get 的模块感知
module-auth | 使用 go.sum 进行身份认证
module-private | 非公共模块的相关配置
packages | 包
testflag | 测试的标志
testfunc | 测试函数

## `go build`

构建编译包及其依赖, 但是不会安装

```bash
go build [-o output] [-i] [build flags] [packages]
```

- 如果要构建的参数是单个目录中的 `.go` 列表文件, 则 build 会将其视为指定单个程序包的源文件列表
- 编译软件包时, build 会忽略以 `_test.go` 结尾的文件
- 编译单个 main 包时, build 将生成的可执行文件输出为第一个源代码文件名或源代码目录名文件(如 `go build ed.go rx.go` 生成 `ed`/`ed.exe`, `go build unix/sam` 生成 `sam`/`sam.exe`). 编译 Windows 系统可以执行文件时, 会添加 ".exe" 后缀
- 编译多个软件包或单个非 main 软件包时, build 会编译软件包, 但是会丢弃生成的对象. 仅用作检查是否可以构建软件包

标志 | 作用
:---: | :---
`-o` | 将生成的可执行文件或对象输出为指定的文件名或输出到指定目录
`-i` | 安装依赖的软件包

以下标志是 `build`, `clean`, `get`, `install`, `list`, `run`, `test` 命令共享的

标志 | 解释
:---: | :---
`-a` | 强制重新构建编译已更新的软件包
`-n` | 打印命令, 但不运行
`-p n` | 可以并行运行的程序数量, 默认是可用的 CPU 数量
`-race` | 启用数据竞争检测, 仅支持 linux/amd64, freebsd/amd64, darwin/amd64 和 windows/amd64
`-msan` |
`-v` | 在编译时打印软件包名称
`-work` | 打印临时工作目录名, 退出时不要删除它
`-x` | 打印命令
`-asm` |
`-buildmode mode` | 构建的模式选择, 可选模式为 `archive`, `c-archive`, `c-shared`, `default`, `shared`, `exe`, `pie`, `plugin`. 详见 `go help buildmode`
`-compiler name` | 编译器的名称 如 `runtime.Compiler`(gccgo 或 gc)
`-ldflags -X arg=value` | 传递编译过程中调用的参数. 如在编译时将代码的 `git` 等信息作为参数值传入到代码变量中.见下面示例
`-mod mode` | 模块的下载模式: readonly 或 vendor
`-pkgdir dir` | 从指定目录安装和加载所有软件包, 在非标准配置进行构建时, 使用 -pkgdir 将生成的软件包保存在单独的位置
`-tags tag,list` | 以逗号分割的构建标记列表

```bash
#!/bin/bash
BUILD_VERSION=${TAG_VERSION}
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
COMMIT_ID=$(git log | head -n1 | awk '{print $2}')

function add_ldflag() {
  local key=${1}
  local val=${2}
  ldflags+=(
    "-X 'main.${key}=${val}'"
  )
}

add_ldflag 'Version' ${BUILD_VERSION}
add_ldflag 'buildDate' ${BUILD_DATE}
add_ldflag 'gitCommit' ${COMMIT_ID}
echo "${ldflags[*]-}"
go build -ldflags "${ldflags[*]-}" ${PRO_ROOT}/main.go -o ${PRO_ROOT}/main
```

## `go clean`

从软件包源目录中删除目标文件

```bash
go clean [clean flags] [build flags] [packages]
```

go 命令在临时目录中构建大多数对象, 因此 `go clean` 主要与其它工具或手动调用 `go build` 留下的对象文件有关

标志 | 解释
:---: | :---
`-i` | 删除相应的已安装的归档文件或二进制文件
`-n` | 打印即将执行的删除命令, 但不运行
`-r` | 递归应用于包的所有依赖项
`-x` | 在执行删除命令时打印它们
`-cache` | 删除所有 `go build` 缓存
`-modcache` | 删除所有模块下载缓存, 包括源代码的版本依赖

## `go doc`

显示帮助文档

```bash
go doc [-u] [-c] [package|[package.]symbol[.methodOrField]]
```

打印指定对象的注释文档(package, const, func, type, var, method, struct), 然后打印其下属对象的单行摘要

标志 | 解释
:---: | :---
`-all` | 显示包所有的帮助文档
`-c` | 匹配符号时要注意大小写
`-cmd` | 将 command (package main) 视为常规程序包, 否则, 显示软件包的顶级文档时, 软件包主要的导出符号将被隐藏
`-src` | 显示该链接的完整源代码
`-u` | 显示未导出以及已导出符号, 方法和字段的文档

## `go env`

输出 Go 环境变量信息

```bash
go env [-json] [-u] [-w] [var ...]
```

默认情况下, env 将信息作为 shell 脚本输出(在 Windows 中为批处理文件). 如果给定一个或多个变量的名称作为参数, 则输出每个变量的值

标志 | 解释
:---: | :---
`-json` | 以 JSON 格式输出环境变量
`-u <arg> ...` | 会取消 `go env -w NAME=VALUE` 设置的指定环境变量
`-w <NAME=VALUE> ...` | 修改默认环境变量值(不能新添自定义环境变量)

## `go fix`

更新软件包以使用新的 API

```bash
go fix [packages]
```

## `go fmt`

格式化 go 代码包, 在指定包中运行 `gofmt -l -w`. 返回已修改文件的名称

```bash
go fmt [-n] [-x] [packages]
```

标志 | 解释
:---: | :---
`-n` | 只打印将要执行的命令
`-x` | 打印并执行命令

## `go get`

```bash
go get [-d] [-f] [-t] [-u] [-v] [-fix] [-insecure] [build flags] [packages]
```

下载指定包及其依赖项, 然后安装它

标志 | 解释
:---: | :---
`-d` | 只下载包, 不要安装
`-f` | 仅在 `-u` 设置后才有效, 强制 `-u` 不验证每个包是否已从其导入路径所隐含的代码仓库导出. 如果是本地的 fork 源文件将很有用
`-fix` | 在解决依赖关系或构建代码之前, 先对下载的包运行 fix 工具
`-insecure` | 允许使用非安全方式从代码库中获取信息并解析自定义域名. 谨慎使用
`-t` | 对包生成测试所需的模块
`-u` | 使用网络更新指定的包及其依赖关系. 默认情况下, 使用网络检查丢失的软件包, 而不使用它来查找对现有包的更新
`-v` | 启用输出详细进度和调试信息

checkout 新的包时, get 将创建目录 `GOPATH/src/<import-path>`. 如果 `GOPATH` 包含多个条目, 则默认使用第一个. 详见 `go help gopath`.

在 checkout 或更新软件包时, get 查找与本地安装的 Go 版本匹配的 brach 或 tag. 如果不存在这样的版本, 将获取包的默认分支.

当 go get checkout 或更新 git 仓库时, 它还会更新该存储库引用的所有 git 子模块

get 永远不会 checkout 或更新保存在 vendor 目录中的代码

## `go install`

编译并安装指定的软件包

```bash
go install [-i] [build flags] [packages]
```

可执行文件安装在环境变量 `GOBIN` 所命名的目录中, 默认为 `$GOPATH/bin`. 如果没有设置 `GOPATH`, 则设置为 `$HOME/go/bin`. `$GOROOT` 中的可执行文件安装在 `$GOROOT/bin` 或 `$GOTOOLDIR` 中, 而不是 `$GOBIN`.

禁用 module-aware, 其它包安装在 `$GOPATH/pkg/$GOOS_$GOARCH`. 启用 module-aware, 将构建并缓存其它包, 但不安装.

标志 | 解释
:---: | :---
`-i` | 同时安装指定包的依赖

## `go list`

列出指定包或其子包包名，每行一个

```bash
go list [-f format] [-json] [-m] [list flags] [build flags] [packages]
```

默认打印包的导入路径, 每行一个. 使用 `...` 可以输出指定包及其子包. 如 `go list net/...` 列出 "net" 及其子包的导入路径, 每行一个

标志 | 解释
:---: | :---
`-f` | 使用包模版的语法指定列出包的输出格式
`-json` | 使用 json 格式输出列出的包的信息
`-deps` | 遍历包名及其依赖, 依赖以深度优先的后续遍历访问它们.
`-export` | 将给定包的信息导出到包的 `Export` 字段设置的文件名
`-find` | 列出指定的包, 但不解析其依赖关系, `Imports` 和 `Deps` 列表为空
`-test` | 列出指定的包及其测试二进制文件, 以将测试二进制文件的确切构造传达给代码分析工具
`-m` | 列出模块 , 而不是包

最常用的标志是 `-f` , 它根据包和模块定义信息, 按照指定格式输出所需信息. 如 `'\{\{ .ImportPath }}'`(删除反斜线) 显示包的导入路径(默认), `'\{\{ .Name }}'`(删除反斜线)显示包的名称等.

包的详细定义信息见如下结构体

```go
type Package struct {
        Dir           string   // 包含包源代码的目录
        ImportPath    string   // 包在目录中的导入路径, `go list` 的默认输出格式
        ImportComment string   // package语句的import注释中的路径
        Name          string   // 包名称
        Doc           string   // 包文档字符串
        Target        string   // 安装路径
        Shlib         string   // 包含此包的共享库（仅在-linkshared时设置）
        Goroot        bool     // 这个包在GOROOT目录下吗?
        Standard      bool     // 这个包是标准go库的一部分吗?
        Stale         bool     // `go install`对这个包有什么作用码?
        StaleReason   string   // explanation for Stale==true
        Root          string   // 包含这个包的GOROOT或GOPATH目录
        ConflictDir   string   // $GOPATH中的这个目录shadows dir
        BinaryOnly    bool     // binary-only package (no longer supported)
        ForTest       string   // package is only for use in named test
        Export        string   // file containing export data (when using -export)
        Module        *Module  // info about package's containing module, if any (can be nil)
        Match         []string // command-line patterns matching this package
        DepOnly       bool     // package is only a dependency, not explicitly listed

        // Source files
        GoFiles         []string // .go source files (excluding CgoFiles, TestGoFiles, XTestGoFiles)
        CgoFiles        []string // .go source files that import "C"
        CompiledGoFiles []string // .go files presented to compiler (when using -compiled)
        IgnoredGoFiles  []string // .go source files ignored due to build constraints
        CFiles          []string // .c source files
        CXXFiles        []string // .cc, .cxx and .cpp source files
        MFiles          []string // .m source files
        HFiles          []string // .h, .hh, .hpp and .hxx source files
        FFiles          []string // .f, .F, .for and .f90 Fortran source files
        SFiles          []string // .s source files
        SwigFiles       []string // .swig files
        SwigCXXFiles    []string // .swigcxx files
        SysoFiles       []string // .syso object files to add to archive
        TestGoFiles     []string // _test.go files in package
        XTestGoFiles    []string // _test.go files outside package

        // Cgo directives
        CgoCFLAGS    []string // cgo: flags for C compiler
        CgoCPPFLAGS  []string // cgo: flags for C preprocessor
        CgoCXXFLAGS  []string // cgo: flags for C++ compiler
        CgoFFLAGS    []string // cgo: flags for Fortran compiler
        CgoLDFLAGS   []string // cgo: flags for linker
        CgoPkgConfig []string // cgo: pkg-config names

        // Dependency information
        Imports      []string          // import paths used by this package
        ImportMap    map[string]string // map from source import to ImportPath (identity entries omitted)
        Deps         []string          // all (recursively) imported dependencies
        TestImports  []string          // imports from TestGoFiles
        XTestImports []string          // imports from XTestGoFiles

        // Error information
        Incomplete bool            // this package or a dependency has an error
        Error      *PackageError   // error loading package
        DepsErrors []*PackageError // errors loading dependencies
}
```

模块的详细定义信息见如下结构体:

```go
type Module struct {
    Path      string       // 模块路径
    Version   string       // 模块版本
    Versions  []string     // 可用的模块版本 (with -versions)
    Replace   *Module      // 替换为指定模块
    Time      *time.Time   // 创建的时间版本信息
    Update    *Module      // 可用的任何更新 (with -u)
    Main      bool         // 是否是主模块
    Indirect  bool         // 是否只是主模块的间接依赖吗?
    Dir       string       // 模块的保存文件目录
    GoMod     string       // go.mod 文件路径
    GoVersion string       // 模块使用的 Go 版本信息
    Error     *ModuleError // 加载模块的错误
}

type ModuleError struct {
    Err string // 错误信息
}
```

## `go mod`

提供对模块的操作

```bash
go mod <command> [arguments]
```

包含如下子命令

command | 解释
:---: | :---
download | 下载模块到本地缓存(默认为 GOPATH/pkg/mod 目录)
edit | 通过工具或脚本编辑 go.mod
graph | 打印模块依赖图
init | 在当前目录下初始化新模块, 创建 go.mod
tidy | 增加缺少的包,删除没有用到的包
vendor | 将依赖复制到 vendor 目录下
verify | 校验依赖
why | 解释为什么需要软件包或模块

### `download`

下载指定模块, 这些模块是主模块依赖的模块, 也可以是 `path@version` 形式的模块查询. 如果没有参数, 则下载主模块的所有依赖.

```bash
go mod download [-json] [modules]
```

go 命令在正常执行期间根据需要自动下载模块. `go mod download` 命令主要用于预填充本地缓存或得到 go 模块代理的响应

标志 | 解释
:---: | :---
`-json` | 打印一系列 json 对象到标准输出, 模数每个下载的模块信息. 与下面 Go Module 结构体相对应

```go
type Module struct {
    Path     string // module path
    Version  string // module version
    Error    string // error loading module
    Info     string // absolute path to cached .info file
    GoMod    string // absolute path to cached .mod file
    Zip      string // absolute path to cached .zip file
    Dir      string // absolute path to cached source root directory
    Sum      string // checksum for path, version (as in go.sum)
    GoModSum string // checksum for go.mod (as in go.sum)
}
```

### `edit`

提供了一个用于编辑 go.mod 的命令行界面, 主要供工具或脚本使用. 它只读取 go.mod, 不查找所涉及的模块信息.

默认情况下, edit 读写主模块的 go.mod 文件,但可以指定其它目标文件

标志 | 解释
:---: | :---
`-fmt` | 重新格式化 go.mod 文件, 而不进行其它更改
`-module` | 更改模块的路径
`-require=path@version`/`-droprequire=path` | 添加给定的 `path@version` 的模块/删除指定 path 的模块
`-exclude=path@version`/`-dropexclude=path@version` | 为给定的 `path@version` 模块添加/删除 exclude 信息
`-replace=old[@v]=new[@v]`/`-dropreplace=old[@v]` | 添加和删除对给定 `path@version` 模块的替换
`-go=version` | 设置预期的 Go 语言版本
`-print` | 以文本格式打印最终的 go.mod, 而不是将其写回到 go.mod
`-json` |  以 JSON 格式打印最终的 go.mod, 而不是将其写回到 go.mod. JSON 格式对应的 Go 结构体如下

```go
type Module struct {
    Path string
    Version string
}

type GoMod struct {
    Module  Module
    Go      string
    Require []Require
    Exclude []Module
    Replace []Replace
}

type Require struct {
    Path string
    Version string
    Indirect bool
}

type Replace struct {
    Old Module
    New Module
}
```

### `init`

初始化一个新的 go.mod 保存在当前目录, 实际上是创建了一个以当前目录为 `/` 的新模块. go.mod 文件必须不存在

```bash
go mod init [module]
```

### `tidy`

```bash
go mod tidy [-v]
```

确保 go.mod 与模块中的源代码匹配, 它添加了构建当前模块的程序包和依赖项所需的所有缺少的模块, 并删除程序包的未使用模块. 它还会将所有缺少的条目添加到 go.sum 中, 并删除所有不必要的条目.

`-v` 标志将有关已删除模块的信息打印为标准错误

## `vendor`

```bash
go mod vendor [-v]
```

用于重置主模块的 vendor 目录, 以使其包含主模块构建和测试所需要的所有包. 它不包括已 vendored 软件包的测试代码.

`-v` 标志将已 vendored 模块和包因为到标准错误

## `go run`

编译并运行指定的 Go 主程序包

```bash
go run [build flags] [-exec xprog] package [arguments...]
```

通常, 该软件包被指定为来自单个目录的 .go 源文件列表, 但也可以是导入路径, 文件系统路径或模式匹配单个已知包.

默认情况下, `go run` 直接运行编译的二进制文件. 如果添加 `-exec` 标志, 那么 `go run` 使用 xprog 调用二进制文件. 如果未提供 `-exec` 标志, 则 GOOS 或 `GOARCH` 与系统不同的情况下, 可以找到名为 `go_$GOOS_$GOARCH_exec` 的程序文件

## `go test`

自动测试由导入路径命名的包, 输出测试结果的摘要信息

```bash
go test [build/test flags] [packages] [build/test flags & test binary flags]
```

`go test` 重新编译每个包以及文件名与 `*_test.go` 模式匹配的文件. 这些附加文件可以包含测试函数,压力测试和示例函数. 每个列出的包都会导致执行单独的测试二进制文件. 文件名以 `_` 或 `.` 开头的文件(包括`_test.go`)将被忽略

用后缀 `_test` 声明包的测试文件将被编译为单独的包, 然后与主测试二进制链接并运行. go 工具将会忽略名为 testdata 的目录, 使其可以保存测试所需的辅助数据

go test 有两种不同的运行模式:

- 本地目录模式: 在没有指定 package 包参数的情况下调用 go test 采用的模式. 在此模式下, go test 编译当前目录中包的源码和测试文件, 然后运行生成的二进制测试文件. 在此模式下, 将禁用缓存. 测试完成后, 输出测试的摘要信息, 包括测试状态(ok 或 FAIL), 包名和测试时间
- 包列表模式: 在显示指定 package 包参数的情况下调用 go test 采用的模式. 在此模式下, go test 编译并测试命令行中列出的每个包, 如果测试通过, 则只输出 ok 摘要行. 否则, 输出完整的测试输出.

在包列表模式下, go test 缓存成功的测试结果, 避免运行不必要的重复测试.

标志 | 解释
:---: | :---
`-args` | 将命令行其余部分(-args 之后的所有内容)传递给测试二进制文件. 一般放在所有参数最后
`-c` | 将测试二进制文件编译为 `pkg.test`, 但不运行它(pkg 是包导入路径的最后一个元素). 可以使用 `-o` 更改文件名
`-exec xprog` | 使用 xprog 运行测试二进制文件
`-i` | 安装测试依赖项的包, 不运行测试
`-json` | 将测试输出转换为适合自动处理的 JSON 文本
`-o file` | 将测试二进制文件编译为指定命名的文件, 仍然运行测试(除非指定了 -c 或 -i)
