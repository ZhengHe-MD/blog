---
title: 调用链追踪系统的设计维度
date: 2020-12-20 23:03:14
category: 系统设计
---

本文将调用链追踪系统的设计维度归结于以下 5 个：调用链数据模型、元数据结构、因果关系、采样策略以及数据可视化。我们可以把这 5 个维度当作一个分析框架，用它帮助我们在理论上解构市面上任意一个调用链追踪系统，在实践中根据使用场景进行技术选型和系统设计。如果你对调研相关系统很感兴趣，也欢迎参与到 [Database of Tracing Systems](https://github.com/ZhengHe-MD/database-of-tracing-systems) 项目中，一起调研市面上的项目，建立起调用链追踪系统的数据库。

<!-- more -->

# 引言

阅读本文并不要求读者具备任何调用链追踪系统相关的理论知识或实践经验，读者具备一定的微服务架构的概念或实践经验即可。期望读者看完这篇文章以后，能掌握调用链追踪系统的核心设计维度，理解其中的设计权衡，并能使用这些维度来分析市面上的新老调用链追踪系统实现，甚至帮助到自己在生产实践中根据使用场景进行技术选型和系统设计。

# 解决的问题

## 微服务的可观测性

> Any organization that designs a system (defined broadly) will produce a design whose structure is a copy of the organization's communication structure.
>
> — Melvin E. Conway

如果有一门学科叫软件社会学，那么康威定律 (Conway's law) 必定是其中的基本定律之一。如果把互联网公司内部的全体信息系统看作是一整个系统，那么这个系统模块结构会向公司的组织架构收敛。从组织架构层面看，公司结构从扁平向多层级演变，信息传递的环节增加，沟通效率随之下降，进而影响公司的行动效率。不论从组员之间的熟悉程度还是从部门目标一致性来看，部门内部的沟通效率要远远高于部门间的沟通效率。因此，如果系统模块结构与组织架构约趋近，公司的沟通效率就能接近极大值。团队的分化通常伴随着服务的拆分，这也是许多公司业务增长以后进行微服务化的动机。微服务化后，公司信息系统就被迫成为了分布式系统。尽管分布式系统带来了种种好处，如持续集成、增量部署、横向扩展、故障隔离，但系统可观测性比起单机系统下降了很多，甚至几乎没有人能够对公司信息系统有全局性的了解。

任意一个分布式系统的终极理想是：“给开发者以分布式的能力，单机的感受”。而调用链追踪系统就是实现终极理想不可或缺的一部分。调用链追踪系统通过收集调用链数据，帮助开发者在观测分布式系统行为时，从以机器为中心 (machine-centric) 走向以请求为中心 (workflow-centric)。调用链 (traces)、日志 (logs)、监控指标 (metrics)，三者合称 Telemetry，有了它们，微服务开发者既能通盘考虑，又能深入局部分析，在系统规模扩大的同时仍然能够掌控全局。

## 使用场景

当系统的调用链信息被串联起来以后，开发者就能基于此展开各种形式的系统行为分析。常见的使用场景可以分为以下几类：

### 异常检测

异常检测指的是定位和排查一些引发系统异常行为的请求。通常这些请求的出现频率通常很低，其出现属于小概率事件。尽管异常事件被采样的概率很低，但它的信息熵大，能给到开发者更多细节信息。这些细节可能体现在：慢请求、慢查询、循环调用未设上限、存在错误级别日志、未覆盖测试的问题逻辑分支等等。如果调用链追踪系统能主动为开发者发现异常问题，将使得风险隐患提前暴露，并被扼杀在摇篮中。

### 稳态分析

稳态分析指的是分析微服务在正常流量下的各方面状态，分析粒度可能包括单个接口、单个服务、多个服务等等；分析范围可能是单个请求或多个请求；分析角度可能包括埋点指标、依赖关系、流量大小等等。稳态分析通常反映的是系统主要流程的健康状态，一些配置的改动，如存储节点修改、客户端日志上报频率，都可能反馈到系统稳态。稳态分析还可以有很多细分场景，如：

* 稳态性能分析：定位和排查系统稳态中的性能问题，这些问题的起因通常与异常检测类似，只是其影响尚不足以触发报警。
* 服务依赖分析：构建接口级别的依赖关系图，节点通常为接口或服务，边通常为接口或服务的调用关系，边的权重则为流量。构建方式可以分为离线构建和在线构建，对应的就是静态关系图和动态关系图。这些信息可以以基础 API 的方式提供给上游应用使用，如 APM。

### 分布式侧写 (profiling)

许多编程语言都提供侧写工具，如 go tool pprof，能通过采集不同资源的使用负载，如 CPU、内存、协程等，分析进程内部不同模块的资源使用模式，最后通过调用树或火焰图等可视化方式呈现给开发者。分布式侧写就是这类侧写工具的分布式版本，开发者通过打开侧写开关，采样分析一段时间，得到微服务之间的资源占用比例，如时延，然后通过类似单机的数据可视化方式分析接口或服务整体的性能瓶颈。

### 资源归因

资源归因解答的主要问题是：“谁该为我的服务成本买单？” 它需要将资源消耗或占用与请求方关联，资源归因也是成本分析的基础。

### 负载建模

负载建模主要指分析和推测系统的行为表现，该场景解答的问题通常可以表述为 “如果出现 XX 变化，系统整体或关键链路状态会发生什么改变？” 常见应用如容量预估、全链路压测、混沌测试等等。

# 基本实现方案

## 如何追踪调用链

在微服务架构下，每个调用链的信息散落在请求经过的各个微服务中，这些信息需要通过某种技术手段收集并串联起来，重建出完整调用链。存在两种基本思路来解决问题，一种是无代码入侵的黑盒法 (blackbox)；另一种是有代码入侵的元数据传播法 (metadata propagation)。

### 黑盒法

黑盒法，顾名思义，就是将整个微服务集合看作一个黑盒，把特定格式的日志收集到存储中心后，利用统计方法推断、重建调用链：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/blackbox.png" width="600px"/>



这种方式的优势就是无代码入侵，但劣势也很明显：推断结果不准确，具体表现在：

* 难以推测异步计算任务关系
* 统计分析需要一定量的数据以及计算资源，耗时长
* 统计分析需要处理数据集不均衡的情况，如请求量很少的接口
* ...

黑盒法在生产实践中并未真正被使用，目前仅作为理论思路，提供参考。

### 元数据传播法

元数据传播法则是将调用链中必要的元数据注入到微服务之间的通信消息中，然后每个服务负责将自己记录的一部分调用链信息上报，这些信息中包含调用链标识、上游服务等信息，最后由后端系统利用这些信息重建调用链。示意图如下：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/metadata-propagation.png" width="600px"/>



元数据传播法与黑盒法正好相反，优势在于调用链重建结果准确，劣势在于有代码入侵。但这些代码埋在统一的微服务治理框架中，避免暴露给一线业务开发。几乎所有生产实践中的调用链追踪系统采用的都是元数据传播法。

## 调用链追踪系统基本架构

尽管市面上存在各式各样的调用链追踪系统，但它们的基本架构相对一致：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/basic-architecture.png" width="720px"/>



### 埋点

每个微服务会在跨进程的连接处埋点 (instrumentation)，如：

* 发送 HTTP/RPC 请求，接收 HTTP/RPC 响应
* 数据库查询
* 缓存读写
* 消息中间件的生产及消费
* ...

每个点上会记录跨进程操作的名称、开始时间、结束时间以及一些必要的标签键值对，这些信息是整个调用链拼图中的一片。

### 采样

实践中无论从计算和存储资源成本消耗上分析，还是从具体使用场景出发，都不一定需要收集所有埋点数据。因此许多调用链追踪系统会要求按照一定的策略上报数据，目的是取得成本与收益之间的平衡，提高投入产出比。

### 上报

数据可以从服务实例中直接发送到处理中心，或经由同一宿主机上的 agent 代理上报。使用 agent 上报的好处之一在于一些计算操作可以在 agent 中统一处理，一些逻辑如压缩、过滤、配置变更等可以集中到 agent 中实现，服务只需要实现很薄的一层埋点、采样逻辑即可，这也能使得调用链追踪方案对业务服务本身的影响降到最低；使用 agent 上报的另一好处是数据处理服务的发现机制对服务本身透明。因此在每台宿主机上部署 agent 是许多调用链追踪系统的推荐部署方案。

### 处理

调用链数据上报到处理中心，通常称后者为收集器 (collector)，由收集器完成必要的后处理，如数据过滤、数据标记、尾部采样、数据建模等等，最后批量写到不同的存储服务中，并建立必要的索引。

### 存储/索引

调用链追踪数据主要有两个特点：体量大、价值随时间的推移而降低。因此存储服务的选型除了数据模型之外，还需要考虑可扩展性以及数据保留策略 (retention policy) 的支持。另外为了便于查询，我们还需要为数据的存储建立合适的索引。

### 可视化

可视化是高效利用调用链数据的最重要一环，高质量的交互体验能帮助研发快速获取所需信息。通常我们可以将可视化分为两种粒度：单个调用链查看、多个调用链聚合分析，在每个粒度上都有许多可视化方案选择。

### 可扩展性

如果不做任何采样，调用链追踪系统需要处理的数据与全站的请求总量正相关。假如全站所有请求平均要经过 20 个服务处理，那么调用链追踪系统将需要承担全站请求总量 20 倍压力，因此其架构设计上的每一层都需要具备可扩展性。

如果采用服务 SDK 直接上报，那么上报层的横向扩容就自动地通过实例的增加实现；如果采用 agent 代理的上报形式，那么横向扩容就可以通过增加宿主机来实现。数据处理层理论上应该是无状态的，支持横向扩容。由于许多调用链数据的处理逻辑需要获取同一调用链的所有数据，那么通过 TraceID 做负载均衡是天然的选择；数据的存储可扩展性会由所使用的存储服务保证。

### 过载控制

瞬时高峰是常见的流量负载模式，因此调用链追踪系统的各个组件也需要考虑过载控制逻辑。既要防止在峰值流量下埋点及上报对在线服务的影响，也需要考虑调用链追踪后端各模块的承载能力。

在数据上报和处理的过程中，agent 或 collector 可以通过维持本地队列来削峰，但如果超出局部队列的容量限制，就要考虑数据丢失与时效性之间的权衡。如果可以容忍数据丢失，就可以像路由器丢包似的直接丢掉无法处理的数据；如果可以放弃峰值时效性，则可以通过高吞吐、存储容量高的消息中间件，如 Kafka，来代替局部队列。

# 设计维度

在 2014 年的一篇[论文](https://www.pdl.cmu.edu/PDL-FTP/SelfStar/CMU-PDL-14-102.pdf)中，研究团队在分析多个当时的调用链追踪系统后，总结出 4 个设计维度：

* Which causal relationships should be reserved?
* How should causal relationships be tracked?
* How should sampling be used to reduce overhead?
* How should traces be visualized?

本节，我们以这篇论文为起点，介绍调用链追踪系统的 5 个设计维度：调用链数据模型、元数据结构、因果关系、采样策略、数据可视化。

## 1. 调用链数据模型

每个调用链追踪系统实现都需要为调用链数据合理建模，它们对数据模型的选择可能影响到埋点、收集、处理、查询等各个环节。其中最常见的数据模型就是 Span Model 和 Event Model。如果你对这个话题感兴趣可以阅读这篇[文章](http://cs.brown.edu/~rfonseca/pubs/leavitt.pdf)。

### Span Model

Span Model 最早由 Google 在 [Dapper](https://research.google/pubs/pub36356/) 中提出，它将一次计算 (computation) 任务，如处理用户请求，表达成一组 spans 集合，其中每个 span 表示计算任务的一部分 (segment)，记录着开始时间和结束时间。其中每个 span 还记录着触发它的 span，即 parent span，标志着系统中的因果关系。假设 `SpanA` 触发了 `SpanB`，那么 `SpanA` 就是 `SpanB` 的父节点。由于父子关系意味着因果关系，那么 spans 之间组成的关系不会形成环，否则就会出现因果循环。因此通常同一个 trace 的 spans 关系可以使用一棵树表示，举例如下：



![](/blog/2020/12/20/design-dimensions-of-tracing-systems/span-model.png)



需要注意的是：在 Span Model 中，每个 span 都只存在一个父节点，即导致某段计算发生的原因只有一个。使用 Span Model 的追踪系统在埋点时需要主动停止 span，停止之后该段计算的信息会被上报到处理中心。从逻辑上看，子节点完成上报后，父节点才会上报；从上报通路上看，二者都由本地线程上报，并无关系。

Span Model 单因多果的关系与调用栈在概念上十分契合，很容易被工程师理解和掌握。然而它并不足以表达所有类型的计算依赖关系，如多因一果：



![](/blog/2020/12/20/design-dimensions-of-tracing-systems/diamond.png)

### Event Model

[X-Trace](http://www.icsi.berkeley.edu/pubs/networking/xtrace07.pdf) 是最早使用 Event Model 的项目。在 X-Trace 中，一个事件 (event) 是计算任务中的一个时刻，计算任务中的因果关系由事件之间的边 (edges) 表示，任意两个事件都可以用一条边连接。值得注意的是，这里的 edge 表示的实际上是 [Lamport (1978)](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) 中提到的 "happens-before" 关系，假设有一条边从 EventA 连到 EventB，那么 "happens-before" 表示 EventA 可能对 EventB 产生影响。但在简单场景下，我们可以直接认为边指代激活关系 (activation relationship) 或依赖关系 (dependency relationships)，二者都是 "happens-before" 关系的子集。与 Span Model 不同的是，Event Model 中每个事件可以有多条入边 (incoming edges)，这让 Event Model 可以轻松表达复杂关系，如 fork/join 或 fan-ins/fan-outs 关系。Event Model 支持更精细化的调用链数据展示，举例如下：



![](/blog/2020/12/20/design-dimensions-of-tracing-systems/event-model.png)



其中虚线框表示某个执行线程；圆点表示事件；箭头表示边。为了便于理解和对比，图中也用实线方框表示 span。

Event Model 的优势在于表达力强，但缺点是相比 Span Model 更加复杂，对工程师来说更不易接受和上手，同时 Span Model 的类似调用栈的可视化也更加简洁。

## 2. 元数据结构

首先，为了防止歧义，这里特别指出：**元数据指的是在进程间传递的调用链追踪相关的数据**。几乎所有调用链追踪系统都采用元数据传播的方式来追踪跨进程的调用链。那么我们应该如何设计进程间传递的元数据结构？从元数据结构的内容可变性和长度限制两个维度可以将元数据结构划分为三种：静态定长、动态定长和动态变长。

### 静态定长

静态定长元数据，即数据的长度固定且在传播过程中不发生变化。静态定长元数据中只包含一个请求级别的唯一固定 ID，即 TraceID。调用链追踪系统可以通过 TraceID 获取所有与同一个请求相关的信息，然后再建立因果关系。由于元数据中只有 TraceID，系统只能借助一些外部信息，如 threadID、宿主机的墙上时钟，来推测因果关系。

### 动态定长

动态定长元数据，即数据的长度固定但在传播过程中可能发生变化。动态定长元数据中除包含 TraceID 之外，还会传递请求来源标识，如 SpanID 或 EventID，其中来源标识可以建立两两节点之间的上下游关系。

### 动态变长

动态变长元数据，即数据的长度和内容都会在传播过程中发生变化。动态变长元数据中通常包含上游所有节点的全量或部分信息，当前节点处理完毕后，会将当前节点信息及上游信息一同往下游传递。每个节点都能获取调用链到当前节点为止的所有信息，因此无需通过额外的组件重建调用链。

## 3. 因果关系

同一请求 (intra-request) 的计算任务之间可能存在因果关系，如：

* 进程 P1 通过 HTTP 或 RPC 调用进程 P2
* 进程 P1 或 P2 写入数据到存储服务中，或从存储服务中读取数据
* 进程 P1 生产消息到 MQ 中，进程 P2 消费到消息并处理
* ...

不同请求 (inter-request) 的计算任务之间也可能存在因果关系，如：

* 请求 R1 和 R2 同时获取某分布式锁，R1 成功，R2 失败
* 请求 R1 写入数据到本地缓存后请求 R2 也写入数据，同时触发批处理
* 请求 R1 写入数据到存储系统后请求 R2 读出对应数据进行处理
* ...

在实践中，开发者习惯以单个请求的视角分析问题，因此调用链追踪系统通常不会关注不同请求之间的因果关系，但会在数据模型上保持对应的表达能力。对于同一请求的计算任务之间的因果关系，通常 SDK 提供方会尽可能地帮助开发者在所有跨进程的连接点上埋点，以此达到追踪目的，如 HTTP/RPC 调用、数据库访问、消息生产和消费等。但有时候源自于 A 请求的计算任务会被 B 请求触发，如下图中的例子所示：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/submitter-and-trigger.png" width="600px" />



Request one 将数据 d1 提交到局部写回缓存 (write-back cache)，Request two 将数据 d2 提交到同一个缓存中，触发 d1 被写出到持久化存储中。这时如何归属 d1 的写出操作就决定了调用链追踪系统是选择提交者角度 (submitter-preserving) 还是触发者角度 (trigger-preserving)。

### 提交者角度

提交者角度意味着，当聚合或批处理操作被另一个请求触发时，该操作将被归属于提交者。如上方左图所示：Request one 留存在写回缓存中的数据因为 Request two 写入数据而最终被清出，此时清出数据的操作归属于 Request one。

### 触发者角度

触发者角度意味着，当聚合或批处理操作被另一个请求触发时，该操作将被归属于触发者。如上方右图所示：Request one 留存在写回缓存中的数据因为 Request two 写入数据而最终被清出，此时清出数据的操作归属于 Request two。

## 4. 采样策略

调用链数据总体体量与业务体量正相关，全量采集调用链数据将会给公司系统整体带来两方面压力：

- 因数据上报造成的每个业务服务的网络 I/O 压力
- 因数据采集、分析造成的调用链追踪服务的计算和存储压力

为了降低这两方面压力，采样是大多数调用链追踪系统的必备模块。实践中常用的采用策略可以分为三类：

- 头部连贯采样：Head-based coherent sampling
- 尾部连贯采样：Tail-based coherent sampling
- 单元采样：Unitary sampling

它们的示意图如下所示：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/sampling-methods.png" width="600px" />



### 头部连贯采样

头部连贯采样指的是请求进入系统时就立即决定是否采样，并且这个决定会随着元数据被传递到下游服务，保证采样的连贯性。由于采样决定做得早，对系统整体带来的压力较小。但也正因为决定做得早，采样的准确度也最低，很难保证采集到的调用链有价值。头部连贯采样还有一种变体，即头部连贯采样配合异常回溯上报：在头部连贯采样的同时，于每个服务节点缓存最近的若干 spans 信息，一旦下游调用出现异常，则可在微服务框架中感知同时回溯到上游节点，保证出现异常的调用链数据能被上报。

### 尾部连贯采样

尾部连贯采样指的是在请求完成时才决定是否采样。在决定之前，系统需要将数据缓存起来，以保证采样的连贯性。由于采样决定做得晚，数据需要全量上报并临时存储一段时间，这将加重上文提到的两方面压力。但也正因为决定做得晚，获取的信息更全，尾部连贯采样能利用一些经验性的规则保证重要的调用链被采集。

### 单元采样

单元采样并不要求连贯性，系统中的每个组件自行决定是否采样，因此这种方案通常无法建立单个请求的调用链信息。

## 5. 数据可视化

调用链数据的可视化通常与使用场景一一对应，高效的可视化形式能更好地赋能工程师，缩短故障排查时间，提升研发生活质量。

### 甘特图 (Gantt charts)

甘特图常被用于展示单个请求的调用链数据，以下是调用链追踪系统最常用的甘特图变体：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/gantt.png" width="480px" />



图的左边通常组织为树状结构，通常父节点表示调用方，子节点表示被调方，兄弟节点之间为并发关系，且从上至下时间单调递增；图的右边展示的是与标准甘特图类似的条状结构。

### 泳道图 (Swimlane)

泳道图可以被用于展示单个请求的调用链数据，相比甘特图更加精细，常用于 [Event Model](https://github.com/ZhengHe-MD/database-of-tracing-systems/blob/main/dimensions/design/tracing-model/README.md#event-model) 展示更复杂的计算关系，举例如下：

![](/blog/2020/12/20/design-dimensions-of-tracing-systems/swimlane.png)

其中泳道，即虚线框，用于表示计算执行单元；圆点展示某时刻发生的事件；箭头表示事件之间的关系。

### 流程图 (Flow graphs)

流程图常被用于展示多个相似请求调用链数据的聚合信息，这些请求的调用链结构应该完全一致。举例如下：

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/flow-graph.png" alt="" style="zoom:50%;" />

图中的节点表示系统中发生的事件，边表示因果关系，权重可以表示事件发生的时间差，它们共同组成一个有向无环图。流程图甚至可以表达 fan-outs 和 fan-ins，即 forks 和 joins 的因果关系，能保留更多的调用链细节信息。

### 调用图 (Call graphs)

调用图被用于展示多个请求的聚合信息，这些请求的调用链结构无需完全一致。调用图上的节点表示系统中的服务、模块或接口，边表示因果关系，权重则可以表示流量、资源占用等自定义信息。调用图中可能出现环，意味着系统中存在环形依赖。调用图示例如下：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/call-graph.png" width="480px" />



### 调用树 (Calling Context Trees)

调用树被用于展示多个请求的聚合信息，这些请求的调用链结构通常不同。调用树根节点到任意叶子节点的路径都是分布式系统中真实存在的调用路径，举例如下：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/cct.png" width="480px" />

## 火焰图 (Flame graph)

火焰图常被用于展示单机程序调用栈耗时信息，如 Go 中的 pprof。它与调用树的结构类似，常被用于展示多个请求的聚合信息，但展示形式不同，能更直观地展示各个组件的耗时信息，举例如下：

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/flame-graph.png" width="480px" />

## 从维度到场景

了解各个设计维度之后，我们一起回顾本文开头提到的场景，试着分析在这些维度上该如何选择。下面以异常检测和分布式侧写为例：

![](/blog/2020/12/20/design-dimensions-of-tracing-systems/use-case-dimension-matrix.png)

**异常检测**：某个请求出问题开发者需要查看完整调用链信息，因此需要连贯采样，又由于问题请求的发生是小概率事件，只能通过尾部连贯采样来保证数据都能被捕获。开发者习惯以从每个请求造成的影响分析问题，因此请求内部的因果关系应该选择触发者视角。甘特图、流程图都是适用单个调用链的可视化方案。元数据结构中，动态定长相对静态定长能更准确地采集上下游关系，相对动态变长能节省网络成本，且后者带来的实时性上的优化对异常检测并不重要，因此动态定长元数据是更合适的选择。

**分布式侧写**：侧写能够帮助开发者查看调用链级别的性能瓶颈问题，但分析对象是聚合的数据，对单个调用链的完整性并无要求，单元采样是成本最低的采样方案。与异常检测类似，触发者视角更符合开发者直觉，且没有额外开销，因此选择触发者视角。侧写的可视化选择毫无悬念：调用树和火焰图。元数据结构中，如果调用链深度可控，动态变长能帮助开发者更快地看到侧写数据；如果深度不可控，动态定长同样满足需求，只是在数据处理环节需要消耗计算资源。

调用链数据模型会影响各个场景的最终实现效果和能力边界，但不影响场景解决方案的有效性，因此这里没有专门讨论。如果在实践中你需要同时解决多个场景，就需要考虑在各个设计维度上取一个包集。

# 案例分析: Jaeger

## 项目历史

Jaeger 的名字源于德语中的猎人，是由 Uber 内部 Observability 团队开发，集成埋点、收集到可视化的完整调用链追踪解决方案。2017 年 4 月 Jaeger 正式开源；2017 年 9 月进入 CNCF 孵化；2019 年 10 月正式从 CNCF 毕业，成为 CNCF 顶级项目。

## 基本架构

Jaeger 的架构与上文提到的调用链追踪系统的基本架构十分类似，它有两种部署架构选择，分别如下面两张图所示：



<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/jaeger-architecture-1.png" width="600px" />

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/jaeger-architecture-2.png" width="600px" />

二者结构大致相同，主要区别在于 jaeger-collector 与 DB 之间加了 Kafka 做缓冲，解决峰值流量过载问题。整个 Jaeger 后端不存在单点故障，Jaeger-collector、Kafka、DB (Cassandra 和 ElasticSearch) 都支持横向扩展。

## 使用场景: 稳态分析

Jaeger 在官网上介绍自己的主要功能如下：

* 分布式上下文传播 (Distributed context propagation)
* 分布式事务监控 (Distributed transaction monitoring)
* 根因分析 (Root cause analysis)
* 服务依赖分析 (Service dependency analysis)
* 性能/时延优化 (Performance/latency optimization)

重建调用链关系需要在进程间传播元数据，因此分布式上下文传播其实是实现调用链追踪数据建模的基础，我们通常不会使用它来传播非调用链追踪相关的数据，如 uid、did 等等。这些数据一般会通过微服务治理框架来传播。后面的分布式事务监控、根因分析、服务依赖分析、性能/时延优化，主要是通过采集侧收集上来的调用链数据及服务 (service)、操作 (operation) 的依赖关系，分析系统行为。

## 调用链数据模型: Span Model

Jaeger 中调用链数据模型遵守了 opentracing 标准，使用的是典型的 Span Model，其核心数据结构如下图所示：

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/data-model.png" width="780px" />

下面是一个具体的例子：

```
# source: https://github.com/opentracing/specification/blob/master/specification.md
Causal relationships between Spans in a single Trace


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)
```

其中 Span 与 Span 之间存在两种因果关系，ChildOf 和 FollowsFrom。ChildOf 关系中，父节点依赖于子节点执行的结果；FollowsFrom 关系中，父节点不依赖于子节点执行的结果，但与之存在因果关系。

## 因果关系: 用户决定，触发者视角为主

Jaeger 采用的调用链数据模型完全能够关联同一个请求中的不同进程，是提交者视角还是触发者视角则取决于 Jaeger 的接入方，选择触发者视角对接入方不存在额外的成本，而选择提交者视角则需要接入方投入额外的精力做定制化开发。因此在绝大多数情况下使用的是触发者视角。

## 元数据结构: 动态定长

Jaeger 在进程间传递的元数据结构如下：

```go
// source: https://github.com/jaegertracing/jaeger-client-go/blob/master/span_context.go
// SpanContext represents propagated span identity and state
type SpanContext struct {
	// traceID represents globally unique ID of the trace.
	// Usually generated as a random number.
	traceID TraceID
	// spanID represents span ID that must be unique within its trace,
	// but does not have to be globally unique.
	spanID SpanID
	// parentID refers to the ID of the parent span.
	// Should be 0 if the current span is a root span.
	parentID SpanID
	// Distributed Context baggage. The is a snapshot in time.
	baggage map[string]string
	// debugID can be set to some correlation ID when the context is being
	// extracted from a TextMap carrier.
	//
	// See JaegerDebugHeader in constants.go
	debugID string
	// samplingState is shared across all spans
	samplingState *samplingState
	// remote indicates that span context represents a remote parent
	remote bool
}
```

利用 traceID 可以确认当前 span 的归属关系；利用 spanID 和 parentID 可以建立上下游进程的父子关系。通常 baggage 中的数据量不会变化。综合考虑：Jaeger 的元数据结构属于动态定长。

## 采样策略: 头部连贯采样

目前 Jaeger 支持三种采样方式：

* Const：要么全采样，要么不采样
* Probabilistic：按固定概率采样
* Rate Limiting：限流采样，即保证每个进程每隔一段时间最多采 k 个

除了在 sdk 初始化时直接写死采样配置外，Jaeger 还支持远程动态调整采样方式，但调整的选择范围仍然必须为上面三种之一。为了防止一些调用量小的请求因为出现概率低而无法获得调用链信息，Jaeger 团队也提出了适应性采样 (Adaptive Sampling) ，但这个提议从 2017 年至今仍然未有推进。

无论是上述哪种方式，是否采样这一决定都是在请求进入系统之时决定，因此结论是：目前 Jaeger 支持头部连贯采样。值得一提的是，Jaeger 团队也在讨论[引入尾部连贯采样的可能性](https://github.com/jaegertracing/jaeger/issues/425)，但尚未有实质性的进展。

## 数据可视化: 甘特图、调用树、调用图

jaeger-ui 项目提供了丰富的调用链数据可视化支持，包括针对单个请求的甘特图、调用树，以及全局服务的调用图。

### 甘特图

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/gantt.png" width="480px" />

### 调用树

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/cct.png" width="480px" />

调用树目前仍在实验阶段，暂时还不是正式功能。

### 调用图

<img src="/blog/2020/12/20/design-dimensions-of-tracing-systems/call-graph.png" width="480px" />

同时还可以聚焦到某个节点，让调用图只显示与该节点相关的服务，即焦点图 (focus graph)。

# 调用链追踪系统数据库

[AP](http://www.cs.cmu.edu/~pavlo/) 在 2014 年建立了网站 [dbdb.io](https://dbdb.io/)，即 Database of Databases，从一些固定的维度来分析市面上琳琅满目的数据库系统。受这个项目启发，我们也可以用本文提到的设计维度，来分析市面上的调用链追踪系统，从而获得更加系统化的理解，并将分析调研的结果沉淀下来。于是我就建立了这个项目 [Database of Tracing Systems](https://github.com/ZhengHe-MD/database-of-tracing-systems)，如果你对此感兴趣，欢迎参与调研，共同建立调用链追踪系统的数据库。

# 参考资料

* [Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://research.google/pubs/pub36356/)
* [So, you want to trace your distributed system? Key design insights from years of practical experience](https://www.pdl.cmu.edu/PDL-FTP/SelfStar/CMU-PDL-14-102.pdf)
* [Canopy: An End-to-End Performance Tracing And Analysis System](https://research.fb.com/wp-content/uploads/2017/10/sosp17-final14.pdf)
* [End-to-End Tracing Models: Analysis and Unification](http://cs.brown.edu/~rfonseca/pubs/leavitt.pdf)
* [Tracing, Fast and Slow](https://www.roguelynn.com/words/tracing-fast-and-slow/)
* [Github: Database of tracing systems](https://github.com/ZhengHe-MD/database-of-tracing-systems)
* [Database of Databases](https://dbdb.io/)
* [X-Trace: A Pervasive Network Tracing Framework](http://www.icsi.berkeley.edu/pubs/networking/xtrace07.pdf)
* [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
* [Evolving Distributed Tracing at Uber Engineering](https://eng.uber.com/distributed-tracing/)
* [Jaeger Docs](https://www.jaegertracing.io/docs/)
* [OpenTracing Specification](https://github.com/opentracing/specification/blob/master/specification.md)

