## 模式匹配与匿名函数

上一章总结了模式在 Scala 中的几种用法，最后提到了匿名函数。
这一章，我们具体的去学习如何在匿名函数中使用模式。

如果你参与过 Coursera 上的 [那门 Scala 课程](https://www.coursera.org/course/progfun) ，
或者写过 Scala 代码，那很可能你已经熟悉匿名函数。
比如说，将一组歌名转换成小写格式，你可能会定义一个匿名函数传递给 `map` 方法：

``` scala
val songTitles = List("The White Hare", "Childe the Hunter", "Take no Rogues")
songTitles.map(t => t.toLowerCase)
```

或者，利用 Scala 的 *占位符语法(placeholder syntax)* 得到更加简短的代码：

``` scala
songTitles.map(_.toLowerCase)
```

目前为止，一切都很顺利。
不过，让我们来看一个稍微有些区别的例子：
假设有一个由二元组组成的序列，每个元组包含一个单词，以及对应的词频，
我们的目标就是去除词频太高或者太低的单词，只保留中间地带的。
需要写出这样一个函数：

``` scala
wordsWithoutOutliers(wordFrequencies: Seq[(String, Int)]): Seq[String]
```

一个很直观的解决方案是使用 `filter` 和 `map` 函数：

``` scala
val wordFrequencies = ("habitual", 6) :: ("and", 56) :: ("consuetudinary", 2) ::
  ("additionally", 27) :: ("homely", 5) :: ("society", 13) :: Nil
def wordsWithoutOutliers(wordFrequencies: Seq[(String, Int)]): Seq[String] =
  wordFrequencies.filter(wf => wf._2 > 3 && wf._2 < 25).map(_._1)
wordsWithoutOutliers(wordFrequencies) // List("habitual", "homely", "society")
```

这个解法有几个问题。
首先，访问元组字段的代码不好看，如果我们可以直接解构出字段，那代码可能更加美观和可读。

幸好，Scala 提供了另外一种写匿名函数的方式：*模式匹配形式的匿名函数*，
它是由一系列模式匹配样例组成的，正如模式匹配表达式那样，不过没有 `match` 。
下面是重写后的代码：

``` scala
def wordsWithoutOutliers(wordFrequencies: Seq[(String, Int)]): Seq[String] =
 wordFrequencies.filter { case (_, f) => f > 3 && f < 25 } map { case (w, _) => w }
```

在两个匿名函数里，我们只使用了一个匹配案例，因为我们知道这个样例总是会匹配成功，
要解构的数据类型在编译期就确定了，没有会出错的可能。
这是模式匹配型匿名函数的一个非常常见的用法。

如果把这些匿名函数赋给其他值，你也会看到它们有着正确的类型：

``` scala
val predicate: (String, Int) => Boolean = { case (_, f) => f > 3 && f < 25 }
val transformFn: (String, Int) => String = { case (w, _) => w }
```


> 不过要注意，必须显示的声明值的类型，因为 Scala 编译器无法从匿名函数中推导出其类型。


当然，也可以定义一系列更加复杂的的匹配案例。
但是你必须的确保对于每一个可能的输入，都会有一个样例能够匹配成功，
不然，运行时会抛出 `MatchError` 。

### 偏函数

有时候可能会定义一个只处理特定输入的函数。
这样的一种函数能帮我们解决 `wordsWithoutOutliers` 中的另外一个问题：
在 `wordsWithoutOutliers` 中，我们首先过滤给定的序列，然后对剩下的元素进行映射，
这种处理方式需要遍历序列两次。
如果存在一种解法只需要遍历一次，那不仅可以节省一些 CPU，还会使得代码更简洁，更具有可读性。

Scala 集合的 API 有一个叫做 `collect` 的方法，对于 `Seq[A]` ，它有如下方法签名：

``` scala
def collect[B](pf: PartialFunction[A, B]): Seq[B]
```

这个方法将给定的 *偏函数(partial function)* 应用到序列的每一个元素上，
最后返回一个新的序列 - 偏函数做了 `filter` 和 `map` 要做的事情。

那偏函数到底是什么呢？
概括来说，偏函数是一个一元函数，它只在部分输入上有定义，
并且允许使用者去检查其在一个给定的输入上是否有定义。
为此，特质 `PartialFunction` 提供了一个 `isDefinedAt` 方法。
事实上，类型 `PartialFunction[-A, +B]` 扩展了类型 `(A) => B`
（一元函数，也可以写成 `Function1[A, B]` ）。
模式匹配型的匿名函数的类型就是 `PartialFunction` 。

依据继承关系，将一个模式匹配型的匿名函数传递给接受一元函数的方法（如：`map`、`filter`）是没有问题的，
只要这个匿名函数对于所有可能的输入都有定义。

不过 `collect` 方法接受的函数只能是 `PartialFunction[A, B]` 类型的。
对于序列中的每一个元素，首先检查偏函数在其上面是否有定义，
如果没有定义，那这个元素就直接被忽略掉，
否则，就将偏函数应用到这个元素上，返回的结果加入结果集。

现在，我们来重构 `wordsWithoutOutliers` ，首先定义需要的偏函数：

``` scala
val pf: PartialFunction[(String, Int), String] = {
  case (word, freq) if freq > 3 && freq < 25 => word
}
```

我们为这个案例加入了 *守卫语句*，不在区间里的元素就没有定义。

除了使用上面的这种方式，还可以显示的扩展 `PartialFunction` 特质：

``` scala
val pf = new PartialFunction[(String, Int), String] {
  def apply(wordFrequency: (String, Int)) = wordFrequency match {
    case (word, freq) if freq > 3 && freq < 25 => word
  }
  def isDefinedAt(wordFrequency: (String, Int)) = wordFrequency match {
    case (word, freq) if freq > 3 && freq < 25 => true
    case _ => false
  }
}
```

当然，前一种方法更为更为简洁。

把定义好的 `pf` 传递给 `map` 函数，能够通过编译期，但运行时会抛出 `MatchError` ，
因为我们的偏函数并不是在所有输入值上都有定义：

``` scala
wordFrequencies.map(pf) // will throw a MatchError
```

不过，把它传递给 `collect` 函数就能得到想要的结果：

``` scala
wordFrequencies.collect(pf) // List("habitual", "homely", "society")
```

这个结果和我们最初的实现所得到的结果是一样的，因此我们可以重写 `wordsWithoutOutliers`：

``` scala
def wordsWithoutOutliers(wordFrequencies: Seq[(String, Int)]): Seq[String] =
  wordFrequencies.collect { case (word, freq) if freq > 3 && freq < 25 => word }
```

偏函数还有其他一些有用的性质，比如说，它们可以被直接串联起来，实现函数式的
[责任链模式](http://en.wikipedia.org/wiki/Chain-of-responsibility_pattern)
（源自于面向对象程式设计）。

偏函数还是很多 Scala 库和 API 的重要组成部分。比如：
[Akka](http://akka.io) 中，actor 处理信息的方法就是通过偏函数来定义的。
因此，理解这一概念是非常重要的。

### 小结

在这一章中，我们学习了另一种定义匿名函数的方法：一系列的匹配样例，
它用一种非常简洁的方式让解构数据成为可能。
而且，我们还深入到偏函数这个话题，用一个简单的例子展示了它的用处。

下一章，我们将深入的学习已经出现过的 `Option` 类型，探索其存在的原因及其使用方式。
