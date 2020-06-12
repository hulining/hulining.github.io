---
title: go 学习笔记之 crypto 包
date: 2020/05/05
tags:
  - go
categories:
  - go
abbrlink: 16535
description: 本文章主要包含 Go crypto 包及其部分子包内置类型和方法的使用.crypto 包及其子包包含了通用加密算法的集合.
---

## `crypto 包`

crypto 包包含了通用加密算法的集合,提供了一些加密过程中基本对象的封装或基本接口的定义.导入方式为 `import "crypto"`.

当前我们项目中常用的加解密的方式无非三种.

- 对称加密: 加解密都使用的是同一个密钥,其中的代表就是 AES,DES(已被攻破)
- 非对加解密: 加解密使用不同的密钥,其中的代表就是 RSA
- 签名算法: 主要用于验证,防止信息被修改,多用于文件校验,数字签名,鉴权协议的等,其中算法主要有 MD5,SHA1,HMAC 等

### 类型定义

它主要包含如下加密过程中的基本类型定义

```go
// Decrypter 是非公开私钥的接口,可用于非对称解密操作.一个示例是保存在硬件模块中的 RSA 密钥
type Decrypter interface {
    // 返回与非公开私钥相对应的公钥
    Public() PublicKey
    // 解密 msg.opts 参数应该是所使用的加密算法函数.详情参见每个实现文档
    Decrypt(rand io.Reader, msg []byte, opts DecrypterOpts) (plaintext []byte, err error)
}

type DecrypterOpts interface{}

// Hash 标识在其它包中实现的加密哈希函数
type Hash uint

// 表示未指定算法的私钥
type PrivateKey interface{}

// 表示未指定算法的公钥
type PublicKey interface{}

// Signer 是非公开私钥的接口,可用于签名操作.
type Signer interface {
    // 返回与非公开私钥相对应的公钥
    Public() PublicKey

    // 使用私钥对 digest 进行签名
    Sign(rand io.Reader, digest []byte, opts SignerOpts) (signature []byte, err error)
}

// 用于使用 Signer 签名的参数
type SignerOpts interface {
    // 返回用于生成传递给 `Signer.Sign` 的消息的哈希函数,或零表示没有哈希完成
    HashFunc() Hash
}
```

### 常量及变量

```go
const (
    MD4         Hash = 1 + iota // import golang.org/x/crypto/md4
    MD5                         // import crypto/md5
    SHA1                        // import crypto/sha1
    SHA224                      // import crypto/sha256
    SHA256                      // import crypto/sha256
    SHA384                      // import crypto/sha512
    SHA512                      // import crypto/sha512
    MD5SHA1                     // no implementation; MD5+SHA1 used for TLS RSA
    RIPEMD160                   // import golang.org/x/crypto/ripemd160
    SHA3_224                    // import golang.org/x/crypto/sha3
    SHA3_256                    // import golang.org/x/crypto/sha3
    SHA3_384                    // import golang.org/x/crypto/sha3
    SHA3_512                    // import golang.org/x/crypto/sha3
    SHA512_224                  // import crypto/sha512
    SHA512_256                  // import crypto/sha512
    BLAKE2s_256                 // import golang.org/x/crypto/blake2s
    BLAKE2b_256                 // import golang.org/x/crypto/blake2b
    BLAKE2b_384                 // import golang.org/x/crypto/blake2b
    BLAKE2b_512                 // import golang.org/x/crypto/blake2b
)
```

### 子包

它包含众多子包,如下

名称 | 简介
:--- | :---
aes | 实现了 AES加密
cipher | 实现了标准分组密码模式(CBC,ECB 等),可以将其包装在低级分组密码实现中
des | 实现了数据加密标准(DES)和三重数据加密算法(TDEA)
dsa | 实现了 FIPS 186-3 中定义的数字签名算法
ecdsa | 实现了 FIPS 186-3 中定义的椭圆曲线数字签名算法
ed25519 | 实现了 ED25519签名算法
elliptic | 实现了在素数域上的几个标准椭圆曲线
hmac | 实现了 Keyed-Hash 消息认证码
md5 | 实现了 RFC 1321 中定义的 MD5 哈希算法
rand | 实现了加密安全的随机数生成器
rc4 | 实现了 RC4 加密
rsa | 实现了 PKCS#1 中定义的 RSA 加密
sha1 | 实现了 RFC 3174 中定义的 SHA-1 哈希算法
sha256 | 实现了 FIPS 180-4 中定义的 SHA224 和 SHA256 哈希算法
sha512 | 实现了 FIPS 180-4 中定义的 SHA-384, SHA-512, SHA-512/224 和 SHA-512/256 哈希算法
tls | 部分实现了 RFC 5246 中指定的 TLS 1.2 和 RFC 8446 中指定的 TLS 1.3
x509 | 解析 X.509 编码的密钥和证书
x509/pkix | 包含用于X.509证书,CRL 和 OCSP 的 ASN.1 解析和序列化的共享低级结构

## `crypto/aes`

aes 包实现了 AES 对称加密.导入方式为 `import "crypto/aes"`

### 常用常量及变量

```go
// AES 块大小.以字节为单位
const BlockSize = 16
```

### 常用方法

```go
// 创建并返回新的 `cipher.Block`
// 参数 key 是 AES 密钥,支持 16,24 或 32 个字节选择使用 AES-128, AES-192 或 AES-256 加密算法
func NewCipher(key []byte) (cipher.Block, error)
```

## `crypto/cipher`

cipher 包实现标准的分组加密模式(CBC,ECB 等),可以将其包装在低级分组加密实现中.导入方式为 `import "crypto/cipher"`

### 常用类型定义

```go
// AEAD 加密模式接口,对关联数据进行身份验证加密
type AEAD interface {
    // 返回必须传递给 Seal 和 Open 的随机数的大小
    NonceSize() int
    // 返回明文长度与密文长度间的最大差值
    Overhead() int

    // 对明文进行加密,返回密文内容
    // 对于给定的密钥,nonce 的长度必须为 NonceSize() 返回的长度
    // 对于一个完整的加密解密过程而言,nonce始终是唯一的
    Seal(dst, nonce, plaintext, additionalData []byte) []byte

    // 对密文进行解密,返回明文内容
    // 对于给定的密钥,nonce 的长度必须为 NonceSize() 返回的长度
    // 对于一个完整的加密解密过程而言,nonce始终是唯一的
    Open(dst, nonce, ciphertext, additionalData []byte) ([]byte, error)
}

// 表示使用给定密钥分组加密的接口.它提供了加密或解密单个分组的功能.模式的实现将该功能扩展为流式分组
type Block interface {
    // 返回加密的分组大小
    BlockSize() int
    // 将 src 中第一个分组加密为 dst. dst 和 src 必须完全重叠或不重叠
    Encrypt(dst, src []byte)
    // 将 src 中第一个分组解密为 dst. dst 和 src 必须完全重叠或不重叠
    Decrypt(dst, src []byte)
}

// 表示基于指定分组模式加密的接口
type BlockMode interface {
    // 返回分组模式加密的分组大小
    BlockSize() int
    // 加密或解密多个分组. src 的长度必须是分组的整数倍,且dst 和 src 必须完全重叠或不重叠
    CryptBlocks(dst, src []byte)
}

// 表示流加密
type Stream interface {
    // 对给定切片中的每个字节与密码密钥流中的字节进行 XOR.dst 和 src 必须完全重叠或不重叠
    XORKeyStream(dst, src []byte)
}

type StreamReader struct {
    S Stream
    R io.Reader
}
type StreamWriter struct {
    S   Stream
    W   io.Writer
    Err error // unused
}
```

### 常用方法

#### `cipher` 包方法

```go
// 返回以GCM模式下以标准随机数长度(12)包装的 128 位 AEAD 分组加密模式
func NewGCM(cipher Block) (AEAD, error)
// 返回以GCM模式下以指定随机数长度(12)包装的 128 位 AEAD 分组加密模式
func NewGCMWithNonceSize(cipher Block, size int) (AEAD, error)

// 返回以分组加密链接模式(CBC)解密的分组模式
// iv 的长度必须与分组大小相同且必须与加密数据的 iv 相同
func NewCBCDecrypter(b Block, iv []byte) BlockMode
// 返回以分组解密链接模式(CBC)解密的分组模式
// iv 的长度必须与分组大小相同且必须与解密数据的 iv 相同
func NewCBCEncrypter(b Block, iv []byte) BlockMode

// 返回以加密反馈模式(CFB)解密的分组模式
// iv 的长度必须与分组大小相同且必须与解密数据的 iv 相同
func NewCFBDecrypter(block Block, iv []byte) Stream
// 返回以加密反馈模式(CFB)加密的 Stream
// iv 的长度必须与分组大小相同且必须与解密数据的 iv 相同
func NewCFBEncrypter(block Block, iv []byte) Stream

// 返回以计数器模式下进行加密/解密的 Stream
// iv 的长度必须与分组大小相同且必须与解密数据的 iv 相同
func NewCTR(block Block, iv []byte) Stream

// 返回以输出反馈模式(OFB)加密或解密的 Stream
// iv 的长度必须与等于 b 的分组大小
func NewOFB(b Block, iv []byte) Stream
```

### 示例

#### AES GCM 模式认证加密解密

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "io"
)

func AESGCMEncrypt(plaintextStr string, keyStr string) (string, string) {
    // 将明文和密钥转换为字节切片
    plaintext := []byte(plaintextStr)
    key := []byte(keyStr)

    // 创建加密分组
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(fmt.Sprintf("key 长度必须 16/24/32长度: %s", err.Error()))
    }

    // 创建 GCM 模式的 AEAD
    aesgcm, err := cipher.NewGCM(block)
    if err != nil {
        panic(err.Error())
    }
    // 创建随机数,这里在实际应用中让它只生成一次.不然每次都需要进行修改
    nonce := make([]byte, aesgcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        panic(err.Error())
    }
    // 生成密文
    ciphertext := aesgcm.Seal(nil, nonce, plaintext, nil)
    // 返回密文及随机数的 base64 编码
    fmt.Println(ciphertext, nonce, key)
    return base64.RawURLEncoding.EncodeToString(ciphertext), base64.RawURLEncoding.EncodeToString(nonce)
}

func AESGCMDecrypt(ciphertextStr string, keyStr string, nonceStr string) string {
    // 将密文,密钥和生成的随机数转换为字节切片
    ciphertext, _ := base64.RawURLEncoding.DecodeString(ciphertextStr)
    nonce, _ := base64.RawURLEncoding.DecodeString(nonceStr)
    key := []byte(keyStr)

    // 创建加密分组
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(err.Error())
    }
    // 创建 GCM 模式的 AEAD
    aesgcm, err := cipher.NewGCM(block)
    if err != nil {
        panic(err.Error())
    }
    // 明文内容
    plaintext, err := aesgcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        panic(err.Error())
    }
    return string(plaintext)
}

func main() {
    plaintext := "hello world"
    key := "123456781234567812345678" // 16,24,32 位的密钥
    fmt.Println("原文：", plaintext)
    ciphertext, nonce := AESGCMEncrypt(plaintext, key)
    fmt.Println("密文: ", ciphertext, nonce)
    fmt.Println("解密结果: ", AESGCMDecrypt(ciphertext, key, nonce))
}
```

#### AES CBC 模式加密解密

```go
import (
    "bytes"
    "crypto/aes"
    "crypto/cipher"
    "encoding/base64"
    "fmt"
)

func AESCBCEncrypt(plaintextStr string, keyStr string) string {
    // 转成字节数组
    plaintext := []byte(plaintextStr)
    key := []byte(keyStr)

    // 分组秘钥
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(fmt.Sprintf("key 长度必须 16/24/32长度: %s", err.Error()))
    }
    // 获取秘钥块的长度
    blockSize := block.BlockSize()
    // 补全码
    plaintext = PKCS7Padding(plaintext, blockSize)
    // 加密模式
    blockMode := cipher.NewCBCEncrypter(block, key[:blockSize])
    // 创建数组
    ciphertext := make([]byte, len(plaintext))
    // 加密
    blockMode.CryptBlocks(ciphertext, plaintext)
    //使用 RawURLEncoding,不要使用 StdEncoding,放在url参数中会导致错误
    return base64.RawURLEncoding.EncodeToString(ciphertext)

}

func AESCBCDecrypt(ciphertextStr string, key string) string {
    //使用 RawURLEncoding,不要使用 StdEncoding 放在url参数中回导致错误
    ciphertext, _ := base64.RawURLEncoding.DecodeString(ciphertextStr)
    k := []byte(key)

    // 分组秘钥
    block, err := aes.NewCipher(k)
    if err != nil {
        panic(fmt.Sprintf("key 长度必须 16/24/32长度: %s", err.Error()))
    }
    // 获取秘钥块的长度
    blockSize := block.BlockSize()
    // 加密模式
    blockMode := cipher.NewCBCDecrypter(block, k[:blockSize])
    // 创建数组
    plaintext := make([]byte, len(ciphertext))
    // 解密
    blockMode.CryptBlocks(plaintext, ciphertext)
    // 去补全码
    plaintext = PKCS7UnPadding(plaintext)
    return string(plaintext)
}

//补码
func PKCS7Padding(plaintext []byte, blocksize int) []byte {
    padding := blocksize - len(plaintext)%blocksize
    padtext := bytes.Repeat([]byte{byte(padding)}, padding)
    return append(plaintext, padtext...)
}

//去码
func PKCS7UnPadding(plaintext []byte) []byte {
    length := len(plaintext)
    unpadding := int(plaintext[length-1])
    return plaintext[:(length - unpadding)]
}

func main() {
    plaintext := "hello world"
    key := "123456781234567812345678" // 16,24,32 位的密钥
    fmt.Println("原文: ", plaintext)
    ciphertext := AESCBCEncrypt(plaintext, key)
    fmt.Println("密文: ", ciphertext)
    plaintext2 := AESCBCDecrypt(ciphertext, key)
    fmt.Println("解密结果: ", plaintext2)
}
```

CFB 模式加密用法与 CBC 加密模式用法基本一致,在创建 `blockMode` 时调用对应的方法即可

## `crypto/md5`

md5 包实现了 RFC 1321 中定义的 MD5 哈希算法.导入方式 `import "crypto/md5"`. `sha1`,`sha256`,`sha512` 变量定义与相关方法与 `md5` 类似,不再赘述

### 常量及变量

```go
// MD5的块大小,以字节为单位
const BlockSize = 64
// MD5 校验和的大小
const Size = 16
```

### 常用方法

```go
// 返回 `hash.Hash`,用于计算 MD5校验和.
func New() hash.Hash
// 返回 data 的 MD5 校验和
func Sum(data []byte) [Size]byte
```

### 示例

### 计算字符串的校验和

```go
import (
    "crypto/md5"
    "fmt"
    "io"
)

func main() {
    // 创建一个 `hash.Hash`
    h := md5.New()
    // 向其中写入数据
    io.WriteString(h, "The fog is getting thicker!")
    io.WriteString(h, "And Leon's getting laaarger!")
    fmt.Printf("%x", h.Sum(nil)) // h.Sum() 用于计算校验和

    // 或直接使用包函数 md5.Sum(data []byte) 计算校验和
    data := []byte("These pretzels are making me thirsty.")
    fmt.Printf("%x", md5.Sum(data))
}
```

### 计算文件的校验和

```go
import (
    "crypto/md5"
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    f, err := os.Open("file.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    h := md5.New()
    // 其实还是将文件中数据读出来,写入到 h 对象中
    if _, err := io.Copy(h, f); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%x", h.Sum(nil))
}

```

## `crypto/rand`

rand 包实现了加密安全的随机数生成器.它在包中定义了全局唯一的 `var Reader io.Reader` 变量提供了以下方法用于生成随机数

### 常量及变量

```go
// 加密安全的随机数生成其的全局共享实例
var Reader io.Reader
```

### 函数

```go
// 返回范围为 [0,max) 的随机整数值
func Int(rand io.Reader, max *big.Int) (n *big.Int, err error)
// 将随机数传入 b.其实是调用了 io.ReadFull(Reader, b),是该函数的辅助函数.
// 当且仅当 n=len(b)时,返回 err=nil
func Read(b []byte) (n int, err error)
```

### 示例

```go
// 创建随机数的切片,使用 `io.ReadFull` 将 rand.Reader 产生的随机数写入 nonce 中
nonce := make([]byte, 12)
if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        panic(err.Error())
}
```

## `crypto/rsa`

rsa 实现了 RSA 非对称加密算法,该算法通过生成密钥对实现加密和加密,公钥文件用于加密,私钥用于解密.

RSA 是一个基本操作,在此程序包中用于实现公钥加密或公钥签名.此包中的 RSA 操作未使用固定时间算法实现.

### 常量及变量

```go
const (
    // PSSSaltLengthAuto 会使 RSA-PSS 签名中的盐在签名时尽可能大,并在验证时自动检测
    PSSSaltLengthAuto = 0
    // PSSSaltLengthEqualsHash 使盐的长度等于签名中使用的哈希的长度
    PSSSaltLengthEqualsHash = -1
)
```

### 类型定义

```go
// RSA 私钥的结构体定义,可通过 `rsa.GenerateKey` 生成对象
type PrivateKey struct {
    PublicKey            // 公钥部分
    D         *big.Int   // private exponent
    Primes    []*big.Int // prime factors of N, has >= 2 elements.
    // Precomputed contains precomputed values that speed up private operations, if available.
    Precomputed PrecomputedValues
}

// RSA 公钥结构体定义,包含在私钥中
type PublicKey struct {
    N *big.Int // modulus
    E int      // public exponent
}

// 用于使用 `crypto.Decrypter` 解密的 OAEP 选项结构体
type OAEPOptions struct {
    // 生成掩码时使用的哈希函数.
    Hash crypto.Hash
    // 任意字节字符,必须等于加密时使用的 Label
    Label []byte
}

// 用于使用 `crypto.Decrypter` PKCS#1 v1.5 解密的 PKCS1v15 选项结构体
type PKCS1v15DecryptOptions struct {
    // 要解密的会话密钥的长度. 如果不为零,解密期间的错误将返回此长度的随机明文,而不是返回错误
    SessionKeyLen int
}

// 用于创建和验证 RSA-PSS 签名的选项结构体
type PSSOptions struct {
    // 控制 PSS 签名中使用盐的长度,可以是一个数字,也可以是特殊 PSSSaltLength 常量之一
    SaltLength int

    // 覆盖传递给 SignPSS 的 hash 函数
    // 这是使用 crypto.Signer 接口时指定哈希函数的唯一方法
    Hash crypto.Hash
}
```

### 常用方法

### `rsa` 包函数

```go
// 使用随机树源 random 生成指定 bits 的 RSA 密钥对
func GenerateKey(random io.Reader, bits int) (*PrivateKey, error)
// 使用 RSA-OAEP 对 msg 进行加密
func EncryptOAEP(hash hash.Hash, random io.Reader, pub *PublicKey, msg []byte, label []byte) ([]byte, error)
// 使用 RSA-OAEP 对 ciphertext 进行解密
func DecryptOAEP(hash hash.Hash, random io.Reader, priv *PrivateKey, ciphertext []byte, label []byte) ([]byte, error)

// 使用 RSA 和 PKCS#1 v1.5 中的补充协议对给定 msg 进行加密解密
func EncryptPKCS1v15(rand io.Reader, pub *PublicKey, msg []byte) ([]byte, error)
func DecryptPKCS1v15(rand io.Reader, priv *PrivateKey, ciphertext []byte) ([]byte, error)

// 使用来自 RSA PKCS＃1 v1.5 的 RSASSA-PKCS1-V1_5-SIGN 计算 hashed 的签名及认证
// hashed 必须是使用 hash 函数对 msg 进行哈希的结果
func SignPKCS1v15(rand io.Reader, priv *PrivateKey, hash crypto.Hash, hashed []byte) ([]byte, error)
func VerifyPKCS1v15(pub *PublicKey, hash crypto.Hash, hashed []byte, sig []byte) error
// 使用 RSASSA-PSS [1]计算 hashed 的签名及认证
// hashed 必须是使用 hash 函数对 msg 进行哈希的结果.opts参数可以为nil,使用默认值
func SignPSS(rand io.Reader, priv *PrivateKey, hash crypto.Hash, hashed []byte, opts *PSSOptions) ([]byte, error)
```

#### `PrivateKey` 结构体方法

`PrivateKey` 结构体实现了 `crypto.Decrypter` 和 `crypto.Signer` 接口.具有以下方法

```go
// 使用 priv 对指定 ciphertext 密文解密,返回解密后的明文即可能产生的错误
// 如果 opts 为空或 *PKCS1v15DecryptOptions,则使用 PKCS#1 v1.5 解密.否则必须为 *OAEPOptions 使用 OAEP 解密
func (priv *PrivateKey) Decrypt(rand io.Reader, ciphertext []byte, opts crypto.DecrypterOpts) (plaintext []byte, err error)
// 执行一些运算,加快将来私钥操作
func (priv *PrivateKey) Precompute()
// 返回公钥
func (priv *PrivateKey) Public() crypto.PublicKey
// 使用私钥签名,从 rand 读取随机数
// 如果 opts 是 *PSSOptions 将使用 PSS签名,否则使用 PKCS#1 v1.5 
func (priv *PrivateKey) Sign(rand io.Reader, digest []byte, opts crypto.SignerOpts) ([]byte, error)
// 对密钥执行基本的完整性检查.返回检查过程中可能出现的错误
func (priv *PrivateKey) Validate() error
```

### 示例

### 对数据进行加密解密

```go
import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "fmt"
    "os"
)

func main() {
    privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        fmt.Println(err)
    }
    msg := []byte("send reinforcements, we're going to advance")
    label := []byte("orders")
    rng := rand.Reader
    // 使用公钥对数据进行加密
    ciphertext, err := rsa.EncryptOAEP(sha256.New(), rng, &privateKey.PublicKey, msg, label)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error from encryption: %s\n", err)
        return
    }
    fmt.Printf("Ciphertext: %x\n", ciphertext)
    // 使用私钥对数据进行解密
    plaintext, err := rsa.DecryptOAEP(sha256.New(), rng, privateKey, ciphertext, label)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error from deecryption: %s\n", err)
        return
    }
    fmt.Printf("Plaintext: %s\n", string(plaintext))
}
```

#### 对数据进行签名认证

```go
import (
    "crypto"
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "fmt"
    "os"
)

func main() {
    privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
    if err != nil {
        fmt.Println(err)
    }
    message := []byte("message to be signed")
    hashed := sha256.Sum256(message)  // 使用 sha256 进行哈希
    // 使用 rsa 私钥进行签名
    signature, err := rsa.SignPKCS1v15(rand.Reader, privateKey, crypto.SHA256, hashed[:])
    // 或使用 rsa PSS 进行签名
    // signature, err := rsa.SignPSS(rand.Reader, privateKey, crypto.SHA256, hashed[:], nil)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error from signing: %s\n", err)
        return
    }
    fmt.Printf("Signature: %x\n", signature)
    // 使用 rsa 公钥进行认证
    err = rsa.VerifyPKCS1v15(&privateKey.PublicKey, crypto.SHA256, hashed[:], signature)
    // 或使用 rsa PSS 进行认证
    // err = rsa.VerifyPSS(&privateKey.PublicKey, crypto.SHA256, hashed[:], signature, nil)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error from verification: %s\n", err)
        return
    }
    fmt.Println("认证通过")
}
```
