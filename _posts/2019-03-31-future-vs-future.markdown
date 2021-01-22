---
layout: post
title: Future vs Future
date: 2019-03-31 18:31:44 +0800
categories: 原创
---

# Java vs Scala

Java和Scala中，Future是对线程执行结果的异步封装。尽管作用类似，但是用法却是大相径庭。

在Java中，如果想要在异步的获取Future对象后，继续表达接下来对线程执行结果的计算，很困难。因为，这个对象是Future对象，范型参数在运行时被擦除了。连类型都不知道，接下来的计算逻辑没办法做类型检查。想要知道类型，就只能获取结果。

如果要直接获取线程执行结果，只能用`get`方法阻塞式的读取。如果不愿意阻塞式的读取结果，只能反复的检查对象的`isDone`标记，然后再读取。

如果是前者，那么线程异步的好处被大大降低了；如果是后者，必须还有实现一个统一的Future对象状态查询的逻辑，例如可能是一个`While`循环，这个明显会是的代码逻辑更复杂。

```Java
Executor e = Executors.newCachedThreadPool()
class Compute {
  int result() {
    return 12;
  }
}

/** 
 * 上面是准备工作
 * 重点来了
 */
Futue<Integer> f = e.submit(() -> new Compute().result())
while (!f.isDone()) {
  ;
}
int processed = f.get() * 2 + 1
```

而Scala的Future就没有这个烦恼。利用for表达式，可以将Future里的值提取出来，继续计算，并将结果封装在Future中返回出来。

```Scala
implicit val executor = ExecutionContext.fromExecutor(Executors.newCachedThreadPool())
class Compute {
  def result: Int = 12 
}

/** 
 * 上面是准备工作
 * 重点来了
 */
val processed: Future[Int] = for {
  f <- Future[Int](new Compute.result)
  result = 2 * f + 1
} yield result
```

# 函数式编程思想的威力

同样是Future，两种语言中的用法却差异巨大。这是因为，两个API背后的设计理念不一样。

Java的Future是指令式编程理念的产物：事无巨细的告知Future对象的状态和状态控制方法，至于怎么用，是使用者的事情。

而Scala的Future则反映出函数式编程思想：将获取的延迟封装起来，其中的值保持透明引用，能够持续参与下文的计算。计算结果仍然封装在Future对象中。

这种获取值的延迟，就是函数式编程中的“副作用”，也就是不属于纯粹的值计算。

像Future这种，封装副作用的数据结构就称为Monad。除了封装获取延迟的Future之外，Scala标准库还提供了：Option封装了可能为null的值，以及Try、Either来封装可能抛出异常的值计算表达式。

此外，[The Essence of Functional Programming](https://page.mi.fu-berlin.de/scravy/realworldhaskell/materialien/the-essence-of-functional-programming.pdf)还列举了一些其他常见的Monad。

# Monad的定义

无论封装的是哪种副作用，Monad总能够提供大致类似的API定义：

```Scala
object MonadA {
  def unit(a: A)
}
trait MonadA extends Monad[A] {
  override def bind[B](fn: A => Monad[B]) = ???
}
```

其中：
 1. `unit`方法能够将值直接封装在Monad中
 2. `bind`方法接受一个函数为入参，将对象的值计算并转换为另一个值，并封装在同一种Monad中

其定义必须满足以下法则：

假设`val m = MonadA.unit(a)`:
 1. `MonadA.unit(a).bind(fn) == fn(a)`；
 2. `m.bind(unit) == m`；
 3. `m.bind(fa).bind(fb) == m.bind(fa(fb))`，这里`fa(fb)`表示两种函数组合而成的新函数。无论是Haskell语法还是Scala语法都有相近的表达方式，但不是这种形式。

# 最后一问

尽管如此Monad在Scala的for表达式加持下，威力巨大。但是，仍然有一个问题需要思考：

> 当不同的Monad类型，聚集在同一段计算流中。这些Monad该如何统一呢？


# 延伸阅读

- [傻瓜函数式编程](https://github.com/justinyhuang/Functional-Programming-For-The-Rest-of-Us-Cn/blob/master/FunctionalProgrammingForTheRestOfUs.cn.md)
- [函数式程序设计为什么至关重要](https://www.byvoid.com/zhs/blog/why-functional-programming)
- [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/chapters)
- [Real World Haskell 中文版](http://cnhaskell.com/index.html)

