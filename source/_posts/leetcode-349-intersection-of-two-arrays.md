---
title: leetcode 349. 两个数组的交集
date: 2020/06/06
tags:
  - leetcode
categories:
  - leetcode
---

## 题目描述

给定两个数组,编写一个函数来计算它们的交集.

- 示例 1:

```text
输入: nums1 = [1,2,2,1], nums2 = [2,2]
输出: [2]
```

- 示例 2:

```text
输入: nums1 = [4,9,5], nums2 = [9,4,9,8,4]
输出: [9,4]
```

< !--more-->

说明:

- 输出结果中的每个元素一定是唯一的。
- 我们可以不考虑输出结果的顺序。

## 题解

### 两个 set

首先将两个数组转换为集合,去除两个集合中的重复元素.然后迭代较小集合是否在较大集合中.

```python
def set_intersection(s1, s2):
    return [x for x in s1 if x in s2]

def intersection(self, nums1, nums2):
    s1 = set(nums1)
    s2 = set(nums2)
    if len(s1) < len(s2):
        return set_intersection(s1, s2)
    else:
        return set_intersection(s2, s1)
```

#### 复杂度分析

- 时间复杂度: *O(m + n)*, `m`,`n` 分别是数组的长度
- 空间复杂度: *O(m + n)*
