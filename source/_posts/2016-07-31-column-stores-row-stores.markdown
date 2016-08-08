---
layout: post
title: "Column-Stores vs. Row-Stores"
date: 2016-07-31 17:23
comments: true
category: Paper Weekend
tags: [storage, paper]
---

> Abadi et al. SIGMOD 2008

在数据分析领域，例如数据仓库、决策支持、BI应用等，面向列的存储结构通常会比面向行的存储结构表现要好一个数量级以上，因为只需要读取需要的属性，列存储对于只读的查询 IO 更高效。本文否定了用列存储的思想优化行存储的方式，提出对两种存储方式都有效的优化，并探讨列存储真正高效的原因。

<!--more-->

### Row vs. column

<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-31/39462037.jpg" width="500px"/>

Row store 更加适合添加和修改一条记录，但读取不必要的数据；Column Store 适合读取相关数据，但写的时候需要多次磁盘IO。所以 Column stores 适合读密集型的数据集。对比如下：

<table border="1" cellspacing="0" cellpadding="0"> <tbody><tr>  <td valign="top" style="background:#5B9BD5;"><p align="left"><strong><span style="color:white;">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; </span></strong></p></td>  <td valign="top" style="background:#5B9BD5;"><p align="center"><strong><span style="color:white;">行式存储</span></strong></p></td>  <td valign="top" style="background:#5B9BD5;"><p align="center"><strong><span style="color:white;">列式存储</span></strong></p></td> </tr> <tr>  <td valign="top" style="background:#DEEAF6;"><p align="center"><strong>优点</strong></p></td>  <td valign="top" style="background:#DEEAF6;"><p align="left">Ø&nbsp; 数据被保存在一起</p>  <p align="left">Ø&nbsp; INSERT/UPDATE容易</p></td>  <td valign="top" style="background:#DEEAF6;"><p>Ø&nbsp; 查询时只有涉及到的列会被读取</p>  <p>Ø&nbsp; 投影(projection)很高效</p>  <p>Ø&nbsp; 任何列都能作为索引</p></td> </tr> <tr>  <td valign="top"><p align="center"><strong>缺点</strong></p></td>  <td valign="top"><p align="left">Ø&nbsp; 选择(Selection)时即使只涉及某几列，所有数据也都会被读取</p></td>  <td valign="top"><p>Ø&nbsp; 选择完成时，被选择的列要重新组装</p>  <p>Ø&nbsp; INSERT/UPDATE比较麻烦</p></td> </tr></tbody></table>

通常来说它比 Row store 在某些应用中更快的原因有：

* 只读取需要的列
* 更好的缓存有效性
* 适合压缩

### Row-oriented Execution

首先看下在商用的行存储DBMS中如何实现列数据库：

1.  Vertical Partitioning

	> The most straightforward way to emulate a column-store approach in a row-store is to fully vertically partition each relation.
	
	垂直切割表格形成两列的tuple表（table key, attribute）, 查询的时候直接访问必要的列。
	
	<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-31/64076524.jpg" width="450px"/>
	
	遇到的问题： 需要添加 Position 列，浪费磁盘和带宽；Tuple 有额外 Header，例如 PostgreSQL 里有 24 bytes。

2. Index-only Plan

	> base relations are stored using a standard, row-oriented design, but an additional unclustered B+Tree index is added on every column of every table.
	
	构造在query中所需要用到的所有列的数据块的集合，所以查询的时候根本不需要查询底层的（按行存储的）表。
	
	<img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-31/2612463.jpg" width="450px"/>
	
	遇到的问题：分开的数据块如果需要全表扫表的话，会很慢。
	
3. Materialized Views
	
	> Create optimal set of MVs for given query workload to provide just the required data, avoid overheads and perform better.
   使用query中所需要用到的列的物化视图集合，尽管使用很多空间，但是可以得到更好的效率，典型的空间换时间。
   
   <img src="http://7xqfqs.com1.z0.glb.clouddn.com/16-7-31/44662651.jpg" width="300px"/>
   
   遇到的问题：适用性比较窄；需要知道查询的先验知识。
  

### Column-oriented Execution

下面来看下面向列存储的一些优化方法：

1. Compression

	> Low information entropy (high data value locality) leads to High compression ratio.
   
   这个比较重要，有结果显示压缩和不压缩性能上会差别一个数量级，具体的好处是：
   
   * 节省磁盘空间
   * 减少磁盘和网络IO
   * 如果能在压缩数据上面做计算，那 CPU 消耗也可以降低

   几个注意点：
   
   * 注意压缩率和解压效率的权衡，使用轻量级的压缩算法，牺牲一些压缩率。
   * 使用类似run-length encoding (1,1,1,2,2 -> 1*3,2*2)，可以直接在压缩数据上做计算。
   * 特别是对于排好序的数据，压缩效率会更高。

2. Late Materialization

	通常来说，查询结果是要得到一个 entity，而不是一个 column，所以需要对多个列进行 join。比较朴素的做法是，数据按列存储在磁盘上，当一个query需要多个 column 的时候，会从这些属性中构造 tuples，接着进行一些其他操作（select，aggregate，join...）。虽然这样做仍然会优于 row-oriented 的存储，但是会有许多优化空间，使用延迟物化就是一个。
	
	举一个例子，`SELECT R.a FROM R WHERE R.c = 5 AND R.b = 10`。首先，`R.c` 和 `R.b` 的输出是一个 bit string，这个其实就是中间结果 position lists，对着两个结果进行 `bitwise and`，最后这个 final position list 提取 `R.a`。
	
	带来的好处有：
	
	* 非必要的 tuple 构建省略了（selection 和 aggregation带来的）
	* 可以直接对着压缩数据进行操作
	* 更好的 cache performance，没有其他属性的数据污染 cache
	* 可以结合下一个优化点

3. Block Iteration

	> Operators operate on blocks of tuples at once like batch processing.
	
	对 Block 而不是 Tuple 进行操作，这个同样可以应用在 Row-oriented stores 上，带来的好处有：
	
	* 如果列是固定宽度的，则可以变成对数组的操作
	* 减少 tuple 的 overhead
	* 有效利用并行化

	
### 结论
	
Vertical portioning 和 Index only plan 并不能起到很好的效果，Materialized view 表现的最好。
	
* Block processing improves the performance by a factor of 5% to 50%* Compression improves the performance by almost a factor of two on avg* Late materialization improves performance by almost a factor of three 

