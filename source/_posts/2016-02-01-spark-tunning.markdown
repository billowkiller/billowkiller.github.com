---
layout: post
title: "Spark Tunning"
date: 2016-02-01 17:23
comments: true
category: Big Data
tags: [spark, tunning]
---

记录一些spark的调优参数。

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-1/51172915.jpg" width="500px"/>

<!--more-->

1. 一个集群有N个节点，每个拥有a cores，mGB的memory

	* num-executor N*3-1， 每个节点包含3个executor，AM的那个节点两个
	* executor-cores min(5, a-1)
	* executor-memory   (m-1)/3 *  (1-spark.yarn.executor.memoryOverhead)，默认为0.07。

2. 并行化调优

		一个stage中task的数量，也就是处理数据的并行度。通常的配置为分配的core数目的2-3倍。task的数量过少会影响：
		
		* 在聚合操作上会产生更大的内存压力
		* 垃圾回收
		* 当记录在内存中无法装载时会spill to disk，磁盘IO和排序。
	
		解决办法：
		
		* 配置INputFormat产生更多的splits。
		* 用更小的block size将输入数据写到HDFS上。
		* repartition transformation, triggering a shuffle: `var rdd2 = rdd2.reduceByKey(_ + _, numPartitions = X)`
	
	* Spark的RDD的partition个数创建task的个数是对应的
	* Partition的个数在spark的RDD中由block的个数决定的
	* 尽可能地增加task的数量使得每个task的数据可以装入分配的内存中。
	Spark能够非常有效的支持段时间任务（例如200ms)，因为他会对所有的任务复用JVM，这样能减小任务启动的消耗。所以，可以放心的使任务的并行度远大于集群的CPU核数。
	
	spark.default.parallelism: 控制Spark中的分布式shuffle过程默认使用的task数量，默认为8个。如果不做调整，数据量大时，就容易运行时间很长，甚至是出Exception，因为8个task无法handle那么多的数据。
	
			spark.default.parallelism 300

2. 减少Java数据结构的消耗，内存中存储未序列化的java对象，硬盘和网络用序列化的数据。
	
	* 使用org.apache.spark.serializer.KryoSerializer
	* fastutil库为原始数据类型提供方便的集合类，且兼容Java标准类库。
	* 内存少于32G，设置JVM参数-XX:+UseCompressedOops以便将8字节指针修改成4字节，设置JVM参数-XX:+UseCompressedStrings以便采用8比特来编码每一个ASCII字符。

2. driver默认内存改成2G（原先1G）；

3. 默认开启预测执行，缓解慢节点带来的计算任务慢的问题：
    	
    		spark.speculation true
    		spark.speculation.quantile 0.75
    		spark.speculation.multiplier 1.5


5. 调整默认GC策略至G1 GC 

	需要测试一些数据来看一下环境里最适合的GC策略，测试结果显示G1没有提升
    	
    		CMS vs parallel GC vs G1 GC
    		G1 GC: InitiatingHeapOccupancyPercent、 ConcGCThreads
    	
6. timeout参数优化
    
    		spark.core.connection.ack.wait.timeout 
    		spark.core.connection.auth.wait.timeout
    		spark.akka.timeout 
    		spark.akka.askTimeout 
    		spark.shuffle.io.connectionTimeout 

7. frameSize 优化（控制Spark中通信消息的最大容量如 task 的输出结果，默认为10M）
    
    		spark.akka.frameSize 100

8. codec 优化（Cache和Shuffle数据压缩所采用的算法）
    
    		spark.io.compression.codec org.apache.spark.io.LZ4CompressionCodec

9. gc log打印
   	
   		-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -Xloggc:/var/logs/spark/spark_gc.log   

10. heap dump
 		
 			-xx:+HeapDumpOnOutOfMemoryError

11. 调整permsize
 
 			--driver-java-options "-XX:MaxPermSize=512m"
 		
12. native lzo

13. 减小数据结构内存

	内存少于32G，设置JVM参数-XX:+UseCompressedOops以便将8字节指针修改成4字节，设置JVM参数-XX:+UseCompressedStrings以便采用8比特来编码每一个ASCII字符
	
	
### Spark Terasort by Databricks

Spark sorted the same data **3X faster using 10X fewer machines**. All the sorting took place on disk (HDFS), without using Spark’s in-memory cache. 

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-1/9096505.jpg" width="600px"/>



