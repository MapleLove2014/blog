---
title: Reverse-Integer解法和分析
author: ZhangMapler
date: 2020-06-23 11:58:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: true
---

# Reverse-Integer问题[Medium]

## 问题描述

将一个有符号的整数反转后返回，溢出时返回0，[官网具体描述点这里](https://leetcode.com/problems/reverse-integer/) .

> 这道题个人觉得还是挺简单的，除了做题，正好可以回顾下计算机组成原理中计算机存储有符号数相关的知识

## 计算机中的有符号数

> 大学的教材找不到了，这里引用 `Linda Null,Julia Lobur: The essentials of computer organization and architecture` 来说明

先看下这本书里面提到的一个自行车仪表盘的例子：

> 仪表盘里程范围为 `[0-900]`, 起始位置为 `0`，前进时指针前拨，后退时指针后拨

1. 当指针处在 `700` 时，从这个仪表盘上是不能得出自行车到底前进了 `700` 还是倒退了 `300`

    ![仪表盘1]({{ "/assets/img/reverse-integer/仪表盘1.png"}})

2. 为了解决这个问题，可以通过将里程范围分为两部分 `[0-500]`为前进，`[501-900]`为后退。
    
    ![仪表盘2]({{ "/assets/img/reverse-integer/仪表盘2.png"}})
    
    当指针在700时，落在后退区间，后退了 `900-700` 公里，或者说前进了 `-(700 - 900)` 公里
    
    当指针在450时，落在前进区间，前进了 `450` 公里

> 计算机中的有符号数基于补码(也叫 `2's complement` )，这个例子中的后退区间就相当于基于补码的负数表示

我们以一个字节8-bit的有符号数的表示为例：

1. 符号位位于最高位（书中给了3个名称，leftmost bit，high-order bit or the most significant bit），0表示正数，1表示负数

2. 数值表示范围为 \\( -2^7 \to 2^7-1 \\)

    > 借用一下Oliver Charlesworth的[回答](https://stackoverflow.com/a/11433824/6147582)
    ```
    01111111 = +127
    01111110 = +126
    01111101 = +125
    ...
    00000001 = +1
    00000000 =  0
    11111111 = -1
    ...
    10000010 = -126
    10000001 = -127
    10000000 = -128
    ```

    用仪表盘表示下这个数值范围，结合上文的自行车仪表盘是不是有种视觉化的感受，而非冰冷的数字

    ![有符号数范围仪表盘]({{ "/assets/img/reverse-integer/有符号数范围仪表盘.png"}})

    > 在大学学习的时候和做这道leetcode题目时，我还是会有点疑惑，就是Oliver Charlesworth回答的问题，为什么负数范围多个一个 `-128` 呢？

    我为什么会有这个问题，主要是我一上来忘记了补码，而是认为符号位表示符号，数据位还是按照无符号数的表示，最后 `符号+数值` 构成一个有符号数。如果这样的话，`1 111 1111 = -127` , `1 000 0000 = -0`. 这样是错误的，因为
    
    1. 用补码不仅是考虑到符号位
    
    2. 还有一个原因就是可以很方便的使用 `加` 来实现 `减`，统一和简化cpu计算单元的硬件设计

    > 至于计算机怎么执行有符号数的 `加` 操作，先不提了，要不然扯太远了

## 简单实现

首先看一下base case

1. 当输入数只有个位数时,即 \\( \lvert x\rvert < 10 \\)，直接返回

2. 当结果溢出时，返回 `0`

实现思路

1. 生成输入数的绝对值的字符串

2. 对字符串反转

3. 对反转后的字符串转换为integer

4. 加上符号

### 实现代码：

```python
def reverse(self, x: int) -> int:
    if abs(x) < 10:
        return x
    # 生成反转后的integer绝对值
    unsignedX = int(str(abs(x))[::-1])
    # 加上符号
    result = unsignedX if x >= 0 else unsignedX * -1
    # 判断是否溢出
    if result > (1 << 31) - 1 or result < -(1 << 31):
        return 0
    return result
```

看一下leetcode提交结果，效率还可以，中等偏上

![submit1]({{ "/assets/img/reverse-integer/submit1.png"}})


### 复杂度分析

输入数最多 `n=32` 位
1. 生成字符串 \\( O(n) \\)

2. 字符串反转 \\( O(n) \\)

3. 字符串转为integer \\( O(n) \\)

时间复杂度为 \\( O(n) \\)

字符串使用 `32-bit` 空间，空间复杂度为 \\( O(1) \\)

## LeetCode Solution

leetcode官网的[solution](https://leetcode.com/problems/reverse-integer/solution/)，的意思是

1. 从个位起遍历输入数的每一位

2. 每次循环，对十进制结果左移一位，并加上遍历位

3. 每次循环，将遍历后的位删除

4. 每次循环检查结果是否溢出

### 实现代码

```python
def reverse2(self, x: int) -> int:       
    if abs(x) < 10:
        return x
    result = 0
    modNumber = 10
    absX = abs(x)
    while absX > 0:
        # 左移一位 + 遍历位
        result = result * 10 + (absX % modNumber)
        # 删除遍历位
        absX //= modNumber
        # 判断是否溢出
        if result > (1 << 31) - 1 or result < -(1 << 31):
            return 0
    # 加符号返回
    return result if x >= 0 else -1 * result
```

> absX 每次循环都会删除遍历位，直至删除所有位为0，结束循环

leetcode提交结果，结果上时间少了4ms，差不太多

![submit2]({{ "/assets/img/reverse-integer/submit2.png"}})

### 复杂度分析

时间复杂度为 \\( O(n) \\)，空间复杂度为 \\( O(1) \\)， 其中 `n` 表示输入数的位数

## 实现对比

从提交结果上也可以看出两种实现表现一致，就是由输入数的位数决定的。可能上文的实现多了几次循环，稍微慢了一丁点点点点点点点，但本质上没啥区别。

如果从工程的选择来看，选择第一种实现。原因有2：

1. 实现简洁，可读性和维护性好

2. 可以很灵活的基于字符串适应超大范围数值的反转

## 总结

做这道题的最大收获是回顾了下有符号数的一些知识点，纠正了个人的一些错误认识。

    