---
layout: post
title: "BigData Performance Tunning"
date: 2016-11-19 17:23
comments: true
category: NoteBook
tags: [tunning]
---

## 单节点优化

### 增加并发吞吐
* IO密集：多进程，多线程，异步，协程
* CPU密集：多进程，多线程

异步：回调机制、事件驱动

协程(同步模型开发的方式达到异步效果)：Ucontext(glibc)  Bthread（baidu rpc）

除非资源达到瓶颈，否则不排队

* 合适并发模型
* 队伍要均衡
* 过长的队伍，及时柔性处理（可丢服务）

*socket队列改成request队列*

<!--more-->

### 去除不必要动作

* 减少网络重连（长连接）
* 减低连接数（连接池）
* 减少线程切换（线程池）
* 减少内存分配和释放（内存池）
* 减少耗时的操作和运算（memset, 浮点运算，除法，指数，对数运算，慎用stl）
* 在线转离线（离线生成词典）

### 避免冲突

* 多线程无锁算法
    * 无锁共享数据
    * copy on write
    
* hash冲突
    * 桶如何分配
    * 如何减少hash冲突
    
* 合理使用锁
    * 所得时间尽可能短
    * 减低冲突概率
    * 避免死锁   

### IO优化

顺序io 600MB/s VS 随机io 100KB/s

随机修改：

* WAL
* LSM-Tree
* 批量去重，减低读写次数（后链入库，hold去重）

随机读取：

* 减少IOPS
* 优化cache，预热cache（发些虚假的query）
* SSD

kafka

* 顺序写磁盘效率比随机写内存还要高，高吞吐
* 重复利用page cache， 直接内存读取直接发送


## 集群优化

### 减低数据传输量
* 数据压缩，cpu和网络io权衡
* 减少交互次数（数据量也降低 header）
* 打包访问
* 减少跨机io

### 均衡

* 负载均衡
  * RR, Random， locality-aware，hash（固定后端）
  
* 热点+打散
    * 自动拆分和融合节点
    * 自动伸缩容量，弹性
* 消除长尾（quorum，预测模式）
* 消峰，限流，缓冲 + 延迟处理（优先级机制）
* 丢弃，降级处理 

### cache

有结果cache、【无结果cache、超时cache】反着玩
只读、读写cache

## 系统优化

容器+混布

全量模型 增量模型

避免局部瓶颈，每个子系统的扩展性很重要

木桶效应

