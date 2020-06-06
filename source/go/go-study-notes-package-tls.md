---
title: go 学习笔记之 tls 包
date: 2020/05/22
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 24693
description: 本文章主要包含 Go crypto/tls 包及其部分子包内置类型和方法的使用.crypto/tls 包部分实现 TLS1.2 及 TLS1.3 相关内容
---

tls 包定义并提供了 TLS 1.2 及 TLS 1.3 传输层安全协议通信过程中使用的对象,函数,方法及加密算法等,为通信双方提供了安全的通信连接.

## 常用类型定义

```go
// tls 证书
type Certificate struct {
    Certificate [][]byte
    // PrivateKey 包含与 Leaf 中公钥对应的私钥.该对象必须以 RSA,ECDSA 或 Ed25519 算法实现 crypto.Signer 接口
    // 对于 TLS1.2 以下的服务,它可以使用 RSA PublicKey 实现 crypto.Decrypter 接口
    PrivateKey crypto.PrivateKey
    // PrivateKey 可使用的签名算法列表
    SupportedSignatureAlgorithms []SignatureScheme // Go 1.14
    // 客户端请求的 OCSP 响应
    OCSPStaple []byte
    // 请求它的客户端的证书时间戳列表
    SignedCertificateTimestamps [][]byte // Go 1.5
    // 叶子证书的解析形式,可使用 x509.ParseCertificate 对其进行初始化,以减少每次握手的过程.如果为 nil,则根据需要解析叶子证书
    Leaf *x509.Certificate
}

// 来自服务端的 CertificateRequest 消息的信息,该消息用于向客户端索取证书协商协议
type CertificateRequestInfo struct {
    // 包含0个或多个 DER 编码的 X.501 专有名称.
    // 这些是服务端希望返回的由其根或中间 CA 签名的证书名称
    // 空切片表示服务端没有首选项
    AcceptableCAs [][]byte
    // 服务端支持的验证签名方案
    SignatureSchemes []SignatureScheme
    // 为此连接协商的 TLS 版本.
    Version uint16 // Go 1.14
}

// 客户端 TLS 连接状态
type ClientSessionState struct {
    // 内含隐藏或非导出字段
}

// 服务器遵循的 TLS 客户端身份验证策略
type ClientAuthType int

// 包含 ClientHello 消息信息
type ClientHelloInfo struct {
    // 客户端支持的加密套件
    CipherSuites []uint16
    // 客户端请求的 ServerName
    ServerName string

    SupportedCurves []CurveID
    SupportedPoints []uint8

    // 客户端支持的签名和哈希方案
    SignatureSchemes []SignatureScheme // Go 1.8
    // 客户端支持的应用层协议
    SupportedProtos []string // Go 1.8
    // 客户端支持的 TLS 版本
    SupportedVersions []uint16 // Go 1.8
    // 底层连接对象,net.Conn 实例
    Conn net.Conn // Go 1.8
    // 包含过滤或未导出的字段
}

// ClientSessionState 对象的缓存
type ClientSessionCache interface {
    // Get搜索与给出的键相关联的*ClientSessionState并用ok说明是否找到
    Get(sessionKey string) (session *ClientSessionState, ok bool)
    // Put将*ClientSessionState与给出的键关联并写入缓存中
    Put(sessionKey string, cs *ClientSessionState)
}

// tls 相关配置
type Config struct {
    // 随机数生成,默认使用 `crypto/rand` 包中加密随机数生成器
    // 必须可以安全地被多个 goroutine  使用
    Rand io.Reader
    // 返回当前时间的函数.默认使用 time.Now
    Time func() time.Time
    // 一个或多个证书链以呈现给连接另一端.将自动选择与之匹配的证书
    // 服务端必须设置 Certificates,GetCertificate 或 GetConfigForClient 之一
    // 客户端可以设置 Certificates 或 GetCertificate
    // 注意: 如果有多个证书,且没有设置可选的 Leaf 字段,选择证书过程可能会导致大量的握手成本
    Certificates []Certificate
    // 证书名称到 Certificates 的映射
    NameToCertificate map[string]*Certificate
    // 返回基于给定 ClientHelloInfo 的证书.当客户端提供 SNI 信息或 Certificates 为空时,才会调用该方法
    GetCertificate func(*ClientHelloInfo) (*Certificate, error) // Go 1.4
    // 服务端从客户端请求证书时,调用该函数获取客户端证书.如果定义了此成员变量,Certificates 内容将被忽略
    GetClientCertificate func(*CertificateRequestInfo) (*Certificate, error) // Go 1.8
    // 收到客户端 ClientHello 后调用此函数为获取客户端配置
    GetConfigForClient func(*ClientHelloInfo) (*Config, error) // Go 1.8
    // TLS 客户端或服务端进行常规证书验证后,调用该函数进行 TLS 握手验证
    VerifyPeerCertificate func(rawCerts [][]byte, verifiedChains [][]*x509.Certificate) error // Go 1.8
    // 客户端验证服务端证书 CA
    // 如果为 nil,则使用主机的 CA 集合
    RootCAs *x509.CertPool
    // 按优先级顺序列出支持的应用层协议
    NextProtos []string
    // TLS 服务端主机名称,用于验证证书的主机名.除非使用 InsecureSkipVerify 跳过主机名认证
    ServerName string
    // TLS 服务端对客户端身份验证的策略,默认为 NoClientCert,不对客户端做身份验证
    ClientAuth ClientAuthType
    // 客户端证书颁发机构,如果 ClientAuth 设置需要验证客户端证书,服务端将使用这些证书颁发机构
    ClientCAs *x509.CertPool
    // 客户端是否跳过验证服务端证书和主机名
    InsecureSkipVerify bool
    // 直到 TLS1.2 版本支持的密码套件列表
    CipherSuites []uint16
    // 是否优先选择服务端密码套件
    PreferServerCipherSuites bool // Go 1.1
    // 是否禁用 ticket 会话和 PSK 会话支持
    SessionTicketsDisabled bool // Go 1.1
    // 使用 SessionTicketKey 提供会话恢复
    SessionTicketKey [32]byte // Go 1.1
    // 客户端 ClientSessionState 缓存
    ClientSessionCache ClientSessionCache // Go 1.3
    // 支持的最低版本,TLs 1.0 为最小值
    MinVersion uint16 // Go 1.2
    // 支持的最高版本,TLS 1.3 为最大值
    MaxVersion uint16 // Go 1.2

    CurvePreferences []CurveID // Go 1.3
    DynamicRecordSizingDisabled bool // Go 1.7
    Renegotiation RenegotiationSupport // Go 1.7
    KeyLogWriter io.Writer // Go 1.8
    // 包含其它过滤或未导出的字段
}

// 安全连接.实现了 `net.Conn` 接口
type Conn struct {
    // contains filtered or unexported fields
}

// 连接状态信息
type ConnectionState struct {
    // 连接使用的 TLS 版本
    Version uint16
    // TLS 握手是否完成
    HandshakeComplete bool
    // 是否连接恢复先前的TLS连接
    DidResume bool
    // 使用的加密套件
    CipherSuite uint16
    // 协商的协议
    NegotiatedProtocol string
    NegotiatedProtocolIsMutual bool
    // 客户端请求的主机名
    ServerName string
    PeerCertificates []*x509.Certificate
    VerifiedChains [][]*x509.Certificate
    SignedCertificateTimestamps [][]byte
    OCSPResponse []byte
    TLSUnique []byte // Go 1.4
    // 包含过滤或未导出的字段
}
// 签名算法
type SignatureScheme uint16
```

## 常量及变量

```go
// 加密套件
const (
    // TLS 1.0 - 1.2 cipher suites.
    TLS_RSA_WITH_RC4_128_SHA                      uint16 = 0x0005
    TLS_RSA_WITH_3DES_EDE_CBC_SHA                 uint16 = 0x000a
    TLS_RSA_WITH_AES_128_CBC_SHA                  uint16 = 0x002f
    TLS_RSA_WITH_AES_256_CBC_SHA                  uint16 = 0x0035
    TLS_RSA_WITH_AES_128_CBC_SHA256               uint16 = 0x003c
    TLS_RSA_WITH_AES_128_GCM_SHA256               uint16 = 0x009c
    TLS_RSA_WITH_AES_256_GCM_SHA384               uint16 = 0x009d
    TLS_ECDHE_ECDSA_WITH_RC4_128_SHA              uint16 = 0xc007
    TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA          uint16 = 0xc009
    TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA          uint16 = 0xc00a
    TLS_ECDHE_RSA_WITH_RC4_128_SHA                uint16 = 0xc011
    TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA           uint16 = 0xc012
    TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA            uint16 = 0xc013
    TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA            uint16 = 0xc014
    TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256       uint16 = 0xc023
    TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256         uint16 = 0xc027
    TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256         uint16 = 0xc02f
    TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256       uint16 = 0xc02b
    TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384         uint16 = 0xc030
    TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384       uint16 = 0xc02c
    TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256   uint16 = 0xcca8
    TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 uint16 = 0xcca9

    // TLS 1.3 cipher suites.
    TLS_AES_128_GCM_SHA256       uint16 = 0x1301
    TLS_AES_256_GCM_SHA384       uint16 = 0x1302
    TLS_CHACHA20_POLY1305_SHA256 uint16 = 0x1303

    // TLS_FALLBACK_SCSV isn't a standard cipher suite but an indicator
    // that the client is doing version fallback. See RFC 7507.
    TLS_FALLBACK_SCSV uint16 = 0x5600

    // Legacy names for the corresponding cipher suites with the correct _SHA256
    // suffix, retained for backward compatibility.
    TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305   = TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
    TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305 = TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
)
// TLS 版本
const (
    VersionTLS10 = 0x0301
    VersionTLS11 = 0x0302
    VersionTLS12 = 0x0303
    VersionTLS13 = 0x0304
    // 过时
    VersionSSL30 = 0x0300
)
// 客户端认证类别
const (
    NoClientCert ClientAuthType = iota
    RequestClientCert
    RequireAnyClientCert
    VerifyClientCertIfGiven
    RequireAndVerifyClientCert
)
// 签名算法
const (
    // RSASSA-PKCS1-v1_5 algorithms.
    PKCS1WithSHA256 SignatureScheme = 0x0401
    PKCS1WithSHA384 SignatureScheme = 0x0501
    PKCS1WithSHA512 SignatureScheme = 0x0601

    // RSASSA-PSS algorithms with public key OID rsaEncryption.
    PSSWithSHA256 SignatureScheme = 0x0804
    PSSWithSHA384 SignatureScheme = 0x0805
    PSSWithSHA512 SignatureScheme = 0x0806

    // ECDSA algorithms. Only constrained to a specific curve in TLS 1.3.
    ECDSAWithP256AndSHA256 SignatureScheme = 0x0403
    ECDSAWithP384AndSHA384 SignatureScheme = 0x0503
    ECDSAWithP521AndSHA512 SignatureScheme = 0x0603

    // EdDSA algorithms.
    Ed25519 SignatureScheme = 0x0807

    // Legacy signature and hash algorithms for TLS 1.2.
    PKCS1WithSHA1 SignatureScheme = 0x0201
    ECDSAWithSHA1 SignatureScheme = 0x0203
)

```

## 常用函数

### `tls` 包函数

```go
// 使用给定 Network 和 addr 创建 TLS 监听器, config 必须至少包含一个证书
func Listen(network, laddr string, config *Config) (net.Listener, error)
// 通过 inner 监听器创建一个新监听器,并包装与服务端的每个连接.
func NewListener(inner net.Listener, config *Config) net.Listener
// 读取并解析一对文件获取公钥私钥.这些文件必须是  pem 编码的.返回文件中包含的证书
func LoadX509KeyPair(certFile, keyFile string) (cert Certificate, err error)
// 解析一对 pem 编码格式的数据获取公钥私钥.返回数据中包含的证书
func X509KeyPair(certPEMBlock, keyPEMBlock []byte) (cert Certificate, err error)
// 创建 LRU(最近最少使用) 缓存策略的 ClientSessionState.如果 capacity<1 会使用默认值
func NewLRUClientSessionCache(capacity int) ClientSessionCache

// 使用 conn 作为下层传输接口返回一个 TL S连接的客户端.config 必须是非 nil 的且必须设置了 ServerName 或者 InsecureSkipVerify 字段
func Client(conn net.Conn, config *Config) *Conn
// 使用 conn 作为下层传输接口返回一个 TLS 连接的服务端.config 必须是非 nil 的且必须含有至少一个证书
func Server(conn net.Conn, config *Config) *Conn
// 使用 net.Dial 连接指定 network(协议)和 addr(地址),然后根据 config 发起 TLS 握手.返回生成的 TLS 连接
func Dial(network, addr string, config *Config) (*Conn, error)
// 使用 net.Dialer 连接指定地址,然后发起 TLS 握手,返回生成的 TLS 连接
func DialWithDialer(dialer *net.Dialer, network, addr string, config *Config) (*Conn, error)
```

### `Conn` 结构体方法

```go
// 关闭连接
func (c *Conn) Close() error
// 返回当前连接的状态信息
func (c *Conn) ConnectionState() ConnectionState
// 握手.本包大多数不需要显示调用,第一次 Read 或 Write 会自动调用
func (c *Conn) Handshake() error
// 返回本地网络地址
func (c *Conn) LocalAddr() net.Addr
// 从 TLS 服务器返回装订的 OCSP 响应
func (c *Conn) OCSPResponse() []byte
// 从连接中读取数据到 b.可能会超时返回一个 net.Error
func (c *Conn) Read(b []byte) (int, error)
// 远程网络地址
func (c *Conn) RemoteAddr() net.Addr
// 设置与连接关联的读写期限.t=0表示完全不会超时
func (c *Conn) SetDeadline(t time.Time) error
func (c *Conn) SetReadDeadline(t time.Time) error
func (c *Conn) SetWriteDeadline(t time.Time) error
// 检查连接的证书链对于 host 是否有效.返回问题的描述
func (c *Conn) VerifyHostname(host string) error
// 将数据写入连接
func (c *Conn) Write(b []byte) (int, error)
```
