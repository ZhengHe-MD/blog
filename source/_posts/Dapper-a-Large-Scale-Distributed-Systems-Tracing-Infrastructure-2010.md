---
title: 'Dapper, a Large-Scale Distributed Systems Tracing Infrastructure (2010)'
date: 2020-02-17 18:16:20
category: 论文
---

早在 2008 年，Google 就已开始分布式调用链追踪的工作，经过两年的打磨后，Dapper 系统问世，并通过这篇文章将其设计公之于众。遗憾的是，Dapper 并不是开源项目，但它的设计理念依然深刻影响到后来的 Jaeger、Zipkin 等开源分布式追踪项目，以及相关的标准 Opentracing、OpenTelemetry。

本文不是原文的精准翻译，而是一次重述和简述，旨在记录分布式调用链追踪要解决的核心问题和潜在解决方案。

<!-- more -->

# Why & Design Goals

云原生环境中，一次请求的处理可能途径多个服务的任意实例，彻底理解系统就需要理解各服务内部的逻辑，理清这些服务之间的关系，甚至有时候还需要了解服务所在物理机的当时状态。系统出现异常时，如果其行为无法被追踪、被理解，就无法为解决异常快速提供线索。

通常这些异常会被监控捕捉，如时延异常、错误日志、程序崩溃，在紧急处理之后，就需要调查案发现场，彻底解决问题。这时候就需要了解每个请求在整个微服务集群内部的行踪。

这就向分布式追踪系统提出了两点要求：

- 处处部署 (ubiquitous deployment)
- 持续监控 (continuous monitoring)

如果部署不完全或者监控有间断，就可能有一小部分历史无法被追踪到，从而影响到问题定位的准确度，使得追踪效果大打折扣。

据此，我们提出追踪系统的 3 个主要设计目标：

1. **低成本 (Low overhead)**：对服务的性能影响应该能够忽略不计
2. **对应用透明 (Application-level transparency)**：应用开发者对追踪系统无感知
3. **扩展性好 (Scalability)**：支持部署到所有服务的所有实例上

在此基础上，数据从采集到可以被查询、分析的延迟越小越好，起到的作用也越大、越及时。

# How

## General Approaches

分布式追踪的设计方案主要可以分为两类：黑盒法（black-box）和标记法（annotation-based）：

### 黑盒法

黑盒法无需任何侵入性代码，只通过统计回归等手段来推测服务之间的关系。它的优势在于无需修改代码，缺点在于记录不准确，且需要大量数据才能够推导出服务间的关系。

### 标记法

标记法需要为每个请求打标记，并通过一个全局标识符将请求途径的所有服务信息串联，复盘整个链路。标记法记录准确，但它的缺点也很明显，需要将标记代码注入到每个服务中。

在 Google 内部，几乎所有应用都使用相同的 threading model、control flow 和 RPC systems，因此可以将打标记的工作集中在少量的公共库中，同样能够达到对应用透明的效果。

## Data Models

通常一个请求在微服务集群中的调用链可以被抽象成树形结构，假设 RequestX 的处理过程如下图所示:：

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LzZkDzl5FDpMB3N6FWj%2F-LzZohKpQwwXEAggeNnR%2FScreen%20Shot%202020-01-27%20at%2010.26.45%20AM.jpg?alt=media&amp;token=4abea3f6-a071-4078-8c2a-8d176fa17f35" style="zoom:40%;" />

相应调用链追踪的树状结构为：

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LzZkDzl5FDpMB3N6FWj%2F-LzZpKdJRxQj9jYRwE2p%2FScreen%20Shot%202020-01-27%20at%2010.26.31%20AM.jpg?alt=media&amp;token=3fd0ff8b-3a5a-4f94-a53b-82b2c789e9f0" style="zoom:50%;" />

整棵树称为一个 trace，树上的节点称为 span。每个 span 都记录着 parent id 和 trace id，表明其所属父节点和调用链，其中没有 parent id 的 span 称为 root span，root span 的 id 就是 trace id。

每个 span 都需要记录其开始时间和结束时间，如果应用开发者有记录其它信息的需求，则可以手动增加相应的标记。

## Pipeline

Dapper 记录、收集调用链信息的流水线主要分成 3 个阶段：

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LzZuNs2PN7fitdgKdSX%2F-LzZw7CCiTRmnBESMsKE%2FScreen%20Shot%202020-01-27%20at%2010.59.13%20AM.jpg?alt=media&amp;token=1251874d-8189-4591-b246-84cc36a4da34" style="zoom:67%;" />

1. span 数据写入本地日志文件
2. dapper daemon 从本地日志文件中收集数据
3. dapper collectors 将数据写入 Bigtable 的大区仓库 (regional repositories)

在 Bigtable 中，每行数据就是一个 trace，且每行可以有任意列，恰好方便存储不定长的 trace/span 数据。Dapper 向开发者提供相应的 API 和 SDK，方便 Google 的开发者能够据此搭建数据分析工具，定制化地辅助线上问题排查。

## Overhead

调用链追踪的主要成本在于：trace generation 和 collection。

### Trace generation

在 Dapper 中，生成 root span 需要 204 ns，生成 non-root span 需要 176 ns。这里面相差的部分就是生成 全局唯一 trace id 的时间成本。

在 Dapper 的 runtime library 中，最耗时的操作就是将 trace 信息写入本地磁盘，但考虑到使用批量和异步写入的方式优化，对被跟踪的服务本身的影响就相对削弱了。

### Trace collection

Dapper daemon 需要从本地日志文件中读取 trace 信息，然后发送给 Dapper collectors。经过 benchmark 测试验证，Dapper daemon 从未使用超过单核 0.3% 的计算资源，且使用的内存空间极小，可忽略不计。同时 Dapper daemon 在 kernel scheduler 中的优先级被设置为最低，必要时会出让计算资源。

在实践中，平均每个 span 的大小约为 426 字节，经过计算，在生产环境中 Dapper 占用的网络带宽大约为总量的 0.01%。

### Effect on production workloads

高吞吐的服务随时都会接收大量的请求，产生大量的 tracing 数据，而这类服务通常又是对性能最敏感的。下表中以 Google 的网页搜索服务集群为例，测量了不同的 trace 采样率对服务本身的影响：

<img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LzzYHlM85L3z5GmLl19%2F-Lzzi5vkmme-1FkOdHC7%2FScreen%20Shot%202020-02-01%20at%203.47.42%20PM.jpg?alt=media&amp;token=3d561901-dede-4f09-b268-87bec6607b54" style="zoom:50%;" />

The experimental errors for these latency and throughput measurements are 2.5% and 0.15% respectively.

从图中可以发现，尽管调用链追踪带来的性能影响不是很大，但并不能忽略不计，对 trace 数据进行抽样是很有必要的。当抽样率小于 1/16 时，影响范围已经小于误差范围。在实践中，我们设置 1/1024 的抽样率就能收集到足够多的 trace 数据。

除此之外，使用更低的采样率可以让 trace 数据在本地磁盘存活更长的时间，为整个搜集框架争取更大的灵活度。

### Adaptive Sampling

调用链追踪的成本与单位时间内收集的 trace 数量成正比。在 Dapper 的首个生产版本中，采用了统一的采样率 1/1024，这种固定采样率不会对高吞吐的在线服务产生不必要的影响。但在这样的采样率下，可能忽略掉一些发生不频繁的重要事件。

因此 Dapper 团队正在研发自适应采样率机制，针对不频繁的重要事件能提高采样率。实际被使用的抽样率会被记录在 trace/span 数据中，帮助后期工具分析。

### Additional Sampling

上面介绍的均匀抽样和自适应抽样都是为了减少对被追踪服务本身性能的影响。但 Dapper 本身还需要控制整体抽样数据的规模，在论文发表时，Dapper 在生产环境中每天将产生 1T 的追踪数据；同时 Dapper 的用户希望追踪数据能够保持两周。因此这里存在着存储资源与追踪密度之间的权衡。除此之外，高抽样率也会提升 Dapper collectors、Bigtable 的吞吐量。因此 Dapper 团队引入了额外的一层抽样，来实现全局的、系统级的控制。

实现的思路很简单，将每个 trace id 哈希到 [0, 1]，如果哈希值小于给定的抽样系数，则通过；大于则拦截。在实践中，额外的抽样给与 Dapper 团队更强的全局控制力。

# Where

## General-Purpose Tools

### Dapper Depot API

Dapper 向开发者开放一系列 API：

- Access by trace id
- Bulk access
- Indexed access

在实践中，Dapper 发现 (service_name, host_machine, timestamp) 的联合索引恰好能满足大部分开发者的需求。

### Dapper user interface

dapper user interface 可以理解为 APM 系统，方便开发者快速对线上问题做根源分析。交互界面和 user story 详见论文。

## Experiences

### Using Dapper during development

Dapper 主要在以下几个方面帮助开发者改进服务：

- Performance：通过调用链示意图发现服务瓶颈
- Correctness：dapper 通过标签帮助开发者发现一些本该访问 master 节点的请求访问了 replicas
- Understanding：dapper 帮助开发者理解自身系统的依赖，增进对复杂系统的理解，为架构层面优化提供依据
- Testing：开发者可以通过观察调用链信息测试系统的行为是否符合预期

### Addressing long tail latency

Google 内部的一位工程师利用 Dapper 提供的接口来推断服务的关键路径，进而减少服务整体时延。

### Inferring service dependencies

服务之间的依赖关系常常是动态变化的，我们基本无法通过扫描配置信息、代码来确定服务之间的依赖关系。 Google 的 "Service Dependencies" 项目正是利用 Dapper 的近期数据来推断依赖关系。

### Network usage of different services

在发现网络带宽使用异常时，Dapper 可以辅助开发者锁定到具体的请求。

### Layered and Shared Storage Systems

一些公共服务通常很难知道其调用方及各自调用量，Dapper 帮助这些公共服务的维护者更好地了解它们。

# References

[Dapper](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf)

