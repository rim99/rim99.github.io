---
layout: post
title: Python并发开发简介
date: 2016-10-30 19:53:02
categories: 笔记
---

Python的并发程序可以使用multiprocessing库、threading库、asyncio库、concurrent.futures库以及selectors库等等协助编写：

 1. multiprocessing库可以创建多个进程，由系统协调调度各个任务；
 2. threading库则是创建多个线程，由Python解释器在一条进程内并发执行任务，始终只占用一个CPU核心资源，会遇到臭名昭著的GIL问题；  
 3. asyncio库则是将任务中的IO密集部分单拎出来，系统可以在等待IO密集部分返回的同时多次执行计算密集代码，所有任务都在一条线程内执行；
 4. concurrent.futures库可以用来创建线程池或进程池，更适合编写粒度较细的并发任务代码。
 5. selectors库是对select库的高级封装；而后者则是对Unix的select、poll、Linux的epoll、BSD的Kqueue等等IO复用方法的低级封装库。
 
multiprocessing库、threading库、asyncio库的有着类似的同步条件：Lock、RLock、Condition、Semaphore、Barrier、Event等：
 
 1. Lock、RLock较为类似，可以用来将一组操作原子化。RLock的特殊之处在于，可以被同一条线程递归调用，当然用完之后也得递归解锁。
 2. Condition可以保证多组线程阻塞在同一位置，等候其他线程的通知，防止因为资源为空时发生错误。
 3. Semaphore用于确保不会有过多线程同时访问某一资源。
 4. Barrier则用于保证所有线程各自在某一位置阻塞，当所有线程都阻塞时，又各自开始执行下一条指令。
 5. Event较为简单，一条线程可能阻塞等候event对象被另一条线程设置为true。

线程的好处是有共享的内存空间，方便线程间的交流。而进程则需要特殊的机制。multiprocessing库提供了queue和pipe两种机制用于进程间交流。这两类的不同之处在于：queue的内容可以被所有进程访问到，pipe只能被两个进程访问到，信息的安全程度较高。multiprocessing库还可以使用Manager对象为各个进程提供共享内存空间。此外，如果需要在进程间传递复杂对象，可以使用Proxy对象。

此外，Python为线程提供了queue.Queue和collections.deque满足线程的交流需求。后者的入队出队操作都是线程安全的。

而asyncio库有自己的queue类可以用于协程间的交流。

asyncio库的使用方法与前两者区别较大。当调用async def定义协程函数时，内部使用await等候另一个协程函数返回。使用asyncio协程库需要清晰的区分出IO密集操作和计算密集操作，也就是手动调度并发任务。多线程库则依靠解释器自动调度并发任务。asyncio库使用方法较为复杂，更多内容请看官方文档——[18.5. asyncio – Asynchronous I/O, event loop, coroutines and tasks¶](https://docs.python.org/3/library/asyncio.html)。

concurrent.futures库可以使用with语句来并发执行粒度较细的并发任务，也可以使用submit()方法来单独执行一个函数，返回一个future对象。future可以用于对对应任务的撤销、结果传递、回调函数设置、异常分析等等操作。

select类库只能处理IOBase的子类对象，主要需要实现fileno()类方法。






 
 
