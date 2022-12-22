---
layout: post
title: 聊聊Python的Web服务器框架（一）
date: 2017-02-19 19:53:02
categories: 原创
---

HTTP/1.1协议是一个基于文本的传输协议。传输报文都是直接以文本的形式传递消息。所以本质上讲，HTTP服务器就是负责解析文本，处理请求，然后组织文本并回传客户端。

Web开发刚刚兴起的时候，HTTP服务器开发这块各家都有自己的实现，有自己的特点。有些报文解析速度快，有一些处理请求速度快，有一些组织回传结果的速度快。为了方便代码复用，实现这些不同特点的服务器模块的按需组织，一些语言就自行定义了一些协议，对Web服务器的开发做出一些建议性的规定。比如，Java的servlet协议。

Python则自定义了WSGI协议。

# WSGI协议

Python官方在PEP-0333和PEP-3333中详细定义了WSGI协议的细节。前者主要适用于Python2.x，后者根据Python3.x的语言细节变化对WSGI协议做了一些调整。

WSGI协议将整个Web服务器程序划分为应用框架端（后文简称为应用端）、服务器网关端（后文简称为服务器端）以及中间件。中间件，在应用端来看就是服务器端，在服务器端来看就是应用端。所以总体来讲，WSGI就只规定了应用端和服务器端之间的交互协议。

在WSGI协议中，服务器端负责监听套接字端口，接收报文，解析报文，向应用端发送解析结果，并接收应用端的响应结果，组织响应报文，向客户端发送响应报文。而应用端，接收到服务器端的解析结果后，根据HTTP方法、URI以及其他报文，执行代码，调用模版生成动态网页，也许会向数据库后端发送查询请求。

WSGI协议的交互核心就是一个`app(environ, start_response)`，是一个回调机制的实现。`app()`方法由应用端实现，`environ`参数是标准的Python built-in字典，其中包含了WSGI环境变量以及服务器对HTTP请求报文的解析结果；`start_response`是一个类似于函数的可调用对象（任何实现了`__call__`方法的实例都是可行的），由服务器端实现，负责组织响应报文。`start_response`接收两个参数`status`, `response_headers`。前者是应用端响应结果的状态，例如`"100 Continue"`、`"200 OK"`、`"404 Not Found"`；后者这是响应报文的报文头，是一个数组，其中的每一个元素又是一个元组，例如`[("Content-type", "text/plain")]`。

WSGI协议规定的是一个同步Web框架，因为要照顾某些无法异步实现的流程。

# 服务器端框架的底层机制实现

服务器端框架直接处理网络IO，对整个Web服务器的性能影响非常大。常见的服务器端框架有多线程、异步两种基础机制。

## bjoern的异步机制

bjoern是一个使用C语言实现的内存占用少而且拥有极强性能的Python服务器端框架。bjoern使用了事件驱动框架libev、高性能HTTP解析框架http-parser。bjoern使用了Python2的C语言API，因此只能用于CPython解释器，且不支持Python3。原作者暂时也没有支持Python3的打算，[issue: Python 3 support (PEP 3333) #29](https://github.com/jonashaag/bjoern/issues/29)。此外bjoern不支持SSL。

bjoern的异步机制是这样的。首先注册一个ev\_io事件保持对主端口（80或者443）的监听状态。一旦套接字可读，表示有新的连接请求，这时候立即调用回调函数，执行accept动作，并注册另一个ev\_io事件保持对客户端连接的监听，并持续接收报文（io可读），调用WSGI协议，持续回传响应（io可写）等等。接收、回传两部均实现事件驱动机制。

同一个ev\_loop，既保持对主端口的监听，同时保持对所有客户端口的报文收发，对其中的所有网络IO事件协调调度。

## cheroot的多线程机制

CherryPy是一个纯Python实现的生产级高稳定高性能框架，包含了应用端和服务器端。而cheroot则是CherryPy框架解耦之后分离出来的服务器端框架，支持Python2和Python3，同时还支持PyPy。

cheroot的多线程机制简单的说就是一个线程池。线程池使用数组维护。其中的线程进行了自定义，可以实现线程的状态查询和管理。cheroot有一个主线程监听常用端口，主线程会将新的连接请求放入一个任务队列当中。线程池中的线程从任务队列中获取任务并执行其余流程。这其实是一个经典的“生产者-消费者”模式。

CPython和PyPy都有GIL，所以线程池是一个相对低效的实现。无论是内存占用还是逻辑流的切换开销，多线程机制远比事件驱动机制的开销要大。

## 看看评测

网上有一篇博文对几个Python服务器端框架进行了评测：[A Performance Analysis of Python WSGI Servers: Part 2](https://blog.appdynamics.com/engineering/a-performance-analysis-of-python-wsgi-servers-part-2/)，可以进行一下简单的分析。

* **并发处理能力**：bjoern优势太明显，显得好像做过弊（当然了，原文已经排除这一可能）。这可以归功于纯C语言的实现和libev。惊喜的是，CherryPy的并发处理能力居然超过了meinheld。后者是部分使用C语言实现的，基于http-parser框架和picoev框架，一个原作者称比libev更适合网络IO的事件驱动框架。按照bjoern和meinheld的官方文档描述，两者具有相似的结构设计，计算密集型流程使用的性能相近的第三方框架，所以两者应该水平相当。我还没有看meinheld的源码，不了解具体原因。 当然了，这次评测没有模拟数据库请求IO过程，所以和真实环境的并发并不完全一样。

```
	![每秒处理请求-All](/static/a_brief_about_web_server_framework_under_python/rps-1.png)
	![每秒处理请求-排除bjoern](/static/a_brief_about_web_server_framework_under_python/rps-2.png)
	
	```
* **处理延迟**：同时连接数不断增加时，CherryPy的处理延迟非常低，几乎为一条水平线。看上去，GIL并没有构成瓶颈。bjoern随着同时连接数的增加，处理延迟略有上升，但斜率很低。这里的处理延迟，表示的是从接收请求到发送响应之间的时间差。本次测试的应用端不涉及模版处理或网页生成，而是直接返回预设参数。

```
	![处理延迟](/static/a_brief_about_web_server_framework_under_python/latency.png)
	
	```
* **内存占用**：内存占用是异步框架的强项。bjoern和meinheld一起处于第一梯队，并和其他框架拉开了巨大的差距。

```
	![内存占用](/static/a_brief_about_web_server_framework_under_python/mem-usage.png)
	```
	
* **异常**：异常是指服务器突然断开连接或者连接超时。这个测试里，CherryPy展示出强大的实力，几乎为0。bjoern也不错，但同时连接数很高的时候，表现开始不那么稳定。 

```   
	![异常](/static/a_brief_about_web_server_framework_under_python/error.png)
	```
	
* **CPU占用**：这个因不同的底层机制而异。

```
	![CPU占用](/static/a_brief_about_web_server_framework_under_python/cpu-usage.png)
	
	```

# 参考阅读

1. [PEP 333 -- Python Web Server Gateway Interface v1.0](https://www.python.org/dev/peps/pep-0333/)
2. [PEP 3333 -- Python Web Server Gateway Interface v1.0.1](https://www.python.org/dev/peps/pep-3333/)
3. [WSGI简介](https://segmentfault.com/a/1190000003069785)
4. [一起写一个 Web 服务器（2）](http://python.jobbole.com/81523/)
5. [Learn about WSGI](http://wsgi.readthedocs.io/en/latest/learn.html)