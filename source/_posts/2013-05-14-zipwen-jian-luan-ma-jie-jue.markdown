---
layout: post
title: "zip文件乱码解决"
date: 2013-05-14 01:09
comments: true
categories: linux
tags: [zip,乱码]
---

在windows上压缩的文件，是以系统默认编码中文来压缩文件。由于zip文件中没有声明其编码，所以linux上的unzip一般以默认编码解压，中文文件名会出现乱码。

虽然2005年就有人把这报告为bug,
但是info-zip的官方网站没有把自动识别编码列入计划，可能他们不认为这是个问题。Sun对java中存在N年的zip编码问题，采用了同样的处理方式。

1.1 

通过unzip行命令解压，指定字符集

    unzip -O CP936 xxx.zip (用GBK, GB18030也可以)

有趣的是unzip的manual中并无这个选项的说明, unzip
--help对这个参数有一行简单的说明。

1.2 

在环境变量中，指定unzip参数，总是以指定的字符集显示和解压文件

解决办法：

引用

    vi /etc/environment

再最后加入后面的代码即可

UNZIP="-O CP936"

ZIPINFO="-O CP936"
