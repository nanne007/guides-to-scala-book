## 提取器

在 Coursera 上，想必你遇到过一个非常强大的语言特性：
[模式匹配](http://en.wikipedia.org/wiki/Pattern_matching) 。
它可以解绑一个给定的数据结构。
这不是 Scala 所特有的，在其他出色的语言中，如 Haskell、Erlang，模式匹配也扮演着重要的角色。

模式匹配可以解构各种数据结构，包括 *列表* 、 *流* ，以及 *样例类* 。
但只有这些数据结构才能被解构吗，还是可以用某种方式扩展其使用范围？
而且，它实际是怎么工作的？
是不是有什么魔法在里面，得以写些类似下面的代码？

``` scala
 case class User(firstName: String, lastName: String, score: Int)
 def advance(xs: List[User]) = xs match {
   case User(_, _, score1) :: User(_, _, score2) :: _ => score1 - score2
   case _ => 0
 }
```

事实证明没有什么魔法，这都归功于[提取器](http://www.scala-lang.org/node/112) 。

提取器使用最为广泛的使用有着与 *构造器* 相反的效果：
构造器从给定的参数列表创建一个对象，
而提取器却是从传递给它的对象中提取出构造该对象的参数。
Scala 标准库包含了一些预定义的提取器，我们会大致的了解一下它们。

样例类非常特殊，Scala会自动为其创建一个 *伴生对象* ：
一个包含了 `apply` 和 `unapply` 方法的 *单例对象* 。
`apply` 方法用来创建样例类的实例，而 `unapply` 需要被伴生对象实现，以使其成为提取器。


### 第一个提取器

`unapply` 方法可能不止有一种方法签名，
不过，我们从只有最简单的开始，毕竟使用更广泛的还是只有一种方法签名的 `unapply` 。
假设要创建了一个 `User` 特质，有两个类继承自它，并且包含一个字段：

``` scala
trait User {
  def name: String
}
class FreeUser(val name: String) extends User
class PremiumUser(val name: String) extends User
```

我们想在各自的伴生对象中为 `FreeUser` 和 `PremiumUser` 类实现提取器，
就像 Scala 为样例类所做的一样。
如果想让样例类只支持从给定对象中提取单个参数，那 `unapply` 方法的签名看起来应该是这个样子：

``` scala
  def unapply(object: S): Option[T]
```

这个方法接受一个类型为 `S` 的对象，返回类型 `T` 的 `Option` ， `T` 就是要提取的参数类型。


> 在Scala中， `Option` 是 `null` 值的安全替代。
> 以后会有一个单独的章节来讲述它，不过现在，只需要知道，
> `unapply` 方法要么返回 `Some[T]` （如果它能成功提取出参数），要么返回 `None` ，
> `None` 表示参数不能被 `unapply` 具体实现中的任一提取规则所提取出。


下面的代码是我们的提取器：


``` scala
trait User {
  def name: String
}
class FreeUser(val name: String) extends User
class PremiumUser(val name: String) extends User
object FreeUser {
  def unapply(user: FreeUser): Option[String] = Some(user.name)
}
object PremiumUser {
  def unapply(user: PremiumUser): Option[String] = Some(user.name)
}
```

现在，可以在REPL中使用它：

``` scala
scala> FreeUser.unapply(new FreeUser("Daniel"))
res0: Option[String] = Some(Daniel)
```

如果调用返回的结果是 `Some[T]` ，说明提取模式匹配成功，如果是 `None` ，说明模式不匹配。

一般不会直接调用它，因为用于提取器模式时，Scala 会隐式的调用提取器的 `unapply` 方法。

``` scala
  val user: User = new PremiumUser("Daniel")
  user match {
    case FreeUser(name) => "Hello" + name
    case PremiumUser(name) => "Welcome back, dear" + name
  }
```

你会发现，两个提取器绝不会都返回 `None` 。
这个例子展示的提取器要比之前所见的更有意义。
如果你有一个类型不确定的对象，你可以同时检查其类型并解构。

这个例子里， `FreeUser` 模式并不会匹配，因为它接受的类型和我们传递给它的不一样。
这样一来， `user` 对象就会被传递给第二个模式，也就是 `PremiumUser` 伴生对象的 `unapply` 方法。
而这个模式会匹配成功，从而返回值就被绑定到 `name` 参数上。

在接下来的文章里，我们会看到一个并不总是返回 `Some[T]` 的提取器的例子。

### 提取多个值

现在，假设类有多个字段：

``` scala
trait User {
  def name: String
  def score: Int
}
class FreeUser(
  val name: String,
  val score: Int,
  val upgradeProbability: Double
) extends User
class PremiumUser(
  val name: String,
  val score: Int
) extends User
```

如果提取器想解构出多个参数，那它的 `unapply` 方法应该有这样的签名：


``` scala
def unapply(object: S): Option[(T1, ..., T2)]
```

这个方法接受类型为 `S` 的对象，返回类型参数为 `TupleN` 的 `Option` 实例，
`TupleN` 中的 `N` 是要提取的参数个数。

修改类之后，提取器也要做相应的修改：

``` scala
trait User {
  def name: String
  def score: Int
}
class FreeUser(
  val name: String,
  val score: Int,
  val upgradeProbability: Double
) extends User
class PremiumUser(
  val name: String,
  val score: Int
) extends User
object FreeUser {
  def unapply(user: FreeUser): Option[(String, Int, Double)] =
    Some((user.name, user.score, user.upgradeProbability))
}
object PremiumUser {
  def unapply(user: PremiumUser): Option[(String, Int)] =
    Some((user.name, user.score))
}
```

现在可以拿它来做模式匹配了：

``` scala
val user: User = new FreeUser("Daniel", 3000, 0.7d)
user match {
  case FreeUser(name, _, p) =>
    if (p > 0.75) "$name, what can we do for you today?"
    else "Hello $name"
  case PremiumUser(name, _) =>
    "Welcome back, dear $name"
}
```

### 布尔提取器

有些时候，进行模式匹配并不是为了提取参数，而是为了检查其是否匹配。
这种情况下，第三种 `unapply` 方法签名（也是最后一种）就有用了，
这个方法接受 `S` 类型的对象，返回一个布尔值：

``` scala
def unapply(object: S): Boolean
```

使用的时候，如果这个提取器返回 `true` ，模式会匹配成功，
否则，Scala 会尝试拿 `object` 匹配下一个模式。

上一个例子存在一些逻辑代码，用来检查一个免费用户有没有可能被说服去升级他的账户。
其实可以把这个逻辑放在一个单独的提取器中：

``` scala
object premiumCandidate {
  def unapply(user: FreeUser): Boolean = user.upgradeProbability > 0.75
}
```

你会发现，提取器不一定非要在这个类的伴生对象中定义。
正如其定义一样，这个提取器的使用方法也很简单：

``` scala
val user: User = new FreeUser("Daniel", 2500, 0.8d)
user match {
  case freeUser @ premiumCandidate() => initiateSpamProgram(freeUser)
  case _ => sendRegularNewsletter(user)
}
```

使用的时候，只需要把一个空的参数列表传递给提取器，因为它并不真的需要提取数据，自然也没必要绑定变量。

这个例子有一个看起来比较奇怪的地方：
我假设存在一个空想的 `initiateSpamProgram` 函数，其接受一个 `FreeUser` 对象作为参数。
模式可以与任何一种 `User` 类型的实例进行匹配，但 `initiateSpamProgram` 不行，
只有将实例强制转换为 `FreeUser` 类型， `initiateSpamProgram` 才能接受。

因为如此，Scala 的模式匹配也允许将提取器匹配成功的实例绑定到一个变量上，
这个变量有着与提取器所接受的对象相同的类型。这通过 `@` 操作符实现。
`premiumCandidate` 接受 `FreeUser` 对象，因此，变量 `freeUser` 的类型也就是 `FreeUser` 。

布尔提取器的使用并没有那么频繁（就我自己的情况来说），但知道它存在也是很好的，
或迟或早，你会遇到一个使用布尔提取器的场景。

### 中缀表达方式

解构列表、流的方法与创建它们的方法类似，都是使用 cons 操作符： `::` 、 `#::` ，比如：

``` scala
val xs = 58 #:: 43 #:: 93 #:: Stream.empty
xs match {
  case first #:: second #:: _ => first - second
  case _ => -1
}
```

你可能会对这种做法产生困惑。
除了我们已经见过的提取器用法，Scala 还允许以中缀方式来使用提取器。
所以，我们可以写成 `e(p1, p2)` ，也可以写成 `p1 e p2` ，
其中 `e` 是提取器， `p1` 、 `p2` 是要提取的参数。

同样，中缀操作方式的 `head #:: tail` 可以被写成 `#::(head, tail)` ，
提取器 `PremiumUser` 可以这样使用： `name PremiumUser score` 。
当然，这样做并没有什么实践意义。
一般来说，只有当一个提取器看起来真的像操作符，才推荐以中缀操作方式来使用它。
所以，列表和流的 `cons` 操作符一般使用中缀表达，而 `PreimumUser` 则不用。

### 进一步看流提取器

尽管 `#::` 提取器在模式匹配中的使用并没有什么特殊的，
但是，为了更好的理解上面的代码，还是进一步来分析一下。
而且，这是一个很好的例子，根据要匹配的数据结构的状态，提取器很可能返回 `None` 。

如下是 *Scala 2.9.2* 源代码中完整的 `#::` 提取器代码：

``` scala
object #:: {
  def unapply[A](xs: Stream[A]): Option[(A, Stream[A]) =
    if (xs.isEmpty) None
    else Some((xs.head, xs.tail))
}
```

如果给定的流是空的，提取器就直接返回 `None` 。
因此， `case head #:: tail` 就不会匹配任何空的流。
否则，就会返回一个 `Tuple2` ，其第一个元素是流的头，第二个元素是流的尾，尾本身又是一个流。
这样， `case head #:: tail` 就会匹配有一个或多个元素的流。
如果只有一个元素， `tail` 就会被绑定成空流。

为了理解流提取器是怎么在模式匹配中工作的，重写上面的例子，把它从中缀写法转成普通的提取器模式写法：

``` scala
val xs = 58 #:: 43 #:: 93 #:: Stream.empty
xs match {
  case #::(first, #::(second, _)) => first - second
  case _ => -1
}
```


首先为传递给模式匹配的初始流 `xs` 调用提取器。
由于提取器返回 `Some(xs.head, xs.tail)` ，从而 `first` 会绑定成 58，
`xs` 的尾会继续传递给提取器，提取器再一次被调用，返回首和尾， `second` 就被绑定成 `43` ，
而尾就绑定到通配符 `_` ，被直接扔掉了。

### 使用提取器

那到底该在什么时候使用、怎么使用自定义的提取器呢？尤其考虑到，使用样例类就能自动获得可用的提取器。

一些人指出，使用样例类、对样例类进行模式匹配打破了封装，
耦合了匹配数据和其具体实现的方式，这种批评通常是从面向对象的角度出发的。
如果想用 Scala 进行函数式编程，将样例类当作只包含纯数据（不包含行为）的
[代数数据类型](http://en.wikipedia.org/wiki/Algebraic_data_type) ，那它非常适合。

通常，只有当从无法掌控的类型中提取数据，或者是需要其他进行模式匹配的方法时，才需要实现自己的提取器。


> 提取器的一种常见用法是从字符串中提取出有意义的值，
> 作为练习，想一想如何实现 `URLExtractor` 以匹配代表 URL 的字符串。


### 小结

在这本书的第一章中，我们学习了 Scala 模式匹配背后的提取器，
学会了如何实现自己的提取器，及其在模式中的使用是如何和实现联系在一起的。
但是这并不是提取器的全部，下一章，将会学习如何实现可提取可变个数参数的提取器。
