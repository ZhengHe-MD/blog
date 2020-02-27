---
title: Log Structured Merge (LSM) Tree & Usages in KV Stores
date: 2020-02-26 23:23:03
tags:
- kv
- data structures & algorithms
categories:
- system design
mathjax:
- true
---

本文转自我个人的 [gitbook](https://zhenghe.gitbook.io/open-courses/database-design/log-structured-merge-lsm-tree-and-usages-in-kv-stores)。

# Background

数据库中的各种奇技淫巧，实际上都来自于内存与磁盘的读写模式和性能区别。

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/memory-disk.jpg" width="500px">

总结如下表：

| Memory           | Disk              |
| ---------------- | ----------------- |
| byte-addressable | block-addressable |
| fast             | slow              |
| expensive        | cheap             |

当数据库中的数据无法一次性装入内存时，数据的读写就可能需要从内存穿透到磁盘。在 OLTP 场景下，每次事务写入和读取的数据数量都不大。但因为磁盘是块存储设备，无论是否需要，写入和读取都以块 (block) 为单位：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/file-block.jpg" width="500px">

这就导致许多不必要的数据传输。除此之外，在磁盘中连续读取或写入相邻的数据块比随机的数据块效率高。因此数据库在组织数据、索引时，为了减少不必要的 I/O，同时提高 I/O 的效率，就需要尽可能做到：

- 让具有局部性的数据、索引在磁盘上相邻存储
- 让具有局部性的数据、索引批量写入到磁盘中

从而达到两个目的：

- 读出的数据都是需要的
- 写入的数据都是有用的

# B+ Tree

B+ tree 作为数据库索引，已经被许多成熟的关系型数据库所验证。如下图所示：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/b+tree.jpg" width="500px">

>  ref: https://janzhou.org/2015/bf-tree-approximate-tree-indexing.html

得益于其矮胖的平衡树形结构，且将所有数据置于叶子节点，B+ 树的读取性能好且稳定。但它的劣势在于，更新数据时成本较高，不仅要更新数据，还要更新索引。如果表中有多个索引，则都要更新，更新每个索引时还可能更新多个节点，因此写数据的性能在一些写要求苛刻的场景下并不能满足要求。

# LSM Tree

在键值数据库场景下，相较于 B+ 树，LSM 树能够提供更高效的写性能。LSM 树的基本思路就是批量写数据，即先将需要写入的数据放在内存中，等到达一定容量时再写入磁盘。其基本结构如下图所示：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/lsm-tree.jpg" width="600px">

## Buffer

在 buffer 中存储着最新写入的键值对数据，这些数据通常按照键的顺序存储在相应的有序数据结构中，如跳表。同时 Buffer 也是数据查询的入口，存储着最新鲜的数据，或者数据的最新版本。

## Sorted String Table (SST)

当 Buffer 中的数据容量到达一定阈值时，会被顺序地写入磁盘中。磁盘中的数据分为多个层次，每层的数据容量指数递增。每层数据又可以继续被划分为一个或多个 SST，每个 SST 中的数据按顺序排列。且同一层级中的多个 SST 之间的键数据通常不会有交集。每层数据量大小都有限制，当数据量达到阈值时，该层的某个 SST 需要与下一层的若干个 SST 合并 (merge)，流入下一层中。因此图中越上层的数据越新、越下层的数据量越大。

通常认为 SST 是不可变 (Immutable) 的数据结构，所有数据更新都通过写入新的 SST 实现，而所有合并操作也是通过生成新的 SST 实现。这种方式对实现数据库的多版本并发控制 (MVCC) 比较友好。

## Writes

写入键值数据时，先将数据插入放到内存中的缓存 (buffer) 数据结构中，写入请求就可以返回响应。这个过程完全在内存中，因此效率很高。这里的写入操作包含插入、更新和删除，对于 LSM Tree 来说，更新和删除也是插入新的键值对，只不过删除存放的值是一个删除标记。

需要注意的是，大部分数据库为了保证 durability，都会将操作日志写入 WAL (write-ahead log) 中，因此这里实际上每次写数据，都有日志落盘。但因为日志是 append-only 的结构，因此这种写入的效率通常很高。

## Merge (Compaction)

上文提到，disk 中每层数据都有容量限制，当 L 层数据超过阈值时，需要将 L 层的某个 SST 与 L+1 层中键的范围与该 SST 有重叠的多个 SST 合并，产生新的一组 SST 存放在 L+1 层中。如果此时 L+1 层中的数据容量也达到阈值，则会触发连续的合并。

在合并的过程中，相同键的值会被合并，只留下最新的值或者被删除，因此合并的过程也是压缩 (compaction) 的过程。

## Lookups

由于上层的数据比下层的新，因此单条数据的查询首先从 buffer 开始，然后到 L0，L1，...，Ln，在每层数据查询时可以使用二分查找。范围查询也类似，可能需要查询多层数据。这种结构对于具有时间局部性的数据效率更高。

有许多优化手段可以避免每次查询都在每层的每个 SST 上执行二分查找。最常见的有两种，fence pointers 和 Bloom filters。

### fence pointers

fence pointers 的思路就是记录每个 SST 中建的最大值和最小值，由于每层中的多个 SST 的键值范围互相不重叠，这样对于单个键的查询，就能将查找范围缩小到某一个 SST 中。

### Bloom filters

bloom filter 则是在 fence pointers 之前的一层过滤器，通过其较低的 FP(false positive)，将大多数不必要的查询拒之门外。

整个数据查询的架构如下图所示：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/lsm-tree-with-bloom-filters.jpg" width="500px">

## Tiering & Leveling

在 LSM 树的实现中，合并时间点的选择决定了该实现的读写性能取舍。通常合并地越频繁，读取数据所涉及的 SST 数量越少，读性能越好，与此同时写操作的放大效应越大，单条数据被合并的次数越多，写性能越差。

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/tiering-leveling.jpg" width="500px">

在 Tiering 中，每一层数据有多个 SST；在 Leveling 中，每一层数据通常只有一个 SST：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/tiering-leveling-2.jpg" width="500px">

其中两层之间的容量比例是很重要的参数，记为 R。R 越大，则层数越少；R 越小，则层数越多。数据的总层数为 $log_{R}N$ 。当 R 很小时，Tiering 与 Leveling 的效果趋于一致。通常 Tiering 中的 R 比较大，而 Leveling 的 R 比较小，如下图所示：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/tiering-leveling-curve.jpg" width="400px">

考虑极端情况：当 R 很小时，相邻两层数据容量差别小，每层数据就是一个 sorted array，在每层中查找数据只需一次二分查找，这种情况下读性能好；当 R 很大时，每层数据约等于没有顺序，就像日志一样，这种情况下写性能好。

很自然的，不同业务场景对读写性能有不同的需求，因此在各个 R 上都有相应的键值数据库存在：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/tiering-leveling-curve-with-products.jpg" width="400px">

# Case Study: couchbase/moss

moss 是由 couchbase 的工程师开发的键值数据库，本节的 case study 来自于 Marty Schoch 在 GopherConn 2017 上的分享：[Building a High-Performance Key/Value Store in Go](https://www.youtube.com/watch?v=ttebJcN5bgQ)。

## General Purpose Key-Value Stores

站在 Go 语言工程师的角度上，通用键值数据库中的 key 和 value 就是 byte slice：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-kv.jpg" width="400px">

通用键值数据库通常需要支持：

1. Get/Set/Delete Values by Key
2. Iterate Ranges of Key-Value Pairs
3. Atomic Batch Updates
4. Isolated Read Snapshots
5. Persistence to Disk

1， 2， 3， 5 项都容易理解，Isolated Read Snapshots 其实就是 MVCC，即当用户开始遍历键值数据时，之后的数据更新对当前的遍历不可见。

## Special Purpose Key-Value Store

moss 团队需要的并非一个通用键值数据库，他们有特殊的性能关注点：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-requirements.jpg" width="500px">

1. 写吞吐
2. 写入数据后到能查询之间的时延

## Off-The-Shelf Solutions

在决定开发自己的键值数据库之前，moss 工程师也调研了一些市面上开源的数据库，如 BoltDB, RocksDB, GoLevelDB 以及 Badger。

### BoltDB

BoltDB 是 Go 语言编写的键值数据库。BoltDB 通过单线程解决并发控制解决方案，支持可序列化事务隔离，通过成熟的 B+ 树来解决内存、磁盘读写模式的不匹配问题，BoltDB 已经在许多商业系统中成功运用，其可靠性也得到了验证。它的优势在于极高的读效率，但由于它使用 B+ 树，天然地其写性能要弱于使用 LSM 树的键值数据库解决方案。

有关 BoltDB 的分析可参考本人的项目 [ZhengHe-MD/learn-bolt](https://github.com/ZhengHe-MD/learn-bolt)。

### RocksDB

RocksDB 是基于 google 的 LevelDB 二次开发的键值数据库，其内部采用了 LSM 树设计，在 Write-Amplification-Factor (WAF)、Read-Amplification-Factor (RAF) 和 Space-Amplification-Factor (SAF) 中取得更灵活的权衡，它支持多线程数据压缩，能够处理 TB 级别数据。相较于 BoltDB，RocksDB 有更好的读写性能平衡，但它使用 C++ 编写，使得 moss 工程师需要利用 cgo 来跨越 Go 和 C++ 的边界，且这一层边界开销也有一定的性能损耗，因此 moss 团队没有采用 RocksDB。

### GoLevelDB

GoLevelDB 是 LevelDB 的 Go 语言原生实现，其同样采用 LSM 树设计，但经过反复调试，moss 团队发现它无法达到其读写性能的高要求，因此放弃。

### Badger

在 moss 团队开始寻找键值数据库解决方案时，badger 项目还未开源，因此无法验证其是否满足需求。

## Design Philosophy

Rob Pike 在 Notes on C Programming 说道：

> Rule3: Fancy algorithms are slow when n is small, and n is usually small. Fancy algorithms have big constants. Until you know that n is frequently going to be big, don't get fancy.
>
> Rule4: Fancy algorithms are buggier than simple ones, and they're much harder to implement. Use simple algorithms as well as simple data structures.

Dave Cheney 提到：

> Simplicity cannot be added later

moss 团队的设计理念非常明确：能简则简。

## Interface Design

### Collection

在 moss 中，每个数据库就是一个 Collection，它只暴露两个接口：

```go
type Collection interface {
    ExecuteBatch(b Batch) error
    Snapshot() (Snapshot, error)
}
```

它的设计十分精简：

- 所有写数据操作都是一个 Batch
- 所有读数据操作都从一个 Snapshot 上发起

### Batch



```go
type Batch interface {
    Set(key, value []byte) error
    Delete(key []byte) error
}
```

### Snapshot



```go
type Snapshot interface {
    Get(key []byte) ([]byte, error)
    Iterator(start, end []byte) (Iterator, error)
}
```

## Basic Idea

moss 的解决方案很简单，用户写入一批键值数据 (这里忽略其值)，如下图所示：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-basic-idea-1.jpg" width="500px">

如果可以将这批数据按 key 排序：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-basic-idea-2.jpg" width="500px">

就可以很方便地查询数据：

- 查询单个键：二分查找
- 范围查询：二分查找 + 循环遍历

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-basic-idea-3.jpg" width="500px">

## Data Layout

将用户插入的一批数据成为 Batch，在数据库内部排序后称为 Segment：



```go
type segment struct {
    data []byte
    meta []uint64
}
```

假设用户执行：



```go
Set([]byte("name"), []byte("marty"))
```

接收该键值对的 segment 会将 "name" 和 "marty" 依次 append 到 data 之后，并在 meta 中增加 2 个 uint64 的元数据：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-data-layout-1.jpg" height="200px">

- 20 表示键值数据在 data 中的偏移量
- s 表示操作名称 (set)
- 4、5 分别表示键和值的长度

元数据中的两个 uint64 具体结构如下：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-data-layout-2.jpg" height="150px">

这样存储的好处是，排序时只需要调整 meta，而无需修改 data。

显然用户会向数据库中插入许多 batch，每当插入新的 batch 时，moss 数据库就先将 batch 原地排序，然后插入到 segmentStack 中，并用锁来保护 segmentStack 的并发安全：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-data-layout-3.jpg" width="400px">

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-data-layout-4.jpg" width="400px">

在 segmentStack 中，新的 segment 在栈顶，老的 segment 在栈底。这种结构与 LSM 树如出一辙，新的数据在栈顶，老的数据在栈底。

## Operation

### Get

当用户要查询单个键时，就从栈顶的 segment 开始，依次对每个 segment 执行二分查找，直到找到键值数据或遍历完整个 segmentStack。

### Iteration

利用小根堆，将每个 segment 中排序最小的元素开始放入小根堆中：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-iteration.jpg" width="500px">

读取堆顶元素，然后从相应的 segment 中补进新元素，直到遍历完成所有 segments，也就完成了 iteration 的操作。

### Atomic Batches

利用锁来保护 segmentStack 的并发安全即可。

### Isolated Snapshots

在 moss 眼中，所有 segments 都是 immutable 的数据，一旦 batch 变成 segment，该 segment 中的数据就不会再发生变化。这时候支持 Isolated Snapshots/MVCC 就很容易了，因为 segmentStack 保存的是对 segment 的指针，只要复制一份 segmentStack 即可满足要求。新到来的写入操作不会改变已存在的 segment。

### Background Merger

当然，如果一直往 segmentStack 中压入 segment，前者会变得很长，数据读取的效率会下降，这时候就需要合并：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-background-merger-1.jpg" width="400px">

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-background-merger-2.jpg" width="400px">

## Database File

moss 的数据库文件形式为 append-only，内部数据存储的内容如下所示：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-database-file-1.jpg" width="400px">

其中 footer 中保存着指向 segments 偏移量的指针，且每个 footer 都保存对之前所有 segments 的指针。数据库启动时，会打开该文件，从后往前找到第一个合法的 Footer。如果最后一个 Footer 没有完整写入，数据库会找到更早的 Footer，当然这会丢失一些数据，但不影响数据库的正常运行。另外考虑到 WAL，数据的持久性依然可以得到保障。

从数据库读取文件后，moss 会将相应的 segments mmap 到内存中，让操作系统来处理内存管理逻辑：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-database-file-2.jpg" width="400px">

此外，moss 还会利用 background merger 将文件中的 segments 合并。类似地，所有 segments 都是 immutable 的数据，每次合并都将生成新的数据文件：

<img src="/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/moss-database-file-3.jpg" width="400px">

从而保障 Isolated Read Snapshots 正常进行。

# References

- Projects
  - [github: couchbase/moss](https://github.com/couchbase/moss)
  - [github: google/leveldb](https://github.com/google/leveldb)
    - [doc/impl.md](https://github.com/google/leveldb/blob/master/doc/impl.md)
  - [github: facebook/rocksdb](https://github.com/facebook/rocksdb)
  - [github: syndtr/goleveldb](https://github.com/syndtr/goleveldb)
- Books
  - [Getting Started with LevelDB](https://www.amazon.com/Getting-Started-LevelDB-Andy-Dent/dp/1783281014)
- Blogs
  - [SSTable and Log Structured Storage: LevelDB](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)
- Courses
  - [CS265: Big Data Systems: An LSM-Tree Based Key-Value Store](http://daslab.seas.harvard.edu/classes/cs265/project.html)
- Videos
  - [Youtube: Scaling Write-Intensive Key-Value Store by Microsoft Research](https://www.youtube.com/watch?v=b6SI8VbcT4w)
  - [Youtube: Building a High-Performance Key-Value Store in Go by Marty Schoch](https://www.youtube.com/watch?v=ttebJcN5bgQ) & [slides](https://speakerdeck.com/mschoch/value-store-in-go?slide=3)
