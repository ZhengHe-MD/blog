---
title: the history of distributed tracing
date: 2020-11-15 17:31:59
tags:
- distributed tracing
categories:
- microservice
---

# Distributed Tracing 简史

2003 年，Microsoft Research 发表了 [Magpie](https://www.usenix.org/legacy/publications/library/proceedings/hotos03/tech/full_papers/barham/barham_html/paper.html)，它的设计要义有两点：Black-box instrumentation，即对代码无侵入性，以及 End-to-End tracing，即支持单个请求粒度的数据分析。

2004 年，UCB 和 Stanford 的学者提出 [Pinpoint](http://roc.cs.berkeley.edu/papers/roc-pinpoint-ipds.pdf)，采集请求所经过的不同服务信息作为数据，请求最终成功与否的信息作为标签，最后通过数据挖掘技术来作分析。Magpie 和 Pinpoint 在提出之时都以离线分析为主，并不支持在线分析，且二者都是在虚拟场景下的实验结果，并未真正投入生产使用。

2010 年，Google 发表 [Dapper](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36356.pdf)，发表之时 Dapper 已经在 Google 核心搜索系统的生产环境上稳定运行两年，就像 Google 的其它著名论文一样，Dapper 的工程实践很快被业界所接纳、模仿和实现。如果你曾经在任意技术大会上听过 Distributed Tracing 话题的演讲，十有八九演讲者会提到 Dapper。Dapper 的一些概念，如 Trace Tree、Span、Annotation，和实践方案，如 Sampling、基于日志 (out-of-band) 和 NoSQL (BigTable) 的 Trace Collection，影响深远，至今仍未过时。

2012：Zipkin

2015：Ben Sigelman left google and started Lightstep

2016：Uber open sourced Jaeger; OpenTracing is accepted  by CNCF, and released

2018：OpenCensus 1.0 released, W3C HTTP context propagation header

2019：OpenTelemetry, W3C tracing context specification entered proposed recommendation status

## 参考文献

* [Magpie: online modelling and performance-aware systems](https://www.usenix.org/legacy/publications/library/proceedings/hotos03/tech/full_papers/barham/barham_html/paper.html)
* [Pinpoint: Problem Determination in Large, Dynamic Internet Services](http://roc.cs.berkeley.edu/papers/roc-pinpoint-ipds.pdf)
* [Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36356.pdf)

