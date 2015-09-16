## 类型 Option

前几章，我们讨论了许多相当先进的技术，尤其是模式匹配和提取器。
是时候来看一看 Scala 另一个基本特性了： Option 类型。

可能你已经见过它在 `Map` API 中的使用；在实现自己的提取器时，我们也用过它，
然而，它还需要更多的解释。
你可能会想知道它到底解决什么问题，为什么用它来处理缺失值要比其他方法好，
而且可能你还不知道该怎么在你的代码中使用它。
这一章的目的就是消除这些问号，并教授你作为一个新手所应该了解的 `Option` 知识。

### 基本概念

Java 开发者一般都知道 `NullPointerException`（其他语言也有类似的东西），
通常这是由于某个方法返回了 `null` ，但这并不是开发者所希望发生的，代码也不好去处理这种异常。

值 `null` 通常被滥用来表征一个可能会缺失的值。
不过，某些语言以一种特殊的方法对待 `null` 值，或者允许你安全的使用可能是 `null` 的值。
比如说，Groovy 有 *安全运算符(Safe Navigation Operator)* 用于访问属性，
这样 `foo?.bar?.baz` 不会在 `foo` 或 `bar` 是 `null` 时而引发异常，而是直接返回 `null`，
然而，Groovy 中没有什么机制来强制你使用此运算符，所以如果你忘记使用它，那就完蛋了！

Clojure 对待 `nil` 基本上就像对待空字符串一样。
也可以把它当作列表或者映射表一样去访问，这意味着， `nil` 在调用层级中向上冒泡。
很多时候这样是可行的，但有时会导致异常出现在更高的调用层级中，而那里的代码没有对 `nil` 加以考虑。

Scala 试图通过摆脱 `null` 来解决这个问题，并提供自己的类型用来表示一个值是可选的（有值或无值），
这就是 `Option[A]` 特质。

`Option[A]` 是一个类型为 `A` 的可选值的容器：
如果值存在， `Option[A]` 就是一个 `Some[A]` ，如果不存在， `Option[A]` 就是对象 `None` 。

在类型层面上指出一个值是否存在，使用你的代码的开发者（也包括你自己）就会被编译器强制去处理这种可能性，
而不能依赖值存在的偶然性。

`Option` 是强制的！不要使用 `null` 来表示一个值是缺失的。

### 创建 Option

通常，你可以直接实例化 `Some` 样例类来创建一个 Option 。

``` scala
val greeting: Option[String] = Some("Hello world")
```

或者，在知道值缺失的情况下，直接使用 `None` 对象：

``` scala
val greeting: Option[String] = None
```

然而，在实际工作中，你不可避免的要去操作一些 Java 库，
或者是其他将 `null` 作为缺失值的JVM 语言的代码。
为此， `Option` 伴生对象提供了一个工厂方法，可以根据给定的参数创建相应的 `Option` ：

``` scala
val absentGreeting: Option[String] = Option(null) // absentGreeting will be None
val presentGreeting: Option[String] = Option("Hello!") // presentGreeting will be Some("Hello!")
```

### 使用 Option

目前为止，所有的这些都很简洁，不过该怎么使用 Option 呢？是时候开始举些无聊的例子了。

想象一下，你正在为某个创业公司工作，要做的第一件事情就是实现一个用户的存储库，
要求能够通过唯一的用户 ID 来查找他们。
有时候请求会带来假的 ID，这种情况，查找方法就需要返回 `Option[User]` 类型的数据。
一个假想的实现可能是：

``` scala
  case class User(
    id: Int,
    firstName: String,
    lastName: String,
    age: Int,
    gender: Option[String]
  )

  object UserRepository {
    private val users = Map(1 -> User(1, "John", "Doe", 32, Some("male")),
                            2 -> User(2, "Johanna", "Doe", 30, None))
    def findById(id: Int): Option[User] = users.get(id)
    def findAll = users.values
  }
```

现在，假设从 `UserRepository` 接收到一个 `Option[User]` 实例，并需要拿它做点什么，该怎么办呢？

一个办法就是通过 `isDefined` 方法来检查它是否有值。
如果有，你就可以用 `get` 方法来获取该值：

``` scala
  val user1 = UserRepository.findById(1)
  if (user1.isDefined) {
    println(user1.get.firstName)
  } // will print "John"
```

这和 [Guava 库](https://code.google.com/p/guava-libraries) 中的 `Optional` 使用方法类似。
不过这种使用方式太过笨重，更重要的是，使用 `get` 之前，
你可能会忘记用 `isDefined` 做检查，这会导致运行期出现异常。
这样一来，相对于 `null` ，使用 `Option` 并没有什么优势。

你应该尽可能远离这种访问方式！

### 提供一个默认值

很多时候，在值不存在时，需要进行回退，或者提供一个默认值。
Scala 为 `Option` 提供了 `getOrElse` 方法，以应对这种情况：

``` scala
  val user = User(2, "Johanna", "Doe", 30, None)
  println("Gender: " + user.gender.getOrElse("not specified")) // will print "not specified"
```

请注意，作为 `getOrElse` 参数的默认值是一个 *传名参数* ，
这意味着，只有当这个 `Option` 确实是 `None` 时，传名参数才会被求值。
因此，没必要担心创建默认值的代价，它只有在需要时才会发生。

### 模式匹配

`Some` 是一个样例类，可以出现在模式匹配表达式或者其他允许模式出现的地方。
上面的例子可以用模式匹配来重写：

``` scala
  val user = User(2, "Johanna", "Doe", 30, None)
  user.gender match {
    case Some(gender) => println("Gender: " + gender)
    case None => println("Gender: not specified")
  }
```

或者，你想删除重复的 `println` 语句，并重点突出模式匹配表达式的使用：

``` scala
  val user = User(2, "Johanna", "Doe", 30, None)
  val gender = user.gender match {
    case Some(gender) => gender
    case None => "not specified"
  }
  println("Gender: " + gender)
```

你可能已经发现用模式匹配处理 `Option` 实例是非常啰嗦的，这也是它非惯用法的原因。
所以，即使你很喜欢模式匹配，也尽量用其他方法吧。

不过在 Option 上使用模式确实是有一个相当优雅的方式，
在下面的 for 语句一节中，你就会学到。

### 作为集合的 Option

到目前为止，你还没有看见过优雅使用 Option 的方式吧。下面这个就是了。

前文我提到过， `Option` 是类型 `A` 的容器，更确切地说，你可以把它看作是某种集合，
这个特殊的集合要么只包含一个元素，要么就什么元素都没有。


虽然在类型层次上， `Option` 并不是 Scala 的集合类型，
但，凡是你觉得 Scala 集合好用的方法， `Option` 也有，
你甚至可以将其转换成一个集合，比如说 `List` 。

那么这又能让你做什么呢？

#### 执行一个副作用

如果想在 Option 值存在的时候执行某个副作用，`foreach` 方法就派上用场了：

``` scala
 UserRepository.findById(2).foreach(user => println(user.firstName)) // prints "Johanna"
```

如果这个 Option 是一个 `Some` ，传递给 `foreach` 的函数就会被调用一次，且只有一次；
如果是 `None` ，那它就不会被调用。

#### 执行映射

`Option` 表现的像集合，最棒的一点是，
你可以用它来进行函数式编程，就像处理列表、集合那样。

正如你可以将 `List[A]` 映射到 `List[B]` 一样，你也可以映射 `Option[A]` 到 `Option[B]`：
如果 `Option[A]` 实例是 `Some[A]` 类型，那映射结果就是 `Some[B]` 类型；否则，就是 `None` 。

如果将 `Option` 和 `List` 做对比 ，那 `None` 就相当于一个空列表：
当你映射一个空的 `List[A]` ，会得到一个空的 `List[B]` ，
而映射一个是 `None` 的 `Option[A]` 时，得到的 `Option[B]` 也是 `None` 。

让我们得到一个可能不存在的用户的年龄：

``` scala
val age = UserRepository.findById(1).map(_.age) // age is Some(32)
```

#### Option 与 flatMap

也可以在 `gender` 上做 `map` 操作：

``` scala
val gender = UserRepository.findById(1).map(_.gender) // gender is an Option[Option[String]]
```

所生成的 `gender` 类型是 `Option[Option[String]]` 。这是为什么呢？

这样想：你有一个装有 `User` 的 `Option` 容器，在容器里面，你将 `User` 映射到 `Option[String]`
（ `User` 类上的属性 `gender` 是 `Option[String]` 类型的）。
得到的必然是嵌套的 Option。

既然可以 `flatMap` 一个 `List[List[A]]` 到 `List[B]` ，
也可以 `flatMap` 一个 `Option[Option[A]]` 到 `Option[B]` ，这没有任何问题：
Option 提供了 `flatMap` 方法。

``` scala
val gender1 = UserRepository.findById(1).flatMap(_.gender) // gender is Some("male")
val gender2 = UserRepository.findById(2).flatMap(_.gender) // gender is None
val gender3 = UserRepository.findById(3).flatMap(_.gender) // gender is None
```

现在结果就变成了 `Option[String]` 类型，
如果 `user` 和 `gender` 都有值，那结果就会是 `Some` 类型，反之，就得到一个 `None` 。

要理解这是什么原理，让我们看看当 `flatMap` 一个 `List[List[A]]` 时，会发生什么？
（要记得， Option 就像一个集合，比如列表）

``` scala
val names: List[List[String]] =
 List(List("John", "Johanna", "Daniel"), List(), List("Doe", "Westheide"))
names.map(_.map(_.toUpperCase))
// results in List(List("JOHN", "JOHANNA", "DANIEL"), List(), List("DOE", "WESTHEIDE"))
names.flatMap(_.map(_.toUpperCase))
// results in List("JOHN", "JOHANNA", "DANIEL", "DOE", "WESTHEIDE")
```

如果我们使用 `flatMap` ，内部列表中的所有元素会被转换成一个扁平的字符串列表。
显然，如果内部列表是空的，则不会有任何东西留下。

现在回到 `Option` 类型，如果映射一个由 `Option` 组成的列表呢？

``` scala
val names: List[Option[String]] = List(Some("Johanna"), None, Some("Daniel"))
names.map(_.map(_.toUpperCase)) // List(Some("JOHANNA"), None, Some("DANIEL"))
names.flatMap(xs => xs.map(_.toUpperCase)) // List("JOHANNA", "DANIEL")
```

如果只是 `map` ，那结果类型还是 `List[Option[String]]` 。
而使用 `flatMap` 时，内部集合的元素就会被放到一个扁平的列表里：
任何一个 `Some[String]` 里的元素都会被解包，放入结果集中；
而原列表中的 `None` 值由于不包含任何元素，就直接被过滤出去了。

记住这一点，然后再去看看 `faltMap` 在 `Option` 身上做了什么。

#### 过滤 Option

也可以像过滤列表那样过滤 Option：
如果选项包含有值，而且传递给 `filter` 的谓词函数返回真， `filter` 会返回 `Some` 实例。
否则（即选项没有值，或者谓词函数返回假值），返回值为 `None` 。

``` scala
UserRepository.findById(1).filter(_.age > 30) // None, because age is <= 30
UserRepository.findById(2).filter(_.age > 30) // Some(user), because age is > 30
UserRepository.findById(3).filter(_.age > 30) // None, because user is already None
```

### for 语句

现在，你已经知道 Option 可以被当作集合来看待，并且有 `map` 、 `flatMap` 、 `filter` 这样的方法。
可能你也在想 Option 是否能够用在 for 语句中，答案是肯定的。
而且，用 for 语句来处理 Option 是可读性最好的方式，尤其是当你有多个 `map` 、`flatMap` 、`filter` 调用的时候。
如果只是一个简单的 `map` 调用，那 for 语句可能有点繁琐。

假如我们想得到一个用户的性别，可以这样使用 for 语句：

``` scala
for {
  user <- UserRepository.findById(1)
  gender <- user.gender
} yield gender // results in Some("male")
```

可能你已经知道，这样的 for 语句等同于嵌套的 `flatMap` 调用。
如果 `UserRepository.findById` 返回 `None`，或者 `gender` 是 `None` ，
那这个 for 语句的结果就是 `None` 。
不过这个例子里， `gender` 含有值，所以返回结果是 `Some` 类型的。

如果我们想返回所有用户的性别（当然，如果用户设置了性别），可以遍历用户，yield 其性别：

``` scala
for {
  user <- UserRepository.findAll
  gender <- user.gender
} yield gender
// result in List("male")
```

#### 在生成器左侧使用

也许你还记得，前一章曾经提到过， for 语句中生成器的左侧也是一个模式。
这意味着也可以在 for 语句中使用包含选项的模式。

重写之前的例子：

``` scala
 for {
   User(_, _, _, _, Some(gender)) <- UserRepository.findAll
 } yield gender
```

在生成器左侧使用 `Some` 模式就可以在结果集中排除掉值为 `None` 的元素。

### 链接 Option

Option 还可以被链接使用，这有点像偏函数的链接：
在 Option 实例上调用 `orElse` 方法，并将另一个 Option 实例作为传名参数传递给它。
如果一个 Option 是 `None` ， `orElse` 方法会返回传名参数的值，否则，就直接返回这个 Option。

一个很好的使用案例是资源查找：对多个不同的地方按优先级进行搜索。
下面的例子中，我们首先搜索 *config* 文件夹，并调用 `orElse` 方法，以传递备用目录：

``` scala
case class Resource(content: String)
val resourceFromConfigDir: Option[Resource] = None
val resourceFromClasspath: Option[Resource] = Some(Resource("I was found on the classpath"))
val resource = resourceFromConfigDir orElse resourceFromClasspath
```

如果想链接多个选项，而不仅仅是两个，使用 `orElse` 会非常合适。
不过，如果只是想在值缺失的情况下提供一个默认值，那还是使用 `getOrElse` 吧。

### 总结

在这一章里，你学到了有关 Option 的所有知识，
这有利于你理解别人的代码，也有利于你写出更可读，更函数式的代码。

这一章最重要的一点是：列表、集合、映射、Option，以及之后你会见到的其他数据类型，
它们都有一个非常统一的使用方式，这种使用方式既强大又优雅。

下一章，你将学习 Scala 错误处理的惯用法。
