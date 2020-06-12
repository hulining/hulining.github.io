---
title: Python 中 is 与 == 的区别
date: 2020/06/08
tags:
  - Python
  - 面试题
categories:
  - Python
abbrlink: 
description: 简述在作等值判断时,"is" 与 "==" 的区别
---

> Python 对象包含的三个基本要素,分别是: id(身份标识),type(数据类型)和value(值)

`is` 和 `==` 都是等值判断的.但对对象等值判断的内容并并不相同

- `==` 用来比较两个对象的 value 值是否相同
- `is` 用来比较两个对象的 id 是否相同

```python
>>> a = 1
>>> b = 1
>>> a is b
True

>>> a = 1.2
>>> b = 1.2
>>> a is b
False

>>> a = 257
>>> b = 257
>>> a is b
False

>>> a = 'hello'
>>> b = 'hello'
>>> a is b
True

>>> a = 'hello world'
>>> b = 'hello workd'
>>> a is b
False

>>> a = (1,2,3)
>>> b = (1,2,3)
>>> a is b
False

>>> a = [1,2,3]
>>> b = [1,2,3]
>>> a is b
False

>>> a = {'cheese':1,'zh':2}
>>> b = {'cheese':1,'zh':2}
>>> a is b
False

>>> a = set([1,2,3])
>>> b = set([1,2,3])
>>> a is b
False
```

如上示例中,当 a,b 值为 [-5,257) 的整数值时,会返回 `True`.其原因是 Python 为了优化速度,内部维护了小整数对象池,避免整数频繁申请和销毁内存空间.此值的区间为 `[-5, 257)`.

同样的,对于字符串也有一个类似的对象池.但尚不了解字符串长度范围.
