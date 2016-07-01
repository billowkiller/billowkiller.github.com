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

