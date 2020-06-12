---
title: Python 迭代器与生成器
date: 2020/06/08
tags:
  - Python
  - 面试题
categories:
  - Python
abbrlink: 
description: 简述 Python 可迭代对象,迭代器,生成器之间的关系,并以简单示例加深对迭代器与生成器的理解.
---

> 迭代协议要求对象的 `__iter__` 方法返回一个特殊的迭代器对象,这个迭代器对象实现了 `__next__` 方法并通过 `StopIteration` 异常标识迭代的完成.

## 可迭代对象 Iterable

```python
# collections.abc.Iterable
class Iterable(metaclass=ABCMeta):

    __slots__ = ()

    @abstractmethod
    def __iter__(self):
        while False:
            yield None

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Iterable:
            return _check_methods(C, "__iter__")
        return NotImplemented
```

> 如果对象实现了可以返回迭代器的 `__iter__` 方法,那么对象就是可迭代的.

对可迭代对象使用 `iter` 函数,返回的对象就是迭代器.解释器需要迭代对象 x 时,会自动调用 `iter(x)`

`iter` 函数做了如下事情:

1. 检查对象是否实现了 `__iter__` 方法,如果实现了,则调用.返回并获取一个迭代器
2. 如果没有实现 `__iter__` 方法,但是实现了 `__getitem__` 方法,Python 会创建一个迭代器,尝试按循序(从索引 0 开始)获取元素
3. 如果尝试失败,Python 抛出 TypeError 异常,通常会提示 "C object is not iterable"( C 对象不可迭代),其中 C 是目标对象所属的类

```python
class Node:

    def __init__(self, value):
        self._value = value
        self._children = []

    def __repr__(self):
        return 'Node({!r})'.format(self._value)

    def add_child(self, node):
        self._children.append(node)

    def __iter__(self):
        return iter(self._children)
```

## 迭代器 Iterator

```python
# collections.abc.Iterator
class Iterator(Iterable):

    __slots__ = ()

    @abstractmethod
    def __next__(self):
        'Return the next item from the iterator. When exhausted, raise StopIteration'
        raise StopIteration

    def __iter__(self):
        return self

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Iterator:
            return _check_methods(C, '__iter__', '__next__')
        return NotImplemented
```

迭代器继承了 `Iterable` 类,实现了如下方法:

- `__next__`: 返回序列中的下一个元素.如果没有元素了,那么抛出 `StopIteration` 异常
- `__iter__`: 返回 `self`,以便在应该使用可迭代对象的地方使用迭代器

示例如下:

```python
s = 'ABC'

# 使用 for 循环隐式创建迭代器
for char in s:
    print(char)

# 显式构建迭代器后,使用 while
it = iter(s)  # 使用可迭代对象显式构建迭代器
while True:
    try:
        print(next(it))  # 不断在迭代器调用 next() 函数,获取下一个元素
    except StopIteration:  # 捕获 StopIteration 异常后,废弃迭代器对象,退出循环
        del it
        break
```

`StopIteration` 异常表明迭代器已经迭代到最后一个元素了.Python 语言内部会处理 for 循环和其他迭代上下文(如列表推导,元组拆包等等)中的 `StopIteration` 异常

> Python 中的迭代器与类型无关,而与协议有关.使用 `hasattr` 检查是否同时具有 `__iter__` 和`__next__` 属性

如下是一个典型的迭代器示例:

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):  # 实现了 __iter__ 方法,该类可迭代
        return SentenceIterator(self.words)  # 根据可迭代协议,__iter__ 方法实例化并返回一个迭代器

class SentenceIterator:

    def __init__(self, words):
        self.words = words
        self.index = 0

    def __next__(self):  # 实现 __next__ 方法,返回按循序索引的单词
        try:
            word = self.words[self.index]
        except IndexError:
            raise StopIteration()
        self.index += 1
        return word

    def __iter__(self):  # 实现 __iter__ 方法,返回迭代器实例,self
        return self
```

## 生成器

```python
# collections.abc.Generator
class Generator(Iterator):

    __slots__ = ()

    def __next__(self):
        """Return the next item from the generator.
        When exhausted, raise StopIteration.
        """
        return self.send(None)

    @abstractmethod
    def send(self, value):
        """Send a value into the generator.
        Return next yielded value or raise StopIteration.
        """
        raise StopIteration

    @abstractmethod
    def throw(self, typ, val=None, tb=None):
        """Raise an exception in the generator.
        Return next yielded value or raise StopIteration.
        """
        if val is None:
            if tb is None:
                raise typ
            val = typ()
        if tb is not None:
            val = val.with_traceback(tb)
        raise val

    def close(self):
        """Raise GeneratorExit inside generator.
        """
        try:
            self.throw(GeneratorExit)
        except (GeneratorExit, StopIteration):
            pass
        else:
            raise RuntimeError("generator ignored GeneratorExit")

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Generator:
            return _check_methods(C, '__iter__', '__next__',
                                  'send', 'throw', 'close')
        return NotImplemented
```

生成器可以理解为特殊的迭代器,它继承自 `Iterator`,实现了如下方法

- `send`: 向生成器发送元素,在调用 `next(g)` 会生成该元素

Python 函数的定义体中有 `yield` 关键字,该函数就是生成器函数.调用生成器函数时,会返回一个生成器对象.也就是说,生成器函数是生成器工厂.

生成器用于 "凭空" 生成元素,见如下示例:

```python
>>> def gen_123():  # 只要函数中包含 yield 关键字,则函数为生成器函数
....    yield 1
....    yield 2
....    yield 3
...
>>> gen_123
<function gen_123 at 0x...>
>>> gen_123() # 调用生成器函数后,会返回一个 generator 生成器对象
<generator object gen_123 at 0x...>

>>> for i in gen_123(): # 生成器是可迭代对象,可以使用 for 循环进行迭代
...    print(i)

>>> g = gen_123() # 将生成器对象赋值给 g,调用 next(g) 会获取 yield 生成的下一个元素
>>> next(g)  # 1
>>> next(g)  # 2
>>> next(g)  # 3
>>> next(g)  # 生成器函数的定义体执行完毕后,生成器对象会抛出 StopIteration 异常
Traceback (most recent call last):
...
StopIteration
```

如下使用生成器代替上述 `SentenceIterator` 类,`Sentence` 就是一个生成器,而不用单独创建一个生成器类

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)  # 将单词全部找到后,赋值给 words 属性的值

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        for word in self.words: # 迭代 self.word
            yield word  # 使用 yield 关键字 "生成" 当前 word
        # return  # 此 return 语句是不必要的,可直接删除
```

### 惰性实现

上述示例在初始化 `__init__` 方法一次性构建好了文本中的单词列表,然后将其绑定为 `self.words` 属性.如下示例是 `Sentence` 类的惰性实现,在处理数据量较大的情况下,可以节省大量内存.

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        # 初始化时,不对 text 进行一次性处理,且不再需要 self.words 属性

    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        for match in RE_WORD.finditer(self.text):  # finditer 函数构建一个迭代器,包含 MatchObject 实例
            yield match.group()  # match.group() 从 MatchObject 实例中取出匹配到的单词,并由 yield 产生
```

### 生成器表达式

生成器表达式可以理解为列表推导的惰性版本,它并不会迫切的构建列表,而是返回一个生成器,按照需要惰性生成元素.如下:

```python
>>> def gen_AB():
... print('start')
... yield 'A'
... print('continue')
... yield 'B'
... print('end.')
...
>>> res1 = [x*3 for x in gen_AB()]  # 列表推导式会自动执行 gen_AB 定义的函数体,输出内容并生成元素
start
continue
end.
>>> for i in res1:  # 对列表推导式生成的列表进行迭代
... print('-->', i)
...
--> AAA
--> BBB
>>> res2 = (x*3 for x in gen_AB()) # 生成器表达式
>>> res2 # 返回生成器对象
<generator object <genexpr> at 0x10063c240>
>>> for i in res2: # for 循环迭代 res2,gen_AB() 定义实体才会真正执行
... print('-->', i)
...
start
--> AAA
continue
--> BBB
end.
```

## 总结

两者都是可迭代对象,都可以使用 `next()` 内置函数返回迭代或生成的元素.在元素"耗尽"时,都会抛出 `StopIteration` 异常

- 迭代器: 使用 `iter()` 函数创建.在创建时,所有待迭代的元素已经产生,且全部加载到内存中
- 生成器: 使用生成器函数或生成器表达式创建.在创建时所有待生成的元素没有产生,不会有内存的损耗.只有在迭代时,生成器函数的函数体或生成式表达式的逻辑体才会执行,生成元素(懒加载).
