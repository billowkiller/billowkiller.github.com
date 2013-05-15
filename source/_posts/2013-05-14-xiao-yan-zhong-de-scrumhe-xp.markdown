---
layout: post
title: "硝烟中的Scrum和XP"
date: 2013-05-14 01:09
comments: true
categories: rework
tags: [scrum,xp, software engineering]
---

**refine from** *硝烟中的Scrum和XP--我们如何实施Scrum*

* * * * *

产品的backlog时Scrum的核心，也是一切的起源，从根本上说，它就是一个需求或故事特性等组成的列表，按照重要性的级别进行排序。它里面包含的是客户想要的东西，并用客户的术语加一描述。

backlog的另外一个名称是故事。包括以下字段：

-   ID
-   Name：一个简短的描述
-   Importance：100以内打分，分数越高越重要
-   Initial estimate：最小单位为stroy
    point，即为人天。估值无需准确，但是要保证相对的正确性。
-   How to demo：简短的测试规范，先做啥，然后做啥，最后做啥，得到什么结果。
-   Notes：相关信息，解释说明，对其他资料的引用等等，简短。

额外的字段，根据需要：

-   Track：当前故事的大致分类（后台系统，优化...）
-   Components：再多个Scrum团队协作的时候很有用，包括数据库，服务器，客户端等组件
-   Requestor：哪个客户活相关人员最先提出的需求，再后续的开发过程中向他反馈
-   Bug tracking ID

产品的backlog应该停留再业务层次上，例如给Events表添加索引，潜在的目标是“提高再后台系统中搜索事件表单的相应速度”，这时需要改写，原先的目标作为一个注释存在。
<!--more-->
产品负责人维护backlog，理解每个故事的含义，不需要知道故事的具体实现，但是要知道为什么这个故事会在这里。其他人向负责人申请故事，负责人对它们划分先后次序。

sprint计划会议产生的成果：

-   sprint目标
-   团队成员名单（以及他们的投入程度）
-   sprint backlog
-   确定好sprint演示日期
-   确定每日Scrum会议的时间和地点

过程中实践TDD（测试驱动开发），包括开发和提问需求方式...

故事可以分成更小的故事，而小故事又可以分成任务。

一些重要的开发概念：

-   结对编程
-   测试驱动开发：Juit/httpnit/JWebUnit，HSQLDB，Jetty，Cobertura，mock
-   增量设计
-   代码集体所有权
-   持续集成：Maven，QuickBuild
-   代码标准

 
