## 无处不在的模式

前两章花费了相当多的时间去解释下面这两件事情：

1. 用模式解构对象是怎么一回事。
2. 如何构造自己的提取器。

现在是时候去了解模式更多的用法了。

### 模式匹配表达式

模式可能出现的一个地方就是 *模式匹配表达式(pattern matching expression)*：
一个表达式 `e` ，后面跟着关键字 `match` 以及一个代码块，这个代码块包含了一些匹配样例；
而样例又包含了 `case` 关键字、模式、可选的 *守卫分句(guard clause)* ，以及最右边的代码块；
如果模式匹配成功，这个代码块就会执行。
写成代码，看起来会是下面这种样子：

``` scala
e match {
  case Pattern1 => block1
  case Pattern2 if-clause => block2
  ...
}
```

下面是一个更具体的例子：

``` scala
case class Player(name: String, score: Int)
def printMessage(player: Player) = player match {
  case Player(_, score) if score > 100000 =>
    println("Get a job, dude!")
  case Player(name, _) =>
    println("Hey, $name, nice to see you again!")
}
```

`printMessage` 的返回值类型是 `Unit` ，其唯一目的是执行一个副作用，即打印一条信息。
要记住你不一定非要使用模式匹配，因为你也可以使用像 Java 语言中的 switch 语句。

但这里使用的模式匹配表达式之所以叫 *模式匹配表达式* 是有原因的：
其返回值是由第一个匹配的模式中的代码块决定的。

使用它通常是好的，因为它允许你解耦两个并不真正属于彼此的东西，也使得你的代码更易于测试。
可把上面的例子重写成下面这样：

``` scala
case class Player(name: String, score: Int)
def message(player: Player) = player match {
  case Player(_, score) if score > 100000 =>
    "Get a job, dude!"
  case Player(name, _) =>
    "Hey, $name, nice to see you again!"
}
def printMessage(player: Player) = println(message(player))
```

现在，独立出一个返回值是 `String` 类型的 `message` 函数，
它是一个纯函数，没有任何副作用，返回模式匹配表达式的结果，
你可以将其保存为值，或者赋值给一个变量。

### 值定义中的模式

模式还可能出现值定义的左边。
（以及变量定义，本书中变量的使用并不多，因为我偏向于使用函数式风格的Scala代码）

假设有一个方法，返回当前的球员，我们可以模拟这个方法，让它始终返回同一个球员：

``` scala
def currentPlayer(): Player = Player("Daniel", 3500)
```

通常的值定义如下所示：

``` scala
val player = currentPlayer()
doSomethingWithName(player.name)
```

如果你知道 Python，你可能会了解一个称为 *序列解包(sequence unpacking)* 的功能，
它允许在值定义（或者变量定义）的左侧使用模式。
你可以用类似的风格编写你的 Scala 代码：改变我们的代码，在将球员赋值给左侧变量的同时去解构它：

``` scala
val Player(name, _) = currentPlayer()
doSomethingWithName(name)
```

你可以用任何模式来做这件事情，但得确保模式总能够匹配，否则，代码会在运行时出错。
下面的代码就是有问题的： `scores` 方法返回球员得分的列表。
为了说明问题，代码中只是返回一个空的列表。

``` scala
def scores: List[Int] = List()
val best :: rest = scores
println("The score of our champion is " + best)
```

运行的时候，就会出现 `MatchError` 。（好像我们的游戏不是那么成功，毕竟没有任何得分）

一种安全且非常方便的使用方式是只解构那些在编译期就知道类型的样例类。
此外，以这种方式来使用元组，代码可读性会更强。
假设有一个函数，返回一个包含球员名字及其得分的元组，而不是先前定义的 `Player` ：

``` scala
def gameResult(): (String, Int) = ("Daniel", 3500)
```

访问元组字段的代码给人感觉总是很怪异：

``` scala
val result = gameResult()
println(result._1 + ": " + result._2)
```

这样，在赋值的同时去解构它是非常安全的，因为我们知道它类型是 `Tuple2` ：

``` scala
val (name, score) = gameResult()
println(name + ": " + score)
```

这就好看多了，不是吗？

### for 语句中的模式

模式在 for 语句中也非常重要。
所有能在值定义的左侧使用的模式都适用于 for 语句的值定义。
因此，如果我们有一个球员得分集，想确定谁能进名人堂（得分超过一定上限），
用 for 语句就可以解决：

``` scala
def gameResults(): Seq[(String, Int)] =
  ("Daniel", 3500) :: ("Melissa", 13000) :: ("John", 7000) :: Nil
def hallOfFame = for {
    result <- gameResults()
    (name, score) = result
    if (score > 5000)
  } yield name
```

结果是 `List("Melissa", "John")` ，因为第一个球员得分没超过 5000。

上面的代码还可以写的更简单，for 语句中，生成器的左侧也可以是模式。
从而，可以直接在左则把想要的值解构出来：

``` scala
def hallOfFame = for {
    (name, score) <- gameResults()
    if (score > 5000)
  } yield name
```

模式 `(name, score)` 总会匹配成功，
如果没有守卫语句 `if (score > 5000)` ，
for 语句就相当于直接将元组映射到球员名字，不会进行过滤。

不过你要知道，生成器左侧的模式也可以用来过滤。
如果左侧的模式匹配失败，那相关的元素就会被直接过滤掉。

为了说明这种情况，假设有一序列的序列，我们想返回所有非空序列的元素个数。
这就需要过滤掉所有的空列表，然后再返回剩下列表的元素个数。
下面是一个解决方案：

``` scala
val lists = List(1, 2, 3) :: List.empty :: List(5, 3) :: Nil
for {
  list @ head :: _ <- lists
} yield list.size
```

上面例子中，左侧的模式不匹配空列表。
这不会抛出 `MatchError` ，但对应的空列表会被丢掉，因此得到的结果是 `List(3, 2)` 。

模式和 for 语句是一个很自然、很强大的结合。
用 Scala 工作一段时间后，你会发现经常需要它。

### 小结

这一章讲述了模式的多种使用方式。
除此之外，模式还可以用于定义匿名函数，
如果你试过用 `catch` 块处理 Scala 中的异常，那你就见过模式的这个用法，
下一章会详细描述。
