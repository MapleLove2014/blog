---
title: Maximum Product Subarray
author: ZhangMapler
date: 2020-10-28 17:37:00 +0800
categories: [LeetCode]
tags: [python, algorithm, dp]
math: false
---

# Maximum Product Subarray 问题[Medium]

## 问题描述

这个问题是在一个整数数组中求乘积最大的子数组，官网链接点[这里](https://leetcode.com/problems/maximum-product-subarray/)就行啦。

## 问题分析

1. 我们可以先来考虑一个最简单的场景，就是所有的数都大于0。此时最大乘积的子数组就是整个数组。
2. 如果稍微复杂一点，假设所有的数大于等于0。那么从头乘到尾显然是错误的。其实在这种场景下我们可以将原数组按0进行分组:
    
    对于输入 `[1, 2, 0, 3, 4, 0, 5]` 可以分为三个子数组：`[1, 2], [3, 4], [5]`。此时只需要求这个子数组乘积的最大值即可，计算过程很显然，我们需要记住上一次计算的子数组最大乘积值，并和当前子数组的乘积值进行比较。最终得到全局最大的乘积值。

3. 回到最初的这个问题，就是不仅允许有0的存在，还允许负数的存在。此时我们不得不考虑负负得正的情况。

    对于输入 `[-1, 2, 3, -4]` 来说，计算的过程中我们不仅要记住上一次计算的最大乘积值（localMax），还要记住上一次计算的最小乘积值（localMin）。因为说不定在不久的将来，这个最小值就能咸鱼翻身了。

4. 另外，要注意的是计算过程中要保证计算的结果是基于连续性的子数组，为此我们要在整个程序计算过程中维持一个`invariant`，即：
    > 上一次计算的最大的乘积值，一定是基于当前计算位置前的相邻的子数组。它可以是前面的一个数，也可以是前面的好几个相邻的数的乘积。最小乘积值同理。

    为了保持这个`invariant`，我们每次都需要对比3个值：当前值、当前值×localMax和当前值×localMin。都有`当前值`是为了保证连续性。当比较后取的是当前值时，说明此时之前的子数组已经没有用了。此时就把之前的断了，直接使用当前值。如果是`当前值×localMax`，说明之前的数依然有作用。而`当前值×localMin`就是会考虑到负负得正的情况。

## 计算过程

1. 初始化localMin和localMax为1，因为1和任何一个数相乘都不变。globalMax初始化为数组中的最小值。
2. 对于数组中的每一个数num，进行如下计算：
    
    2.1 分别求出 `num, num×localMax， num×localMin`中的最大值min和最小值max；
    2.2 根据max和之前的globalMax更新globalMax；
    2.3 根据min和max更新localMin和localMax；
3. 最终globalMax即为结果。

## 实现

这里给出python代码的实现：

```python
class Solution:
    def maxProduct(self, nums):
        if not nums:
            return 0
        localMin = localMax = 1
        globalMax = min(nums)
        for n in nums:
            newLocalMax = max(n, n*localMin, n*localMax)
            newLocalMin = min(n, n*localMin, n*localMax)
            globalMax = max(globalMax, newLocalMax)
            localMax = newLocalMax
            localMin = newLocalMin
        return globalMax
```
## 参考

https://medium.com/@aliasav/algorithms-maximum-product-subarray-b09e520b4baf