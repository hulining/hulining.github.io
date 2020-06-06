---
title: go 学习笔记之 strings 包
date: 2020/05/04
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 33042
description: 本文章主要包含 Go strings 包及其内置类型和方法的使用.
---

`strings` 包实现了用于操作字符的简单函数.导入方式为 `import "strings"`

# 常用类型定义

```go
// 从字符串读取数据的封装,实现了 `io.Reader`,`io.Seeker`,`io.ReaderAt`,`io.WriterTo``io.ByteScanner`,`io.RuneScanner` 接口
type Reader struct {
    // 内含隐藏或非导出字段
}

// 进行字符替换的封装
type Replacer struct {
    // 内含隐藏或非导出字段
}
```

# 常用函数

# `strings` 包常用函数

`strings` 包中提供了如下常用函数

```go
// 创建从指定字符串读取数据的 Reader 对象实例.返回 Reader 对象的指针
func NewReader(s string) *Reader
// 使用多组 old,new 字符串创建 Replacer 对象.返回 Replacer 对象的指针
func NewReplacer(oldnew ...string) *Replacer
// 判断字符串 s 是否有 prefix 前缀
func HasPrefix(s, prefix string) bool
// 判断字符串 s 是否有 suffix 后缀
func HasSuffix(s, suffix string) bool
// 判断字符串 s 是否包含子串 substr
func Contains(s, substr string) bool
// 判断字符串 s 是否包含字符串 chars 中的任一字符
func ContainsAny(s, chars string) bool
// 字符串 s 中包含 sep 子串的个数
func Count(s, sep string) int
// 子串 sep 在 s 中第一次出现的位置, 不存在则返回 -1
func Index(s, sep string) int
// 字符 c 在 s 中第一次出现的位置,不存在则返回 -1
func IndexByte(s string, c byte) int
// 子串 sep 在 s 中最后一次出现的位置, 不存在则返回 -1
func LastIndex(s, chars string) int
// s 中每个单词的首字母都改为标题格式的字符串拷贝
func Title(s string) string
// 将所有字母都转为对应的小写字母的拷贝
func ToLower(s string) string
// 将所有字母都转为对应的大写字母的拷贝
func ToUpper(s string) string
// 返回 count 个 s 串联的字符串
func Repeat(s string, count int) string
// 将 s 中前 n 个 old 子串都替换为 new 的新字符串, 如果 n<0 会替换所有 old 子串
func Replace(s, old, new string, n int) string
// 返回将 s 前后 cutset 都删除的字符串
func Trim(s string, cutset string) string
// 返回将 s 前后空白都删除的字符串
func TrimSpace(s string) string
// 返回将 s 前面 cutset 删除的字符串
func TrimLeft(s string, cutset string) string
// 返回将 s 后面 cutset 删除的字符串
func TrimRight(s string, cutset string) string
// 使用 sep 作为分隔符将 s 分割,返回分割后的字符串切片
func Split(s, sep string) []string
// 返回分割后的字符串切片.n>0,将 s 分割为 n 项, 最后一个子字符串包含未进行切割的部分, 并.如果 n 等于 0 , 返回 nil. n<0,返回所有字符串切片
func SplitN(s, sep string, n int) []string
// 将字符串切片 a 连接成一个字符串,中间用 sep 分割
func Join(a []string, sep string) string
```

## `Reader` 结构体方法

```go
// 以下是 `io` 包中的相关接口的实现
func (r *Reader) Read(b []byte) (n int, err error)
func (r *Reader) ReadByte() (b byte, err error)
func (r *Reader) Seek(offset int64, whence int) (int64, error)
func (r *Reader) ReadAt(b []byte, off int64) (n int, err error)
func (r *Reader) WriteTo(w io.Writer) (n int64, err error)
```

## `Replacer` 结构体方法

```go
// 返回对字符串 s 替换进行完后的拷贝
func (r *Replacer) Replace(s string) string
// 向 w 中写入 s 的所有替换进行完后的拷贝
func (r *Replacer) WriteString(w io.Writer, s string) (n int, err error)
```