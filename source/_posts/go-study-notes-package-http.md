---
title: go 学习笔记之 http 包
date: 2020/05/10
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 10194
description: '本文章主要包含 Go http 包提供的内置类型,接口,函数方法的使用.http 包提供了 HTTP 客户端和服务端的实现.'
---

`http` 包提供了 HTTP 客户端和服务端的实现.它提供了 HTTP 通信过程中各种对象的定义及实现.导入方式为 `import "net/http"`

对于客户端,它可以使用 `Get`,`Head`,`Post`和`PostForm` 等方法直接发送对应的 HTTP 请求,也可以通过 `Client` 类型自定义客户端,从而调用其中方法发送 HTTP 请求.

```go
// 直接发送请求,并接收响应
resp, err := http.Get("http://example.com/")
resp, err := http.PostForm("http://example.com/form",
    url.Values{"key": {"Value"}, "id": {"123"}})
// 客户端必须显式关闭响应体
defer resp.Body.Close()
```

```go
// 自定义 Client 实例对象,并使用该对象发送请求
client := &http.Client{
    CheckRedirect: redirectPolicyFunc,
}
resp, err := client.Get("http://example.com")

// 也可以预生成请求,然后由客户端发送
req, err := http.NewRequest("GET", "http://example.com", nil)
// ...
req.Header.Add("If-None-Match", `W/"wyzzy"`)
resp, err := client.Do(req)
// ...
```

对于传输过程,它提供了支持代理,TLS 配置,keep-alive,压缩等传输方式的 `Transport` 类型

```go
// 自定义 Transport 对象实例,并使参数在传输过程中生效
tr := &http.Transport{
    MaxIdleConns:       10,
    IdleConnTimeout:    30 * time.Second,
    DisableCompression: true,
}
client := &http.Client{Transport: tr}
resp, err := client.Get("https://example.com")
```

对于服务端,它提供了 `ListenAndServe` 函数使用给定的地址和处理程序启动 HTTP 服务器,还提供了 `Server` 类型用于自定义服务端及处理请求的函数

```go
http.Handle("/foo", fooHandler)
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})
log.Fatal(http.ListenAndServe(":8080", nil))
```

```go
s := &http.Server{
    Addr:           ":8080",
    Handler:        myHandler,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
}
log.Fatal(s.ListenAndServe())
```

`http` 包的 `Transport` 和 `Server` 类型对于简单的配置都自动支持 HTTP/2 协议.要使用更为复杂的 HTTP/2 协议,请直接导入 `golang.org/x/net/http2`,并使用其 `ConfigureTransport` 或 `ConfigureServer` 类型的相关函数.

## 常用类型定义

```go
// 客户端类型结构体定义,用于自定义客户端
type Client struct {
    // 发出 HTTP 请求机制.默认使用 `DefaultTransport`
    Transport RoundTripper
    // 指定重定向策略.如果不为 nil,则在 HTTP 重定向之前调用它.否则使用默认策略,在连续 10 个请求后停止
    // req 与 via 分别为即将到来的请求和已发出的请求.
    CheckRedirect func(req *Request, via []*Request) error
    Jar CookieJar  // 指定 CookieJar
    Timeout time.Duration  // 超时时间
}

// 处理请求的函数接口
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// 处理请求的函数
type HandlerFunc func(ResponseWriter, *Request)

// 请求头
type Header map[string][]string

// 请求对象
type Request struct {
    Method string  // 请求方法
    URL *url.URL  // 请求 URL
    Proto      string // 请求协议,对于客户端来说,默认是 HTTP/1.1 或 HTTP/2
    Header Header  // 请求头
    Body io.ReadCloser  // 请求体
    GetBody func() (io.ReadCloser, error) // 获取请求体的方法
    ContentLength int64  // 请求长度,-1 表示未知
    TransferEncoding []string
    Close bool  // 发送请求后是否关闭 TCP 连接
    Host string  // 请求主机,如果为空,则使用 URL.Host 的值

    // 以下三个属性需要调用`ParseForm` 或 `ParseMultipartForm` 后才可获取属性值
    Form url.Values  // 已解析的表单数据,包含 URL 参数,PATCH,POST,PUT 表单数据.
    PostForm url.Values // 已解析的表单数据,包含 PATCH,POST,PUT 表单数据
    MultipartForm *multipart.Form // 已解析的表单数据,主要用于文件上传

    RemoteAddr string  // 远程地址
    RequestURI string  // 请求 URI
    TLS *tls.ConnectionState  // TLS 连接信息
    Response *Response // 响应
    // contains filtered or unexported fields
}
// 响应对象
type Response struct {
    Status     string // 状态
    StatusCode int    // 状态码
    Proto      string // 协议
    Header Header  // 响应头
    Body io.ReadCloser  // 响应体
    ContentLength int64  // 响应长度
    Uncompressed bool // 报告是否以压缩方式发送了响应
    Request *Request  // 请求
    TLS *tls.ConnectionState // TLS 相关信息
}
// 服务端结构体定义,用于自定义服务端
type Server struct {
    Addr string  // HTTP 服务监听的地址,默认为":80"
    Handler Handler // 调用处理请求的程序,默认为 `http.DefaultServeMux`
    TLSConfig *tls.Config  // TLS 相关配置
    ReadTimeout time.Duration  // 读取整个请求的超时时间
    ReadHeaderTimeout time.Duration // 读取请求头的超时时间
    WriteTimeout time.Duration  // 响应的超时时间
    IdleTimeout time.Duration // 启动 keep-alived 后,等待下一个请求的超时时间
    MaxHeaderBytes int // 控制请求头的最大字节数
    ConnState func(net.Conn, ConnState) // 客户端连接状态更改时的回调函数
    ErrorLog *log.Logger // 错误日志记录器.默认为 log 包提供的标准 logger
    // contains filtered or unexported fields
}
```

## 常用常量及变量

```go
// 常见 HTTP 响应状态码定义(仅包含常见部分),详见 https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml
const (
    StatusOK                   = 200 // RFC 7231, 6.3.1
    StatusNoContent            = 204 // RFC 7231, 6.3.5

    StatusMovedPermanently = 301 // RFC 7231, 6.4.2
    StatusFound            = 302 // RFC 7231, 6.4.3
    StatusTemporaryRedirect = 307 // RFC 7231, 6.4.7
    StatusPermanentRedirect = 308 // RFC 7538, 3

    StatusBadRequest                   = 400 // RFC 7231, 6.5.1
    StatusUnauthorized                 = 401 // RFC 7235, 3.1
    StatusPaymentRequired              = 402 // RFC 7231, 6.5.2
    StatusForbidden                    = 403 // RFC 7231, 6.5.3
    StatusNotFound                     = 404 // RFC 7231, 6.5.4
    StatusMethodNotAllowed             = 405 // RFC 7231, 6.5.5
    StatusRequestTimeout               = 408 // RFC 7231, 6.5.7
    StatusRequestEntityTooLarge        = 413 // RFC 7231, 6.5.11
    StatusRequestURITooLong            = 414 // RFC 7231, 6.5.12
    StatusUnsupportedMediaType         = 415 // RFC 7231, 6.5.13
    StatusTooManyRequests              = 429 // RFC 6585, 4

    StatusInternalServerError           = 500 // RFC 7231, 6.6.1
    StatusNotImplemented                = 501 // RFC 7231, 6.6.2
    StatusBadGateway                    = 502 // RFC 7231, 6.6.3
    StatusServiceUnavailable            = 503 // RFC 7231, 6.6.4
    StatusGatewayTimeout                = 504 // RFC 7231, 6.6.5
)
// 默认的传输方式
var DefaultTransport RoundTripper = &Transport{
    Proxy: ProxyFromEnvironment,
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
        DualStack: true,
    }).DialContext,
    ForceAttemptHTTP2:     true,
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
// 默认请求处理实例对象,该类型实现了 Handler 接口
var DefaultServeMux = &defaultServeMux
```

## 常用函数

## `http` 包函数

```go
// 在 DefaultServeMux 中对给定的 pattern 注册请求处理程序
func Handle(pattern string, handler Handler)
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))

// 监听并使用 handler 处理请求.如果 handler 为 nil,则使用 `DefaultServeMux`
func ListenAndServe(addr string, handler Handler) error
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error

// 使用 Listener 实例及给定 handler 请求处理启动服务
func Serve(l net.Listener, handler Handler) error
func ServeTLS(l net.Listener, handler Handler, certFile, keyFile string) error

// 创建请求对象
func NewRequest(method, url string, body io.Reader) (*Request, error)
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error)

// 创建 ServeMux 对象
func NewServeMux() *ServeMux

// 返回对应 Handler 对象,返回值可用于传入 Handle
// 以 root 目录作为根目录为 HTTP 请求提供服务
func FileServer(root FileSystem) Handler
// 使用 "404 page not found" 响应每个请求,响应状态码为 404
func NotFoundHandler() Handler
// 重定向处理请求
func RedirectHandler(url string, code int) Handler
// 从 URL 的路径中删除给定前缀并调用处理程序
func StripPrefix(prefix string, h Handler) Handler
// 返回给定时间限制下的运行 h 的 Handler,并在超时时提示 msg
func TimeoutHandler(h Handler, dt time.Duration, msg string) Handler

// 以下函数多用于作为 handler 变量向 HandleFunc 传入或通过自定义 Handler 调用
func NotFound(w ResponseWriter, r *Request)
// 使用给定文件或目录的内容答复请求
func ServeFile(w ResponseWriter, r *Request, name string)
// 设置响应的 cookie
func SetCookie(w ResponseWriter, cookie *Cookie)
```

### `Client` 结构体方法

```go
// 关闭等待连接
func (c *Client) CloseIdleConnections()
// 使用配置的客户端发送请求并返回响应
func (c *Client) Do(req *Request) (*Response, error)
// 向 url 发送 Get,Head,Post,Post(提交表单) 请求
func (c *Client) Get(url string) (resp *Response, err error)
func (c *Client) Head(url string) (resp *Response, err error)
func (c *Client) Post(url, contentType string, body io.Reader) (resp *Response, err error)
// 向 url 发送 Post 请求,data 的键值编码为请求正文.(`url.Values` 为 `map[string][]string` 格式的字典对象)
// 此时 Content-Type 请求头设置为 application/x-www-form-urlencoded
func (c *Client) PostForm(url string, data url.Values) (resp *Response, err error)
```

## `Header` 结构体方法

```go
func (h Header) Add(key, value string)
func (h Header) Clone() Header
func (h Header) Del(key string)
func (h Header) Get(key string) string
func (h Header) Set(key, value string)
func (h Header) Values(key string) []string
func (h Header) Write(w io.Writer) error
func (h Header) WriteSubset(w io.Writer, exclude map[string]bool) error
```

### `Request` 结构体方法

```go
func (r *Request) AddCookie(c *Cookie)  // 添加 cookie
func (r *Request) BasicAuth() (username, password string, ok bool) // 返回请求基本认证中的用户名密码
func (r *Request) Cookie(name string) (*Cookie, error) // 返回指定名称的 Cookie
func (r *Request) Cookies() []*Cookie  // 返回所有 cookie
// 返回表单中指定 key 中的第一个文件对象
func (r *Request) FormFile(key string) (multipart.File, *multipart.FileHeader, error)
func (r *Request) FormValue(key string) string // 返回表单中 key 的值
func (r *Request) ParseForm() error  // 解析表单
func (r *Request) ParseMultipartForm(maxMemory int64) error // 解析带有文件的表单
func (r *Request) Referer() string  // 返回引用 URL
func (r *Request) SetBasicAuth(username, password string)  // 设置基本认证请求的用户名密码
func (r *Request) UserAgent() string // 返回请求的客户端代理
func (r *Request) Write(w io.Writer) error // 将请求写入文件
```

### `Response` 结构体方法

```go
func (r *Response) Cookies() []*Cookie  // 响应的 cookie
func (r *Response) Location() (*url.URL, error) // 返回响应的`Location` 响应头
```

### `ServeMux` 结构体方法

```go
// 注册处理请求函数
func (mux *ServeMux) Handle(pattern string, handler Handler)
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request))
// 返回给定请求的处理程序
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string)
```

### `Server` 结构体方法

```go
// 立即关闭服务
func (srv *Server) Close() error
 // 监听并启动服务
func (srv *Server) ListenAndServe() error
func (srv *Server) ListenAndServeTLS(certFile, keyFile string) error
// 注册一个函数,当 server 关闭时调用
func (srv *Server) RegisterOnShutdown(f func())
// 在 listener 上接受连接,并为每个连接创建一个新的 goroutine 处理请求
func (srv *Server) Serve(l net.Listener) error
func (srv *Server) ServeTLS(l net.Listener, certFile, keyFile string) error
// 设置是否启用 keep-alive
func (srv *Server) SetKeepAlivesEnabled(v bool)
// 优雅的关闭服务
func (srv *Server) Shutdown(ctx context.Context) error
```

## 示例

### 文件服务器

```go
import "net/http"

func main() {
    // StripPrefix 用于将请求前缀删除,请求 `/tmpfiles/` 会请求到 `/`
    // http.Handle("/tmpfiles/", http.StripPrefix("/tmpfiles/", http.FileServer(http.Dir("/usr/share/doc"))))

    // 会自动解析 index.html.如果没有,则会返回路径下的文件链接
    http.ListenAndServe(":8080", http.FileServer(http.Dir("/usr/share/doc")))
}
```

### 调用 `PostForm` 发送请求及数据

```go
import (
    "fmt"
    "net/http"
    "net/url"
)

func main() {
    data := url.Values{
        "name": []string{"name"},
        "age":  []string{"20"},
        "addr": []string{"beijing", "shanghai", "guangzhou"},
    }
    res, _ := http.PostForm("http://www.httpbin.org/post", data)
    defer res.Body.Close()
    buf := make([]byte, 1024)
    n, _ := res.Body.Read(buf)
    fmt.Println(string(buf[:n]))
```

## `Handle` 与 `HandleFunc`

```go
import (
    "io"
    "net/http"
)

func main() {
    h1 := func(w http.ResponseWriter, _ *http.Request) {
        io.WriteString(w, "Hello from a HandleFunc #1!\n")
    h2 := func(w http.ResponseWriter, _ *http.Request) {
        io.WriteString(w, "Hello from a HandleFunc #2!\n")
    }
    http.HandleFunc("/h1", h1)
    http.HandleFunc("/h2", h2)
    http.Handle("/notfount", http.NotFoundHandler())
    http.ListenAndServe(":8080", nil)
}
```

### 自定义`Handler`

```go
import (
    "fmt"
    "net/http"
    "net/url"
    "time"
)

type CustomHandler struct {
    name string
}

func (c CustomHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    time.Sleep(time.Second)
    fmt.Fprintf(w, "respose: %v", c.name) // 向 w 中写入响应
    http.ServeFile(w, req, c.name)  // 会将文件内容返回给响应
}

func main() {
    handler := CustomHandler{"index.html"}
    http.ListenAndServe(":80", http.TimeoutHandler(handler, time.Nanosecond, "请求超时"))
}
```
