---
title: Python 类中类方法,静态方法与属性方法
date: 2020/06/19
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 简单介绍 Python 类中普通方法,类方法,静态方法与属性方法的定义区别及使用场景.以作备忘.
---

## 类变量与实例变量

- 类变量: 定义类时定义的变量,类的所有实例共享的属性和方法.可以通过类的实例与类进行方法
- 实例变量: 在实例初始化时定义,是每个实例唯一的属性.只能通过类的实例进行访问

如下是 Python 官方文档 - [类变量和实例变量](https://docs.python.org/3/tutorial/classes.html#class-and-instance-variables)中给出的示例.

`Dog` 类中定义了类变量 `kind`,值为 `canine`,此后实例化的所有实例都具有这个共享的属性 `kind = 'canine'`

而 `Dog` 类在实例化(初始化)时,会传入 `name` 参数,并赋值给实例的 `name` 属性,此参数在每个实例中是唯一的.在访问时,只能通过 `self.name` (这里的 `self` 表示类的实例) 进行访问.

```python
class Dog:
    kind = 'canine'         # 所有实例共享的类变量 class variable shared by all instances
    def __init__(self, name):
        self.name = name    # 每个实例唯一的实例变量 instance variable unique to each instance

>>> d = Dog('Fido')
>>> e = Dog('Buddy')
>>> d.kind
'canine'
>>> e.kind
'canine'
>>> d.name
'Fido'
>>> e.name
'Buddy'
>>> Dog.kind
'canine'
```

## 类方法与静态方法

### 示例

- 普通方法: Python 类中定义的普通方法,无需要任何额外的修饰符或装饰器.其第一个参数必须为 `self`,表示调用该方法的实例.只能通过类的实例进行调用.普通方法的方法体中可以使用类变量与实例变量
- 类方法: Python 类中通过 [`@classmethod`](https://docs.python.org/3/library/functions.html#classmethod) 装饰器装饰的方法.其第一个参数必须为 `cls`,表示调用该方法的类.可以通过类名或类的实例进行调用.类方法的方法体中只能使用类变量,不能使用实例变量
- 静态方法: Python 类中通过 [`staticmethod`](https://docs.python.org/3/library/functions.html#staticmethod) 装饰器修饰的方法.其可以传入任意参数.可以通过类名或类的实例进行调用.静态方法的方法体中均不能访问类变量与实例变量

参考 stackflow 上有关[静态方法与类方法的区别](https://stackoverflow.com/questions/136097/difference-between-staticmethod-and-classmethod)的相关回答,示例如下

```python
class A(object):
    def foo(self, x):
        print("executing foo(%s, %s)" % (self, x))
    @classmethod
    def class_foo(cls, x):
        print("executing class_foo(%s, %s)" % (cls, x))
    @staticmethod
    def static_foo(x):
        print("executing static_foo(%s)" % x)

a = A()
```

- 普通方法调用

```python
a.foo(1)
# executing foo(<__main__.A object at 0x0000016FC6A4BBA8>, 1)

print(a.foo)
# <bound method A.foo of <__main__.A object at 0x000002A2F312BB70>>
```

调用过程中,对象的实例 `a` 作为第一个参数 `self` 传入.

- 类方法调用

```python
a.class_foo(1) # 通过类的实例调用
# executing class_foo(<class '__main__.A'>,1)
print(a.class_foo)
# <bound method A.class_foo of <class '__main__.A'>>

A.class_foo(1) # 通过类直接调用
# executing class_foo(<class '__main__.A'>,1)
print(A.class_foo)
# <bound method A.class_foo of <class '__main__.A'>>
```

在使用类的实例进行调用时,类方法会将该实例的类作为第一个参数 `cls` 隐式传入.且类方法可以通过类直接调用,而无需实例化.

类方法的一个应用场景是通过类直接调用类方法创建可继承的替代构造函数.详细可以参考[这里](https://stackoverflow.com/questions/1950414/what-does-classmethod-do-in-this-code/1950927).

- 静态方法调用

```python
a.static_foo(A) # 通过类的实例调用
# executing static_foo(<class '__main__.A'>)
print(a.static_foo)
# <function static_foo at 0xb7d479cc>

A.static_foo(a) # 通过类直接调用
# executing static_foo(<__main__.A object at 0x000002A2F312BB70>)
print(A.static_foo)
# <function static_foo at 0xb7d479cc>
```

静态方法可通过类的实例或类直接调用,而且其传入的参数为任意参数.看起来与直接在 Python 模块中定义方法没有什么区别.

静态方法用于具有与类的类中的一些逻辑连接的功能.

考虑如下场景,当每个类中分别包含一个只有在本类或对象中使用的函数时,如果将其定义在模块中,会导致模块中具有很多这种函数,从而变得臃肿,不容易代码组织,及重构.而定义为静态方法,只需要对类进行组织即可,重构时也会变得简单.

### 总结

以上,可总结如下:

类型 | 定义方式 | 参数 | 调用方式 | 访问属性的权限 | 本质 | 应用场景
:---: | :---: | :---: | :---: | :---: | :---: | :---:
普通方法| 在类中直接定义 | 第一个参数必须传入 `self` 参数 | 通过类的实例进行调用 | 可以访问实例变量和类变量 | 与对象实例绑定的方法 | 实例调用
类方法 | 通过 `@classmethod` 装饰器 | 第一个参数必须是 `cls` 参数 | 通过类名或类的实例进行调用 | 不能访问实例变量,可以访问类变量 | 与类绑定的方法 | 创建可继承的构造函数
静态方法| 通过 `@staticmethod` 装饰器 | 可传入任意参数 | 通过类名或类的实例直接调用 | 不能访问实例变量和类变量 | 本质是逻辑上的类函数 | 组织代码,而不是将所有函数的变体定义在模块中

## 属性方法

Python 中支持使用 `@property`,`@<property>.setter`,`@<property>.deleter` 将类中定义的方法转换为类对象的属性.其中 `<property>` 表示被 `@property` 装饰的返回实例属性(包括类共享属性及实例唯一属性)的方法,且以上三个装饰器修饰的方法名必须一致.如:

```python
class C:
    def __init__(self):
        self._x = None
    @property # 添加装饰器,将 x 方法转换为 C 对象实例的一个属性,可通过 `c.x` 直接执行此方法体内容,从而返回属性值
    def x(self):
        """I'm the 'x' property."""
        return self._x
    @x.setter # 添加装饰器,设置可通过 `c.x = xxx` 直接调用此方法体内容,从而修改属性值
    def x(self, value):
        self._x = value
    @x.deleter # 添加装饰器,设置可通过 `del c.x` 直接调用此方法体内容,从而删除或修改属性值
    def x(self):
        del self._x

>>> c = C()
>>> c.x = 1 # 可直接修改
>>> c.x # 可直接访问
>>> del c.x # 可直接删除
```

其实,Python 中内置了 `property` 类,其返回一个 `属性` 对象.其中 `fget` 是用于获取属性值的函数,`fset` 是用于设置属性值的函数,`fdel` 是用于删除该属性的函数,`doc` 是该属性的帮助文档
.

```python
class property(fget=None, fset=None, fdel=None, doc=None):
    # ...
    pass
```

其使用方式如下:

```python
class C:
    def __init__(self):
        self._x = None

    def getx(self):
        return self._x

    def setx(self, value):
        self._x = value

    def delx(self):
        del self._x

    x = property(getx, setx, delx, "I'm the 'x' property.")
```
