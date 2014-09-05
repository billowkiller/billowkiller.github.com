---
layout: post
title: "Classical Sync Problem"
date: 2014-08-23 02:18
comments: true
categories: rework
tags: [classical problem, multithread]
---

简单的记录下，后期再整理吧。

----------

## 生产者、消费者 ##

### 多个生产者，单个消费者 ###

<!--more-->

**1. 使用互斥锁 && 条件变量**

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc1_zps5e6026ee.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc2_zps96f163b4.png)

**2. 使用信号量**

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc3_zps90f678c0.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc4_zpsff637264.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc5_zpsc1ce838d.png)

### 多个生产者，多个消费者 ###

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc6_zps5fd624fc.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc7_zps77aab9e4.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc8_zps7ec189ae.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/pc9_zps88ec073a.png)

## 多个缓冲区读写 ##

最简单的双缓冲区示意图

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/db1_zps098b0d0f.png)

**多缓冲区生产者消费者代码**

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/db2_zps9fda343a.png)

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/db3_zps03dac974.png)