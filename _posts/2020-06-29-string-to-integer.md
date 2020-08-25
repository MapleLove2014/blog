---
title: String-To-Integer解法和代码重构
author: ZhangMapler
date: 2020-06-29 11:32:00 +0800
categories: [LeetCode]
tags: [python, algorithm, refactor]
math: false
---

# String-To-Integer问题[Medium]

## 问题描述

将一个字符串转换为整数，注意溢出，前空格等case，[官网具体描述点这里](https://leetcode.com/problems/string-to-integer-atoi/) .

## 实现

实现思路：顺寻遍历源字符串，并作如下处理

1. 跳过串头空格

2. 将空格后可能包含正负号的整数字符串转换为整数

3. 忽略掉整数后面的非数字字符串

4. 返回整数，注意加上正负号

### 实现代码：

```python
def myAtoi(self, s: str) -> int:
    if not s:
        return 0
    # 遍历索引
    startIndex = 0
    # 跳过空格
    while startIndex < len(s):
        if s[startIndex] == ' ':
            startIndex += 1
        else:
            break
    if startIndex == len(s):
        return 0
    sign = {'-': -1, '+':1}
    numbers = { str(x) : x for x in range(10) }
    
    # 获取正负号
    signNumber = 1
    if s[startIndex] in sign:
        signNumber = sign[s[startIndex]]
        startIndex += 1
    if startIndex == len(s) or s[startIndex] not in numbers:
        return 0
    result = 0
    
    # 转换
    while startIndex < len(s):
        if s[startIndex] not in numbers:
            break
        result = result * 10 + numbers[s[startIndex]]

        # 越界判断
        if result * signNumber > (1 << 31 ) - 1:
            return (1 << 31) - 1
        elif result * signNumber < -(1 << 31):
            return -( 1 << 31)
        startIndex += 1

    return result * signNumber
```

> 遍历过程中，各种边界要特别注意，数组越界，正负号位置，等等


## 重构

上面的实现代码，看上去真的很难受，问题主要有以下几点：

1. `myAtoi` 函数过长

2. `缩进` 有 `3` 层

    > 虽然比标准的 `4` 层要少，但要尽最大努力减少

3. `startIndex` 变量生命周期太长

    > 变量的生命周期长，其声明和结束横跨的代码逻辑较多，也就意味着变量的变化较多。可读性，维护性都比较差。

4. `局部变量` 略多

5. `注释` 较多

    > 代码注释较多时，往往意味着程序逻辑复杂，业务耦合严重。严格的软件工程理论要求 `Single Responsibility` , 当一个函数 `只做一件事情`，并且 `函数名` 能清晰的表达出 `要做的这件事情` 时，完全可以省略 `注释`，甚至是 `函数注释` 。


重构方法其实很简单


1. 对于 `缩进` 的通用处理方法是尝试对内层缩进 `Extract Method`

2. 对于 `局部变量` 多的问题，可以采用 `Replace Temp With Query`

3. 函数过长时一般要将程序分块，可以根据实际代码逻辑按照 `前后顺序` 或者 `业务逻辑`分块，然后 `Extract Method`。然而做这一步之前一般要先处理 `局部变量`，`内部的逻辑耦合` 等，所以一般来说 `Extract Method` 放到最后做比较合适

4. 仔细分析转换逻辑可以发现，业务逻辑大致可以分为

    - 字符串预处理，包括前置处理和后置处理
    - 字符转换为整数
    - 溢出处理
    - 生成结果返回

    并且可以抽取出 `正负号判断`，`正负号获取`，`数字判断` 等公共方法复用

重构后的代码如下

```python
def afterRefactor(self, s: str) -> int:
    integerStr = self.preprocess(s)
    if not integerStr:
        return 0
    if len(integerStr) == 1 and self.isSign(integerStr[0]):
        return 0
    
    result = 0
    startIndex = 1 if self.isSign(integerStr[0]) else 0
    while startIndex < len(integerStr):
        result = result * 10 + NUMBER_DICT[integerStr[startIndex]]
        overflow, bound = self.boundOverflow(self.getSignNumber(integerStr[0]), result)
        if overflow:
            return bound
        startIndex += 1
    return result * self.getSignNumber(integerStr[0])


def preprocess(self, s):
    return self.processTail(self.processHeader(s))

def processHeader(self, s):
    if not s:
        return ''
    for i in range(len(s)):
        if s[i] != ' ':
            return s[i:]
    return ''

def processTail(self, s):
    if not s:
        return ''
    for i in range(len(s)):
        if i == 0:
            if not self.isSign(s[i]) and not self.isNumber(s[i]):
                return ''
        elif not self.isNumber(s[i]):
            return s[:i]
    return s

def isNumber(self, c):
    return c in NUMBER_DICT

def isSign(self, c):
    return c == '-' or c == '+'

def getSignNumber(self, c):
    return -1 if c == '-' else 1

def boundOverflow(self, signNumber, absoluteValue):
    if self.upperBoundOverflow(signNumber * absoluteValue):
        return (True, (1 << 31 ) - 1)
    if self.lowerBoundOverflow(signNumber * absoluteValue):
        return (True, -(1 << 31))
    return (False, None)

def upperBoundOverflow(self, integer):
    return integer > (1 << 31 ) - 1

def lowerBoundOverflow(self, integer):
    return integer < -(1 << 31)
```

可以明显感觉出重构后的代码

1. 直观的舒服

2. 每个函数都是只负责一件事情

3. 函数长，缩进，局部变量多以及变量生命周期长的问题都不存在

4. 代码完全不需要注释，`afterRefactor` 仅作说明

5. 代码复用性提升

6. 业务逻辑更加清晰，维护性更强

## 总结

很简单的一道题，用来练练重构也不错

    