---
title: Python 单双下划线及魔术方法
date: 2020/06/09
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 本文主要介绍 Python 单双下划线的区别及常见魔术方法
---

## Python中单下划线和双下划线

首先看如下代码:

```python
>>> class MyClass():
...     def __init__(self):
...         self.__superprivate = "Hello"
...         self._semiprivate = ", world!"
...
>>> mc = MyClass()
>>> print(mc.__superprivate)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: myClass instance has no attribute '__superprivate'
>>> print(mc._semiprivate)
, world!
>>> print(mc.__dict__)
{'_MyClass__superprivate': 'Hello', '_semiprivate': ', world!'}
```

- `_foo`: 单下划线是一种约定,用来指定变量私有(逻辑上的,其实仍可以访问).不能用 `from module import *` 导入,其他方面和公有一样访问.
- `__foo`: 解析器用 `_<classname>__foo` 来代替这个名字,以区别和其他类相同的命名,必须通过 `对象名._类名__xxx` 这样的方式进行访问.
- `__foo__`: Python 内部魔术方法,用来区别其他用户自定义的命名,以防冲突.例如 `__init__()`,`__del__()`,`__call__()` 等.

## Python 常用魔术方法或属性

### `__abs__()`

如果类实现了该方法,可通过 `abs()` 内置函数直接调用.如

```python
class C(int):
    def __init__(self, x):
        self._x = x

    def __abs__(self):
        """ 通过 abs() 内置函数调用"""
        print("__abs__")
        return super().__abs__()

if __name__ == "__main__":
    c = C(-2)
    print(abs(c))

```

### `__eq__()`

如果类实现了该方法,可通过 `==` 判断两个对象是否相等.如

```python
class C():
    def __init__(self, x, y):
        self._x = x
        self._y = y

    def __eq__(self, other):
        print("__eq__")
        return self._x == other._x and self._y == other._y

if __name__ == '__main__':
    c = C("x", "y")
    d = C("x", "y")
    print(c == d)
```

### `__hash__()`

如果类实现了该方法,可通过 `hash()` 内置函数直接调用.如

```python
class C():
    def __init__(self, x, y):
        self._x = x
        self._y = y
    def __hash__(self):
        print("__hash__")
        return super().__hash__()

if __name__ == '__main__':
    c = C(1, 5)
    print(hash(c))

```

### `__iter__()`

> 如果对象实现了可以返回迭代器的 `__iter__` 方法,那么对象就是可迭代的.

如果对象实现了该方法,可通过 `iter()` 函数直接调用.更多信息参见 {% post_link python-iterator-and-generator %}.

### __mro__

此属性值为元组,子类在访问父类方法或属性时,会按照元组元素顺序寻找基类的方法或属性.如

```python
class A():
    def foo(self):
        print("a.foo")

class B(A):
    pass

class C(A):
    def foo(self):
        print("c.foo")

class D(B, C):
    pass

d = D()
d.foo()
print(D.__mro__)

# 输出为
# c.foo
# (<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>, <class 'object'>)
```

在方法解析期间寻找基类时要考虑的类的元组

### `__repr__()` 与 `__str__()`

如果类实现了该方法,可通过 `str()` 内置函数直接调用.如

```python
class C():
    def __init__(self, x, y):
        self._x = x
        self._y = y

    def __repr__(self):
        print("__repr__")
        return "%s,%s" % (self._x, self._y)

    def __str__(self):
        print("__str__")
        return super().__str__()

if __name__ == '__main__':
    c = C("x", "y")
    # print(c)  # 会默认调用 __str__()
    print(str(c))

# 输出如下,可以看出调用顺序为 __str__ -> __repr__
# __str__
# __repr__
# x,y
```
