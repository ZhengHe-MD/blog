---
title: 调用链追踪系统在伴鱼：实践篇
date: 2021-03-04 17:03:51
tags:
categories:
- system design
---

> 此文同时发表在[伴鱼技术博客](https://tech.ipalfish.com/blog/2021/03/04/implementing-tail-based-sampling/)上

在 [理论篇](https://zhenghe-md.github.io/blog/2020/12/20/design-dimensions-of-tracing-systems/) 中，我们介绍了伴鱼在调用链追踪领域的调研工作，本篇继续介绍伴鱼的调用链追踪实践。在正式介绍前，简单交代一下背景：2015 年，在伴鱼服务端起步之时，技术团队就做出统一使用 Go 语言的决定。这个决定的影响主要体现在：

1. 内部基础设施无需做跨语言支持
2. 技术选型会有轻微的语言倾向

## 1. 早期实践

### 1.1 对接 Jaeger

2019 年，公司内部的微服务数量逐步增加，调用关系日趋复杂，工程师做性能分析、问题排查的难度变大。这时亟需一套调用链追踪系统帮助我们增强对服务端全貌的了解。经过调研后，我们决定采用同样基于 Go 语言搭建的、由 CNCF 孵化的项目 [Jaeger](https://www.jaegertracing.io/)。当时，服务的开发和治理都尚未引入 `context`，不论进程内部调用还是跨进程调用，都没有上下文传递。因此早期引入调用链追踪的工作重心就落在了服务及服务治理框架的改造，包括：

* 在 HTTP、Thrift 和 gRPC 的服务端和客户端注入上下文信息
* 在数据库、缓存、消息队列的接入点上埋点
* 快速往既有项目仓库中注入 `context` 的命令行工具

部署方面：测试环境采用 **all-in-one**，线上环境采用 [**direct-to-storage**](https://www.jaegertracing.io/docs/1.22/architecture/) 方案。整个过程前后大约耗时一个月，我们在 2019 年 Q3 上线了第一版调用链追踪系统。配合广泛被采用的 prometheus + grafana 以及 ELK，我们在微服务群的可观测性上终于凑齐了调用链 (traces)、日志 (logs) 和监控指标 (metrics) 三个要素。

下图是第一版调用链追踪系统的数据上报通路示意图。服务运行在容器中，通过 opentracing 的 sdk 埋点，Jaeger 的 go-sdk 上报到宿主机上的 Jaeger-agent，后者再将数据进一步上报到 Jaeger-collector，最终将调用链数据写入 ES，建立索引，即图中的 Jaeger backends。

<img src="./distributed-tracing-v1.png" alt="distributed-tracing-v1" style="zoom:50%;" />

### 1.2 遇到的问题

在 [理论篇](https://tech.ipalfish.com/blog/2020/12/29/design-dimensions-of-tracing-systems/) 中，我们介绍过 Jaeger 支持三种采样方式：

* Const：要么全采样，要么不采样
* Probabilistic：按固定概率采样
* Rate Limiting：限流采样，即保证每个进程在固定时间内最多采 k 个

这些采样方式都属于头部连贯采样 (head-based coherent sampling)，我们在 [理论篇](https://tech.ipalfish.com/blog/2020/12/29/design-dimensions-of-tracing-systems/) 中曾讨论过其优劣势。伴鱼的生产环境中使用的是限流采样策略：每个进程每秒最多采 1 条 trace。这种策略虽然很节省资源，但其缺点在一次次线上问题排查中逐渐暴露：

1. 一个进程中包含多个接口：不论按固定概率采样还是限流采样，都会导致**小流量接口一直采集不到调用链数据而饿死 (starving)**。
2. 线上服务出错是小概率事件，导致出错的请求被采中的概率更小，就导致**采到的调用链信息量不大，引发问题的调用链却丢失**的问题。

## 2. 调用链通路改造

### 2.1 使用场景

2020 年，我们不断收到业务研发的反馈：**能不能全量采集 trace？**

这促使我们开始重新思考如何改进调用链追踪系统。我们做了一个简单的容量预估：目前 Jaeger 每天写入 ES 的数据量接近 100GB/天，如果要全量采集 trace 数据，保守假设平均每个 HTTP API 服务的总 QPS 为 100，那么完整存下全量数据需要 10TB/天；乐观假设 100 名服务器研发每人每天查看 1 条 trace，每条 trace 的平均大小为 1KB，则整体信噪比千万分之一。可以看出，这件事情本身的 ROI 很低，考虑到未来业务会持续增长，存储这些数据的价值也会继续降低，因此全量采集的方案被放弃。退一步想：全量采集真的是本质需求吗？实际上并非如此，我们想要的其实是**「有意思」的 trace 全采，「没意思」的 trace 不采**。

根据 [理论篇](https://tech.ipalfish.com/blog/2020/12/29/design-dimensions-of-tracing-systems/) 中介绍的应用场景，实际上第一版调用链追踪系统只支持了稳态分析，而业务研发亟需的是异常检测。要同时支持这两种场景，我们必须借助尾部连贯采样 (tail-based coherent sampling)。相对于头部连贯采样在第一个 span 处就做出是否采样的决定，尾部连贯采样可以让我们在获取 (接近) 完整的 trace 信息后再做出判断。理想情况下，只要能合理地制定采样的判断逻辑，我们的调用链追踪系统就能做到「有意思」的 trace 全采，「没意思」的 trace 不采。

### 2.2 架构设计

Jaeger 团队从 2017 年就开始[讨论](https://github.com/jaegertracing/jaeger/issues/425)引入 tail-based sampling 的可能性，但至今未有定论。在一次与 Jaeger 工程师 [jpkrohling](https://github.com/jpkrohling) 的一对一沟通中，对方也证实了目前 Jaeger 尚没有相关的支持计划。因此，我们不得不另辟蹊径。经过一番调研，我们找到了刚刚进驻 CNCF SandBox 的 [OpenTelemetry](https://github.com/open-telemetry)，它的 [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector) 子项目恰好能支持我们在现有架构上引入尾部连贯采样。

#### 2.2.1 OpenTelemetry Collector

整个 OpenTelemetry 项目目的在于统一市面上可观测性数据 (telemetry data) 的标准，同时提供推动这些标准实施的组件和工具。opentelemetry-collector 就是这个生态中的一个重要组件，它的架构图如下：

<img src="./open-telemetry-collector-architecture.png" alt="otelcol-architecture" style="zoom:40%;" />

collector 内部有 4 个核心组件：

* Receivers：负责接收不同格式的 telemetry data，对于 trace 来说就是 Zipkin、Jaeger、OpenCensus 以及其自研的 OTLP。除此之外，还可以支持从 Kafka 中接收以上格式的数据。
* Processors：负责实施处理逻辑，如打包、过滤、修饰、采样等等，尾部采样逻辑就可以在这里实现。
* Exporters：负责将处理后的 telemetry data 按指定的格式重新输出到后端服务中，如 Zipkin、Jaeger、OpenCensus 的 backend，也可以输出到 Kafka 或另一组 collector 中。
* Extensions：提供一些核心流程之外的插件，如分析性能问题的 pprof，健康监测的 health 等等。

利用不同的 Receivers、Processors、Exporters，我们可以组装出一个或多个 Pipelines。

opentelemetry-collector 项目被拆成两部分：主项目 [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector) 和社区贡献项目 [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)，前者负责管理核心逻辑、数据结构以及通用的 Receivers、Processors、Exporters、Extensions 实现，后者则负责接收一些社区贡献的组件，当然贡献者主要都来自于可观测性 SaaS 解决方案供应商，如 DataDog、Splunk、LightStep 以及一些公有云厂商。opentelemtry-collector 项目的插件化管理方式，使得**定制化开发 Receiver、Processor、Exporter 的成本很低**，我们在做概念验证时，基本可以在一两个小时内开发并测试完毕一个组件。除此之外，opentelemetry-collector-contrib 还提供了开箱即用的 [tailsamplingprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)。

由于 opentelemetry-collector 是 OpenTelemetry 团队推动标准实施的重要组件，且 OpenTelemetry 本身并没有提供独立的数据存储、索引实现的计划，因此它在与市面上流行的调用链追踪框架，如 Zipkin、Jaeger、OpenCensus，的兼容性上下了很大功夫。通过 Receivers 和 Exporters 的使用，我们可以用它替换 jaeger-agent，也可以放在 jaeger-agent 与 jaeger-collector 之间，必要时还可以在 jaeger-agent 和 jaeger-collector 之间部署多层 collectors。除此之外，如果有一天想换掉 jaeger-backend，比如新发布的 [Grafana Tempo](https://grafana.com/oss/tempo/)，我们也能很轻松的完成，并且利用多个 Pipelines 或一个 Pipeline 多个 Exporters 灰度上线，逐步替换。

#### 2.2.2 上报通路

基于以上的调研和现有的架构，我们设计了第二版调用链追踪的架构，如下图所示：

<img src="./distributed-tracing-v2.png" alt="distributed-tracing-v2" style="zoom:45%;" />

用一组 opentelemetry-collector 替换 jaeger-agent，即图中的 otel-agent；同时在 otel-agent 与 jaeger-collector 之间增加另一组 opentelemetry-collector，即图中的 otel-collector。otel-agent 收集宿主机上不同服务上报的 trace 数据，打包批量发送给 otel-collector，后者负责做尾部连贯采样，将**「有意思」**的 trace 继续输出到原始架构中的 jaeger-collector，后者负责将数据投入 ElasticSearch 中，建立索引。

这里还有一个问题需要解决：整个架构要做到高可用，势必要部署多个 otel-collector 实例，如果使用简单的负载均衡策略，不同 otel-agents、以及单个 otel-agent 不同时刻上报的数据，可能被随机上报到某个 otel-collector 实例，这就意味着，同一个 trace 的不同 spans 无法集中到同一个 otel-collector 实例上，既会导致同一个 trace 的不同 spans 的决定不一致，即不是连贯采样；也会导致尾部采样时提供判断的数据不全。解决方案很简单：**让 otel-agent 按照 traceID 做负载均衡**。

在调研阶段我们正好看到 opentelemetry-collector 社区也有此支持[计划](https://github.com/open-telemetry/opentelemetry-collector/issues/1724)，并且前面提到的工程师 jpkrohling 正在通过增加 [loadbalancingexporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter) 解决它，虽然直接使用 bleeding edge 的版本有一定的风险，我们还是决定尝试。在概念验证阶段，我们也的确发现了新功能的若干问题，但通过反馈 ([#1621](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/1621)) 和贡献 ([#1677](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/1677)、[#1690](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/1690)) 的方式一一解决，最终获得了可以按预期执行尾部连贯采样的调用链追踪系统。

### 2.3 采样规则

尾部连贯采样的数据通路已经准备就绪，下一步就是确定和实施采样规则。

#### 2.3.1 「有意思」的调用链

什么是「有意思」的调用链？研发在分析、排障过程中想查询的任何调用链就是「有意思」的调用链。但落实到代码的必须是确定性的规则，根据日常排障经验，我们先确定了以下三种情形：

* 在调用链上打印过 ERROR 级别日志
* 在调用链上出现过大于 200ms 的数据库查询
* 整个调用链请求耗时超过 1s

满足任意条件，就认为这个调用链「有意思」。在伴鱼，只要服务打印了 ERROR 级别的日志就会触发报警，研发人员就会收到 im 消息或电话报警，如果能保证触发报警的调用链数据必采，研发人员的排障体验就会有很大的提升；我们的 DBA 团队认为超过 200ms 的查询请求都被判定为慢查询，如果能保证这些请求的调用链必采，就能大大方便研发排查导致慢查询的请求；对于在线服务来说，时延过高会令用户体验下降，但具体高到什么程度会引发明显的体验下降我们暂时没有数据支撑，因此先配置为 1s，支持随时修改阈值。

当然，以上条件并不绝对，我们完全可以在之后的实践中根据反馈调整、新增规则，如单个请求引起的数据库、缓存查询次数超过某阈值等。

#### 2.3.2 采样流水线

在第二版系统中，我们期望同时支持稳态分析与异常检测，因此采样过程既要按概率或限流方式采集一部分 trace，也需要按上文拟定的「有意思」规则采集令一部分 trace。截止到本文撰写前，[tailsamplingprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor) 支持 4 种策略：

* `always_sample`：全采样
* `numeric_attribute`：某数值属性位于 `[min_value, max_value]` 之间
* `string_attribute`：某字符串属性位于集合 `[value1, value2, ...]` 之中
* `rate_limiting`：按照 spans 数量限流，由参数 `spans_per_second` 控制

其中可以为我们所用的有 `numeric_attribute`、`string_attribute` 以及 `rate_limiting`。

「按概率或限流采集一部分 trace」可以利用 `rate_limiting` 实现吗？`rate_limiting` 只能按照每秒通过的 spans 个数限流，但 spans 数量在高峰期、低峰期、业务所处阶段都不一样，每个 trace 的平均 spans 数量也会随着微服务群规模以及依赖关系发生变化，因此设置 `spans_per_second` 将让我们很难对这个参数的最终效果合理评估，因此直接使用 `rate_limiting` 的方案被否决。

「按规则采集另一部分 trace」可以直接使用 `numeric_attribute` 和 `string_attribute` 实现吗？span 中还存在布尔 (bool) 类型的 tag，以「在调用链上如果打印了 ERROR 级别日志」为例，按照规范我们会记录 `span.SetTag("error" , true)`，但 tailsamplingprocessor 并未支持 `bool_attribute`；此外，未来我们可能会有更复杂的组合条件，这时仅靠 `numeric_attribute` 和 `string_attribute` 也无法实现。

经过再三分析，我们最终决定利用 Processors 的链式结构，组合多个 Processor 完成采样，流水线如下图所示：



<img src="./sampling-pipeline.png" alt="sampling-pipeline" style="zoom:50%;" />



其中 `probattr` 负责在 trace 级别按概率抽样，`anomaly` 负责分析每个 trace 是否符合「有意思」的规则，如果命中二者之一，trace 就会被打上标记，即 `sampling.priority`。最后在 `tailsamplingprocessor` 上配置一条规则即可，如下所示：

```yaml
 tail_sampling:
    policies:
      [
        {
          name: sample_with_high_priority,
          type: numeric_attribute,
          numeric_attribute: { key: "sampling.priority", min_value: 1, max_value: 1 }
        }
      ]
```

这里 `sampling.priority` 是整数类型，当前取值只有 0 和 1。按上面的配置，所以 `sampling.priority = 1` 的trace 都会被采集。后期可以增加更多的采集优先级，在必要的时候可以多采样 (upsampling) 或降采样 (downsampling)。

## 3. 部署实施

采样规则确立后，整个解决方案就已跑通，下一步就是进入部署实施阶段。

### 3.1 上线准备

#### 3.1.1 基础库改造

##### 动态更新 Tracer

在第一版系统中，每个进程启动时会从 [apollo](https://github.com/ctripcorp/apollo) 上获取采样配置，传递给 Jaeger sdk，后者利用配置初始化 GlobalTracer。GlobalTracer 会在 trace 的第一个 span 出现时，决定是否采集，并把这个决定传递下去，即头部连贯采样。在实施新架构时，我们需要 Jaeger sdk 将所有 trace 数据尽数上报。为了让这个过程更加平滑，我们对 Jaeger sdk 配置做出两方面改造：

* 支持对每个服务下发不同的采样配置，方便灰度发布
* 支持动态更新采样配置，使采样策略配置不依赖发布

##### 日志库改造

为了保证打印过 ERROR 级别日志的调用链必采，我们也在通用日志库的相应位置上给 span 打上 error 标签。

#### 3.1.2 监控看板配置

opentelemetry-collector 内部利用 OpenCensus sdk 埋了很多有用的监控指标，并按照 [Open Metrics](https://github.com/OpenObservability/OpenMetrics/blob/master/specification/OpenMetrics.md) 规范暴露数据。因为指标数量并不多，我们将大多数指标都配置了到 Grafana 看板中，包括：

* Receivers：[obsreport_receiver](https://github.com/open-telemetry/opentelemetry-collector/blob/68c5961f7bc2d8d55fd431ab07367fc16f8b26d0/obsreport/obsreport_receiver.go)
* Processors： [obsreport_processor](https://github.com/open-telemetry/opentelemetry-collector/blob/68c5961f7bc2d8d55fd431ab07367fc16f8b26d0/obsreport/obsreport_processor.go), [tailsamplingprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/metrics.go)
* Exporters：[obsreport_exporter](https://github.com/open-telemetry/opentelemetry-collector/blob/68c5961f7bc2d8d55fd431ab07367fc16f8b26d0/obsreport/obsreport_exporter.go)

下面介绍几个在实践中我们认为比较重要的指标：

##### xxx_receiver_accepted/refused_spans

这里的 xxx 指代任意一个 pipeline 中使用的 receiver。实际上这里有两个具体指标：receiver 接收的 spans 数量和 receiver 拒绝的 spans 数量。二者可以与其它指标结合，分析系统在当前状况下的入口流量瓶颈。

##### xxx_exporter_send(failed)_spans

这里的 xxx 指代任意一个 pipeline 中使用的 exporter。实际上这里有两个具体指标：exporter 发送成功的 spans 数量和 exporter 发送失败的 spans 数量。二者可以与其它指标结合，分析系统在当前状况下的出口流量瓶颈。

##### otelcol_processor_tail_sampling_sampling_trace_dropped_too_early

要介绍上面这个指标，需要简单了解 tailsamplingprocessor 的工作原理。在分布式环境中，tailsamplingprocessor 永远无法确定一个 trace 的所有 spans 在当前时刻是否收集完毕，因此需要设置一个超时时间，即 [这里](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor) 的 `decision_wait`，下面假设 `decision_wait = 5s`。Trace Data 进入 processor 后，会被分别放入两个数据结构：

<img src="./tailsamplingprocessor-traces.png" alt="tailsamplingprocessor-traces" style="zoom:50%;" />

一个固定大小的队列和一个哈希表，二者合起来实现 trace data 的 LRU cache。同时 processor 会将所有进入到其中的 traces 按照每秒一个 batch 组织起来，内部一共维持 5 个 batch (`decision_wait`)。每隔一秒钟，将最老的 batch 取出来，对其中的 traces 分别判断是否符合采样规则，符合则将其传递给后续的 processors：

<img src="./tailsamplingprocessor-ticker.png" alt="tailsamplingprocessor-ticker" style="zoom:50%;" />

如果在做采样决策时，发现相应的 trace 已经被 LRU cache 清出，则认为「trace dropped too early」，后者意味着 tailsamplingprocessor 已经超负荷。理论上这个指标如果不等于 0，尾部连贯采样功能就处于异常状态。

### 3.2 灰度方案

上文提到过，实施改造需要让 Jaeger sdk 全量上报 trace。由于「是否上报」这个决定在请求入口服务 (即 HTTP 服务) 做出，并会随着跨进程调用传播到下游服务，同时伴鱼服务端内部的入口服务已经按照业务拆分，因此灰度的过程就可以按入口服务进行，从流量小的、级别低入口服务开始上线观察，再逐渐加入流量大的、级别高的入口服务，最终默认打开全量采样，并在这个过程中发现、解决潜在问题。

### 3.3 资源消耗优化

新版架构所需资源与旧版差别不大：

* 存储：ES 索引
* 计算：部署在每个 k8s 节点上的 otel-agent，以及 otel-collector 集群 (新增)

在逐步上线到所有入口服务之前，我们做了比较充分的风险评估。开启全量采集后，主要增加了宿主机的网络 i/o，在千兆网卡 (约 300MB/s) 支持下，增加后的 i/o 量远远未达到瓶颈。实施过程中，业务团队也确实没有感知。不过在灰度上线过程中，我们也发现并解决了若干问题。

#### 3.3.1 热点服务问题

不同服务的请求量不同。个别服务的上报量过大会导致不同 otel-agent 的流量不均衡，在高峰期造成 otel-agent 的 CPU 经常超过预警线。我们通过增加热点服务实例，减小单个实例的请求量的方式，间接地均衡了每个 otel-agent 承载的流量。

#### 3.3.2 过滤下推

在生产环境中，我们默认维持最近 7 天的 trace 数据。在分析 ES 中索引 `jaeger-span-*` 的过程中，意料之中地，我们看到了 power law 的存在：

![](./span-distribution.png)

仔细分析可以发现，50% 以上的 span 都属于 `apolloConfigCenter.*`。熟悉 apollo 的研发应该知道，通常 apollo sdk 会通过长轮询的方式从元数据中心拉取配置信息，并缓存到本地，而服务在应用层获取配置都是通过本地缓存获取。因此实际上这里的所有 `apolloConfigCenter.*` 只是本地访问，而不是跨进程调用，span 数据价值比较低，可以忽略。于是我们开发了通过正则匹配 spanName 过滤 span 的 processor，并部署到 otel-agent 上，我们称之为过滤下推。部署上线后，ES 索引体积下降超过 50%，目前每天索引体积为 40-50 GB；otel-collector 和 otel-agent 的 CPU 消耗也降低了接近 50%。

### 3.4 制定 SLO

在伴鱼服务端内，线上问题排查的第一入口通常是 im 消息，报警平台会将导致报警的 traceID 以及日志注入到消息中，并提供一个链接页面，方便研发快速查看报警相关的调用链信息及其在整个调用链上每个服务打印的日志。基于此，我们制定了新版调用链追踪系统的 SLO：

| 名称     | 研发关心的 trace 数据采集成功率                              |
| -------- | ------------------------------------------------------------ |
| SLI 规范 | 研发关心且被采集的 trace 个数/研发关心的 trace 个数          |
| SLI 实现 | 包含 trace 数据的服务报警信息条数/服务报警信息条数           |
| 查询     | `sum(alertmanager_alert_total{trace_exists="true"})/sum(alertmanager_alert_total)` |
| 目标     | 99%                                                          |

目前我们刚刚在报警平台中支持此 SLI 的埋点。目前还有个别服务尚未升级相关依赖，因此该指标尚不能反映所有服务的情况，我们会继续推动各服务全面升级，按照以上 SLO 来要求新版系统。

## 4. 小结

借助开源项目，我们得以通过花费极少的人力，解决当前伴鱼内部调用链追踪应用的稳态分析及异常检测需求，同时也为开源项目和社区做出微小的贡献。调用链追踪是可观测性平台的重要组件，未来我们将继续把一些精力放在 telemetry data 的整合上，为研发提供更全面、一致的服务观测分析体验。

## 5. 参考

* [调用链追踪系统在伴鱼：理论篇](https://www.infoq.cn/article/NVdIDI5lZBfmaIHWNVxS)
* [Building your own OpenTelemetry Collector distribution](https://medium.com/opentelemetry/building-your-own-opentelemetry-collector-distribution-42337e994b63)
* [A brief history of OpenTelemetry (So Far)](https://www.cncf.io/blog/2019/05/21/a-brief-history-of-opentelemetry-so-far/)
* [OpenTelemetry Agent and Collector: Telemetry Built-in into All Services](https://www.youtube.com/watch?v=cHiFSprUqa0)
* [Github: OpenTelemetry-CNCF](https://github.com/open-telemetry)
  * [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)
  * [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)

