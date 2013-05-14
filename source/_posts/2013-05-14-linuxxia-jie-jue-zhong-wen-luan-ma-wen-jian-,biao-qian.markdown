---
layout: post
title: "linux下解决中文乱码——文件，标签"
date: 2013-05-14 01:09
comments: true
categories: [linux, 乱码]
---

文件是在WIndows 下创建的,Windows
的文件名中文编码默认为GBK,而Linux中默认文件名编码为UTF8,由于编码不一致所以导致了文件名乱码的问题，解决这个问题需要对文件名进行转码。

-   **文件与文件夹乱码，使用convmv来解决**

convmv 使用方法：

convmv -f 源编码 -t 新编码 [选项] 文件名

常用参数：\
-r 递归处理子文件夹\
–notest 真正进行操作，默认情况下是不对文件进行真实操作\
–list 显示所有支持的编码\
–unescap 可以做一下转义，比如把%20变成空格

举例：

在当前目录下

    convmv -f GBK -t UTF-8 --notest * //所有文件转gbk为utf8

-   **转换 mp3 标签编码，使用mutagen**

安装：

    sudo apt-get install python-mutagen

举例：<!--more-->

    mid3iconv -e gbk *.mp3 //当前目录下的mp3文件

    find . -iname “*.mp3” -execdir mid3iconv -e GBK {} \; //所有文件包括子文件夹

 
