---
title: HTTPS 加密机制
date: 2020/06/09
tags:
  - TLS
  - 面试
categories:
  - Linux
abbrlink: 
description: 简述 HTTPS TLS/SSL 加密机制
---

## FAQ

### 为什么需要加密

HTTP 的内容是明文传输层的,明文数据会经过中间代理服务器,路由器,通信服务运行商等多个物理节点,信息在传输过程中很容易被劫持

### 什么是中间人攻击

由于 HTTP 是明文传输的,传输内容完全暴露在互联网上,代理服务器,路由器等中间人可以直接篡改传输信息不被双方察觉,这就是中间人攻击.

### 什么是对称加密

对称加密使用同一密钥对明文和密文进行加密和解密.

对称加密最大的问题是这个密钥怎么让传输双方知晓,同时不被中间人劫持篡改.

### 什么是非对称加密

非对称加密包含两把密钥,公钥和私钥.用公钥加密的内容必须用私钥才能解开,私钥加密的内容只有公钥能解开.

采用非对称加密的客户端向服务端发起请求,服务端将公钥明文传输给客户端.客户端得到公钥后,使用公钥进行加密并传输到服务端,保证了单向传输的安全性.

那么如何将非对称加密过程中服务端公钥传输给客户端,而不会被中间人劫持/篡改?

> - 数字证书

网站在使用 HTTPS 之前,需要向 CA 机构申请颁发一份数字证书,数字证书里有证书持有者信息,公钥信息,服务端将证书传输给客户端,客户端从证书中取得公钥.

那么如何防止证书在传输过程中被篡改呢?

> - 数字签名

证书内容生成一份签名,比对证书内容和签名是否一致就能察觉是否被篡改.这种技术叫做数字签名

数字签名的制作过程如下:

- CA 证书拥有非对称加密的私钥和公钥
- CA 对证书明文信息进行 hash(hash 后再加密可以提升性能.其它原因参见[这里](crypto.stackexchange.com/a/12780))
- 对 hash 后的值用私钥加密,得到数字签名

上述证书明文与数字签名共同组成了数字证书,客户端验证过程如下:

- CA 证书已经保存在客户端中,如浏览器或命令行指定证书
- 客户端发起请求,服务端响应证书明文信息 T 和数字签名 S
- 使用 CA 证书的公钥对数字签名 S 进行解密,得到 S'
- 使用 CA 证书中说明的 hash 算法对明文进行 hash 得到 T'
- 比较 S' 是否与 T' 相同,判断证书是否可信

### HTTP S必须在每次请求中都要先在 SSL/TLS 层进行握手传输密钥吗?

服务器会为每个客户端软件维护一个 session ID,在 TSL 握手阶段传给浏览器,浏览器生成好密钥传给服务器后,服务器会把该密钥存到相应的 session ID 下,之后浏览器每次请求都会携带 session ID,服务器会根据 session ID 找到相应的密钥并进行解密加密操作,这样就不必要每次重新制作/传输密钥了.

## SSL/TLS 加密总览

SSL/TLS 安全层协议位于应用层和 TCP/IP 层之间,可以使得通信双方建立安全的通信通道,实现安全的数据传输.

SSL/TLS 协议可分为两层.如下:

- 握手协议层,由三个子协议组成: 握手协议(Handshake Protocol),密钥交换协议(Change Cipher Spec Protocol)和告警协议(Alert Protocol).
- 记录协议层(不做详细介绍)

![SSL/TLS Protocol Layers](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/ssl-tls-encryption-details/ssl-tls-protocol-layers.gif)

### 握手过程

#### 子协议

握手协议层包含如下子协议

- 握手协议

此子协议用于协商客户端与服务端之间的会话信息,包括会话 ID,证书协商,要使用的密钥规范,压缩算法以及用于生成密钥的共享加密信息等

- 密钥交换

此子协议用于更改客户端和服务端用于加密的元数据,子协议包含一条用于告知 SSL/TLS 会话中的另一方希望更改为一组新的密钥的消息.根据握手子协议的交换信息来计算密钥.

- 告警协议

告警消息用于向另一方指示状态更改或错误信息.警报可以将正常情况和错误情况通知给另一方.高兴消息的完整列表可以在 RFC 2246 "TLS 协议版本 1.0 中找到.在关闭连接,接收到无效消息,无法解密消息或用户取消操作时,通常会发送告警信息.

#### 功能

> 身份认证

握手协议使用 X.509 证书进行身份认证

> 加密

SSL/TLS 同时使用了对称加密与非对称加密.

SSL/TLS 客户端使用公钥对服务端进行认证,生成一个会话密钥.服务端与客户端使用会话密钥加密/加密传输的数据.

> 哈希算法

在握手过程中,对数据使用的哈希算法也必须达成一致.哈希算法用于检查所传输数据的完整性.

## SSL/TLS 握手过程详解

握手协议是一系列已经排序的消息,用于协商数据传输会话中的参数信息.下图说明了握手协议中的消息序列.

![Handshake Protocol Messages](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/ssl-tls-encryption-details/handshake-protocol-message.gif)

### 第一阶段: Initial Client Message to Server

客户端向服务端发送请求包含如下信息:

#### `Client Hello`: Client Hello 消息

客户端通过向服务端发送 `Client Hello` 消息来启动会话.`Client Hello` 消息包括

- **Version Number**: 客户端发送其支持的 SSL/TLS 安全通信协议的最高版本号.
  - 2 表示 SSL 2.0
  - 3 表示 SSL 3.0
  - 3.1 表示 TLS
- **Randomly Generated Data**: `ClientRandom[32]`.一个随机值,由客户端的日期和时间 4 字节的数字和 28 字节随机数字组成.该随机值将与服务端随机值一起生成 master secret,用于从中分发加密密钥.
- **Session Identification**: 会话标识(如果有).包含 Session ID 可以使客户端恢复上一次会话.恢复上一次会话很有用,因为创建新会话需要占用大量处理器资源.
- **Cipher Suite**: 加密套件.客户端上支持的加密套件列表.如,`TLS_RSA_WITH_DES_CBC_SHA`.
- **Compression Algorithm**: 压缩算法.请求的压缩算法,当前阶段为空.

`Client Hello` 消息示例如下:

```text
ClientVersion 3,1
ClientRandom[32]
SessionID: None (new session)
Suggested Cipher Suites:
   TLS_RSA_WITH_3DES_EDE_CBC_SHA
   TLS_RSA_WITH_DES_CBC_SHA
Suggested Compression Algorithm: NONE
```

### 第二阶段: Server Response to Client

服务端向客户端发送响应包含如下信息:

#### `Server Hello`: Server Hello 消息

服务端响应包含 `Server Hello` 消息,`Server Hello` 消息包括

- **Version Number**: 服务端发送通信双方都支持的 SSL/TLS 安全通信协议的最高版本号.
  - 2 表示 SSL 2.0
  - 3 表示 SSL 3.0
  - 3.1 表示 TLS
- **Randomly Generated Data**: `ServerRandom[32]`.由服务端的日期和时间 4 字节的数字和 28 字节随机数字组成.该随机值将与客户端随机值一起生成 master secret,用于从中分发加密密钥.
- **Session Identification**: 会话标识(如果有).有如下三个类别
  - New session ID: 客户端未指定要恢复的会话,由服务端生成新的 ID.当客户端指要恢复会话,但服务端无法恢复会话时,也会生成一个新会话 ID
  - Resumed Session ID: 客户端指定要恢复的会话,且服务端愿意恢复该会话
  - Null: 创建一个新会话,但服务端不会在以后恢复它,因此不会返回任何 ID
- **Cipher Suite**: 加密套件.服务端与客户端均支持的最高加密套件.如果为空,则会话以 "handshake failure" 告警结束
- **Compression Algorithm**: 压缩算法.指定压缩算法,当前阶段为空.

`Server Hello` 消息示例如下:

```text
Version 3,1
ServerRandom[32]
SessionID: bd608869f0c629767ea7e3ebf7a63bdcffb0ef58b1b941e6b0c044acb6820a77
Use Cipher Suite:
TLS_RSA_WITH_3DES_EDE_CBC_SHA
Compression Algorithm: NONE
```

#### `Server Certificate`: 服务端证书

服务端将其证书发送给客户端,服务端证书包含服务的公钥.客户端将使用此公钥对服务端进行身份验证,并加密 master secret.

#### `Server Key Exchange`: 服务端密钥交换消息(可选)

服务端创建一个临时密钥并将其发送给客户端.客户端使用此密钥在此后过程加密客户端密钥交换消息.

仅当公钥算法未提供加密客户端密钥交换消息所需的密钥材料时(如,服务端证书不包含公钥时),才执行此步骤.

#### `Client Certificate Request`: 客户端证书请求(可选)

服务端请求客户端身份认证.

此步骤用于服务端需要确认客户端身份的场景,多用于敏感场景下的双向认证.如,银行客户端与服务端要求双向认证

#### `Server Hello Done`

此消息表明服务端响应已完成,等待客户端响应

### 第三阶段: Client Response to Server

#### `Client Certificate`: 客户端证书(可选)

客户端证书,如果服务端发送了客户端证书请求,则客户端将其证书发送到服务器以进行客户端身份验证.客户的证书包含客户端的公钥

#### `Client Key Exchange`: 客户端密钥交换

客户端使用 Hello 阶段的两个随机值计算 master secret 后,将发送客户端密钥交换消息.master secret 会先通过服务端证书的公钥加密,然后再传输到服务器.双方在本地计算 master secret,并从中获取会话密钥.

#### `Certificate Verify`: 证书认证消息(可选)

客户端发送了客户端认证消息时,才发送此消息

#### `Change Cipher Spec`: 加密套件修改消息

此消息通知服务器,将使用刚刚协商的密钥和算法对"Client Finished"消息之后的所有消息进行加密.

#### `Client Finished`: 客户端握手结束消息

此消息是前面发送的所有内容的 hash 值,供服务器校验.此消息表示客户端的握手阶段已经结束.

### 第四阶段: Server Final Response to Client

#### `Change Cipher Spec Message`: 加密套件修改消息

此消息通知客户端服务器将开始使用刚刚协商的密钥对消息进行加密

#### `Server Finished Message`: 服务端握手结束消息

此消息是前面发送的所有内容的 hash 值,供客户端校验.此消息表示服务端的握手阶段已经结束

至此,整个握手阶段全部结束.接下来,客户端与服务器进入加密通信,完全是使用普通的 HTTP 协议,只不过用"会话密钥"加密内容.

## 事件驱动的 Alert Messages

告警子协议是握手协议的组件,该协议可以从任何一方发送事件驱动的告警消息.收到告警消息后,会话要么结束,要么让接收者选择是否接受会话.

下表列出了告警消息及说明,已备后续查阅

Alert Message | Fatal? | Description
:---: | :---: | :---:
unexpected_message | Yes | 非法消息
bad_record_mac | Yes | 错误的 MAC
decryption_failed | Yes | 无法解密 TLSCiphertext TLS 密文
record_overflow | Yes | 记录超过 2^14+1024 字节
handshake_failure | Yes | 不可接受的加密套件
bad_certificate | Yes | 证书出现问题
unsupported_certificate | No | 不支持的证书
certificate_revoked | No | 证书已被吊销
certificate_expired | No | 证书已过期
certificate_unknown | No | 证书不受信任
illegal_parameter | Yes | 非法加密参数
unknown_ca | Yes | CA 不受信任
access_denied | Yes | 发送方不同意协商
decode_error | Yes | 无法解码消息
decrypt_error | Yes | 握手加密操作失败
export_restriction | Yes | Not in compliance with export regulations
protocol_version | Yes | 双方不都支持的协议版本
insufficient_security | Yes | 没有达到安全要求
internal_error | Yes | 与协议无关的错误
user_canceled | Yes | 与协议无关的失败
no_renegotiation | No | 协商请求被拒绝

---

参考:

- [彻底搞懂HTTPS的加密机制](https://zhuanlan.zhihu.com/p/43789231)
- [Overview of SSL/TLS Encryption](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc781476(v=ws.10))
- [SSL/TLS in Detail](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc785811(v=ws.10))
