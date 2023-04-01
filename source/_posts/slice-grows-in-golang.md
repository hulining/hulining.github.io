---
title: golang 切片扩容
date: 2023-04-01 18:18:34
tags:
  - go
  - 源码阅读
  - slice
categories:
  - go
description: 'golang 切片扩容计算方式'
---

## 容量计算

在 golang 中,当切片中元素个数大于其容量时,该切片会触发切片扩容.那么扩容后,切片的容量是多少呢?我们应该如何计算呢?

现有如下代码,以 `int32` 与 `int64` 为例分别计算切片扩容后的长度和容量.

```go
import (
 "flag"
 "fmt"
)

func main() {
 var arr32 []int32
 var arr64 []int64
 var num = flag.Int("num", 0, "append element num")
 flag.Parse()
 for i := 0; i < *num; i++ {
  arr32 = append(arr32, int32(i))
 }
 fmt.Println(len(arr32), cap(arr32))

 for i := 0; i < *num; i++ {
  arr64 = append(arr64, int64(i))
 }
 fmt.Println(len(arr64), cap(arr64))
}

```

上述代码在 go/1.18.1 版本之前执行结果如下:

```bash
# go version before 1.18.1

# go run main.go -num 513
513 1024
513 1024

# go run main.go -num  1025
1025 1344
1025 1280
```

在 go/1.18.1 版本之后执行结果如下:

```bash
# go version after 1.18.1

# go run main.go -num 257
257 512
257 512

# go run main.go -num 513
513 864
513 848

# go run main.go -num 1024
1024 1344
1024 1280
```

可以看到切片的扩容后容量大小与 golang 版本及切片中元素类型(主要是元素所占的 bytes 数)有一定的关系.

## 源码阅读

下面我们通过阅读 golang 切片相关源码来搞清楚产生上述差异的原因

### 1.18 之前

以 go/1.17.10 为例,我们来尝试阅读切片扩容的逻辑

```go
func growslice(et *_type, old slice, cap int) slice {
    // 省略部分代码,扩容开始
    // 1. 主要是计算 newcap 的值
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        // 1.1 当 cap 小于 1024 时,扩容为双倍
        if old.cap < 1024 {
            newcap = doublecap
        } else {
            // 1.2 当 cap 大于 1024 时,扩容为之前的 1.25 倍
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // 1.3 当 newcap 溢出时,设置为当前的容量
            if newcap <= 0 {
                newcap = cap
            }
        }
    }

    // 2. 内存对齐.大概计算公式如下:
    // 2.1 计算保存扩容后切片所需的所有 bit 数,计算公式为 `capmem = newcap * element_size`
    // 2.2 将 `capmem` 带入 `roundupsize` 函数,根据此函数取得对应数组中对应的值,该值即内存对齐后该切片所需的最小字节数. `roundupsize` 函数的计算结果可以在 `runtime/sizeclasses.go` 中的注释表中查到,计算结果为表中 `bytes/obj` 列不小于 `capmem` 的最小值
    // 2.3 最后将所需字节数 / element_size 即为扩容后切片的容量
    switch {
    // ... 省略部分代码
    default:
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }

    // 3. 分配内存,并将原来的元素移到新的切片中
    // ... 省略部分代码
    memmove(p, old.array, lenmem)

    return slice{p, old.len, newcap}
}
```

因此计算步骤大致可以总结如下:

1. 根据计算公式获取应该扩容的容量.
   1. 如果当前切片容量小于 1024,扩容切片容量 `newcap = oldcap * 2`
   2. 如果当前切片容量大于等于 1024,扩容切片容量 `newcap = oldcap * 1.25`
2. 内存对齐
   1. 根据上一步得到的容量及元素计算出所需要的字节数 `newcap * element_size`.如 `bytes= newcap * 4(int32)`, `bytes=newcap * 8(int64)`.
   2. 通过 `runtime/sizeclasses.go` 查表,找到 `bytes/obj` 列中大于该 bytes 的最小值.
   3. 使用该值除以 `element_size` 即为扩容后的容量大小

### 1.18 之后

```go
func growslice(et *_type, old slice, cap int) slice {
    // 省略部分代码
    // 1. 与之前的版本类似,同样是计算 newcap 的值,只是在计算公式上有所变动
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        // 1.1 当容量 <256 时,默认为 2 倍扩容
        const threshold = 256
        if old.cap < threshold {
            newcap = doublecap
        } else {
            for 0 < newcap && newcap < cap {
                // 1.2 当旧切片容量不小于 256 时,按照如下公式扩容.这样保证可以在容量较小时双倍扩容,在容量较大时,少量扩容
                newcap += (newcap + 3*threshold) / 4
            }
            // 1.3 同样的,当 newcap 溢出时,设置为当前的容量
            if newcap <= 0 {
                newcap = cap
            }
        }
    }

    // 2. 与 1.18.1 之前一样,做内存对齐
    // ...省略部分代码
    switch {
    
    default:
        // 对齐方式与之前一样,不再赘述
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }

    //3. 分配内存,并将原来的元素移到新的切片中
    // ... 省略部分代码
    memmove(p, old.array, lenmem)

    return slice{p, old.len, newcap}
}
```

因此计算步骤大致可以总结如下:

1. 根据计算公式获取应该扩容的容量.
   1. 如果当前切片容量小于 256 时,扩容切片容量 `newcap = oldcap * 2`
   2. 如果当前切片容量大于等于 1024,扩容切片容量 `newcap = oldcap + (oldcap + 256 * 3) /4`
2. 内存对齐
   1. 根据上一步得到的容量及元素计算出所需要的字节数 `newcap * element_size`.如 `bytes= newcap * 4(int32)`, `bytes=newcap * 8(int64)`.
   2. 通过 `runtime/sizeclasses.go` 查表,找到 `bytes/obj` 列中大于该 bytes 的最小值.
   3. 使用该值除以 `element_size` 即为扩容后的容量大小

## 验证

在 1.18.1 版本之前

```bash
# go version before 1.18.1

# go run main.go -num 513
513 1024
# 对于 int32 类型的切片来说, newcap = 512 * 2 = 1024, bytes = 1024 * 4 = 4096. 查表不小于 bytes 的值刚好为 4096, 因此最终容量为 4096 / 4 = 1024
513 1024
# 对于 int64 类型的切片来说, newcap = 512 * 2 = 1024. bytes = 1024 * 8 = 8192. 查表不小于 bytes 的值刚好为 8192, 因此最终容量为 8192 / 8 = 1024

# go run main.go -num  1025
1025 1344
# 对于 int32 类型的切片来说, newcap = 1024 * 1.25 = 1280, bytes = 1280 * 4 = 5120. 查表不小于 bytes 的值为 5376, 因此最终容量为 5376 / 4 = 1344
1025 1280
# 对于 int64 类型的切片来说, newcap = 1024 * 1.25 = 1280, bytes = 1280 * 8 = 10240. 查表不小于 bytes 的值刚好为 10240, 因此最终容量为 10240 / 8 = 1280
```

在 1.18.1 版本之后

```bash
# go version after 1.18.1

# go run main.go -num 257
257 512
# 对于 int32 类型的切片来说, newcap = 256 + (256 + 256 * 3) / 4 = 512, bytes = 512 * 4 = 2048. 查表不小于 bytes 的值刚好为 2048, 因此最终容量为 2048 / 4 = 512
257 512
# 对于 int32 类型的切片来说, newcap = 256 + (256 + 256 * 3) / 4 = 512, bytes = 512 * 8 = 4096. 查表不小于 bytes 的值刚好为 4096, 因此最终容量为 4096 / 8 = 512

# go run main.go -num 513
513 864
# 对于 int32 类型的切片来说, newcap = 512 + (512 + 256 * 3) / 4 = 832, bytes = 832 * 4 = 3328. 查表不小于 bytes 的值为 3456, 因此最终容量为 3456 / 4 = 864
513 848
# 对于 int32 类型的切片来说, newcap = 512 + (512 + 256 * 3) / 4 = 832, bytes = 832 * 8 = 6656. 查表不小于 bytes 的值为 6784, 因此最终容量为 6784 / 8 = 848

# go run main.go -num 1024
1024 1344
# 需要注意, 当 num 为 1024 时, 对于 int32 类型的切片来说, oldcap = 864.
# newcap = 864 + (864 + 256 * 3) / 4 = 1272 , bytes = 832 * 4 = 5088. 查表不小于 bytes 的值为 5376, 因此最终容量为 5376 / 4 = 1344
1024 1280
# 需要注意, 当 num 为 1024 时, 对于 int32 类型的切片来说, oldcap = 848.
# newcap = 848 + (848 + 256 * 3) / 4 = 1252 , bytes = 1252 * 8 = 10016. 查表不小于 bytes 的值为 10240, 因此最终容量为 10240 / 8 = 1280
```

## 参考

- [浅谈 Go 1.18.1的切片扩容机制](https://yufengbiji.com/posts/golang-slice)
- [go1.17.10-growslice](https://github.com/golang/go/blob/go1.17.10/src/runtime/slice.go#L162-L277)
- [go1.18.1-growslice](https://github.com/golang/go/blob/go1.18.1/src/runtime/slice.go#L166-L288)
- [sizeclasses 表](https://github.com/golang/go/blob/master/src/runtime/sizeclasses.go)
