---
layout: post
title: asyncio异步编程学习笔记
date: 2016-10-12 19:53:02
categories: 笔记
---

asyncio模块是Python在3.4版里引入的。该模块通过协程(Coroutines)机制，使常规的单线程模式代码，能够实现对套接字等资源的多路复用、网络服务器和客户端的并发运行等等操作。

该模块包含了下列内容：

* event loop组类，模块的基础组件，用于封装实际执行的代码
* coroutines和tasks类，基于`yield from`语句([PEP 380](https://www.python.org/dev/peps/pep-0380))，有助于以单线程形式编写并发代码
* transport和protocol抽象层，类似于[Twisted模块](https://twistedmatrix.com/trac/)内的对应部分
* 对UDP，TCP，SSL，子进程管道，延迟调用(delay calls)等的支持
* Future类，模仿了[concurrent.futures](https://docs.python.org/3.5/library/concurrent.futures.html#module-concurrent.futures)模块，能够配合event loop组件使用
* 对coroutines和Future类的撤销(cancellation)支持
* 用于单线程多个协程之间同步的组件，类似于threading模块的相应组件
* 能够将任务存放至线程池的同步I/O调用接口

异步编程和同步编程有很大的不同。[ Develop with asyncio ](https://docs.python.org/3.5/library/asyncio-dev.html#asyncio-dev)解释了异步编程中常见的陷阱，并给出了对应方法。

# 1 Base Event Loop和Event loops

base event loop类是asyncio模块中的核心组件，能够：

* 注册、执行并取消延迟（或超时）的函数调用
* 为各类通信机制构建“服务器－客户”模型
* 创建子进程、并建立相应的与外部程序的通信机制
* 委托消费函数(costly function)从线程池中调用线程

base event loop类线程不安全。

event loops类将base event loop类和多路复用机制（因系统而异）封装在一起。

对应文档：[18.5.1. Base Event Loop
](https://docs.python.org/3.5/library/asyncio-eventloop.html)、[18.5.2. Event Loops](https://docs.python.org/3.5/library/asyncio-eventloops.html)

# 2 协程(Coroutines)

asyncio可以配合协程使用。协程可以用`async def`来写，也可以使用generator来写。如果你用Python 3.5，而且不考虑向前兼容，建议使用前者。

generator类协程最好用`@asyncio.coroutine`装饰。该装饰符和`async def`是兼容的。generator类协程需要用`yield from`语句的配合(见[PEP 380](https://www.python.org/dev/peps/pep-0380))。

"协程(coroutine)"，或者"generator"，实际上用来描述两类概念：

* 一种是定义协程的函数，可以称作*协程函数(coroutine function)*，调用`iscoroutinefunction()`返回`True`。
* 一种是调用协程函数得到的实例，可以称作*协程实例(coroutine object)*，调用`iscoroutine()`返回`True`。该实例代表了一个最终会完成的计算过程或者是I/O过程。

协程能够做到这些：

* `result = await future`或者`result = yield from future`，能够挂起当前协程直到`future`执行完毕或者抛出异常——如果`future`被撤销，就会抛出`CancelledError`异常。后文提到的`task`也是`future`的子类。
* `result = await coroutine`或者`result = yield from coroutine`，会等候`coroutine`执行完毕。这个`coroutine`必须是个协程函数。
* `return expression`，将结果传递给前两者那样的等候协程。
* `raise exception`，将异常抛给前两者那样的等候协程。

除了使用`await`或者`yield from`来调用协程外，只有ensure_future()函数或者BaseEventLoop.create_task()方法来启动协程函数。

协程只能在event loop运行时执行。

# 3 Task

Task是asyncio.Future的子类。

asyncio.Future类和concurrent.futures.Future类很像。它们的区别有：

* `result()`和`exception()`方法不接受timeout参数，而且如果实例没有正常执行结束，就会抛出错误
* 使用`add_done_callback()`方法注册的回调函数总要通过event loop类的`call_soon_threadsafe()`方法来调用
* `wait()`和`as_completed()`方法和concurrent.futures的方法不兼容

Task负责将协程封装起来，在event loop中调度任务。Task类线程不安全。

当正在执行的协程A调用其他协程B时，task就挂起A，执行B。等到B执行完毕后，将结果返回给A，并继续执行A。

对应的，event loop一次只执行一个协程。要想多个协程同时执行，那就得在多个线程中同时执行event loop。

调用`asyncio.Task.cancel()`会抛出`CancelledError`异常；如果协程实例没有抛出`CancelledError`异常，或者调用协程没有接到的话，调用`asyncio.Future.cancelled()`会返回`True`。

coroutine和task的文档：[18.5.3. Tasks and coroutines](https://docs.python.org/3.5/library/asyncio-task.html)

# 4 Transport

asyncio模块将底层的通信机制抽象为transport类。通常，无需手动创建transport实例。当调用BaseEventLoop方法时，程序会自动创建相应的通信实例（目前支持SSL、TCP、UDP、子进程管道等四种通信机制）。

transport类线程不安全。

# 5 Protocal

在asyncio模块中，protocal类负责解析传入数据，生成输出数据；而transport类负责对数据的实际传输，如I/O操作、缓存等。

使用protocal时，需要根据实际情况覆写某些方法。

transport类、protocal类的文档：
[18.5.4. Transports and protocols](https://docs.python.org/3.5/library/asyncio-protocol.html)

# 6 Streams

streams类文档：[18.5.5. Streams](https://docs.python.org/3.5/library/asyncio-stream.html)

# 7 Subprocess

asyncio模块支持不同的线程中执行子进程(subprocess)，但是有些限制： 

* event loop必须运行在主线程中
* 在执行子进程之前，必须先在主进程中调用`get_child_watcher()`函数建立好观察员(child watcher)实例。

asyncio.subprocess.Process类是线程不安全的，其文档为[18.5.6. Subprocess](https://docs.python.org/3.5/library/asyncio-subprocess.html)。

# 8 同步条件

asyncio模块提供了类似于threading模块的同步机制，如同步锁、信号量等等。

对应文档：[18.5.7. Synchronization primitives](https://docs.python.org/3.5/library/asyncio-sync.html)

# 9 Queue

asyncio.Queue有三个子类：Queue，PriorityQueue，LifoQueue，均类似于queue模块的对应类。但不接受timeout参数。

`asyncio.wait_for()`方法可以撤销超时的任务。

asyncio.Queue类用来协调生产者协程和消费者协程。他们是线程不安全的。

对应文档：[18.5.8. Queues](https://docs.python.org/3.5/library/asyncio-queue.html)

(Python之asyncio)[https://vvl.me/2016/03/python-coroutines/]
