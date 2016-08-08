---
layout: post
title: "Realtime Data Processing at Facebook"
date: 2016-07-13 17:23
comments: true
category: Paper Weekend
tags: [realtime, paper]
---

> Chen et al. SIGMOD 2016

论文说明Facebook是如何建立实时系统的。有段话说明batch和stream的关系：

> Streaming versus batch processing is not an either/or decision. Originally, all data warehouse processing at Facebook was batch processing. We began developing [some of our streaming systems] about five years ago… using a mix of streaming and batch processing can speed up long pipelines by hours. Furthermore, streaming-only systems can be authoritative. We do not need to treat realtime system results as an approximation and batch results as “the truth.”

<!--more-->

典型的实时系统有Twitter的`Storm`和`Heron`、Google的`Millwheel`，LinkedIn的`Samza`。Facebook有`Puma`、`Swift`、`Stylus`，在实时系统中他们有以下几方面的考虑：

1. 易用性：处理要求有多复杂，sql是否足够，还是需要通用语言？用户对新应用的编写，测试、调试和部署有多快，如何对应用进行监控？
2. 性能：延迟，吞吐量，每台机器的以及聚合时候的？
3. 容错性：容忍错误种类，数据处理和输出时的语义保证，系统如何存储恢复内存状态？
4. 扩展性：数据分片以及重新分片的并行处理，数据量增加和减小时候系统如何处理，对于老数据的重放？
5. 正确性：ACID保证？


数据处理需求允许秒级的延迟，所以fb中的数据传输是通过一个持久存储的；分离传输和处理带来的好处是易用、容错和可扩展以及一部分正确性保证。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-13/63234135.jpg" width="500px">

* scribe

    Scribe从各种数据源上收集数据，放到一个共享队列上，然后push到后端的中央存储系统上。当中央存储系统出现故障时，scribe可以暂时把日志写到本地文件中，待中央存储系统恢复性能后，scribe把本地日志续传到中央存储系统上。
需要注意的是，各个数据源须通过thrift向scribe传输数据（每条数据记录包含一个category和一个message）。可以在scribe配置用于监听端口的thrift线程数。在后端，scribe可以将不同category的数据存放到不同目录中，以便于进行分别处理。后端的日志存储方式可以是各种各样的store，包括file（文件），buffer（双层存储，一个主储存，一个副存储），network（另一个scribe服务器），bucket（包含多个store，通过hash的将数据存到不同store中），null(忽略数据)，thriftfile（写到一个Thrift TFileTransport文件中）和multi（把数据同时存放到不同store中）

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-14/99114695.jpg" width="500px"/>

* puma 使用支持Java UDF的类sql语言，目的在于快速开发部署
* swift 使用脚本语言支持流式处理，简单的API和checkpointing。
* stylus C++编写，类似Strom的流处理引擎。
* Laser 高吞吐低延迟的内存KV存储，基于RocksDB（基于Google的LevelDB）[benchmark](https://influxdata.com/blog/benchmarking-leveldb-vs-rocksdb-vs-hyperleveldb-vs-lmdb-performance-for-influxdb/)
* Scuba fast slice-and-dice analysis data store，支ad-hoc query和可视化
* Hive  数据仓库

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-13/55531065.jpg" width="600px"/>

* Language paradigm
    * 声明语言，SQL，编写作为最简单和快速，群众基础好。缺点是表达能力不够。
    * 函数语言，使用预制的算子串联处理过程，简单并且对算子的组合增加表达能力。
    * 过程语言，C++，JAVA，Python，灵活，性能最高，对数据结构和执行过程有完全掌控力，但是要求也最高。S4/Storm/Heron/Samza。
    
* Data transfer
    * 直接传输，RPC或者内存消息队列。延迟少，毫秒级。
    * Broker，另外一个broker进程处理数据传输，增加overhead但是扩展性好。支持多对多的传输，支持反压。典型的是Heron。
    * 持久存储，persistent message bus。多路复用，输入输出不同的速度，不同的时间点，重复读。
    
    邻近节点处于DAG中，两种依赖关系，窄依赖和宽依赖，实现在数据传输中。
    
* Processing semantics

    stream processor有三种行为：
    
    * 处理input event，反序列化输入event，查询外部系统，更新内存状态，并没有side-effect
    * 输出产生，根据input event和内存状态，向下游产生输出。可以在检查点之前或之后。
    * 保存检查点，内存状态、input stream offset、output value
    
    根据这些行为的实现方式，有两种语义信息：
    
    * 状态语义，input event是at-least once, at-most once, exactly
    * 输出语义，output value....
    
    无状态的处理器只有输出语义，有状态的有两种语义。状态语义只依赖于保存offset和内存状态的顺序。
    
    * at-least-once，先内存状态后offset
    * at-most-once，先offset，再内存状态
    * exactly，原子的
    
    输出语义依赖数据保存到checkpint的顺序：
    
    * at-least-once，先emit output再offset和内存状态
    * at-most-once，先offset和内存状态，再emit output
    * exactly，原子的事务性
    
    可以做一些side-effect-free的操作加速吞吐量，例如保存checkpoint的同时进行反序列化。
    
* State-saving mechanism 

     * Replication
     * local database persistence
     * remote database persistence
     * upstream backup
     * global consistent snapshot
 
* Backfill processing

    * stream only.
    * 保存两个系统，一个用于batch，一个用于流处理
    * 开发流处理系统可以运行在batch环境中，spark streaming 和 flink. 
    
### Lessons learned

1. 拥有多个不同的实时处理系统是有好处的. “Writing a simple application lets our users deploy something quickly and prove its value first, then invest the time in building a more complex and robust application.”
2. 回放对于debug来说很重要.
3. “Writing a new streaming application requires more than writing the application code. The ease or hassle of deploying and maintaining the application is equally important. Laser and Puma apps are deployed as a service. Stylus apps are owned by the individual teams who write them, but we provide a standard framework for monitoring them.” Making Puma apps self-service was key to scaling to the hundreds of data pipelines using Puma today.
4. 使用告警来检查应用的处理速度低于输入速度，未来考虑自动缩放。
5. Streaming versus batch processing does not need to be an either/or decision.
6. 易用性很重要.


