---
layout: post
title: Servlet 4 摘译之Servlet接口篇
date: 2018-02-15 19:53:02
categories: 翻译
---


Servlet是可以由Java实现的Web服务器组件动态加载的Java类，能够生成动态内容。

Servlet容器，有时也叫servlet引擎，负责调用Servlet API。Servlet容器通过“请求-响应”方式实现了servlet和web客户端的通信。Servlet容器需要实现MIME类型的请求和响应的编解码。所有servlet容器都应实现HTTP1.1和HTTP2.0协议，但对于HTTPS的支持不做强制要求。

# Servlet接口

`javax.servlet.GenericServlet`、`javax.servlet.http.HttpServlet`是两个常用的`javax.servlet.Servlet`接口的实现类，其中前者是个抽象类，定义了`service`抽象方法。通常，开发人员会继承后者实现自己的方法。

`service`方法其实是`Servlet`接口定义的，用于处理来自客户端的请求。每一次请求都会调用一次，因此需要合适的并发机制应对并发请求。

## 1 `HttpServlet`抽象类

`HttpServlet`抽象类提供了若干具体的Http协议相关的方法可以被`service`方法自动调用：

方法        |   Http请求
:--:       |    :--:
doGet      |   GET
doPost     |   POST
doDelete   |   DELETE
doHead     |   HEAD
doOptions  |   OPTIONS
doTrace    |   TRACE

`HttpServlet`抽象类的`doHead`方法只返回`doGet`方法的响应的头部；`doOptions`方法返回的是servlet所支持的http方法；`doTrace`方法返回的是客户端发送的`TRACE`方法的头部，主要用来debug。

`Servlet`设计目标是作为请求的终点，因此不支持`CONNECT`方法（主要用于代理）。

## 2 一个Servlet

通常，每一个Sevlet容器在每一个JVM环境里只生成一个`Servlet`对象。因此，开发人员需要自己处理好并发请求带来的同步问题。

如果实现了`SingleThreadModel`接口，那么Servlet容器会维护一个对象池，但此时容器在同一时间只允许一个线程调用其中一个对象。

`SingleThreadModel`**已经是官方弃用API**。

## 3 Servlet生命周期

`Servlet`生命周期通过接口的`init`、`service`、`destory`方法来管理。

### 3.1 加载与实例化

Sevlet容器负责`Servlet`实现类的加载与实例化，可以是容器启动后立即执行，也可以延迟执行。

Sevlet容器启动时，必须知晓`Servlet`实现类的文件位置。可以是本地文件系统，也可以是其他网络系统。

Sevlet容器使用一般的Java类加载机制来加载`Servlet`实现类，加载完成后会立即实例化。

### 3.2 初始化

`Servlet`类实例化完成后，容器应该立刻初始化servlet，并在响应请求之前完成。

`Servlet`对象初始化，一般会加载持久化的配置信息，初始化开销较大的对象以及执行其他一次性操作。

初始化动作通过调用`init`方法来完成，参数是一个`javax.servlet.ServletConfig`对象。

初始化过程有可能会抛出`UnavailableException`或者`ServletException`异常。这时，`Servlet`对象不会进入活跃状态，而是会被容器释放。

初始化失败后，容器稍后可以尝试实例化并初始化一个新的`Servlet`对象。

**特别注意**

>`init`方法的调用不同于一般意义的静态方法。
>开发人员必须在`init`方法成功调用后，才能认为servlet进入可用状态。

### 3.3 处理请求

`Servlet`对象成功的初始化后，容器就可以处理用户请求了。用户请求包含在`ServletRequest`的对象中，而`Servlet`对象的响应通过调用`ServletResponse`对象方法来完成。这两个对象都是`service`方法的参数。对于http请求，有相应的`HttpServletRequest`、`HttpServletResponse`对象。

需要注意的是，`Servlet`对象在其整个生命周期内也许不会处理到客户端的请求。

#### 3.3.1 多线程

Servlet容器应当能够使用多个线程并发地处理请求。

#### 3.3.2 处理请求的异常

`Servlet`对象在处理请求的过程中有可能会抛出`ServletException`或者`UnavailableException`。

前者表示`Servlet`对象在处理过程中出错，该异常应当Servlet容器被妥善处理。

后者表示`Servlet`对象无法处理请求。如果是**永久性**的无法处理，容器应该调用`destory`方法释放`Servlet`对象，解除活跃状态。所有因此而拒绝的请求都应该接受到`SC_NOT_FOUND`404返回状态码。如果是**临时性**的无法处理，容器应该拒绝向`Servlet`对象转发请求，并返回`SC_SERVICE_UNAVAILABLE`503返回状态码，而且响应头部应该包含`Retry-After`字段，说明该临时性状态何时解除。

#### 3.3.3 异步处理

Servlet容器支持异步的处理请求。容器可以有多条线程：一些在组织响应报文，调用`complete`方法；另一些可能在调用`AsyncContext.dispatch`方法分发请求。

一个典型的异步调用流程应该是：

1. `Servlet`对象接收请求,开始处理
2. `Servlet`对象发送耗时请求
3. `Servlet`对象返回，没有响应
4. 当请求资源可用时，处理请求的线程可以选择继续执行，也可以通过`AsyncContext.start`将该任务分发至容器中的其他资源（下面是官方文档的原文）

第十五章“Web Application Environment”和“Propagation of Security Identity in EJBTM Calls”两节中提到的特性，适用于初期响应请求的线程，或者请求通过`AsyncContext.dispatch`方法分发至容器。 其他线程可以通过`AsyncContext.start(Runnable) `方法直接操作响应对象。

`@WebServlet`和`@WebFilter`注解有一个布尔属性`asyncSupported`，默认值为`false`。如果这个属性为`true`，调用`startAsync`可以在不同的线程中异步的处理请求，传参为请求和响应对象的指针，然后在初始线程中返回。响应对象会按照请求对象经过的过滤器链逆序返回。当`AsyncContext`调用`complete`方法后，响应对象才会定型。当异步任务正在执行，但`startAsync`方法还在分发任务时，**【应用】**应该负责处理对响应对象和请求对象的并发获取。

`asyncSupported`属性为`true`的`Servlet`对象可以向属性为`false`的`Servlet`对象分发请求。这种情况下，后者的`service`方法退出的时候，响应对象才会定型。而容器需要调用`AsyncContext.complete`方法已通知感兴趣的`AsyncListener`对象。过滤器应该调用`AsyncListener.onComplete`方法清楚其所关联的资源，保证异步任务的成功返回。

同步`Servlet`对象向异步`Servlet`对象分发任务是非法的。但是`IllegalStateException`异常会在调用`startAsync`方法的抛出。因此，同步`Servlet`对象可以及时转成异步类型。

>This would allow a servlet to either function as a synchronous or an asynchronous servlet.

`AsyncContext.dispatch`方法可以实现：异步任务在任意线程上执行，向响应对象中写入内容。这个线程未必知道请求所经过的过滤器链。所以过滤器必须在请求传入的时候，包装响应对象。这样，写入响应对象的内容，仍然要被过滤器所处理。这样就保证了在任意线程向响应对象写入内容的效果是一致的。













