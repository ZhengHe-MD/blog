---
title: Consistent Hashing and Random Trees (1997)
date: 2020-02-18 17:33:21
tags:
- data structures & algorithms
categories:
- papers-we-love
mathjax:
- true
---

论文作者的贡献主要包含两部分：Consistent Hashing 和 Random Trees。Consistent Hashing 主要用于解决分布式哈希表 (Distributed Hash Table, DHT) 的桶增减带来的重新哈希问题；Random Trees 主要用于分布式缓存中的热点问题，它利用了 Consistent Hashing。下文主要关注 Consistent Hashing。

# Contribution

在分布式环境下，单台机器的负载有限，我们需要将请求散列到不同的机器上，利用更多的机器实现服务的横向扩容。这时候就需要 Hash Function，好的 Hash Function 能够帮我们均匀地分布到不同的机器上。但传统的 Hash Function 通常是静态的，桶的数量固定。在多台机器组成的服务中，每台机器就是一个桶，但机器在运行的过程中很可能出现崩溃，在请求数量波动较大时，需要动态地增减机器。如果每次桶的数量发生变化时都需要重新散列所有请求，可能造成多方面影响：

- 来自于同一个用户的请求在桶发生变化时将被打到不同的节点，可能导致数据不一致 (考虑 monotonic consistency)
- 所有的 Client 都需要知道当前最新的 Hash Function 配置，在网络中传播这个配置需要时间

Consistent Hashing 的提出就是希望能够缓解/解决这个问题，使得每次桶数量发生变化时不需要重新散列桶内的所有元素，而是将受影响的数量控制在很小的范围内。

# Definitions

作者从四个方面讨论了好的 Consistent Hash Function 应该满足的性质：

- Balance：元素应当尽量均匀地分布到不同的桶内 (with high probability)
- Monotonicity：当增加新的桶时，元素只可能从旧桶移动到新桶，而不可能从旧桶移动到其它旧桶
- Spread：在不同的用户眼里，相同的元素可能被散列到不同的桶中，我们称之为不同的观点。Spread 要求总的观点数量必须有一个上限。好的 Consistent Hash Function 应当让 spread 尽量小。
- Load：类似 spread，Load 性质是针对不同的用户，但它规定的是单个桶中不同元素数量的上限，即每个桶中的，至少有一个用户认为其中含有的，元素数量存在上限。好的 Consitent Hash Function 应当让这个上限尽量小。

# Construction

作者提出一种构建好的 Consistent Hash Function 的方法：

假设我们有两个随机的函数 $r_{B}$ 和 $r_{I}$ ：

- $r_{B}$将 buckets 映射到区间 $[0, 1]$ 上
- $r_{I}$将 items 映射到区间 $[0, 1]$ 上

那么一个好的 Consistent Hash Function 可以定义为 $f_{V}(i) = min(|r_B(b) - r_{I}(i)|)$ ，即将 item 放在距离它最近的 bucket 中。这里的 V 代表着不同的观点，由于观点数量会随着 buckets 的增减而增加，每个观点的内容会随着 buckets 的增减而变化，因此这个 Consistent Hash Function 并不是一个静态的、固定的函数。

这里还有一点，$r_{B}$ 通常需要将每个 bucket 映射到 $[0,1]$区间的多个点上。假设整个区间上的 bucket 数量上限为 $C$ ，那么 $r_{B}$ 需要将每个 bucket 映射到 $k\log{C}$ 个点上，其中 $k$ 为常数，且 $r_{B}$ 在映射单个 bucket 的不同点时，是相互独立的。

接着论文证明了上述函数满足 Monotonicity、Balance、Spread 和 Load 四个特性，这里简要说明一下证明的直觉：

- Monotonicity：新增 bucket 时，新的 bucket 会被映射到 $k\log{C}$ 个随机的点上，而这些点周围的 items 可能会被新的 bucket 抢过来，因为新的 bucket 离它们更近，但绝不会出现 items 被旧的 bucket 抢走的情况，因此 Monotonicity 被满足。
- Balance：因为每个 bucket 的 $k\log{C}$ 个点会随机分布在 $[0, 1]$ 上，因此每个 bucket 将管辖不超过 $\frac{O(1)}{V}$ 长度的区间 (with high probability)。由于 items 会被随机地、均匀地分布在整个区间上，因此 Balance 被满足。
- Spread：假设某个 item 被 $r_{l}$映射到 $[0, 1]$上的某个点，那么每个观点 $V$ 中，最接近 item 的 bucket 只有一个。由于 buckets 会被 $r_{B}$随机、均匀地映射到整个区间，因此即使随着 buckets 数量的增减，观点的数量会增加，但可以想象，不可能所有新 bucket 对应的映射点都在 item 周围，位于它周围的 bucket 映射点数量应当有个上限。因此 Spread 被满足。
- Load：类似 spread 的推理，把 item 和 bucket 位置对调即可。即对于每个 bucket 来说，映射到它附近的 items 数量应当有个上限。因此 Load 被满足。

# Implementation

Consistent Hashing 在许多场景下都有应用，比如：

- 负载均衡
- P2P 系统
- 数据库分片
- ...

尽管场景很多，但它们对 DHT 的要求不一样：

## 负载均衡

在云原生环境下，为了保证服务的扩展性和可用性，通常会同时启动一个服务的多个实例。如何把不同的请求合理地反向代理到不同实例，是负载均衡器需要解决的问题。

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-Ls0CbucFiDOfzdxatcA%2F-Ls0HCUULJNLchbiL-F6%2FLoad-Balancer.jpg?alt=media&amp;token=71a7a887-90dd-42c7-bbe2-8c83e410a2d6" style="zoom:67%;" />

为了保证服务的数据单调一致性，一些服务要求同一个用户的请求必须被路由到同一个实例。这时候就需要对用户的身份标识数据做一次哈希操作，找到固定的实例。但在云原生环境下，同一个服务的实例数量以及地址可能常常发生变化，为使得因实例变动影响到的用户数量最小，就需要使用 Consistent Hashing。

和其它场景不同，Consistent Hashing 在负载均衡场景下，实例数量发生变化时，实际上没有主动重新哈希已有数据的过程，因为它不需要保持对应关系数据。

### 实现思路

材料：哈希函数： $H(x)：string  \to [0, max]$

方案：将每个节点哈希到区间 $[0, max]$ 上随机的 $k$ 个点，将所有节点的所有哈希值排序后保存在一个数组 $A$ 中。每次新来一个请求，就将请求的 ID 哈希到区间 $[0, max]$ 上的某个点，然后利用二分查找找到比 k 大的最小点，后者对应的节点就是目标实例。

算法复杂度分析：

| 操作    | 时间复杂度   |
| ------- | ------------ |
| AddNode | $O(klog(n))$ |
| DelNode | $O(klog(n))$ |
| Get     | $O(logkn)$   |

### 实现示例

- [stathat/consistent](https://github.com/stathat/consistent)。

## P2P 系统

### Pastry

- video
  - [Routing in Distributed Hash Tables | Anne-Marie Kermarrec](https://www.youtube.com/watch?v=WqQRQz_XYg4&t=303s)
- paper
  - [Pastry: Scalable, decentralized object location and routing for large-scale peer-to-peer systems](http://rowstron.azurewebsites.net/PAST/pastry.pdf)

## 数据库分片

(TODO)

# 参考

- [Consistent Hashing and Random Trees](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)