---
layout: post
title: "Hadoop Introduction"
date: 2015-12-20 17:23
comments: true
category: Big Data
tags: [mapreduce, hadoop]
---


看图说话，用一些图表来记录Hadoop。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/22389512.jpg" width="500px"/>

<!--more-->

### MapReduce

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/71525619.jpg" width="500px"/>

schema-on-read表示hadoop更容易处理非结构化或半结构化的数据，因为可以在数据处理的时候进行解析，因此在加载数据的时候更加轻量化。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/99564088.jpg" width="500px"/>

### HDFS

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/60582298.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/98566811.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/1953976.jpg" width="500px"/>

NameNode维护整个文件系统的文件目录树，文件目录的元信息和文件的数据块索引。这些信息以 FSImage 和 EditLog 方式存储在本地文件系统中。 FSImage 是某一时刻的镜像，后续修改都是放在 EditLog 中。 Second NameNode 会间隔将 FSImage 和 EditLog 进行合并后放在 NameNode 上。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/94295030.jpg" width="500px"/>

### Yarn

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/80769446.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/31902359.jpg" width="400px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/37369292.jpg" width="500px"/>

1. Fair Scheduler
	Facebook开发的适合共享环境的调度器，支持多用户多分组管理，每个分组可以配置资源量，也可限制每个用户和每个分组中的并发运行作业数量；每个用户的作业有优先级，优先级越高分配的资源越多。

2. Capacity Scheduler
	Yahoo开发的适合共享环境的调度器，支持多用户多队列管理，每个队列可以配置资源量，也可限制每个用户和每个队列的并发运行作业数量，也可限制每个作业使用的内存量；每个用户的作业有优先级，在单个队列中，作业按照先来先服务（实际上是先按照优先级，优先级相同的再按照作业提交时间）的原则进行调度。

**Fair Scheduler vs Capacity Scheduler**

* 相同点

	* 均支持多用户多队列，即：适用于多用户共享集群的应用环境
	* 单个队列均支持优先级和FIFO调度方式
	* 均支持资源共享，即某个queue中的资源有剩余时，可共享给其他缺资源的queue

* 不同点
	
	* 核心调度策略不同。 计算能力调度器的调度策略是，先选择资源利用率低的queue，然后在queue中同时考虑FIFO和memory constraint因素；而公平调度器仅考虑公平，而公平是通过作业缺额体现的，调度器每次选择缺额最大的job（queue的资源量，job优先级等仅用于计算作业缺额）。

	* 内存约束。计算能力调度器调度job时会考虑作业的内存限制，为了满足某些特殊job的特殊内存需求，可能会为该job分配多个slot；而公平调度器对这种特殊的job无能为力，只能杀掉这种task。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-5/75633723.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-5/56706884.jpg" width="450px"/>

### HADOOP IO

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/15159126.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/90632065.jpg" width="500px"/>

其他流行的兼容Hadoop的序列化框架还有：Avro、Thrift、google protobuf


<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/11927837.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/70087429.jpg" width="500px"/>

### HADOOP JOBS

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/38335644.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/37931472.jpg" width="500px"/>

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-6-29/10857296.jpg" width="500px"/>

