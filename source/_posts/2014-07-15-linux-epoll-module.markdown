---
layout: post
title: "Linux epoll module"
date: 2014-07-15 02:18
comments: true
categories: linux
tags: [linux, epoll]
---

综合了几个blog以及自己查到的一些资料，总结下Linux中的IO多路复用，主要是`epoll`模型。

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

`poll`的实现和`select`非常相似，只是描述fd集合的方式不同，poll使用`pollfd`结构而不是select的`fd_set`结构，其他的都差不多。poll用**nfds**参数指定最多监听多少个文件描述符和事件，能达到系统允许打开的最大文件描述符数目，这点和`epoll`一样。

## epoll

`epoll`是对`select`和`poll`的改进，能避免上述的三个缺点。我们先看一下`epoll`和`select`和`poll`的调用接口上的不同，`select`和`poll`都只提供了一个函数——`select`或者`poll`函数。而`epoll`提供了三个函数：

- `epoll_create`，创建一个epoll句柄；
- `epoll_ctl`，注册要监听的事件类型；
- `epoll_wait`，等待事件的产生。

对于第一个缺点，`epoll`的解决方案在`epoll_ctl`函数中。每次注册新的事件到`epoll`句柄中时（在`epoll_ctl`中指定`EPOLL_CTL_ADD`），会把所有的fd拷贝进内核，而不是在`epoll_wait`的时候重复拷贝。`epoll`保证了每个fd在整个过程中只会拷贝一次。

对于第二个缺点，`epoll`的解决方案不像`select`或`poll`一样每次都把current轮流加入fd对应的设备等待队列中，而只在`epoll_ctl`时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。`epoll_wait`的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用`schedule_timeout()`实现睡一会，判断一会的效果，和`select`实现中的是类似的）。

对于第三个缺点，`epoll`没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以`cat /proc/sys/fs/file-max`察看,一般来说这个数目和系统内存关系很大。

### 触发模型

#### 1. EAGAIN

先说下`EAGIN`这个返回值。

在一个非阻塞的socket上调用read/write函数, 返回`EAGAIN`或者`EWOULDBLOCK`(注: EAGAIN就是EWOULDBLOCK)。从字面上看, 意思是:EAGAIN: 再试一次，EWOULDBLOCK: 如果这是一个阻塞socket, 操作将被block，perror输出: Resource temporarily unavailable

**总结:**

这个错误表示资源暂时不够，能read时，读缓冲区没有数据，或者write时，写缓冲区满了。遇到这种情况，如果是阻塞socket，read/write就要阻塞掉。而如果是非阻塞socket，read/write立即返回-1， 同时`errno`设置为`EAGAIN`。

所以，**对于阻塞socket，read/write返回-1代表网络出错了。但对于非阻塞socket，read/write返回-1不一定网络真的出错了。可能是Resource temporarily unavailable。这时你应该再试，直到Resource available。**

#### 2. LT & ET

`epoll`除了提供`select\poll`那种IO事件的**电平触发(Level Triggered)**外，还提供了**边沿触发(Edge Triggered)**，这就使得用户空间程序有可能缓存IO状态，减少`epoll_wait/epoll_pwait`的调用，提供应用程序的效率。

- **LT(level triggered)：**水平触发，缺省方式，同时支持block和no-block socket，在这种做法中，内核告诉我们一个文件描述符是否被就绪了，如果就绪了，你就可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错的可能性较小。传统的`select\poll`都是这种模型的代表。

- **ET(edge-triggered)：**边沿触发，高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪状态时，内核通过`epoll`告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如：你在发送、接受或者接受请求，或者发送接受的数据少于一定量时导致了一个EWOULDBLOCK错误)。但是请注意，如果一直不对这个fs做IO操作(从而导致它再次变成未就绪状态)，内核不会发送更多的通知。

**区别：**LT事件不会丢弃，而是只要读buffer里面有数据可以让用户读取，则不断的通知你。而ET则只在事件发生之时通知。

**ET方式注意事项：** 必须使用**非阻塞套接口**，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。最好以下面的方式调用ET模式的epoll接口：

- 基于非阻塞文件句柄
- 只有当read(2)或者write(2)返回**EAGAIN**时才需要挂起，等待。但这并不是说每次read()时都需要循环读，直到读到产生一个EAGAIN才认为此次事件处理完成，**当read()返回的读到的数据长度小于请求的数据长度时，就可以确定此时缓冲中已没有数据了**，也就可以认为此事读事件已处理完成。

		while(rs)
		{
		  buflen = recv(activeevents[i].data.fd, buf, sizeof(buf), 0);
		  if(buflen < 0)
		  {
		    // 由于是非阻塞的模式,所以当errno为EAGAIN时,表示当前缓冲区已无数据可读
		    // 在这里就当作是该次事件已处理处.
		    if(errno == EAGAIN)
		     break;
		    else
		     return;
		   }
		   else if(buflen == 0)
		   {
		     // 这里表示对端的socket已正常关闭.
		   }
		   if(buflen == sizeof(buf)
		     rs = 1;   // 需要再次读取
		   else
		     rs = 0;
		}

**还有，假如发送端流量大于接收端的流量(意思是epoll所在的程序读比转发的socket要快),由于是非阻塞的socket,那么`send()`函数虽然返回,但实际缓冲区的数据并未真正发给接收端,这样不断的读和发，当缓冲区满后会产生`EAGAIN`错误(参考`man send`),同时,不理会这次请求发送的数据.所以,需要封装`socket_send()`的函数用来处理这种情况,该函数会尽量将数据写完再返回，返回-1表示出错。在`socket_send()`内部,当写缓冲已满(`send()`返回-1,且`errno`为`EAGAIN`),那么会等待后再重试.这种方式并不很完美,在理论上可能会长时间的阻塞在`socket_send()`内部,但暂没有更好的办法.**

#### 3. 正确的accept

正确的accept，accept 要考虑 3 个问题

1. **阻塞模式 accept 存在的问题**

	考虑这种情况：TCP连接被客户端夭折，即在服务器调用accept之前，客户端主动发送RST终止连接，导致刚刚建立的连接从就绪队列中移出，如果套接口被设置成阻塞模式，服务器就会一直阻塞在accept调用上，直到其他某个客户建立一个新的连接为止。但是在此期间，服务器单纯地阻塞在accept调用上，就绪队列中的其他描述符都得不到处理。

	**解决办法是把监听套接口设置为非阻塞**，当客户在服务器调用accept之前中止某个连接时，accept调用可以立即返回-1，这时源自Berkeley的实现会在内核中处理该事件，并不会将该事件通知给epool，而其他实现把errno设置为ECONNABORTED或者EPROTO错误，我们应该忽略这两个错误。

2. **慢系统调用被中断**
	
	当阻塞与某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回是，该系统调用可能返回一个`IENTR`错误。我们必须对慢系统调用返回`EINTR`有所准备。对于accept，以及诸如`read`、`write`、`select`和`open`之类函数来说，**需要自己重启被中断的系统调用**。不过有一个函数我们不能重启：`connect`。

2. **ET模式下accept存在的问题**

	考虑这种情况：**多个连接同时到达**，服务器的TCP就绪队列瞬间积累多个就绪连接，由于是边缘触发模式，epoll只会通知一次，accept只处理一个连接，导致TCP就绪队列中剩下的连接都得不到处理。

	**解决办法是用while循环抱住accept调用**，处理完TCP就绪队列中的所有连接后再退出循环。如何知道是否处理完就绪队列中的所有连接呢？accept返回-1并且errno设置为EAGAIN就表示所有连接都处理完。

综合以上两种情况，服务器应该使用非阻塞地accept，accept在ET模式下的正确使用方式为：


	while ((conn_sock = accept(listenfd,(struct sockaddr *) &remote, (size_t *)&addrlen)) > 0) {
	    handle_client(conn_sock);
	}
	if (conn_sock == -1) {
	    if (errno != EAGAIN && errno != ECONNABORTED && errno != EPROTO && errno != EINTR)
	    perror("accept");
	}

#### 4. 其他

**EPOLLONETSHOT**

epoll模式中事件可能被触发多次，比如socket接收到数据交给一个线程处理数据，在数据没有处理完之前又有新数据达到触发了事件，另一个线程被激活获得该socket，从而产生多个线程操作同一socket，即使在ET模式下也有可能出现这种情况。采用EPOLLONETSHOT事件的文件描述符上的注册事件只触发一次，要想重新注册事件则需要调用epoll_ctl重置文件描述符上的事件，这样前面的socket就不会出现竞态。

如果一个工作线程处理完某个socket上的一次请求之后，又收到该socket上新的客户请求，则该线程将继续为这个socket服务，并且因为该socket上注册了EPOLLONESHORT事件，其他线程没有机会接触这个soket。如果工作线程之后没有收到该socket的下一批用户数据，则放弃该socket服务。同时重置该socket上的注册事件，使得epoll有机会再次检查到该socket上的EPOLLIN事件，进而是的其他线程有机会为该socket服务。

**注意：监听socket linstenfd上是不能注册EPOLLONESHOT事件，否则应用程序只能处理一个客户连接。后续的客户请求不再触发listenfd上的EPOLLIN事件。**

**EPOLLONESHOT和ET一样都是为了能进一步减少可读、可写和异常事件被触发的次数。**

## 总结
**epoll高效的原因：**

- `epoll`把用户关心的文件描述符上的时间放在内核的一个事件表中，从而无需向`select`和`poll`那样每次调用都要重复传入描述符集合。
- 另一个原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了
- 使用`mmap`加速内核与用户空间的消息传递