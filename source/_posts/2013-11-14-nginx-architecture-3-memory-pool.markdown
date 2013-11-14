---
layout: post
title: "Nginx architecture(3)--memory pool"
date: 2013-11-14 15:12
comments: true
categories: rework
tags: Nginx architecture
---

## 1. 概述
	
Nginx里内存的使用大都十分有特色：申请了永久保存，抑或伴随着请求的结束而全部释放，还有写满了缓冲再从头接着写。这么做的原因也主要取决于Web Server的特殊的场景，内存的分配和请求相关，一条请求处理完毕，即可释放其相关的内存池，降低了开发中对内存资源管理的复杂度，也减少了内存碎片的存在。

<img src="http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/nginx_zps7d735b88.jpg" alt="Nginx logo"/>

<!--more-->

所以在Nginx使用内存池时总是只申请，不释放，使用完毕后直接destroy整个内存池。

Nginx的内存模型实现得很精巧，代码也很简洁。总体来说，所有的内存池遵守一个宗旨：申请大块内存，避免“细水长流”。主要实现代码在`ngx_palloc.h`和`ngx_palloc.c`文件中。使用内存池的好处是：

1. 在大量的小块内存的申请和释放的时候，能更快地进行内存分配；
2. 减少内存碎片，防止内存泄露；

这种方式处理起来缺点也是显而易见的，即申请一块大的内存比如会导致内存空间的浪费，但是比起频繁地`malloc`和`free`，这样做的代价是非常小的，典型的以空间换时间的做法。

## 2. 内存分配

nginx内存分配将内存需求分成了两种：a) 大块内存, b) 小内存。内存大小的判定依据是申请的内存是否比同页大小与pool的size两者都小。

对于大块内存，单独利用malloc来申请，并且使用单向链表管理起来；

对于小块内存，则从已有的pool数据区中划分出一部分出来，这里的内存划分出去后没有特殊的结构来保存，而是等到申请对象生命周期结束后一起释放。小块内存的存储方式非常类似于sk_buffer，通过tail，end指针来表示多少内存已经被分配出去。

下图是linux kernel中数据结构sk_buff的使用示意图。具体可见[http://linux.chinaunix.net/techdoc/system/2008/08/04/1023273.shtml](http://linux.chinaunix.net/techdoc/system/2008/08/04/1023273.shtml)
![linux kernel struct sk_buff](http://blogimg.chinaunix.net/blog/upfile/070330110745.jpg)

nginx中分配内存时总是先判断申请内存是否属于大块内存，如果是则调用`ngx_palloc_large`申请大内存；如果是小内存则在已经存在的内存池中分配。

内存池的作用在于解决小块内存池频繁申请问题，对于大块内存是可以忍受直接申请的，这块内存会直接挂在内存池头部的large字段下。nginx中每块大内存都对应有一个头部结构，这个头部结构式用来将所有大内存串成一个链表用的。这个头部结构不是直接向操作系统申请的，而是单做小块内存直接在内存池中申请的。这样的大块内存在使用完后，可能需要第一时间释放，节省内存空间。

nginx专门提供了接口函数用来释放内存池中的某个大块内存，但它只会释放大内存，不会释放其对应的头部结构，毕竟头部结构式当做小内存在内存池中申请的；遗留下来的头部结构会作为下次申请大内存之用。

![nginx内存分配流程](http://blog.chinaunix.net/attachment/201306/16/24830931_1371367358NKxn.jpg)

## 3. 深入内存池

先来看看内存池中的数据结构以及之间关系的示意图。

![](http://images.cnblogs.com/cnblogs_com/xiekeli/201210/201210171140226537.png)

从这张图中可以看到`ngx_pool_data_t`和`ngx_pool_s`的结构。可以看出，nginx的内存池实际是一个由`ngx_pool_data_t`和`ngx_pool_s`构成的链表。

主要的数据结构为：

	`ngx_pool_data_t`中：
	
	last：是一个unsigned char 类型的指针，保存的是当前内存池分配到末位地址，即下一次分配从此处开始。
	
	end：内存池结束位置；
	
	next：内存池里面有很多块内存，这些内存块就是通过该指针连成链表的，next指向下一块内存。
	
	failed：内存池分配失败次数。

---

	`ngx_pool_s`
	
	d：内存池的数据块；
	
	max：内存池数据块的最大值；
	
	current：指向当前内存池；
	
	chain：该指针挂接一个ngx_chain_t结构；
	
	large：大块内存链表，即分配空间超过max的情况使用；
	
	cleanup：释放内存池的callback
	
	log：日志信息

 

以上是内存池涉及的主要数据结构，内存池对外的主要方法有：

|| **创建内存池**	 || ngx_pool_t *  ngx_create_pool(size_t size, ngx_log_t *log); ||
|| **销毁内存池**	 || void ngx_destroy_pool(ngx_pool_t *pool); ||
|| **重置内存池**	|| void ngx_reset_pool(ngx_pool_t *pool); ||
|| **内存申请（对齐）** || void *  ngx_palloc(ngx_pool_t *pool,  size_t size); ||
|| **内存申请（不对齐）** || void *  ngx_pnalloc(ngx_pool_t *pool, size_t size); ||
|| **内存清除** || ngx_int_t  ngx_pfree(ngx_pool_t *pool, void *p); ||


需要说明的是，内存池中并没有真正意义调用malloc等函数申请内存，而是移动指针标记而已，所以`内存对齐`的活，C编译器帮不了你了，得自己动手。并且不同的操作系统对齐方式不同。

这几个函数具体的功能可以见[http://www.cnblogs.com/xiekeli/archive/2012/10/17/2727432.html](http://www.cnblogs.com/xiekeli/archive/2012/10/17/2727432.html)