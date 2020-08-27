---
title: 只用一次遍历删除链表倒数第n个元素
author: ZhangMapler
date: 2020-08-27 10:37:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Remove Nth Node From End of List 问题[Medium]

## 问题描述

[Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) ：单向链表删除倒数第n个元素。

## 2次遍历
    
学过基础数据结构的同学，对于这个问题应该来说还是比较简单的。从头开始两次遍历：
1. 第一次先确定链表的长度 `m`。
2. 第二次遍历到第 `m-n` 个元素 `k`，`k` 此时正好位于第 `n` 个元素的前驱节点，直接删除即可。

举个栗子，我们有这样一个链表：
> a->b->c->d->e

第一次遍历可以得到长度为 `5`，假设 `n=2`那么第二次遍历三个元素到节点 `c`，正好是被删除节点的前驱节点。此时只需要将 `c->next` 指向节点 `e` 即可完成删除。

> 单向链表节点的删除，一定要定位到其前驱节点才能删除。

接着再考虑下边界条件：
1. 当链表长度小于 `n` 时，说明不存在倒数第 `n` 个节点，无需做任何操作。
2. 当链表长度等于 `n` 时，说明要删除的元素是第一个元素 `head` 节点，那么只需要返回 `head->next` 即可。

实现很简单，这里就不贴代码了，要注意边界条件喔。

## 1次遍历

通过观察可以发现，倒数第n+1个节点（被删除节点的前驱节点）和最后一个节点之间的距离是 `n-1` 个节点，`c` 和 `e` 节点间的距离是 `2 = n`。
```
    first second
      |     | 
a->b->c->d->e
```
此时我们用距离为2的两个固定相连的游标分别指向这两个节点，然后往前平移直到链表头部。此时第一个游标指向头部，第二个游标指向第3个元素。即：
```
first second
|     | 
a->b->c->d->e
```
妙就妙在，我们再做一次反向操作，平移到尾部。此时 `first` 游标正好指向被删除节点的前驱节点，`second` 指向尾节点。这个时候就可以愉快的删除了。

基于此，我们可以设计实现1次遍历的方法。为了好判断，`second` 游标指向最后一个节点的下一个节点 `null`。
1. 首先 `second` 游标从头遍历 `n+1` 个元素（包括尾节点的next节点null）。
2. `first` 和 `second` 相同步伐往后挪动，直到 `second=null` 时，结束遍历过程。
3. 根绝 `first` 指向的节点进行删除操作。

考虑下边界条件：
1. 当链表长度等于 `n=2` 时，第一次 `second` 走到 `next` 节点，向后走了 `n` 个节点。
    ```
    first second
    |       | 
    a->b->null
    ```

2. 当链表长度小于 `n=2` 时，第一次 `second` 还未遍历够 `n` 个元素便走到了 `null` 节点。

    ```
    first second
    |       | 
    a ---> null
    ```

ok，来看下实现代码：

```python
def onePass(self, head, n):
    store = head
    first = head
    second = head
    count = 0
    while second and count < n + 1:
        count += 1
        second = second.next
    if not second and count < n + 1:
        return store if count < n else store.next
    while second:
        second = second.next
        first = first.next
    first.next = first.next.next
    return store
```

## 优化

其实上面的一次遍历看起来挺容易的，其实真正写程序时很容易出错，特别时边界条件的判断。我调了好几次，才通过的。

有什么方法可以把边界条件去掉吗？LeetCode上的[Solution](https://leetcode.com/problems/remove-nth-node-from-end-of-list/solution/)给出了一个方法，就是引入一个**Dummy Node**。它是怎么做的呢？

Dummy Node和普通的节点没有区别，只不过把它放在链表首节点的前驱节点，然后让 `first` 和 `second` 都指向Dummy Node。

> dummy->a->b->c->d->e

这个时候，我们再来看边界条件：
1. 当链表长度和 `n` 相等时，由于增加了一个Dummy Node，`second` 便可以正好遍历 `n+1` 个元素到 `null` 节点。此时 `first` 正好指向被删除节点（链表首节点）的前驱节点Dummy Node。
    ```
    first      second
      |           | 
    dummy->a->b->null
    ```
    这样正好可以和正常删除的逻辑统一，完事后返回 `dummy->next` 即可。

2. 当链表长度小于 `n` 时，上文分析不变。

> 其实LeetCode原题是保证了**Given n will always be valid.**，也就意味着链表长度不会小于 `n`。那么就可以不用考虑第二种情况了。

代码可以参考官网的Solution，我就不贴了。

### 复杂度分析

假设链表长度为 `n`，时间复杂度为 `O(n)`。2次遍历的时间复杂度也是 `O(n)` 喔。

## 总结

说实话，边界条件那里老出错，真的很烦。有的时候，好像你看上去都把逻辑理顺了。但真正去写的时候，还是一堆错误。