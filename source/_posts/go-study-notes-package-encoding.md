---
title: go 学习笔记之 encoding 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 10194
description: '本文章主要包含 Go encoding 及其子包的内置函数的使用.encoding 包定义了其它包共享的接口,这些包将数据与字节表示形式和文本形式的互相转换.'
---

`encoding` 包定义了其它包共享的接口,其子包实现了数据与字节表示形式和文本形式的互相转换,主要包括:

- `encoding/bash64` 实现了bas64 格式编码与解码.
- `encoding/json` 实现了 json 格式数据的编码与解码.
- `encoding/pem` 实现了 PEM 数据编码,常用于 TLS 密钥和证书中
- `encoding/xml` 实现了 xml 格式数据的编码与解码

# `encoding/base64` 包

## 常量及变量

```go
const encodeStd = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
const encodeURL = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_"
const (
    StdPadding rune = '=' // 标准填充字符
    NoPadding  rune = -1  // 不填充
)
// 未填充的标准原生编码格式
var RawStdEncoding = StdEncoding.WithPadding(NoPadding)
// 未填充的 URL 编码格式,常用于 URL 或文件名中
var RawURLEncoding = URLEncoding.WithPadding(NoPadding)
// 标准原生编码格式
var StdEncoding = NewEncoding(encodeStd)
// 标准 URL 编码格式,常用于 URL 或文件名中
var URLEncoding = NewEncoding(encodeURL)
```

## 常用函数

- 包函数

```go
// 返回给定字母定义的填充 Encoding 对象,可以使用该对象的 WithPadding 方法修改填充字符或禁用填充字符
func NewEncoding(encoder string) *Encoding
```

- `Encoding` 结构体方法

```go
// 对已编码的数据 src 进行解码,最多将 DecodedLen(len(src)) 个字节写入 dst
func (enc *Encoding) Decode(dst, src []byte) (n int, err error)
// 对已编码后的 s 进行解码,返回解码后的数据
func (enc *Encoding) DecodeString(s string) ([]byte, error)
// 对数据 src 进行编码,最多将 DecodedLen(len(src)) 个字节写入 dst
func (enc *Encoding) Encode(dst, src []byte)
// 对数据 src 进行编码,返回编码后的字符串表示
func (enc *Encoding) EncodeToString(src []byte) string
// 修改填充字符或禁用填充字符
func (enc Encoding) WithPadding(padding rune) *Encoding
```

## 示例

```go
import (
	"encoding/base64"
	"fmt"
)

func main() {
	dataForEncoding := []byte("string for encoding")
	strAfterEncode := base64.StdEncoding.EncodeToString(dataForEncoding)
    fmt.Println(strAfterEncode)
    
    strForDecode := "c29tZSBkYXRhIHdpdGggACBhbmQg77u/"
	dataAfterDecode, err := base64.StdEncoding.DecodeString(strForDecode)
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Printf("%q\n", dataAfterDecode)
}
```

# `encoding/json` 包

`encoding/json` 实现了 json 格式数据的编码与解码.主要用于对象实例与 JSON 格式数据间的相互转换.

```go
// 将对象实例转换为 JSON 格式数据
func Marshal(v interface{}) ([]byte, error)
// 将对象实例转换为 JSON 格式数据,每个 JSON 元素以新的缩进行及 prefix 开头,且根据嵌套深度跟一个或多个 indent 副本,可用于美化输出
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error)
// 将 JSON 格式数据转换为对象实例
func Unmarshal(data []byte, v interface{}) error
```

当结构体定义时,字段带有 `json: "jsonField"` 格式的标签,则会按照 `jsonField` 名称解析对象字段.示例如下:

```go
import (
	"encoding/json"
	"fmt"
	"os"
)

type ColorGroup struct {
	ID     int      `json:"id"`
	Name   string   `json:"name"`
	Colors []string `json:"colors"`
}

func main() {
	group := ColorGroup{
		ID:     1,
		Name:   "Reds",
		Colors: []string{"Crimson", "Red", "Ruby", "Maroon"},
	}
	b, err := json.MarshalIndent(group, "", "  ")
	if err != nil {
		fmt.Println("error:", err)
	}
	os.Stdout.Write(b)
	fmt.Println()

	color := []byte(`{"id":2,"name":"Greens","colors":["Crimson","Green"]}`)
	var colorGroup ColorGroup
	err = json.Unmarshal(color, &colorGroup)
	fmt.Println(colorGroup)
}
```

# `encoding/pem` 包

`pem` 包实现了 PEM 数据编码,常用于 TLS 密钥和证书中

```go
// 可以用于 PEM 编码的结构体,编码后格式如下
/*
-----BEGIN Type-----
Headers
base64-encoded Bytes
-----END Type-----
*/
type Block struct {
    Type    string            // The type, taken from the preamble (i.e. "RSA PRIVATE KEY").
    Headers map[string]string // Optional headers.
    Bytes   []byte            // The decoded bytes of the contents. Typically a DER encoded ASN.1 structure.
}
```

```go
// 将 b PEM 编码后的数据写入 out
func Encode(out io.Writer, b *Block) error
// 对 PEM 编码后的数据进行解码,返回 Block 对象,一般带有 TLS 私钥或公钥
func Decode(data []byte) (p *Block, rest []byte)
```


# `encoding/xml` 包

与 `encoding/json` 类似,`encoding/xml` 实现了 xml 格式数据的编码与解码.主要用于对象实例与 xml 格式数据间的相互转换.

```go
// 将对象实例转换为 xml 格式数据
func Marshal(v interface{}) ([]byte, error)
// 将对象实例转换为 xml 格式数据,每个 xml 元素以新的缩进行及 prefix 开头,且根据嵌套深度跟一个或多个 indent 副本,可用于美化输出
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error)
// 将 xml 格式数据转换为对象实例
func Unmarshal(data []byte, v interface{}) error
```

当结构体定义时,字段带有 `xml: "xmlField"` 格式的标签,则会按照 `xmlField` 名称解析对象字段.示例如下:

```go
import (
	"encoding/xml"
	"fmt"
	"os"
)

type Address struct {
	City, State string
}
type Person struct {
	XMLName   xml.Name `xml:"person"`
	Id        int      `xml:"id,attr"`
	FirstName string   `xml:"name>first"`
	LastName  string   `xml:"name>last"`
	Age       int      `xml:"age"`
	Height    float32  `xml:"height,omitempty"`
	Married   bool
	Address
	Comment string `xml:",comment"`
}

func main() {
	person := &Person{Id: 13, FirstName: "John", LastName: "Doe", Age: 42}
	person.Comment = " Need more details. "
	person.Address = Address{"Hanga Roa", "Easter Island"}

	output, err := xml.MarshalIndent(person, "", "  ")
	if err != nil {
		fmt.Printf("error: %v\n", err)
	}
	os.Stdout.Write(output)
}
```

