---
layout: post
title: "Signal and fork"
date: 2014-06-28 04:18
comments: true
categories: linux
tags: [linux, signal, process]
---

当线程调用fork时，就为子进程创建了整个进程地址空间的副本。子进程与父进程是完全不同的进程，只要两者都没有对内存作出改动，父进程和子进程之间还可以共享内存副本。注意一下几个情况：

1. 子进程通过继承整个地址空间的副本，**从父进程那里继承了所有互斥量、读写锁和条件变量的状态**。也就是说，如果它在父进程中被锁住，则它在子进程中也是被锁住的。
2. 只有调用fork()的线程被复制到子进程（子进程中线程的ID），如果子进程中包含占有锁的线程的副本，那么子进程就没有办法知道它占有了那些锁并且需要释放那些锁，**容易造成死锁**。
3. thread-specific data的销毁函数和清除函数都不会被调用。在多线程中调用fork()可能会引起内存泄露。比如在其他线程中创建的thread-specific data，在子进程中将没有指针来存取这些数据，**造成内存泄露**。

因为以上这些问题，**在线程中调用fork()的后，我们通常都会在子进程中调用exec()**。因为exec()能让父进程中的所有互斥量，条件变量（pthread objects）在子进程中统统消失（用新数据覆盖所有的内存）。对于那些要使用fork()但不使用exec()的程序，pthread API提供了一个新的函数

	pthread_atfor(void (*prepare_func)(void), void(*parent_func)(void), void (*child_func)(void))

prepare_func在父进程调用fork之前调用，parent_func在fork执行后在父进程内被调用，child_func在fork执行后子进程内被调用。除非你打算很快的exec一个新程序，否则应该避免在一个多线程的程序中使用fork。