---
layout: post
title: Python中实现协程并发查询数据库
date: 2016-12-11 19:53:02
categories: 原创
---
 

这周又填了一个以前挖下的坑。

这个博客系统使用Psycopg2库实现与PostgreSQL数据库的通信。前期，只是泛泛地了解了一下SQL语言，然后就胡乱拼凑出这么一个简易博客系统。

10月份找到工作以后，认真读了《数据库系统概念》这本书，对数据库有了更深的认识。然后就开始对博客系统的数据库查询模块开始重构。

# 改进之前

之前，我的查询步骤很简单，就是：

> 前端提交查询请求 --> 建立数据库连接 --> 新建游标 --> 执行命令 --> 接受结果 --> 关闭游标、连接

这几大步骤的顺序执行。

这里面当然问题很大：

1. 建立数据库连接实际上就是新建一个套接字。这是进程间通信的几种方法里，开销最大的了。
2. 在“执行命令”和“接受结果”两个步骤中，线程在阻塞在数据库内部的运行过程中，数据库连接和游标都处于闲置状态。

这样一来，每一次查询都要顺序的新建数据库连接，都要阻塞在数据库返回结果的过程中。当前端提交大量查询请求时，查询效率肯定是很低的。

# 第一次改进

之前的模块里，问题最大的就是第一步——建立数据库连接套接字了。如果能够一次性建立连接，之后查询能够反复服用这个连接就好了。

所以，首先应该把数据库查询模块作为一个单独的守护进程去执行，而前端app作为主进程响应用户的点击操作。那么两条进程怎么传递消息呢？翻了几天Python文档，终于构思出来：用队列queue作为生产者（web前端）向消费者（数据库后端）传递任务的渠道。生产者，会与SQL命令一起，同时传递一个管道pipe的连接对象，作为任务完成后，回传结果的渠道。确保，任务的接收方与发送方保持一致。

作为第二个问题的解决方法，可以使用线程池来并发获取任务队列中的task，然后执行命令并回传结果。

# 第二次改进

第一次改进的效果还是很明显的，不用任何测试手段。直接点击页面链接，可以很直观地感觉到反应速度有很明显的加快。

但是对于第二个问题，使用线程池还是有些欠妥当。因为，CPython解释器存在GIL问题，所有线程实际上都在一个解释器进程里调度。线程稍微开多一点，解释器进程就会频繁的切换线程，而线程切换的开销也不小。线程多一点，甚至会出现“抖动”问题（也就是刚刚唤醒一个线程，就进入挂起状态，刚刚换到栈帧或内存的上下文，又被换回内存或者磁盘），效率大大降低。也就是说，**线程池的并发量很有限**。

试过了多进程、多线程，只能在单个线程里做文章了。

## Python中的asyncio库

Python里有大量的协程库可以实现单线程内的并发操作，比如Twisted、Gevent等等。Python官方在3.5版本里提供了asyncio库同样可以实现协程并发。asyncio库大大降低了Python中协程的实现难度，就像定义普通函数那样就可以了，只是要在`def`前面多加一个`async`关键词。`async def`函数中，需要阻塞在其他`async def`函数的位置前面可以加上`await`关键词。


```Python
import asyncio

async def wait():
	await asyncio.sleep(2)
	
async def execute(task):
	process_task(task)
	await wait()
	continue_job()
```

`async def`函数的执行稍微麻烦点。需要首先获取一个`loop`对象，然后由这个对象代为执行`async def`函数。


```Python
loop = asyncio.get_event_loop()
loop.run_until_complete(execute(task))
loop.close()
```

`loop`在执行`execute(task)`函数时，如果遇到`await`关键字，就会暂时挂起当前协程，转而去执行其他阻塞在`await`关键词的协程，从而实现协程并发。

不过需要注意的是，`run_until_complete()`函数本身是一个阻塞函数。也就是说，当前线程会等候一个`run_until_complete()`函数执行完毕之后，才会继续执行下一部函数。所以下面这段代码并不能并发执行。


```Python
for task in task_list: 
	loop.run_until_complete(task)
```

对于这个问题，asyncio库也有相应的解决方案：`gather`函数。

```Python
loop = asyncio.get_event_loop()
tasks = [asyncio.ensure_future(execute(task))
			for task in task_list]
loop.run_until_complete(asyncio.gather(*tasks))
loop.close()
```

当然了，`async def`函数的执行并不只有这两种解决方案，还有`call_soon`与`run_forever`的配合执行等等，更多内容还请参考官方文档。

## Python下的I/O多路复用

协程，实际上，也存在上下文切换，只不过开销很轻微。而I/O多路复用则完全不存在这个问题。

目前，Linux上比较火的I/O多路复用API要算epoll了。Tornado，就是通过调用C语言封装的epoll库，成功解决了C10K问题（当然还有Pypy的功劳）。

在Linux里查文档，可以看到epoll只有三类函数，调用起来比较方便易懂。

1. 创建epoll对象，并返回其对应的文件描述符（file descriptor）。

    ```C
    int epoll_create(int size); 
    int epoll_create1(int flags);
    ```
   
2. 控制监听事件。第一个参数`epfd`就对应于前面命令创建的epoll对象的文件描述符；第二个参数表示该命令要执行的动作：监听事件的新增、修改或者删除；第三个参数，是要监听的文件对应的描述符；第四个，代表要监听的事件。

    ```C
    int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
    ```

3. 等候。这是一个阻塞函数，调用者会等候内核通知所注册的事件被触发。

    ```C
    int epoll_wait(int epfd, struct epoll_event *events,
          	           int maxevents, int timeout);
    int epoll_pwait(int epfd, struct epoll_event *events,
   		             int maxevents, int timeout,
   		             const sigset_t *sigmask);
    ```

在Python的select库里：

1. `select.epoll()`对应于第一类创建函数；
2. `epoll.register()`，`epoll.unregister()`，`epoll.modify()`均是对控制函数`epoll_ctl`的封装；
3. `epoll.poll()`则是对等候函数`epoll_wait`的封装。

Python里epoll相关API的最大问题应该是在`epoll.poll()`。相比于其所封装的`epoll_wait`，用户无法手动指定要等候的事件，也就是后者的第二个参数`struct epoll_event *events`。所以没法实现精确控制。

### 如果一定要精确控制

那只能使用替代方案：`select.select()`函数。

根据Python官方文档，`select.select(rlist, wlist, xlist[, timeout])`是对Unix系统中select函数的直接调用，与C语言API的传参很接近。前三个参数都是列表，其中的元素都是要注册到内核的文件描述符。如果想用自定义类，就要确保实现了`fileno()`方法。

这三个参数分别对应于：

1. `rlist`: 等候直到可读
2. `wlist`: 等候直到可写
3. `xlist`: 等候直到异常。这个`异常`的定义，要查看系统文档。

`select.select()`，类似于`epoll.poll()`，先注册文件和事件，然后保持等候内核通知，是阻塞函数。

select的缺点是：

1. 复杂度为O(n)。每次都要轮询所有注册的描述符，然后返回结果；
2. 每个进程只能支持注册1024个文件描述符，因为每一个select都在进程的内存中请求固定大小的空间，使用位图索引建立描述符集合数据结构，线程请求注册描述符的时候，select将描述符放入其中。

而epoll的ET边缘触发模式则是在文件描述符可用时方才所出响应，复杂度为O(1)。


## 实际应用

Psycopg2库支持对异步和协程，但和一般情况下的用法略有区别。普通数据库连接支持不同线程中的不同游标并发查询；而异步连接则不支持不同游标的同时查询。所以异步连接的不同游标之间必须使用I/O复用方法来协调调度。

所以，我的大致实现思路是这样的：首先并发执行大量协程，从任务队列中提取任务，再向连接池请求连接，创建游标，然后执行命令，并返回结果。在获取游标和接受查询结果之前，均要阻塞等候内核通知连接可用。

其中，连接池返回连接时，会根据引用连接的协程数量，返回负载最轻的连接。这也是自己定义AsyncConnectionPool类的目的。

我的代码位于：[bottle-blog/dbservice.py](https://github.com/rim99/bottle-blog/blob/master/dbservice.py)

## 存在问题

当然了，这个流程目前还一些问题。

首先就是每次轮询拿到任务之后，都会走这么一个流程。

```
获取连接 --> 新建游标 --> 执行任务 --> 关闭游标 --> 取消连接引用
```

本来，最好的情况应该是：在轮询之前，就建好游标；在轮询时，直接等候内核通知，执行相应任务。这样可以减少轮询时的任务量。但是如果协程提前对应好连接，那就不能保证在获取任务时，保持各连接负载均衡了。

所以这一块，还有工作要做。

然后就是，应该建立一个缓存机制。将一定时限的查询结果存储起来，可以直接返回，避免短时间内重复查询相同内容。

最后，请允许我吐槽一下Python的epoll相关文档：简直太弱了！！！必须看源码才能弄清楚功能。

# 参考文档

1. [18.5. asyncio — Asynchronous I/O, event loop, coroutines and tasks](https://docs.python.org/3.5/library/asyncio.html)
2. [epoll机制:epoll_create、epoll_ctl、epoll_wait、close](http://blog.csdn.net/yusiguyuan/article/details/15027821)
3. [epoll(7) - Linux man page](https://linux.die.net/man/7/epoll)
4. [cpython/Modules/selectmodule.c](https://github.com/rim99/cpython/blob/master/Modules/selectmodule.c)
5. [18.3. select — Waiting for I/O completion](https://docs.python.org/3.5/library/select.html)
6. [Psycopg 2.6.2 documentation >> Asynchronous support](http://initd.org/psycopg/docs/advanced.html#asynchronous-support)
7. [psycopg2/lib/pool.py](https://github.com/psycopg/psycopg2/blob/master/lib/pool.py)
