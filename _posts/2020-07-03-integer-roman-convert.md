---
title: Integer和Roman相互转换解法分析
author: ZhangMapler
date: 2020-07-03 11:15:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Integer和Roman相互转换问题[Medium]

## 问题描述

整数和罗马字符串两种表示方式相互转换： [Roman To Integer](https://leetcode.com/problems/roman-to-integer), [Integer To Roman](https://leetcode.com/problems/integer-to-roman) .

## 问题分析

这种转换显然需要使用一张映射表

| 罗马符号 	| 数值 	|
|-	|-	|
|I  	|1  	|
|IV  	|4  	|
|V  	|5  	|
|IX  	|9  	|
|X  	|10  	|
|XL  	|40  	|
|L  	|50 	|
|XC  	|90  	|
|C  	|100  	|
|CD  	|400  	|
|D  	|500  	|
|CM  	|900  	|
|M  	|1000  	|

Roman转到Integer时，只需要遍历字符串，去除映射的值，累加到结果上。Integer转Roman时，过程正好相反。

## Roman To Integer

遍历字符串，先尝试匹配两个罗马字符，对应 `IV,IX,...`。如果不存在，匹配单个字符。将映射的数值累加到结果上即可。

### 代码实现

```python
def romanToInteger(self, s: str) -> int:
    romans = {
        'I': 1,
        'V': 5,
        'IV': 4,
        'X': 10,
        'IX' : 9,
        'L': 50,
        'XL': 40,
        'C': 100,
        'XC': 90,
        'D': 500,
        'CD': 400,
        'M': 1000,
        'CM': 900
    }
    if not s:
        return 0
    i = 0
    result = 0
    while i < len(s):
        if i + 1 < len(s) and s[i:i+2] in romans:
            result += romans[s[i:i+2]]
            i += 2
        else:
            result += romans[s[i]]
            i += 1

    return result
```

### 复杂度分析

假设罗马字符串的长度为 `n` ，程序顺序遍历该字符串，时间复杂度为 `O(n)`

## Integer To Roman

反转映射表，将整数值作为key，罗马字符作为value。将所有的key（整数值）顺序压入堆栈 `S`，初始 `sValue > num`，在每次循环中

> 一定要 `顺序` 压入堆栈哦，保证从最大的值开始减

1. 如果 `sValue > num` 说明当前数值已经不匹配当前的罗马字符串所表示的最小值，继续弹出 `sValue`，直到满足 `sValue <= num`

2. 否则，根据 `key = sValue` 从映射表中取出对应的罗马字符，append到result中

### 代码实现

```python
def intToRoman(self, num: int) -> str:
    romans ={ v:k for k,v in {
        'I':  1, 
        'V':  5, 
        'IV': 4, 
        'X':  10, 
        'IX': 9, 
        'L':  50, 
        'XL': 40, 
        'C':  100, 
        'XC': 90, 
        'D':  500, 
        'CD': 400, 
        'M':  1000, 
        'CM': 900 

    }.items()}
    # 顺序压入堆栈
    values = list(sorted(romans.keys()))
    if num == 0:
        return ''
    result = []
    value = 4000
    while num > 0:
        if num - value >= 0:
            result.append(romans[value])
            num -= value
        else:
            value = values.pop()

    return ''.join(result)

```

### 复杂度分析

排序那里先忽略，因为也可以直接硬编码顺序赋值映射表和堆栈。假设 `num` 对应的罗马字符串长度为 `n`，复杂度为 `O(n)`.

> 至于这个 `n` 怎么算，我也没太大兴趣去研究了，反正对本题也没啥影响

其实还有一种方法，就是在每弹出一次 `value` 时，在满足 `value <= num` 的情况下直接 `num / value` 求出可以减几次，然后将对应的罗马字符重复这个次数后append到结果中。其实两者本质上差不太多，因为`除` 操作在cpu中最简单的实现就是 `循环减` 。

## 总结

题目不是很难，感觉扩充了下自己关于罗马数字的知识，挺好的。