记录一些spark的调优参数

<!--more-->

1. 一个集群有N个节点，每个拥有a cores，mGB的memory
	* num-executor N*3-1， 每个节点包含3个executor，AM的那个节点两个
	* executor-cores min(5, a-1)
	* executor-memory   (m-1)/3 *  (1-spark.yarn.executor.memoryOverhead)，  默认为0.07。

2. driver默认内存改成2G（原先1G）；

3. 默认开启预测执行：
    	
    	spark.speculation true
    	spark.speculation.quantile 0.75
    	spark.speculation.multiplier 1.5

4. 调高默认parallelism 数量
    
    	spark.default.parallelism 300

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

7. frameSize 优化
    
    	spark.akka.frameSize 100

8. codec 优化
    
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



