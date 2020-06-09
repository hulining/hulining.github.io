---
title: Python 多线程/多进程与 GIL
date: 2020/06/09
tags:
  - Python
  - GIL
  - 面试
categories:
  - Python
abbrlink: 
description: 简述 Python GIL
---

## 全局解释器锁 GIL

CPython 解释器本身不是线程安全的,因此有全局解释器锁(GIL),用于保证同一时刻只允许使用一个线程执行 Python 字节码.

> 这是 CPython 解释器的局限,与 Python 语言本身无关. Jython 和 IronPython 则没有这种限制.

在 Python 多线程下,每个线程执行之前都需要获取 GIL 后才能执行线程代码中的代码,直到遇到 IO 阻塞或达到线程的最长执行时间后释放 GIL,等待下一次调度.而每次释放 GIL 锁,多个线程会进行锁竞争,切换线程,会造成资源损耗.这就是为什么即便在多核 CPU ,Python 的多线程效率可能并不高.

对于 CPU 密集型代码来说,多线程之间存在锁竞争,切换线程,会造成不必要的资源损耗,所以 **Python 的多线程对 CPU 密集型代码并不友好**.因此对于多核场景,推荐使用多进程,每个进程中的单个线程有独立的 GIL,互不干扰,这样就可以真正意义上的并行执行.

对于 IO 密集型代码来说,多线程能够有效有效提升效率,避免 IO 操作过程中的等待时间,所以 **Python 的多线程对 IO 密集型代码比较友好**.

## Python 中有 GIL,为什么还需要锁或线程同步?

GIL 保护 Python 解释器.这意味着:

- 您不必担心 Python 解释器由于多线程而出错
- 在同一时刻,只允许一个线程获取到 GIL,并执行,而一段时间内,多个线程是顺序执行的.

但是 GIL 并不会保证您的代码原子执行.见如下示例:

```python
import threading
import time

total = 0
lock = threading.Lock()

def increment_n_times(n):
    global total
    for i in range(n):
        total += 1

def safe_increment_n_times(n):
    global total
    for i in range(n):
        lock.acquire()
        total += 1
        lock.release()

def increment_in_x_threads(x, func, n):
    threads = [threading.Thread(target=func, args=(n,)) for i in range(x)]
    global total
    total = 0
    begin = time.time()
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    print('finished in {}s.\ntotal: {}\nexpected: {}\ndifference: {} ({} %)'
           .format(time.time()-begin, total, n*x, n*x-total, 100-total/n/x*100))

print('unsafe:')
increment_in_x_threads(70, increment_n_times, 100000)

print('\nwith locks:')
increment_in_x_threads(70, safe_increment_n_times, 100000)
```

输出如下

```text
unsafe:
finished in 0.9840562343597412s.
total: 4654584
expected: 7000000
difference: 2345416 (33.505942857142855 %)

with locks:
finished in 20.564176082611084s.
total: 7000000
expected: 7000000
difference: 0 (0.0 %)
```

对于对线程,如果没有锁,则会出现错误(33% 的加 1 过程失败).而带锁的计算结果是准确地,但速度要慢 20 倍.

为了解释上述过程没有加锁情况下,出现错误的原因,可以通过 `dis` 库函数查看 `x + 1` 的字节码(bytecode)执行过程.

```python
>>> import dis
>>> dis.dis(lambda x: x+1)
  1           0 LOAD_FAST                0 (x)
              2 LOAD_CONST               1 (1)
              4 BINARY_ADD
              6 RETURN_VALUE
```

这个操作需要多个 bytecodes 操作,在执行这个操作的多条 bytecodes 过程中可能就进行线程切换了,这样就出现了数据竞争的情况.

---

参考:

- [Python有GIL为什么还需要线程同步？](https://www.zhihu.com/question/23030421)
- [Why do we need locks for threads, if we have GIL?](https://stackoverflow.com/questions/40072873/why-do-we-need-locks-for-threads-if-we-have-gil/40072999)
- [why-does-python-provide-locking-mechanisms-if-its-subject-to-a-gil](https://stackoverflow.com/questions/26873512/why-does-python-provide-locking-mechanisms-if-its-subject-to-a-gil)
