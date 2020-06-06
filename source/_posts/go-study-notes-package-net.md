---
title: go 学习笔记之 net 包
date: 2020/05/10
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 63612
description: '本文章主要包含 Go net 包提供的内置类型,接口,函数方法的使用,使用 net 包可以建立一个简单的客户端与服务端通信.'
---

`net` 包为网络 I/O 提供了可移植的接口,包括 TCP/IP,UDP,域名解析和 Unix 套接字等.

该软件包提供了对底层网络原语的访问,大多数客户端仅需要 `net` 包提供的 `Dial` 和 `Listen` 函数, `Conn` 和 `Listener` 接口相关的的 `Accept` 方法. `crypto/tls` 软件包使用相同的接口提供 `Dial` 和 `Listen` 函数.

客户端与服务端最简单的通信主要代码示例如下:

```go
// 省略部分内容...
// 服务端,监听 tcp localhost:8080 端口
ln, err := net.Listen("tcp", "localhost:8080")
if err != nil {
    // handle error
}
for {
    conn, err := ln.Accept()  // 一直接收连接请求
    if err != nil {
        // handle error
    }
    go handleConnection(conn)  // 处理连接的函数
}

// 客户端调用 Dial 函数与 tcp localhost:8080 建立连接.
conn, err := net.Dial("tcp", "localhost:8080")
if err != nil {
    // handle error
}
// 向 conn 写入数据,也就是向服务端发送请求,相关数据会交给服务端处理
fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
// 服务端处理后的数据会写入 conn,客户端就可以从中读取相关响应
status, err := bufio.NewReader(conn).ReadString('\n')  
// ...
```

> 域名解析

域名解析相关方法因操作系统而异.

Unix 系统上,可以使用纯 Go 解析器将 DNS 请求直接发送到 `/etc/resolv.conf` 列出的两个 DNS 服务器,也可以使用 cgo 的解析器调用 C 语言库,如 getaddrinfo 和 getnameinfo.
默认情况下使用纯 Go 解释器,可通过如下方式进行修改
```bash
export GODEBUG=netdns=go    # 强制使用 Go 解析器
export GODEBUG=netdns=cgo   # 强制使用 cgo 解析器
```

Windows 系统上,解析器始终调用 C 语言库函数,如 GetAddrInfo 和 DnsQuery

# 常用类型定义

```go
// 表示网络端点地址
// 该接口的实现包括 IPAddr,IPNet,TCPAddr,UDPAddr,UnixAddr
type Addr interface {
    Network() string // 网络类型
    String() string  // 地址+端口
}
// 通用网络连接,多个 goroutine 可以安全调用 Conn 上的方法
// 该接口的实现包括 tls.Conn,IPConn,TCPConn,UDPConn,UnixConn
type Conn interface {
    Read(b []byte) (n int, err error) // 从连接中读数据
    Write(b []byte) (n int, err error) // 向连接中写数据
    Close() error // 关闭连接
    LocalAddr() Addr // 连接的本地地址
    RemoteAddr() Addr // 连接的远程地址
    SetDeadline(t time.Time) error // 设置连接的截止时间,0 表示不会超时
    SetReadDeadline(t time.Time) error
    SetWriteDeadline(t time.Time) error
}
// 监听器接口定义
// 该接口的实现包括 TCPListener,UnixListener
type Listener interface {
    Accept() (Conn, error)  // 等待连接
    Close() error  // 关闭监听
    Addr() Addr  // 返回监听者的地址
}
```
# 常用函数

# `net` 包函数

```go
// 向给定的 network, address 发起连接
func Dial(network, address string) (Conn, error)
func DialTimeout(network, address string, timeout time.Duration) (Conn, error)

// 在给定的 network, address 进行监听.返回监听的实例对象
func Listen(network, address string) (Listener, error)

// 解析给定 host 的 IP 地址,返回 IP 列表
func LookupIP(host string) ([]IP, error)
// 将给的字符串解析为 IP 对象
func ParseIP(s string) IP
```

# 示例

## 简单通信

- server.go

```go
import (
    "fmt"
    "net"
)

func main() {
    listener, err := net.Listen("tcp", "localhost:9090")
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("server listen %v\n", listener.Addr())
    for {
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println(err)
            break
        }
        go handleConn(conn)
    }
}
func handleConn(conn net.Conn) {
    fmt.Printf("conn established from remote %v\n", conn.RemoteAddr())
    buf := make([]byte, 1024)
    for {
        rn, err := conn.Read(buf)
        if err != nil {
            fmt.Println(err)
            break
        }
        message := string(buf[:rn])
        fmt.Printf("client send: %v\n", message)
        response := "response:" + message
        conn.Write([]byte(response))
    }
}
```

- client.go

```go
import (
    "fmt"
    "net"
)

func main() {
    conn, err := net.Dial("tcp", "localhost:9090")
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("conn from %v to %v\n", conn.LocalAddr(), conn.RemoteAddr())

    conn.Write([]byte("send message"))
    buf := make([]byte, 1024)
    n, err := conn.Read(buf)
    fmt.Printf("recv response: %v", string(buf[:n]))
}
```


