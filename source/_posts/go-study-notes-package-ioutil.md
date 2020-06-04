---
title: go 学习笔记之 ioutil 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 10272
description: 本文章主要包含 Go ioutil 包及其内置类型和方法的使用.ioutil 包提供了一些基本 IO 操作的函数
---

`ioutil` 包是 `io` 包的子包,它提供了一些基本 IO 操作的函数.导入方式为 `import "io/ioutil"`

# 常用函数

以下是 `ioutil` 包中常用的函数

```go
// 从 r 读取所有数据,直到出现错误或 EOF.返回读取的数据及可能出现的错误
func ReadAll(r io.Reader) ([]byte, error)
// 读取指定目录下的文件.返回目录下按文件名排序的 `os.FileInfo` 列表及可能出现的错误.
// 只能读取一层.
func ReadDir(dirname string) ([]os.FileInfo, error)
// 返回读取的指定文件内容及可能出现的错误
func ReadFile(filename string) ([]byte, error)

// 在指定 dir 目录下创建带有 prefix 前缀的随机临时目录.返回创建的随机临时目录路径名及可能发生的错误
// dir 如果为空,则设定为 `os.TempDir()`.
func TempDir(dir, prefix string) (name string, err error)
// 在指定 dir 目录下创建带有 pattern 前缀的随机临时目录.返回创建的随机文件的 `*os.File` 及可能发生的错误
// dir 如果为空,则设定为 `os.TempDir()`.
func TempFile(dir, pattern string) (f *os.File, err error)
// 将 data 写入指定 filename.返回可能发生的错误
// 如果文件不存在,则以 perm 权限创建该文件;否则清空文件
func WriteFile(filename string, data []byte, perm os.FileMode) error
```

# 示例

## 拷贝文件

```go
import (
    "fmt"
    "io/ioutil"
)

func copyFileExample(src, dest string) (err error) {
    data, err := ioutil.ReadFile(src)
    err = ioutil.WriteFile(dest, data, 0)
    return
}

func main() {
    err := copyFileExample("src", "dest")
    if err != nil {
        fmt.Println(err)
    }
}
```

## 读取指定目录及其子目录所有文件

```go
import (
    "fmt"
    "io/ioutil"
    "os"
    "strings"
)

func findFile(dir string) {
    PathSeparator := string(os.PathSeparator)
    fileInfos, err := ioutil.ReadDir(dir)
    if err != nil {
        return
    }
    for _, fileInfo := range fileInfos {
        if fileInfo.IsDir() {
            dirname := strings.Join([]string{dir, fileInfo.Name()}, PathSeparator)
            findFile(dirname)
        } else {
            fmt.Printf("%v%v%v\n", dir, PathSeparator, fileInfo.Name())
        }
    }

}
func main() {
    findFile(".")
}
```