---
title: 使用 prefix tree 解决 longest common prefix 问题和优化
author: ZhangMapler
date: 2020-07-06 11:20:00 +0800
categories: [LeetCode]
tags: [python, algorithm]
math: false
---

# Longest common prefix 问题[Easy]

## 问题描述

[Longest common prefix](https://leetcode.com/problems/roman-to-integer) ：给定几个字符串，求它们的公共前缀。

## 问题分析

这道题方法很多，水平扫描，垂直扫描，分治，二分搜索等等，但这里只谈一种方法 `Prefix tree`. 原因如下：

1. 数据结构`prefix tree`专为这个问题而生

2. 性能强劲，简单易懂

我们先看下 `prefix tree`

## Prefix tree

> 很多地方叫 `prefix trie`，具体这个 `tree` 和 `trie` 有什么区别呢，感兴趣的可以看下stackoverflow上的[回答](https://stackoverflow.com/a/4737954/6147582)，我这里统一使用 `tree` 来说。

每处理一个字符串时

1. 顺序遍历字符串，把字符串的前缀字符作为根节点，再根据树的递归定义逐层向下扩展

2. 当前字符如果已经在节点中时，进行复用，并继续遍历下一个字符

3. 直到遍历完所有字符串

它的优点很明显：

1. 当字符串的公共前缀较长时，可以节省存储空间

2. 很方便的得到最长公共前缀

    > 只需要从根节点往下遍历，当节点分叉(有多个子节点)时停止，将遍历路径上的字符拼接起来即可

## 代码实现

实现思路

1. 每个`Node`节点保存一个字符，有一个指向`父节点`的指针，和一个保存所有`孩子节点的数组`，为了快速的判断字符是否已经存在在当前节点的子节点中，增加一个 `char-child` 的字典.

    > 显然 `Node` 是可以存多个字符的，比如将所有的 `单链` 上的节点数据拼接保存在一个节点上，这就是另外一个数据结构 `compressed tree`。为了编程的方便，生成 `compressed tree` 可以先用单节点保存数据，然后进行压缩。

2. 顺序遍历每个字符串，从`根节点`开始，判断字符是否存在。如果存在，沿着该节点继续往后遍历，否则添加该字符节点并向后遍历。

    > `根节点`不保存数据，所以每遍历一个`字符`时，从当前`局部根节点`的`子结点`中判断字符是否存在。

需要注意的是：

1. 很显然如果一个字符串为空，说明前缀也是空，程序直接结束

2. 考虑下输入 `[a, aa]`, 构造出的前缀树为：

    ```
    root -> a -> a
    ```
    根据上文的算法，最长的公共前缀为 `aa`，显然是错误的。原因在于长字符串有可能将一些短字符串覆盖掉从而使得程序不知道遍历路径可能已经越过了短字符串，导致错误。解决方法也很简单，每个字符串后拼接一个唯一的标识，如 `[a$0, a$1]`, 后面求LCP时只需要遍历到将 `$` 节点结束即可。这样一改，前缀树就变成了：

    ```
    root -> a -> $ -> 0
             \
              -> a -> $ -> 1
    ```

先看下Node，除了说的几个字段，还添加了几个方便的方法

```python
class Node:
    def __init__(self, parent, char):
        self.parent = parent
        self.char = char
        self.children = []
        self.keys = {}

    def hasSingleChild(self):
        return self.children and len(self.children) == 1

    def hasChild(self, char):
        return char in self.keys
    
    def addChild(self, child):
        if child.char not in self.keys:
            self.children.append(child)
            self.keys[child.char] = child

    def getChild(self, char):
        return self.keys[char]
        
    def getSingleChild(self):
        return self.children[0]
```

构造前缀树并输出最长公共前缀
```python
class Solution:
    def longestCommonPrefix(self, strs: List[str]) -> str:
        tree = self.build(strs)
        if not tree or not tree .hasSingleChild():
            return ''
        result = []
        while tree.hasSingleChild():
            node = tree.getSingleChild()
            if node.char == '$':
                break
            result.append(node.char)
            tree = node
        return ''.join(result)


    def build(self, strs):
        if not strs or len(strs) == 0:
            return None
        root = Node(None, 'root')
        for i in range(len(strs)):
            if not strs[i]:
                return None
            self.buildTree(root, strs[i] + '${}'.format(i))
        return root

    def buildTree(self, root, string, index = 0):
        if not string or index == len(string):
            return
        if root.hasChild(string[index]):
            self.buildTree(root.getChild(string[index]), string, index + 1)
        else:
            node = Node(root, string[index])
            root.addChild(node)
            self.buildTree(node, string ,index + 1)
```

## 复杂度分析

1. 构造前缀树，需要遍历所有字符串的所有字符。复杂度为 `O(S)` ， S为所有的字符个数。

2. 寻找最长公共前缀只需要遍历从根节点开始的单链节点，假设长度为 `m`，则复杂度为 `O(m)`

综上，时间复杂度为 `O(S)`

## 优化

> 下文所提优化仅针对于此最长公共前缀问题

仔细观察求最长公共前缀的方法可以发现，根本不需要构造所有的节点。因为任何一个字符串都会包含这个最长公共前缀。那么刚开始只需要选择一个字符串构造出完全节点，其他的字符串进行构造时，在分叉的节点上就可以结束，因为最后求LCP时，在分叉的节点上也会结束。

在之前的方法中如果求 `["flower","flow","flight"]`, 构造的树为

```
root -> f -> l -> o -> w -> e -> r -> $ -> 0
             \         \
              \         -> $ -> 1
               \
                -> i -> g -> h -> t -> $ -> 2
```

优化后的树如下

```
root -> f -> l -> o -> w -> e -> r -> $ -> 0
```

> flower 完全构造，flow遍历到 `l` 停止，flight遍历到 `l` 停止

当然，我们也可以在构造前对所有的字符串排序`["flow","flower","flight"]`，字符串长度最小的最先构造：

```
root -> f -> l -> o -> w -> $ -> 0
```

既然分叉时不构造，那么怎么判断一个节点是不是分叉节点呢。很简单只需要在 `Node` 节点中增加一个 `count` 字段保存所有的 `distinct child` 的计数即可。如果节点分叉，那么这个值一定大于 `1`.

```
root -> f -> l -> o -> w -> $ -> 0
count:  1    2    1    2   ...
```

我们看下实现代码，先给出 `Node`

```python
class Node:
    def __init__(self, parent, char):
        self.parent = parent
        self.char = char
        self.children = []
        self.keys = {}
        # ----- here -----  
        self.count = 0

    def hasSingleChild(self):
        return self.count == 1

    def containChild(self, char):
        return char in self.keys
    
    def addChild(self, child):
        if child.char not in self.keys:
            self.children.append(child)
            self.keys[child.char] = child
            self.count += 1

    def getChild(self, char):
        return self.keys[char]
        
    def getSingleChild(self):
        return self.children[0]


    def increaseChild(self):
        self.count += 1
        
    def hasChild(self):
        return self.count > 0

    def hasNoChild(self):
        return not self.hasChild()
```

构造树：

```python
class Solution:
    def longestCommonPrefix(self, strs) -> str:
        tree = self.build(strs)
        if not tree or not tree .hasSingleChild():
            return ''
        result = []
        # Note 1
        while tree.hasSingleChild():
            node = tree.getSingleChild()
            if node.char == '$':
                break
            result.append(node.char)
            tree = node
        return ''.join(result)


    def build(self, strs):
        if not strs or len(strs) == 0:
            return None
        root = Node(None, 'root')
        for i in range(len(strs)):
            if not strs[i]:
                return None
            self.buildTree(root, strs[i] + '${}'.format(i))
        return root

    def buildTree(self, root, string, index = 0):
        if not string or index == len(string):
            return

        if root.hasNoChild():
            node = Node(root, string[index])
            root.addChild(node)
            self.buildTree(node, string ,index + 1)
        elif root.containChild(string[index]):
            self.buildTree(root.getChild(string[index]), string, index + 1)
        else:
            # Note 2
            root.increaseChild()

```

> `Note 1`，Node#hasSingleChild方法要用 `count` 来判断，不能在使用 `children` 字段，因为我们忽略了分叉节点的构造，即使是分叉节点这个 `children` 也只包含一个子节点

> `Note 2`，这里说明当前的 `root` 节点有子节点，但不包含当前的字符。这说明此节点是分叉节点，只需要增加计数，结束遍历

## 复杂度分析

Worst Case: 所有字符串长度都相等，长度为 `L`，构造前缀树需要遍历所有的字符，时间复杂度依然为 `O(S + L) = O(S)`

Good Case: 字符串按长度排序后进行构造，假设最短字符串长度为 `minLen`，最长公共前缀长度为 `LCP`，那么时间复杂度为 `O(minLen + LCP) = O(minLen)`

当字符串长度都比较大且公共前缀长度偏小时，可以直观的看出优化的好处。可以很大程度上减少无用节点的构造，并且节省内存。

## 总结

前缀树作为一种很基本的数据结构，在解决LCP问题时，真的是一把利器。同时也要针对使用场景对其进行优化，使其发挥更大的威力。