---
title: Palindrome Number解法分析
author: ZhangMapler
date: 2020-06-30 10:57:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Palindrome Number问题[Easy]

## 问题描述

判断一个整数是否 `Palindrome`，[官网具体描述点这里](https://leetcode.com/problems/palindrome-number/) .

## 简单实现

看到这个题目，我想到两种简单的方法

1. 将整数转为字符串，判断是否和逆序字符串相等

2. 将整数反转为另一个整数，判断两个整数是否相等

两者本质上没区别，考虑到代码简洁，使用方法1实现。

实现前先看下 base case

1. 当整数为 `负数` 时，返回 `False`

2. 当整数位 `0-9` 时，返回 `False`


### 实现代码

```python
def isPalindrome(self, x: int) -> bool:
    if x < 0:
        return False
    if x < 10:
        return True

    return str(x) == str(x)[::-1]
```

## 优化

观察 `palindrome number` 可以发现，其前半部分和后半部分反转的数值相等。

![图1]({{ "/assets/img/palindrome-number/pn1.png"}})

如果只根据后半部分的反转数值就能对前半部分进行判断，那么可以将整个循环过程减少一半。那具体该怎么做呢？让我们回忆下 [reverse integer](https://blog.zhangmapler.top/posts/reverse-integer/) 中的做法：

![图2]({{ "/assets/img/palindrome-number/pn2.png"}})

在循环过程中 
1. `源整数` 每次 `右移` 一位，`反转数` 每次 `左移` 一位，并加上 `源整数` 移出的 `位`

2. 当 `源整数`的位数不再大于 `反转数` 的位数时（`源整数` <= `反转数`）

    > 说明 `反转数` 至少是 `源数` 的 `后半部分`（当 `源数` 总位数是奇数时，比如 `12321`中 `12 <= 123`）

那么对于 `偶数位` 来说，只要在这个转折点上判断两者是否相同即可。否则，将 `反转数` 右移掉一位后进行比较即可。

另外需要注意如果 `源数` 大于0且能够被 10 整除，即最后一位是 `0`，反转后的数肯定至少少1位，即不是 `palindrome number`，返回False

### 实现代码

```python
def optimized(self, x):
    if x < 0 :
        return False
    if x < 10:
        return True
    if x > 0 and x % 10 == 0:
        return False
    
    reversedHalf = 0
    while x > reversedHalf:
        reversedHalf = reversedHalf * 10 + x % 10
        x //= 10
    
    return x == reversedHalf or x == reversedHalf // 10
```

## 性能分析

首先看一下leetcode的submit结果

![图3]({{ "/assets/img/palindrome-number/pn3.png"}})

咦，为什么优化后的程序不仅慢了20多ms，内存使用也多了一点。我猜可能的原因为python底层int转string的真正实现可能并不是简单的每次处理一位，并和反转数做相加。java的 `Integer#toString()` 方法采用了每次处理2位的方式，并且采用的是移位+映射表来进一步优化。

1. 首先每次处理2位，可以减少1次除运算。cpu乘和除的操作远远大于加操作的耗时。

2. 另外使用移位+映射表 `O(1)` 避免了求余操作（移位操作应该是cpu中运算最快的操作之一，远远小于求余）

优化后的方法只是减少了一半循环次数，时间复杂度和空间复杂度保持不变。这个 `减少一半` 的优化能体现在什么地方呢？我想如果大部分的测试数据都是超级大整数时，可能会有用武之地。

## 总结

这次优化好像也可以说明 `算法 4th ed` 书中所说 `往往简洁的代码性能越高` 这句经验之谈，当然了也要看具体业务场景的。

    