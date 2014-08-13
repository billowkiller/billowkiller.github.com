---
layout: post
title: "Unix Network Programming Notes"
date: 2014-07-22 09:18
comments: true
categories: book
tags: [linux， network]
---

记录Linux网络编程中的一些知识点...

----------


### 网络编程可能会遇到的三种情况 ###

1. 当`fork`子进程时，必须捕获`SIGCHLD`信号。
1. 当捕获信号时，必须处理被中断的系统调用；
1. `SIGCHLD`的信号处理函数必须正确编写，应使用`waitpid`函数以免留下僵死进程。

	`waitpid`是可以非阻塞的等待信号终止，因此可以使用循环调用。

<!--more-->

### 网络socket ###

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/socket_zpsdcdfab1b.png)

**对应的TCP分组**

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/TCP_zpsb7790efe.png)

### shutdown函数 ###

终止网络连接的通常方法是调用`close`函数。不过`close`有两个限制，可以使用`shutdown`来避免。

1. `close`把描述符的引用计数减1，仅在该计数变为0时才关闭套接字。使用`shutdown`可以不管引用计数就激发TCP的正常连接终止序列。
1. `close`终止读和写两个方向的数据传送。

说明：

- `shutdown`根本没有关闭socket,任何与socket关联的资源直到调用closesocket才释放。
- TCP连接的socket是全双工的，也就是说它可以发送和接收数据，但是一个方向上的数据流动和另一个方向上的数据流动是不相关的，shutdown函数的功能也就是体现在这里，它通过设置how选择关闭哪条数据通道(发送通道和接收通道)，如果关闭了发送通道，那么这个socket其实还可以通过接收通道接受数据.
- 当通过以how=1(`SHUT_WR`)的方式调用`shutdown`，就可以保证对方收到一个EOF，而不管其他的进程是否已经打开了套接字，而调用`close`或closesocket就不能保证，因为直到套接字的引用计数减为0时才会发送FIN消息给对方，也就是说，直到所有的进程都关闭了套接字。

**为了保证在连接撤销之前，保证双方都能接收到对等方的所有数据，在调用closesocket之前，先调用shutdown关闭发送数据通道。**


### tcp中close_wait状态出现的原因

`close_wait`出现的原因: 就是某一方在网络连接断开后，对等方没有检测到这个错误（对方断开）而没有调   用 closesocket，导致了这个状态的出现.

模拟这样一个环境:服务器192.168.1.112:4500在接收到一个客户端的连接后，休眠五秒后，服务器关闭与客户 端通讯的socket后正常退出，而客户端在连接服务器后，等待用户输入字符后，发送给客户端。现在有这样几个问题:

1.    服务器在休眠五秒后，正常退出了，但是由于客户端还在等待用户输入，此时服务器端TCP的状态是什么？(`FIN_WAIT_2`)，客户端的TCP状态是什么?(`CLOSE_WAIT`)

2.       服务器在休眠五秒后，正常退出了，在服务器退出后，如果客户端异常退出，那么服务器端TCP的状态是什么？客户端的TCP状态是什么?

       在服务器正常退出后，客户端异常退出，那么客户端就会向服务器发送RST标志，然后客户端和服务器端的TCP状态都是`CLOSED`

3.       服务器在休眠五秒后，正常退出了，在服务器退出后,从客户端输入数据后，向服务器发送，此时服务器怎样处理这个数据?

       客户端通过PSH标志向服务器段发送数据，能够发送成功，但因为服务器的TCP处于(`FIN_WAIT_2`)状态，此时服务器会向客户端发送一个RST标示，并且服务器端口状态和客户端的TCP状态都变为`CLOSED`。

4.       在服务器休眠的过程中，杀死服务器进程，服务器端TCP状态是什么?客户端的TCP状态是什么?

	在服务器休眠的过程中，杀死服务器进程，此时服务器方会向客户端发送一个RST标志，服务器TCP状态是`CLOSED`，客户端的TCP状态也是`CLOSE`.
	在服务器休眠五秒后，如果不关闭与客户端通讯的Socket直接正常退出，此时，服务器方也向客户端发送了RST标志。
 
对于上面的四个问题，必须注意到服务器正常断开的时候，向客户端发送的FIN根本不能被客户端的所正常处理，因为客户端正处于接收用户的输入。所以由于每次都是服务器主动断开，但是服务器TCP状态却有可能不能进入到`Time_Wait`状态。

### 服务器终止可能出现的情况 ###

1. `accept、read、write、select、open`等慢系统调用中断，系统调用可能返回`EINTR`错误。需要重启被中断的系统调用。并且编写捕获信号的程序时，必须对慢系统调用返回`EINTR`有所准备。
2. 三路握手完成后，客户TCP发送一个RST。服务器进程在调用`accept`的时候RST到达。accept返回一个错误给服务器进程，POSIX指出返回的errno值必须是`ECONNABORTED`（“software caused connection abort.”）。服务器忽略它，再次调用accept。
3. 服务器进程终止。服务端调用`kill`命令杀死服务器子进程。子进程中所有打开的描述符都被关闭，进而会向客户发送一个FIN，客户TCP则响应一个ACK。而客户进程此时阻塞在`fgets`调用上。由于FIN的接收并没有告知客户服务器进程已经终止，所以客户进程照常发送数据。此时服务端响应RST。客户进程看不到这个RST，因为它在调用`writen`后立即调用`readline`，并且由于接收到FIN，**所调用的readline立即返回0（EOF）**。于是以出错信息（“server terminated prematurely”）退出。
4. 如果客户不理会`readline`函数返回的错误，写更多的数据到服务器上。当一个进程向某个已收到RST的套接字执行写操作时，内核向该进程发送一个`SIGPIPE`信号。该信号的默认行为是终止进程，因此进程必须捕获它以免不情愿地被终止。
5. 服务器主机崩溃。客户端到服务端之间网络断掉，或者服务端断电等，物理连接断掉了，这种情况下客户端不会退出（此情况称为**半开连接**），`send`函数正常执行，不会感觉到自己出错。因为由于物理网络断开，服务端不会给客户端回应错误消息。此时，客户TCP持续重传数据分节，试图从服务器上接收一个ACK。最终返回的错误是`ETIMEDOUT`。然而如果某个中间路由器判定服务器主机已不可达，从而响应一个“destination unreachable”ICMP消息，那么返回的错误是`EHOSTUNREACH`或`ENETUNREACH`。
6. 服务器主机崩溃后重启。服务进程对客户端`send`来的消息会产生RST响应。客户收到RST时，客户正阻塞于`read`调用，导致该调用返回`ECONNESET`错误。
7. 服务器主机关机。unix系统关机时，`init`进程通常先给所有进程发送`SIGTERM`信号（可捕获），等待固定一段时间后，给所有仍在运行的进程发送`SIGKILL`信号。接下里就和3一样。

下图是检测各种TCP条件的方法

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/2014-08-01183756__zpse1970a67.jpg)

### SO_LINGER套接字选项

在默认情况下，当调用`close`关闭socket的使用，`close`会立即返回；但是，如果send buffer中还有数据，系统会试着先把send buffer中的数据发送出去，然后close才返回。

`SO_LINGER`选项则是用来修改这种默认操作的。于SO_LINGER相关联的一个结构体如下:

	#include <sys/socket.h>
	struct linger {
	      int l_onoff  //0=off, nonzero=on(开关)
	      int l_linger //linger time(延迟时间)
	}

当调用`setsockopt`之后,该选项产生的影响取决于`linger`结构体中`l_onoff`和`l_linger`的值:

- 当`l_onoff`被设置为0的时候,将会关闭`SO_LINGER`选项,即TCP或则SCTP保持默认操作:`close`立即返回、`l_linger`值被忽略.
- `l_lineoff`值非0，`l_linger`为0，那么当`close`某个连接时TCP将终止该连接。send buffer中未被发送的数据将被丢弃,并向对方发送一个RST信息.值得注意的是，由于这种方式，是非正常的4中握手方式结束TCP链接，所以，TCP连接将不会进入`TIME_WAIT`状态，这样会导致新建立的可能和就连接的数据造成混乱。

设置`SO_LINGER`套接字选项后，`close`的成功返回只是告诉我们先前发送的数据（和FIN）已由对端TCP确认，而不能告诉我们对端应用进程是否已读取数据。如果不设置该套接字选项，那么我们连对断TCP是否确认了数据都不知道。
让客户知道服务器已读取其数据的一个方法是改为调用`shutdown`，并设置它的第二个参数为`SHUT_WR`，而不是调用`close`。

![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/1_zps7ad163b0.png)

关闭连接的本地端（客户端）时，根据所调用的函数（`close`和`shutdown`）以及是否设置了`SO_LINGER`套接字选项，**可在以下3个不同的时机返回**。

1. `close`立即返回，根本不等待（默认情况）。
1. `close`一直拖延到接受了对于客户端FIN的ACK才返回。
1. 后跟一个`read`调用的`shutdown`一直等到接受对端FIN才返回。

**下图汇总了对shutdown的两种可能调用和对close的三种可能调用，以及它们对TCP套接字的影响。**
![](http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/_zps2eb27157.png)

### SO_REUSEADDR套接字选项

SO_REUSEADDR套接字选项能起到以下功能。

1. SO_REUSEADDR允许一个server程序listen监听并bind到一个端口,既是这个端口已经被一个正在运行的连接使用了.

	例如，以下情况：

	1. 一个监听(listen)server已经启动
	1. 	当有client有连接请求的时候,server产生一个子进程去处理该client的事物.
	1. 	server主进程终止了,但是子进程还在占用该连接处理client的事情.虽然子进程终止了,但是由于子进程没有终止,该socket的引用计数不会为0，所以该socket不会被关闭.
	1. 	server程序重启.
	
	**所有的TCP server都必须设定此选项,用以应对server重启的现象.**

2. SO_REUSEADDR允许多个server绑定到同一个port上,只要这些server指定的IP不同

	SO_REUSEADDR需要在bind调用之前就设定。另外，还可以在绑定IP通配符。但是最好是先绑定确定的IP，最后绑定通配符IP。运行在这些端口上的服务器实例可以相同，也可以不同。在TCP中，不允许建立起一个已经存在的相同的IP和端口的连接。但是在UDP中，是允许的，特别是在多播中。