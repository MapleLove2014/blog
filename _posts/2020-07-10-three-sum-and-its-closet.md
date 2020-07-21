---
title: Three Sum 问题解法分析
author: ZhangMapler
date: 2020-07-10 11:43:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Three Sum 问题[Medium]

## 问题描述

[Three Sum](https://leetcode.com/problems/roman-to-integer) ：给定一个整数数组，求所有非重复的3个数，满足条件：3个数之和为 `0` 。

## Brute Force
    
穷举所有的 `3个数` 复杂度为 `O(n^3)`，肯定会 `Time Limit Exceeded`.

## 分治法

我自己想到的思路是，对于求解问题 `find(nums, n, sum)`

> `nums` 表示整数数组，`n` 表示所找的整数个数，`sum` 表示所找整数之和

1. 分解问题，`find(nums, n, sum)`可以分解为以下2个子问题

    - 如果3个数包含当前元素，求解剩下的整数中2个数，满足条件：2数之和为 `sum - nums[i]`，即 `find(nums[i+1:], n-1, sum-nums[i])`。求出结果后，将当前元素和求出的2数结果进行拼接返回

    - 如果3个数不包含当前元素，求解剩下的整数中的3个数，满足条件：3数之和为 `sum` ，即 `find(nums[i+1:], n, sum)`

2. 使用同样的计算方法递归求解子问题，直到

    - `nums`的个数和`n`相等时，直接计算`sum(nums) == sum` ?

    - `nums`的个数小于 `n`，则没有满足条件的 `n` 个数，直接返回空

    - `n == 1`时，停止分解子问题，遍历当前nums每个元素，判断 `nums[i] == sum` ？

3. 最后，合并两个子问题各自返回的结果，最后进行去重返回

### 代码实现

```python
def threeSum(self, nums):
    find, results = self.find(nums, 3, 0)
    if not find:
        return []
    return list({ str(sorted(t)): list(t) for t in results }.values())

def find(self, nums, n, s):
    if len(nums) == n:
        return (True, [tuple(nums)]) if sum(nums) == s else (False, None)
    if len(nums) < n:
        return (False, None)
    if n == 1:
        result = [ (e, ) for e in nums if e == s ] 
        return (True, result) if result else (False, None)
    
    result = []
    isFoundWithFirstNumber, withResults = self.find(nums[1:], n - 1, s - nums[0])
    if isFoundWithFirstNumber:
        result.extend([ (nums[0],) + t for t in withResults])
    isFoundWithoutFirstNumber, withoutResults = self.find(nums[1:], n, s)
    if isFoundWithoutFirstNumber:
        result.extend(withoutResults)
    if not isFoundWithFirstNumber and not isFoundWithoutFirstNumber:
        return (False, None)
    return (True, result)
```

### 复杂度分析

这个方法本质上还是会穷举所有的 `3个数` ， 时间复杂度为 `O(n^3)`。

## Two pointer approach

自己还是没能独立做出这个问题，只好去搜索了，官网的solution还不给看。GeeksforGeeks上有篇[文章](https://www.geeksforgeeks.org/find-a-triplet-that-sum-to-a-given-value/)，提到3种方法，这里只说 `Method 2`，也就是 `Two pointer approach`.

该方法的思路是

1. 先对 `nums` 顺序排序

2. 顺序遍历 `nums` 中的每一个元素，索引为 `i`，创建另外两个索引 `j=i+1`, `k=len(nums)-1`分别指向当前元素后的所有数的首尾，并计算 `loopSum = nums[i] + nums[j] + nums[k]`

    - 当 `loopSum = 0` 时，满足条件，将此3个数保存

    - 当 `loopSum > 0 ` 时，说明当前3数偏大。此时尾指针 `k` 往前移动，因为已经排过序，下一轮的 `loopSum` 肯定会变小，更有可能找到满足条件的3个数

    - 当 `loopSum < 0` 时，同上，头指针 `j` 向后移动

3. 最后将结果去重返回

另外看以下base case

1. 所有整数的个数小于 `3` 时，直接返回空

2. 遍历过程中如果当前元素大于0，直接停止结算，返回结果。因为后面的数无论怎么组合和都会大于0

### 代码实现

```python
def twoPointers(self, nums):
    if not nums or len(nums) < 3:
        return []
    sortedNums = sorted(nums)
    resultDict = {}
    i = 0
    for i in range(len(nums) - 2):
        if sortedNums[i] > 0:
            break
        j = i + 1
        k = len(nums) - 1
        while j < k:
            if sum([sortedNums[i], sortedNums[j], sortedNums[k]]) == 0:
                resultDict['{}-{}-{}'.format(sortedNums[i], sortedNums[j], sortedNums[k])] = [sortedNums[i], sortedNums[j], sortedNums[k]]
                k -= 1
                j += 1
            elif sum([sortedNums[i], sortedNums[j], sortedNums[k]]) > 0:
                k -= 1
            else:
                j += 1
    return list(resultDict.values())
```

### 复杂度分析

两层循环，时间复杂度 `O(n^2)`，但是是 `leetcode` 上提交还是会超时。这就有点头大了，感觉这已经是比较快的了，难道还能再优化？

## 再次优化

当然不是我自己想出来的，来自[Rohan Paul
的文章](https://medium.com/@paulrohan/solving-the-classic-two-sum-and-three-sum-problem-in-javascript-7d5d1d47db03)的最后一步优化

仔细观察上文的遍历过程可以发现，每遍历一个元素 `nums[i]` 时，程序都会从剩下的整数 `nums[i+1:]` 中计算出能和 `nums[i]` 组合成满足条件所有的3个数中的另外2个数的集合 `S(i)`

> 举个例子，`[-4, -2, -2, -1, 0, 1, 1, 2, 4]`，当遍历到 `nums[1]=-2` 时，程序便会从 `[-2, -1, 0, 1, 1, 2, 4]` 中计算出 `S(i) = {[1, 1],[-2, 4,[0, 2]}`

那么可以思考下 `nums[i] = nums[i+1]` 的情况，当程序遍历到 `nums[i+1]` 时，会从剩下的整数 `nums[i+2:]` 中计算 `S(i+1)`，可以很容易的发现一条规律

```
S(i) = UNION( S(i+1) , S(i,i+1) )

S(i,i+1) 表示 所有满足条件的3个数，其中两个数是 nums[i] 和 nums[i+1]
```

即 `S(i+1)` 是 `S(i)` 的子集，既然这样我们就没必要再计算 `nums[i+1]`了

那如果这个重复的整数不连续呢，比如 `nums[i] = nums[j], j > i+1` 时，这个结论依然成立吗？

当然成立，只不过变成了以下形式

```
S(i) = UNION(S(i, (i+1,j-1)),  S(j))

S(i, (i+1,j-1)) 表示 所有满足条件的3个数，其中两个数是 nums[i] 和 nums[i+1]到nums[j-1]中间的一个数
```

> 以上规律的说明并不严谨，也没有严格的数学论证，是一种很直观的 `idea`。本人呢也没兴趣去推导（盲猜可能会用到数学归纳法），Anyway 不会推导也并不影响解这道题，毕竟程序员的第一要务还是解决问题，这些理论的推导就交给专业搞这方面研究的人员去做

### 代码实现

```python
### 省略 ...

for i in range(len(nums) - 2):
    if sortedNums[i] > 0:
        break

    # here: 发现重复元素，直接跳过
    if i > 0 and sortedNums[i - 1] == sortedNums[i]:
        continue

    j = i + 1
    k = len(nums) - 1

### 省略 ...

```

### 复杂度分析

Bad Case: 所有的整数都不相同，复杂度依然是 `O(n^2)`

Best Case: 所有的整数都相同，只需要计算第一个数即可，时间复杂度 `O(n)`

Average Case: 假设有 `m` 个相同的整数，那么时间复杂度为 `O((n-m+1)*n)`

> 也就是优化到这一步，leetcode终于提交成功了。敢情就考到这里呗？

## 总结

做这道题真的挺烦的，感觉每次优化后应该没问题，结果已提交就超时，少一个优化点都不行，服了。不管怎样，最后这个优化点还是挺鸡贼的，要想的很仔细。