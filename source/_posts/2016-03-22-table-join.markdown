---
layout: post
title: "Table Join Semantics"
date: 2016-03-19 14:00
comments: true
category: "rework"
tags: [table, join]
---

这篇文章主要是记录下各种不同 `join` 的语义。

在最高层面上，`join` 分为三种：

* INNER
* OUTER
* CROSS

<!--more-->

***

1. `INNER JOIN` - fetches data if present in both the tables.

2. `OUTER JOIN` are of 3 types:

	* `LEFT OUTER JOIN` - fetches data if present in the left table.
	* `RIGHT OUTER JOIN` - fetches data if present in the right table.
	* `FULL OUTER JOIN` - fetches data if present in either of the two tables.

3. `CROSS JOIN`, as the name suggests, does [n X m] that joins everything to everything. Similar to scenario where we simply lists the tables for joining (in the FROM clause of the SELECT statement), using commas to separate them.

重点关注前两种，因为在语法中它们包含 `join` 这个关键词。语法如下：

```
<join_type> ::= 
    [ { INNER | { { LEFT | RIGHT | FULL } [ OUTER ] } } [ <join_hint> ] ]
    JOIN
```

`OUTER`是一个可选词，有没有对于语义并没有什么区别。解释下上面的语法，可以看到以下的用法是等价的：

```
A LEFT JOIN B            A LEFT OUTER JOIN B
A RIGHT JOIN B           A RIGHT OUTER JOIN B
A FULL JOIN B            A FULL OUTER JOIN B
A INNER JOIN B           A JOIN B
```

<img src="http://i.stack.imgur.com/qje6o.png" width="500px"/>

结果可以看到是这样的：

```
Table A | Table B     Table A | Table B      Table A | Table B      Table A | Table B
   1    |   5            1    |   1             1    |   1             1    |   1
   2    |   1            2    |   2             2    |   2             2    |   2
   3    |   6            3    |  null           3    |  null           -    |   -
   4    |   2            4    |  null           4    |  null           -    |   -
                        null  |   5             -    |   -            null  |   5
                        null  |   6             -    |   -            null  |   6

                      OUTER JOIN (FULL)     LEFT OUTER (partial)   RIGHT OUTER (partial)
```

