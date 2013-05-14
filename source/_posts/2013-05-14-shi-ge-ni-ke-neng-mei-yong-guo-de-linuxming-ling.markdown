---
layout: post
title: "十个你可能没用过的Linux命令"
date: 2013-05-14 01:09
comments: true
categories: [linux, command]
---

***excerpted** from [图灵社区](http://www.ituring.com.cn/article/1782)*

* * * * *

 

如果你是一个硬件系统管理员或者Linux工程师，你可能会记得大多数Linux命令行技巧。下面的这些Linux命令行技巧通常不被Linux用户所使用。

**1.使用*pgrep*快速查找一个PID**

***pgrep***遍历目前正在运行的进程然后列出符合查找规则的进程ID（PID）。

~~~~ {.prettyprint}
pgrep ssh
~~~~

这条命令会列出所有与ssh有关的进程。

**2.执行上次执行过的命令**

这个标题有些绕口，但是它是名副其实的。

~~~~ {.prettyprint}
!!
~~~~
<!--more-->
这会执行你上一次在命令行中执行过的命令。

**3.执行最近一次以XX开头的命令**

如果你想要从命令行历史中执行一个s开头的命令时，你可以使用如下命令：

~~~~ {.prettyprint}
!s
~~~~

它会执行最近一次在命令行中执行且以字母s开头的命令。

**4.反复执行一个命令并在屏幕上输出**

***watch***会反复运行一个命令，并在屏幕上打印输出。它可以让你实时的观察程序的输出变化。默认情况下，程序每2秒运行一次。***watch***命令与***tail***命令很相似。

~~~~ {.prettyprint}
watch -d ls -l
~~~~

这条命令会监视当前目录的所有文件，并且高亮文件所发生的改变。

**5.在VI/VIM中快速保存**

如果你很匆忙，你可以通过***【SHIFT + zz】*** 快速从vi的插入模式中退出。

**6.快速登出终端** 你可以快速使用***【CTRL+D】***快速登出终端。

**7.返回你上一个所在目录**

你可以使用如下命令返回你上一次所在的目录：

~~~~ {.prettyprint}
cd -
~~~~

**8.聪明地创建父目录**

如下命令可以帮助你创建所有你需要的目录，即便是他们还不存在。为什么要浪费时间做一些愚蠢的事情比如：***mkdir
make ; cd make ; mkdir all ; cd all ; mkdir of ; cd of
…*** 你说到点子上了，使用***mkdir -p***！

~~~~ {.prettyprint}
mkdir -p /home/adam/make/all/of/these/directories/
~~~~

**9.删除一整行**

如果你已经输入一长串的命令，但是你又不在需要他们了，那么你可以使用如下命令直接删除一整行：

~~~~ {.prettyprint}
CTRL+U
~~~~

**10.设置文件的时间戳**

下面这条命令会把文件的时间戳设置成2008-01-01
8:00。日期格式是(YYMMDDhhmm)

~~~~ {.prettyprint}
touch -c -t 0801010800 filename.c
~~~~

你还能想到哪些为大多数人所指的Linux命令？

**【摘自回复】**

**访问上一个命令的最后一个参数** 如果你之前执行了这样一条命令 cp
assignment.htm /home/phill/reports/2008/
然后你可以冲 **\_\$** 访问刚才那条命令最后一个参数"*/home/phill/reports/2008/*"，例如：

~~~~ {.prettyprint}
cd $_
~~~~

**清除光标右边的内容** 上文有一个小错误，***【Ctrl +
U】***并不是删除一整行，而是删除光标左边的内容，如果光标停留在行首，那么***【Ctrl
+ U】***将无任何作用，这个时候，需要删除光标右边内容：

~~~~ {.prettyprint}
ctrl-k
~~~~

` `
