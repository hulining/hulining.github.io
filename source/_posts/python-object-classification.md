---
title: Python 对象分类
date: 2020/06/17
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 本文主要整理 Python 中一些特殊的对象,如 可迭代对象,可调用对象等.以作备忘.
---

## 可调用对象

可调用对象,顾名思意,就是可以通过调用运算符 `()` 调用的对象.可以使用内置的 `callable()` 判断对象能否调用.

Python 模型文档中列出了 7 种可调用对象

- 用户使用 `def` 语句或 `lambda` 表达式定义的函数
- 内置函数: 使用 C (CPython)语言实现的函数,如 `len`
- 内置方法: 使用 C 语言实现的方法,如 `dict.get`
- 方法: 在类的定义体中定义的函数
- 类: 调用类时会运行类的 `__new__` 方法创建一个实例,然后运行 `__init__` 方法进行初始化.最后将实例返回给调用方.
- 类的的实例: 如果类定义了 `__call__` 方法,那么它的实例可以使用 `()` 调用.如

```python
class A(object):
    def __init__(self, name):
        self.name = name
    def __call__(self):
        return self.name

>>> a = A('tom')
>>> a
<__main__.A object at 0x000002647592C240>
>>> callable(a)
True
>>> a()
'tom'
```

- 生成器函数: 带有 `yield` 或 `yield from` 表达式的函数或方法.调用生成器函数,返回生成器对象.如:

```python
def gen():
    yield 'AB'
>>> callable(gen)
True
>>> gen()
<generator object gen at 0x000002647590A9E8>
```

## 可迭代对象

- 如果对象实现了能返回迭代器的 `__iter__` 方法,那么对象就是可迭代的
- 实现了 `__getitem__` 方法,而且其参数是从零开始的索引,这种对象也是可迭代对象.如序列

对可迭代对象调用 `iter()` 函数,会返回该对象的迭代器

```python
>>> iter('aaa')
<str_iterator object at 0x000002647592CB00>
>>> iter([1, 2, 3])
<list_iterator object at 0x000002647592CB38>
>>> iter((1, 2, 3))
<tuple_iterator object at 0x000002647592CBA8>
>>> iter({1, 2, 3})
<set_iterator object at 0x0000026475928F30>
>>> iter({'a' : 'b'})
<dict_keyiterator object at 0x00000264756B9868>

class A(object):
    def __init__(self, data):
        self._data = data
    def __iter__(self):
        return iter(self._data)

>>> a = A('string')
>>> iter(a)
<str_iterator object at 0x000002647592CB38>
```

## 可散列对象

在 [Python 词汇表](https://docs.python.org/3/glossary.html#term-hashable)中,关于可散列类型的定义有这样一段话:

> 如果一个对象是可散列的,那么在这个对象的生命周期中,它的散列值是不变的,而且这个对象需要实现 `__hash__()` 方法.

Python 中原子不可变数据类型(`str`,`bytes` 和数值类型(如 `int,float`))都是可散列类型,`frozenset` 也是可散列的,因为根据其定义,`frozenset` 里只能容纳可散列类型.而对于元组,当一个元组包含的所有元素都是可散列类型的情况下,它才是可散列的.

见如下示例

```python
>>> tt = (1, 2, (30, 40))
>>> hash(tt)
8027212646858338501
>>> tl = (1, 2, [30, 40])
>>> hash(tl)
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
>>> tf = (1, 2, frozenset([30, 40]))
>>> hash(tf)
-4118419923444501110

class A:
    def __init__(self, data):
        self.data = data
    def __hash__(self):
        return hash(id(self)) # 这里的数据必须是可散列对象

>>> a = A('string')
>>> hash(a)
-2622071085978536803
```

## 可变对象与不可变对象

详见 {% post_link python-mutable-and-immutable-object %}.
