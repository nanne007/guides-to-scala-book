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
