---
layout: post
title: 如何构建领域模型 
date: 2018-06-18 14:52:02
categories: 笔记
---


>之前简单的了解过DDD（领域驱动设计）的常见概念，但是不了解怎么去应用。最近在Spring.io看到一篇博客[Event Storming and Spring with a Splash of DDD](https://spring.io/blog/2018/04/11/event-storming-and-spring-with-a-splash-of-ddd)，正好就是这方面的内容。学习学习，记录一下。
>本文只是笔记，需要配合原文一起理解。

确保项目又快又好地交付的几点原则：

* 理解：帮助团队更好的理解业务流程，让领域模型贴近实际。
* 解耦：把大的软件系统解耦成为小的模块，这样每一个模块都可以有适合自己的结构。
* 实施：持续重构，将大的单体系统分解为小的分布式系统。当然某些时候，这样做未必好。
* 部署：建立测试驱动开发、持续集成、持续交付的习惯。
* 价值观：使用趁手的工具，降低编码本身的耗时，让团队有更多的时间理解业务。

作者写了两个demo，一个使用了CQRS + EventSourcing架构，另个就是原文的主角，端到端的DDD架构。源代码在[这里](https://github.com/ddd-by-examples)。

>CQRS架构是一种读写分离的高性能架构，有利于体系水平伸缩。但是根据Martin Fowler的[介绍](https://www.martinfowler.com/bliki/CQRS.html)，这种结构的适用场景并不多。不恰当的使用会导致代码复杂，反而降低了性能和维护性。

原文的业务场景是一个简单的信用卡管理系统。

# 如何更好的理解业务

作者推荐的方法是——[事件风暴 Event Storming](http://www.infoq.com/cn/news/2016/07/event-storming-ddd)。

首先在空白的展示板上，用橙色的贴纸写下**会发生的事情**，这就是领域事件。

然后在这些橙色贴纸旁边标记出引发这些事件的原因：

1. 直接指令，写在蓝色贴纸上；
2. 其他事件，就把这些橙色贴纸放在一起；
3. 有些事件是周期性发生的，就写一个小的`TIME`标注吧。

一些查询指令，也会引发领域事件。但是这些事件，对整个系统的影响并不大，只是读取信息，所以用绿色贴纸标记出来。

最关键的一步是，这些触发操作很可能是需要一定前提条件的。这些条件被称作变量，用黄色卡片标记。并放置在原因与事件之间。

# 如何分解系统

分解系统需要首先定位出业务模块的边界，也就是有界上下文：一起使用或变化的东西应该放在一起。

所以，变量应该与事件放在一起；如果绿色贴纸关联的模型结果会随着某个事件发生变化，这也应该放在一起。

试着找找这样的模式：

1. 命令CmdA引发事件EventA；
2. 事件EventA影响了视图SomeView；
3. 命令CmdB正好与SomeView有关联；
4. 这也就是说CmdA和CmdB有很强的耦合关系。

当然了，上面这个只是个例子。未必适合所有场景。只要定位出耦合关系就好了。

最后一步，决定各个模块如何沟通，也就是上下文映射。

# 领域模型的实现

完成领域建模之后，就可以开始愉快的写代码了。

不过首先得问下自己，要不要搞分布式的微服务，[这篇文章](https://content.pivotal.io/blog/should-that-be-a-microservice-keep-these-six-factors-in-mind)会给你答案。即使是不需要做分布式系统，也应该利用接口解耦，保持模块间良好的上下文边界。

然后就是开发了。

作者推荐在TDD中使用[Spring Cloud Contract](https://cloud.spring.io/spring-cloud-contract/)来模拟服务之间的的请求返回过程；使用[Spring Cloud Pipelines](https://cloud.spring.io/spring-cloud-pipelines/)来实现持续集成和持续交付。

# 延伸阅读

* [大家一直在谈的领域驱动设计（DDD），我们在互联网业务系统是这么实践的](https://mp.weixin.qq.com/s/jMWuMuIvI1cFThC-WQGbHQ)
* [领域驱动设计之领域模型](http://www.cnblogs.com/netfocus/archive/2011/10/10/2204949.html)
* [CQRS架构简介](http://www.cnblogs.com/netfocus/p/4055346.html)
* [命令和查询责任分离 (CQRS) 模式](https://docs.microsoft.com/zh-cn/azure/architecture/patterns/cqrs)
* [事件溯源模式](https://docs.microsoft.com/zh-cn/azure/architecture/patterns/event-sourcing)
* [Spring Cloud Contract in a polyglot world](https://spring.io/blog/2018/02/13/spring-cloud-contract-in-a-polyglot-world)


