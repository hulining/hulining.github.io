---
title: Bash 特性
date: 2020/12/7
tags:
  - Linux
  - Bash
categories:
  - Linux
abbrlink: 
description: 汇总常见的 Bash 特性,以作备忘
---

## Shell 语法

### 复合命令

```bash
(list) # 表示在子 shell 中运行命令
{ list; } # 表示仅在本 shell 中运行命令.且命令必须以 `;` 结尾,命令与 `{}` 之间必须有空格
((expression)) # 多用于算术运算
[[ expression ]] # 多用于条件判断
```

### 带有空格的文件名处理

```bash
find . -type f -name "*[[:space:]]*" -print | 
while read name ;do
  new=`echo $name | sed s'@[[:space:]]@_@'g`;
  echo "mv $name $new" ;
  mv "$name" $new;
done

# rename
```

## Parameter

### 位置参数

```bash
$* # 参数列表
$@ # 参数列表
$# # 参数个数
$? # 最后运行命令的结束代码
$$ # shell 本身的 processID. () 中运行的子 shell 返回值也为本 shell 的 processID
$0 # shell 脚本或命令本身的文件名
```

## 扩展

### 参数扩展

`$` 符号用于参数扩展,命令替换或算术扩展.可以使用花括号(`{}`)将要扩展的参数名称或符号括起来,该括号是可选的,但要保证扩展的变量不受紧随其后字符的影响.

```bash
${parameter} # 原始格式,表示变量值.如果第一个字符是感叹号(!),则会引入变量间接访问.如 name=admin;admin=pass,则 ${!name} 的值为 pass
${parameter:-word} # 如果 parameter 没有设置或为空,则使用默认值为 word.(parameter 值不变)
${parameter:=word} # 如果 parameter 没有设置或为空,则将 parameter 的值设置为 word
${parameter:?word} # 如果 parameter 没有设置或为空,则将 word 作为错误信息输出
${parameter:+word} # 如果 parameter 不为空,则输出 word 的值.(parameter 值不变)
${parameter:offset} # 子字符串, offset 从 0 开始
${parameter:offset:length} # 子字符串, offset 从 0 开始,子字符串长度为 length
${#parameter} # parameter 变量值的长度
${parameter/pattern/string} # 使用 string 替换 parameter 中 pattern 的模式匹配

${parameter^pattern} # ^ 表示将模式匹配的内容转换为大写.
${parameter^^pattern} # ${parameter^^} 将所有内容转换为大写输出 
${parameter,pattern} # ,表示将模式匹配的内容转换为小写
${parameter,,pattern} # ${parameter,,} 将所有内容转换为小写输出
```

### 算术运算

```bash
$((expression))
sum=$[ ${v1} + ${v2} ]
(( sum=${v1} + ${v2} ))
let sum=${v1}+${v2}     # 这里运算符两端必须没有空格
`expr ${v1} + ${v2}`    # 这里运算符号两端必须要有空格
```

## 条件判断语句

Bash 中的条件判断语句使用 `[[ 复合命令 ]]`,`test` 和 `[ 内置命令 ]` 对文件属性,字符串进行判断或进行算术比较.

### 文件相关

下面是常用的文件属性进行条件判断语句

```bash
-a file # 文件存在,则返回 True
-b file # 文件存在且是块文件,则返回 True
-c file # 文件存在且是字符文件,则返回 True
-d file # 文件存在且是目录,则返回 True
-e file # 文件存在,则返回 True
-f file # 文件存在且是文本文件,则返回 True
-h file # 文件存在且是链接文件,则返回 True
-p file # 文件存在且是管道文件,则返回 True
-r file # 文件存在且可读,则返回 True
-s file # 文件存在且大小不为0,则返回 True
-w file # 文件存在且可写,则返回 True
-x file # 文件存在且可执行,则返回 True
-L file # 文件存在且是链接文件,则返回 True
-S file # 文件存在且是套接字文件,则返回 True
```

### 字符串与算术运算相关

```bash
-v varname # 设置了 varname 变量,则返回 True
-z string # 字符串长度为 0,则返回 True
-n string # 字符串长度不为 0,则返回 True
string1 == string2 # 两个字符串相等,则返回 True
string1 != string2
string1 < string2
string1 > string2

arg1 OP arg2 # OP 支持的方式为 -eq, -ne, -lt, -le, -gt, -ge
```
