---
title: LFU Implementation With O(1) Complexity (2010)
date: 2020-02-17 18:27:16
tags:
- data structures & algorithms
categories:
- papers-we-love
mathjax:
- true
---

# Abstract

缓存置换算法 (Cache Eviction Algorithm) 在操作系统、数据库以及其它系统中被广泛用于缓存置换模块，当缓存空间不足时，它利用局部性原理 (Principle of Locality) 预测未来数据的使用模式，将最不可能被访问的数据清出从而提高缓存命中率。目前已经存在的缓存置换算法包括 MRU (Most Recently Used)、MFU (Most Frequently Used)、LRU (Least Recently Used) 以及 LFU (Least Frequently Used) 等。每个算法都有其各自的适用场景。到目前为止，应用范围最广的是 LRU，主要原因在于 LRU 贴近大多数应用的实际负载模式 (workloads)，同时 LRU 拥有 $O(1)$ 时间复杂度的成熟实现方案。与 LRU 类似，LFU 同样与大多数应用的负载模式相近，但目前 LFU 最佳实现方案的时间复杂度是$O(log_2{n})$ ，不如 LRU。本文，我们提出一种同样达到 $O(1)$ 时间复杂度的 LFU 实现方案，它支持的操作包括插入、访问以及删除。

<!-- more -->

# Introduction

本文将按顺序介绍：

- LFU 的典型使用场景
- LFU 的接口说明
- 目前 LFU 的最佳实现方案
- 时间复杂度为$O(1)$ 的 LFU 实现方案

# Uses of LFU

LFU 的一个典型使用场景就是 HTTP 的缓存代理应用 (caching network proxy application)。它位于网络服务与用户之间，如下图所示：

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LtT8ewAV2T9BqtnuARR%2F-LtTA_ECgl4tFvFiT3Ct%2FScreen%20Shot%202019-11-12%20at%201.55.13%20PM.jpg?alt=media&amp;token=52931771-0cc6-44b4-a76a-f9bade6ff75a" style="zoom:50%;" />

它通过将大多数用户可能请求的静态文件放入缓存中，来优化网络利用率，提高服务的响应速度。这种缓存代理需要满足：

1. 在有限的存储资源中缓存尽可能多的、更可能被重复使用的数据
2. 实现的成本应该尽可能小，保证代理在高负荷下也能正常工作

当缓存空间占满时，我们需要尽可能地将那些使用频率最低的资源清理出去 (eviction)，这恰好符合 LFU 的定义。LRU 也许也是一个不错的选择，但在特定场景下，如轮流地访问 (round-robin fashion) 一连串不同的数据导致缓存不断地被置换，LRU 将可能失效。

# Dictionary Operations Supported by An LFU Cache

LFU 的接口如下所示：

```
type LFU interface {
    // 放入键值数据
    Set(k string, v interface{})
    // 用键查值
    Get(k string) (v interface{}, ok bool)
    // 清出使用频率最低的键值对
    EvictOne(n int)
}
```

# The Currently Best Known Complexity of the LFU algorithm

在此论文发布之时，当前的 LFU 最佳实现方案满足：

- Set: $O(logn)$ 
- Get: $O(logn)$ 
- EvictOne: $O(logn)$ 

它通过 binomial heap 和 hash table 这两个抽象数据类型 (Abstract Data Type, ADT) 组合实现，它们的具体实现可以是 min-heap 和 hash-map：

- min-heap：以访问次数为值
- hash-map：以缓存的键，如 url 为键，以缓存的数据，如 html 文件为值，建立 hash-map

hash-map 的时间复杂度可以看作是 $O(1)$ ，因此整个 LFU 算法的复杂度受制于 min-heap 的时间复杂度。下面我们逐个分析：

- Set：
  - 放入 hash-map 中
  - 记录使用频次为 1，插入到 min-heap 中
- Get：
  - 从 hash-map 中取数据
  - 将 min-heap 中它的使用频次自增 1，然后调整 min heap，使其恢复不变性
- Eviction：
  - 从 min-heap 顶部取出访问频次最小的数据，并调整 min heap，使其恢复不变性
  - 从 hash-map 中删除相应键值数据

# The Proposed LFU Algorithm

本文提出的 LFU 通过一个两级 Doubly Linked List 和一个 Hash Map 来实现，其基本结构如下图所示：

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-Lv0zTTOwqyAvtbMELtY%2F-Lv0zi3zAaMYdZJA-uIy%2FScreen%20Shot%202019-12-01%20at%2011.05.16%20PM.jpg?alt=media&amp;token=d46c31d9-1d6d-4b95-9a24-aa0a6667d4cd" style="zoom:50%;" />

我们将两级 Doubly Linked List 分为横向和纵向，具体说明如下：

- head 与横向 Doubly Linked List 相连，后者的节点由近及远地、从低到高地存储访问频次数据
- 横向 Doubly Linked List 中的每个节点都指向一个纵向 Doubly Linked List
- 纵向 Doubly Linked List 中的节点存储的是出现频次相同的元素，且每个节点都维护着一个指向其对应的横向 Doubly Linked List 节点的指针。

以上图为例：

- 元素 x、y 都只被访问 1 次
- 元素 z、a 都只被访问 2 次
- 元素 b 被访问 5 次
- 元素 c 被访问 9 次
- 通过键 kx 可以找到位于频度为 1 的纵向 Doubly Linked List 的节点 x，而通过 x 又能访问其对应的横向 Doubly Linked List 的节点 1

下面我们分析其 3 个核心 API 的算法复杂度。

## Set

```go
func (c *cache) Set(k string, v interface{}) {
    // 如果没有频率为 1 的纵向 Doubly Linked List，则新建一个，并插入 kv 数据
    // 如果有，直接将 kv 数据插入其中
    var item *kvItem
    if c.horizontalDDL.Len() == 0 || c.horizontalDDL.first.freq != 1 {
        node := &DDLNode{
            freq: 1,
            verticalDDL: NewDDL(),
        }
        
        item = &kvItem{
            k: k,
            v: v,
            parent: node,
        }
        node.verticalDDL.PushFront(item)
        c.horizontalDDL.PushFront(node)
    } else {
        item = &kvItem{
            k: k,
            v: v,
            parent: c.horizontalDDL.first,
        }
        c.horizontalDDL.first.vertialDDL.Push(item)
    }
    
    // 插入 kv 到 Hash Map 中
    c.hashMap[k] = item
    return
}
```

由于 Set 操作只涉及频度为 1 的纵向 Doubly Linked List，因此时间复杂度为 $O(1)$ 。

## Get

```go
func (c *cache) Get(k string) (vv interface{}, ok bool) {
    // 如果 k 不在 Hash Map 中，则返回
    v, ok := c.kv[k]
    if !ok {
        return
    }
    
    // 如果 k 在，则找到其对应的纵向 Doubly Linked List 节点 vNodeK
    // 假设 vNodek 对应的横向 Doubly Linked List 节点 hNodeK 的频率为 freq
    // 将 vNodek 移动到 freq + 1 的纵向 Doubly Linked List 上即可
    vNodeK = v
    hNodeK = v.parent
    
    if hNodeK.next.freq != hNodeK.freq+1 {
        node := &DDLNode{
            freq: hNodeK.freq+1,
            verticalDDL: NewDDL(),
        }
        hNodeK.verticalDDL.Remove(vNodeK)
        node.verticalDDL.PushFront(node)
        c.horizontalDDL.InsertAfter(hNodeK, node)
    } else {
        hNodeK.verticalDDL.Remove(vNodeK)
        hNodeK.next.vertialDDL.PushFront(vNodeK)
    }
    
    return
}
```

Hash Map 访问的时间复杂度为 $O(1)$ ，更新 Doubly Linked List 只涉及相邻的两个横向节点及其对应的纵向 Doubly Linked List，因此时间复杂度 $O(1)$。因此 Get 的时间复杂度为 $O(1)$。

## EvictOne 

```go
func (c *cache) EvictOne() {
    // 取出频度最小的纵向 Doubly Linked List
    front := c.horizontalDDL.first.vertialDDL.RemoveFront()
    // 从 Hash Map 中删除其对应的键值对
    delete(c.kv, front.k)
    return
}
```

EvictOne 时，只需要从频度最小的纵向 Doubly Linked List 上删除任意元素即可，其时间复杂度为 $O(1)$；Hash Map 删除的时间复杂度为 $O(1)$，因此 EvictOne 的时间复杂度为 $O(1)$。

# 参考

- [An O(1) algorithm for implementing the LFU cache eviction scheme](https://github.com/papers-we-love/papers-we-love/blob/master/caching/a-constant-algorithm-for-implementing-the-lfu-cache-eviction-scheme.pdf)
- [Github: ZhengHe-MD/lfu](https://github.com/ZhengHe-MD/lfu)