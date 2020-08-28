---
title: Manacher's Algorithm
author: ZhangMapler
date: 2020-08-28 10:37:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Longest Palindromic Substring 问题[Medium]

## 问题描述和分析

[Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) ：求满足Palindromic条件的最长子字符串。

这个问题官网一共给了[5种解法](https://leetcode.com/problems/longest-palindromic-substring/solution/)，我们主要来讨论最后两种：EAC（Expand Around Center）和MA（Manacher's Algorithm）。

## EAC
先说EAC，根据Palndromic字符串中心对称的特点，我们可以尝试对源字符串中的每一个元素，使用两个指针向其两侧扩展搜索。当两个指针所指元素不等时，搜索停止。
```
 <-|->
abcdcba
```
这里要注意的是如果目标字符串（最长Palndromic字串）长度为偶数，也就意味着中心是两个相同的元素。这个时候如果还是以一个元素为中心搜索，那么必然会错过这个目标串。为此，我们还需要在单元素为中心搜索的基础上，进行双元素为中心的搜索。
```
 <-||->
abcddcba
```
So，实现的思路基本上有了，顺序遍历源字符串的每一个元素：
1. 以当前元素为中心，向两侧搜索直到元素不一致时停止。
2. 以当前元素加下一个元素为中心，向两侧搜索直到元素不一致时停止。

贴下代码：
```python
def centerAround(self, s):
    if not s or len(s) == 0:
        return ''
    start = 0
    end = 0
    for i in range(len(s)):
        len1, axes1 = self.expand(s, i, i)
        len2, axes2 = self.expand(s, i, i + 1)
        maxLen = max(len1, len2)
        if maxLen > end - start + 1:
            axes = axes1 if len1 > len2 else axes2
            start = axes[0]
            end = axes[1]
    return s[start:end+1]


def expand(self, s, i, j):
    if i < 0 or j >= len(s) or s[i] != s[j]:
        return 0, None
    while i >= 0 and j < len(s) and s[i] == s[j]:
        i -= 1
        j += 1
    i += 1
    j -= 1
    return j - i + 1, [i, j]
```

## 双元素搜索优化
有没有什么方法可以优化双中心搜索呢？加入我们对源字符串 `abcddcba` 见缝插针，插入特殊字符 `#`，变为 `#a#b#c#d#d#c#b#a#`。
可以发现以单元素为中心搜索也能得到最终结果，最后只需要把字符 `#` 删除掉即可。
```
     <--|-->
#a#b#c#d#d#c#b#a#
```
## MA

基于我们上面的分析，接着来讨论MA（Manacher's Algorithm）。 它的核心思想是什么呢？我们先来看一个例子：
```
     k
···cbcbc···
```
我们先使用ECA搜索，假如此时正好以 `k` 位置为中心搜索，可以得到一个局部结果 `cbcbc`。这个就是后面搜索的基础窗口。

接着使用ECA从下一个位置 `s[k+1]=b` 继续搜索。但这个时候观察可以发现，`k+1` 位置正好在刚才的搜索结果 `cbcbc` 中。这一点很重要，后面还会提到。

因为Palindromic的对称特性，`k+1` 位置在 `cbcbc` 上的对称位置为 `2k-(k+1)`，且 `s[k+1] = s[2k-(k+1)]`。
```
      t'  t 
··· c b c b c ···
```
注意此时 `s[t'-1]至s[t'+1] = s[t-1]至s[t+1]`。那么我们对 `t` 进行搜索时，就可以得到一个基础的搜索中心为 `t-2`和`t+2`，而不用在 `t` 位置上开始搜索。
```
   <-- t-2      t      t+2 -->
··· c   b   c   b   c   x ···
```
这个 `2` 怎么得出的呢？因为 `t'` 会先于 `t` 进行搜索，这个时候我们可以将 `t'` 位置上的搜索广度保存起来。当搜索 `t` 时，先不着急搜索。而是先计算出对称位置，拿出它的搜索广度，并在此基础上再进行搜索。

此例中 `t'` 的搜索广度为 `2`，包括当前 `t'` 位置字符。那么在 `t` 位置上搜索时，就可以跳过两个字符，从而减少搜索次数。这便是MA相较于ECA的核心优化点，它充分利用了之前搜索的知识。

那么看到这，可能会有几个疑问冒出来了：
1. `s[t'-1]至s[t'+1]` 有可能不等于 `s[t-1]至s[t+1]` 呀？
    
    确实，比如 `cbcbc` 换成 `abcba`。但是相同的道理，此时 `t'` 的搜索广度为 `1`，并不影响上述的分析。

2. 如果 `t` 加上对称位置的搜索广度超过当前窗口怎么办呢？

    这个的确会发生，比如：
    ```
        l   t'   k    t   r 
    ··· a b c b  c  b c x y ···
    ```
    - 当遍历到 `t'` 搜索时，会将窗口更新为 `bcb`，其搜索广度为 `2`。
    - 当遍历到 `k` 时，由于超过了当前窗口 `bcb`，故将当前窗口更新为 `cbcbc`。
    - 当遍历到 `t` 时，其正好位于当前窗口的边界，而其对称位置 `t'` 的搜索广度为 `2`。那么加上这个广度后，就要从 `l`和`r` 开始搜索了，当然结果也是错误的。所以当超出时，我们要取广度和 `t` 位置距当前窗口的距离的较小的一方。

3. 窗口什么时候更新呢？

    有两种情况：
    - 当当前遍历位置超过当前窗口时，当前窗口也就没有用了。我们可以先把当前元素作为当前窗口。
    - 当当前搜索结果大于当前窗口时，更长的结果意味着在新的窗口中，下一个遍历位置的最小搜索广度可能会增大。因为我们取较小值时，其中的遍历位置到窗口边界的距离增大了。意味着我们得到的最小搜索广度可能会变大。

4. 那如果窗口为偶数呢，即中心元素为两个。同样我们使用插入 `#` 的方法来避免这个问题。

ok，根据分析，我们来描述下算法的步骤：
1. 对源字符串插入 `#`。
2. 初始化当前窗口的中心位置和边界为0，所有广度为0。
3. 顺序遍历源字符串的每一个字符，索引为 `i`：
    - 当 `i` 大于当前窗口时，根据ECA进行搜索。
        - 根据搜索结果更新当前窗口。
        - 保存当前搜索广度到 `p[i]`。
    - 当 `i` 位于当前窗口中时，求 `minExpand = min(i在当前窗口对称位置的搜索广度，i到窗口边界的距离`。
        - 从 `i+minExpand` 开始向两边搜索，直到两边字符不等时结束。
        - 保存当前搜索广度到 `p[i]`。
4. 根据广度数组 `p` 即可求出最大的Palindromic字符串。当然也可以放在循环中进行更新。
    
> 关于广度值的描述中，我是包括了当前元素。实现中也可以不用包含，都可以。

代码如下：
```python
def manacherAlgorithm(self, s):
    if not s or len(s) == 0:
        return ''
    # 插入特殊字符
    s = '#' + '#'.join(s) + '#'
    # 循环中更新最长Palindromic字符串位置
    start = 0
    end = 0
    # 当前窗口中心
    c = 0
    # 当前窗口边界
    l = 0
    r = 0
    # 广度数组
    p = [0] * len(s)
    for i in range(len(s)):
        # 超过当前窗口后，暂时以当前元素替代为当前窗口
        if i > r:
            c = i
            l = i
            r = i
        # 计算最小搜索广度，当超出边界后，将较小值更新为当前元素的搜索广度
        mirror = 2 * c - i
        if mirror - p[mirror] < l:
            p[i] = r - i
        else:
            p[i] = p[mirror]

        # 以最小搜索中心为中心向左右搜索，通过p[i]自增进行搜索
        while i - p[i] - 1 >= 0 and i + p[i] + 1 < len(s) and s[i - p[i] - 1] == s[i + p[i] + 1]:
            p[i] += 1
        
        # 当前搜索结果超出当前窗口，更新当前窗口
        if i - p[i] < l or i + p[i] > r:
            c = i
            l = i - p[i]
            r = i + p[i]

        # 保存最长搜索结果
        if r - l + 1 > end - start + 1:
            start = l
            end = r
    
    return s[start: end + 1].replace('#', '')
```

## 总结

我觉得关于MA算法只要搞清楚他的核心思想，分析下边界条件，还是不难写的。这Manacher观察能力好强，我怎么就看不出来这个优化点呢？
