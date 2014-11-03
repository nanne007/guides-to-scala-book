## 实战中的 Promise 和 Future

上一章介绍了 Future 类型，以及如何用它来编写高可读性、高组合性的异步执行代码。

Future 只是整个谜团的一部分：
它是一个只读类型，允许你使用它计算得到的值，或者处理计算中出现的错误。
但是在这之前，必须得有一种方法把这个值放进去。
这一章里，你将会看到如何通过 Promise 类型来达到这个目的。

### 类型 Promise

之前，我们把一段顺序执行的代码块传递给了 `scala.concurrent` 里的 `future` 方法，
并且在作用域中给出了一个 `ExecutionContext`，它神奇地异步调用代码块，返回一个 Future 类型的结果。

虽然这种获得 Future 的方式很简单，但还有其他的方法来创建 Future 实例，并填充它，这就是 Promise。
Promise 允许你在 Future 里放入一个值，不过只能做一次，Future 一旦完成，就不能更改了。

一个 Future 实例总是和一个（也只能是一个）Promise 实例关联在一起。
如果你在 REPL 里调用 `future`  方法，你会发现返回的也是一个 Promise：

``` scala
import concurrent.Future
import concurrent.Future

scala> import concurrent.future
import concurrent.future

scala> import concurrent.ExecutionContext.Implicits.global
import concurrent.ExecutionContext.Implicits.global

scala> val f: Future[String] = future { "Hello World!" }
f: scala.concurrent.Future[String] = scala.concurrent.impl.Promise$DefaultPromise@2b509249
```

你得到的对象是一个 `DefaultPromise`  ，它实现了 `Future`  和 `Promise` 接口，
不过这就是具体的实现细节了（译注，有兴趣的读者可翻阅其实现的源码），
使用者只需要知道代码实现把 Future 和对应的 Promise 之间的联系分的很清晰。

这个小例子说明了：除了通过 Promise，没有其他方法可以完成一个 Future，
`future`  方法也只是一个辅助函数，隐藏了具体的实现机制。

现在，让我们动动手，看看怎样直接使用 Promise 类型。

#### 给出承诺

当我们谈论起承诺能否被兑现时，一个很熟知的例子是那些政客的竞选诺言。

假设被推选的政客给他的投票者一个减税的承诺。
这可以用 `Promise[TaxCut]`  表示：

``` scala
import concurrent.Promise
case class TaxCut(reduction: Int)
// either give the type as a type parameter to the factory method:
val taxcut = Promise[TaxCut]()
// or give the compiler a hint by specifying the type of your val:
val taxcut2: Promise[TaxCut] = Promise()
// taxcut: scala.concurrent.Promise[TaxCut] = scala.concurrent.impl.Promise$DefaultPromise@66ae2a84
// taxcut2: scala.concurrent.Promise[TaxCut] = scala.concurrent.impl.Promise$DefaultPromise@346974c6
```

一旦创建了这个 Promise，就可以在它上面调用 `future`  方法来获取承诺的未来：

``` scala
 val taxCutF: Future[TaxCut] = taxcut.future
 // `> scala.concurrent.Future[TaxCut] `  scala.concurrent.impl.Promise$DefaultPromise@66ae2a84
```

返回的 Future 可能并不和 Promise 一样，但在同一个 Promise 上调用 `future`  方法总是返回同一个对象，
以确保 Promise 和 Future 之间一对一的关系。

#### 结束承诺

一旦给出了承诺，并告诉全世界会在不远的将来兑现它，那最好尽力去实现。
在 Scala 中，可以结束一个 Promise，无论成功还是失败。

##### 兑现承诺

为了成功结束一个 Promise，你可以调用它的 `success`  方法，并传递一个大家期许的结果：

``` scala
  taxcut.success(TaxCut(20))
```

这样做之后，Promise 就无法再写入其他值了，如果偏要再写，会产生异常。

此时，和 Promise 关联的 Future 也成功完成，注册的回调会开始执行，
或者说对这个 Future 进行了映射，那这个时候，映射函数也该执行了。

一般来说，Promise 的完成和对返回的 Future 的处理发生在不同的线程。
很可能你创建了 Promise，并立即返回和它关联的 Future 给调用者，而实际上，另外一个线程还在计算它。

为了说明这一点，我们拿减税来举个例子：

``` scala
object Government {
  def redeemCampaignPledge(): Future[TaxCut] = {
    val p = Promise[TaxCut]()
    Future {
      println("Starting the new legislative period.")
      Thread.sleep(2000)
      p.success(TaxCut(20))
      println("We reduced the taxes! You must reelect us!!!!1111")
    }
    p.future
  }
}
```

这个例子中使用了 Future 伴生对象，不过不要被它搞混淆了，这个例子的重点是：Promise 并不是在调用者的线程里完成的。

现在我们来兑现当初的竞选宣言，在 Future 上添加一个 `onComplete`  回调：

``` scala
import scala.util.{Success, Failure}
val taxCutF: Future[TaxCut] = Government.redeemCampaignPledge()
println("Now that they're elected, let's see if they remember their promises...")
taxCutF.onComplete {
  case Success(TaxCut(reduction)) =>
    println(s"A miracle! They really cut our taxes by $reduction percentage points!")
  case Failure(ex) =>
    println(s"They broke their promises! Again! Because of a ${ex.getMessage}")
}
```

多次运行这个例子，会发现显示屏输出的结果顺序是不确定的，而且，最终回调函数会执行，进入成功的那个 case 。

##### 违背诺言

政客习惯违背诺言，Scala 程序员有时候也只能这样做。
调用 `failure`  方法，传递一个异常，结束 Promise：

``` scala
case class LameExcuse(msg: String) extends Exception(msg)
object Government {
  def redeemCampaignPledge(): Future[TaxCut] = {
     val p = Promise[TaxCut]()
     Future {
       println("Starting the new legislative period.")
       Thread.sleep(2000)
       p.failure(LameExcuse("global economy crisis"))
       println("We didn't fulfill our promises, but surely they'll understand.")
     }
     p.future
   }
}
```

这个 `redeemCampaignPledge`  实现最终会违背承诺。
一旦用 `failure`  结束这个 Promise，也无法再次写入了，正如 `success`  方法一样。
相关联的 Future 也会以 `Failure`  收场。

如果已经有了一个 Try，那可以直接把它传递给 Promise 的 `complete`  方法，以此来结束这个它。
如果这个 Try 是一个 Success，关联的 Future 会成功完成，否则，就失败。

### 基于 Future 的编程实践

如果想使用基于 Future 的编程范式以增加应用的扩展性，那应用从下到上都必须被设计成非阻塞模式。
这意味着，基本上应用层所有的函数都应该是异步的，并且返回 Future。

当下，一个可能的使用场景是开发 Web 应用。
流行的 Scala Web 框架，允许你将响应作为 `Future[Response]`  返回，而不是等到你完成响应再返回。
这个非常重要，因为它允许 Web 服务器用少量的线程处理更多的连接。
通过赋予服务器 `Future[Response]`  的能力，你可以最大化服务器线程池的利用率。

而且，应用的服务可能需要多次调用数据库层以及（或者）某些外部服务，
这时候可以获取多个 Future，用 for 语句将它们组合成新的 Future，简单可读！
最终，Web 层再将这样的一个 Future 变成 `Future[Response]`。

但是该怎样在实践中实现这些呢？需要考虑三种不同的场景：

#### 非阻塞IO

应用很可能涉及到大量的 IO 操作。比如，可能需要和数据库交互，还可能作为客户端去调用其他的 Web 服务。

如果是这样，可以使用一些基于 Java 非阻塞 IO 实现的库，也可以直接或通过 Netty 这样的库来使用 Java 的 NIO API。
这样的库可以用定量的线程池处理大量的连接。

但如果是想开发这样的一个库，直接和 Promise 打交道更为合适。

#### 阻塞 IO

有时候，并没有基于 NIO 的库可用。比如，Java 世界里大多数的数据库驱动都是使用阻塞 IO。
在 Web 应用中，如果用这样的驱动发起大量访问数据库的调用，要记得这些调用是发生在服务器线程里的。
为了避免这个问题，可以将所有需要和数据库交互的代码都放入 `future`  代码块里，就像这样：

``` scala
// get back a Future[ResultSet] or something similar:
Future {
  queryDB(query)
}
```

到现在为止，我们都是使用隐式可用的全局 `ExecutionContext`  来执行这些代码块。
通常，更好的方式是创建一个专用的 `ExecutionContext`  放在数据库层里。
可以从 Java的 `ExecutorService`  来它，这也意味着，可以异步的调整线程池来执行数据库调用，应用的其他部分不受影响。

``` scala
import java.util.concurrent.Executors
import concurrent.ExecutionContext
val executorService = Executors.newFixedThreadPool(4)
val executionContext = ExecutionContext.fromExecutorService(executorService)
```

#### 长时间运行的计算

取决于应用的本质特点，一个应用偶尔还会调用一些长时间运行的任务，它们完全不涉及 IO（CPU 密集的任务）。
这些任务也不应该在服务器线程中执行，因此需要将它们变成 Future：

``` scala
Future {
  longRunningComputation(data, moreData)
}
```

同样，最好有一些专属的 `ExecutionContext`  来处理这些 CPU 密集的计算。
怎样调整这些线程池大小取决于应用的特征，这些已经超过了本文的范围。

### 总结

这一章里，我们学习了 Promise - 基于 Future 的并发范式的可写组件，以及怎样用它来完成一个 Future；
同时，还给出了一些在实践中使用它们的建议。

下一章会讨论 Scala 函数式编程是如何增加代码可用性（一个长久以来和面向对象编程相关联的概念）的。
