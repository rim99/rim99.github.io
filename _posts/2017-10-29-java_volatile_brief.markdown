---
layout: post
title: Java volatile关键字总结
date: 2017-10-29 19:53:02
categories: 笔记
---
 

> 《Java并发编程实战》一书在“线程安全”部分，指出了volatile关键字在“多线程下变量的安全发布”方面有很好的性能表现，但并没有具体解释。所以我搜集一些资料，并记录在这里。


# volatile的作用

当一个变量用volatile声明之后：

1. 这个变量的最新修改总可以保证被任何一个线程读取到，保证了变量修改的及时发布。因为，这个变量的读写是从内存操作的，而不是CPU缓存。

2. 在JVM执行读写这个变量的语句前后，代码不会被重排序。在执行读或写操作之前的所有代码都会执行完毕之后，才会执行读写操作，再然后才会执行后续的代码。当然了，之前的代码或者之后的代码各自有可能重排序，但是相互之间不会。


# JVM实现细节

不同构架的CPU会有不同的重排序特性。

常见的X86构架处理器，只“会对写-读操作进行重排序”，而“不会对读-读，读-写和写-写操作做重排序”。因此，在JVM编译的时候，会在volatile变量写操作之后，添加一条Lock前缀指令。

> Lock前缀实现了类似(内存屏障)的能力.
> 
> 1. 它先对总线/缓存加锁，然后执行后面的指令，最后释放锁后会把高速缓存中的脏数据全部刷新回主内存。
> 2. 在Lock锁住总线的时候，其他CPU的读写请求都会被阻塞，直到锁释放。Lock后的写操作会让其他CPU相关的cache line失效，从而从新从内存加载最新的数据。这个是通过缓存一致性协议做的。

Lock前缀指令，会引起处理器缓存回写到内存。同时根据MESI缓存一致性协议，处理器将该缓存标记为“脏”。一旦另一颗核心读取到相应的本地缓存时，发现为“脏”，就会去内存重新读取最新值。


# volatile的使用注意事项

可以看出来，volatile变量执行写操作的时候，因为内存屏障会产生更多的开销。

而且volatile只保证对变量的读或写操作的原子性。复合操作的原子性需要使用原子类、synchronized、锁结构等来实现。


# 参考资料

1. [聊聊并发（一）——深入分析Volatile的实现原理](http://www.infoq.com/cn/articles/ftf-java-volatile#anch83712)
1. [深入理解Java内存模型（四）——volatile
](http://www.infoq.com/cn/articles/java-memory-model-4)
2. [聊聊高并发（三十五）Java内存模型那些事（三）理解内存屏障](http://blog.csdn.net/iter_zc/article/details/42006811)
3. [Java并发编程：volatile关键字解析](http://www.cnblogs.com/dolphin0520/p/3920373.html)
4. [Java并发：volatile内存可见性和指令重排](http://blog.csdn.net/jiyiqinlovexx/article/details/50989328)

# 延伸阅读

1. [内存屏障与JVM并发](http://www.infoq.com/cn/articles/memory_barriers_jvm_concurrency)
2. [Linux内核的内存屏障](http://ifeve.com/linux-memory-barriers/#cache-coherency)
3. [正确使用 Volatile 变量](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)
