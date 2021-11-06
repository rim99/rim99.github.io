---
layout: post
title: "Goroutine的GMP模型"
date: 2021-11-06 14:28:28 +0800
categories: 原创
---

Goroutine是Go语言实现的并发编程的基础设施之一，是对顺序计算流程的抽象。从形式来看，Goroutine很像是线程Thread，因为同一个Goroutine里的逻辑都是顺序执行的。不过，Goroutine在实现上更为轻量。一个Goroutine的栈空间在默认情况下只有2KB，仅在必要时弹性变化。《Go语言并发之道》一书中对Linux系统线程和Goroutine的上下文切换开销做了对比。Linux系统线程切换一次上下文的耗时平均为1.4us，而Goroutine仅仅200多ns，是线程耗时的大约15%。

## GMP模型

Goroutine虽然看上去像是线程，但其实不是线程。操作系统的最小执行逻辑单元是线程。所有的方法都是依托于线程之上才得以执行。Goroutine也同样的道理，需要借助线程才能得到执行。

Go语言的运行时在执行Goroutine时采用了GMP模型：
* G - Goroutine
* M - Machine
* P - Processor

P指的是Go运行时的逻辑处理器。M状态机，就是对线程的抽象。一个P始终对应于一个M，但是一个M可以对应于多个G。

## Goroutine

与一般的方法/结构体不同，Goroutine需要能够独立的顺序执行普通方法。因此，Goroutine需要有类似与线程的PC计数器和栈来记录执行状态。Goroutine在自身挂起时保存状态，在恢复执行的时候将先前记录的状态恢复起来。

状态机M在执行Goroutine，本质上就是在更新相应Goroutine内部的PC计数器和执行栈。不过，M不能无限制的只执行一个Goroutine。否则，并发无从谈起。线程需要在合适情况下，切换到其他Goroutine上继续执行。为了保证性能，Goroutine不能随意的被挂起/恢复。因为Goroutine上下文切换需要内存随机IO来读取对应的状态。无节制的把时间花在内存IO上会导致CPU循环的浪费。所以说Goroutine的切换点的选择至关重要。在Go运行时的实现中，Goroutine会在如下情况下被切换：
1. 发生阻塞。例如发生系统调用、或者阻塞在互斥锁、Channel上。
2. 在调用函数之前，Goroutine需要增大栈空间。此时需要内存IO，因此让出处理器资源更为合理。

## Machine

如上所述，Goroutine的执行是非抢占式调度。M在内部维护一个本地队列，从而按顺序执行Goroutine。M的本地任务队列容量有限，因此更多执行就绪的Goroutine放置在Global队列中。M在执行完本地队列里的所有Goroutine之后，会在Global队列中提取任务继续执行。这个模型本质上是一个工作窃取（work-stealing）模型。

从一个Goroutine挂起到另一个Goroutine得到执行，这之间所有的操作都是由一个特殊的Goroutine `g0`来执行的。

刚刚挂起的Goroutine是没法继续执行的，因此被放置在另一个队列中。待到挂起条件解除，这个Goroutine就会放入就绪队列中等待执行。

## Processor

Go运行时的逻辑处理器，并不完全是实际物理机的CPU，其数量由runtime的GOMAXPROCS来决定

## 参考资料

1. [Go: What Does a Goroutine Switch Actually Involve?](https://medium.com/a-journey-with-go/go-what-does-a-goroutine-switch-actually-involve-394c202dddb7)
2. [Go: g0, Special Goroutine](https://medium.com/a-journey-with-go/go-g0-special-goroutine-8c778c6704d8)
3. [Go: Goroutine, OS Thread and CPU Management](https://medium.com/a-journey-with-go/go-goroutine-os-thread-and-cpu-management-2f5a5eaf518a)
