---
title: go http 包处理请求过程分析
date: 2020/07/06
tags:
  - go
  - http
categories:
  - go
abbrlink: 
description: 这里将阅读 "net/http" 源码做一下记录,以作备忘.
---

先上一个简单的通过 "net/http" 包编写的服务端程序

```go
import "net/http"

type CustomHandler struct {
}

// 实现 http.Handler 接口中的 ServeHTTP 方法
func (ch CustomHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("hello"))
}

func main() {
    // 创建 CustomHandler 实例对象,传入 ListenAndServe 函数
    http.ListenAndServe("8080", CustomHandler{})
}
```

现在我们来看一下 `http.ListenAndServe("8080", CustomHandler{})` 这段代码是如何处理请求的

## 创建对象

### 创建 `http.Server` 对象

```go
// net/http/server.go#L3084
func ListenAndServe(addr string, handler Handler) error {
    // 通过传入的 addr(监听地址),与 Handler 实例(CustomHandler{}),创建 Server 对象
    server := &Server{Addr: addr, Handler: handler}
    // 并调用 Server 对象的 ListenAndServe 方法.
    return server.ListenAndServe()
}
```

### 创建 `net.TCPListener` 对象,并监听指定地址

`server.ListenAndServe()` 代码做了如下事情

1. 创建 `net.TCPListener` 对象
2. 使用创建的 `net.TCPListener` 监听地址

```go
// // net/http/server.go#2818
func (srv *Server) ListenAndServe() error {
    // 如果 srv 关闭,则返回 ErrServerClosed = errors.New("http: Server closed") 错误
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    // 判断 srv.Addr(我们传入的是 "8080") 地址是否为空
    // 若为空,则设置为 ":http",这里应该会默认监听 80 端口
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    //  创建 TCPListener 对象,用于后续监听端口,接收连接
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(ln)
}
```

其中,`net.Listen()` 函数会根据传入的参数创建 `net.TCPListener` 或 `net.UnixListener` 对象,用于后续监听 TCP 端口或 Unix 套接字.源代码如下:

```go
// net/dial.go#L705
func Listen(network, address string) (Listener, error) {
    var lc ListenConfig
    return lc.Listen(context.Background(), network, address)
}

// net/dial.goL#623
func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
    addrs, err := DefaultResolver.resolveAddrList(ctx, "listen", network, address, nil)
    if err != nil {
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: nil, Err: err}
    }
    sl := &sysListener{
        ListenConfig: *lc,
        network:      network,
        address:      address,
    }
    var l Listener
    // first 会调用传入的函数 isIPv4,判断调用者中元素是否是 ipv4 地址
    la := addrs.first(isIPv4)
    // 判断 la 类型,如果是 `TCPAddr`,则调用 listenTCP 创建 `TCPListener`
    // 如果是 `UnixAddr`,则调用 listenUnix 创建 `UnixListener`
    switch la := la.(type) {
    case *TCPAddr:
        l, err = sl.listenTCP(ctx, la)
    case *UnixAddr:
        l, err = sl.listenUnix(ctx, la)
    default:
        return nil, &OpError{Op: "listen", Net: sl.network, Source: nil, Addr: la, Err: &AddrError{Err: "unexpected address type", Addr: address}}
    }
    if err != nil {
        return nil, &OpError{Op: "listen", Net: sl.network, Source: nil, Addr: la, Err: err} // l is non-nil interface containing nil pointer
    }
    return l, nil
}
```

## 建立连接

代码跳转到 `srv.Serve(ln)`,这里是 Server 对象处理 TCPListener 接受连接的过程.

```go
// net/http/server.go#L2871
func (srv *Server) Serve(l net.Listener) error {
    if fn := testHookServerServe; fn != nil {
        fn(srv, l) // 调用钩子函数
    }
    // ... 跳过部分代码 ...
    var tempDelay time.Duration // accept 接收连接失败的睡眠时长,
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        // 调用 net.Listener 的 Accept() 接受连接,返回 `TCPConn` TCP 连接
        rw, err := l.Accept()
        if err != nil {
            select {
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
            }
            // 失败后的睡眠时长,开始为 5ms,后续每次乘 2,直到为 1s
            if ne, ok := err.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return err
        }
        connCtx := ctx
        if cc := srv.ConnContext; cc != nil {
            connCtx = cc(connCtx, rw)
            if connCtx == nil {
                panic("ConnContext returned nil")
            }
        }
        tempDelay = 0
        c := srv.newConn(rw) // Server 通过 `l.Accept()` 返回的 TCPConn 正式建立连接.
        c.setState(c.rwc, StateNew) // 设置建立连接的状态为 StateNew
        go c.serve(connCtx) // 连接正式服务请求上下文
    }
}
```

## 响应请求

`c.serve(connCtx)` 响应一个请求连接,这里也是真正处理请求的部分

```go
// net/http/server.go#L1765
func (c *conn) serve(ctx context.Context) {
    // 设置连接的远程地址
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    // 连接出错时,关闭连接
    defer func() {
        if err := recover(); err != nil && err != ErrAbortHandler {
            const size = 64 << 10
            buf := make([]byte, size)
            buf = buf[:runtime.Stack(buf, false)]
            c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
        }
        if !c.hijacked() {
            c.close()
            c.setState(c.rwc, StateClosed)
        }
    }()
    // 处理 tls 连接
    if tlsConn, ok := c.rwc.(*tls.Conn); ok {
        if d := c.server.ReadTimeout; d != 0 {
            c.rwc.SetReadDeadline(time.Now().Add(d))
        }
        if d := c.server.WriteTimeout; d != 0 {
        c.rwc.SetWriteDeadline(time.Now().Add(d))
        }
        // TLS 连接握手过程
        if err := tlsConn.Handshake(); err != nil {
            // If the handshake failed due to the client not speaking
            // TLS, assume they're speaking plaintext HTTP and write a
            // 400 response on the TLS conn's underlying net.Conn.
            if re, ok := err.(tls.RecordHeaderError); ok && re.Conn != nil &&           tlsRecordHeaderLooksLikeHTTP(re.RecordHeader) {
                io.WriteString(re.Conn, "HTTP/1.0 400 Bad Request\r\n\r\nClient sent an HTTP request to an HTTPS server.\n")
                re.Conn.Close()
                return
            }
            c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
            return
        }
        c.tlsState = new(tls.ConnectionState)
        *c.tlsState = tlsConn.ConnectionState()
        if proto := c.tlsState.NegotiatedProtocol; validNextProto(proto) {
            if fn := c.server.TLSNextProto[proto]; fn != nil {
                h := initALPNRequest{ctx, tlsConn, serverHandler{c.server}}
                fn(c.server, tlsConn, h)
            }
            return
        }


    // HTTP/1.x from here on.

    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx()
    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)
    for {
        // 读取请求
        w, err := c.readRequest(ctx)
        if c.r.remain != c.server.initialReadLimitSize() {
            // If we read any bytes off the wire, we're active.
            c.setState(c.rwc, StateActive)
        }
        // 错误请求
        if err != nil {
            const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection:        close\r\n\r\n"
            switch {
            case err == errTooLarge:
                // Their HTTP client may or may not be
                // able to read this if we're
                // responding to them and hanging up
                // while they're still writing their
                // request. Undefined behavior.
                const publicErr = "431 Request Header Fields Too Large"
                fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
                c.closeWriteAndWait()
                return  
            case isUnsupportedTEError(err):
                // Respond as per RFC 7230 Section 3.3.1 which says,
                //      A server that receives a request message with a
                //      transfer coding it does not understand SHOULD
                //      respond with 501 (Unimplemented).
                code := StatusNotImplemented
                // We purposefully aren't echoing back the transfer-encoding's value,
                // so as to mitigate the risk of cross side scripting by an attacker.
                fmt.Fprintf(c.rwc, "HTTP/1.1 %d %s%sUnsupported transfer encoding", code,   StatusText(code), errorHeaders)
                return  
            case isCommonNetReadError(err):
                return // don't reply
            default:
                publicErr := "400 Bad Request"
                if v, ok := err.(badRequestError); ok {
                    publicErr = publicErr + ": " + string(v)
                }
                fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
                return
            }
        }
        // Expect 100 Continue support
        req := w.req
        if req.expectsContinue() {
            if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
                // Wrap the Body reader with one that replies on the connection
                req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
            }
        } else if req.Header.get("Expect") != "" {
            w.sendExpectationFailed()
            return
        }
        c.curReq.Store(w)
        if requestBodyRemains(req.Body) {
            registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
        } else {
            w.conn.r.startBackgroundRead()
        }
        // HTTP cannot have multiple simultaneous active requests.[*]
        // Until the server replies to this request, it can't read another,
        // so we might as well run the handler in this goroutine.
        // [*] Not strictly true: HTTP pipelining. We could let them all process
        // in parallel even if their responses need to be serialized.
        // But we're not going to implement HTTP pipelining because it
        // was never deployed in the wild and the answer is HTTP/2.
        // 响应请求
        serverHandler{c.server}.ServeHTTP(w, w.req)
        // 处理请求完成的后续操作
        w.cancelCtx()
        if c.hijacked() {
            return
        }
        w.finishRequest()
        if !w.shouldReuseConnection() {
            if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
                c.closeWriteAndWait()
            }
            return
        }
        c.setState(c.rwc, StateIdle)
        c.curReq.Store((*response)(nil))
        if !w.conn.server.doKeepAlives() {
            // We're in shutdown mode. We might've replied
            // to the user without "Connection: close" and
            // they might think they can send another
            // request, but such is life with HTTP/1.1.
            return
        }
        if d := c.server.idleTimeout(); d != 0 {
            c.rwc.SetReadDeadline(time.Now().Add(d))
            if _, err := c.bufr.Peek(4); err != nil {
                return
            }
        }
        c.rwc.SetReadDeadline(time.Time{})
    }
}
```

### `ServeHTTP` 方法

`serverHandler{c.server}.ServeHTTP(w, w.req)` 是响应请求的方法.该方法会将响应写入响应体.

```go
// net/http/server.go#L2799
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    // 这里 handler 变量为我们传入的 Handler 实例 `CustomHandler{}`
    // 后续会分析 handler 为空的情况
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
    handler = globalOptionsHandler{}
    }
    // 调用 handler 的 ServeHTTP 方法,也就是会将 "hello" 写入响应体
    handler.ServeHTTP(rw, req)
}
```

如上代码中会调用我们传入的 `Handler` 实例 `CustomHandler{}` 的 `ServeHTTP` 方法,写入响应体.

## 使用 `http.HandleFunc` 又如何呢

先看如下代码:

```go
import "net/http"

func Hello(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("hello"))
}
func main() {
    http.HandleFunc("/hello", Hello)
    http.ListenAndServe("8080", nil)
}
```

如上代码中使用 `http.HandleFunc` 函数定义了对 `/hello` URL 请求的处理函数为 `Hello` 函数.

那么 `http.HandleFunc` 函数做了什么呢?

### `http.HandleFunc` 函数

```go
// net/http/server.go#L2451
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    // 使用 DefaultServeMux 调用 HandleFunc.
    // 其中 DefaultServeMux = &defaultServeMux 为默认的 ServeMux 对象
    // var defaultServeMux ServeMux
    DefaultServeMux.HandleFunc(pattern, handler)
}
```

其中,`ServeMux` 及 `muxEntry` 结构体如下

```go
type ServeMux struct {
    mu    sync.RWMutex  // 锁
    m     map[string]muxEntry // 保存注册的 URL 及其对应的 muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
    h       Handler // 保存 pattern 对应的 Handler
    pattern string // 保存 pattern
}
```

代码跳转到 `ServeMux` 对象的 `HandleFunc` 方法,如下:

```go
// net/http/server.go#L2436
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    if handler == nil {
        panic("http: nil handler")
    }
    // 调用 ServeMux 的 Handle 方法
    mux.Handle(pattern, HandlerFunc(handler))
}
```

其中,`HandlerFunc` 其实是形如 `func(ResponseWriter, *Request)` 的函数.调用 `HandlerFunc` 的 `ServeHTTP` 其实是调用了 `func(ResponseWriter, *Request)`.这里很重要!!!

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

代码跳转到 `ServeMux` 对象的 `Handle` 方法,如下:

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock() // 加锁
    defer mux.mu.Unlock() // 最后解锁
    if pattern == "" {
        panic("http: invalid pattern")
    }
    if handler == nil {
        panic("http: nil handler")
    }
    // 如果注册的 url 已经在,则报错
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }
    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    e := muxEntry{h: handler, pattern: pattern}
    mux.m[pattern] = e
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }
    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```

`ServeMux` 对象的 `Handle` 方法做了以下工作

1. 对 `mux.mu` 加锁
2. 判断是否存在 mux.m[pattern],如果存在,则报错
3. 新建 `e = muxEntry{h: handler, pattern: pattern}`,并设置 `mux.m[pattern] = e`. 将传入的 `HandlerFunc` 保存到 `mux.m`,也就是 `muxEntry` 对象中.
4. 如果注册 `pattern` 最后一个字符为 '/',则将 `e` 添加到 `mux.es` 中,并进行排序
5. 如果注册 `pattern` 开始字符为 '/',则设置 `mux.hosts` 为 `true`
6. 方法执行完毕对 `mux.mu` 解锁

至此 `http.HandleFunc("/hello", Hello)` 函数执行完毕.

### 再看响应请求的 `ServeHTTP` 方法

`http.ListenAndServe("8080", nil)` 前面过程与之前讨论过程基本一致.只需要看响应请求过程中的 `serverHandler{c.server}.ServeHTTP(w, w.req)` 方法.

```go
// net/http/server.go#L2799
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    // 这里 handler 为 nil,使用 DefaultServeMux
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
    handler = globalOptionsHandler{}
    }
    // 此时再调用 ServeHTTP 为调用 DefaultServeMux 的 ServeHTTP 方法
    handler.ServeHTTP(rw, req)
}
```

代码跳转到 `handler.ServeHTTP(rw, req)`,此时 handler 为 `ServeMux` 类型

```go
// net/http/server.go#L2378
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    // 此时 h 变量为我们在 http.HandleFunc 中为指定 url 定义的 HandlerFunc
    h, _ := mux.Handler(r)
    // 调用 HandlerFunc 的 ServeHTTP 其实就是调用了它自己.形如 `func(ResponseWriter, *Request)` 的函数,将数据写入响应体.
    h.ServeHTTP(w, r)
}
```

代码跳转到 `mux.Handler(r)`,如下:

```go
// net/http/server.go#L2322
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    // ... some code ...
    host := stripHostPort(r.Host)
    path := cleanPath(r.URL.Path)
    // 我们只考虑最简单的情况
    return mux.handler(host, r.URL.Path)
}

// net/http/server.go#L2359
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    // 加锁解锁
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    // 最终返回的是请求对应 HandlerFunc
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}
```

> 在看 `mux.match` 方法之前应该先对 `http.HandleFunc("/",Hello)` 有一定的理解.

代码跳转到 `mux.match(host + path)`, `mux.match` 方法会为传入的的路径选择合适的 `Handler`.具体如下:

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    // 如果 mux.m 中刚好有传入的路径,则直接返回其对应的 HandlerFunc
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }
    // 检查最长合法匹配路径. `mux.es` 中包含所有以/从最长到最短排序的模式.
    // 并返回其 HandlerFunc
    for _, e := range mux.es {
        if strings.HasPrefix(path, e.pattern) {
            return e.h, e.pattern
        }
    }
    return nil, ""
}
```

以上就是使用 `http` 包自定义服务并处理请求的全部过程.

通过 `http.HandlerFunc` 函数注册 `Handler` 与 直接向 `http.ListenAndServe` 函数传入 `Handler` 的区别在于

- `http.HandleFunc` 函数实际上注册的是 `http.HandlerFunc`,其本质是一个形如 `func (http.ResponseWriter, *http.Request)` 的函数.在处理请求时调用 `ServeHTTP` 实际上是调用了自己.且 `http.HandleFunc` 可对不同的 URL 注册不同的 `http.HandleFunc`.
- `http.ListenAndServe` 函数实际上传入的是实现了 `Handler` 接口的实例.在处理请求时调用 `ServeHTTP` 实际上是实例的 `ServeHTTP`. 可能看起来较为直观.
