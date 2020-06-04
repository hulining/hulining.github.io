---
title: go 学习笔记之 bufio 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 54358
description: '本文章主要包含 Go bufio 包及其内置类型和方法的使用.bufio 包实现了缓冲的 IO,提供了 IO 缓冲和文本类型 IO 的一些支持'
---

`bufio` 包实现了缓冲的 IO. 它包含分别实现 `io.Reader()` 和 `io.Writer()` 接口的 `Reader` 和 `Writer`对象,提供了 IO 缓冲区和文本类型 IO 的一些支持.导入方式为 `import "bufio"`

# 常用类型定义

```go
// Reader 是实现了 `io.Reader` 接口的缓冲对象
type Reader struct {
    // 内含隐藏或非导出字段
}

// Writer 是实现了 `io.Writer` 接口的缓冲对象.
// 如果在写入 Writer 时发生错误,将不再接收数据,所有后续写入和 `Flush()`` 都将返回错误
// 写入所有数据后,客户端应显示调用 `Flush()` 方法确保所有数据都发送到基本 `io.Writer`
type Writer struct {
    // 内含隐藏或非导出字段
}

type ReadWriter struct {
    *Reader
    *Writer
}

// Scanner 提供了一些方便的接口来扫描缓冲区数据,如用换行符分隔行的文件.
// 通过连续调用 `Scan()` 方法将逐步浏览文件 'token'(文件内容),并跳过 token 之间的字节.token 的规范是由 `SplitFunc` 类型的分割函数定义的.默认的分割函数将输入分割成行,并去掉行尾的换行标志.
// 预定义的分割函数可以将输入分割成行,字节,unicode,空白分隔的 word.调用者可以定制自己的分割函数
// 
// 需要更多对错误管理的控制或 token 很大,或必须从 reader 连续读取的程序,应使用 `bufio.Reader`代替
type Scanner struct {
    // 内含隐藏或非导出字段
}

// 用于 Scanner 做词法分析的分割函数
// 函数的返回值是每次扫描的长度,分割后的字符(token)的切片表示及可能发生的错误
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)  
```

# 常用常量及变量

```go
const (
    // 用于缓冲一个 token,实际需要的最大 token 尺寸可能小一些,例如缓冲中需要保存一整行内容
    MaxScanTokenSize = 64 * 1024
)
```

# 常用函数

## `bufio` 包函数

```go
// 返回一个带有默认大小缓冲区的 `Reader` 对象
func NewReader(rd io.Reader) *Reader
// 返回一个带有指定大小缓冲区的 `Reader` 对象,默认值为 4096,最小值为 16
func NewReaderSize(rd io.Reader, size int) *Reader
// 返回一个带有默认大小缓冲区的 `Writer` 对象
func NewWriter(w io.Writer) *Writer
// 返回一个带有指定大小缓冲区的 `Writer` 对象
func NewWriterSize(w io.Writer, size int) *Writer
// 使用 Reader,Writer 对象创建一个新的 ReadWriter 对象 
func NewReadWriter(r *Reader, w *Writer) *ReadWriter


// 以下预定义函数用于 `scanner.Split()` 参数传递,是 Scanner 的分割函数

// 将缓冲区数据流按照字节分割作为 token
func ScanBytes(data []byte, atEOF bool) (advance int, token []byte, err error)
// 将缓冲区数据流按照 runes 分割作为 token
func ScanRunes(data []byte, atEOF bool) (advance int, token []byte, err error)
// 将缓冲区数据流按照单词分割作为 token,分割符号为空白字符
func ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error)
// 将缓冲区数据流按行分割作为 token,分割符号为 '\n'
func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)
```

## `Reader` 结构体方法

```go
// 丢弃缓冲区中的数据,清除任何错误,将 b 设置为从 r 读取数据
func (b *Reader) Reset(r io.Reader)
// 返回缓冲区现有的可读取的字节数
func (b *Reader) Buffered() int
// 返回缓冲区的下 n 个字节及可能发生的错误.此函数不会移动读取指针位置
func (b *Reader) Peek(n int) ([]byte, error)
// 读取缓冲区数据写入到 p.返回写入 p 的字节数及可能发生的错误.读取到结尾时,n 为 0 且 err 为 io.EOF
func (b *Reader) Read(p []byte) (n int, err error)
// 读取缓冲区数据并返回一个字节及可能发生的错误
func (b *Reader) ReadByte() (c byte, err error)

// 按行读取,返回读取到的 byte 切片(不包括行尾换行符号),是否未读到行尾及发生的错误
// 如果行超过缓冲区大小,则一次只读取缓冲区大小的数据,且 isPrefix 为 true.本行的剩下的数据留作下一次读取,直到行读取结束,isPrefix 返回 false
// 多使用 `ReadBytes('\n')` 或 `ReadString('\n')` 方法代替
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
// 读取数据遇到 delim 字符.返回读取的字符切片及可能发生的错误
func (b *Reader) ReadBytes(delim byte) (line []byte, err error)
// 读取缓冲区数据直到遇到 delim 字符.返回读取到的字符串表示及可能发生的错误.实际是调用了 ReadBytes 方法
func (b *Reader) ReadString(delim byte) (line string, err error)
// 将缓冲数据流写入到 w
func (b *Reader) WriteTo(w io.Writer) (n int64, err error)  
```

## `Writer` 结构体方法

```go
func (b *Writer) Reset(w io.Writer)  // 丢弃缓冲中的数据,清除任何错误,将 b 设置为向 w 写入数据
func (b *Writer) Buffered() int  // 返回缓冲中现有的可写入的字节数
func (b *Writer) Write(p []byte) (nn int, err error)  // 将 p 的内容写入缓冲.返回写入的字节数及可能发生的错误.如果 nn < len(p),err 不为空
func (b *Writer) WriteString(s string) (int, error)  // 将 s 写入缓冲.返回写入的字节数及可能发生的错误.如果 nn < len(p),err 不为空
func (b *Writer) WriteByte(c byte) error  // 将单个字节 c 写入缓冲,返回可能发生的错误
func (b *Writer) Flush() error  // 将缓冲区数据持久化到文件中.返回可能发生的错误
func (b *Writer) ReadFrom(r io.Reader) (n int64, err error)  // 实现 `io.ReaderFrom` 接口.从 r 中读取数据到缓冲区
```

## `Scanner` 结构体方法

```go
// 设置 Scanner 的分割函数.必须在 `Scan()` 之前调用
func (s *Scanner) Split(split SplitFunc)
// 获取当前位置的 token.让 Scanner 的扫描位置移动到下一个 token
func (s *Scanner) Scan() bool
// 返回字符切片格式的最近一次 `Scan()` 方法调用生成的 token
func (s *Scanner) Bytes() []byte
// 返回字符串格式的最近一次 `Scan()` 方法调用生成的 token
func (s *Scanner) Text() string
// 返回 Scanner 扫描过程中发生的错误
func (s *Scanner) Err() error
```

# 示例

现有一文件 `filename` 内容如下:
```
hello
the second line: nice to meet you
the third line: see you next time
```

## `bufio` 读写示例

```go
import (
    "bufio"
    "fmt"
    "io"
    "os"
)

// `bufio.Writer.WriteString()` 方法使用示例
func WriterExample(bw *bufio.Writer) {
    defer bw.Flush() // 需要显式将缓冲区中数据持久化到文件
    bw.WriteString("this new add line\n")
}

// `bufio.Reader.ReadLine()` 方法使用示例
func ReadLineExample(br *bufio.Reader) {
    for {
        line, isPrefix, err := br.ReadLine()
        if err == io.EOF {
            break
        }
        if isPrefix {
            fmt.Printf("该行未结束,读到内容为: %v", string(line))
        } else {
            fmt.Printf("该行结束,读到或剩余内容为: %v", string(line))
        }
        fmt.Println() // 读取内容行尾的换行符不被保留,需手动换行
    }
}

// `bufio.Reader.ReadString()` 方法使用示例
func ReadStringExample(br *bufio.Reader) {
    for {
        line, err := br.ReadString('\n') // 使用 ReadString 按行读取
        if err == io.EOF {
            fmt.Print(line) // 最后一行若没有换行符号,err 为 io.EOF,需要打印最后一行
            break
        }
        fmt.Print(line)
    }
}


func main() {
    file, err := os.OpenFile("filename", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0)
    if err != nil {
        fmt.Println("err", err)
        return
    }
    defer file.Close()

    br := bufio.NewReaderSize(reader, 16) // 设置为最小缓冲区,用于演示效果
    ReadLineExample(br)
    file.Seek(0, 0)  // 将文件指针移到开始位置
    fmt.Println("------")
    ReadStringExample(br)
    file.Seek(0, 0)
    bw := bufio.NewWriter(file)
    WriterExample(bw)
}

// 输出为以下内容: 
// 该行结束,读到或剩余内容为: hello
// 该行未结束,读到内容为: the second line:
// 该行未结束,读到内容为:  nice to meet yo
// 该行结束,读到或剩余内容为: u
// 该行未结束,读到内容为: the third line: 
// 该行未结束,读到内容为: see you next tim
// 该行结束,读到或剩余内容为: e
// ------
// hello
// the second line: nice to meet you
// the third line: see you next time
```
通过以上示例可以看到:
- `ReadLine()` 超过缓冲区大小的数据被分为多次读取,且行尾的换行符不被保留
- 文件增加 "this new add line\n" 内容

## `bufio.Scanner` 扫描示例

- 统计单词个数
```go
import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    const input = "Now is the winter of our discontent,\nMade glorious summer by this sun of York.\n"
    scanner := bufio.NewScanner(strings.NewReader(input))
    scanner.Split(bufio.ScanWords) // 为 scanner 设置分割函数
    // 统计单个个数
    count := 0
    for scanner.Scan() {
        count++
    }
    if err := scanner.Err(); err != nil {
        fmt.Fprintln(os.Stderr, "reading input:", err)
    }
    fmt.Printf("%d\n", count)
}
```

- 使用自定义分割函数

如下示例尝试将分割分割的字符串转换为数字格式

```go
import (
    "bufio"
    "fmt"
    "strconv"
    "strings"
)

func main() {
    const input = "1234 5678 5681 1234567890123456"
    scanner := bufio.NewScanner(strings.NewReader(input))
    // 创建自定义分割函数
    split := func(data []byte, atEOF bool) (advance int, token []byte, err error) {
        advance, token, err = bufio.ScanWords(data, atEOF)  // 获取到以空格分割的 token
        if err == nil && token != nil {
            _, err = strconv.ParseInt(string(token), 10, 32)  // 尝试转换为十进制数字
        }
        return
    }
    // 设置分割函数为自定义的分割函数 split
    scanner.Split(split)
    // 开始扫描,并打印扫描到的文本
    for scanner.Scan() {
        fmt.Printf("%s\n", scanner.Text())
    }
    if err := scanner.Err(); err != nil {
        fmt.Printf("Invalid input: %s", err)
    }
}
```