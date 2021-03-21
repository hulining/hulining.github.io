---
title: Python 单例模式
date: 2020/06/08
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 通过单例模式可以保证系统中一个类只有一个实例而且该实例易于外界访问,从而方便对实例个数的控制并节约系统资源.
---

单例模式是一种常用的软件设计模式.在它的核心结构中只包含一个被称为单例类的特殊类.通过单例模式可以保证系统中一个类只有一个实例而且该实例易于外界访问,从而方便对实例个数的控制并节约系统资源.

以下是实现单例模式的几种方式:

## 使用共享的类变量

下面 `__new__` 方法中使用共享的类变量 `_instance` 用于保存实例.若为空,则创建新的,否则返回此实例.

```python
class Person(object):
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super().__new__(cls)
        return cls._instance

    def __repr__(self):
        return "name: %s, age: %s" % (self.name, self.age)

p1 = Person('p1', 10)
p2 = Person('p2', 20)

print(p1, p2)
print(p1 is p2) # True
```

## 装饰器实现

```python
def singleton(cls):
    instances = {}
    def getinstance(*args, **kw):
        if cls not in instances:
            instances[cls] = cls(*args, **kw)
        return instances[cls]
    return getinstance

@singleton
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return "name: %s, age: %s" % (self.name, self.age)

p1 = Person('p1', 10)
p2 = Person('p2', 20)

print(p1, p2)
print(p1 is p2) # True
```

## 创建实例后 import 到其它模块

```python
# person.py
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return "name: %s, age: %s" % (self.name, self.age)

    def foo(self):
        print(id(self))

p = Person("p1", 20)

# to use
from persion import p

print(p)
p.foo()
```
