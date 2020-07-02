---
title: ContainerWithMostWater解法分析
author: ZhangMapler
date: 2020-07-02 11:58:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Container With Most Water问题[Medium]

## 问题描述

一个表示木板高度的数组，求两边木板围成的最大面积，也就是可以 `contain most water` ，[官网具体描述点这里](https://leetcode.com/problems/container-with-most-water/) .

## 问题分析

这个Medium的问题一度给我一种错觉：这道题很简单。当然也没错，`Brute Force` 方法很容易就能写出来，时间复杂度 `O(n^2)`。有没有快一点的呢？

## 2个思路

我还是想了一阵子，隐隐约约有2个思路（都是歪的，建议直接跳过）

1. 能不能参考 `0-1背包问题` 的动态规划解法呢？

    > 从头开始遍历，递归的求解以每个木板为头的最大面积。但是，仔细一琢磨，`maxArea[i,?]` 和 `maxArea[i+1,?]` 的计算并没有直接的关系。因为面积只需要首尾高度和宽度相乘就可以了，并不能通过子问题的结果来减少父问题的计算。

2. 参考 `最长整数和问题` 的分治方法

    > 围成最大面积的首尾木板一定位于 `前一半木板`, `中间连续木板部分`, `后一半木板` 中。基于此，将问题分为3个子问题进行求解，最后合并步骤中返回子问题中最大面积的首尾木板。乍一听好像可行，但是问题分解到最后发现由于面积计算的特殊性，导致和方法1一样的问题，没有减少计算。

好吧，我承认我能力不太行，最后还是看了 [Solution](https://leetcode.com/problems/container-with-most-water/solution/) . `Brute Force` 方法就不说了，直接看 `Approach 2: Two Pointer Approach`.

## Two Pointer Approach

Intuition: 两个指针分别指向首尾木板，然后每次计算，将两个指针逐步靠拢，直到相遇结束，并记录最大的面积。方法的关键在于指针如何靠拢？

计算面积时，是由首尾木板的短板控制的。我们的靠拢的目标是 `寻找更大的面积` ，这一点很重要。当靠拢时，宽度肯定会缩小，那么只有找到一个高的下一个木板，面积才有可能超过当前的面积。显然，如果移动首尾木板中的长板，则新围成的面积一定不会大于当前的面积。这就是 `Two Pointer Approach` 的核心思想，移动短板很好理解。但是当两个木板高度相同时移动哪一个呢？还是两个同时靠拢呢？

![图1]({{ "/assets/img/container-with-most-water/cwmw1.png"}})

上图中 `i,j` 对应的木板高度相同, 面积为 `O`。`bc`木板围成最大的面积，即绿色面积为 `Q`。`i-b`和`c-j`之间没有一个木板 `z` 可以和`i`或者`j`围成一个面积大于 `Q` 的区域。

> `i-b`和`c-j`之间的木板高度一定小于 `i` 木板高度

我们假设 `Q > O`，我们靠拢的目标是 `寻找更大的面积` ，也就是 `Q`。那么我不管算法怎么靠拢的，只要靠拢后你能够定位到这块区域就可以了，从而保证算法的正确性。

> 面积是短板控制的，在宽度减小的情况下只有增加高度才能增大面积。如果 `i-j` 之间有更大的面积区域 `x-y`，那么`x`和`y`木板的高度一定大于`i`或者`j`木板的高度。

那么假设先移动`i`，这样我们就可以忽略`i-b`之间高度小于`b`木板高度的木板。如果 `i-b`之间有高于`b`木板高度的木板，那么最大面积就一定不是`Q`了。根据基本的短板移动的原则，慢慢的 `i`和`j`会分别靠拢在`b`和`c`，记录下最大面积 `Q`。

如果这里可以理解的话，其实可以发现，先移动 `j` 也是同样的结果，同时移动`i`和`j`亦然，都可以保证一定可以定位到最大面积区域，保证正确性。


### 实现代码

```python
def maxArea(self, height: List[int]) -> int:
    left = 0
    right = len(height) - 1
    maxArea = 0
    while right > left:
        maxArea = max(maxArea, min(height[left], height[right]) * (right - left))
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return maxArea
```

### 复杂度分析

遍历一遍数组，时间复杂度为 `O(n)`，其中 `n` 指数组长度。

## 总结

就像官网上所说的一样，`Two Pointer Approach` 不是很直观的能够理解，总是会想这样做真的对吗？短板移动还好理解点，相同的时候确实会有一点疑惑。所以本篇blog也是主要针对这一点展开的，希望能对你有帮助。如果有什么错误或者疑惑可以在评论区贴出来，一起讨论。

    