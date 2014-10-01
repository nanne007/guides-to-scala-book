## 序列提取

上一章讲述了如何实现自定义的提取器以及如何在模式匹配中使用它们，
但是只讨论了如何从给定的数据结构中分解固定数目的参数。
对某种数据结构来说，Scala 提供了提取任意多个参数的模式匹配方法。

比如，你可以匹配只有两个、或者只有三个元素的列表：

``` scala
val xs = 3 :: 6 :: 12 :: Nil
xs match {
 case List(a, b) => a * b
 case List(a, b, c) => a + b + c
 case _ => 0
}
```

除此之外，也可以使用通配符 `_*` 匹配长度不确定的列表：

``` scala
val xs = 3 :: 6 :: 12 :: 24 :: Nil
xs match {
 case List(a, b, _*) => a * b
 case _ => 0
}
```

这个例子中，第一个模式成功匹配，把 `xs` 的前两个元素分别绑定到 `a` 、`b` ，
而剩余的列表，无论其还有多少个元素，都直接被忽略掉。

显然，这种模式的提取器是无法通过上一章介绍的方法来实现的。
需要一种特殊的方法，来使得一个提取器可以接受某一类型的对象，将其解构成列表，
且这个列表的长度在编译期是不确定的。

`unapplySeq` 就是用来做这件事情的，下面的代码是其可能的方法签名：

``` scala
 def unapplySeq(object: S): Option[Seq[T]]
```

这个方法接受类型 `S` 的对象，返回一个类型参数为 `Seq[T]` 的 `Option` 。

### 例子：提取给定的名字

现在我们举一个例子来展示如何使用这种提取器。

假设有一个应用，其某处代码接收了一个表示人名且类型为 `String` 的参数，
这个字符串可能包含了这个人的第二个甚至是第三个名字（如果这个人不止有一个名字）。
比如说， `Daniel` 、 `Catherina Johanna` 、 `Matthew John Michael` 。
而我们想做的是，从这个字符串中提取出单个的名字。

下面的代码是一个用 `unapplySeq` 方法实现的提取器：

``` scala
object GivenNames {
  def unapplySeq(name: String): Option[Seq[String]] = {
    val names = name.trim.split(" ")
    if (name.forall(_.isEmpty)) None
    else Some(names)
  }
}
```

给定一个含有一个或多个名字的字符串，这个提取器会将其解构成一个列表。
如果字符串不包含有任何名字，提取器会返回 `None` ，提取器所在的那个模式就匹配失败。

下面对提取器进行测试：

``` scala
  def greetWithFirstName(name: String) = name match {
    case GivenNames(firstName, _*) => "Good morning, $firstname!"
    case _ => "Welcome! Please make sure to fill in your name!"
  }
```

`greetWithFirstName("Daniel")` 会返回 "Good morning, Daniel!"，
而 `greetWithFirstName("Catherina Johanna")` 会返回 "Good morning, Catherina!"。

### 固定和可变的参数提取

有些时候，需要提取出至少多少个值，
这样，在编译期，就知道必须要提取出几个值出来，再外加一个可选的序列，用来保存不确定的那一部分。

在我们的例子中，假设输入的字符串包含了一个人完整的姓名，而不仅仅是名字。
比如字符串可能是"John Doe"、"Catherina Johanna Peterson"，其中，
"Doe"、"Peterson"是姓，"John"、"Catherina"、"Johanna"是名。
我们想做的是匹配这样的字符串，把姓绑定到第一个变量，
把第一个名字绑定到第二个变量，第三个变量存放剩下的任意个名字。

稍微修改 `unapplySeq` 方法就可以解决上述问题：

``` scala
def unapplySeq(object: S): Option[(T1, .., Tn-1, Seq[T])]
```

`unapplySeq` 返回的同样是 `Option[TupleN]` ，只不过，其最后一个元素是一个 `Seq[T]` 。
这个方法签名看起来应该很熟悉，它和之前的一个 `unapply` 签名类似。

下列代码是利用这个方法生成的提取器：

``` scala
object Names {
  def unapplySeq(name: String): Option[(String, String, Seq[String])] = {
    val names = name.trim.split(" ")
    if (names.size < 2) None
    else Some((names.last, names.head, names.drop(1).dropRight(1)))
  }
}
```

仔细看看其返回值，及其构造 `Some` 的方式。
代码返回一个类型参数为 `Tuple3` 的 `Option` ，这个元组包含了姓、名、以及由剩余的名字构成的序列。

如果这个提取器用在一个模式中，那只有当给定的字符串至少含有姓和名时，模式才匹配成功。

下面用这个提取器重写 `greeting` 方法：

``` scala
def greet(fullName: String) = fullName match {
  case Names(lastName, firstName, _*) =>
    "Good morning, $firstName $lastName!"
  case _ =>
    "Welcome! Please make sure to fill in your name!"
}
```

你可以在 REPL 中或者 worksheet 上试试这些代码。

### 小结

这一章里，我们学会了怎样去实现和使用返回不定长度值序列的提取器。
提取器是一个相当强大的工具，你可以灵活的重用它们，从而提供一种有效的方法来扩展要匹配的模式。

下一章，我会给出模式匹配在 Scala 中的不同用法（现在所见到的只是冰山一角）。
