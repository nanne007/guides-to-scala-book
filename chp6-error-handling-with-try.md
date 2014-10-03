## Try 与错误处理

当你在尝试一门新的语言时，可能不会过于关注程序出错的问题，
但当真的去创造可用的代码时，就不能再忽视代码中的可能产生的错误和异常了。
鉴于各种各样的原因，人们往往低估了语言对错误处理支持程度的重要性。

事实会表明，Scala 能够很优雅的处理此类问题，
这一部分，我会介绍 Scala 基于 Try 的错误处理机制，以及这背后的原因。
我将使用一个在 *Scala 2.10* 新引入的特性，该特性向 *2.9.3* 兼容，
因此，请确保你的 Scala 版本不低于 *2.9.3*。

### 异常的抛出和捕获

在介绍 Scala 错误处理的惯用法之前，我们先看看其他语言（如，Java，Ruby）的错误处理机制。
和这些语言类似，Scala 也允许你抛出异常：

``` scala
case class Customer(age: Int)
class Cigarettes
case class UnderAgeException(message: String) extends Exception(message)
def buyCigarettes(customer: Customer): Cigarettes =
  if (customer.age < 16)
    throw UnderAgeException(s"Customer must be older than 16 but was ${customer.age}")
  else new Cigarettes
```

被抛出的异常能够以类似 Java 中的方式被捕获，虽然是使用偏函数来指定要处理的异常类型。
此外，Scala 的 `try/catch` 是表达式（返回一个值），因此下面的代码会返回异常的消息：

``` scala
val youngCustomer = Customer(15)
try {
  buyCigarettes(youngCustomer)
  "Yo, here are your cancer sticks! Happy smokin'!"
} catch {
    case UnderAgeException(msg) => msg
}
```

### 函数式的错误处理

现在，如果代码中到处是上面的异常处理代码，那它很快就会变得丑陋无比，和函数式程序设计非常不搭。
对于高并发应用来说，这也是一个很差劲的解决方式，比如，
假设需要处理在其他线程执行的 actor 所引发的异常，显然你不能用捕获异常这种处理方式，
你可能会想到其他解决方案，例如去接收一个表示错误情况的消息。

一般来说，在 Scala 中，好的做法是通过从函数里返回一个合适的值来通知人们程序出错了。
别担心，我们不会回到 C 中那种需要使用按约定进行检查的错误编码的错误处理。
相反，Scala 使用一个特定的类型来表示可能会导致异常的计算，这个类型就是 Try。

#### Try 的语义

解释 Try 最好的方式是将它与上一章所讲的 Option 作对比。

`Option[A]` 是一个可能有值也可能没值的容器，
`Try[A]` 则表示一种计算：
这种计算在成功的情况下，返回类型为 `A` 的值，在出错的情况下，返回 `Throwable` 。
这种可以容纳错误的容器可以很轻易的在并发执行的程序之间传递。

Try 有两个子类型：

1. `Success[A]`：代表成功的计算。
2. 封装了 `Throwable` 的 `Failure[A]`：代表出了错的计算。


如果知道一个计算可能导致错误，我们可以简单的使用 `Try[A]` 作为函数的返回类型。
这使得出错的可能性变得很明确，而且强制客户端以某种方式处理出错的可能。

假设，需要实现一个简单的网页爬取器：用户能够输入想爬取的网页 URL，
程序就需要去分析 URL 输入，并从中创建一个 `java.net.URL` ：

``` scala
import scala.util.Try
import java.net.URL
def parseURL(url: String): Try[URL] = Try(new URL(url))
```

正如你所看到的，函数返回类型为 `Try[URL]`：
如果给定的 url 语法正确，这将是 `Success[URL]`，
否则， `URL` 构造器会引发 `MalformedURLException` ，从而返回值变成 `Failure[URL]` 类型。

上例中，我们还用了 Try 伴生对象里的 `apply` 工厂方法，这个方法接受一个类型为 `A` 的 *传名参数*，
这意味着， `new URL(url)` 是在 `Try` 的 `apply` 方法里执行的。

`apply` 方法不会捕获任何非致命的异常，仅仅返回一个包含相关异常的 Failure 实例。

因此， `parseURL("http://danielwestheide.com")` 会返回一个 `Success[URL]` ，包含了解析后的网址，
而 `parseULR("garbage")` 将返回一个含有 `MalformedURLException` 的 `Failure[URL]`。

#### 使用 Try

使用 Try 与使用 Option 非常相似，在这里你看不到太多新的东西。

你可以调用 `isSuccess` 方法来检查一个 Try 是否成功，然后通过 `get` 方法获取它的值，
但是，这种方式的使用并不多见，因为你可以用 `getOrElse` 方法给 Try 提供一个默认值：

``` scala
val url = parseURL(Console.readLine("URL: ")) getOrElse new URL("http://duckduckgo.com")
```

如果用户提供的 URL 格式不正确，我们就使用 DuckDuckGo 的 URL 作为备用。

#### 链式操作

Try 最重要的特征是，它也支持高阶函数，就像 Option 一样。
在下面的示例中，你将看到，在 Try 上也进行链式操作，捕获可能发生的异常，而且代码可读性不错。

#### Mapping 和 Flat Mapping

将一个是 `Success[A]` 的 `Try[A]` 映射到 `Try[B]` 会得到 `Success[B]` 。
如果它是 `Failure[A]` ，就会得到 `Failure[B]` ，而且包含的异常和 `Failure[A]` 一样。

``` scala
parseURL("http://danielwestheide.com").map(_.getProtocol)
// results in Success("http")
parseURL("garbage").map(_.getProtocol)
// results in Failure(java.net.MalformedURLException: no protocol: garbage)
```

如果链接多个 `map` 操作，会产生嵌套的 Try 结构，这并不是我们想要的。
考虑下面这个返回输入流的方法：

``` scala
import java.io.InputStream
def inputStreamForURL(url: String): Try[Try[Try[InputStream]]] ` parseURL(url).map { u `>
 Try(u.openConnection()).map(conn => Try(conn.getInputStream))
}
```

由于每个传递给 `map` 的匿名函数都返回 Try，因此返回类型就变成了 `Try[Try[Try[InputStream]]]` 。

这时候， `flatMap` 就派上用场了。
`Try[A]` 上的 `flatMap` 方法接受一个映射函数，这个函数类型是 `(A) => Try[B]`。
如果我们的 `Try[A]` 已经是 `Failure[A]` 了，那么里面的异常就直接被封装成 `Failure[B]` 返回，
否则， `flatMap` 将 `Success[A]` 里面的值解包出来，并通过映射函数将其映射到 `Try[B]` 。

这意味着，我们可以通过链接任意个 `flatMap` 调用来创建一条操作管道，将值封装在 Success 里一层层的传递。

现在让我们用 `flatMap` 来重写先前的例子：

``` scala
def inputStreamForURL(url: String): Try[InputStream] =
 parseURL(url).flatMap { u =>
   Try(u.openConnection()).flatMap(conn => Try(conn.getInputStream))
 }
```

这样，我们就得到了一个 `Try[InputStream]`，
它可以是一个 Failure，包含了在 `flatMap` 过程中可能出现的异常；
也可以是一个 Success，包含了最后的结果。

#### 过滤器和 foreach

当然，你也可以对 Try 进行过滤，或者调用 `foreach` ，既然已经学过 Option，对于这两个方法也不会陌生。

当一个 Try 已经是 `Failure` 了，或者传递给它的谓词函数返回假值，`filter` 就返回 `Failure`
（如果是谓词函数返回假值，那 `Failure` 里包含的异常是 `NoSuchException` ），
否则的话， `filter` 就返回原本的那个 `Success` ，什么都不会变：

``` scala
def parseHttpURL(url: String) ` parseURL(url).filter(_.getProtocol `= "http")
parseHttpURL("http://apache.openmirror.de") // results in a Success[URL]
parseHttpURL("ftp://mirror.netcologne.de/apache.org") // results in a Failure[URL]
```

当一个 Try 是 `Success` 时， `foreach` 允许你在被包含的元素上执行副作用，
这种情况下，传递给 `foreach` 的函数只会执行一次，毕竟 Try 里面只有一个元素：

``` scala
 parseHttpURL("http://danielwestheide.com").foreach(println)
```


> 当 Try 是 Failure 时， `foreach` 不会执行，返回 `Unit` 类型。


### for 语句中的 Try

既然 Try 支持 `flatMap` 、 `map` 、 `filter` ，能够使用 for 语句也是理所当然的事情，
而且这种情况下的代码更可读。
为了证明这一点，我们来实现一个返回给定 URL 的网页内容的函数：

``` scala
import scala.io.Source
def getURLContent(url: String): Try[Iterator[String]] =
  for {
   url <- parseURL(url)
   connection <- Try(url.openConnection())
   is <- Try(connection.getInputStream)
   source = Source.fromInputStream(is)
  } yield source.getLines()
```

这个方法中，有三个可能会出错的地方，但都被 Try 给涵盖了。
第一个是我们已经实现的 `parseURL` 方法，
只有当它是一个 `Success[URL]` 时，我们才会尝试打开连接，从中创建一个新的 `InputStream` 。
如果这两步都成功了，我们就 `yield` 出网页内容，得到的结果是 `Try[Iterator[String]]` 。

当然，你可以使用 `Source#fromURL` 简化这个代码，并且，这个代码最后没有关闭输入流，
这都是为了保持例子的简单性，专注于要讲述的主题。

> 在这个例子中，`Source#fromURL`可以这样用：
> ``` scala
> import scala.io.Source
> def getURLContent(url: String): Try[Iterator[String]] =
>   for {
>     url <- parseURL(url)
>     source = Source.fromURL(url)
>   } yield source.getLines()
> ```
> 用 `is.close()` 可以关闭输入流。

#### 模式匹配

代码往往需要知道一个 Try 实例是 Success 还是 Failure，这时候，你应该想到模式匹配，
也幸好， `Success` 和 `Failure` 都是样例类。

接着上面的例子，如果网页内容能顺利提取到，我们就展示它，否则，打印一个错误信息：

``` scala
import scala.util.Success
import scala.util.Failure
getURLContent("http://danielwestheide.com/foobar") match {
  case Success(lines) => lines.foreach(println)
  case Failure(ex) => println(s"Problem rendering URL content: ${ex.getMessage}")
}
```

#### 从故障中恢复

如果想在失败的情况下执行某种动作，没必要去使用 `getOrElse`，
一个更好的选择是 `recover` ，它接受一个偏函数，并返回另一个 Try。
如果 `recover` 是在 Success 实例上调用的，那么就直接返回这个实例，否则就调用偏函数。
如果偏函数为给定的 `Failure` 定义了处理动作，
`recover` 会返回 `Success` ，里面包含偏函数运行得出的结果。

下面是应用了 `recover` 的代码：

``` scala
import java.net.MalformedURLException
import java.io.FileNotFoundException
val content = getURLContent("garbage") recover {
  case e: FileNotFoundException => Iterator("Requested page does not exist")
  case e: MalformedURLException => Iterator("Please make sure to enter a valid URL")
  case _ => Iterator("An unexpected error has occurred. We are so sorry!")
}
```

现在，我们可以在返回值 `content` 上安全的使用 `get` 方法了，因为它一定是一个 Success。
调用 `content.get.foreach(println)` 会打印 *Please make sure to enter a valid URL*。

### 总结

Scala 的错误处理和其他范式的编程语言有很大的不同。
Try 类型可以让你将可能会出错的计算封装在一个容器里，并优雅的去处理计算得到的值。
并且可以像操作集合和 Option 那样统一的去操作 Try。

Try 还有其他很多重要的方法，鉴于篇幅限制，这一章并没有全部列出，比如 `orElse` 方法，
`transform` 和 `recoverWith` 也都值得去看。

下一章，我们会探讨 Either，另外一种可以代表计算的类型，但它的可使用范围要比 Try 大的多。
