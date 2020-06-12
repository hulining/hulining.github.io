---
title: go 学习笔记之 os 包
date: 2020/05/04
tags:
  - go
categories:
  - go
abbrlink: 45949
description: 本文章主要包含 Go os 包及其内置类型和方法的使用.os 包提供了与平台无关的接口以便于为我们提供对系统进行操作函数.
---

`os` 包提供了与平台无关的接口以便于为我们提供对系统进行操作函数.导入方式为 `import "os"`

## 常用类型定义

```go
// 文件类型掩码封装
type FileMode uint32

// 文件对象结构体
type File struct {
    // 内含隐藏或非导出字段
}

// 文件信息接口
type FileInfo interface {
    Name() string       // 文件的名字（不含扩展名）
    Size() int64        // 普通文件返回值表示其大小,其他文件的返回值含义各系统不同
    Mode() FileMode     // 文件的模式位
    ModTime() time.Time // 文件的修改时间
    IsDir() bool        // 等价于Mode().IsDir()
    Sys() interface{}   // 底层数据来源（可以返回nil）
}
```

## 常用常量及变量

以下提供了 `os` 包中的常用常量,包含文件的类型权限,属性等

````go
const (
    // 单个字符是 String() 方法格式化的缩写
    ModeDir        FileMode = 1 << (32 - 1 - iota) // d: 目录
    ModeAppend                                     // a: 追加
    ModeExclusive                                  // l: 执行
    ModeTemporary                                  // T: 临时文件
    ModeSymlink                                    // L: 链接文件
    ModeDevice                                     // D: 设备文件
    ModeNamedPipe                                  // p: 管道文件(FIFO)
    ModeSocket                                     // S: Unix 套接字文件
    ModeSetuid                                     // u: setuid
    ModeSetgid                                     // g: setgid
    ModeCharDevice                                 // c: Unix 字符设备
    ModeSticky                                     // t: sticky
    ModeIrregular                                  // ?: 非常规文件

    // 类型的掩码,对于常规文件,设置为 none
    ModeType = ModeDir | ModeSymlink | ModeNamedPipe | ModeSocket | ModeDevice | ModeCharDevice | ModeIrregular

    ModePerm FileMode = 0777 // Unix 权限位
)
const (
    // 必须指定 O_RDONLY, O_WRONLY, O_RDWR 之一
    O_RDONLY int = syscall.O_RDONLY // 只读方式打开文件
    O_WRONLY int = syscall.O_WRONLY // 只写方式打开文件
    O_RDWR   int = syscall.O_RDWR   // 读写方式打开文件
    // 下面的值可以用来控制行为
    O_APPEND int = syscall.O_APPEND // 追加写入数据
    O_CREATE int = syscall.O_CREAT  // 文件不存在则创建
    O_EXCL   int = syscall.O_EXCL   // 与 O_CREATE 一起使用,文件必须不存在
    O_SYNC   int = syscall.O_SYNC   // 以同步 I/O 方式打开
    O_TRUNC  int = syscall.O_TRUNC  // 打开时清空文件内容
)
const (
    SEEK_SET int = 0 // 相对于文件开始位置,已过时,而使用 io.SeekStart
    SEEK_CUR int = 1 // 相对于文件当前位置,已过时,而使用 io.SeekCurrent
    SEEK_END int = 2 // 相对于文件结尾位置,已过时,而使用 io.SeekEnd
)
const (
    PathSeparator     = '/' // Unix 操作系统指定的路径分隔符
    PathListSeparator = ':' // Unix 操作系统指定的表分隔符
)
const (
    PathSeparator     = '\\' // Windows 操作系统指定的路径分隔符
    PathListSeparator = ';' // Windows 操作系统指定的表分隔符
)

var (
    Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
    Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
    Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

## 常用函数

### `os` 包函数

#### 系统相关

```go
// 返回系统主机名及可能发生的错误
func Hostname() (name string, err error)
// 返回环境变量的字符串副本,形式为 "key=value"
func Environ() []string
// 返回指定环境变量的值
func Getenv(key string) string  
// 设置环境变量(仅在当前进程生效),返回可能发生的错误
func Setenv(key, value string) error
// 删除当前进程的所有环境变量
func Clearenv()
// 使程序按照给定的状态码退出,程序会立即终止, defer 函数不会执行
func Exit(code int)
func Getuid() int  // 返回调用者的用户ID
func Getgid() int  // 返回调用者的组ID
func Getpid() int  // 返回当前程序的进程ID
func Getppid() int  // 返回当前进程的父进程ID
```

#### 文件相关

```go
// 返回描述指定的文件的 FileInfo 及可能发生的错误
func Stat(name string) (fi FileInfo, err error)
// 采用 0666 模式创建文件,如果文件存在则清空.返回文件描述符为 O_RDWR 的 IO 文件对象及可能发生的错误
func Create(name string) (file *File, err error)
// 以只读方式打开文件.返回文件描述符为 O_RDONLY 的只读文件对象及可能发生的错误
func Open(name string) (file *File, err error)
// 以指定的文件描述符 flag,指定的模式打开或创建文件.返回文件对象及可能发生的错误
func OpenFile(name string, flag int, perm FileMode) (file *File, err error)
// 判断在文件操作过程中发生的错误是否是文件已存在的错误
func IsExist(err error) bool
// 判断在文件操作过程中发生的错误是否是文件不存在的错误
func IsExist(err error) bool
// 判断在文件操作过程中发生的错误是否是因权限问题而引发的错误
func IsPermission(err error) bool
// 返回当前工作目录路径.默认工作目录为 $GOPATH/src
func Getwd() (dir string, err error)
// 将当前工作目录修改为执行目录,并返回可能发生的错误
func Chdir(dir string) error
// 修改指定文件权限.返回可能发生的错误
func Chmod(name string, mode FileMode) error
// 修改指定文件属主属组.返回可能发生的错误
func Chown(name string, uid, gid int) error
// 修改指定文件的访问时间和修改时间.返回可能发生的错误
func Chtimes(name string, atime time.Time, mtime time.Time) error
// 使用指定权限创建目录.如果上级目录不存存在,则会报错.返回可能发生的错误
func Mkdir(name string, perm FileMode) error
// 使用指定权限创建目录.如果上级目录不存存在,则会递归创建.返回可能发生的错误.
func MkdirAll(path string, perm FileMode) error
// 移动文件.返回可能发生的错误
func Rename(oldpath, newpath string) error
// 删除文件或目录.如果目录不为空,则报错.返回可能发生的错误
func Remove(name string) error
// 删除 path 及其目录下所有文件
func RemoveAll(path string) error
// 创建 newname 指向 oldname 的符号链.返回可能发生的错误
func Symlink(oldname, newname string) error
// 返回用于保存临时文件的默认目录.
// windows 下为 C:\Users\<Username>\AppData\Local\Temp, linux 下为 /tmp
func TempDir() string
```

### `File` 结构体方法

```go
// 返回文件的名称
func (f *File) Name() string
// 返回描述指定的文件的 FileInfo 及可能发生的错误
func (f *File) Stat() (fi FileInfo, err error)
// 返回文件的整数类型的Unix文件描述符
func (f *File) Fd() uintptr
// 修改文件权限,返回可能发生的错误
func (f *File) Chmod(mode FileMode) error
// 修改文件属主属组,返回可能发生的错误
func (f *File) Chown(uid, gid int) error
// 保留 size 大小的文件内容,多出的部分就会被丢弃.返回可能发生的错误
func (f *File) Truncate(size int64) error
// 从文件对象中最多读取 len(b) 字节数据并写入 b.返回读取的字节数和可能遇到的任何错误.文件终止标志是读取 0 个字节且err 为 io.EOF
func (f *File) Read(b []byte) (n int, err error)
// 从指定的位置读取len(b)字节数据并写入b
func (f *File) ReadAt(b []byte, off int64) (n int, err error)
// 向文件中写入len(b)字节数据,返回写入的字节数和可能发生的错误,如果 n!=len(b), nil 不为空
func (f *File) Write(b []byte) (n int, err error)
// 向文件中写入字符串,类似于 Write
func (f *File) WriteString(s string) (ret int, err error)
// 向文件指定位置写入字节数据,类似于 Write
func (f *File) WriteAt(b []byte, off int64) (n int, err error)
// 设置下一次读/写的位置,返回新的偏移量(相对于开头)及可能发生的错误.
// offset 为相对偏移量,whence 决定相对位置: 0为相对文件开头,1为相对当前位置,2为相对文件结尾
func (f *File) Seek(offset int64, whence int) (ret int64, err error)
// 将文件系统的最近写入的数据在内存中的拷贝刷新到硬盘中稳定保存
func (f *File) Sync() (err error)
// 关闭文件.返回可能发生的错误
func (f *File) Close() error
```

## 示例

### 文件读写示例

```go
import (
    "fmt"
    "os"
)

func main() {
    file, err := os.OpenFile("filename", os.O_RDWR|os.O_CREATE|os.O_SYNC|os.O_APPEND, 0)  // 打开文件方式为 读写,不存在则创建, 写同步, 追加
    if err != nil {
        fmt.Println("err", err)
        return
    }
    defer file.Close()

    // 文件写入
    var bytes = []byte{'a', 'b', 'c', 'd', '\n'}
    file.Write(bytes)  // 以 []bete 方式写入文件
    str := "this is string to write\n"
    var strBytes = []byte(str)
    file.Write(strBytes)
    file.WriteString(str)  // 向文件写入字符串
    // file.Sync() // 若 OpenFile 没有 os.O_TRUNC 标识,则需要显示同步,以上写入操作才会同步到文件中

    // 文件读取
    buffer := make([]byte, 1024)
    file.Seek(0, 0)  // 在写时,文件读取指针已经移到最后了,需要设置从文件开始位置开始读取. 如果是心打开的文件,可以不设置
    for {
        len, err := file.Read(buffer)  // 读取文件内容到 buffer 中
        if len == 0 && err != nil {
            break
        }
        fmt.Print(string(buffer[:len]))  // 输出文件内容
    }
}
```

`os` 包没有直接提供对文件按行读取的方法,我们需要借助 `{% post_link go-study-notes-package-time bufio%}` 包来实现
