---
title: leetcode 217. 存在重复元素
date: 2020/06/06
tags:
  - leetcode
categories:
  - leetcode
---

## 题目描述

给定一个整数数组,判断是否存在重复元素

- 示例 1:

```text
输入: [1,2,3,1]
输出: true
```

- 示例 2:

```text
输入: [1,2,3,4]
输出: false
```

- 示例 3:

```text
输入: [1,1,1,3,3,4,3,2,4,2]
输出: true
```

< !--more-->

## 题解

### 集合

创建空集合,判断数组值是否在集合中.如果存在,则返回.否则添加到集合中,直到数组结束.

```python
def containsDuplicate(nums):
    s = set()
    for x in nums:
        if x in s:
            return true
        else:
            s.add(x)
    return false

```

#### 复杂度分析

- 时间复杂度: *O(n)*
- 空间复杂度: *O(m)*

### 集合的与原数组长度是否一致

将数组转换为集合后,判断集合长度是否与数组长度是否一致.若一致,则没有重复元素,否则存在重复元素.

```python
def containsDuplicate(nums):
    return len(nums) > len(set(nums))
```

#### 复杂度分析

- 时间复杂度: *O(n)*
- 空间复杂度: *O(m)*
