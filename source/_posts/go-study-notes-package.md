---
title: go 学习笔记之包管理
date: 2020/05/04
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 19238
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中包的定义及其依赖的管理方式.'
---

## 工作空间

工作空间通常在 `GOPATH` 环境变量列表中定义(可通过 `go env` 查看), 由 src, bin, pkg 三个目录组成. `src` 用于保存源码文件, `bin,pkg` 主要影响 `go install/get` 命令, 它们将编译结果(可执行文件或静态库)安装到这两个目录下, 实现增量编译.

编译器按照 GOPATH 设置的路径搜索目标文件. 不同操作系统, GOPATH 列表分割符不同, Unix 使用 `:`, Windows 使用 `;`. `go get` 默认将下载的第三方包保存在列表的第一个工作空间内.

## 包

Go 语言使用包(package)来组织源代码, 包(package)是多个 Go 源码的集合.

任何源文件必须属于某个包, 同时源文件的第一行有效代码必须通过 `package pacakgeName ` 语句声明该源文件所属的包.

一般来说, 包有以下特点或注意事项:

- 包名一般小写, 使用简短且有意义的名称
- 包名一般要和所在的目录同名, 也可以不同, 包名中不能包含 `-` 等特殊符号
- 包一般使用域名作为目录名称,这样能保证包名的唯一性, 比如 GitHub 项目的包一般会放到 `GOPATH/src/github.com/userName/projectName` 目录下
- 包名为 `main` 的包为应用程序的入口包, 编译不包含 main 包的源码文件时不会得到可执行文件
- 一个文件夹下的所有源码文件只能属于同一个包, 同样属于同一个包的源码文件不能放在多个文件夹下

### 导入包

要在代码中使用其它包, 需要使用 `import` 关键字导入. `import` 语句通常在包声明语句下面. 语法为:

```go
import "path/to/package"
// 或
import (
    "path/to/package1"
    //"path/to/package2"
)
```

一般情况下, 要导入的包名路径需要使用 `""` 包裹, 且包名是以 `GOPATG/src/` 后开始计算的, 使用 `/` 进行路径分割. 如, `import "custom/lib"` 表示将 `GOPATH/src/custom/lib` 包中内容导入.

包支持 4 中引用格式:

1. 标准引用格式: `import "fmt"`
2. 自定义别名引用: `import f "fmt"`, 此时我们可以使用 `f.` 代替 `fmt.` 调用 fmt 包中的方法. 别名引用也多用于导入包重名或包名过长的情况
3. 省略引用格式: `import . "fmt"`, 此时我们可不使用 `fmt.` 而直接调用 `fmt` 包内的方法, 如 `fmt.Println()` -> `Println()`
4. 匿名格式引用: `import _ "fmt"`, 此时我们只是希望执行包内的初始化 `init()` 函数, 而不使用包的内容

在导入包时, 有以下几点需要注意:

- 而未使用的包导入会被编译器视为错误
- 一个包可以有多个 `init()` 函数, 包加载时会执行全部的 `init()` 函数, 但并不能保证执行顺序, 所以不建议在一个包中放入多个 `init()` 函数，而是将需要初始化的逻辑放到一个 `init()` 函数里面.
- 包不能出现环形引用的情况, 比如包 a 引用了 b, b 引用了 c, c 又引用了 a, 此时编译不能通过
- 包可以被重复引用, 且 Go 编译器能够保证被重复引用包的 `init()` 函数只会执行一次
- **所有保存在 `internal` 目录下的包(包括自身)仅能被其父目录下的包(含所有层次的子目录)访问**

```
src/
|
+-- main.go
|
+-- lib/            # 内部包 internal/{a, b} 仅能被 lib, lib/x, lib/x/y 访问
|    |
|	 +-- internal/  # 内部包之间可相互访问
|	 |      |
|    |      +-- a/  # 可以导入外部包, 如 lib/x/y
|	 |      |
|    |      +-- b/
|	 |
|    +-- x/
|        |
|        +-- y
|
+-- z/              # z 包不可导入 lib/internal 下所有内容, 但可以导入 lib/x 包
```

在执行 main 包的 main 函数之前, Go 程序先对整个程序的包进行初始化. 包内的源码文件都可以定义一到多个初始化函数, 编译器首先确保完成所有全局变量初始化, 然后开始执行 `init()` 初始化函数,直到这些全部结束后, 运行时才进入 `main.main` 入口函数.

1. 从 main 函数引用的包开始, 逐级查找包的引用, 直到找到没有引用其它包的包
2. 单个包在初始化过程中, 先初始化常量, 然后是全局变量, 最后执行包的 init 函数

![Go 包的初始化](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/go-study-notes-package/package_initialization_process_in_go.jpg)

### 包的封装

所有成员在包内均可访问, 无论是否在同一源码文件中. 但只有名称首字母大写的成员为可导出成员, 可在包外访问, 类似于 java 中的 `public`. 名称首字母小写的成员不可导出, 仅能在本包中访问, 类似于 `private`.

可通过提供可导出的工厂模式的函数, 对外提供一个构造函数, 同时对其属性对外提供 Get/Set 方法.
```go
// src/model/person.go
package model

type person struct {
    Name string
    age int // 其它包不能直接访问
}

func NewPerson(name string) *person {
    return &person{name}
}

func (p *person) SetAge(age int) {
    p.age = age
}
func (p *person) GetAge() int {
    return p.age
}
```
```go
// src/main.go
package main

import (
    "fmt"
    "model"
)
func main() {
    p := model.NewPerson("smith")
    p.SetAge(18)
    fmt.Println(p)
    fmt.Println(p.Name, " age =", p.GetAge())
}
```

## 依赖管理

Go Modules 是 Go 语言的一种依赖管方式, 该功能是在 Go 1.11 版本中出现的, 且在 Go 1.12 版本不断改进, Go 1.13 版本完善优化后, 演变为官方推荐的包依赖管理方式.

Go Modules 使用 `go.mod` 和 `go.sum` 管理程序中第三方包的依赖. 

`go.mod` 定义了模块的名称(可用作导入的包名), Go 的版本以及第三方包模块的依赖及版本等信息, 并提供了`module`, `require`, `replace` 和 `exclude` 四个命令来实现 go module 的功能.

- `module` 语句指定包的名字(路径)
- `require` 语句指定的依赖项模块
- `replace` 语句指定替换依赖项模块
- `exclude` 语句指定可以忽略依赖项模块


[github.com/gin-gonic/gin](https://github.com/gin-gonic) 示例如下:  
```
module github.com/gin-gonic/gin

go 1.13

require (
	github.com/gin-contrib/sse v0.1.0
	github.com/go-playground/validator/v10 v10.2.0
	github.com/golang/protobuf v1.3.3
	github.com/json-iterator/go v1.1.9
	github.com/mattn/go-isatty v0.0.12
	github.com/stretchr/testify v1.4.0
	github.com/ugorji/go/codec v1.1.7
	gopkg.in/yaml.v2 v2.2.8
)
```

`go.sum` 记录每个依赖包的版本和哈希值, 示例文件如下所示:

```
github.com/davecgh/go-spew v1.1.0/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/davecgh/go-spew v1.1.1 h1:vj9j/u1bqnvCEfJOwUhtlOARqs3+rkHYY13jYWTU97c=
github.com/davecgh/go-spew v1.1.1/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/gin-contrib/sse v0.1.0 h1:Y/yl/+YNO8GZSjAhjMsSuLt29uWRFHdHYUb5lYOV9qE=
// ...
```

那么如何使用 Go Modules 呢?

### 启用 go module 功能

- go 版本 >= v1.11
- 设置 `GO111MODULE` 环境变量为 on. 

`GO111MODULE` 有三个值, 默认为 `auto`
- `on`: 启用 go module, 编译时会忽略 GOPATH 和 vendor 文件夹，只根据 go.mod 下载依赖
- `off`: 禁用 go module, 编译时会从 GOPATH 和 vendor 文件夹中查找包
- `auto`: 当项目在 GOPATH/src 目录之外, 并且项目根目录有 go.mod 文件时，开启 go module

可以通过设置 `GOPROXY` 环境变量设置获取依赖包时的代理.

```
go env -w GOPROXY=https://goproxy.cn,direct
或
export GOPROXY=https://goproxy.cn
```
目前公开的代理服务地址有: 

```
https://goproxy.io/
https://goproxy.cn/
https://mirrors.aliyun.com/goproxy/
````

### 使用 go module 功能

```
# 初始化 go module, 创建一个 go.mod 文件,包含 module 信息和 go 版本信息
go mod init  [module_name]
# 自动将包及其依赖打包成 module, 并修改 go.mod 文件, 增加缺少的包,删除没有用到的包
go mod tidy
```