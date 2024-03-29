---
title: go 学习笔记之 x509 包
date: 2020/05/22
tags:
  - go
categories:
  - go
abbrlink: 42320
description: 本文章主要包含 Go crypto/x509 包及其部分子包内置类型和方法的使用.crypto/x509 包解析X.509编码密钥和证书
---

`crypto/x509` 包主要用于解析 X.509 编码的密钥和证书.

`crypto/x509/pkix` 包含用于 X.509 证书,CRL 和 OCSP 的 ASN.1 解析和序列化的共享低级结构

在 Unix 系统上,可使用环境变量 SSL_CERT_FILE 和 SSL_CERT_DIR 覆盖系统证书文件和证书文件目录的默认位置

## 类型定义

```go
// certificates 证书的集合
type CertPool struct {
    // contains filtered or unexported fields
}

// X509 证书
type Certificate struct {
    // 证书的 subject 信息,包含 CN,O,L,S,C等字段
    Subject pkix.Name
    // contains filtered or unexported fields
}
// key 的扩展用法
type ExtKeyUsage int
// Key 的用法
type KeyUsage int

// 证书签署请求
type CertificateRequest struct {
    // 签名
    Signature          []byte
    // 签名算法
    SignatureAlgorithm SignatureAlgorithm
    // 私钥算法
    PublicKeyAlgorithm PublicKeyAlgorithm
    // 私钥
    PublicKey          interface{}
    // 证书签署请求的 subject
    Subject pkix.Name
    // contains filtered or unexported fields
}

// 验证的选项
type VerifyOptions struct {
    DNSName       string
    Intermediates *CertPool
    Roots         *CertPool // 默认使用系统根证书
    CurrentTime   time.Time // 当前时间
    KeyUsages []ExtKeyUsage
    MaxConstraintComparisions int
}
```

## 常量及变量

```go
// 私钥算法
const (
    UnknownPublicKeyAlgorithm PublicKeyAlgorithm = iota
    RSA
    DSA
    ECDSA
    Ed25519
)
// 签名算法
const (
    UnknownSignatureAlgorithm SignatureAlgorithm = iota
    MD2WithRSA
    MD5WithRSA
    SHA1WithRSA
    SHA256WithRSA
    SHA384WithRSA
    SHA512WithRSA
    DSAWithSHA1
    DSAWithSHA256
    ECDSAWithSHA1
    ECDSAWithSHA256
    ECDSAWithSHA384
    ECDSAWithSHA512
    SHA256WithRSAPSS
    SHA384WithRSAPSS
    SHA512WithRSAPSS
    PureEd25519
)
// 扩展 Key 用法
const (
    ExtKeyUsageAny ExtKeyUsage = iota
    ExtKeyUsageServerAuth
    ExtKeyUsageClientAuth
    ExtKeyUsageCodeSigning
    ExtKeyUsageEmailProtection
    ExtKeyUsageIPSECEndSystem
    ExtKeyUsageIPSECTunnel
    ExtKeyUsageIPSECUser
    ExtKeyUsageTimeStamping
    ExtKeyUsageOCSPSigning
    ExtKeyUsageMicrosoftServerGatedCrypto
    ExtKeyUsageNetscapeServerGatedCrypto
    ExtKeyUsageMicrosoftCommercialCodeSigning
    ExtKeyUsageMicrosoftKernelCodeSigning
)
// key 用法
const (
    KeyUsageDigitalSignature KeyUsage = 1 << iota
    KeyUsageContentCommitment
    KeyUsageKeyEncipherment
    KeyUsageDataEncipherment
    KeyUsageKeyAgreement
    KeyUsageCertSign
    KeyUsageCRLSign
    KeyUsageEncipherOnly
    KeyUsageDecipherOnly
)
```

## 常用函数

### `x509` 包函数

```go
// 基于 template 创建一个新的由 parent 签署的 X509 v3 证书.返回 DER 编码的证书
// 如果 parent=template,则该证书是自签证书.
// 如果是自签证书生成证书的 AuthorityKeyId 会从父级的 SubjectKeyId 中获取,否则使用 template 中的值
// pub,priv 分别为签署人的公钥和私钥, priv 必须是实现 `crypto.Signal` 接口的 `PrivateKey` 类型.当前支持的类型为 *rsa.PublicKey,*ecdsa.PublicKey 和 *ed25519.PublicKey.
func CreateCertificate(rand io.Reader, template, parent *Certificate, pub, priv interface{}) (cert []byte, err error)
// 基于 template 创建一个证书签署请求
// priv 是用于签署 CSR 的私钥,并将其公钥放在 CSR 中,priv 必须实现 `crypto.Signal` 接口
func CreateCertificateRequest(rand io.Reader, template *CertificateRequest, priv interface{}) (csr []byte, err error)
// 使用 password 对 PEM 块进行解密,返回 DER 编码的字节切片
func DecryptPEMBlock(b *pem.Block, password []byte) ([]byte, error)
// 使用指定 alg 和 password 对给定 DER 编码的 data 进行加密.返回加密后的 PEM 块
func EncryptPEMBlock(rand io.Reader, blockType string, data, password []byte, alg PEMCipher) (*pem.Block, error)
// 判断 PEM 块是否使用密码加密
func IsEncryptedPEMBlock(b *pem.Block) bool

// 返回一个空的 CertPool
func NewCertPool() *CertPool
// 返回系统证书池的副本.CertPool 的任何变动都不会写入磁盘
func SystemCertPool() (*CertPool, error)

// 从 ASN.1 数据中解析证书.返回证书或证书列表
func ParseCertificate(asn1Data []byte) (*Certificate, error)
func ParseCertificates(asn1Data []byte) ([]*Certificate, error)
```

### `CertPool` 结构体方法

```go
// 向证书池中添加证书
func (s *CertPool) AddCert(cert *Certificate)
// 尝试解析 PEM 编码的证书,并将其添加到证书池中
func (s *CertPool) AppendCertsFromPEM(pemCerts []byte) (ok bool)
// 返回证书池中所有证书的 DER 编码的 subject 的列表
func (s *CertPool) Subjects() [][]byte
```

### `Certificate` 结构体方法

```go
// 检查 crl 中签名是否来自 c
func (c *Certificate) CheckCRLSignature(crl *pkix.CertificateList) error
// 使用指定签名算法 algo 验证 signature 是否是通过 c 的私钥签署了 signed 签名
func (c *Certificate) CheckSignature(algo SignatureAlgorithm, signed, signature []byte) error
// 验证 c 上的签名是否是来自 parent 的有效签名
func (c *Certificate) CheckSignatureFrom(parent *Certificate) error
// 返回证书签名的 DER 编码的 CRL 证书吊销列表,包含指定的的吊销证书列表
func (c *Certificate) CreateCRL(rand io.Reader, priv interface{}, revokedCerts []pkix.RevokedCertificate, now, expiry time.Time) (crlBytes []byte, err error)
// 使用 opts.Roots 中的证书,验证 c
func (c *Certificate) Verify(opts VerifyOptions) (chains [][]*Certificate, err error)
// 验证 c 是否是指定主机的有效证书
func (c *Certificate) VerifyHostname(h string) error
```

## 示例

### 生成自签书,并使用自签证书签署

```go
import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "crypto/x509/pkix"
    "io/ioutil"
    "log"
    "math/big"
    "time"
)

func main() {
    // ca 证书
    ca := &x509.Certificate{
        SerialNumber: big.NewInt(1653),
        Subject: pkix.Name{
            Country:            []string{"China"},
            Organization:       []string{""},
            OrganizationalUnit: []string{""},
        },
        NotBefore:             time.Now(),
        NotAfter:              time.Now().AddDate(10, 0, 0),
        SubjectKeyId:          []byte{1, 2, 3, 4, 5},
        BasicConstraintsValid: true,
        IsCA:                  true,
        ExtKeyUsage:           []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth, x509.ExtKeyUsageServerAuth},
        KeyUsage:              x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
    }
    // 私钥及公钥 rsa 格式
    caSelfSignedPrivateKey, _ := rsa.GenerateKey(rand.Reader, 1024)
    caSelfSignedPublicKey := &caSelfSignedPrivateKey.PublicKey
    // 自签证书 []byte
    caSelfSigned, err := x509.CreateCertificate(rand.Reader, ca, ca, caSelfSignedPublicKey, caSelfSignedPrivateKey)
    if err != nil {
        log.Println("create ca failed", err)
        return
    }
    caSelfSignedFile := "ca.pem"
    log.Println("write to", caSelfSignedFile)
    ioutil.WriteFile(caSelfSignedFile, caSelfSigned, 0777) // 将自签证书写入文件

    caSelfSignedPrivateKeyFile := "ca.key"
    caSelfSignedPrivateKeyDER := x509.MarshalPKCS1PrivateKey(caSelfSignedPrivateKey) // 将私钥转换为 DER 格式
    log.Println("write to", caSelfSignedPrivateKeyFile)
    ioutil.WriteFile(caSelfSignedPrivateKeyFile, caSelfSignedPrivateKeyDER, 0777) // 将 DER 编码私钥写入文件

    // 待签署证书及其私钥公钥
    cert := &x509.Certificate{
        SerialNumber: big.NewInt(1658),
        Subject: pkix.Name{
            Country:            []string{"China"},
            Organization:       []string{""},
            OrganizationalUnit: []string{""},
        },
        NotBefore:    time.Now(),
        NotAfter:     time.Now().AddDate(10, 0, 0),
        SubjectKeyId: []byte{1, 2, 3, 4, 6},
        ExtKeyUsage:  []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth, x509.ExtKeyUsageServerAuth},
        KeyUsage:     x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
    }
    certPrivateKey, _ := rsa.GenerateKey(rand.Reader, 1024)
    certPublicKey := &certPrivateKey.PublicKey

    // 使用自签CA 对 证书签署
    certSigned, err2 := x509.CreateCertificate(rand.Reader, cert, ca, certPublicKey, caSelfSignedPrivateKey)
    if err != nil {
        log.Println("create cert2 failed", err2)
        return
    }

    certFile := "cert.pem"
    log.Println("write to", certFile)
    ioutil.WriteFile(certFile, certSigned, 0777) // cert 写入文件

    certPrivateKeyFile := "cert.key"
    certPrivateKeyDER := x509.MarshalPKCS1PrivateKey(certPrivateKey) // 将私钥转换为 DER 编码格式
    log.Println("write to", certPrivateKeyFile)
    ioutil.WriteFile(certPrivateKeyFile, certPrivateKeyDER, 0777) // 私钥写入文件

    ca_tr, _ := x509.ParseCertificate(caSelfSigned)
    cert_tr, _ := x509.ParseCertificate(certSigned)
    err = cert_tr.CheckSignatureFrom(ca_tr)
    log.Println("check signature", err)
}
```
