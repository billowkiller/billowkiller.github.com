---
layout: post
title: "MillWheel: Fault-Tolerant Stream Processing at Internet Scale"
date: 2016-11-20 17:23
comments: true
category: Paper Weekend
tags: [paper，streaming, google]
---

## Intro and Issues
MillWheel 是 Google 开发的一个低延时的流式处理系统。用户指定计算的 DAG 图，以及图中每个节点的计算代码，MillWheel 负责数据流的计算，保证整个计算过程的分布式以及容错性。从宏观上看，MillWheel 提供了数据计算的幂等性，在用户视角提供不重不丢的语义。

在论文中，还存在一个使用场景。Google 的 Zeitgeist 服务用来监测网络搜索的趋势，输入端是持续不断的搜索内容，经过异常检测，异常检测通过模型预测来排除false positive，输出 spike 或者 dip 的搜索。这要求解决：

* Persistent storage： 模型预测和峰值判断分别依赖于长期和短期的存储。
* Low Watermarks: 流量低谷的判断需要有一个标志来判断期待时间窗口内的数据都到达。
* Duplicate Prevention: 在各种情况下用户无需考虑重复流量，系统保证数据传输和处理的 exactly-once 语义。

<!--more-->

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-11-22/53383350.jpg" width="600px"/>

综合而言，对比于一般的分布式流式系统，存在一些特别的要求：

* 处理数据乱序的问题。
* 处理数据到达延时问题。
* 提供 exactly-once 的保证，尤其在 failover 的时候。
* 需要有持久存储，读写权限暴露给用户，提供保证一致性。

## Big Ideas

从对系统的要求来看，有两个重要的概念：`low watermark` 和 `exactly-once`。low watermark 解决前两个问题，exactly-once 解决后两个问题。

### low watermark

low watermark 在原文的作用如下：

>The low watermark for a computation provides a bound on the timestamps of future 
records arriving at that computation.

>Strip away outliers and offer heuristic low watermark values for pipelines that are more interested in speed than accuracy. 


即时间窗口内所有事件的时间下限，目的是为了让这个窗口可以正常的划出，更快的计算出结果。它其实是提供了一个折衷, 这个折衷就是在watermark这个点认为不会有比 watermark 值更老的数据到来了，本质上是一个barrier, 即系统中所有正在流动的数据的 timestamp 都大于或等于该 watermark。

之所有需要有这个概念，是由于网络环境或者故障导致数据流乱序，窗口无法判断何时数据已经全部到达。这里的时间计算一般是基于eventtime的，即时间发生时间，如logtime。文中也给了一个应用的例子：

>In the case of Zeitgeist, our input would be a continuously arriving set of search queries, and our output would be the set of queries that are spiking or dipping.

it is important to be able to distinguish whether a flurry of expected Arabic queries at t = 1296167641 is simply delayed on the wire, or actually not there.

波谷由于数据量小，对到达的数据比较敏感，所以需要判断是真的到达波谷，还仅仅是由于延迟导致数据还在传输途中。之前的做法是再设置一个安全时间，延迟几分钟再计算，但安全时间如何设置？过长则影响时效性，过短无法奏效。或者采取右端开放窗口的补偿方法，先产出数据，当延迟数据到来时再予以修改，但需要业务场景适用、下游系统支持修改。

除了aggregation计算对时序性有要求外，一些策略计算可能也会要求数据完整、有序。那么 low watermark 的定义如下：
	
	low watermark of A = min(oldest work of A, low watermark of C: C outputs to A)
	If there are no input streams, the low watermark and oldest work values are equivalent.

A 点的 low watermark 的值等于 A 点所有待处理的消息的 timestamp 和 A 所有上游 C 的 low watermark 的最小值。这是一个递归的定义，终止条件处于流的源头，即由外围注入的injection或者数据订阅端生成；生成之后会驱动 watermark 传递给下游。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-11-23/12047055.jpg" width="600px"/>

由上图可以看出，low watermark 是根据事件到达后的处理时间递增的，反映的是系统中所有 *pending work*；并且事件其实并不是有序到达和处理的，但是在某个时间可以确保所有的事件都已经到达了，即在文中所强调的：

>By waiting for the low watermark of a computation to advance past a certain value, the user can determine that they have a complete picture of their data up to that time

对于处于 low watermark 下的事件，有两种处理方案：抛弃或者更正已有的状态。

low watermark 除了提供系统事件处理的完成情况，还有一个作用，触发trigger（文中用Timer）。trigger 实际是窗口内部累积状态的记录器，根据不同的触发类型来累积状态，达到一定的条件就允许外围触发，将窗口内的数据计算后 emit 给下游。MillWheel 的触发条件有 wall time 和 low watermark value，这个交给用户来定义。

### exactly-once

这个概念涉及两个方面：

*	Persistent storage available at all nodes in the stream graph
*  Exactly once delivery semantics.

前者的概念好理解，Persistent state 在 MillWheel 中表现为一个键值对，值是 protobuf 的一个二进制字符串。用户代码可以方便的获取和设置各种状态。常用的状态为窗口数据聚合中保存的 counter，join 要使用的 buffer 等。

MillWheel 的状态存储在 bigtable 或者 Spanner 中，另外它还提供一个 soft state，用于内存 cache 或者 聚合。为了保证可能存在的状态不一致性，对所有的 per-key update 都封装在一个原子的操作中，防止在处理过程中可能出现的各种意外。

但是在 failover 或者 load balancing 的时候，计算会被迁移，这时候可能存在 zombie writer 或者 stale writer。因此，为了保证原子性，需要保证每个 key 只有一个 writer 可以写入。MillWheel 的解决方案是给每个 writer 附带一个 sequencer token，后台存储的协调者在每个写入前会检查 token 的有效性使得新 worker 可以阻止其他的 sequencer 的有效性。这其实就是一个 lease 机制，在一段时间内对于特定的key只有一个worker可以写入。

保证 Exactly Once 语义有个很重要的简化途径是将用户的非幂等操作变成幂等。通过以下措施来保证：

* Exactly Once delivery
* Strong Production

Exactly Once delivery 通常的方法是保证 at least once 基础上进行去重。MillWheel 也是通过这个方法保证，发送方发送数据后没有接收到 ack 重新发送数据。另外接收方接收到数据会进行 state modificaton，这时候将数据的 unique ID 也一并原子写入到后端存储；接收端为了加速去重，使用 bloom filter，这时候需要考虑 false postive，如果发生 filter miss，需要去后端存储进行比较，如果确认是重复数据，会返回 duplicate ACK 通知发送方。

在发送方发送数据之前可以进行 checkpoint，用于 failover 后恢复原来的状态。在发送的数据被 ACK 后，checkpoint可以删除。这个 `Checkpoint->Delivery->ACK->GC` 的运行模式在 MillWheel 中称为 `Strong Production`。即使用户代码不是幂等的，在 Strong Production 中，无论 retry 多少次，用户的运行逻辑都是正确的。

与之对应的是 `Weak Production`，这时的用户代码是幂等的，无需strong procution，即 checkpointing；无需 exactly once delivery（把 deduplication 逻辑去掉）。这在延迟和资源消耗上无疑更加友好，但是对于取消 checkpointing 有一个 straggler latency 问题。在 strong production中，只要 checkpoint 就可以返回上游表示这个阶段的任务结束，但是现在需要下游返回 ack 才可以保证，并且这是级联的。Millwheel 采用 checkpointing 一部分 straggler 的 pending production 来缓解，如下图：

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-11-23/46647429.jpg" width="400px/">

## Other Details

MillWheel 集群可以动态扩缩容，根据 key interval 划分计算，并以此通过 master 进行负载分配、均衡。当监控进程检测到机器的负载过高的时候，会进行 key interval 的重分配：split、merge、move。

failover 时，key interval 被分配到新的 owner，它可以从后端存储中获取计算需要的元数据，包括 heap of pending timers 以及 queue of checkpointed productions。一个节点的例子如下，每个计算都是一个pipeline，即输入计算输出。数据的传输通过RPC。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/public/16-11-24/94589760.jpg" width="400px"/>


另外关于 low watermark 的管理和流动，Millwheel 是基于一个全局的订阅发布中央系统。

* 每个 processer 将自己持有的所有work（in fly, stored, pending）计算出最小的 timestamp发给中央系统。
* 不同的 processer 在不同的 key interval 下，low watermark 也是根据 key interval 来存储。
* 为了保证计算的准确性，订阅发布系统会查询后台存储，以保证每个 key interval 都有 low watermark。感兴趣的节点会订阅每个发送端的 low watermark，计算所有的最小值作为 low watermark。
* 之所以不在中央系统算最小值是为了保证一致性：中央系统的 low watermark 不能大于节点上的，否则在 failover 后容易出现 low watermark 的回退。在节点上计算最小值则保证中央系统永远不会领先于节点上的。


