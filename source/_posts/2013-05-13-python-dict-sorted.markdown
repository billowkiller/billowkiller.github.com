---
layout: post
title: "python dict sorted"
date: 2013-05-13 02:18
comments: true
categories: language
tags: [python, dict, 排序, sort]
---

<i>**from** [http://www.cnblogs.com/linyawen/archive/2012/03/15/2398292.html](http://www.cnblogs.com/linyawen/archive/2012/03/15/2398292.html)</i>

* * * * *

我们知道Python的内置dictionary数据类型是无序的，通过key来获取对应的value。可是有时我们需要对dictionary中
的item进行排序输出，可能根据key，也可能根据value来排。到底有多少种方法可以实现对dictionary的内容进行排序输出呢？下面摘取了
一些精彩的解决办法。 

<!--more-->

\#最简单的方法，这个是按照key值排序： 

{% codeblock lang:python %}
def sortedDictValues1(adict):
    items = adict.items()
    items.sort()
    return [value for key, value in items] 
{% endcodeblock %}



\#又一个按照key值排序，貌似比上一个速度要快点 

{% codeblock lang:python %}
def sortedDictValues2(adict):
    keys = adict.keys()
    keys.sort()
    return [dict[key] for key in keys] 
{% endcodeblock %}


\#还是按key值排序，据说更快。。。而且当key为tuple的时候照样适用 

{% codeblock lang:python %}
def sortedDictValues3(adict): 
    keys = adict.keys() 
    keys.sort() 
    return map(adict.get, keys) 
{% endcodeblock %}


\#一行语句搞定： 

{% codeblock lang:python %}
[(k,di[k]) for k in sorted(di.keys())] 
{% endcodeblock %}

\#来一个根据value排序的，先把item的key和value交换位置放入一个list中，再根据list每个元素的第一个值，即原来的value值，排序： 

{% codeblock lang:python %}
def sort\_by\_value(d): 
    items=d.items() 
    backitems=[[v[1],v[0]] for v in items] 
    backitems.sort() 
    return [ backitems[i][1] for i in range(0,len(backitems))] 
{% endcodeblock %}

\#还是一行搞定： 

{% codeblock lang:python %}
[ v for v in sorted(di.values())] 
{% endcodeblock %}

\#用lambda表达式来排序，更灵活： 

{% codeblock lang:python %}
sorted(d.items(), lambda x, y: cmp(x[1], y[1]))
{% endcodeblock %}

, 或反序： 
{% codeblock lang:python %}
sorted(d.items(), lambda x, y: cmp(x[1], y[1]), reverse=True) 
{% endcodeblock %}


\#用sorted函数的key= 参数排序： 
\# 按照key进行排序 

{% codeblock lang:python %}
print sorted(dict1.items(), key=lambda d: d[0]) 
{% endcodeblock %}

\# 按照value进行排序 

{% codeblock lang:python %}
print sorted(dict1.items(), key=lambda d: d[1]) 
{% endcodeblock %}


下面给出python内置sorted函数的帮助文档： 

sorted(...) 

sorted(iterable, cmp=None, key=None, reverse=False) --\> new sorted
list 


看了上面这么多种对dictionary排序的方法，其实它们的核心思想都一样，即把dictionary中的元素分离出来放到一个list中，对list排序，从而间接实现对dictionary的排序。这个“元素”可以是key，value或者item。 

\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#

一上转 

按照value排序可以用 

{% codeblock lang:python %}
sorted(d.items, key=lambda d:d[1]) 
{% endcodeblock %}

若版本低不支持sorted 

将key,value 以tuple一起放在一个list中 

l = [] 

l.append((akey,avalue))... 

用sort（） 

l.sort(lambda a,b :cmp(a[1],b[1]))(cmp前加“-”表示降序排序)

 

 

 

 
