---
layout: post
title: "CSAPP学习笔记 - 并发编程"
date: 2016-05-01 19:53:02 +8000
categories: 笔记
---


>本文摘自CSAPP第12章，并加入了网上的参考资料。

# 并发与并行

拿线程来举例。

## 并发

当有多个线程在执行时，系统如果只有一个CPU，则根本不可能真正同时进行一个以上的线程。它只能把CPU运行时间划分成若干个时间段，再将时间段分配给各个线程执行。在一个时间段的线程代码运行时，其它线程处于挂起状。这种方式我们称之为**并发(Concurrent)**。

## 并行

当系统有一个以上CPU时，不同的CPU可以执行两个不同的线程。两个线程互不抢占CPU资源，可以同时进行，这种方式我们称之为**并行(Parallel)**。

>CSAPP的定义是——*如果两个及以上的逻辑控制流在时间上有重叠，那么这些流就是并发的*。

## 区别

并发和并行是即相似又有区别的两个概念。并行是指两个或者多个事件在同一时刻发生；而并发是指两个或多个事件在同一时间间隔内发生。在多道程序环境下，并发性是指在一段时间内宏观上有多个程序在同时运行；但在单处理机系统中，每一时刻却仅能有一道程序执行，故微观上这些程序只能是分时地交替执行。倘若在计算机系统中有多个处理机，则这些可以并发执行的程序便可被分配到多个处理机上，实现并行执行，即利用每个处理机来处理一个可并发执行的程序。这样，多个程序便可以同时执行。 

# 并发的实现

现代操作系统主要有三种基本的构造并发程序的方法。

## 进程

在多进程(Multi-processing)模式下，每一个逻辑控制流都是一个进程，由内核来调度和维护。每一个进程都有独立的虚拟地址空间。父进程和子进程共享文件表，但不共享用户地址空间。进程之间共享信息比较麻烦，需要显示的使用IPC机制。

在三种模式中，进程上下文较大，切换时的开销也更多，对性能的损耗较高。

多进程模式主要用`fork`、`exec`、`waitpid`等函数来构造，进程间通信可以用`pipe`、`mknod`、`mkfifo`等函数来实现。

可以参考：

* [Linux下的多进程编程初步](http://www.linuxdiyf.com/viewarticle.php?id=6195)
* [(整理)并发编程之一：多进程](http://konglingchun.is-programmer.com/posts/12222.html)
* [(转)并发网络编程学习之路（二）：多进程与进程池（续）](http://konglingchun.is-programmer.com/posts/12243.html)


## 线程

多线程(Multi-threading)模式运行在单一进程中，因此共享着进程的虚拟地址空间的整个内容。线程由内核自动调度。每个线程有自己的线程上下文，包括线程ID、栈、栈指针、程序计数器、通用目的寄存器和条件码。线程上下文比进程上下文要小很多，切换时的开销也小、也更快。

线程之间共享虚拟地址空间，但不共享寄存器。

Unix系统和C语言都支持Posix线程(Pthread)。可以参考：

* [POSIX线程](https://zh.wikipedia.org/wiki/POSIX%E7%BA%BF%E7%A8%8B)
* [POSIX Threads Programming](https://computing.llnl.gov/tutorials/pthreads/)
* [POSIX thread (pthread) libraries](http://www.yolinux.com/TUTORIALS/LinuxTutorialPosixThreads.html)
* [Multithreaded Programming (POSIX pthreads Tutorial)](http://randu.org/tutorials/threads/)

需要注意的是，线程有可结合与可分离的区别。可结合线程的资源需要显式的结束线程后才能回收，而可分离线程在线程停止运行后自动回收。

## I/O多路复用

通常的，每一次I/O操作，如果操作不能立刻返回结果，都会导致线程挂起，直到返回结果。当设定为**非阻塞I/O模式**后，如果操作不能立刻返回正确结果，就会返回一个I/O错误，从而保证线程不会被阻塞。对于非阻塞模式，必须使用轮询来获取结果。

而I/O多路复用(I/O multiplexing)模式维护着一个描述符集合，通过轮询获取一个就绪集合（也是描述符集合的子集），然后将就绪集合引入单一线程中执行后续操作。

I/O多路复用模式运行在单一线程中，大大减少了性能占用，没有切换开销，同时可以很方便的共享数据。但是，I/O多路复用模式的代码量较高，而且随着并发粒度的提高，代码量会持续增长。

I/O多路复用模式可以使用`select`、`epoll`等函数来构造。

>并发粒度——每个逻辑流在每个时间片内执行的指令数量。

可以参考：

* [IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇）](http://blog.csdn.net/historyasamirror/article/details/5778378)
* [聊聊IO多路复用之select、poll、epoll详解](http://blog.jobbole.com/99912/)
* [IO多路复用之select总结](http://www.cnblogs.com/Anker/archive/2013/08/14/3258674.html)
* [epoll精髓](http://www.cnblogs.com/OnlyXP/archive/2007/08/10/851222.html)
* [实例浅析epoll的水平触发和边缘触发，以及边缘触发为什么要使用非阻塞IO](http://www.cnblogs.com/yuuyuu/p/5103744.html)
* [Linux IO多路复用之epoll网络编程(含源码)](http://www.cnblogs.com/ggjucheng/archive/2012/01/17/2324974.html)
* [IO多路复用的几种实现机制的分析](http://blog.csdn.net/zhang_shuai_2011/article/details/7675797)
* [(摘)I/O多路复用详解（一）](http://konglingchun.is-programmer.com/posts/12146.html)
* [Reactor模式](http://blog.csdn.net/xiaocaidexuexibiji/article/details/11135803)
* [IO设计模式：Reactor和Proactor对比](https://segmentfault.com/a/1190000002715832)

I/O多路复用模式是三者当中性能最好的，但是有局限性——只能用在I/O密集型程序中。对于计算密集型程序或I/O性能足够强的系统，该模式并非最佳选择。










