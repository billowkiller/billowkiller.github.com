---
layout: post
title: "Spark Streaming"
date: 2015-10-27 21:04
comments: true
category: Big Data
tags: spark
---

流式计算通常是为了满足日益增长的数据的实时获取和低延时计算的需求。通常来说，一个优秀的流式计算引擎需要满足一下的一些需求：

1. 不重不丢的保证（在节点或计算失败的时候，内存中的计算状态能被正确的恢复）
2. 低延迟
3. 高吞吐量
4. 强大的计算模型
5. 容错机制的低开销
6. 流控

<img src="https://spark.apache.org/images/spark-logo.png" width="200px"/>

<!--more-->

介绍完了spark后就可以来说说spark streaming，毕竟spark streaming是完全构建在spark之上，熟悉了spark的RDD原理之后就比较容易理解spark streaming。说白了，spark streaming中的流式计算是`伪实时`的，之所以是伪实时的是因为它将实时的处理变成时间跨度较小的批量处理。没错，就是`将一段时间间隔中的数据变成RDD`，然后利用spark原有的架构处理这段时间内的数据，接着在时间维度上对这些处理后的RDD进行迭代计算。也就是将流式计算分解为一系列微小的、原子的批量作业，每个微批量作业如果失败则可以重新计算。这种分解的思想可以应用在批量计算框架、也可以应用在流式计算框架上，例如Storm Trident。

这样的处理带来了几个明显的好处：

* 高吞吐量
* 不从不丢的保证
* 快速falut recovery
* 更方便处理慢节点
* 实时和批量统一的编程接口

下面介绍下spark streaming中的具体实现来理解这几点。

### 计算模型

![image](http://blog.selfup.cn/wp-content/uploads/2014/08/streaming-flow.png)

首先，Spark Streaming把实时输入数据流以时间片（如1秒）为单位切分成块。Spark Streaming会把每块数据作为一个RDD，并使用RDD操作处理每一小块数据。每个块都会生成一个Spark Job处理，最终结果也返回多块。

举个栗子来说明下这个过程：

    pageViews = readStream("http://..."， “ls”)
    ones = pageViews.map(event => (event.url, 1))
    counts = ones.runningReduce((a, b) => a+b)

上述代码的作用是根据URL的数量计算访问次数。处理的过程为首先通过HTTP获取到一个pageview事件的RDD，经过*transformation*操作变成(URL, 1), 最后的操作计算相同的URL数目。整个streaming过程可以用下图来表示：

<img src="http://img-storage.qiniudn.com/15-10-18/95269436.jpg" width="500px"/>

整个的处理过程可以看出，输入流被分成每个都是1秒的batch，经过处理后生成resultRDD，这个resultRDD在每个时间间隔中都会产生一个，并经过reduce迭代计算。在上图中最终的reduce的输入参数就包括上一个时间间隔的resultRDD和这个时间间隔中map操作的结果。上图也可以看出这些RDD的lineage graph，在节点失败的时候，可以根据这个lineage graph以partition的粒度为单位重新执行任务计算丢失的分片，可以看出这些计算的任务都是可以并行执行的。同时，对于慢节点来说，因为`计算是无状态的`，且每个job的结果是可以确定的，spark streaming可以执行类似于hadoop中的预测模型——在其他节点上计算同样的任务。另外，spark streaming也会有些checkpoint来防止无限制的恢复计算。

对比于其他流式处理的方式，spark streaming在处理失败任务和慢节点上无疑更有效率。它可以从`时间和partition`两个维度上并行地计算数据来加快恢复速度。而对于像Strom的流式处理来说，往往是通过**上游数据backup**或者**同时复制执行相同的作业**来保证数据处理的可靠性。这两种处理方式对于资源的消耗无疑都是巨大的，且恢复的时间也比较长。

* 其中对于前者来说，每个节点都需要保存上一个checkpoing后所发送数据的拷贝，在节点失败时由standby机器重新计算上游节点发送过来的数据，因为这些计算都是有状态的，所以恢复的时间比较长，Storm就只保证“at least once”语义来提高处理速度。Trident之所以能够保证不重不丢是使用了数据库来复制计算状态。
* 对于后者无疑更加消耗资源，并且需要保证两个数据处理操作接受到的上游数据的顺序是一样的。

同spark一样，只有当某个Output Operations原语被调用时，stream才会开始真正的计算过程。现阶段支持的Output方式有以下几种：

* print()
* foreachRDD(func)
* saveAsObjectFiles(prefix, [suffix])
* saveAsTextFiles(prefix, [suffix])
* saveAsHadoopFiles(prefix, [suffix])

###流式处理中的几点难点

在流式处理中通常都会有几个难点需要考虑。

* 时间窗口问题
* 数据一致性问题
* 内存状态管理问题

我们来看下spark streaming是如何解决的。

1. **时间窗口问题。**spark streaming是根据数据到达系统的时间将记录放到对应的RDD中，这种时间窗口划分是基于墙上时间的，好处可以保证系统及时产生一个新的batch运行job，并且可以让程序运行在数据生成的地方，不必再进行分发。

    这种基于墙上时间的统计有一个非常严重的问题是不能回放数据流。当数据流是实时产生的时候，“墙上时间”的一分钟也就只会有一分钟的event被产生出来。但是如果统计的数据流是基于历史event的，那么一分钟可以产生消费的event数量只受限于数据处理速度。另外event在分布式采集的时候也遇到有快有慢的问题，一分钟内产生的event未必可以在一分钟内精确到达统计端，这样就会因为采集的延迟波动影响统计数据的准确性。所以产生了另外一种时间窗口划分的方法。
    
    另一种时间窗口划分的方法是基于外部时间的，例如日志时间。spark streaming提供两种方法来处理这种情况：

    * 延迟处理，等待一定时间来处理每个batch。
    * 用户的应用程序中保证乱序事件的正确处理。
    
    以上也说明在`批处理的流式计算模型是受限的`，很多情况下只能依靠用户的应用程序来做处理，例如实时的统计5s内的pv；其次这种方式也没有很好的流控技术手段，如果有突发的大量数据产生，会导致结果产生的时间更长，甚至是将系统的JVM撑爆。最后实时性也是受限的，只能达到次秒级的处理延迟，毕竟是要等待一个时间batch的处理完成。

2. **数据一致性问题。**什么是数据一致性，举个栗子，要统计网站中来自各个国家的page view，把不同国家的pv统计放在不同的节点上处理。但是现在统计英国的节点处理速度要慢于法国的，这将导致两个节点上数据的时间状态不一致。在流式处理中，数据的一致性的保证同时意味着资源的消耗。流式处理的数据一致性有三种解决思路，在上文中也有提到，这里概括下：

    * 上游备份策略：重启的时候重放kafka的历史数据，恢复内存状态
    * 中间状态持久化：把统计的状态放到外部的持久的数据库里，不放内存里
    * 同时跑两份：同时有两个完全一样的统计任务，重启一个，另外一个还能正常运行。
    
    而在spark streaming中，数据一致性天然得到保证的。因为记录根据时间来分片，所以中间的resultRDD反应的是当前时间和之前时间所计算出来的结果，无论计算和结果被分配到哪个节点上都不会有节点间数据不一致的情况。也就是数据的不重不丢可以得到保证。
    
3. **内存状态管理问题。**
做流式统计的有两种做法：

    * 依赖于外部存储管理状态：比如没收到一个event，就往redis里发incr增1
    * 纯内存统计：在内存里设置一个counter，每收到一个event就+1

    第一种会把整个压力全部压到数据库上，造成处理速度下降；第二种的状态相对来说容易管理一些，计算直接是基于这个内存状态做的。如果重启丢失了，重放一段历史数据就可以重建出来。内存的问题是它总是不够用的，解决的方法是input分割和把存储移到外边去。在内存计算中把窗口统计的中间状态落地的好处是显而易见的：重启之后不用通过重算来恢复内存状态。但是这种对外部数据库使用不小心就会导致两个问题：

    * 处理速度慢。不用一些批量的操作，数据库操作很快就会变成瓶颈
    * 数据库的状态不一致。内存的状态重启了就丢失了，外部的状态重启之后不丢失。重放数据流就可能导致数据的重复统计
                     
    在spark streaming中支持传统批量计算中的无状态*transformation*操作，例如`map`、`reduce`、`groupBy`和`join`。这就避免了普通流式计算中麻烦的状态保存问题。但spark streaming中也支持多个时间间隔中有状态的*transformation*操作，包括：
    
    1. Windowing: 生成滑动窗口RDD。`words.window("5s")`将产生一个RDD包含[0,5),[1,6),[2,7)的时间间隔。
    2. 增量聚合：在滑动窗口的基础上进行RDD的聚合操作，也就是`reduceByWindow`。在下图的*a*中对应的代码为`pairs.reduceByWindow("5s", (a, b) => a+b)`，也就是计算5s内的计数之后。图*b*的代码为`pairs.reduceByWindow("5s", (a, b) => a+b, (a, b) => a-b)`。其实很简单，第一个lambda表达式为进入滑动窗口的处理函数，第二个表达式为离开滑动窗口的处理函数。这样也就不用重复求和了。
    ![](http://img-storage.qiniudn.com/15-10-18/97873121.jpg)
    3. 状态跟踪：
    如下图所示，就是保存上一个时间间隔的RDD与本次的记录进行groupBy加map计算的到状态的变化情况。
    ![](http://img-storage.qiniudn.com/15-10-18/91458919.jpg)
    
    在对这些带状态的操作的处理过程中也就用到了上述的所属的利用外存在保存中间的状态。spark streaming中这只发生在intervel之间，所以整个内存的状态管理会比传统的流式处理简单许多，而且高效，不需要对每一步都进行状态同步，状态恢复的成本也比较低，上文中提到的可以在多个节点上并行计算恢复。


###System Architecture
![](http://img-storage.qiniudn.com/15-10-18/82954556.jpg)

Spark streaming和Spark的系统结构有些许改动，如上图所示主要包括3个部分：

* *master* that tracks the D-Stream lineage graph and schedules tasks to compute new RDD partitions.
* *Worker* nodes that receive data, store the partitions of input and computed RDDs, and execute tasks.
* A *client* library used to send data into the system.

从上图中可以看出，Spark Streaming和传统的流式系统最大的区别就是Spark Streaming将计算分成小的，无状态的确定性的任务，这些任务会在集群的任意节点上运行。并且相对于传统流式系统的拓扑结构来说，无需消耗大量时间将将任务进行迁移，Spark Streaming可以很好的对机器上的节点进行负载均衡，处理失败任务并且对慢节点进行预测。

对比于Spark的系统，Spark Streaming做了一下的改进：

* 网络传输。使用异步I/O获取远端数据。
* TimeStep pipelining。Spark的调度器可以在当前任务还未完成的时候可以提交下一个时间分片的任务。
* 任务调度：优化任务调度器，例如调整控制消息的大小，可以在每隔几百毫秒时间内启动几百个并行任务。
* 存储层：支持异步的RDD checkpoint，RDD是不可变的，所以异步存储不会阻塞现有的计算。
* Lineage切割：控制RDD linage graph的大小，在checkpoint之前的lineage便可以删除。

当master fail的时候可以进行HA，所有的worker重新连接到新的master上，将原来的checkpoint和原始数据重新计算。因为所有的操作都是确定性的，所以RDD是可以重复计算，也就是说在HA的时候丢失一些正在运行的计算任务不会对最终结果造成什么影响。所有的元数据都是存储在HDFS上的，包括：

1. RDD的lineage graph，代表用户代码的Scala函数对象。
2. 上一个checkpoint的时间
3. RDD的ID。因为HDFS的checkpoint文件会在每个时间片重新命名。

###FAQ
1. Dstream与RDD之间的关系

	首先来看下Spark streaming的代码`val ssc = new StreamingContext(sc, Seconds(2))`。在这句的作用是定义Dstream生成的时间间隔，`2s`就是这个时间间隔，也叫`batch interval`。具体说来**一个streaming batch对应一个RDD**，也就是这个batch interval里产生的数据。
	
	在这个RDD中，有n个partition，n = batch interval / block interval。 `block interval`是spark steaming内部定义的一个变量`spark.streaming.blockInterval`，通常是200ms。上述例子就产生10个partitions。
	
	Blocks由一个receiver产生，receiver就是流式数据的接收端，每个receiver被分配到一个host上，所以上述的10个partitions就由一个node产生，同时被`复制到第二个节点上做容错`。注意，这里产生了data locality的问题。好的做法是，分配多个receivers接收数据，最后使用union合并数据做processing。当然还可以对Dstream做`repartition`操作提高并行度。
	
2. Spark Streaming 的容错性
	
	对于文件这样的源数据，这个driver恢复机制足以做到零数据丢失，因为所有的数据都保存在了像HDFS或S3这样的容错文件系统中了。但对于像Kafka和Flume等其它数据源，有些接收到的数据还只缓存在内存中，尚未被处理，它们就有可能会丢失。这是由于Spark应用的分布操作方式引起的。当driver进程失败时，所有在standalone/yarn/mesos集群运行的executor，连同它们在内存中的所有数据，也同时被终止。
	
	首先driver会利用checkpoint来保存灾备需要的数据，有两种各类型的checkpoint，Metadata Checkpoint（保存streaming计算相关数据） 和 Data Checkpoint（保存生成RDD数据）
	
	对于Spark Streaming来说，从诸如Kafka和Flume的数据源接收到的所有数据，在它们处理完成之前，一直都缓存在executor的内存中。纵然driver重新启动，这些缓存的数据也不能被恢复。为了避免这种数据损失，Spark 1.2发布版本中引进了预写日志（Write Ahead Logs）功能。
	
	当启用了预写日志以后，所有收到的数据同时还保存到了容错文件系统的日志文件中。因此即使Spark Streaming失败，这些接收到的数据也不会丢失。另外，接收数据的正确性只在数据被预写到日志以后接收器才会确认，已经缓存但还没有保存的数据可以在driver重新启动之后由数据源再发送一次。这两个机制确保了零数据丢失，即所有的数据或者从日志中恢复，或者由数据源重发。
	
	[http://www.csdn.net/article/2015-03-03/2824081](http://www.csdn.net/article/2015-03-03/2824081)



