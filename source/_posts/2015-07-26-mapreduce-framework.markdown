---
layout: post
title: "mapreduce framework"
date: 2015-07-26 17:23
comments: true
category: Big Data
tags: [mapreduce]
---

##MapReduce框架

mapReduce 的输入是hdfs上存储的一系列文件集。在hadoop中，这些文件被一种定义了如何分割一个文件成分片的input format来分割，一个分片是一个文件基于字节的可以被一个map任务加载的一个块。

* 每个map任务被分为以下阶段：`record reader`，`mapper`，`combiner`，`partitioner`。Map任务的输出叫中间数据，包括keys和values，发送到reduce端。
* Reduce任务分为以下阶段：`shuffle`，`sort`，`reduce`，`output format`。运行map任务的节点会尽量选择数据所在的节点。这种情况下，不会出现网络传输，在本地节点就可以完成计算。

过程如图所示，接下来的章节会一一介绍。

![](http://7xqfqs.com1.z0.glb.clouddn.com/16-7-3/75294350.jpg)

<!--more-->

### Input Format

Record reader会把根据input fromat生成输入分片翻译成records。Record reader的目的是把数据解析成记录，而不是解析数据本身。它把数据以键值对的形式传递给mapper。通常情况下键是偏移量，值是这条记录的整个字节块。从Input file到map的中间过程如下图所示

<img src="http://i12.tietuku.com/691b0fd1648b0497.png" width="450px" />

InputFormat其实做了三件事：

* 校验job的input configuration（比如，查看数据是否存在）。
* split输入的数据文件为逻辑上分片InputSplit，每个InputSplit给接下来的map task处理。
* 创建RecordReader从InputSplit中解析出一个个键值对，这个键值对就是Record。

如果要自定义InputFormat则最重要的是两个方法：

* `public List<InputSplit> getSplits(JobContext context)`
* `public RecordReader createRecordReader(InputSplit split, TaskAttemptContext context)`

通常在处理文本文件的时候，为了保证记录的完整性，RecorderReader会读取超过InputSplit边界的数据。

![](http://i5.tietuku.com/392e9f9f99737b4d.jpg)

在上图中共有三个InputSplit，在hadoop中默认的InputSplit大小为HDFS中每个Block的大小，所以共产生三个map task，它们读取数据的情况如下：

|   | Start | Actual Start | End | Line(s) |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Mapper1  | B1:0  | B1:0 | B2:150 | L1, L2, L3 |
| Mapper2  | B2:128  | B2:150 | B3:300 | L4, L5, L6 |
| Mapper3  | B3:256  | B3:300 | B3:300 | N/A |

现有的InputFormat包括：

* TextInputFormat，Hadoop中默认的InputFormat，每行都是一个record，以偏移量为key
* KeyValueTextInputFormat， 可以指定key value分割符的TextInputFormat
* NLineInputFormat, mapper接受固定行数的记录
* SequenceFileInputFormat，二进制的KV
	* SequenceFileAsTextInputFormat，文本形式的KV
	* SequenceFileAsBinaryInputFormat
	* FixedLengthInputFormat
* MultipleInputs
* DBInputFormat

### Map

Map阶段，会对每个从RecordReader读取的Record键值对执行用户代码，这些键值对又叫中间键值对。键和值的选择不是任意的，并且对MapReduce job的成功非常重要。键会用来分组，值是reducer端用来分析的数据。。

### Combiner
Combiner 是一个map阶段分组数据，可选的，局部reducer。它根据用户提供的方法在一个mapper范围内根据中间键去聚合值。例如：数的总和是各个部分数量的和，你可以先计算中间的数目，最后再把所有中间数目加起来。很多情况下，这样能减少数据的网络传输量。发送（hello world，1）三次很显然要比发送（hello world，3）需要更多的网络传输字节量。

<img src="http://i12.tietuku.com/009804b71667c5b9.png" width="500px" />

上图所示就是combiner发生的时机，它发生在map side写入磁盘的时候，可能会发生两次，一次是在Spill的时候，一次是在Merge的时候。在这两个阶段之前，是有一个sort的过程，这是为了最大化地提高combiner效率。其实从结构和作用来看，combiner函数和reducer函数的作用是相同的，很多情况下，Combiner也是直接用Reducer。

### Partitioner

Partitioner很简单，它决定哪个reducer接收该mapper数据记录，默认的是一个hash函数，对key进行hash之后取reducer个数的模。Partitioner会获取从mapper（或combiner）来的键值对，并分割成分片，每个reducer一个分片。默认用哈希值，典型使用md5sum。然后partitioner根据reduce的个数执行取余运算：key.hashCode() % (number of reducers)。这样能随即均匀的根据key分发数据到reduce，但仍然要保证不同mapper的相同key要到同一个reduce。Partitioner也可以自定义，使用更高级的样式，例如排序。然而，更改partitioner很少用。Partitioner的每个map的数据会写到本地磁盘，并等待对应的reducer检测，拿走数据。

### Shuffle and sort
虽然说这个阶段主要是在Reduce端，但是Map端的一些行为也会影响到Reduce端。

<img src="http://i5.tietuku.com/952140624345e77f.png" width="500px" />

上图是Map端的Shuffle过程，从整个过程来看，最重的部分在于Spill和Merge这两个I/O相关的操作，所有的数据需要从磁盘中读取后又重新写入到磁盘中，所以理想情况下是能够将所有mapper的output都装入到内存中。

Reduce任务开始于shuffle和sort阶段，Reduce将会开启多个fetcher从每个节点上的shuffle service中获取数据流。这一阶段获取partitioner的输出文件，并下载到reduce运行的本地机器。在下载的过程中，map的output首先会写入到内存中并进行排序，在数据达到一定量之后spill到磁盘，会有一个后台程序不断merge这些spilled file。最终，所有的output会合并成一个大的文件。这一阶段不能自定义，由框架自动处理。需要做的只是key的选择和可以自定义个用于分组的比较器。整个过程如下图所示。

<img src="http://i5.tietuku.com/d11419d045f44d9a.png" width="500px" />

### Reduce
Reduce 任务会把分组的数据作为输入并对每个key组执行reduce方法代码。方法会传递key和可以相关的所有值得迭代集合。很多的处理会在这个方法里执行，也就会有很多的模式。一旦reduce方法完成，会发送0或多个键值对到output format。跟map一样，不同的reduce依据不同的逻辑情形而不同。

### Output format
Output Format会把reduce阶段的输出键值对根据record writer写到文件里。默认用tab分割键值对，用换行分割不同行。这里也可以自定义为更丰富的输出格式，最后，数据被写到hdfs。整个过程类似于InputFormat。

<img src="http://i12.tietuku.com/cbdb549a3e19f898.png" width="500px" />

LazyOutputFormat 用来保证output (part-r-nnnnn) files有数据，不会存在空文件。

### Output Commiter

Hadoop使用`OutputCommitter`来保证作业和任务的事务性。在旧的API中需要显示的使用`setOutputCommitter`或者设置`mapred.output.committer.class`。
而在新的API中，`OutputCommitter`是由`OutputFormat`通过`getOutputCommitter()`方法决定的。默认的是`FileOutputCommitter`，它适用于所有的基于文件的MapReduce。

`OutputCommitter`API如下：

	public abstract class OutputCommitter {
		public abstract void setupJob(JobContext jobContext) throws IOException; 
		public void commitJob(JobContext jobContext) throws IOException { } 
		public void abortJob(JobContext jobContext, JobStatus.State state) throws IOException { }
		public abstract void setupTask(TaskAttemptContext taskContext) throws IOException;
		public abstract boolean needsTaskCommit(TaskAttemptContext taskContext) throws IOException;
		public abstract void commitTask(TaskAttemptContext taskContext) throws IOException;
		public abstract void abortTask(TaskAttemptContext taskContext) throws IOException;
	}
	
`setupJob`方法在job运行前就被调用，用来服务于作业的初始化，类似创建作业和task的临时目录。

job成功后会调用`commitJob()`方法，用于删除临时目录和创建*_SUCCESS*文件。如果job失败，则调用`abortJob()`方法
表示作业失败或者被kill，默认情况下会删除临时目录。

在task级别的操作类似。Hadoop框架会保证在一个task中的多个task attempt中，只有一个会commit，其他abort。有两种情况：

* 第一个attempt失败的时候会abort，第二个attempt如果成功则会commit。
* 对于预测任务，只要有一个首先成功的话会commit，另外一个则abort。

