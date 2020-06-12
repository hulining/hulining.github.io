---
title: go 学习笔记之 io 包
date: 2020/05/05
tags:
  - go
categories:
  - go
abbrlink: 6060
description: '本文章主要包含 Go io 包及其内置类型和方法的使用.io 包提供了对 IO 操作的基本接口,要封装了一些已有的实现.'
---

`io` 包提供了对 IO 原语的基本接口. 它主要封装了一些已有的实现(如 os 包中的),并将这些抽象为使用性的功能和一些其它相关接口.

## 常用类型定义

```go
// 读相关接口
type Reader interface {
     // 从数据流里将 len(p) 个字节读入 p.返回读取的字节数(0<=n<=len(p)和可能发生的错误.在文件读取时, 返回 0和 io.EOF 表示读取到文件结尾
    Read(p []byte) (n int, err error)
}

type ReaderAt interface {
    // 从数据流里 off 位置开始将 len(p) 个字节读入 p.返回读取的字节数(0<=n<=len(p)和可能发生的错误
    ReadAt(p []byte, off int64) (n int, err error)
}

type ReaderFrom interface {
    // 从 r 读取数据直到发生错误或文件结束(EOF). 返回读取的字节数 n 及遇到的错误(EOF除外)
    ReadFrom(r Reader) (n int64, err error)
}

type ByteReader interface {
    // 返回读取的单个字节及可能发生的错误
    ReadByte() (c byte, err error)
}

// 写相关接口
type Writer interface {
    // 将 p 的 len(p) 个字节写入数据流.返回写入的字节数(0<=n<=len(p)和可能发生的错误.
    Write(p []byte) (n int, err error)
}

type WriterAt interface {
    // 从数据流 off 位置开始,将 p 的 len(p) 个字节写入数据流.返回写入的字节数(0<=n<=len(p)和可能发生的错误.
    WriteAt(p []byte, off int64) (n int, err error)
}

type WriterTo interface {
    // 写入数据到 w 直到发生错误.返回读取的字节数 n 及遇到的错误
    WriteTo(w Writer) (n int64, err error)
}

type StringWriter interface {
    // 将字符串 s 写入到数据流中,返回写入的字节数(0<=n<=len(s)和可能发生的错误
    WriteString(s string) (n int, err error)
}

// 读写接口
type ReadWriter interface {
    Reader
    Writer
}

//关闭接口
type Closer interface {
    // 关闭数据流
    Close() error
}

// 读写指针移位接口
type Seeker interface {
    // 设置下一次读写的位置,whence 设置参照点,offset 设置偏移量.
    // whence 可选值为 `io.SeekStart`(0,文件开始),`io.SeekCurrent`(1,当前位置),`io.SeekEnd`(2,结尾位置).
    Seek(offset int64, whence int) (int64, error)
}
```

## 常用常量及变量

```go
const (
    SeekStart   = 0 // 相对于文件开始位置
    SeekCurrent = 1 // 相对于文件当前位置
    SeekEnd     = 2 // 相对于文件结尾位置
)

// 当无法得到更多输入时,Read 方法返回 EOF.可以作为文件读取结束的表示
var EOF = errors.New("EOF")  
```

## 常用方法

## `io` 包方法

```go
// 将 src 的数据拷贝到 dst.返回拷贝的字节数和可能发生的错误
// 底层调用了 `CopyBuffer(dst, src, nil)`, 使用默认的缓冲区,默认大小为 32 * 1024
func Copy(dst Writer, src Reader) (written int64, err error)
// 从 src 拷贝 n 个字节数据到 dst,直到发生错误或 EOF.返回复制的字节数和可能发生的错误
func CopyN(dst Writer, src Reader, n int64) (written int64, err error)
// 将 src 的数据拷贝到 dst.返回拷贝的字节数和可能发生的错误. buf 可以指定指定缓冲区的大小
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error)
// 将 r 中 min 个字节读取到 buf 中.返回复制的字节数及可能发生的错误
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)
// 将 r 中 len(buf) 个字节读取到 buf 中.返回复制的字节数及可能发生的错误
// 其实是调用了,ReadAtLeast(r, buf, len(buf)).如果 n<len(buf),则产生非 io.EOF 错误
func ReadFull(r Reader, buf []byte) (n int, err error)
//  将字符串 s 写入 w 中.返回写入的字节长度及可能发生的错误
//  如果 w 实现了 StringWriter 接口, `w.WriteString()` 将直接被调用.参见 https://github.com/golang/go/blob/master/src/io/io.go#L293
func WriteString(w Writer, s string) (n int, err error)
```

## 示例

- 拷贝文件

```go
import (
    "fmt"
    "io"
    "os"
)

func copyFileExample(src, dest string) (err error) {
    srcFile, err := os.Open(src)
    defer srcFile.Close()
    destFile, err := os.OpenFile(dest, os.O_CREATE|os.O_RDWR|os.O_SYNC, 0)
    defer destFile.Close()
    _, err = io.Copy(destFile, srcFile)
    return
}

func main() {
    err := copyFileExample("src", "dest")
    if err != nil {
        fmt.Println(err)
    }
}
```
