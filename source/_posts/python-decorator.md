---
title: Python 装饰器
date: 2020/06/17
tags:
  - Python
  - 装饰器
  - 面试
categories:
  - Python
abbrlink: 
description: 高阶函数 + 函数嵌套 = 装饰器.本文主要介绍 Python 装饰器相关主要内容
---

在理解学习装饰器之前,首先要将 Python 中函数视作对象.如下我们在定义一个函数后,确定函数对象本身是 `function` 类的实例,并通过别的名称使用函数,将函数作为参数传递.

```python
def func(n):
    return 1 if n < 2 else n * func(n-1)

>>> func(5)
120
>>> type(func) # 查看函数的类型为 function,我们定义的函数即为 function 的一个实例
<class 'function'>

>>> fact = func # 通过别的名称使用函数
>>> fact
<function func at 0x000001C68E521E18>
>>> fact(5)
120
>>> map(func, range(11)) # 函数可以作为参数传递
<map object at 0x...>
>>> list(map(fact, range(11)))
[1, 1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
```

## 高阶函数

> 接受函数作为参数,或将函数作为结果返回的函数是高阶函数.

如上述示例中的 `map` 函数,它以我们定义的 `func` 函数作为参数传递,实现高阶函数的效果.另外,内置函数 `sorted` 函数也是.可选的 `key` 参数用于提供一个函数,此函数会应用到各个元素上,并以函数的返回结果进行排序.

```python
>>> fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
>>> sorted(fruits, key=len)
['fig', 'apple', 'cherry', 'banana', 'raspberry', 'strawberry']
```

最为人熟知的高阶函数有 `map`,`filter`, `sorted`,`reduce`.其中 `map`,`filter` 可以被列表推导式或生成器推导式所替代.`reduce` 函数从 Python 3.0 开始就不会内置函数了,需要从 `functools` 包导入(`from functools import reduce`).

## 装饰器原理

**装饰器函数其实是这样一个接口约束,它必须接受一个可调用对象作为参数,然后返回一个可调用对象.使用装饰器的好处就是在不用更改原函数的代码前提下给函数增加新的功能**.

> 可调用对象详见 {% post_link python-object-classification %} 中可调用对象小节.

假设有如下装饰器:

```python
# 此装饰器可能不能实现"面向切面编程"的效果,此处仅作为演示示例
def deco(func):
    def inner():
        print('running inner()')
    return inner

@decorate
def target():
    print('running target()')
```

上述代码的效果与下述写法一样:

```python
def deco(func):
    def inner():
        print('running inner()')
    return inner

def target():
    print('running target()')

target = decorate(target)
# 可以看到调用 target 实际上调用的不一定是原来的那个 target 函数,而是 decorate(target) 返回的可调用对象
```

严格的来说,装饰器只是语法糖.装饰器可以像常规可调用对象那样调用,其参数是另一个函数.

- 装饰器能将被装饰的函数替换成其它函数
- 装饰器在加载模块时立即执行

### Python 何时执行装饰器

装饰器在被装饰的函数定义之后立即运行.这通常发生在导入模块时.如下所示:

```python
# registration.py
registry = []

def register(func):
    # 定义一个装饰器,将被装饰的函数添加到 registry 中,然后返回该函数
    print('running register(%s)' % func)
    registry.append(func)
    return func

@register
def f1():
    print('running f1()')

# 上述代码 @register 可替换为
# f1 = register(f1)


@register
def f2():
    print('running f2()')

# 上述代码 @register 可替换为
# f1 = register(f3)

def f3():
    print('running f3()')
def main():
    print('running main()')
    print('registry ->', registry)
    f1()
    f2()
    f3()

if __name__=='__main__':
    main()
```

直接执行,结果如下:  

```text
$ python3 registration.py
running register(<function f1 at 0x100631bf8>)
running register(<function f2 at 0x100631c80>)
running main()
registry -> [<function f1 at 0x100631bf8>, <function f2 at 0x100631c80>]
running f1()
running f2()
running f3()
```

若作为模块导入,输出如下

```python
>>> import registration
running register(<function f1 at 0x10063b1e0>)
running register(<function f2 at 0x10063b268>)
>>> registration.registry  # 查看 registry 的值
[<function f1 at 0x10063b1e0>, <function f2 at 0x10063b268>]
```

上述情况应该不难理解.装饰器若在导入的模块中应用,则相当于在导入的模块中调用了两次装饰器的定义函数(上述示例中为 `register`),并将函数的返回值赋值给了 `f1` 与 `f2`.具体可参见注释部分.

考虑到真实代码中的常用方式,上述代码有两个不同的地方:

- 上述代码中装饰器函数与被装饰函数在同一模块中定义,而真实情况中,装饰器通常在一个模块中定义,然后应用到其它模块中的函数上.此时就不是导入时运行了,而可以理解为应用时运行(python 解释器解释到被装饰函数或类时,装饰器运行)
- `register` 装饰器返回的函数与通过参数传入的相同.但是,实际上,大多数装饰器会在内部定义一个函数,然后将其返回.

## 装饰器实现

### 计时器实现

如下示例实现了一个装饰器,用于计算被装饰函数的运行时间

```python
# clockdeco_demo.py
import time

def clock(func):
    def clocked():
        start = time.time()
        result = func()
        name = func.__name__
        end = time.time()
        print('%s function spent %s' % (name, (end-start)))
        return result
    return clocked
```

使用如上定义的装饰器:

```python
from clockdeco_demo import clock

@clock
def target():
    time.sleep(2)

if __name__ == '__main__':
    target()

# 输出如下
# target function spent 2.0003678798675537
```

我们来分析一下如上装饰器的使用过程使用.可以转换为如下代码

```python
from clockdeco_demo import clock

def target():
    time.sleep(2)

target = clock(target)

if __name__ == '__main__':
    target()
```

后续分析如下

```python
# @clock 可以转换为如下代码,其中,`clock(target)`返回 `clocked` 函数,相当于对 clocked 函数添加了别名
target = clock(target) -> clocked

# 调用后,实际上是执行了 clocked 函数.
# 而 clocked 函数又通过闭包引用了在上一步骤中传入的参数 `func=target`,因此返回的结果仍然是 `target` 函数的返回结果
# 并在此过程中计算了函数的执行时间,并打印输出
target() = clocked()

# 因此如果我们要装饰带参数的函数时,只需要在 clocked 函数定义时添加相关参数即可.
```

修改过的装饰器函数如下:

```python
import time

def clock(func):
    def clocked(*args, **kwargs): # 在函数定义时添加了变参,应对装饰函数带有参数的情况
        start = time.time()
        result = func(*args, **kwargs)
        name = func.__name__
        end = time.time()
        print('%s function spent %s' % (name, (end-start)))
        return result
    return clocked
```

### 带参数的装饰器

假设有这样一种场景,我想通过装饰器打印日志输出,并可以通过装饰器中参数来调整日志的输出级别.如 `@logging(level='INFO')`,`@logging(level='DEBUG')`,这要如何实现呢?

OK,我们先来看如下带参数的装饰器(与前一小节中差不多):

```python
# 定义装饰器
def logging(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        name = func.__name__
        print('%s function loggging %s %s' % (name,*args, *kwargs))
        return result
    return wrapper

# 正常使用
@logging
def app(name, age):
    user={
        'name':name,
        'age': age,
    }
    return user

# 错误使用,会导致编译不通过
# @logging(level='INFO')
# def err_app(name, age):
    # user={
        # 'name':name,
        # 'age': age,
    # }
    # return user


if __name__ == '__main__':
    print(app('tom',20))

# 输出如下
# app function loggging tom 20
# {'name': 'tom', 'age': 20}
```

那么,我们如何在装饰器上使用添加函数呢?

在如上示例中,`@logging` 等价于 `app = logging(app) -> wrapper`.如果装饰器带上函数,则可以理解为 `@logging(level='INFO')` 等价于 `app = logging(level='INFO')(app)-> wrapper(app)`,就相当于继续调用内层函数 `wrapper`,并将 `app` 作为参数传入.很显然,上述代码中,内层函数 `wrapper` 返回结果为 `app` 函数的返回结果 `user`,是不可调用对象.因此,我们需要使内层函数 `wrapper` 继续返回一个可调用对象 `inner_wrapper`,而所有操作(打印日志,执行原始 `app` 函数)均在 `inner_wrapper` 中执行.代码如下:

```python
def logging(level):
    def wrapper(func):
        def inner_wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            name = func.__name__
            print('%s: %s function loggging %s %s' % (level, name,*args, *kwargs))
            return result
        return inner_wrapper
    return wrapper
```

尝试使用一下:

```python
@logging('INFO')
def info_app(name, age):
    user={
        'name':name,
        'age': age,
    }
    return user

@logging('ERROR')
def err_app(name, age):
    user={
        'name':name,
        'age': age,
    }
    return user

if __name__ == '__main__':
    print(app('tom',20))
    print(err_app('jerry',19))

# 输出如下
INFO: info_app function loggging tom 20
{'name': 'tom', 'age': 20}
ERROR: err_app function loggging jerry 19
{'name': 'jerry', 'age': 19}
```

此时再来分析一下 `@logging(level='INFO')`,它可以做如下转换:

```python
app = logging(level='INFO')(app) -> wrapper(app) -> inner_wrapper

# 调用 app,其实是调用 inner_wrapper,返回值为 result
app() = inner_wrapper() -> result
```

上述代码的转换,其实是在不带参数的装饰器的基础上又在最外层嵌套了一层函数.

### 装饰器类

装饰器类的实现其实与装饰器函数的实现大同小异.只要符合**接受一个可调用对象作为参数,并返回一个可调用对象**的类就是一个装饰器类.

与装饰器函数类似,装饰器类的实现,需要我们:

- 将被装饰的对象必须作为类的初始化参数传入装饰器类.
- 在装饰器类的 `__call__` 方法中定义装饰器进行的操作,修饰并调用被装饰的可调用对象,并返回被装饰对象的调用结果

见如下实例

```python
class logging(object):
    def __init__(self, func):
        '''
        装饰器类的初始化函数,必须传入一个参数
        :param func: 被装饰的对象
        '''
        self.func = func

    def __call__(self, *args, **kwargs):
        '''
        在此方法中定义装饰器的内容,并返回被装饰对象的调用结果
        :param args, kwargs: 为被装饰对象的参数
        '''
        print("[DEBUG]: enter function {func}()".format(func=self.func.__name__))
        return self.func(*args, **kwargs)

@logging
def say(something):
    print("say {}!".format(something))

if __name__ == '__main__':
    say('decorator class')
```

在上述示例中,只要知道 `@logging` 等价于 `say = logging(say)` 就不难理解,这一步骤实际上做的事情是以 `say` 函数作为初始化参数创建了 `logging` 类的实例对象.在调用 `say('decorator class')` 时,实际上是对 `logging` 类的实例进行了调用.从而调用了实例的 `__call__` 方法.

#### 带参数的装饰器类

参参数的装饰器类可以看作是装饰器类与带参数的装饰器函数的结合体.

与装饰器函数类似,我们需要

- 将装饰器类的参数传入装饰器类的初始化函数中
- 在装饰器类的 `__call__` 方法中定义装饰器进行的操作,修饰并调用被装饰的可调用对象,并返回一个可调用对象

```python
class logging(object):
    def __init__(self, level='INFO'):
        self.level = level

    def __call__(self, func):
        '''
        在此方法中定义装饰器的内容,并返回可调用对象
        :param func: 被装饰对象
        :return: 返回可调用对象
        '''
        def wrapper(*args, **kwargs):
            print("[{level}]: enter function {func}()".format(
                level=self.level,
                func=func.__name__))
            return func(*args, **kwargs)
        return wrapper

@logging(level='INFO')
def say(something):
    print("return {}!".format(something))
    return something

if __name__ == '__main__':
    print(say('tom'))
```

### 可通过方法自定义属性的装饰器

参考 `Python Cookbook 中文第 3 版 - "9.5 可自定义属性的装饰器"`章节.

- Q: 你想写一个装饰器来包装一个函数,并且在运行时允许用户提供参数控制装饰器行为
- A: 引入一个访问函数,使用 nonlocal 来修改内部变量.然后这个访问函数被作为一个属性赋值给包装函数

```python
from functools import wraps, partial
import logging

def attach_wrapper(obj, func=None):
    if func is None:
        return partial(attach_wrapper, obj)
        # 在这里很巧妙的使用了 partial 函数,将 obj 默认传入 attach_wrapper 中,再次调用时,只需要传入一个 func 参数即可
    setattr(obj, func.__name__, func) # 将传入的func参数设置为obj的属性,并在后续将func返回
    return func

def logged(level, name=None, message=None):
    def decorate(func):
        logname = name if name else func.__module__
        log = logging.getLogger(logname)
        logmsg = message if message else func.__name__

        @wraps(func)
        def wrapper(*args, **kwargs):
            log.log(level, logmsg)
            return func(*args, **kwargs)

        @attach_wrapper(wrapper)
        def set_message(newmsg):
            nonlocal logmsg
            logmsg = newmsg

        # set_message = attach_wrapper(wrapper)(set_message)
        # set_message = partial(attach_wrapper, wrapper)(set_message)
        # 这一步骤中的 partial(func,arg1)(arg2) 函数作用为将 arg1 使用作为 func 的参数传递,返回可调用对象,并在下次传参调用时,只需传入一个参数即可.如
        # 假设 `add = lambda x, y: x+y` 则有 `a = partial(add,1)(2)` 等价于 `add(1,2)
        # partial 函数的作用详见官方文档 https://docs.python.org/3/library/functools.html#functools.partial 或博客 https://blog.csdn.net/cassiepython/article/details/76653897
        # set_message = attach_wrapper(wrapper, set_message)
        #               setattr(wrapper, 'set_message', set_message) # 为 wrapper 添加 set_message 属性,为 set_message 函数,以备在 main 方法中可以调用 set_message()
        #               return set_message # 并返回原始函数

        return wrapper
    return decorate

# Example use
@logged(logging.WARNING)
def add(x, y):
    return x + y

# add = logged(logging.WARNING)(add) -> decorate(add) -> wrapper

if __name__ == '__main__':
    add.set_message('Add called')
    add(10, 20)
    # wrapper(10,20) -> 打印日志信息,并返回原始函数的执行结果
```

## 装饰器注意事项

### 装饰器内代码的执行顺序

```python
def logging(level):
    print('outer function begin')
    def wrapper(func):
        print('wrapper function begin')
        def inner_wrapper(*args, **kwargs):
            print('inner function begin')
            print("[{level}]: enter function {func}()".format(
                level=level,
                func=func.__name__))
            print('inner function end')
            return func(*args, **kwargs)
        print('wrapper function end')
        return inner_wrapper
    print('outer function end')
    return wrapper

@logging(level='debug')
def hello():
   print('hello')

# 输出结果
outer function begin
outer function end
wrapper function begin
wrapper function end
inner function begin
[debug]: enter function hello()
inner function end
hello
```

### 错误的属性

任何被装饰器装饰过的对象,其实已经变成装饰器函数返回的可调用对象.导致被装饰对象的所有内置属性,都会变成装饰器函数返回的可调用对象的内置属性.

```python
from datetime import datetime
def logging(func):
    def wrapper(*args, **kwargs):
        """print log before a function."""
        print("[DEBUG] {}: enter {}()".format(datetime.now(), func.__name__))
        return func(*args, **kwargs)
    return wrapper

@logging # 等同于 say=logging(say),导致 say 变量的所有内置属性,都会变成 wrapper 函数的内置属性
def say(something):
    """say something"""
    print("say {}!".format(something))

print(say.__name__)  # wrapper
```

解决: 使用标准库里的 `functools.wraps` 装饰器对待返回的可调用对象进行装饰,可以基本解决这个问题

```python
from functools import wraps

def logging(func):
     # 使用内置库里的 wraps 对内置函数进行装饰
    @wraps(func)
    def wrapper(*args, **kwargs):
        """print log before a function."""
        print "[DEBUG] {}: enter {}()".format(datetime.now(), func.__name__)
        return func(*args, **kwargs)
    return wrapper

@logging
def say(something):
    """say something"""
    print("say {}!".format(something))

print(say.__name__)  # say
print(say.__doc__) # say something
```

## 装饰器第三方库

### [decorator](https://decorator.readthedocs.io/en/latest/)

```python
from decorator import decorator

@decorator
def logging(func, *args, **kwargs):
    print "[DEBUG] {}: enter {}()".format(datetime.now(), func.__name__)
    return func(*args, **kwargs)

@logging
def say(something):
    pass
```

### [wrapt](https://wrapt.readthedocs.io/en/latest/)

```python
import wrapt

# without argument in decorator
@wrapt.decorator
def logging(wrapped, instance, args, kwargs):  # instance is must
    print "[DEBUG]: enter {}()".format(wrapped.__name__)
    return wrapped(*args, **kwargs)

@logging
def say(something):
    pass
```
