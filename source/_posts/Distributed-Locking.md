---
title: Distributed Locking
date: 2020-03-22 14:26:51
tags:
categories:
- system design
---

提到分布式锁，很多人也许会脱口而出 "redis"，可见利用 redis 实现分布式锁已被认为是最佳实践。这两天有个同事问我一个问题：”如果某个服务拿着分布式锁的时候，redis 实例挂了怎么办？重启以后锁丢了怎么办？利用主从可以吗？加 fsync 可以吗？“

因此我决定深究这个话题。

# Efficiency & Correctness

在单机/实例中，如果只希望单个任务被执行一次，使用本地环境提供的 Lock 原语即可实现；但在云原生环境下，服务通常会被横向扩展到物理隔离的多个实例中，这时候就需要分布式锁。总体来说，对分布式锁的要求可以从两个角度来考虑：

* 效率 (Efficiency)：为了避免一个任务被执行多次，每个执行方在任务启动时先抢锁，在绝大多数情况下能避免重复工作。即便在极其偶然的情况下，分布式锁服务故障导致同一时刻有两个执行方抢到锁，使得某个任务被执行两次，总体看来依然无伤大雅。
* 正确性 (Correctness)：多个任务执行方仅能有一方成功获取锁，进而执行任务，否则系统的状态会被破坏。比如任务执行两次可能破坏文件结构、丢失数据、产生不一致数据或其它不可逆的问题。

以效率和正确性为横轴和纵轴，得到一个直角坐标系，那么任何一个 (分布式) 锁解决方案就可以认为是这个坐标系中的一个点：

<img src="/blog/2020/03/22/Distributed-Locking/correctness-and-efficiency.jpg" width="600px">

# Solutions

在进入分布式锁解决方案之前，必须要明确：**分布式锁只是某个特定业务需求解决方案的一部分**，业务功能的真正实现是**业务服务**、**分布式锁**、**存储服务**以及**其它有关各方**共同努力的结果。

本节分别讨论**侧重效率**和**侧重正确性**的两类解决方案，并给出相应的实现思路。

## For Efficiency

如果任务的执行具备幂等性，或即使少量任务重复执行对整体功能没有影响，开发者就可以选择侧重效率的解决方案。

### Design Requirements

侧重效率解决方案的设计要求可以概括如下：

* **High Efficiency**：抢锁、释放锁操作高效 (这里暂不设定具体的 QPS)
* **Weak Safety**：在绝大多数时刻，只有一个 client 可以拥有锁
* **Liveness**
  * Deadlock free：如果 client 抢锁后崩溃或者出现网络分区，其它 client 不能永远等待下去
  * Basic Fault tolerance/Availability：有简单的容错机制，能抵御小型故障，保持锁服务可用性

### Implementation: Single Redis Instance

如果你面对的业务场景侧重效率，那么基于单实例 redis 的解决方案就是你的菜。redis 是内存数据库，请求的执行效率基本能满足要求。下面介绍的就是 redis 官方提供的解决方案，你也可以[阅读原文](https://redis.io/topics/distlock)。

##### Lock/Unlock

```lua
-- Lock
SET resource_name my_random_value NX PX 30000
```

resource_name 是分布式锁的 id，它的唯一保证只有一个 client 可以获取锁；NX 表示仅在 resource_name 在 redis 中不存在时才执行，用于保证 Week Safety；PX 30000 表示 30 秒超时，用来保证 Deadlock free，my_random_value 是每个取锁请求的 id，释放锁的逻辑如下：

```lua
-- Unlock.lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

释放锁时，当且仅当入参与之前加锁时用的 my_random_value 相等时才删除相应的键值对。my_random_valu 的唯一保证释放锁 (unlock) 操作的安全性，保证锁不会被误释放或恶意释放，一种误释放的场景如下图所示：

<img src="/blog/2020/03/22/Distributed-Locking/random-value.jpg" width="600px">

1. client 1 获取锁成功。因为某种原因，如网络延迟、GC 导致 client 1 执行任务时间超过 TTL
2. client 2 获取锁成功
3. client 1 完成任务，执行释放锁操作，此时如果没有 my_random_value，client 2 获取的锁将被误释放
4. client 3 获取锁成功

此时，client 2 和 3 都认为自己取锁成功，违背了 Week Safety。

##### Fault Tolerance & Availability

如果光有单实例，实例挂了以后锁服务不可用，为了提高容错率，一种简单的办法是利用主从做故障转移。master 节点挂掉后，slave 节点顶上，有一定的容错能力。但为了效率，redis 的 replication 是异步的，如果 master 在接收加锁请求并回复 ack 给 client 后，同步数据到 slave 之前崩溃，此时虽然服务可用，但其它 client 可以立即取锁成功，违背了 Week Safety。即便使用 [WAIT](https://redis.io/commands/wait) 命令，等待所有 replicas 返回 ack 或者超时，也只能在一定程度上提高安全性，毕竟 WAIT 命令没有分布式共识算法在其后支持，同时这也是在牺牲效率来换取安全性。

## For Correctness

如果任务的执行不具备幂等性，且系统可能因为任务重复执行陷入麻烦，开发者就必须考虑侧重正确性的解决方案。

### Design Requirements

侧重正确性解决方案的设计要求概括如下：

* Strong Safety: 在任意时刻，只要持有时间不超过 TTL，只能有一个 client 能取到锁
* Liveness
  * Deadlock free：如果 client 抢锁后崩溃或者出现网络分区，其它 client 不能永远等待下去
  * Strong Fault Tolerance & Availability：高容错、高可用

### Implementation: Concensus Service

在云原生环境下，一个系统要做到高容错、高可用，就必须能够横向扩容，横向扩容的结果就是任何在单机/实例下证明有效的算法都要能够合理地迁移到多实例中。以单实例版 redis 分布式锁为例，要迁移到多实例上，拿掉 Single Point of Failure (SPOF) 就意味着失去 Single Source of Truth (SSOT)。在多实例环境下要实现 Strong Safety，就必须引入经过理论与实践检验的共识算法，如 Raft、Viewstamped Replication、Zab 以及 Paxos，这些算法在 asynchronous model with unreliable failure detectors (翻译成人话就是**算法的正确性不依赖系统中任何事件发生时间**) 的故障模型下，依然能在一定故障 (少于半数节点故障) 存在的情况下保证系统对一些信息达成共识。

本节就不 (mei) 详 (neng) 细 (li) 展开对 Concensus Service 的讨论，感兴趣的可以自行阅读 Martin Kleppman DDIA 一书的 Consistency & Consensus 一节，这里我们可以直接假设它的存在。

### The Whole Story

理想状况下，开发者利用侧重正确性解决方案的最终目的是：**在任何情况下每个任务仅被执行一次**。仅靠侧重正确性的分布式锁方案还不够。考虑这样一个场景：

<img src="/blog/2020/03/22/Distributed-Locking/exactly-once.jpg" width="680px">

1. client 1 获取锁成功，但不巧遇到了 stop-the-world GC 暂停，导致锁过期
2. client 2 获取锁成功，执行任务，并成功写入结果到 storage 中
3. client 1 GC 结束后，执行任务，并成功写入结果到 storage 中

同样的任务重复执行，client 2 写入的数据被 client 1 写入的数据覆盖，出现更新丢失 (lost updates)。上述场景中，lock service 完全按照要求运行，但结局依然感人肺腑。

但如果 lock service 能和 storage 配合起来，就能解决更新丢失问题：

<img src="/blog/2020/03/22/Distributed-Locking/fencing-token.jpg" width="680px">

1. client 1 获取锁成功，同时获得单调自增 token = 33，遇到 stop-the-world GC 暂停，导致锁过期
2. client 2 获取锁成功，同时获得单调自增 token = 34，执行任务，并成功写入 storage
3. client 1 GC 结束后，执行任务，写入 storage 时，后者发现 token 比最新写入的 token 小，拒绝执行

从上面这个例子可以看出，拥有一个侧重正确性的分布式锁解决方案仅仅是一个完整的任务执行去重方案必要条件。这与消息系统中的 exactly-once delivery guarantee 十分类似，exactly-once 语义的正确实现，需要参与各方的正确合作才可能实现。

# Redlock & Debate

(TODO, Martin Kleppman with Antirez)

# Back to the Question

回到本文开篇的问题。

问：“如果某个 client 拿着锁执行任务时，redis 挂了怎么办？”

答："看业务要求，大部分场景下挂了就随它去吧，通过主从、集群部署、甚至打开 fsync 、使用 WAIT 命令可以有限地增加容错能力和可用性，但因为没有共识算法的支持，分布式锁的正确性仍然无法保证。如果不能接受，就采用侧重正确性的分布式锁方案，但请务必关注具体业务场景的完整解决方案。“

# References

* [Martin Kleppmann: How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
* [redis: Distributed locks with Redis](https://redis.io/topics/distlock)

* [Is Redlock safe?](http://antirez.com/news/101)

* [Unreliable Failure Detectors for Reliable Distributed Systems](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf)
