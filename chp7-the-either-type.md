## 类型 Either

上一章介绍了 Try，它用函数式风格来处理程序错误。
这一章我们介绍一个和 Try 相似的类型 - Either，
学习如何去使用它，什么时候去使用它，以及它有什么缺点。

不过首先得知道一件事情：
在写作这篇文章的时候，Either 有一些设计缺陷，很多人都在争论到底要不要使用它。
既然如此，为什么还要学习它呢？
因为，在理解 Try 这个错综复杂的类型之前，不是所有人都会在代码中使用 Try 风格的异常处理。
其次，Try 不能完全替代 Either，它只是 Either 用来处理异常的一个特殊用法。
Try 和 Either 互相补充，各自侧重于不同的使用场景。

因此，尽管 Either 有缺陷，在某些情况下，它依旧是非常合适的选择。

### Either 语义

Either 也是一个容器类型，但不同于 Try、Option，它需要两个类型参数：
`Either[A, B]` 要么包含一个类型为 `A` 的实例，要么包含一个类型为 `B` 的实例。
这和 `Tuple2[A, B]` 不一样， `Tuple2[A, B]` 是两者都要包含。

Either 只有两个子类型： Left、 Right，
如果 `Either[A, B]` 对象包含的是 `A` 的实例，那它就是 Left 实例，否则就是 Right 实例。

在语义上，Either 并没有指定哪个子类型代表错误，哪个代表成功，
毕竟，它是一种通用的类型，适用于可能会出现两种结果的场景。
而异常处理只不过是其一种常见的使用场景而已，
不过，按照约定，处理异常时，Left 代表出错的情况，Right 代表成功的情况。

### 创建 Either

创建 Either 实例非常容易，Left 和 Right 都是样例类。
要是想实现一个 “坚如磐石” 的互联网审查程序，可以直接这么做：

``` scala
import scala.io.Source
import java.net.URL
def getContent(url: URL): Either[String, Source] =
 if(url.getHost.contains("google"))
   Left("Requested URL is blocked for the good of the people!")
 else
   Right(Source.fromURL(url))
```

调用 `getContent(new URL("http://danielwestheide.com"))` 会得到一个封装有
`scala.io.Source` 实例的 Right，
传入 `new URL("https://plus.google.com")` 会得到一个含有 `String` 的 Left。

### Either 用法

Either 基本的使用方法和 Option、Try 一样：
调用 `isLeft` （或 `isRight` ）方法询问一个 Either，判断它是 Left 值，还是 Right 值。
可以使用模式匹配，这是最方便也是最为熟悉的一种方法：

``` scala
getContent(new URL("http://google.com")) match {
 case Left(msg) => println(msg)
 case Right(source) => source.getLines.foreach(println)
}
```

#### 立场

你不能，至少不能直接像 Option、Try 那样把 Either 当作一个集合来使用，
因为 Either 是 **无偏(unbiased)** 的。

Try 偏向 Success：
`map` 、 `flatMap` 以及其他一些方法都假设 Try 对象是一个 Success 实例，
如果是 Failure，那这些方法不做任何事情，直接将这个 Failure 返回。

但 Either 不做任何假设，这意味着首先你要选择一个立场，假设它是 Left 还是 Right，
然后在这个假设的前提下拿它去做你想做的事情。
调用 `left` 或 `right` 方法，就能得到 Either 的 `LeftProjection` 或 `RightProjection `实例，
这就是 Either 的 *立场(Projection)* ，它们是对 Either 的一个左偏向的或右偏向的封装。

#### 映射

一旦有了 Projection，就可以调用 `map` ：

``` scala
val content: Either[String, Iterator[String]] =
  getContent(new URL("http://danielwestheide.com")).right.map(_.getLines())
// content is a Right containing the lines from the Source returned by getContent
val moreContent: Either[String, Iterator[String]] =
  getContent(new URL("http://google.com")).right.map(_.getLines)
// moreContent is a Left, as already returned by getContent

// content: Either[String,Iterator[String]] = Right(non-empty iterator)
// moreContent: Either[String,Iterator[String]] = Left(Requested URL is blocked for the good of the people!)
```

这个例子中，无论 `Either[String, Source]` 是 Left 还是 Right，
它都会被映射到 `Either[String, Iterator[String]]` 。
如果，它是一个 Right 值，这个值就会被 `_.getLines()` 转换；
如果，它是一个 Left 值，就直接返回这个值，什么都不会改变。

LeftProjection也是类似的：


``` scala
val content: Either[Iterator[String], Source] =
  getContent(new URL("http://danielwestheide.com")).left.map(Iterator(_))
// content is the Right containing a Source, as already returned by getContent
val moreContent: Either[Iterator[String], Source] =
  getContent(new URL("http://google.com")).left.map(Iterator(_))
// moreContent is a Left containing the msg returned by getContent in an Iterator

// content: Either[Iterator[String],scala.io.Source] = Right(non-empty iterator)
// moreContent: Either[Iterator[String],scala.io.Source] = Left(non-empty iterator)
```

现在，如果 Either 是个 Left 值，里面的值会被转换；如果是 Right 值，就维持原样。
两种情况下，返回类型都是 `Either[Iterator[String, Source]` 。


> 请注意， `map` 方法是定义在 Projection 上的，而不是 Either，
> 但其返回类型是 Either，而不是 Projection。

> 可以看到，Either 和其他你知道的容器类型之所以不一样，就是因为它的无偏性。
> 接下来你会发现，在特定情况下，这会产生更多的麻烦。
> 而且，如果你想在一个 Either 上多次调用 `map` 、 `flatMap` 这样的方法，
> 你总需要做 Projection，去选择一个立场。


#### Flat Mapping

Projection 也支持 flat mapping，避免了嵌套使用 map 所造成的令人费解的类型结构。

假设我们想计算两篇文章的平均行数，下面的代码可以解决这个 “富有挑战性” 的问题：

``` scala
val part5 = new URL("http://t.co/UR1aalX4")
val part6 = new URL("http://t.co/6wlKwTmu")
val content = getContent(part5).right.map(a =>
  getContent(part6).right.map(b =>
    (a.getLines().size + b.getLines().size) / 2))
// => content: Product with Serializable with scala.util.Either[String,Product with Serializable with scala.util.Either[String,Int]] = Right(Right(537))
```

运行上面的代码，会得到什么？
会得到一个类型为 `Either[String, Either[String, Int]]` 的玩意儿。
当然，你可以调用 `joinRight` 方法来使得这个结果 **扁平化(flatten)** 。

不过我们可以直接避免这种嵌套结构的产生，
如果在最外层的 RightProjection 上调用 `flatMap` 函数，而不是 `map` ，
得到的结果会更好看些，因为里层 Either 的值被解包了：

``` scala
val content ＝ getContent(part5).right.flatMap(a =>
  getContent(part6).right.map(b =>
    (a.getLines().size + b.getLines().size) / 2))
// => content: scala.util.Either[String,Int] = Right(537)
```

现在， `content` 值类型变成了 `Either[String, Int]` ，处理它相对来说就很容易了。

#### for 语句

说到 for 语句，想必现在，你应该已经爱上它在不同类型上的一致性表现了。
在 for 语句中，也能够使用 `Either` 的 Projection，但遗憾的是，这样做需要一些丑陋的变通。

假设用 for 语句重写上面的例子：

``` scala
def averageLineCount(url1: URL, url2: URL): Either[String, Int] =
  for {
    source1 <- getContent(url1).right
    source2 <- getContent(url2).right
  } yield (source1.getLines().size + source2.getLines().size) / 2
```

这个代码还不是太坏，毕竟只需要额外调用 `left` 、 `right` 。

但是你不觉得 yield 语句太长了吗？现在，我就把它移到值定义块中：

``` scala
def averageLineCountWontCompile(url1: URL, url2: URL): Either[String, Int] =
  for {
    source1 <- getContent(url1).right
    source2 <- getContent(url2).right
    lines1 = source1.getLines().size
    lines2 = source2.getLines().size
  } yield (lines1 + lines2) / 2
```

试着去编译它，然后你会发现无法编译！如果我们把 for 语法糖去掉，原因可能会清晰些。
展开上面的代码得到：

``` scala
def averageLineCountDesugaredWontCompile(url1: URL, url2: URL): Either[String, Int] =
  getContent(url1).right.flatMap { source1 =>
    getContent(url2).right.map { source2 =>
      val lines1 = source1.getLines().size
      val lines2 = source2.getLines().size
      (lines1, lines2)
    }.map { case (x, y) => x + y / 2 }
  }
```

问题在于，在 for 语句中追加新的值定义会在前一个 `map` 调用上自动引入另一个 `map` 调用，
前一个 `map` 调用返回的是 Either 类型，不是 RightProjection 类型，
而 Scala 并没有在 Either 上定义 `map` 函数，因此编译时会出错。

这就是 Either 丑陋的一面。要解决这个例子中的问题，可以不添加新的值定义。
但有些情况，就必须得添加，这时候可以将值封装成 Either 来解决这个问题：

``` scala
def averageLineCount(url1: URL, url2: URL): Either[String, Int] =
  for {
    source1 <- getContent(url1).right
    source2 <- getContent(url2).right
    lines1 <- Right(source1.getLines().size).right
    lines2 <- Right(source2.getLines().size).right
  } yield (lines1 + lines2) / 2
```

认识到这些设计缺陷是非常重要的，这不会影响 Either 的可用性，但如果不知道发生了什么，它会让你感到非常头痛。

#### 其他方法

Projection 还有其他有用的方法：

1. 可以在 Either 的某个 Projection 上调用 `toOption` 方法，将其转换成 Option。

   假如，你有一个类型为 `Either[A, B]` 的实例 `e` ， `e.right.toOption` 会返回一个 `Option[B]` 。
   如果 `e` 是一个 Right 值，那这个 `Option[B]` 会是 Some 类型，
   如果 `e` 是一个 Left 值，那 `Option[B]` 就会是 `None` 。
   调用 `e.left.toOption` 也会有相应的结果。
2. 还可以用 `toSeq` 方法将 Either 转换为序列。

#### Fold 函数

如果想变换一个 Either（不论它是 Left 值还是 right 值），可以使用定义在 Either 上的 `fold` 方法。
这个方法接受两个返回相同类型的变换函数，
当这个 Either 是 Left 值时，第一个函数会被调用；否则，第二个函数会被调用。

为了说明这一点，我们用 `fold` 重写之前的一个例子：

``` scala
val content: Iterator[String] =
  getContent(new URL("http://danielwestheide.com")).fold(Iterator(_), _.getLines())
val moreContent: Iterator[String] =
  getContent(new URL("http://google.com")).fold(Iterator(_), _.getLines())
```

这个示例中，我们把 `Either[String, String]` 变换成了 `Iterator[String]` 。
当然，你也可以在变换函数里返回一个新的 Either，或者是只执行副作用。
`fold` 是一个可以用来替代模式匹配的好方法。

### 何时使用 Either

知道了 Either 的用法和应该注意的事项，我们来看看一些特殊的用例。

#### 错误处理

可以用 Either 来处理异常，就像 Try 一样。
不过 Either 有一个优势：可以使用更为具体的错误类型，而 Try 只能用 `Throwable` 。
（这表明 Either 在处理自定义的错误时是个不错的选择）
不过，需要实现一个方法，将这个功能委托给 `scala.util.control` 包中的 `Exception` 对象：

``` scala
import scala.util.control.Exception.catching
def handling[Ex <: Throwable, T](exType: Class[Ex])(block: => T): Either[Ex, T] =
  catching(exType).either(block).asInstanceOf[Either[Ex, T]]
```

这么做的原因是，虽然 `scala.util.Exception` 提供的方法允许你捕获某些类型的异常，
但编译期产生的类型总是 `Throwable` ，因此需要使用 `asInstanceOf` 方法强制转换。

有了这个方法，就可以把期望要处理的异常类型，放在 Either 里了：

``` scala
import java.net.MalformedURLException
def parseURL(url: String): Either[MalformedURLException, URL] =
  handling(classOf[MalformedURLException])(new URL(url))
```

`handling` 的第二个参数 `block` 中可能还会有其他产生错误的情形，
而且并不是所有情形都会抛出异常。
这种情况下，没必要为了捕获异常而人为抛出异常，相反，只需定义你自己的错误类型，最好是样例类，
并在错误情况发生时返回一个封装了这个类型实例的 Left。

下面是一个例子：

``` scala
case class Customer(age: Int)
class Cigarettes
case class UnderAgeFailure(age: Int, required: Int)
def buyCigarettes(customer: Customer): Either[UnderAgeFailure, Cigarettes] =
  if (customer.age < 16) Left(UnderAgeFailure(customer.age, 16))
  else Right(new Cigarettes)
```

应该避免使用 Either 来封装意料之外的异常，
使用 Try 来做这种事情会更好，至少它没有 Either 这样那样的缺陷。

#### 处理集合

有些时候，当按顺序依次处理一个集合时，里面的某个元素产生了意料之外的结果，
但是这时程序不应该直接引发异常，因为这样会使得剩下的元素无法处理。
Either 也非常适用于这种情况。

假设，在我们 “行业标准般的” Web 审查系统里，使用了某种黑名单：

``` scala
type Citizen = String
case class BlackListedResource(url: URL, visitors: Set[Citizen])

val blacklist = List(
  BlackListedResource(new URL("https://google.com"), Set("John Doe", "Johanna Doe")),
  BlackListedResource(new URL("http://yahoo.com"), Set.empty),
  BlackListedResource(new URL("https://maps.google.com"), Set("John Doe")),
  BlackListedResource(new URL("http://plus.google.com"), Set.empty)
)
```

`BlackListedResource` 表示黑名单里的网站 URL，外加试图访问这个网址的公民集合。

现在我们想处理这个黑名单，为了标识 “有问题” 的公民，比如说那些试图访问被屏蔽网站的人。
同时，我们想确定可疑的 Web 网站：如果没有一个公民试图去访问黑名单里的某一个网站，
那么就必须假定目标对象因为一些我们不知道的原因绕过了筛选器，需要对此进行调查。

下面的代码展示了该如何处理黑名单的：

``` scala
al checkedBlacklist: List[Either[URL, Set[Citizen]]] =
  blacklist.map(resource =>
    if (resource.visitors.isEmpty) Left(resource.url)
    else Right(resource.visitors))
```

我们创建了一个 Either 序列，其中 `Left` 实例代表可疑的 URL， `Right` 是问题市民的集合。
识别问题公民和可疑网站变得非常简单。

``` scala
val suspiciousResources = checkedBlacklist.flatMap(_.left.toOption)
val problemCitizens = checkedBlacklist.flatMap(_.right.toOption).flatten.toSet
```

`Either` 非常适用于这种比异常处理更为普通的使用场景。

### 总结

目前为止，你应该已经学会了怎么使用 Either，认识到它的缺陷，以及知道该在什么时候用它。
鉴于 Either 的缺陷，使用不使用它，全都取决于你。
其实在实践中，你会注意到，有了 Try 之后，Either 不会出现那么多糟糕的使用情形。

不管怎样，分清楚它带来的利与弊总没有坏处。
