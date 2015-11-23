---
layout: post
title: "Spark Introduction"
date: 2015-10-27 21:00
comments: true
categories: rework
tags: spark
---

简单介绍下Spark。Spark是分布式的内存计算模型，对于两类计算形式能够极大地提升处理的效率：迭代计算，和交互式的数据挖掘工具。迭代计算例如PageRank、K-means聚类、逻辑回归等要求计算结果的重新利用，交互式的数据挖掘例如针对一份的数据集进行多次特定查询也要求原始数据的重复利用。对比于MR作业，它不需要外部存储的介入，从而提升处理速度。接下来将会从Spark的数据模型、编程模型和架构来介绍Spark。

![image](https://spark.apache.org/images/spark-logo.png)

<!--more-->

### RDD Data Model
那么如何利用多个节点的内存进行分布式的计算呢，这就是Spark的核心RDD（Resilient Distributed Dataset）。RDD是一个个记录的集合，只读和分片的。它只能通过外部存储或者其他RDD生成。RDD的这些变化操作名称为*transformation*，在Spark编程中还有另外一个操作需要知道，*action*，意思是返回一个值或者将数据导出到外部存储，像`count`、`collect`、`save`。一个spark程序往往是从外部存储中定义一个或多个RDD开始的，接着经过各种*transformation*，最后*action*。同时*action*也是计算的开始，在spark中计算是延迟的，各种*transformation*形成了computation pipeline在*action*中进行计算。

下表是对RDD中的*transformation*和*action*操作：
![](http://ww2.sinaimg.cn/large/74311666jw1ex310kinr3j210c0fujwg.jpg)

RDD还有两个操作：*persistence*和*partitioning*。其实顾名思义，*persistence*即RDD的持久化操作，这是为了避免重复计算。*partitioning*是重新分区，为了得到更好的执行并发度。

合理的分区可以有效减少shuffle的数据量，根据特定的应用场景执行不同的*partitioning*。例如在PageRank中根据域名来对URL进行Hash，因为很多链接都是内部的。下图表示*partitioning*的效果。

![](http://img-storage.qiniudn.com/15-10-16/6543188.jpg)

持久化操作是也是为了更高的计算效率。例如在下图中，Spark有两个action，所以产生两个job。两个job各自计算RDD1和RDD2，对于数据量大的RDD无疑会影响性能，所以可以将RDD进行*persistence*操作，可以存入内存、硬盘或者二者结合。后续缓存的资源可以手动清除或者通过LRU算法自动清除。
![](http://ww2.sinaimg.cn/large/74311666jw1ex30g3zpazj20u60f2tb9.jpg)

在上文中我们提到RDD是有`transformation`和`lazy compute`的特性的。这两个特性使得RDD不需要一直被实例化，只需要保存这个RDD是如何产生的，在延迟计算的时候便可以通过这些dependence信息进行RDD transformation操作。以上就是RDD lineage，一个RDD的血统关系图。RDD的只读特性也是为了更好地描述这个lineage graph。在上图中两个Output的产生也可以直观地看到RDD的lineage信息。那么这个linage究竟有什么好处呢，RDD可以根据dependency信息直接追踪到在外存中的数据，在发生错误的时候直接通过外存的原始数据计算丢失或错误的RDD分片信息。

### RDD Programming Model

RDD的编程模型需要通过适当的接口来表示RDD是如何经过一系列的transformations来达到现在的状态。在Spark中，RDD的编程模型暴露了5方面的信息：

* dateset中的分片信息
* 父RDD的依赖关系
* 基于其父RDD的计算RDD方法
* 分片的shceme元数据
* 数据存放的位置
![](http://ww3.sinaimg.cn/large/74311666jw1ex32afwa8gj20qq0ds41v.jpg)

在第3个方法中，也就是根据父RDD分片计算本RDD的分片有两种情况：每个父RDD分片最多被一个子RDD分片依赖；父RDD分片被多个子RDD分片依赖。这两种情况分别对应*narrow dependency*和*wide dependency*。为什么区分这两种依赖关系呢，这和RDD的延迟计算特性有关系。

* 在*narrow dependency*中，一个节点上的RDD分片可以通过pipeline的方式从原始数据开始计算，多个分片可以并行的进行。而对于*wide dependency*需要所有父RDD的分片可用，而且涉及到data shuffle。
* *narrow dependency*在节点失效后的恢复的效率更高，因为可以并行地在不同的节点上计算丢失的分片。而在*wide dependency*中，一个父RDD分片可能被多个子RDD分片使用，所以可能导致这些子RDD的祖先分片都有丢失，所以需要重新完整的计算。

整个spark作业的执行调度也是和这两种dependency有关。当一个用户执行一个*action*操作的时候，spark的调度器检测RDD的lineage graph，建立一个DAG图来执行，DAG图中的每个节点就是一个*stage*。在每个*stage*中包含了多个pipeline的*transformation*操作，这些RDD的关系全都是
*narrow dependency*。每个*stage*的边界都是由*wide dependency*的shuffle操作，或者已经计算好的分片。调度器这时候就可以建立执行任务计算每个stage中缺失的*partitions*，直到得到最终的RDD。

那么如何合理划分 stage，并确定 task 的类型和个数？
![](http://img-storage.qiniudn.com/15-10-16/48846624.jpg)

可以看到在上图中，每个stage中的数据都是形成了pipeline计算的，这里的pipeline思想是：**数据用的时候再算，而且数据是流到要计算的位置的**。有两层意思，一是延迟计算，二是计算本地化。比如在第一个 task 中，从 FlatMappedValuesRDD 中的 partition 向前推算，只计算要用的（依赖的） RDDs 及 partitions。在第二个 task 中，从 CoGroupedRDD 到 FlatMappedValuesRDD 计算过程中，不需要存储中间结果（MappedValuesRDD 中 partition 的全部数据）。

在有*wide dependency*的时候需要Shuffle后无法进行pipeline。那么我们可以**从后往前推算，遇到 *wide dependency*就断开，遇到*narrow dependency*就将其加入该stage。每个stage里面task 的数目由该stage最后一个RDD中的partition个数决定**。因此上图中最后一个stage的id是0，stage 1  stage 2都是stage 0的parents。

整个的computing chain也是根据数据依赖关系自后向前建立，遇到*wide dependency*后形成 stage。computing chain从后到前建立，而实际计算出的数据从前到后流动，那么RDD内部是如何实现计算的呢。在每个stage中，每个RDD中的compute()调用parentRDD.iter()来将parent RDDs中的 records一个个fetch过来。
    
>代码实现：每个 RDD 包含的 getDependency() 负责确立 RDD 的数据依赖，compute() 方法负责接收 parent RDDs 或者 data block 流入的 records，进行计算，然后输出 record。经常可以在 RDD 中看到这样的代码firstParent[T].iterator(split, context).map(f)。firstParent 表示该 RDD 依赖的第一个 parent RDD，iterator() 表示 parentRDD 中的 records 是一个一个流入该 RDD 的，map(f) 表示每流入一个 recod 就对其进行 f(record) 操作，输出 record。为了统一接口，这段 compute() 仍然返回一个 iterator，来迭代 map(f) 输出的 records。

在实现中，每个task是基于数据的本地性进行分配的，任务是在该分片的节点上执行的。而对于*wide dependecy*，会在拥有父分片的节点上计算中间结果，这是为了减少fault recovery的时间。

### Spark Architecture
在上一节中我们确定了Spark stage和task的生成和执行方式，那么这些stage和task在整个spark application中又处于什么样的位置呢，上文中我们提到的每个job又是什么意思呢，为什么一个application中又会有好多个job，它们之间的关系又是什么样的呢？

![image](http://spark-internals.books.yourtion.com/markdown/PNGfigures/deploy.png)

从部署图中可以看到

* 整个集群分为 Master 节点和 Worker 节点，相当于 Hadoop 的 Master 和 Slave 节点。
* Master 节点上常驻 Master 守护进程，负责管理全部的 Worker 节点。
* Worker 节点上常驻 Worker 守护进程，负责与 Master 节点通信并管理 executors。
* Driver 官方解释是 “The process running the main() function of the application and creating the SparkContext”。Application 就是用户自己写的 Spark 程序（driver program），比如 WordCount.scala。
* 每个 Worker 上存在一个或者多个 ExecutorBackend 进程。每个进程包含一个 Executor 对象，该对象持有一个线程池，每个线程可以执行一个 task。
* 每个 application 包含一个 driver 和多个 executors，每个 executor 里面运行的 tasks 都属于同一个 application。

在最开始的时候我们介绍了RDD的`action`操作，还提到每个`action`就是一个job，事实上上表中的`action`操作其实是论文中的，实际上还有更多的`action`。如下表：

<table>
<thead>
<tr>
<th style="text-align:left">Action</th>
<th style="text-align:left">finalRDD(records) =&gt; result</th>
<th style="text-align:left">compute(results)</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">reduce(func)</td>
<td style="text-align:left">(record1, record2) =&gt; result, (result, record i) =&gt; result</td>
<td style="text-align:left">(result1, result 2) =&gt; result, (result, result i) =&gt; result</td>
</tr>
<tr>
<td style="text-align:left">collect()</td>
<td style="text-align:left">Array[records] =&gt; result</td>
<td style="text-align:left">Array[result]</td>
</tr>
<tr>
<td style="text-align:left">count()</td>
<td style="text-align:left">count(records) =&gt; result</td>
<td style="text-align:left">sum(result)</td>
</tr>
<tr>
<td style="text-align:left">foreach(f)</td>
<td style="text-align:left">f(records) =&gt; result</td>
<td style="text-align:left">Array[result]</td>
</tr>
<tr>
<td style="text-align:left">take(n)</td>
<td style="text-align:left">record (i&lt;=n) =&gt; result</td>
<td style="text-align:left">Array[result]</td>
</tr>
<tr>
<td style="text-align:left">first()</td>
<td style="text-align:left">record 1 =&gt; result</td>
<td style="text-align:left">Array[result]</td>
</tr>
<tr>
<td style="text-align:left">takeSample()</td>
<td style="text-align:left">selected records =&gt; result</td>
<td style="text-align:left">Array[result]</td>
</tr>
<tr>
<td style="text-align:left">takeOrdered(n, [ordering])</td>
<td style="text-align:left">TopN(records) =&gt; result</td>
<td style="text-align:left">TopN(results)</td>
</tr>
<tr>
<td style="text-align:left">saveAsHadoopFile(path)</td>
<td style="text-align:left">records =&gt; write(records)</td>
<td style="text-align:left">null</td>
</tr>
<tr>
<td style="text-align:left">countByKey()</td>
<td style="text-align:left">(K, V) =&gt; Map(K, count(K))</td>
<td style="text-align:left">(Map, Map) =&gt; Map(K, count(K))</td>
</tr>
</tbody>
</table>

用户的 driver 程序中一旦出现 action()，就会生成一个 job，比如 foreach() 会调用sc.runJob(this, (iter: Iterator[T]) => iter.foreach(f))，向 DAGScheduler 提交 job。如果 driver 程序后面还有 action()，那么其他 action() 也会生成 job 提交。所以，driver 有多少个 action()，就会生成多少个 job。这就是 Spark 称 driver 程序为 application（可能包含多个 job）而不是 job 的原因。

每一个 job 包含 n 个 stage，最后一个 stage 产生 result。。在提交 job 过程中，DAGScheduler 会首先划分 stage，然后先提交无 parent stage 的 stages，并在提交过程中确定该 stage 的 task 个数及类型，并提交具体的 task。无 parent stage 的 stage 提交完后，依赖该 stage 的 stage 才能够提交。从 stage 和 task 的执行角度来讲，一个 stage 的 parent stages 执行完后，该 stage 才能执行。

Spark就先介绍到这里，没有涉及到spark的具体内部实现，包括job的调度，shuffle的过程、文件的管理、cache和broadcast机制等。
