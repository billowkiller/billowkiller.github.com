---
layout: post
title: "grep example"
date: 2013-05-14 01:09
comments: true
categories: tools
tags: [grep,linux,command]
---

***from [http://blog.csdn.net/pan\_tian/article/details/7685815](http://blog.csdn.net/pan_tian/article/details/7685815)***

* * * * *

[grep](http://www.linuxso.com/command/grep.html) 语法
-----------------------------------------------------

*	grep 'word' filename

*	grep 'string1 string2' filename

*	cat otherfile | grep 'something'

*	command | grep 'something'

*	command option1 | grep 'data'

*	grep --color 'data' fileName


基本的用法
----------

在某个文件里搜索error字符串

`$ grep "error" log.txt`
<!--more-->
忽略大小写搜索(-i)
------------------

`$ grep -i "ErroR" log.txt`

所有子目录下的搜索(-r)
----------------------

`$ grep -r "exception" log.txt`

全字匹配搜索(-w)
----------------

如果你搜索boo，查询结果可能包含fooboo，boo123,
booooom,等等，可以使用-w来限定全字匹配

`$ grep -w "boo" /path/to/file`

全字匹配搜索两个不同单词
------------------------

`$ grep -w 'word1|word2' /path/to/file` 

统计字符串出现的次数(-c)
------------------------

`$ grep -c 'word' /path/to/file`

另外加-n的话， 会在结果中，列出匹配字符串的序列号，并且会列出内容

`$ grep -n 'word' /path/to/file` 

列出“不”包含字符串的行(-v)
--------------------------

`$ grep -v bar /path/to/file`

只列出文件名(-l)
----------------

`$ grep -l 'main' *.pls`

高亮显示(--color)
-----------------

`$ `grep --color oracle
/etc/[passwd](http://www.linuxso.com/command/passwd.html)

<img src="http://www.linuxso.com/uploads/allimg/120628/0043022131-1.jpg" alt="grep highlight"/>

UNIX / Linux pipes + grep 
--------------------------

`ls -l | grep -i xyz`

`ls 列出当前目录下的文件和文件夹,| 是管道传递给后面的一个程序,grep再是进行模式匹配`

`例如：ls *.pls | grep -i --color "MM"`


`========EOF=========`
