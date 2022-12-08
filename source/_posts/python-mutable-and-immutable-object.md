---
title: Python 可变对象与不可变对象及其复制
date: 2020/06/18
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 本文主要整理 Python 中可变对象与不可变对象的区别,以及它们在实际应用时应当注意的内容.另外,本文还描述了 Python 两种对象在复制时的本质.
---

## 可变对象与不可变对象

可变对象与不可变对象的区别在于对象本身是否可变.

在 Python 常用的数据类型中,

- 可变对象: list, dict, set
- 不可变对象: int, float, str, bool, tuple

如下示例直观的展示了 `list` 可变对象 与 `tuple` 不可变对象的区别

```python
# 可变对象
>>> a = [1, 2, 3]
>>> a[1] = 4
>>> a
[1, 4, 3]
# 不可变对象
>>> b = (1, 2, 3)
>>> b[1] = 4
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
```

### 元组的相对不可变性

元组与多数 Python 集合(list,dict,set 等等)一样,保存的是对象的引用.如果引用的元素是可变的,即便元组本身不可变,元素依然可变.也就是说,元组的不可变性其实是指 `tuple` 数据结构的物理内容(即保存的引用)不可变,与引用的对象无关

```python
>>> t1 = (1, 2, [30, 40])
>>> t2 = (1, 2, [30, 40])
>>> t1 == t2
True
>>> id(t1[-1])
4302515784
>>> t1[-1].append(99)
>>> t1
(1, 2, [30, 40, 99])
>>> id(t1[-1])
4302515784
>>> t1 == t2
False
```

## 深浅拷贝

对于不可变对象来说,不管是深拷贝还是浅拷贝,都是对该对象添加一次引用,其内存地址及内部元素的内存地址均不会改变.如

```python
import copy

# 定义函数,打印其内存地址,并判断是否相同

imutable_object = ('a', ('b', ('c', 'd'), 'e'), 'f') # 不可变对象
imutable_copy_object = copy.copy(imutable_object)
imutable_deepcopy_object = copy.deepcopy(imutable_object)

print('%s\t%s\t%s' % (id(imutable_object), id(imutable_copy_object), id(imutable_deepcopy_object)))
print('%s\t%s\t%s' % (id(imutable_object[1]), id(imutable_copy_object[1]), id(imutable_deepcopy_object[1])))
print('%s\t%s\t%s' % (id(imutable_object[1][1]), id(imutable_copy_object[1][1]), id(imutable_deepcopy_object[1][1])))
# 输出如下:
2364244967072   2364244967072   2364244967072
2364244737480   2364244737480   2364244737480
2364244918728   2364244918728   2364244918728

# 不管是深拷贝还是浅拷贝,不可变对象的每一层元素的地址不会改变
```

而对于可变对象来说,深拷贝与浅拷贝的拷贝粒度是不相同的,如

```python
import copy

mutable_object = ['a', ['b', ['c', 'd'], 'e'], 'f'] # 可变对象
mutable_copy_object = copy.copy(mutable_object)
mutable_deepcopy_object = copy.deepcopy(mutable_object)

print('%s\t%s\t%s' % (id(mutable_object), id(mutable_copy_object), id(mutable_deepcopy_object)))
print('%s\t%s\t%s' % (id(mutable_object[0]), id(mutable_copy_object[0]), id(mutable_deepcopy_object[0])))
print('%s\t%s\t%s' % (id(mutable_object[1]), id(mutable_copy_object[1]), id(mutable_deepcopy_object[1])))
print('%s\t%s\t%s' % (id(mutable_object[1][1]), id(mutable_copy_object[1][1]), id(mutable_deepcopy_object[1][1])))

# 输出如下
1625855263240   1625853920840   1625855164296  # 对象的内存地址均发生改变
1625816559208   1625816559208   1625816559208  # 对象中的不可变对象元素地址未发生改变,个人推测可能是 Python 在底层做了优化
1625855263048   1625855263048   1625855164232  # 浅拷贝对象中可变对象元素地址未发生改变,深拷贝中可变对象元素地址发生改变
1625846665032   1625846665032   1625855156360  # 浅拷贝对象中第二层可变对象元素地址未发生改变,深拷贝中第二层可变对象元素地址发生改变
```

因此总结如下

- 浅拷贝可以理解为对象中不可变对象元素进行一次完整拷贝,并直接引用原来的可变对象元素.原始对象与浅拷贝对象不是同一个对象,只是值相同而已.修改浅深贝对象中的可变对象会对原始对象造成影响.
- 深拷贝可以理解为对象中数据进行一次完整的拷贝,会重新分配内存地址.原始对象与深拷贝对象不是同一个对象,只是值相同而已.修改浅深贝对象不会对原始对象造成影响.

![深拷贝与浅拷贝](/images/copy-and-deep-copy.jpg)

```python
import copy
a = [1,2,[3,4]]
b = copy.copy(a)
c = copy.deepcopy(a)
b.insert(0,0)
b[-1].append(5)

```

![深拷贝与浅拷贝的修改](/images/copy-and-deep-copy-modify.jpg)

可在此[网址](http://www.pythontutor.com/live.html#mode=edit)查看如上代码完整的过程.

### 什么时候发生深/浅拷贝,各自的应用场景有哪些

以列表为例,如下情况发生浅拷贝:

- 使用 copy 方法,如 `copy_list = copy.copy(list_1)`
- 构造方法,如 `copy_list = list(list_1)`
- 切片,如 `copy_list = list_1[:]`

应用如下:

```python
# 构造 person 模版,存款为0,并 copy 两个人,有共同 saving,比如说共同存款帐号
person = ['name', ['saving', 0]]
person_1 = person[:]
person_2 = person[:]
person_1[0] = 'name_1'
person_2[0] = 'name_2'
person_2[1][1] += 100 # 其中一个向其中存了 100
>>> person # 此时发现共同存款为 100
['name', ['saving', 100]]
```

其它情况发生的都是深拷贝

## 参数传递

Python 中函数的传参与 Go 中函数参数传递类似.均可以理解为对变量值进行拷贝后进行传入函数中.只不过,对于不可变参数来说,传入的是对象的内存地址.而对于可变对象来说,传入的是指向对象内存地址的内存地址.

Python 唯一支持的参数传递模式是共享传参.共享传参指函数的各个形式参数获得实参中各个引用的副本.也就是说,函数内部的形参是实参的别名.

这种情况下,函数可能会修改作为参数传入的可变对象,但是无法修改那些对象的标识.

```python
>>> def f(a, b):
...     a += b
...     return a
...
>>> x = 1
>>> y = 2
>>> f(x, y)
3
>>> x, y
(1, 2)
>>> a = [1, 2]
>>> b = [3, 4]
>>> f(a, b)
[1, 2, 3, 4]
>>> a, b
([1, 2, 3, 4], [3, 4])
>>> t = (10, 20)
>>> u = (30, 40)
>>> f(t, u)
(10, 20, 30, 40)
>>> t, u
((10, 20), (30, 40))
```

对于上述示例,可以总结如下:

对于函数传参调用 `f(obj_1, obj_2)` 来说,函数内部其实是将 `obj_1` 添加了引用(或设置了别名/标签) `a`,`obj_2` 添加了引用(或设置了别名/标签)`b`,即执行了如下代码 `a=obj_1`,`b=obj_2`.因此,在函数中的 `+=` 操作会影响到原始对象.只不过对于不可变对象来说,`+=` 操作会重新分配内存空间,然后赋值给同名变量.

### 不要使用可变对象作为参数的默认值

如果使用可变对象作为默认值,则在新创建对象时,所有的赋值操作都是对该可变对象添加新的引用(或贴新的标签),而其实底层使用的是同一个对象.

如下是一个将空列表作为默认参数传入示例,简单说明可变默认值可能引发的问题.

```python
class HauntedBus:
    """备受幽灵乘客折磨的校车"""
    def __init__(self, passengers=[]):
        self.passengers = passengers
    def pick(self, name):
        self.passengers.append(name)
    def drop(self, name):
        self.passengers.remove(name)
```

下面是对该类进行的引用

```python
>>> bus1 = HauntedBus(['Alice', 'Bill']) # bus1 不使用默认的参数值,对后续对象创建没有影响
>>> bus1.passengers
['Alice', 'Bill']
>>> bus1.pick('Charlie')
>>> bus1.drop('Alice')
>>> bus1.passengers
['Bill', 'Charlie']

>>> bus2 = HauntedBus() # 一开始,bus2 是空的,因此把默认的空列表赋值给 self.passengers
>>> bus2.pick('Carrie')
>>> bus2.passengers
['Carrie']
>>> bus3 = HauntedBus() # bus3 一开始也是空的,因此还是赋值默认的列表
>>> bus3.passengers # 但是默认列表不为空,原因是 bus2 和 bus3 使用了同一个空列表.赋值操作只是对该空列表添加了引用.
['Carrie']
>>> bus3.pick('Dave') # 修改 bus3 的列表,也会影响到 bus2 的列表
>>> bus2.passengers
['Carrie', 'Dave']
>>> bus2.passengers is bus3.passengers # bus2 bus3 中列表为同一个列表
True
```

为解决上述问题,我们做如下修改:

```python
class TwilightBus:
    """让乘客销声匿迹的校车"""
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = [] # 如果不传入参数,则创建并使用空列表
        else:
            self.passengers = passengers # 否则,使用传入的参数
    def pick(self, name):
        self.passengers.append(name)
    def drop(self, name):
        self.passengers.remove(name)

>>> bus2 = TwilightBus() # 一开始,bus2 是空的,因此创建一个新的空列表并将其赋值给 self.passengers
>>> bus2.pick('Carrie')
>>> bus2.passengers
['Carrie']
>>> bus3 = TwilightBus() # bus3 一开始也是空的,因此创建一个新的空列表并将其赋值给 self.passengers
>>> bus3.passengers # 默认为空
[]
>>> bus3.pick('Alice')
>>> bus2.passengers # 对 bus3 中 passengers 操作不会响应到 bus2 中的 passengers
['Carrie']
>>> bus2.passengers is bus3.passengers
False

# >>> a = []
# >>> b = []
# >>> a is b
# False
```

以上示例虽然解决了前一个示例中两个对象引用一个列表的问题,但是当传入参数是可变对象(如 list) 时,所有对该 list 的操作会影响到外部 list.如

```python
>>> basketball_team = ['Sue', 'Tina', 'Maya', 'Diana', 'Pat'] # 假设我们这个队里有 5 个学生
>>> bus = TwilightBus(basketball_team) # 使用队学生初始化 TwilightBus
>>> bus.drop('Tina') # 一个学生下车
>>> bus.drop('Pat')
>>> basketball_team  # 此时 basketball_team 变为了只有 3 个学生,显然是不符合逻辑的
['Sue', 'Maya', 'Diana']
```

产生上述问题的原因是由于对函数传入可变对象后,在函数中的所有操作都会影响到原始对象.因此做如下改进

```python
class Bus:
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []
        else:
            self.passengers = list(passengers)
            # 对传入参数进行拷贝后复制给 self.passengers,函数内所有操作在 self.passengers 上进行,而不影响传入的 passengers 参数
            # 这种传入参数更为灵活,可以是元组或其他任何可迭代对象
    def pick(self, name):
        self.passengers.append(name)
    def drop(self, name):
        self.passengers.remove(name)
```
