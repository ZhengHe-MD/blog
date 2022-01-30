---
title: Dynamic Hash Tables (1988)
date: 2020-02-18 12:36:49
tags:
- data structures & algorithms
categories:
- papers-we-love
mathjax:
- true
---

# 摘要

Linear Hashing 和 Spiral Storage 是两种动态哈希算法。这两种算法最初都是为了优化外部存储 (secondary/external storage) 数据访问而设计的。本文将这两种算法引入到内存中，即键值数据可以一次性读入内存的场景，对比、分析二者之间，以及与其它动态哈希算法的性能。实验结果表明：Linear Hashing 的性能上要优于 Spiral Storage，实现难度上要小于 Spiral Storage。其它纳入对比范围的动态哈希算法包括 Unbalanced Binary Tree 以及支持周期性 rehashing 版本的 Double Hashing。Linear Hashing 的查询时间与 Double Hashing 相当，同时远远优于 Unbalanced Binary Tree，即使在 tree 很小的场景上也如此；在载入键值数据的表现上，三者相当。总体而言，Linear Hashing 是一个简单、高效的动态哈希算法，非常适用于键空间未知的场景。

<!-- more -->

# 简介

许多为外部文件存储而设计的动态哈希算法在过去的若干年中被提出，这些算法允许外部文件根据内部存储的纪录数量而优雅地扩大和缩小。在外部文件存储场景中，外部存储比内存读写慢很多，它的特点总结如下：

- 按块读写数据，如 4096 字节为 1 块
- 倾向于读写连续块
- 倾向于批量读写
- 在每一层都设有缓存来优化性能
- 根据磁盘中数据块的读写次数来衡量程序的性能

Linear Hashing 和 Spiral Storage 在上述场景中都已被验证有效。本文将介绍如何将二者引入到 Internal Hash Table 场景中，在数学上分析其预期性能，并通过实验验证预期。从实验结论上看，两种方法对于 Internal Hash Table 、键 (key) 空间未知的场景同样有效。其中 Linear Hashing 更快且更容易实现。

所有 Hashing 技术都有一个特点：当负载提高时，基本操作的复杂度，如插入、查询、删除，也将提高。如果希望 Hash Table 的性能维持在一个可接受的下限之上，我们必须通过某种方式为其分配新的空间。传统的方案是创建一个新的、更大的哈希表，然后将所有数据重新哈希到新的哈希表上。动态哈希算法的不同之处在于，它允许哈希表逐渐扩大，每次增加一个桶。当新桶加入到寻址空间时，只需要重新哈希原来的一个桶即可，而不需要将所有数据全部重新哈希。

# Linear Hashing

通常在 Hash 算法中需要一个 Hash Function，将输入参数转化成一个整数：

$$\begin{equation}
f: x \to h(x)
\end{equation}$$

Linear Hashing 的核心思想在于不使用 $h(x)$ 中的所有 bit 信息。当表较小时，使用较少的 bit 信息；当表较大时，使用较多的 bit 信息。在下文中我们假设：

- $n$ ：桶的数量
- $i$ ：使用的 bit 位数
- $r$ ：记录的数量

Linear Hashing 将使用 $h(x)$ 的后 $log_2(n)$ 位的数据作为实际使用的值。举例如下，假设两个键 ：wizard 和 witch 作为 $x$ 输入到 Hash Function 中得到：

$$\begin{equation}
h(wizard) \to 1100_2 , h(witch) \to 1111_2
\end{equation}$$

若此时 n=2n = 2n=2 ，那么最终 wizard 的 Hash 值为 0，witch 的为 1。如下图所示：

![img](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LsnxfPgCUDno7musJkV%2F-Lso-fwy8iHiTgqoL_iQ%2FScreen%20Shot%202019-11-04%20at%209.23.37%20AM.jpg?alt=media&token=acc77407-f8f8-4b0b-b56d-ac60fe476f49)

当表的负载到达一定程度后，需要增加新桶时，则将桶的数量 $n$ 加 1；如果 $n > 2^i$ ，则将使用的 bit 位数 $i$ 加 1。因此确定某个 Hash 值将对应到哪个桶的过程如下：



```
func bucket(hash uint) uint {
    m := hash & ((1<<i)-1)
    if m < n {
        return m
    } else {
        return m ^ (1<<(i-1))
    }
}
```

增加新桶的具体图形化实例请看论文原文。

# Spiral Storage

在使用 Linear Hashing 时，查询、插入、删除的预期成本存在抖动。假如 Linear Hashing 使用的 Hash Function 能够均匀地将键散列到不同的桶上，那么发生分裂时，当前分裂的桶、新桶以及其它桶内的数据将出现不均匀的现象。如果我们能够刻意在散列分布上制造一些不均匀，使得即将分裂的桶是数据最多的桶，那么就有可能减少操作的预期成本。

基本思想如下图所示：

![img](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LtN0Uy9ZfWzBYiJkB1x%2F-LtN3YI4_lsAW3H24CQk%2FScreen%20Shot%202019-11-11%20at%209.26.44%20AM.jpg?alt=media&token=df6f7958-36d6-4c39-8541-15888e8a56ff)

> 将数据更多地散列到位置靠前的桶内，分裂时直接将最前面的桶中的数据分裂到两个新的桶上。

一个典型的分布例子如下所示：

![img](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LtN0Uy9ZfWzBYiJkB1x%2F-LtN48if5IZlq-P5cETf%2FScreen%20Shot%202019-11-11%20at%209.29.28%20AM.jpg?alt=media&token=1630f43d-3d56-4c84-9a57-2413ceb94df6)

# 其它

## Linear Hashing vs. Consistent Hashing

Linear Hashing 允许桶顺序地往后添加、往前删除，而 Consistent Hashing 允许按任意顺序添加和删除桶。二者的使用场景并不相同。

# 参考

- papers
  - [Linear hashing: A new tool for file and table addressing](https://pdfs.semanticscholar.org/907f/7ca06059a53e8c8bb40d3a5329340e838a1b.pdf?_ga=2.231818847.194162924.1572794970-185720434.1566181423)
  - [Spiral Storage: Incrementally augmentable hash addressed storage](http://wrap.warwick.ac.uk/46327/1/WRAP_Martin_cs-rr-027.pdf)
- blogs
  - [Hackthology: Linear Hashing](https://hackthology.com/linear-hashing.html)
  - [Hackthology: An in Memory Go Implementation of Linear Hashing](https://hackthology.com/an-in-memory-go-implementation-of-linear-hashing.html)
- web
  - [Hashing Tutorial](http://research.cs.vt.edu/AVresearch/hashing/)
  - [Wikipedia: Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)
- implementations
  - [timtadh: linhash](https://github.com/timtadh/file-structures/tree/master/linhash)