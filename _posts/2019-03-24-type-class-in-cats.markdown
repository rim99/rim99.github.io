---
layout: post
title: "Cats中常见的Type Class" 
date: 2019-03-24 14:27:03 +8000
categories: 笔记
---

> 本文为[Scala with Cats]()的阅读笔记

# 1 Type Class

Type Class概念主要依靠隐式转换来提高代码的表达力。

假定有

```Scala
final case class Person(name: String, email: String)
```

Type Class 首先需要一个范型特质作为基本类型。

## 范型特质

```Scala
trait JsonWriter[A] {
  def write(value: A): Json
}
```

## 隐式转换的使用

### 接口函数

面向用户，接受用户显式的入参，同时也会接受一个范型特质的实例作为隐式入参。

```Scala
object Json {
  // 接口函数
  def toJson[A](value: A)(implicit w: JsonWriter[A]): Json =
    w.write(value)
}
```

使用方式：

```Scala
Json.toJson(Person("Dave", "dave@example.com"))
```

### 隐式类

也可以是一个接受用户入参的隐式类。

```Scala
object JsonSyntax {
  // 隐式转换类
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit w: JsonWriter[A]): Json =
      w.write(value)
  }
}
```

使用方式：

```Scala
Person("Dave", "dave@example.com").toJson
```


# 2 Monoid & Semigroup

Monoid 和 Semigroup 都是一种 Type Class 。

Monid总会提供一个满足结合律的`combine`方法，并提供一个empty对象。

```Scala
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

同时总满足：

```Scala
def associativeLaw[A](x: A, y: A, z: A) (implicit m: Monoid[A]): Boolean = {
  m.combine(x, m.combine(y, z)) ==
    m.combine(m.combine(x, y), z)
}

def identityLaw[A](x: A) (implicit m: Monoid[A]): Boolean = {
  (m.combine(x, m.empty) == x) &&
    (m.combine(m.empty, x) == x)
}
```

而 Semigroup 则要求略低一些，只需要提供一个满足结合律的`combine`方法即可。

```Scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}
```

Cats库为用户提供了`|+|`方法作为combine的等价实现，以期实现更好的代码表意能力。

## 应用范围

由于combine满足结合律，可以方便地执行数据集的加合。因此，在spark微批处理系统中，很适合分段处理数据然后对结果加合。除此之外，满足结合律的combine方法很适合作为Fork-Join任务的Join操作，线程安全且无序。

此外，在CRDT数据结构协商分布式一致性的时候，也很适合。



# 3 Functor

Functor是一种能够在一段上下文中完成一系列运算的抽象概念，例如`List`，`Option`等等。

考虑`List(1,2,3).map(_.toString)`。如果将这段代码理解为对List内每一个对象进行遍历后生成新的List，那么这种思维就还停留在指令式编程的范畴内。在函数式编程中，应该将这段代码视为对`List[Int]`到`List[String]`的“一次性”转换。注重行为，不过多在意实现方式。

`List`、`Option`、`Either`皆是如此。

`Future`也可以算成一种 Functor ，因为能够提供一系列的运算操作。但是`Future`对象的内部状态是变化的，所以不满足`引用透明`的要求，因此又不够“函数式”。

## 定义

```Scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}
```

Functor的map方法满足定律：

```Scala
fa.map(g(f(_))) == fa.map(f).map(g)
```

## Contravariant Functor

Contravariant Functor 在常规 Functor 的基础上提供了一个 `contramap`　方法能够产生新的Functor类型。例如

```Scala
// 定义
trait Printable[A] {
  def format(value: A): String
  def contramap[B](func: B => A): Printable[B] = ???
}

implicit val stringPrintable: Printable[String] =
  new Printable[String] {
    def format(value: String): String =
      "\"" + value + "\""
}
implicit val booleanPrintable: Printable[Boolean] = new Printable[Boolean] {
  def format(value: Boolean): String =
      if(value) "yes" else "no"
  }

// 假定有
final case class Box[A](value: A)

// 利用隐式转换来正常执行
format(Box("hello world"))
format(Box(true))
```

## Invariant Functor

Invariant Functor提供了一个 `imap`　方法，大致相当于 `map` 和 `contramap` 的结合。

都是产生新的Functor对象，`map` 相当于是在调用链尾追加函数行为，`contramap` 相当于是在调用链头预置函数行为，而 `imap` 则是通过双向映射来完成。

例如，

```Scala
// 定义
trait Codec[A] {
  def encode(value: A): String
  def decode(value: String): A
  def imap[B](dec: A => B, enc: B => A): Codec[B] = ???
}

// 基础Functor实例
implicit val stringCodec: Codec[String] =
  new Codec[String] {
    def encode(value: String): String = value
    def decode(value: String): String = value
  }

// 调用生成新的Codec类型实例
implicit val intCodec: Codec[Int] =
    stringCodec.imap(_.toInt, _.toString)
implicit val booleanCodec: Codec[Boolean] =
    stringCodec.imap(_.toBoolean, _.toString)
```

## 应用

在Hadoop这种大数据集的框架中，MapReduce往往需要将大数据集拆分，同时并行化处理。而这和 Functor 的 `map` 方法很契合。

# 4 Monad

## 定义

```Scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]
  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]
}
```

`pure`方法用实际值来构造一个Monad实例，而`flatMap`用来组成顺序计算路径的中间部分。

Monad 方法定义满足下面三条规则：

```Scala
val a = ???
def func(a: Any) = ???

// 第一条
pure(a).flatMap(func) == func(a)

val m = ???
// 第二条
m.flatMap(pure) == m

def f(a: Any) = ???
def g(a: Any) = ???
// 第三条
m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
```

## Cats 中常用的 Monad

### Identity

`cats.Id`是对实际类型的直接包装，大多用于编写 Monad 相关的单元测试。

### Either

Either 实际上是 Scala 标准库提供的类型。但是在 Scala 2.11 版本之前 Either 还没有 Monad 的方法，因此并不是严格意义上的 Monad。Cats 库对这个问题有对应的解决方案。

在 Scala 中 Option 可以避免出现“null指针”传递的情况，但是不能反映造成null的异常信息。而且在函数式编程中，抛异常是一种“副作用（side-effect）”。Either则可以解决这个问题。Either对象有左右两个位置填充值，一般情况下正常的值放在右侧，异常放在左侧。而且只能在同时设置其中一个值。

### MonadError

cats库提供了MonadError类型，类似于Either，但是能够更好的封装异常。

```Scala
trait MonadError[F[_], E] extends Monad[F] {
  // Lift an error into the `F` context:
  def raiseError[A](e: E): F[A]
  // Handle an error, potentially recovering from it:
  def handleError[A](fa: F[A])(f: E => A): F[A]
  // Test an instance of `F`,
  // failing if the predicate is not satisfied:
  def ensure[A](fa: F[A])(e: E)(f: A => Boolean): F[A]
}
```

其中F是一种Monad类型，E是异常类型。

### Eval

函数式编程中会区分计算模型是立即求值（eager-evaluate）还是惰性求值（lazy-evaluate）。在此基础之上，Eval Monad会更进一步考虑值是否会被记忆（memorized）。

例如，

```Scala
val x = <expression>
```

这里对x的赋值会被立即计算，并将结果保存在x中。也就是 eager & memorized 。

而，

```Scala
def y = <expression>
```

这里对y的调用，是一种延迟计算，而且每次调用都会计算一次，也就是 lazy & not memorized 。

再考虑，

```scala
lazy val z = <expression>
```

这里z是一种 lazy & memorized 的计算方式。

在这三种计算概念基础之上，Eval Monad有三种子类型：`Now`,`Later`和`Always`，分别对应于上述`x`,`y`,`z`三种情况。

#### 特别的Eval.defer

通常，如果没有尾递归优化，常见的递归函数计算深度过大时会造成栈溢出（stackoverflow）。例如，

```scala
def factorial(n: BigInt): BigInt = if(n == 1) n else n * factorial(n - 1)
```

`Eval.defer` 方法可以避免栈溢出。看例子，

```scala
def factorial(n: BigInt): Eval[BigInt] =
   if(n == 1) {
     Eval.now(n)
   } else {
     Eval.defer(factorial(n - 1).map(_ * n))
   }
```

这里defer方法创建新的Eval对象，将计算推迟至最后一刻，保证了栈安全（stack-safety）。当然，defer方法也是有代价的，新创建的Eval对象停留在堆区里。因此深度依然有限，限制于堆区的大小。

### Writer

Writer Monad 可以用来记录伴随计算逻辑产生的日志，并确保日志能够单独集中在一起打印，避免多线程环境下，多个计算逻辑的日志杂糅在一起。

Writer Monad 包含两个成员，value存放逻辑计算结果，written存放逻辑执行产生的日志序列。

在执行逻辑计算的时候，同一逻辑的日志被集中在一起，避免了与其他计算逻辑日志混杂在一起。

### Reader

Reader Monad 通常用来做依赖注入。

Reader Monad是对计算行为的抽象。不同的Reader可以组合在一起成为新的Reader。Reader Monad是一种函数级别的抽象。

```Scala
case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
type DbReader[A] = Reader[Db, A]
def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(db => db.usernames.get(userId))
def checkPassword(username: String, password: String): DbReader[Boolean] =
  Reader(db => db.passwords.get(username).contains(password))

val db = Db(
  Map(1 -> "Alice", 2 -> "Bob"),
  Map("Alice" -> "right", "Bob" -> "yyy")
)

val res1 = findUsername(1)(db)
println(res1) // Some(Alice)

val res2 = findUsername(2)(db)
println(res2) // None

def dbReaderInjected[A](dbReader: DbReader[A]): Id[A] = dbReader(db)
val res3 = res1.forall(name =>
  dbReaderInjected[Boolean](checkPassword(name, "bad passwd")))
println(res3) // false

val res4 = for {
  name <- findUsername(1)(db)
} yield checkPassword(name, "right")(db)
println(res4) // Some(true)
```

### State

State Monad 一般用来存储状态。



# 5 Semigroupal

`Semigroupal` 可以用来合并上下文（combine context）。他的定义如下：

```Scala
trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

其中，`fa`和`fb`是两个顺序无关的计算流。

例如，`Validated`可以用来校验表单，一并给出所有非法输入。

## 6 Applicative

实际上，在函数式编程领域中，`Semigroupal` 概念普及程度并不高。而`Aplicative`，也就是`Applicative functor`更为人知。其简化定义如下，

```Scala
trait Apply[F[_]] extends Semigroupal[F] with Functor[F] {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)] = ap(map(fa)(a => (b: B) => (a, b)))(fb)
}
trait Applicative[F[_]] extends Apply[F] {
  def pure[A](a: A): F[A]
}
```

`Applicative` 之于 `Apply` 正如同 `Monoid` 之于 `Semigroup`：多实现了 `pure` 方法，以封装其他值。

而`Monad`在`Applicative`的基础上多实现了`flatMap`方法。

**`Applicative`用于执行多个独立的effect-ful方法；而`Monoid`的执行流是串行的，如果一个方法抛出异常，后面的计算是不会执行的。**

## 7 Foldable

`Foldable` 是对 `foldLeft` 和 `foldRight` 的抽象。

两种方法都是对一个线性数据结构（如List）中的每一个元素依次迭代计算，区别是迭代顺序不同，一个是从头部开始，另一个从尾部开始。

## 8 Traverse

同样是对线性数据结构元素的遍历，`Foldable` 的两个方法容易让我们陷入其中的实现细节中，需要理解元素式如何聚合在一起的。

而 `Traverse` 则拥有更高一层的抽象。最好例子来自Scala的标准库：
1. `Future.traverse`：将一个 `List[A]` 利用函数 `A=>Future[B]` 转换出 `Future[List[B]]`
2. `Future.sequence`：将一个一个 `List[Future[A]]` 转换出 `Future[List[A]]`

`Traverse` 可以对List容器中的所有元素并行的执行计算。

## 延伸阅读

* [Applicatives vs Monads](https://www.tobyhobson.com/applicatives-vs-monads/)
