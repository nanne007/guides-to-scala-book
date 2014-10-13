## 类型 Future

作为一个对 Scala 充满热情的开发者，你应该已经听说过 Scala 处理并发的能力，或许你就是被这个吸引来的。
相较于大多数编程语言低级的并发 API，Scala 提供的方法可以让人们更好的理解并发以及编写良构的并发程序。

本章的主题- Future 就是这种方法的两大基石之一。（另一个是 actor）
我会解释 Future 的优点，以及它的函数式特征。

如果你想动手试试接下来的例子，请确保 Scala 版本不低于 2.9.3，
Future 在 2.10.0 版本中引入，并向后兼容到 2.9.3，最初，它是 Akka 库的一部分（API略有不同）。

### 顺序代码为什么会变坏

假设你想准备一杯卡布奇诺，你可以一个接一个的执行以下步骤：

1. 研磨所需的咖啡豆
2. 加热一些水
3. 用研磨好的咖啡豆和热水制做一杯咖啡
4. 打奶泡
5. 结合咖啡和奶泡做成卡布奇诺

转换成 Scala 代码，可能会是这样：

``` scala
import scala.util.Try
// Some type aliases, just for getting more meaningful method signatures:
type CoffeeBeans = String
type GroundCoffee = String
case class Water(temperature: Int)
type Milk = String
type FrothedMilk = String
type Espresso = String
type Cappuccino = String

// dummy implementations of the individual steps:
def grind(beans: CoffeeBeans): GroundCoffee = s"ground coffee of $beans"
def heatWater(water: Water): Water ` water.copy(temperature `  85)
def frothMilk(milk: Milk): FrothedMilk = s"frothed $milk"
def brew(coffee: GroundCoffee, heatedWater: Water): Espresso = "espresso"
def combine(espresso: Espresso, frothedMilk: FrothedMilk): Cappuccino = "cappuccino"

// some exceptions for things that might go wrong in the individual steps
// (we'll need some of them later, use the others when experimenting with the code):
case class GrindingException(msg: String) extends Exception(msg)
case class FrothingException(msg: String) extends Exception(msg)
case class WaterBoilingException(msg: String) extends Exception(msg)
case class BrewingException(msg: String) extends Exception(msg)

// going through these steps sequentially:
def prepareCappuccino(): Try[Cappuccino] = for {
  ground <- Try(grind("arabica beans"))
  water <- Try(heatWater(Water(25)))
  espresso <- Try(brew(ground, water))
  foam <- Try(frothMilk("milk"))
} yield combine(espresso, foam)
```

这样做有几个优点：
可以很轻易的弄清楚事情的步骤，一目了然，而且不会混淆。（毕竟没有上下文切换）
不好的一面是，大部分时间，你的大脑和身体都处于等待的状态：
在等待研磨咖啡豆时，你完全不能做任何事情，只有当这一步完成后，你才能开始烧水。
这显然是在浪费时间，所以你可能想一次开始多个步骤，让它们同时执行，
一旦水烧开，咖啡豆也磨好了，你可以制做咖啡了，这期间，打奶泡也可以开始了。

这和编写软件没什么不同。
一个 Web 服务器可以用来处理和响应请求的线程只有那么多，
不能因为要等待数据库查询或其他 HTTP 服务调用的结果而阻塞了这些可贵的线程。
相反，一个异步编程模型和非阻塞 IO 会更合适，
这样的话，当一个请求处理在等待数据库查询结果时，处理这个请求的线程也能够为其他请求服务。


> "I heard you like callbacks, so I put a callback in your callback!"

在并发家族里，你应该已经知道 nodejs 这个很酷的家伙，nodejs 完全通过回调来通信，
不幸的是，这很容易导致回调中包含回调的回调，这简直是一团糟，代码难以阅读和调试。

Scala 的 Future 也允许回调，但它提供了更好的选择，所以你不怎么需要它。

> "I know Futures, and they are completely useless!"

也许你知道些其他的 Future 实现，最引人注目的是 Java 提供的那个。
但是对于 Java 的 Future，你只能去查看它是否已经完成，或者阻塞线程直到其结束。
简而言之，Java 的 Future 几乎没有用，而且用起来绝对不会让人开心。

如果你认为 Scala 的 Future 也是这样，那大错特错了！

### Future 语义

*scala.concurrent*  包里的 `Future[T]`  是一个容器类型，代表一种返回值类型为 `T`  的计算。
计算可能会出错，也可能会超时；从而，当一个 future 完成时，它可能会包含异常，而不是你期望的那个值。

Future 只能写一次： 当一个 future 完成后，它就不能再被改变了。
同时，Future 只提供了读取计算值的接口，写入计算值的任务交给了 Promise，这样，API 层面上会有一个清晰的界限。
这篇文章里，我们主要关注前者，下一章会介绍 Promise 的使用。

### 使用 Future

Future 有多种使用方式，我将通过重写 “卡布奇诺” 这个例子来说明。

首先，所有可以并行执行的函数，应该返回一个 Future：

```  scala
import scala.concurrent.future
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.util.Random

def grind(beans: CoffeeBeans): Future[GroundCoffee] = Future {
  println("start grinding...")
  Thread.sleep(Random.nextInt(2000))
  if (beans == "baked beans") throw GrindingException("are you joking?")
  println("finished grinding...")
  s"ground coffee of $beans"
}

def heatWater(water: Water): Future[Water] = Future {
  println("heating the water now")
  Thread.sleep(Random.nextInt(2000))
  println("hot, it's hot!")
  water.copy(temperature = 85)
}

def frothMilk(milk: Milk): Future[FrothedMilk] = Future {
  println("milk frothing system engaged!")
  Thread.sleep(Random.nextInt(2000))
  println("shutting down milk frothing system")
  s"frothed $milk"
}

def brew(coffee: GroundCoffee, heatedWater: Water): Future[Espresso] = Future {
  println("happy brewing :)")
  Thread.sleep(Random.nextInt(2000))
  println("it's brewed!")
  "espresso"
}
```

上面的代码有几处需要解释。

首先是 Future 伴生对象里的 `apply`  方法需要两个参数：

```  scala
object Future {
  def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T]
}
```

要异步执行的计算通过传名参数 `body`  传入。
第二个参数是一个隐式参数，隐式参数是说，函数调用时，如果作用域中存在一个匹配的隐式值，就无需显示指定这个参数。
`ExecutionContext` 可以执行一个 Future，可以把它看作是一个线程池，是绝大部分 Future API 的隐式参数。

`import scala.concurrent.ExecutionContext.Implicits.global`  语句引入了一个全局的执行上下文，确保了隐式值的存在。
这时候，只需要一个单元素列表,可以用大括号来代替小括号。
调用 `future`  方法时，经常使用这种形式，使得它看起来像是一种语言特性，而不是一个普通方法的调用。

这个例子没有大量计算，所以用随机休眠来模拟以说明问题，
而且，为了更清晰的说明并发代码的执行顺序，还在“计算”之前和之后打印了些东西。

计算会在 Future 创建后的某个不确定时间点上由 `ExecutionContext`  给其分配的某个线程中执行。

#### 回调

对于一些简单的问题，使用回调就能很好解决。
Future 的回调是偏函数，你可以把回调传递给 Future 的 `onSuccess`  方法，
如果这个 Future 成功完成，这个回调就会执行，并把 Future 的返回值作为参数输入：

```  scala
 grind("arabica beans").onSuccess { case ground =>
   println("okay, got my ground coffee")
 }
```

类似的，也可以在 `onFailure`  上注册回调，只不过它是在 Future 失败时调用，其输入是一个 `Throwable`。

通常的做法是将两个回调结合在一起以更好的处理 Future：在 `onComplete`  方法上注册回调，回调的输入是一个 Try。

```  scala
 import scala.util.{Success, Failure}
 grind("baked beans").onComplete {
   case Success(ground) => println(s"got my $ground")
   case Failure(ex) => println("This grinder needs a replacement, seriously!")
 }
```

传递给 `grind`  的是 “baked beans”，因此 `grind`  方法会产生异常，进而导致 Future 中的计算失败。

#### Future 组合

当嵌套使用 Future 时，回调就变得比较烦人。
不过，你也没必要这么做，因为 Future 是可组合的，这是它真正发挥威力的时候！

你一定已经注意到，之前讨论过的所有容器类型都可以进行 `map`  、 `flatMap`  操作，也可以用在 for 语句中。
作为一种容器类型，Future 支持这些操作也不足为奇！

真正的问题是，在还没有完成的计算上执行这些操作意味这什么，如何去理解它们？

#### Map 操作

Scala 让 “时间旅行” 成为可能！
假设想在水加热后就去检查它的温度，
可以通过将 `Future[Water]`  映射到 `Future[Boolean]`  来完成这件事情：

``` scala
 val tempreatureOkay: Future[Boolean] = heatWater(Water(25)) map { water =>
   println("we're in the future!")
   (80 to 85) contains (water.temperature)
 }
```

`tempreatureOkay`  最终会包含水温的结果。
你可以去改变 `heatWater`  的实现来让它抛出异常（比如说，加热器爆炸了），
然后等待 “we're in the future!” 出现在显示屏上，不过你永远等不到。

写传递给 `map`  的函数时，你就处在未来（或者说可能的未来）。
一旦 `Future[Water]`  实例成功完成，这个函数就会执行，只不过，该函数所在的时间线可能不是你现在所处的这个。
如果 `Future[Water`  失败，传递给 `map`  的函数中的事情永远不会发生，调用 `map`  的结果将是一个失败的 `Future[Boolean]`。

#### FlatMap 操作

如果一个 Future 的计算依赖于另一个 Future 的结果，那需要求救于 `flatMap`  以避免 Future 的嵌套。

假设，测量水温的线程需要一些时间，那你可能想异步的去检查水温是否 OK。
比如，有一个函数，接受一个 `Water`  ，并返回 `Future[Boolean]`  ：

```  scala
def temperatureOkay(water: Water): Future[Boolean] = future {
  (80 to 85) contains (water.temperature)
｝
```

使用 `flatMap`（而不是 `map`）得到一个 `Future[Boolean]`，而不是 `Future[Future[Boolean]]`：

```  scala
val nestedFuture: Future[Future[Boolean]] = heatWater(Water(25)) map {
  water => temperatureOkay(water)
}

val flatFuture: Future[Boolean] = heatWater(Water(25)) flatMap {
  water => temperatureOkay(water)
}
```

同样，映射只会发生在 `Future[Water]`  成功完成情况下。

#### for 语句

除了调用 `flatMap`  ，也可以写成 for 语句。上面的例子可以重写成：

```  scala
val acceptable: Future[Boolean] = for {
  heatedWater <- heatWater(Water(25))
  okay <- temperatureOkay(heatedWater)
} yield okay
```

如果有多个可以并行执行的计算，则需要特别注意，要先在 for 语句外面创建好对应的 Futures。

``` scala
def prepareCappuccinoSequentially(): Future[Cappuccino] =
  for {
    ground <- grind("arabica beans")
    water <- heatWater(Water(25))
    foam <- frothMilk("milk")
    espresso <- brew(ground, water)
  } yield combine(espresso, foam)
```

这看起来很漂亮，但要知道，for 语句只不过是 `flatMap` 嵌套调用的语法糖。
这意味着，只有当 `Future[GroundCoffee]`  成功完成后， `heatWater`  才会创建 `Future[Water]`。
你可以查看函数运行时打印出来的东西来验证这个说法。

因此，要确保在 for 语句之前实例化所有相互独立的 Futures：

```  scala
def prepareCappuccino(): Future[Cappuccino] = {
  val groundCoffee = grind("arabica beans")
  val heatedWater = heatWater(Water(20))
  val frothedMilk = frothMilk("milk")
  for {
    ground <- groundCoffee
    water <- heatedWater
    foam <- frothedMilk
    espresso <- brew(ground, water)
  } yield combine(espresso, foam)
}
```

在 for 语句之前，三个 Future 在创建之后就开始各自独立的运行，显示屏的输出是不确定的。
唯一能确定的是 “happy brewing” 总是出现在后面，
因为该输出所在的函数 `brew` 是在其他两个函数执行完毕后才开始执行的。
也因为此，可以在 for 语句里面直接调用它，当然，前提是前面的 Future 都成功完成。

#### 失败偏向的 Future

你可能会发现 `Future[T]`  是成功偏向的，允许你使用 `map`、`flatMap`、`filter` 等。

但是，有时候可能处理事情出错的情况。
调用 `Future[T]`  上的 `failed`  方法，会得到一个失败偏向的 Future，类型是 `Future[Throwable]`。
之后就可以映射这个 `Future[Throwable]`，在失败的情况下执行 mapping 函数。

### 总结

你已经见过 Future 了，而且它的前途看起来很光明！
因为它是一个可组合、可函数式使用的容器类型，这让我们的工作变得异常舒服。

调用 `future`  方法可以轻易将阻塞执行的代码变成并发执行，但是，代码最好原本就是非阻塞的。
为了实现它，我们还需要 `Promise`  来完成 `Future`，这就是下一章的主题。
