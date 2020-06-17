---
title: Python 中 __init__ 和 __new__ 的区别
date: 2020/06/08
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 简述 Python 中 __init__ 与 __new__ 两个函数的区别
---

- `__new__` 是实例对象真正的创建函数,在实例对象创建之前被调用,用于创建实例并返回该实例对象
- `__init__` 是实例对象的初始化函数,在实例对象创建完成后被调用,用于设置实例对象的初始化属性

区别 | `__new__` | `__init__`
:---: | :---: | :---:
类型 | 静态方法,但不用使用 `@staticmethod` 装饰器显示声明 | 实例方法
参数 | 至少包含 `cls`,代表当前类 | 至少包含 `self`,代表当前实例
返回值 | 返回对象的实例 | 禁止返回任何值
调用顺序 | 在对象创建前调用 | 对象创建后初始化过程中调用

说明

- 如果 `__new__` 创建的是当前类的实例,会自动调用_ `__init__` 函数进行初始化,否则后面的 `__init__` 函数不会被调用

```python
class A(object):
    def __new__(cls, *args, **kwargs):
        print('__new__ method in class A')
        return super(A, cls).__new__(cls)

    def __init__(self, name):
        print('__init__ method in class A')
        self.name = name

a = A("name")
```

[牛客网](https://www.nowcoder.com/profile/701230/myFollowings/detail/5726157)上有关区别的一道题:

```text
__new__和__init__的区别，说法正确的是？ ABCD

A. __new__ 是一个静态方法,而 __init__ 是一个实例方法
B. __new__ 方法会返回一个创建的实例,而 __init__ 什么都不返回
C. 只有在 __new__ 返回一个 cls 的实例时,后面的 __init__ 才能被调用
D. 当创建一个新实例时调用 __new__,初始化一个实例时用 __init__
````

### 使用自定义 `__new__` 函数对不可变类型(int, str, tuple)进行修改

```python
class AbsoluteValue(int):
    # 对 __new__ 函数进行重写,以 abs(value) 创建其实例
    def __new__(cls, value):
        return super(AbsoluteValue, cls).__new__(cls, abs(value))

abv = AbsoluteValue(-1)
print(abv)
```

### 使用自定义 `__new__` 函数实现单例模式

```python
class Person(object):
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return "name: %s, age: %s" % (self.name, self.age)

    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, 'instance'):
            cls.instance = super(Person, cls).__new__(cls)
        return cls.instance

p1 = Person('p1', 20)
p2 = Person('p2', 10)

print(p1, p2)
print(p1 is p2) # True
```
