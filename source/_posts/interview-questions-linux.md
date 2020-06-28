---
title: 运维面试题之 Linux
date: 2020/06/04
tags:
  - Linux
  - 面试
categories:
  - Linux
abbrlink: 
description: 总结整理常见 Linux 面试题,以作备忘
---

## 命令或 Bash 相关

### Bash 中两个数做运算的几种方式

```bash
sum=$[ ${v1} + ${v2} ]
(( sum=${v1} + ${v2} ))
let sum=${v1}+${v2}     # 这里运算符两端必须没有空格
`expr ${v1} + ${v2}`    # 这里运算符号两端必须要有空格
```

### `$?,$*` 等相关参数都有什么意义

```bash
$$: shell 本身的 processID
$?: 最后运行命令的结束代码
$0: shell 脚本文件名
$*: 参数列表
$@: 参数列表
$#: 参数个数
${#str}: str 变量字符长度
```

### `read` 命令从管道中读取字节流

```bash
echo "sinaops" | read a ; echo $a # 输出为空
echo "sinaops" | while read a ; do echo $a ; done # 输出 sinaops
```

### `find` 使用

```bash
# 把 /data 目录及其子目录下所有以扩展名 .txt 结尾的文件中包含 magedu 的字符串全部替换为 magestudy
# {} 表示找到的文件
find /data -type f -name "*.txt" -exec sed -i s'@magedu@magestudy@' {} \;
```

### `grep,sed,awk` 的运用

```bash
# 统计域名出现次数 "http://hi.baidu.com/browse/"
# 与统计 IP 访问次数基本一致
awk -F '/' '{num[$3]++}END{for (name in num){print  name,num[name]}}' file

# 打印奇数/偶数行
# n 表示读取模式空间的下一行; N 表示追加下一行到模式空间
sed -n 'p;n' file # 奇数行
sed -n 'n;p' file # 偶数行
# 奇数行和偶数行合并
sed 'N;s/\n/ /g' file

# 奇数行与偶数行交换
sed -r 'N;s@(.*)\n(.*)@\2\n\1@g' file

# 文件中两列数据,分别为 ip,status_code.统计状态码为 200 中,出现次数最多的 IP
awk '/200$/{ip_num[$1]++}END{for(ip in ip_num){print ip,ip_num[ip]}}' ip_code | awk '/NR==1/{print $1,"出现次数最多,为",$2}' | sort -nrk 2 | awk '{if (FNR==1){print $1,"出现次数最多,为",$2}}'
```

### 为 `history` 添加时间戳,如何防止个人 history 操作泄露

设置 `export HISTTIMEFORMAT='%F %T' ` 环境变量后,以后记录的命令历史操作就会添加时间

- 可以将 `history` 记录的个人操作清空,使用 `history -c` 清空当前命令历史,并使用 `history -w` 将已经清空的命令历史写入到命令历史文件 `~/.bash_history` 中(或直接清空该文件),下次登录边不再有命令历史
- 可以设置 `export HISTCONTROL=ignorespace` 环境变量,使得命令历史忽略记录以空格开始的命令.在执行敏感命令时,以空格开始
- 可以设置 `export HISTIGNORE=*` 环境变量,使命令历史忽略记录所有命令.使用 `export HISTIGNORE=` 恢复记录.

## 系统相关

### 进程的有效用户与实际用户

- Linux系统中某个可执行文件属于 root 并且有 setid,当一个普通用户 mike运行这个程序时,产生的进程的有效用户和实际用户分别是 root 和 mike

### `fork()` 创建进程

下面程序共有多少进程

```c
/*
* 在父进程中,fork 返回新创建子进程的进程 ID
* 在子进程中,fork 返回 0
* 如果出现错误,fork 返回一个负值
*/
#include <unistd.h>
#include <stdio.h>

int main()
{
    fork() || fork();
    sleep(100);
    retrun 0;
}
/*
* 共创建 3 个
* main() -> fork() -> fork()
*/

int main()
{
    fork() && fork();
    sleep(100);
    retrun 0;
}
/*
* 共创建 3 个
* main() -> fork()
*        -> fork()
*/
int main()
{
    fork() && fork() || fork();
    sleep(100);
    return 0;
}
/*
* 共创建 5 个
* main() -> fork() -> fork()
*        -> fork() -> fork()
*/

int main()
{
    fork(); // 新创建 1 个
    fork() && fork() || fork(); // 新创建 (1+1)*2+(1+1)*2 = 8 个
    fork(); // 新创建 1+8+1=10 个
    sleep(100);
    return 0;
}
// 共创建 20 个进程
```

## `ln -s` 与 `mv` 对某文件操作时,对 inode 和 block 有什么影响

```bash
# 理解 blocks 为磁盘存储块, inode 为指向该存储块的地址
ln -s afile bfile
# 原始文件不变
# 新文件 inode 与原始文件不同,且 block 为 0
# 原因: 创建符号链接,系统为符号链接分配 inode,符号链接不存储数据,只是作为引用,故 block 为 0

mv afile bfile
# 原始文件不存在
# 新文件与原始文件 inode 与 block 相同
# 原因: 修改文件名后,系统只是将文件名做改变,inode 与存储 block 不变

ln afile bfile
# 原始文件不变
# 新文件 inode 和 block 与原始文件相同
# 原因: 创建硬链接,相当于为原始文件指定一个别名,inode 与存储 block 不变,这两个文件名指向系统底层存储是一样的

cp afile bfile
# 原始文件不变
# 新文件 block 与 原始文件相同,inode 不同
# 原因: 拷贝文件,相当于对系统底层存储做拷贝,系统为新文件重新分配 inode,文件内容一直,block不变
```

### `free` 命令查看内存时,buffer 和 cache 各表示什么含义?如何清理 cache 缓存

这二者是为了提高IO性能的.

- buffer是即将要被写入磁盘的,buffer 能够使分散的写操作集中进行,减少磁盘碎片和硬盘的反复寻道,从而提高系统性能;
- cache是被从磁盘中读出来的.cached 是把读取过的数据保存起来,重新读取时若命中就不要去读硬盘,若没有命中就读硬盘

[清理缓存](https://blog.csdn.net/qq_29663071/article/details/81178289)

```bash
sync
echo 1 > /proc/sys/vm/drop_caches  # 释放页面缓存
echo 2 > /proc/sys/vm/drop_caches  # 释放索引,inodes 节点缓存,可能会降低磁盘索引效率
echo 3 > /proc/sys/vm/drop_caches  # 释放页面缓存,索引,inode 缓存
```

### top 页面含义

```text
当前时间及运行时长, 登录用户数, 平均负载
任务数: 总数,运行中,休眠中,停止,僵尸进程
CPU占比: 用户空间, 内核空间,, 空闲进程,等待进程
内存: 总物理内存,使用,剩余,buffer 缓冲
Swap: 总,使用,剩余,cache 缓存

进程ID 用户 优先级 nice值 虚拟内存 物理内存 共享内存 进程状态 CPU 内存 运行时长 命令
```

## `kill` 信号

```text
1 HUP 终端断线
2 INT 中断,同 Ctrl + C
3 QUIT 退出
9 KILL 强制终止
15 TERM 优雅的终止
18 CONT 继续(bg/fg命令)
19 STOP 暂停,同 Ctrl + Z
```

### `tcpdump` 抓包

```text
-i <DEV>: 抓取指定网络接口的数据包
-c <COUNT>: 抓取多少个数据包
-w <FILE>: 抓取数据包保存在文件中,而不是直接输出

expression: tcpdump 的表达式

type: 指定抓取类型.host,net,port,portrange,默认host
dir: 指定抓取传输方向.src,dst,src and dst,
proto: 指定抓取数据包协议.ip,tcp,udp

tcpdump [src | dst]host <HOST>: 指定主机
tcpdump host <HOST1> and <HOST2>: 抓取 HOST1 和 HOST2之间的数据包
tcpdump [tcp | udp] port <PORT>: 抓取执行端口数据包
tcpdump host <HOST> and port <PORT>: 抓取主机和端口数据包
tcpdump net <NET>: 抓取网络数据包
```

### 日志出现 `kernel:nf_conntrack:tablefull,dropping packet` 错误

此错误表示连接跟踪表已满,开始丢包,可能的原因是由于频繁的连接/关闭,或者网络的一些 TCP 的连接导致的

解决方法:

- 加大跟踪表的大小 `net.netfilter.nf_conntrack_max`
- 禁用一些不必跟踪的连接状态
- 禁用 `ip_vs`,`nf_conntrack` 模块

### sar 命令

统计,报告,保存系统运行状态信息

```text
sar [m [n]]: 每隔 m 秒报告一次,共报告 n 次,默认是 CPU 状态信息
-b 报告 IO 和传输速率
-f [filename] 从历史文件中报告数据
-n [DEV,IP,TCP,UDP] 报告网络状态信息
-o [filename] 将统计结果保存到二进制文件中
-P [0..ALL] 查看 CPU 相关信息
-r 报告内存利用率相关信息
-S 报告 Swarp 利用率相关信息
-u 报告 CPU 利用率状态信息
-v 报告 inode 相关信息
```

### 文件描述符

文件描述符是一个非负的索引值,指向内核中的"文件记录表".

- 当打开一个现存文件或创建一个新文件时,内核就向进程返回一个文件描述符
- 当需要读写文件时,文件描述符可作为参数传递给相应的函数
- Linux 下所有对设备和文件的操作都使用文件描述符来进行

常见文件描述符如下

- 0: 表示标准输入,对应宏为: `STDIN_FILENO`,函数 scanf() 使用的是标准输入
- 1: 表示标准输出,对应宏为: `STDOUT_FILENO`,函数 printf() 使用的是标准输出
- 2: 表示标准出错处理,对应的宏为: `STDERR_NO`
