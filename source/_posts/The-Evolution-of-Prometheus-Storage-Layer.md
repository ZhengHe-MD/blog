---
title: Prometheus TSDB 的存储层演进 —— PromConf 演讲笔记
date: 2020-02-27 09:50:01
tags:
- tsdb
categories:
- system design
---

> 注：如果只想了解 Prometheus TSDB 的存储层现状，可以直接移步[ Ganesh Vernekar 的博客](https://ganeshvernekar.com/blog/)，他写了 7 篇系列文章介绍这个主题。

Prometheus 无疑是时下最流行的监控平台，它负责定期从不同的采集目标拉取样本数据，然后持久化到内建的时序数据库中，向外部提供便捷的查询接口。本文主要探讨的是 Prometheus 存储层的演进过程，整理自 Prometheus 团队在历届 PromConf 上的分享以及相应的文档。

<!-- more -->

# 时序数据库

TSDB 是 Promtheus 的一个重要模块，它是一个时序数据库实现。在进入正题前，我们有必要先了解时序数据库是什么。时序数据库是数据库的一种，与关系型数据库、图数据库等概念并列，专用于存储随时间变化的数据，如股票价格、传感器数据、机器状态监控等等。**样本 (Sample)** 是变量在某个时刻的取值，同一个变量在不同时刻的取值按时间顺序合并就是**时序 (Time Series)**。以股票价格为例，样本是某支股票在某时刻的价格绝对值，时序则是这支股票的全部历史。

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/ts-sample.jpg" width="500px">

每个样本由**时序标识**、**时间戳**和**数值** 3 部分构成，其所属的时序就由一系列样本构成。由于时间是连续的，我们不可能、也没有必要记录时序在每个时刻的数值，因此在实现上，时序的数据与**采样间隔**相关。采样间隔越小、样本总量越大、捕获细节越多；采样间隔越大、样本总量越小、遗漏细节越多。

数据的高效查询离不开索引，对于时序数据而言，唯一的、天然的索引就是时间。因此时序数据库的存储层相比于关系型数据库要简单得多。仔细思考，你可能会发现时序数据在某种程度上就是键值数据的一个子集，因此键值数据库天然地可以作为时序数据的载体。目前在业界中，一个单机时序数据库能容纳百万量级的时序数据。要从中检索时序，建立高效的索引很重要。

## 基本问题

时序数据库要解决的基本问题，基本涵盖在下图中：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/tsdb-fundamental-problem.jpg" width="500px">

在 [Log Structured Merge (LSM) Tree & Usages in KV Stores](/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/) 一文中，我写过这样一句话：「许多数据库的奇技淫巧都是在解决内存与磁盘的读写模式、性能的不匹配问题」。时序数据库也是数据库的一种，只要需要持久化，自然不会例外。但与键值数据库相比，时序数据库存储有更特殊的读写特征，Prometheus 的作者之一 Björn Rabenstein 将这种特点称为「垂直写，水平读」。

图中每条横线就是一个时序，每个时序按照 (准) 固定间隔采集的样本数据构成。时序数据库中有很多不断写入新样本的活跃时序，我们可以用一个垂直的窄方框表示这种数据写入模式，即每个活跃时序都会产生新的样本数据；用户在查询时，常常需要观察某个、或某组时序在某个时间段内的变化趋势，或执行聚合计算。相似的时序具有局部性，将它们在物理上相邻存储对于读写性能的提升都有帮助，于是数据读取模式可以用一个水平的方框表示。这就是上面提到的「垂直写、水平读」。

# Prometheus TSDB 的存储层演进

Prometheus 是为云原生环境的监控而生，在它的设计中至少需要考虑两个因素：

1. 实例可能随时出现、消失，时序也会随着实例出现和消失。在某些时刻系统中可能存在大量时序，只有部分处于活跃状态，这会在多方面带来挑战：
   * 如何存储大量时序避免资源浪费
   * 如何定位被查询的少数几个时序
2. 监控系统本身应该尽量少地依赖外部服务，否则外部服务失效将引发监控系统失效

对于第 2 点，Prometheus 团队选择放弃集群，以单机模式开发，并且在单机系统中使用本地 TSDB 做数据持久化，完全不依赖外部数据库；第 1 点则需要存储、索引、查询引擎层合作解决。Prometheus TSDB 存储层的演进分成 3 个阶段：

* 第一代: Prototype
* 第二代: Prometheus V1
* 第三代: Prometheus V2

*注意：这里只关注 Prometheus 时序数据的存储，不涉及索引、WAL 等其它数据的存储。*

## Data Model

尽管数据模型是存储层之上的抽象，理论上它不应该影响存储层的设计。但理解数据模型能够帮助我们更快地理解存储层。

在 Prometheus 中，每个时序实际上由多个**标签** (labels) 一起标识，如：

```
api_http_requests_total{path="/users",status=200,method="GET",instance="10.111.201.26"}
```

该时序的名字为 api_http_requests_total，标签为 path、status、method 和 instance，只有时序名字和标签键值完全相同的时序才是同一个时序。在实现上，时序名字就是一个隐藏标签：

```
{__name__="api_http_requests_total",path="/users",status=200,method="GET",instance="10.111.201.26"}
```

对于用户来说，标签之间不存在先后顺序，用户可能关注：

* 所有 api 调用的 status
* 某个 path 调用的成功率、QPS
* 某个实例、某个 path 调用的成功率
* ...

## 第一代: Prototype

在 Prototype 阶段，Prometheus 直接利用开源的键值数据库 (LevelDB) 作为本地持久化存储，并采用与 [BigTable 推荐的时序数据方案](https://cloud.google.com/bigtable/docs/schema-design-time-series?hl=en#server_metrics)类似的 schema 设计：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/bigtable-ts-schema.jpg" width="600px">

将**时序名称、标签 (固定顺序)、时间戳**拼接成每个样本的键，于是同一个时序的数据就能够连续存储在键值数据库中，提高范围查询的效率。但从图中可以看出，这种方式存储的键很长，尽管键值数据库内部会对数据进行压缩，但是在内存中这样存储数据很浪费空间，无法满足项目的设计要求。Prometheus 希望能在内存中压缩数据，从而同时容纳更多活跃的时序数据，同时在磁盘中也能按类似的方式压缩编码，提高效率。时序数据比通用键值数据有更显著的特征。即使键值数据库能够压缩数据，但针对时序数据的特征，使用特殊的压缩算法能够取得更好的压缩率。因此最终这个方案没有被采纳。

## 第二代: Prometheus V1

### 压缩

##### 为什么要压缩？

我们一起做一个计算，假设监控系统的技术要求如下：

* 500 万活跃时序
* 30 秒采样间隔
* 1 个月数据留存

那么经过计算可以得到具体的存储要求：

* 平均每秒采集样本 166000 个
* 存储样本总量为 4320 亿

假设没有任何压缩，不算时序标识，每个样本需要 16 个字节存储空间 (时间戳 8 个字节、数值 8 个字节)，整个系统的存储总量为**7TB**，如果想保存 6 个月数据，则总共需要**42TB**，那么如果能找到一种有效的方式压缩数据，就能同时在节点的内存和磁盘中存放更多、更久的时序数据。

##### Chunk

上文介绍过：时序数据库要解决的根本问题是「垂直写，水平读」。每次采样都会需要为每个活跃时序写入一条样本数据，但如果每次都将每个时序新样本数据 (16 个字节) 落到 HDD/SSD 中，不仅效率底下，还可能减少块存储设备的寿命。因此 Prometheus V2 将数据按固定长度切割相同大小的 chunks，方便压缩、批量读写。

访问时序数据时，Prometheus 使用 3 层抽象，如下图所示：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/iterator.jpg" width="600px">

应用层使用 Series Iterator 顺序访问时序中的样本，而 Series Iterator 底下由一个个 Chunk Iterator 拼接而成，每个 Chunk Iterator 负责将压缩编码的时序数据解码返回。这样做的好处是，**每个 chunk 甚至可以使用完全不同的方式编码**，方便开发团队尝试不同的编码方案。

##### 时间戳的压缩: Double Delta

由于数据采样间隔固定，实际前后两个样本点时间戳的差值几乎也相差无几，如 15s，16s。我们可以更近一步，只存储差值的差值，那么如果一切顺利，我们几乎不用再为新的时间戳花费额外的空间，这便是所谓的 "**Double Delta**"。本质上分析，如果未来所有的采集时间戳都可以精准预测，那么每个新时间戳的信息熵为 0 比特。但现实并不完美，网络可能延迟、中断，实例可能遇到 GC、重启，采样间隔随时有可能波动：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/pretty-regular-sample-interval.jpg" width="500px">

这种波动的幅度终究是有限的，Prometheus 采用了和 Facebook 的内存时序数据库 Gorilla 类似的方式编码时间戳，详情可以参考我的另一篇博客 [Gorilla]((/blog/2020/02/16/Gorilla-A-Fast-Scalable-In-Memory-Time-Series-Database-2015/)) 以及 Björn Rabenstein 在 PromConn 2016 的演讲 [ppt](https://docs.google.com/presentation/d/1TMvzwdaS8Vw9MtscI9ehDyiMngII8iB_Z5D4QW4U4ho/edit#slide=id.g15afea0287_0_16) ，细节比较琐碎，这里不赘述。

##### 样本值的压缩

Prometheus 和 Gorilla 中的每个样本值都是 float64 类型。Gorilla 利用 float64 的二进制表示 (IEEE754) 将前后两个样本值 XOR 来寻找压缩的空间，能获得 1.37 bytes/sample 的压缩能力。Prometheus V2 采用的方式比较简单：

* 如果可能的话，使用整型 (8/16/32 位) 存储，否则用 float32，最后实在不行就直接存储 float64
* 如果数值增长得很规律，则不使用额外的空间存储

以上做法给 Prometheus V1 带来了 3.3 bytes/sample 的压缩能力。相比于为完全存储于内存中的 Gorilla 相比，这样的压缩比对于 Prometheus 已经够用。尽管 V1 没有在这方面深耕，Prometheus V2 最终融合了 Gorilla 采用的压缩技术，获得更高的压缩比。

### Chunk 编码

Prometheus V1 将每个时序分割成大小为 1KB 的 chunks，如下图所示：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/chunk-encoding.jpg" width="600px">

在内存中保留着最新写入的 chunk，称为 head chunk，它负责接收新的样本。每当一个 head chunk 写满 1KB 时，会立即被冻结，这时它已经是一个完整的 chunk，从此刻开始它的数据不可变，同时生成一个新的 head chunk 负责接收新的数据。每个完整的 chunk 会被尽快地持久化到磁盘中。内存中保存着每个时序最近被写入或被访问的 chunks，当 chunks 数量过多时，存储引擎会将超过的 chunks 通过 LRU 策略清出。

在 Prometheus V1 中，每个时序都会被存储到在一个独占的文件中，这也意味着大量的时序将产生大量的文件。存储引擎会定期地去检查磁盘中的时序文件，如果发现已有 chunk 数据超过保留时间，就会将其删除。

由于需要根据不同的 PromQL 对原始时序数据聚合计算，Prometheus 查询引擎需要将必要的数据完全读入内存后才能运行。因此在执行之前，存储引擎需要将不在内存中的 chunks 预加载到内存中：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/chunk-preloading.jpg" width="600px">

如果在内存中的 chunks 持久化之前系统发生崩溃，会产生数据丢失。为了减少数据丢失，Prometheus V1 使用 checkpoint 存储各个时序中尚未写入磁盘的 chunks。

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/checkpoint.jpg" width="600px">

### Prometheus V1 vs. Gorilla

与 Prometheus 不同，Gorrila 是纯内存时序数据库，我们可以对比一下 Prometheus V1 与 Gorilla，来进一步理解它们设计过程中采用不同决定的原因。

| Features/Settings | Prometheus V1                                                                 | Gorilla                                              |
| ----------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------- |
| Storage           | Demultiplexing to local disk<br />尽量单机、无依赖、支持大量时序存储                           | In-memory only<br />为了查询快，必须全部在内存，可以分片               |
| Resolution        | 1ms                                                                           | 1s                                                   |
| Chunks            | Fixed-size chunks (1KB)<br />优化 I/O                                           | Fixed-time blocks (2h)<br />优化 I/O，同时方便检索 blocks     |
| Decoding          | Random accessibility & decoding<br />希望用户不用关心编解码<br />直接通过 HTTP API 获取易于理解的数据 | Not concerned with decoding<br />为了查询快，减少网络延迟，在客户端解码 |
| Compression       | 3.3 bytes/sample<br />够用的压缩率                                                  | 1.37 bytes/sample<br />极致的压缩率，内存更昂贵                  |

## 第三代: Prometheus V2

### 上一代的问题

Prometheus V1 中，每个时序数据对应一个磁盘文件的实现弊端在生产中逐渐显现：

* 由于在云原生环境下，只要部署单元发生变化，新的时序就会不断产生，导致存储层所需的文件数量远远高于活跃的时序数量。任其发展迟早可能会将文件系统的 inodes 消耗殆尽。而且一旦发生，恢复系统将异常麻烦。不仅如此，在新旧时序大量更迭时，由于旧时序数据尚未从内存中清出，系统的内存消耗量也会飙升，造成 OOM；
* 即便使用 chunks 来批量读写数据，系统每秒钟仍要向磁盘写入数千个 chunks，造成 I/O 压力；如果通过增大每批写入的量来减少 I/O 次数，又会造成内存的压力；
* 同时保持打开时序文件的设计需要消耗大量的资源。而如果在查询前后打开、关闭文件，又会增加查询的时延；
* 数据超过留存时间时需要删除相关的 chunks，这意味着每隔一段时间就要对数百万的文件执行一次删除操作，这个过程可能需要持续数小时；
* 通过周期性地将未持久化的 chunks 写入 checkpoint 文件理论上确实可以减少数据丢失，但是如果执行数据恢复需要很长时间，那么实际上又错过了新的数据，还不如不恢复。

因此 Prometheus 的第三代存储引擎的主要改变就是放弃「一个时序对应一个文件」。

### 磁盘文件布局设计

第三代存储引擎在磁盘中的文件结构如下图所示：

```sh
$ tree ./data
./data
├── b-000001
│   ├── chunks
│   │   ├── 000001
│   │   ├── 000002
│   │   └── 000003
│   ├── index
│   └── meta.json
├── b-000004
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
├── b-000005
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
└── b-000006
    ├── meta.json
    └── wal
        ├── 000001
        ├── 000002
        └── 000003
```

根目录下，顺序排列着编了号的 blocks，每个 block 中包含 index 和 chunk 文件夹，后者里面包含编了号的 chunks，每个 chunk 包含**许多不同时序的样本数据**。其中 index 文件中的信息可以用来帮助快速锁定时序的标签及其可能的取值，进而找到相关的时序和持有该时序样本数据的 chunks。值得注意的是，最新的 block 文件夹中还包含一个 wal 文件夹，后者将承担数据故障恢复的职责。

### 许多小数据库

第三代存储引擎将所有时序数据按时间分片，即在时间维度上将数据划分成互不重叠的 blocks，如下图所示：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/partition.jpg" width="600px">

每个 block 实际上就是一个小型数据库，内部存储着该时间窗口内的所有时序数据，并且拥有自己的 index 和 chunks。除了正在接收最新数据的 block 之外，其它 blocks 都不可变。

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/many-little-databases.jpg" width="600px">

为了防止数据丢失，所有新采集的数据都会先被写入到 WAL 日志中，在系统恢复时能快速地将其中的数据恢复到内存中。在查询时，需要将查询发送到不同的 block 中，再将结果聚合。

按时间将数据分片赋予了存储引擎新的能力：

* 查询某个时间范围内的数据时可以直接忽略在时间范围外的 blocks；
* Block 持久化到磁盘只涉及到少量文件的写入；
* 内存中缓存了更长时间的数据，提高查询效率；
* 每个 chunk 不再是固定大小，压缩方式也可以单独指定；
* 删除超过留存时间的数据很简单：直接删除整个文件夹即可。

### mmap

第三代引擎将数百万的小文件合并成少量大文件，也让 mmap 成为可能。利用 mmap 将文件 I/O 、缓存管理交给操作系统，降低 OOM 发生的频率。

### 碎片整理 (Compaction)

写入数据时，每个 block 最好不要太大，实现中默认使用的是 2 小时，从而避免在内存中积累过多的数据。读取数据时，若查询涉及到多个时间段，就需要对许多个 block 分别执行查询，然后再合并结果。假如需要查询一周的数据，那么这个查询将涉及到 80 多个 blocks，降低数据读取的效率。

为了既能写得快，又能读得快，新版设计还引入了 compaction 机制，后者将一个或多个 blocks 中的数据合并成一个更大的 block，在合并的过程中会自动丢弃被删除的数据、合并多个版本的数据、重新结构化 chunks 来优化查询效率，如下图所示：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/compaction.jpg" width="600px">

### 保留时间 (Retention)

当数据超过保留时间时，删除旧数据非常容易：

<img src="/blog/2020/02/27/The-Evolution-of-Prometheus-Storage-Layer/retention.jpg" width="600px">

直接删除在边界之外的 block 文件夹即可。如果边界在某个 block 之内，则暂时将它留存，直到边界超出为止。Compactor 会将旧的 blocks 合并成更大的 block 来加速查询；但在 retention 中，如果 blocks 太大又会增加磁盘占用。因此 compaction 与 retention 的策略之间存在着一定的互斥关系。Prometheus 在启动参数中支持对单个 block 的大小作出限制，来平衡二者。

看到这里，相信你已经发现了：这不就是 [LSM Tree](/blog/2020/02/26/Log-Structured-Merge-LSM-Tree-Usages-in-KV-Stores/) 吗？每个 block 就是按时间排序的 SSTable，内存中的 block 就是 MemTable。

### 样本值的压缩

第三代存储引擎融合了 Gorilla 的 XOR float encoding 等多种方案，将压缩比提升到 1-2 bytes/sample。这些方案按优先级排列如下：

1. Zero encoding：如果完全可预测，则无需额外空间
2. Integer double-delta encoding：如果是整型，可以利用 double delta，将不等的前后间隔分成 6/13/20/33 bits 几种，来优化空间使用
3. XOR float encoding：参考 [Gorilla](/blog/2020/02/16/Gorilla-A-Fast-Scalable-In-Memory-Time-Series-Database-2015/)
4. Direct encoding：直接存 float64

平均下来能取得 1.28 bytes/sample 的压缩能力。

# 参考资料

* [PromCon 2017: Storing 16 Bytes at Scale - Fabian Reinartz](https://www.youtube.com/watch?v=b_pEevMAC3I&feature=youtu.be), [slides](https://promcon.io/2017-munich/slides/storing-16-bytes-at-scale.pdf)
* [Writing a Time Series Database from Scratch](https://fabxc.org/tsdb/)
* [PromCon 2016: The Prometheus Time Series Database - Björn Rabenstein](https://www.youtube.com/watch?v=HbnGSNEjhUc), [slides](https://docs.google.com/presentation/d/1TMvzwdaS8Vw9MtscI9ehDyiMngII8iB_Z5D4QW4U4ho/edit#slide=id.g59e2f6081_1_0)
* [Percona Live Open Source Database Conference 2017: Life of a PromQL query](https://www.youtube.com/watch?v=evPYwNzoltU&t=782s)
* [Prometheus 1.8 doc: storage](https://prometheus.io/docs/prometheus/1.8/storage/)
* [Prometheus 2.16 doc: storage](https://prometheus.io/docs/prometheus/latest/storage/)
* [Google Cloud: Schema Design for Time Series Data](https://cloud.google.com/bigtable/docs/schema-design-time-series?hl=en#server_metrics)
