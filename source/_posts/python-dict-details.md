---
title: Python 字典与集合
date: 2020/06/19
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 本文主要介绍 Python 中字典的实现及相关注意事项
---

Python 中字典有如下注意事项

- 键必须是可散列的.一个可散列对象必须满足以下要求:
  1. 支持 `hash()` 函数,并且通过 `__hash__()` 方法得到的散列值是不变的.
  2. 支持通过 `__eq__()` 方法检测相等性
  3. 若 `a == b` 为真,则 `hash(a) == hash(b)` 也为真
- 字典在内存上的开销巨大.但键查询很快,dict 的实现是典型的空间换时间
- 键的次序取决于添加顺序,但如果添加相同的键值对到字典中,该字典是相等的.如

```python
DIAL_CODES = [
    (86, 'China'),
    (91, 'India'),
    (1, 'United States'),
    (62, 'Indonesia'),
    (55, 'Brazil'),
    (92, 'Pakistan'),
    (880, 'Bangladesh'),
    (234, 'Nigeria'),
    (7, 'Russia'),
    (81, 'Japan'),
]
d1 = dict(DIAL_CODES)
print('d1:', d1.keys())
d2 = dict(sorted(DIAL_CODES))
print('d2:', d2.keys())
d3 = dict(sorted(DIAL_CODES, key=lambda x:x[1]))
print('d3:', d3.keys())
assert d1 == d2 and d2 == d3

# 输出如下:
# d1: dict_keys([86, 91, 1, 62, 55, 92, 880, 234, 7, 81])
# d2: dict_keys([1, 7, 55, 62, 81, 86, 91, 92, 234, 880])
# d3: dict_keys([880, 55, 86, 91, 62, 81, 234, 92, 7, 1])
# 最后的 assert 为 True
```

- 往字典里添加新键可能会改变已有键的顺序.**不要对字典同时进行迭代和修改**

无论何时往字典里添加新的键,Python 解释器都可能为字典进行扩容.扩容导致的结果就是要新建一个更大的散列表,并把字典里已有的元素添加到新表里.这个过程中可能会发生新的散列冲突,导致新散列表中键的次序变化.因此,**不要对字典同时进行迭代和修改**

> 由于集合与字典内部实现一致.以上提到的内容,同样适用于集合
