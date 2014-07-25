---
layout: post
title: "Linux epoll module"
date: 2014-07-15 02:18
comments: true
categories: linux
tags: [linux, epoll]
---

`select`，`poll`，`epoll`都是Linux下IO多路复用的机制。Windows下为`IOCP`模型。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个文件描述符就绪，能够通知程序进行相应的读写操作。其中文件描述符是一个简单的整数，用以标明每一个被进程所打开的文件和socket，包括`filefd`、`socketfd`、`signalfd`、`timerfd`、`eventfd`等。`eventfd` 是一个比 `pipe `更高效的线程间事件通知机制。

但`select`，`poll`，`epoll`本质上都是**同步I/O**，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而**异步I/O**则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

<!--more-->

## select实现

![](http://www.ibm.com/developerworks/cn/linux/l-async/figure4.gif)

**`select`的几大缺点：**

- <u>每次调用`select`，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大</u>
- <u>同时每次调用`select`都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大</u>
- <u>`select`支持的文件描述符数量太小了，默认是`1024`</u>

## poll实现

`poll`的实现和`select`非常相似，只是描述fd集合的方式不同，poll使用`pollfd`结构而不是select的`fd_set`结构，其他的都差不多。

## epoll

`epoll`是对`select`和`poll`的改进，能避免上述的三个缺点。我们先看一下`epoll`和`select`和`poll`的调用接口上的不同，`select`和`poll`都只提供了一个函数——`select`或者`poll`函数。而`epoll`提供了三个函数：

- `epoll_create`，创建一个epoll句柄；
- `epoll_ctl`，注册要监听的事件类型；
- `epoll_wait`，等待事件的产生。

对于第一个缺点，`epoll`的解决方案在`epoll_ctl`函数中。每次注册新的事件到`epoll`句柄中时（在`epoll_ctl`中指定`EPOLL_CTL_ADD`），会把所有的fd拷贝进内核，而不是在`epoll_wait`的时候重复拷贝。`epoll`保证了每个fd在整个过程中只会拷贝一次。

对于第二个缺点，`epoll`的解决方案不像`select`或`poll`一样每次都把current轮流加入fd对应的设备等待队列中，而只在`epoll_ctl`时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。`epoll_wait`的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用`schedule_timeout()`实现睡一会，判断一会的效果，和`select`实现中的是类似的）。

对于第三个缺点，`epoll`没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以`cat /proc/sys/fs/file-max`察看,一般来说这个数目和系统内存关系很大。

### 触发模型

`epoll`除了提供`select\poll`那种IO事件的**电平触发(Level Triggered)**外，还提供了**边沿触发(Edge Triggered)**，这就使得用户空间程序有可能缓存IO状态，减少`epoll_wait/epoll_pwait`的调用，提供应用程序的效率。

- **LT(level triggered)：**水平触发，缺省方式，同时支持block和no-block socket，在这种做法中，内核告诉我们一个文件描述符是否被就绪了，如果就绪了，你就可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错的可能性较小。传统的`select\poll`都是这种模型的代表。

- **ET(edge-triggered)：**边沿触发，高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪状态时，内核通过`epoll`告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如：你在发送、接受或者接受请求，或者发送接受的数据少于一定量时导致了一个EWOULDBLOCK错误)。但是请注意，如果一直不对这个fs做IO操作(从而导致它再次变成未就绪状态)，内核不会发送更多的通知。

**区别：**LT事件不会丢弃，而是只要读buffer里面有数据可以让用户读取，则不断的通知你。而ET则只在事件发生之时通知。


## 总结
**epoll高效的两个原因：**

- `epoll`会复用文件描述符集合来传递结果而不是迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，
- 另一个原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了