---
layout: post
title: "Hadoop Tunning"
date: 2015-12-01 09:50
comments: true
category: Big Data
tags: 
---

Hadoop的调优涉及到整个MapReduce的各个过程，并且要对每个参数的意义和Counter信息有一定的了解。也就是说，根据Counter的信息推测哪些MapReduce阶段可能存在性能瓶颈，并且根据这个瓶颈理解对应的Hadoop 框架中的处理逻辑，进而可以调整相关参数的大小或程序的行为。

从大面上来看，MapReduce的瓶颈可能存在以下图中的各个部分。

<img src="http://ww2.sinaimg.cn/large/74311666jw1eyjx8zpfz2j20rq0cegna.jpg" width="500px" />

<!--more-->

在开始调优之旅前，先来个开胃小菜。进行调优的过程中，我们首先要知道整个job的运行情况。

Hadoop 2中的JobHistory是能够提供作业运行时各个参数指标的展示工具，可以通过这个UI界面查看所有的Counter。另一方面如果不方便通过界面的方式查看，则可以利用Hadoop自带的命令行工具查看，方法是`hadoop job -history <history file>`, 这个history file的位置通常是在mapreduce.jobhistory.done-dir目录下，可以用如下方式查找`hadoop fs -lsr <done-dir> | grep job_1398974791337_0037`。

## Map Optimizations

在Map端的优化通常会涉及到输入的数据和它的处理过程，以及你的应用程序代码。Mapper需要读取作业的输入，输入文件的不同也会影响到作业运行的效率，例如文件是否是splittable，数据的本地性和input split的数量等等。

### Data Locality

在分布式计算中有条著名的准则**Pushing compute to the data**，map的task应该尽可能的被安排在数据存放的节点上。可以用Counter来判断作业是否符合这条准则:

* HDFS_BYTES_READ：这个值应当不大于input file的block size
* DATA_LOCAL_MAPS：这个值应当为1
* RACK_LOCAL_MAPS：这个值应当为0

以下几种情况可能会发生non-local read:

* 不能分割的大文件，这样mapper就必须从不同的节点中读取blocks。
* 文件格式支持split，但是用的input format不对。典型的情况是LZOP格式，需要先建立索引后再进行读取。
* Yarn的调度器不能在某个节点上产生map container，通常是由于集群under load。

解决方法是：

* 尽量保持unsplittable文件的大小接近一个block的大小。
* 设置yarn的`scheduler.capacity.node-locality-delay`，引入跳过的调度次数，来增加map task分配到数据节点上的概率。
* 使用Twitter提供的LZO Input Format来处理lzop数据，或者使用bzip2格式的文件。

### Map Number Overwhelm

当输入有很多的input split的时候，每个input split都需要一个mapper来执行，而每个mapper都是一个单独的进程。这样会给调度器和集群带来极大的压力。原因通常有两个：

* input data由很多的小文件组成，Hadoop会为每个小文件生成一个mapper，最终时间会大量消耗在启动进程上。
* 每个文件并不是很小（和block size相当），但总体的数据量很大，横跨上千个HDFS的blocks。这样每个block也会分配给单独的mapper。

如果是第一种情况，可以先聚合这个小文件，或者使用类似avro的文件格式来存储。或者直接用`CombineFileInputFormat`来处理以上这两种情况，它可以在一个mapper里处理多个HDFS block的数据。`CombineFileInputFormat`会首先检查input files的所有blocks，简历每个block到data nodes的映射关系，接着将同一个节点上的blocks聚合到一个input split中以保持data locality。它有三个配置项来调整：

* mapreduce.input.fileinputformat.split.minsize.per.node
* mapreduce.input.fileinputformat.split.minsize.per.rac
* mapreduce.input.fileinputformat.split.maxsize

以上的默认配置会造成每个节点上尽量形成一个最大的input split，影响作业的并行性，可以用以上几个配置来调整。`CombineFileInputFormat`还包括两个具体的类：

* `CombineTextInputFormat`
* `CombineSequenceFileInputFormata`

### Input Split Computation

如果提交作业的client是在集群局域网之外，那么input split的计算可能带来高成本。

当输入的数据源是HDFS时，client需要做如下事情，包括file listing, file status retrieving，input files数量比较多的时候，整个过程带来数据传输的延迟是比较可观的。

可以通过设置`yarn.app.mapreduce.am.compute-splits-in-cluster`将input split的计算交给AppMaster处理，这是在集群内部进行的。

## Shuffle Optimizations

### Using the Combiner

combiner可以有效的减少mapper和reducer之间通信的数据量。

### Using Binary Comparators

MapReduce在做sorting或者merging的时候，使用`RawComparator`比较map output key。内置的`Writable` classes（`Text`、`IntWritable`）有byte-level的比较器，无需将二进制的数据重新组装成实际的对象，所以能够快速进行序列化对象的比较。

用户可以在自己构造的`Writable`对象里面实现`WritableComparable`接口，这处理起来会比较容易，但是另一方面要注意MapReduce中map output data是以byte的形式存储的，这会导致在shuffle和sort的阶段需要从byte到object的转化才可以完成对象的比较。

可以看到在Hadoop的内置`Writable`对象不仅实现了`WritableComparable`接口，还自定义继承自`WritableComparator`的比较器。`WritableComparator`有什么作用呢，可以看下它的一些方法申明。

    public class WritableComparator implements RawComparator {
        public int compare(byte[] b1, int s1, int l1,
                           byte[] b2, int s2, int l2); 
    }

可以看到这是byte-level的Comparator，`Writable`对象正是覆盖了这里面的compare方法。在内置的`Writable`对象中都实现了`WritableComparator`，所以无需担心内置对象的效率。但是自己所构造的对象也可以实现`WritableComparator`方法来提高效率。

例如一个拥有firstName和lastName的Person对象：


    private String firstName;
    private String lastName;

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeUTF(lastName);
        out.writeUTF(firstName);
    }

<img src="http://i5.tietuku.com/e5eba32886b5773d.png" width="600px" />

    public int compare(byte[] b1, int s1, int l1, byte[] b2, int s2,
                         int l2) {
        int lastNameResult = compare(b1, s1, b2, s2);
        if (lastNameResult != 0) {
            return lastNameResult;
        }
        int b1l1 = readUnsignedShort(b1, s1);
        int b2l1 = readUnsignedShort(b2, s2);
        return compare(b1, s1 + b1l1 + 2, b2, s2 + b2l1 + 2);
    }

    public static int compare(byte[] b1, int s1, byte[] b2, int s2) {
        int b1l1 = readUnsignedShort(b1, s1);
        int b2l1 = readUnsignedShort(b2, s2);
        return compareBytes(b1, s1 + 2, b1l1, b2, s2 + 2, b2l1);
    }

    public static int readUnsignedShort(byte[] b, int offset) {
        int ch1 = b[offset];
        int ch2 = b[offset + 1];
        return (ch1 << 8) + (ch2);
    }

### Tunning the shuffle internals

在mapper中，output record首先被存储在一个内存buffer中，当这个buffer增长到一定大小的时候，数据被spill到磁盘中的一个文件。整个过程持续到mapper完成所有的output record生成。过程如下：

<img src="http://i5.tietuku.com/952140624345e77f.png" width="500px" />

在整个阶段中，I/O相关的splling和merging是最耗时的，所以理想状况应该是所有的output数据都可以装入buffer中，这样只有一个文件被spill到磁盘。这对所有的作业来说自然是不大可能，但是如果mapper可以通过filter或者project的方法减少input data，那么可以好好调整下`mapreduce.task.io.sort.mb`的大小，因为这个数据直接关系buffer的大小。可以通过检查下面的Counters来调整map端的shuffle：

* MAP_OUTPUT_BYTES  用这个数据来估计是否可以调整`mapreduce.task.io.sort.mb`来装入map的output record
* SPILLED_RECORDS/MAP_OUTPUT_RECORDS  这两个数据的理想情况是一致的，表示只有一个spill发生。
* FILE\_BYTES\_READ/FILE_BYTES\_WRITTEN  比较这两个数据和MAP\_OUTPUT\_BYTES可以理解在splling和merging阶段发生的读写副作用

在reduce方面，map的output通过每个节点上运行的shuffle service进程被发送到对应的reducer。在reducer中，map output是被写入到一个内存buffer中，在数据接收的过程中buffer中的数据被排好序，并在到达一定数据量的时候写入磁盘。同时有一个后台进程负责不断merge这个小的spllied file到merged files中，当所有的fetcher获取了所有的outputs，会有一个最终的merging发生，这时候数据也就从merged files到reducer了。也就是如下图的这个过程：

<img src="http://i5.tietuku.com/d11419d045f44d9a.png" width="500px" />

通map端的调优一样，reduce端也是尽量将数据存入内存中，减少splling和merging发生的次数，但是这个过程并不如map端一样明显，因为数据是在边接收边merging的。默认情况下，无论数据是否可以装入内存中，splling总会发生，所以可以调整`mapreduce.reduce.merge.memtomem.enabled`为true启动memory-to-memory的merge。map端的Counter同样适用于reduce。

以下的参数可以用来调整Hadoop的shuffle行为：


| Name | Default | Map or Reduce | Description |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| mapreduce.task.io.sort.mb | 100 (MB)  | Map | The total amount of buffer memory in megabytes to use when buffering map outputs. This should be approximately 70% of the map task’s heap size. |
| mapreduce.map.sort.spill.percent | 0.8  | Map | Note that collection will not block if this threshold is exceeded while a spill is already in progress, so spills may be larger than this threshold when it is set to less than 0.5. |
| mapreduce.task.io.sort.factor | 10  | Map and Reduce | The number of streams to merge at once while sorting files. This determines the number of open file handles. Larger clusters with 1,000 or more nodes can bump this up to 100. |
| mapreduce.reduce.shuffle.parallelcopies | 5  | Reduce | The default number of parallel transfers run on the reduce side during the copy (shuffle) phase. Larger clusters with 1,000 or more nodes can bump this up to 20. |
| mapreduce.reduce.shuffle.input.buffer.percent | 0.7  | Reduce | The percentage of memory to be allocated from the maximum heap size to store map outputs during the shuffle. |
| mapreduce.reduce.shuffle.merge.percent | 0.66  | Reduce | The usage threshold at which an in-memory merge will be initiated, expressed as a percentage of the total memory allocated to storing in-memory map outputs, as defined by mapreduce.reduce.shuffle.input.buffer.percent. |
| mapreduce.reduce.merge.memtomem.enabled | false  | Reduce | If all the map outputs for each reducer can be stored in memory, then set this property to true. |

Shuffle的原则是使用filter和project减少数据量，使用combiner以及压缩map的output，尽可能的减少mapper和reducer质检传递的数据，减低IO带来的开销。这样之后再利用上述提到的参数来调整Shuffle的过程。

## Recuder Optimizations

### The Number of Reducers

大多数情况下map端的并行度是由框架根据input files和input format来自动决定的，但在Reduce端，reducer的数量是由用户自行决定的。太少的reducers意味着没能充分利用集群的资源，太多的reducer则会让调度器疲于奔命，如果没有太多的资源供所有reducer运行则会拖累reducer的执行效率。在一些使用场景中，不能避免的需要使用少量的reducer来运行作业，例如数据写入DB系统中。另外一些场景中需要确认数据是否会发生倾斜，以及如何partition的，数据量是否会让reducer发生OOM的情况。

## General tunning tips

* 压缩
* 使用类似于Avro或Parquet的数据格式存储数据，带来的好处是空间利用率，序列化和反序列化更加有效。
    * 在Hadoop中，text并不是一个有效的数据格式，空间利用率低，解析成本高，特别是用到正则的时候。
    * 尽可能考虑使用二进制的文件存储格式。

## Tunning tools

### stack dumps

ssh到task运行的机器上，执行下面的命令：

    ps aux | grep container_1393242034820_0001_01_000002
    kill -s SIGQUIT 554284
    kill -s SIGQUIT 554284
    kill -s SIGQUIT 554284

`SIGQUIT`信号的发送应该要间隔几秒，当JVM收到这个信号的时候会执行stack dump，这样可以了解到程序在这段时间内的运行情况。最后可以在task 的output file中查看dump出来的栈信息。


### Profiling Map and Reduce Task

可以使用HPROF结合一些Mapreduce job method来进行Profiling。HPROF是JVM内置的java profiling工具，Hadoop内置了对HPROF的支持。可以在driver中加入如下的代码：

    job.setProfileEnabled(true);
    job.setProfileParams(
        "-agentlib:hprof=depth=8,cpu=samples,heap=sites,force=n," +
            "thread=y,verbose=n,file=%s");
    job.setProfileTaskRange(true, "0,1,5-10");
    job.setProfileTaskRange(false, "");

在`setProfileParams`方法中设置的参数会在每个container中建立一个名为profile.out的文件，这个文件可以很容易被解析。可以通过ssh到目标机器查看或者通过JobHistory UI界面查看。

profile.out包括一些stack traces，还包括内存和CPU时间的信息。以下是一个profile.out文件：

<img src="http://i5.tietuku.com/ba1dcbebf426b1a8.png" width="600px" />

可以看出来第一个问题是在`String.split`这个方法的使用上，它采用正则表达式来分割字符串，这个是相当耗时的一个举措，可以用Apache Commons Lang library的`StringUtils.split`来替换。另外一个是发生在Text的构造上，可以只构造一个Text实例，采用`set`方法进行设置，这样会更加有效率。

需要注意，使用HPROF会给程序的执行带来额外的负担，需要持续的收集profiling的信息，所以在正常的运行过程中不应该加上。


