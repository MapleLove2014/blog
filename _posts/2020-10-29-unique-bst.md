---
title: Unique Binary Search Trees
author: ZhangMapler
date: 2020-10-29 10:33:00 +0800
categories: [LeetCode]
tags: [python, algorithm, dp]
math: false
---

# Unique Binary Search Trees 问题[Medium]

## 问题描述

这个问题是给一个整数n，然后将`[1, n]`共n个数构建成一棵二叉搜索树（Binary Search Tree，BST）。求所有的不同的BST。

## 问题分析

解题前先回顾下BST的递归定义：根结点的左右子树都是BST，而且左子树<根结点<右子树。
BST的递归查找过程：从根结点查找，如果是根结点，则查找成功。否则查找值和根结点值比较后，再去左BST或右BST进行查找。
BST的插入过程：先进行查找，如果查找未命中。则在未命中的位置创建新的结点，并返回给父结点进行链接。

OK，回到这个问题。我们最终构建的BST的每个结点都有可能会出现1-n中的一个数，再结合BST的特性。我们可以给出该问题的递归动态规划解法：
1. 对于 `[1, n]` 中的每一个数 `i` 构建一个根节点 `root`;
2. 对于 `[1, i-1]` 从步骤1开始递归求解此问题，返回所有构建的 `Left BST, LBST`;
3. 对于 `[i+1, n]` 从步骤1开始递归求解此问题，返回所有构建的 `Right BST, RBST`;
4. 最后，将步骤1创建的节点分别连接LBST和RBST。

需要注意的是当求解的子问题是从i到j时：
1. 如果 `j > i`，说明此时没有满足的数可以构成父节点的LBST或者RBST，此时返回NULL。
2. 如果 `j = i`, 说明此时要构建的子树只有一个节点，这个节点的值就是 `j`。

另外如果根结点求解的LBST和RBST都为空，说明当前构建的树只有一个根节点，这一点和 `2` 是一致.

## 实现

这里会给出两个问题的实现，分别是求数量和BST。

求数量时，简单一点。一个根节点为`i`的所有BST数量为 `LBST × RBST`。那么最终结果就是这些根节点BST数量的累加。
> 注意当LBST和RBST都为0时，也是一个BST，只不过这个BST只包含根节点。实现上采用返回1的方式。
为了避免重复计算，引入一个回忆簿 `mem[i][j]`，表示数`[i, j]`所能构成的BST数量。

```python
class Solution:
    def numTrees(self, n):
        if n <= 0:
            return 0
        mem = [[0] * (n+2) for _ in range(n+2)]
        return self.search((1, n), mem)

    def search(self, nrange, mem):
        n, x = nrange
        if n >= x:
            return 1
        if mem[n][x] != 0:
            return mem[n][x]
        count = 0
        for i in range(n, x+1):
            root = TreeNode(i)
            lefts = self.search((n, i-1), mem)
            rights = self.search((i+1, x), mem)
            if lefts <= 0 or rights <= 0:
                continue
            count += lefts * rights
        mem[n][x] = count
        return count
```

接着我们继续求出所有的BST，这里就要注意了。每当我们求出一个BST时，要把这个BST复制出来，不能直接保存这个BST。

因为根据计算完当前的BST后，还会回溯回上一级进行尝试其他的BST，这样就把之前保存的BST修改了。

```python
class Solution:
    def generateTrees(self, n):
        if n <= 0:
            return []
        return self.search((1, n))

    def search(self, nrange):
        n, x = nrange
        if n > x:
            return [None]
        if n == x:
            return [TreeNode(n)]
        result = []
        for i in range(n, x+1):
            root = TreeNode(i)
            lefts = self.search((n, i-1))
            rights = self.search((i+1, x))
            if not lefts or not rights:
                continue
            for left in lefts:
                for right in rights:
                    root.left = left
                    root.right = right
                    result.append(self.copyTree(root))
        return result
            
    def copyTree(self, root):
        if not root:
            return root
        cp = TreeNode(root.val)
        cp.left = self.copyTree(root.left)
        cp.right = self.copyTree(root.right)
        return cp
```

## 总结

写这道题前，一定要先复习下BST的定义，以及查找和插入的过程。

然后根据BST的递归定义，会马上想到求解该问题的递归方法。那么只需要按照同样的方法求根节点的左右子树即可。最后根节点分别拼接左右子树就行了。

最后要特别注意边界条件，求所有的BST时，要注意拷贝。

