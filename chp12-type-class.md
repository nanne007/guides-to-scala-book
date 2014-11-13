# 类型类

前两章讨论了几种保持 DRY 和灵活性的函数式编程技术：

1.  函数组合（function composition）
2.  部分函数应用（partial function application）
3.  柯里化（currying）

这一章依旧围绕代码灵活性而来，不过不再讨论作为头等公民的函数，而是类型系统（注意：并不是要真的去研究类型系统）。
你将学习 **类型类** ！

可能你会觉得这没有实际意义，认为这是被 Haskell 狂热分子带入 Scala 社区的异国情调，显然不是这样。
类型类已经成为 Scala 标准库，甚至是很多流行的、广泛使用的第三方开源库的重要组成部分，了解和熟悉类型类是很有必要的。

本章会讨论：

1.  类型类的概念，
2.  它为什么有用，
3.  使用它如何受益，
4.  如何实现类型类，并用于实践。

## 问题

我们用例子，而不是一个对类型类的抽象解释，开始本文的主题，例子简化了概念，也相当实用。

假设想提供一系列可以操作数字集合的函数，主要是计算它们的聚合值。
进一步假设只能通过索引来访问集合的元素，只能使用定义在 Scala 集合上的 `reduce` 方法。
（施加这些限制，是因为要实现的东西，Scala 标准库已经提供了）
最后，假定得到的值已排序。

我们先从 `median` ， `quartiles` ， `iqr` 的一个粗暴实现开始：

``` scala
    object Statistics {
      def median(xs: Vector[Double]): Double = xs(xs.size / 2)
      def quartiles(xs: Vector[Double]): (Double, Double, Double) =
        (xs(xs.size / 4), median(xs), xs(xs.size / 4 * 3))
      def iqr(xs: Vector[Double]): Double = quartiles(xs) match {
        case (lowerQuartile, _, upperQuartile) => upperQuartile - lowerQuartile
      }
      def mean(xs: Vector[Double]): Double = {
        xs.reduce(_ + _) / xs.size
      }
    }
```

`median` 将数据集分成两半，下四分位数和上四分位数（ `quartiles` 方法返回的元组的第一、第三个元素）分别分割了数据集的 25% 。
`iqr` 方法返回四分差（上四分卫数和下四分位数的差）。

现在我们想支持更多的类型，比如，`Int`，所以应该为这个类型实现上面这些方法，对吧？

不！不能想当然的为 `Vector[Int]` 重载上面的方法（诡异的技巧除外），因为类型参数会被擦除，而且这样做有代码冗余的嫌疑。

要是 `Int` 和 `Double` 扩展自一个共同的基类，或者都实现了一个像是 `Number` 这样的特质，那该多好！

你可能会想着去把上述方法需要的参数类型替换成更通用的类型，看起来会是这样：

``` scala
    object Statistics {
      def median(xs: Vector[Number]): Number = ???
      def quartiles(xs: Vector[Number]): (Number, Number, Number) = ???
      def iqr(xs: Vector[Number]): Number = ???
      def mean(xs: Vector[Number]): Number = ???
    }
```

这样做，不仅丢掉了先前的类型信息，还违背了扩展性：不能强制第三方的数字类型扩展 `Number` 特质。
幸运的是，本例并不存在这样一个通用的特质。

对于这种问题，Ruby 的做法是 **猴子补丁（monkey patching）** ，扩展新类型让它看起来像一个 `Number` ，但是这样会污染全局命名空间。
年轻时遭到 “四人帮” 打击的 Java 开发者，则会认为 **适配器（Adpater）** 能解决上面所有问题：

> “四人帮”这里指的是设计模式一书的作者：Erich Gamma、Richard Helm、Ralph Johnson 和 John Vlissides，
> 具体见：http://en.wikipedia.org/wiki/Design_Patterns

``` scala
    object Statistics {
      trait NumberLike[A] {
        def get: A
        def plus(y: NumberLike[A]): NumberLike[A]
        def minus(y: NumberLike[A]): NumberLike[A]
        def divide(y: Int): NumberLike[A]
      }
      case class NumberLikeDouble(x: Double) extends NumberLike[Double] {
        def get: Double = x
        def minus(y: NumberLike[Double]) = NumberLikeDouble(x - y.get)
        def plus(y: NumberLike[Double]) = NumberLikeDouble(x + y.get)
        def divide(y: Int) = NumberLikeDouble(x / y)
      }
      type Quartile[A] = (NumberLike[A], NumberLike[A], NumberLike[A])
      def median[A](xs: Vector[NumberLike[A]]): NumberLike[A] = xs(xs.size / 2)
      def quartiles[A](xs: Vector[NumberLike[A]]): Quartile[A] =
        (xs(xs.size / 4), median(xs), xs(xs.size / 4 * 3))
      def iqr[A](xs: Vector[NumberLike[A]]): NumberLike[A] = quartiles(xs) match {
        case (lowerQuartile, _, upperQuartile) => upperQuartile.minus(lowerQuartile)
      }
      def mean[A](xs: Vector[NumberLike[A]]): NumberLike[A] =
        xs.reduce(_.plus(_)).divide(xs.size)
    }
```

上述代码解决了扩展性问题：使用这个库的用户可以将类型通过 `NumberLike` 适配器传递过来，无需重新编译统计库。

但是，把数字封装在适配器里，这样的代码会令人厌倦，无论读写，而且和统计库交互时，必须创建一大堆适配器实例。

## 类型类来救援

对目前所介绍的方法来说，类型类是一个强大的替代。
类型类是 Haskell 语言一个突出的特征，虽然它的名字里有类，但它和面向对象编程里的类没有任何关系。

一个类型类 `C` 定义了一些行为，要想成为 `C` 的一员，类型 `T` 必须支持这些行为。
一个类型 `T` 到底是不是 类型类 `C` 的成员，这一点并不是与生俱来的。
开发者可以实现类必须支持的行为，使得这个类变成类型类的成员。
一旦 `T` 变成 类型类 `C` 的一员，参数类型为类型类 `C` 成员的函数就可以接受类型 `T` 的实例。

这样，类型类支持临时的、追溯性的多态，依赖类型类的代码支持扩展性，且无需创建任何适配器对象。

### 创建类型类

Scala 中，类型类可以通过技术组合来实现和使用，比之 Haskell，它在 Scala 里的参与度更高，而且带给开发者更多的控制。

创建一个类型类涉及到几个步骤。

首先，我们来定义一个特质：

``` scala
    object Math {
      trait NumberLike[T] {
        def plus(x: T, y: T): T
        def divide(x: T, y: Int): T
        def minus(x: T, y: T): T
      }
    }
```

上述代码创建了名为 `NumberLike` 的类型类特质。
类型类总会带着一个或多个类型参数，通常是无状态的，比如：里面定义的方法只对传入的参数进行操作。
前文的适配器操作的是它自己的字段和接受的一个参数，而这里定义的方法都需要两个参数，其中第一个参数对应适配器中的字段。

### 提供默认成员

第二步通常是在伴生对象里提供一些默认的类型类特质实现，之后你会知道为什么要这么做。
在这之前，先来实现 `Double` 和 `Int` 的类型类特质：

``` scala
    object Math {
      trait NumberLike[T] {
        def plus(x: T, y: T): T
        def divide(x: T, y: Int): T
        def minus(x: T, y: T): T
      }
      object NumberLike {
        implicit object NumberLikeDouble extends NumberLike[Double] {
          def plus(x: Double, y: Double): Double = x + y
          def divide(x: Double, y: Int): Double = x / y
          def minus(x: Double, y: Double): Double = x - y
        }
        implicit object NumberLikeInt extends NumberLike[Int] {
          def plus(x: Int, y: Int): Int = x + y
          def divide(x: Int, y: Int): Int = x / y
          def minus(x: Int, y: Int): Int = x - y
        }
      }
    }
```

两件事情：
第一，这两个实现基本相同。但不总是这样，毕竟 `NumberLike` 只是一个很小的域。
后面会给出类型类的一些例子，当为这些例子实现多个类型时，重复的余地就少很多。
第二， `NumberLikeInt` 做整数除法的时候，会损失一些精度，请忽略这一事实，这只是为简单起见。

你也许会发现，类型类的成员通常是单例对象，而且会有一个 `implicit` 关键字位于前面，
这是类型类在 Scala 中成为可能的几个重要因素之一，在某些条件下，它让类型类成员隐式可用。
更多相关的知识在下一节。

### 运用类型类

有了类型类和两个默认实现之后，就可以根据它们来实现统计。
我们先将重点放在 `mean` 方法上：

``` scala
    object Statistics {
      import Math.NumberLike
      def mean[T](xs: Vector[T])(implicit ev: NumberLike[T]): T =
        ev.divide(xs.reduce(ev.plus(_, _)), xs.size)
    }
```

这样的代码初看起来可能有点吓人，实际上是相当简单，方法带有一个类型参数 `T` ，接受类型为 `Vector[T]` 的参数。

将参数限制在特定类型类的成员上，是通过第二个 `implicit` 参数列表实现的。
这是什么意思？这是说，当前作用域中必须存在一个隐式可用的 `NumberLike[T]` 对象，比如说，当前作用域声明了一个 **隐式值(implicit value)**。
这种声明很多时候都是通过导入一个有隐式值定义的包或者对象来实现的。

当且仅当没有发现其他隐式值时，编译器会在隐式参数类型的伴生对象中寻找。
作为库的设计者，将默认的类型类实现放在伴生对象里意味着库的使用者可以轻易的重写默认实现，这正是库设计者喜闻乐见的。
用户还可以为隐式参数传递一个显示值，来重写作用域内的隐式值。

让我们来验证下默认的实现是否可以被正确解析：

``` scala
    val numbers = Vector[Double](13, 23.0, 42, 45, 61, 73, 96, 100, 199, 420, 900, 3839)
    println(Statistics.mean(numbers))
```

漂亮极了！试试 `Vector[String]` ，你会在编译期得到一个错误，这个错误指出参数 `ev: NumberLike[String]` 没有隐式值可用。
如果你不喜欢这个错误消息，你可以用 `@implicitNotFound` 为类型类添加批注，来自定义错误消息：

``` scala
    object Math {
      import annotation.implicitNotFound
      @implicitNotFound("No member of type class NumberLike in scope for ${T}")
      trait NumberLike[T] {
        def plus(x: T, y: T): T
        def divide(x: T, y: Int): T
        def minus(x: T, y: T): T
      }
    }
```

### 上下文绑定

总是带着这个隐式参数列表显得有些冗长。
对于只有一个类型参数的隐式参数，Scala 提供了一种叫做 **上下文绑定(context bound)** 的简写。
为了说明这一使用方法，我们用它来实现剩下的统计方法：

``` scala
    object Statistics {
      import Math.NumberLike
      def mean[T](xs: Vector[T])(implicit ev: NumberLike[T]): T =
        ev.divide(xs.reduce(ev.plus(_, _)), xs.size)
      def median[T : NumberLike](xs: Vector[T]): T = xs(xs.size / 2)
      def quartiles[T: NumberLike](xs: Vector[T]): (T, T, T) =
        (xs(xs.size / 4), median(xs), xs(xs.size / 4 * 3))
      def iqr[T: NumberLike](xs: Vector[T]): T = quartiles(xs) match {
        case (lowerQuartile, _, upperQuartile) =>
          implicitly[NumberLike[T]].minus(upperQuartile, lowerQuartile)
      }
    }
```

上下文绑定 `T: NumberLike` 意思是，必须有一个类型为 `NumberLike[T]` 的隐式值在当前上下文中可用，这和隐式参数列表是等价的。
如果想要访问这个隐式值，需要调用 `implicitly` 方法，就像上述 `iqr` 方法所做的那样。
如果类型类需要多个类型参数，就不能使用上下文绑定语法了。

### 自定义的类型类成员

含有类型类的库的使用者，或迟或早会想将他自己的类型加入到类型类成员中。
比如说，可能想将统计用在 Joda Time 的 `Duration` 实例上。

我们来试试吧。首先将 Joda Time 加入到路径里：

``` scala
    libraryDependencies += "joda-time" % "joda-time" % "2.1"

    libraryDependencies += "org.joda" % "joda-convert" % "1.3"
```

现在，只需创建 `NumberLike` 的一个隐式实现：

``` scala
    object JodaImplicits {
      import Math.NumberLike
      import org.joda.time.Duration
      implicit object NumberLikeDuration extends NumberLike[Duration] {
        def plus(x: Duration, y: Duration): Duration = x.plus(y)
        def divide(x: Duration, y: Int): Duration = Duration.millis(x.getMillis / y)
        def minus(x: Duration, y: Duration): Duration = x.minus(y)
      }
    }
```

导入包含这个实现的包或者对象，就可以计算一堆 durations 的平均值了：

``` scala
    import Statistics._
    import JodaImplicits._
    import org.joda.time.Duration._

    val durations = Vector(standardSeconds(20), standardSeconds(57), standardMinutes(2),
      standardMinutes(17), standardMinutes(30), standardMinutes(58), standardHours(2),
      standardHours(5), standardHours(8), standardHours(17), standardDays(1),
      standardDays(4))
    println(mean(durations).getStandardHours)
```

## 使用场景

`NumberLike` 类型类是一个非常好的例子，但 Scala 已经有 `Numeric` 了。
对于集合的类型参数 `T` ，只要存在一个可用的 `Numeric[T]`，就可以在该集合上调用 `sum` 、 `product` 这样的方法。
标准库中另一个使用比较多的类型类是 `Ordering`，可以为自定义类型提供一个隐式排序，用在 Scala 集合的 `sort` 方法。

标准库中还有更多这样的类型类，不过，Scala 开发者并不需要与它们中的每一个都打交道。

第三方库中一个非常常见的用例是对象序列化和反序列化，尤其是 JSON 对象。
使一个类成为某个格式器类型类的成员，就可以自定义类的序列化方式，序列化成 JSON、XML 或者是任何新的格式。

Scala 类型和数据库驱动支持的类型之间的映射，通常也是通过类型类获得自定义和可扩展性的。

## 总结

一旦开始用 Scala 来做些正式的工作，不可避免的会遇到类型类。
希望读者在读完这一章后，能够利用好这一强大技术。

Scala 类型类使得在开发 Scala 应用时，一方面可以有无限可追加的扩展，
另一方面又可以保留尽可能多的具体类型信息。

和其他语言应对这种问题的方法想比，Scala 给予了开发者完全的控制权，因为类型类的实现可以被轻易的重写，而且在全局命名空间里不可用。

你将看到这种技术在编写由其他人使用的库时尤其有用，在应用程序代码中，为了减少模块之间的耦合，类型类也是有用武之地的。
