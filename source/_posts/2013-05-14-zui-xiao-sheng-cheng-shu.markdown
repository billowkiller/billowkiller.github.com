---
layout: post
title: "最小生成树"
date: 2013-05-14 01:09
comments: true
categories: algorithm
tags: [algorithm, spanning tree]
---

算法使用的是二叉堆，时间为O(ElgV)。如果V小于E的话，使用Prim更好。

Kruskal算法：

<p>O(ElgE): E&lt;V<sup>2&nbsp;</sup>,所以有 lgE=O(lgV)</p>


集合A是一个森林，加入集合A中的安全边总是图中连接两个不同连通分支的最小权边。

使用不相交集合数据结构。

测试边时，即测试两端点是否在同一棵树上。

若不在则可以对集合进行合并。

 
<!--more-->
Prim算法：

集合A形成单棵树，添加集合A的安全边总是连接树与一个不在树中的顶点的最小权边。

使用最小优先队列。优先队列基于Key值，key[v]是所有将v与树中某一顶点相连的边中的最小权值。

一开始除了根节点，其他节点的key为无穷大。
