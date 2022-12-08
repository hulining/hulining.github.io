---
title: Python 协程之 yield 和 yield from 表达式
date: 2020/06/15
tags:
  - Python
  - 协程
  - 面试
categories:
  - Python
abbrlink: 
description: 协程 coroutine,又称微线程.本文章主要记录 Python yield 与 yield from 表达式创建的协程的相关内容,以作备忘.
---

由于 GIL 的存在,Python 多线程性能甚至比单线程更糟.

协程是一种用户态的轻量级线程,它可以在两个函数或逻辑流中自由切换,但这一过程并没有函数调用.协程实际上是运行在单个线程中,因此没有线程上下文切换的开销.

协程拥有自己的寄存器上下文和栈,协程调度切换时,将寄存器上下文和栈保存到其他地方,在切回来的时候,恢复之前保存的寄存器上下文和栈.因此协程能够保留上一次调用时的状态,能够进入上一次切换时所处的逻辑流的位置.

从句法上看,协程与生成器类似,都是定义体中包含 `yield` 关键字的函数.可是,在协程中,

- `yield` 关键字通常出现在表达式的右边(如 `datum = yield`)
- 可以产出值,也可以不产出(如果 `yield` 关键字后面没有表达式,那么产出 `None`)
- 协程可以从调用方通过调用 `.send(datum)` 函数接收数据,而不是 `next()` 函数

不管数据流如何移动,可以把 `yield` 看作是一种流程控制工具.协程可以把控制器交给中心调度程序,从而激活其它协程.

## yield 表达式 - 由生成器进化为协程

协程的底层架构在 [PEP 342—Coroutines via Enhanced Generators](https://www.python.org/dev/peps/pep-0342/) 中定义,并在 Python 2.5 实现了.自此之后 `yield` 关键字可以在表达式中使用,且生成器 API 增加了 `.send(value)` 方法.**生成器的调用方可以使用 `.send(...)` 方法发送数据,发送数据会成为生成器函数中 `yield` 表达式的值.**因此,生成器可以作为协程使用.

```python
def simple_coroutine():  # 定义生成器函数(带有 yield 关键字)
    print('-> coroutine started')
    x = yield  # yield 在表达式中使用,此处协程只从调用方接收数据,产出值为 None
    print('-> coroutine received:', x)

>>> my_coro = simple_coroutine()  # 与创建生成器方式一样,调用生成器函数得到生成器对象
>>> my_coro
<generator object simple_coroutine at 0x000001E193DC8AF0>
>>> next(my_coro)  # 首先调用 next() 函数,启动生成器,并在 yield 关键字处暂停,等待接收数据
-> coroutine started
>>> my_coro.send(42)
# 这一步骤做了两件事
# - 向生成器发送数据 42 后,发送数据作为 yield 关键字表达式产出数据,并赋值给变量 x.# - 协程恢复,一直运行到下一个 yield 表达式或终止.
-> coroutine received: 42
Traceback (most recent call last):  # 此处控制权流动到定义体的末尾,生成器像往常一样抛出异常.
  File "<stdin>", line 1, in <module>
StopIteration
```

协程一定处于如下四个状态中的一个,当前状态可使用 `inspect.getgeneratorstate(...)` 函数确定.

- `GEN_CREATED`: 协程创建,等待开始执行
- `GEN_RUNNING`: 协程正在执行.(多线程应用才能看到这个状态)
- `GEN_SUSPENDED`: 在 `yield` 表达式处暂停
- `GEN_CLOSED`: 执行结束

`send()` 方法的参数会成为暂停的 `yield` 表达式的值,所以,仅当协程处于暂停状态时才能调用 `send()` 方法传入非空值.如果协程还没激活(即,状态是 'GEN_CREATED'),则需要调用 `next(my_coro)` 或 `my_coro.send(None)`,传入 None,激活协程.

> 如果创建协程对象后立即把 None 之外的值发给它,会出现下述错误 `TypeError: can't send non-None value to a just-started generator`.

```python
def simple_coro2(a):
    print('-> Started: a =', a)
    b = yield a
    print('-> Received: b =', b)
    c = yield a + b
    print('-> Received: c =', c)

>>> my_coro2 = simple_coro2(14)
>>> from inspect import getgeneratorstate
>>> getgeneratorstate(my_coro2) # 通过 getgeneratorstate 函数查看协程状态为 GEN_CREATED
'GEN_CREATED'
>>> next(my_coro2) # 启动协程,执行到第一个 yield 表达式,然后产出 a 的值,并且暂停,等待传入值.
-> Started: a = 14
14
>>> getgeneratorstate(my_coro2) # 当前协程状态为暂停
'GEN_SUSPENDED'
>>> my_coro2.send(28) # 将 28 发给暂停的协程,计算 yield 表达式,得到 28,并赋值给 b.并继续向下执行,直到下一个 yield 表达式,产出 a + b 的值,协程暂停.等待传入值.
-> Received: b = 28
42
>>> my_coro2.send(99) # 向当前协程传入 99,计算 yield 表达式,并赋值给变量 c,向下执行,直到协程终止,导致生成器对象抛出 StopIteration 异常
-> Received: c = 99
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
StopIteration
>>> getgeneratorstate(my_coro2) # 此时协程处于关闭状态
'GEN_CLOSED'
```

### 生产者消费者模型

传统的生产者-消费者模型是一个线程写消息,一个线程取消息,通过锁机制控制队列和等待,但一不小心就可能死锁.

以下是使用协程实现的生产者消费者模型.

```python
def consumer():
    r = ''
    while True:
        n = yield r
        if not n:
            return
        print('[CONSUMER] Consuming %s...' % n)
        r = '200 OK'

def produce(c):
    c.send(None)
    n = 0
    while n < 5:
        n = n + 1
        print('[PRODUCER] Producing %s...' % n)
        r = c.send(n)
        print('[PRODUCER] Consumer return: %s' % r)
    c.close()

c = consumer()
produce(c)
```

`consumer()` 函数是一个生成器函数,调用生成器函数得到生成器对象 `c`,传入 `produce()` 后:

1. 调用 `c.send(None)` 启动协程.流程控制跳转到 `consumer()` 函数中执行,直到遇到 `yield` 表达式.此时协程处于暂停状态,生成 `r`(为空字符串),并等待数据发送.
2. `produce()` 继续执行,并调用 `c.send(n)` 向协程发送数据
3. `consumer()` 函数通过 `yield` 接收到数据,赋值给变量 `n`,继续执行,打印消息 -> 赋值变量 r -> 进入下一次循环后生成 `r`(为 200 OK),直到遇到 `yield` 表达式
4. `produce()` 继续生产下一条数据,重复上述生产消费模型,直到跳出循环
5. `produce()` 调用 `c.close()` 关闭生成器/协程

### 预激协程的装饰器

调用 `my_coro.send(x)` 之前,记住一定要调用 `next(my_coro)` 或 `my_coro.send(None)`.为了简化协程的用法,有时会使用一个预激装饰器.示例如下

```python
from functools import wraps
from inspect import getgeneratorstate

def coroutine(func):
    @wraps(func)
    def primer(*args,**kwargs): # 把被装饰的生成器函数替换成这里的 primer 函数.调用 primer 函数时,返回预激后的生成器
        gen = func(*args,**kwargs) # 调用被装饰的函数,获取生成器对象
        next(gen) # 预激生成器
        return gen # 返回
    return primer

@coroutine
def simple_coro(a):
    print('-> Started: a =', a)
    b = yield a
    print('-> Received: b =', b)
    c = yield a + b
    print('-> Received: c =', c)

>>> my_coro = simple_coro(14) # 生成器/协程在函数调用时就被预激了
-> Started: a = 14
>>> getgeneratorstate(my_coro) # 此时协程状态为暂停状态
'GEN_SUSPENDED'
>>> my_coro.send(28) # 可以直接发送数据
-> Received: b = 28
42
# ...
```

### 终止协程及异常处理

协程中未处理的异常会向上冒泡,传给 `next()` 函数或 `send()` 方法的调用方(即触发协程的对象).未处理的异常会导致协程终止.如下:

```python
from inspect import getgeneratorstate
def simple_coro(a):
    print('-> Started: a =', a)
    b = yield a
    print('-> Received: b =', b)
    c = yield a + b
    print('-> Received: c =', c)

>>> my_coro = simple_coro(14)
>>> next(my_coro)
-> Started: a = 14
14
>>> my_coro.send('spam') # 发送数据后,在 a + b 过程中会抛出异常
-> Received: b = spam
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 5, in simple_coro
TypeError: unsupported operand type(s) for +: 'int' and 'str'
>>> my_coro.send(60)  # 未处理的异常导致协程终止,再次发送数据,抛出 StopIteration
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>>  getgeneratorstate(my_coro) # 查看此时协程状态为 GEN_CLOSED
'GEN_CLOSED'
```

从 Python 2.5 开始,客户端代码可以在生成器对象上调用如下两个方法,显示地将异常发送给协程.

- `generator.throw(exc_type[, exc_value[, traceback]])`

生成器在暂停的 `yield` 表达式处显式地抛出异常.如果生成器处理了抛出的异常,代码会向前执行到下一个 `yield` 表达式;否则,异常会向上冒泡,传到调用方的上下文中.

- `generator.close()`

使生成器在暂停的 `yield` 表达式处抛出 `GeneratorExit` 异常,并进行处理.如果生成器抛出 `StopIteration` 异常(通常指运行到结尾),调用方不会报错,从而关闭生成器.

如下是协程中处理异常的示例代码

```python
class DemoException(Exception):
    pass

def demo_exc_handling():
    print('-> coroutine started')
    while True:
        try:
            x = yield
        except DemoException: # 处理自定义异常
            print('*** DemoException handled. Continuing...')
        else: # 否则打印接收到的内容
            print('-> coroutine received: {!r}'.format(x))

>>> from inspect import getgeneratorstate
>>> exc_coro = demo_exc_handling()
>>> next(exc_coro)
-> coroutine started
>>> exc_coro.send(11)
-> coroutine received: 11
 >>> exc_coro.throw(DemoException) # 显示抛出 DemoException 异常
*** DemoException handled. Continuing...
>>> getgeneratorstate(exc_coro)  # 显示当前协程状态为 GEN_SUSPENDED,表示仍可以传入数据
'GEN_SUSPENDED'
>>> exc_coro.send(22) # 仍可以继续发送数据
-> coroutine received: 22
>>> exc_coro.throw(ZeroDivisionError) # 显示传入无法处理的异常后,协程终止
Traceback (most recent call last):
...
ZeroDivisionError
>>> getgeneratorstate(exc_coro)
'GEN_CLOSED'

# >>> exc_coro.close() # 或显示关闭后,查看状态为 GEN_CLOSED,此时不可以发送数据
# >>> getgeneratorstate(exc_coro)
# 'GEN_CLOSED'
```

### 协程返回值

先看如下示例,`averager` 协程返回的结果是一个 `namedtuple`,两个字段分别是项数(count)和平均值(average).

```python
from collections import namedtuple

Result = namedtuple('Result', 'count average')

def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break # 如果激活协程后,调用方发送 None.则跳出循环,返回 Result 对象
        total += term
        count += 1
        average = total/count
    return Result(count, average)
```

我们首先来看一下,向激活的协程发送 `None` 会返回什么

```python
>>> coro_avg = averager()
>>> next(coro_avg)
>>> coro_avg.send(10)
>>> coro_avg.send(30)
>>> coro_avg.send(6.5)
>>> coro_avg.send(None) # 发送 None
Traceback (most recent call last):
...
StopIteration: Result(count=3, average=15.5) # 可以看到 StopIteration 异常的值为我们想要返回的 Result 对象
```

因此我们可以捕获 `StopIteration` 异常,获取 averager 返回的值

```python
>>> coro_avg = averager()
>>> next(coro_avg)
>>> coro_avg.send(10)
>>> coro_avg.send(30)
>>> coro_avg.send(6.5)
>>> try:
...     coro_avg.send(None)
... except StopIteration as exc: # 捕获 StopIteration 异常,将其 value 赋值给 result
...     result = exc.value
...
>>> result
Result(count=3, average=15.5)
```

以上获取带有 `yield` 关键字表达式的协程的返回值虽然要绕个圈子,但这是 PEP 380 定义的方式.而 `yield from` 结构会在内部自动捕获 `StopIteration` 异常.这种处理方式与 for 循环处理 `StopIteration` 异常的方式一样: 循环机制以用户易于理解的方式处理异常.对 `yield from` 结构来说,解释器不仅会捕获 `StopIteration` 异常,还会把异常的 `value` 属性的值变成 `yield from` 表达式的值.

## yield from 表达式

首先要知道,`yield from <expr>` 表达式是全新的语言结构,其中 `<expr>` 为可迭代对象.

### yield from 表达式做了什么

- `yield from <expr>` 对 `<expr>` 可迭代对象所做的第一件事是调用 `iter(<expr>)`,从中获取迭代器.

```python
def gen():
    for c in 'AB':
        yield c
    for i in range(1, 3):
        yield i

# 可以转化为如下

def gen():
    # `yield from` 可用于简化 for 循环中的 `yield` 表达式
    yield from 'AB'
    yield from range(1, 3)
```

- `yield from <expr>` 表达式会捕获 `<expr>` 可迭代对象迭代结束后的 `StopIteration` 异常,并把异常的 `value` 属性的值变成 `yield from` 表达式的值

- 当 `<expr>` 是另一个生成器时,允许子生成器执行带有返回值的 `return` 语句,并且该返回值将成为 `yield from` 表达式 的值

### 原理

`yield from` 的主要功能是打开双向通道,把最外层的调用方与最内层的子生成器连接起来,这样二者可以直接发送和产出值,还可以直接传入异常.而不用在位于中间的协程中添加大量处理异常的代码.

`yield from` 表达式中包含 3 个概念:

- 委派生成器: 包含 `yield from <iterable>` 表达式的生成器函数
- 子生成器: 从 `yield from` 表达式中 `<iterable>` 部分获取的生成器
- 调用方: PEP 380 使用"调用方"这个术语指代调用委派生成器的客户端代码

![yield-from](/images/yield-from.png)

委派生成器在 `yield from` 表达式处暂停时,调用方可以直接把数据发给子生成器,子生成器再把产出的值发给调用方.子生成器返回结束后(调用方传入向子生成器传入 None 时),子生成器会抛出 `StopIteration` 异常,并把返回值附加到异常对象上,此时委派生成器会恢复,并捕获异常.

如下代码示例描述起来可能更直观一点.

```python
from collections import namedtuple

Result = namedtuple('Result', 'count average')

# 子生成器, 带有 yield 表达式的生成器函数
def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield # main 函数中的客户代码发送的各个值绑定到这里的 term 变量上
        if term is None: # 如果发送数据为 None,则跳出循环,返回 Result 对象
            break
        total += term
        count += 1
        average = total/count
    return Result(count, average) # 返回 Result 对象

# 委派生成器,带有 yield from 表达式的生成器函数
def grouper(results, key):
    while True:
        # 每次迭代时会新建一个 averager 实例,每个实例都是作为协程使用的生成器对象
        # grouper 发送的每个值都会经由 yield from 处理,通过管道传给 averager 实例.
        # grouper 会在 yield from 表达式处暂停，等待 averager 实例处理客户端发来的值.
        # averager 实例运行完毕后,返回的值绑定到 results[key] 上.
        results[key] = yield from averager()

# 客户端代码，即调用方
def main(data):
    results = {}
    for key, values in data.items():
        group = grouper(results, key) # 调用 grouper 函数,返回生成器对象.group 作为协程使用
        next(group) # 启动/激活协程
        for value in values:
            group.send(value) # 将 data.key 中数据传给 group 协程.传入的值最终到达 averager 函数中 `term = yield` 那一行.grouper 函数永远不知道传入的值是什么
        group.send(None) # 重要!
        # 数据发送完毕后,传入 None ,终止当前的 averager 实例,返回 Result 对象.
        # 让 grouper 继续运行,再创建一个 averager 实例,处理下一组值
    print(results) # 待全部 key 循环完后,打印最终结果

data = {
    'girls;kg': [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
    'girls;m': [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
    'boys;kg': [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
    'boys;m': [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
}

if __name__ == '__main__':
    main(data)
```

上述示例阐明了如下四点,参见 [PEP 380 Proposal](https://www.python.org/dev/peps/pep-0380/#proposal)小节

- 子生成器产生的任何值都将直接传递给调用方(如上示例中的 main 函数代码)
- 使用 `send()` 方法发给委派生成器的值都直接传给子生成器.如果发送的值是 None,那么会调用子生成器的 `__next__()` 方法.如果发送的值不是 None,那么会调用子生成器的 `send()` 方法.如果调用的方法抛出 `StopIteration` 异常,那么委派生成器恢复运行,而任何其他异常都会向上冒泡,传给委派生成器
- 生成器退出时,子生成器中的 `return expr` 表达式会触发 `StopIteration(expr)` 异常抛出
- `yield from` 表达式的值是子生成器终止时传给 `StopIteration` 异常的第一个参数

以下是 `RESULT = yield from EXPR` 简化的伪代码,没有处理委派生成器调用 `.throw(...)` 和 `.close()` 方法,且只处理了 `StopIteration` 异常

```python
# _i 子生成器
# _y 子生成器产出的值
# _r 最终返回结果,yield from 表达式的值
# _s 调用方发给委派生成器的值,这个值会转发给子生成器
# _e 异常对象,伪代码中始终是 StopIteration 实例
_i = iter(EXPR) # EXPR 是可迭代对象,因此调用 iter(EXPR) 获取迭代器 _i(这是子生成器)
try:
    _y = next(_i) # 预激子生成器,将产出结果保存在 _y 中
except StopIteration as _e:
    _r = _e.value # 如果抛出 StopIteration 异常,则获取异常对象的 value 属性,并赋值给 _r
else:
    while 1: # 运行循环,委派生成器会阻塞,只作为调用方和子生成器之间的通道
        _s = yield _y # 产出子生成器当前产出的元素,并等待调用方发送数据(代码示例中为 `group.send`),并赋值给变量 _s
        try:
            _y = _i.send(_s) # 将调用方发送的 _s 发送到子生成器中,尝试让子生成器向前执行
        except StopIteration as _e: # 如果子生成器抛出 StopIteration 异常,获取其 value 属性的值,赋值给 _r,然后退出循环,让委派生成器恢复运行
            _r = _e.value
            break
RESULT = _r # _r 是我们想要返回的结果,是整个 yield from 表达式的值
```

另外可以参考 [PEP 380 Formal Semantics](https://www.python.org/dev/peps/pep-0380/#formal-semantics) 中对 `RESULT = yield from EXP` 表达式给出的完整伪代码.

---

参考:

- [流畅的 Python](http://product.dangdang.com/25071121.html)
- [廖雪峰 Python 教程 - 异步IO - 协程](https://www.liaoxuefeng.com/wiki/1016959663602400/1017968846697824)
