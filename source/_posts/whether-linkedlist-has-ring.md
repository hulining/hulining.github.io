---
title: 判断一个链表是否有环
date: 2020/06/28
tags:
  - 数据结构
  - 算法
  - LinkedList
categories:
  - 算法与数据结构
description: 1. 判断一个链表是否有环,如果有,则找到环的入口点.2. 判断两个单链表是否相交,如果相交,给出相交的第一个点
---

## 判断一个链表是否有环,如果有,则找到环的入口点

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

    def __repr__(self):
        retrun '%s' % self.value

node1 = Node(1)
node2 = Node(2)
node3 = Node(3)
node4 = Node(4)
node5 = Node(5)
node1.next = node2
node2.next = node3
node3.next = node4
node4.next = node5
node5.next = node2
```

### 要求

- 不允许修改链表结构
- 时间复杂度O(n),空间复杂度O(1)

### 思考

#### 集合

创建一个存储节点 ID 为的集合,来存储曾经遍历过的节点.从头节点开始,依次遍历单链表的每一个节点.每遍历到一个新节点,就用新节点和集合当中存储的节点作比较,如果发现集合中存在相同节点 ID,则说明链表有环;否则存入新的节点.

```python
def whether_has_ring(node):
    node_set = set()
    while node:
        if node in node_set:
            return node
        node_set.add(node)
        node = node.next
    return None
```

或利用集合没有重复元素的特性,通过判断集合长度来判断链表是否具有环型结构

```python
def whether_has_ring(node):
    node_set = set()
    n = 0
    while node:
        node_set.add(node)
        n += 1
        if len(node_set) != n:
            return node
        node = node.next
    return None
```

- 时间复杂度: O(n)
- 空间复杂度: O(n)

#### 快慢指针

使用 slow,fast 两个指针对链表进行遍历.slow 指针一次遍历 1个节点,fast 指针一次遍历 2 个节点.当快慢指针相交时,则表示链表有环.

若有环,则使用一个指针指向链表头节点,一个指针指向第一次的交汇点,一起向后遍历,当两个指针相等时,此时相交的位置即为环的入口点.

```python
def whether_has_ring(node):
    slow = node
    fast = node
    while fast.next and fast.next.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast: # 如果快慢指针相交,则表示有环.并
            slow = node
            while slow.next and fast.next:
                slow = slow.next # 使慢从头开始遍历
                fast = fast.next # 快指针从相交位置开始遍历
                if slow == fast: # 再次相交的位置为环的入口点
                    return slow
    return None
```

- 时间复杂度: O(n)
- 空间复杂度: O(1)

## 判断两个单链表是否相交

### 集合

如果两个链表相交,那么它们一定会有公共的结点.

创建一个存储节点 ID 为的集合,来存储链表 A 中节点.从链表 B 头节点开始,依次遍历单链表的每一个节点.每遍历到一个新节点,就用新节点和集合当中存储的节点作比较,如果发现集合中存在相同节点 ID,则说明链表相交;

```python
def whether_intersect(node1, node2):
    node_set = set()
    node = node1
    while node:
        node_set.add(node)
        node = node.next
    node = node2
    while node:
        if node in node_set:
            return node
        node = node.next
    return None
```

- 时间复杂度: O(m + n) = O(n)
- 空间复杂度: O(n)

### 尾节点法

如果两个链表相交,那么它们的最后一个节点一定相等.

首先判断两个链表尾节点是否相等,并获得两个链表的长度,l1,l2.若相等,使长链表先遍历 l1-l2 的绝对值长度.然后长短链表依次遍历.节点相等则为链表相交的位置.

```python
def whether_intersect2(node1, node2):
    l1 = 1 # l1 表示 node1 的长度
    l2 = 1 # l1 表示 node2 的长度
    node_a = node1 # node_a 表示遍历 node1 的指针
    node_b = node2 # node_b 表示遍历 node2 的指针

    # node_a, node_b 分别遍历到结尾,并判断是否相等
    while node_a.next:
        node_a = node_a.next
        l1 += 1
    while node_b.next:
        node_b = node_b.next
        l2 += 1
    if node_a == node_b: # 如果相等,则都从头开始
        node_a = node1
        node_b = node2
        if l1 > l2: # l1 > l2,则 node_a 遍历 l1 -  l2 长度
            for i in range(l1 - l2):
                node_a = node_a.next
        if l1 < l2: # l1 < l2,则 node_b 遍历 l2 -  l1 长度
            for i in range(l2 - l1):
                node_b = node_b.next
        while node_a and node_b: # 然后开始遍历,并判断是否相交
            if node_a == node_b:
                return node_a
            node_a = node_a.next
            node_b = node_b.next
    return None
```

- 时间复杂度: O(2(m+n)) = O(n)
- 空间复杂度: O(1)

### 利用有环链表

将其中一个链表首位相接,若相交,则剩下的结构为有环链表.利用判断是否是有环链表及获取有环链表的入口点即可解决问题

```python
def whether_intersect3(node1, node2):
    # 将 node1 转换为环
    node_a = node1
    while node1.next:
        node1 = node1.next
    node1.next = node_a

    # 调用上面的函数,判断 node2 是否有环
    return whether_has_ring(node2)
```

- 时间复杂度: O(2(m+n)) = O(n)
- 空间复杂度: O(1) 或 O(n)
