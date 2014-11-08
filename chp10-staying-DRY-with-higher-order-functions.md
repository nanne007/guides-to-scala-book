## 高阶函数与 DRY ##

前几章介绍了 Scala 容器类型的可组合性特征。
接下来，你会发现，Scala 中的一等公民——函数也具有这一性质。

组合性产生可重用性，虽然后者是经由面向对象编程而为人熟知，但它也绝对是纯函数的固有性质。
（纯函数是指那些没有副作用且是引用透明的函数）

一个明显的例子是调用已知函数实现一个新的函数，当然，还有其他的方式来重用已知函数。
这一章会讨论函数式编程的一些基本原理。
你将会学到如何使用高阶函数，以及重用已有代码时，遵守 [DRY](http://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 原则。

### 高阶函数 ###

和一阶函数相比，高阶函数可以有三种形式：

1. 一个或多个参数是函数，并返回一个值。
2. 返回一个函数，但没有参数是函数。
3. 上述两者叠加：一个或多个参数是函数，并返回一个函数。

看到这里的读者应该已经见到过第一种使用：我们调用一个方法，像 `map` 、 `filter` 、 `flatMap` ，并传递另一个函数给它。
传递给方法的函数通常是匿名函数，有时候，还涉及一些代码冗余。

这一章只关注另外两种功能：一个可以根据输入值构建新的函数，另一个可以根据现有的函数组合出新的函数。
这两种情况都能够消除代码冗余。

### 函数生成 ###

你可能认为依据输入值创建新函数的能力并不是那么有用。
函数组合非常重要，但在这之前，还是先来看看如何使用可以产生新函数的函数。

假设要实现一个免费的邮件服务，用户可以设置对邮件的屏蔽。
我们用一个简单的样例类来代表邮件：

``` scala
case class Email(
  subject: String,
  text: String,
  sender: String,
  recipient: String
)
```


想让用户可以自定义过滤条件，需有一个过滤函数——类型为 `Email => Boolean` 的谓词函数，
这个谓词函数决定某个邮件是否该被屏蔽：如果谓词成真，那这个邮件被接受，否则就被屏蔽掉。

``` scala
type EmailFilter = Email => Boolean
def newMailsForUser(mails: Seq[Email], f: EmailFilter) = mails.filter(f)
```

注意，类型别名使得代码看起来更有意义。

现在，为了使用户能够配置邮件过滤器，实现了一些可以产生 `EmailFilter` 的工厂方法：

``` scala
  val sentByOneOf: Set[String] => EmailFilter =
	senders =>
	  email => senders.contains(email.sender)
  val notSentByAnyOf: Set[String] => EmailFilter =
	senders =>
	  email => !senders.contains(email.sender)
  val minimumSize: Int => EmailFilter =
	n =>
	  email => email.text.size >= n
  val maximumSize: Int => EmailFilter =
	n =>
	  email => email.text.size <= n
```

这四个 *vals* 都是可以返回 `EmailFilter` 的函数，
前两个接受代表发送者的 `Set[String]` 作为输入，后两个接受代表邮件内容长度的 `Int` 作为输入。

可以使用这些函数来创建 `EmialFilter` ：

``` scala
  val emailFilter: EmailFilter = notSentByAnyOf(Set("johndoe@example.com"))
  val mails = Email(
	subject = "It's me again, your stalker friend!",
	text = "Hello my friend! How are you?",
	sender = "johndoe@example.com",
	recipient = "me@example.com") :: Nil
  newMailsForUser(mails, emailFilter) // returns an empty list
```

这个过滤器过滤掉列表里唯一的一个元素，因为用户屏蔽了来自 `johndoe@example.com` 的邮件。
可以用工厂方法创建任意的 `EmailFilter` 函数，这取决于用户的需求了。

### 重用已有函数 ###

当前的解决方案有两个问题。第一个是工厂方法中有重复代码。
上文提到过，函数的组合特征可以很轻易的保持 DRY 原则，既然如此，那就试着使用它吧！

对于 `minimumSize` 和 `maximumSize` ，我们引入一个叫做 `sizeConstraint` 的函数。
这个函数接受一个谓词函数，该谓词函数检查函数内容长度是否OK，邮件长度会通过参数传递给它：

``` scala
  type SizeChecker = Int => Boolean
  val sizeConstraint: SizeChecker => EmailFilter =
	f =>
	  email => f(email.text.size)
```

这样，我们就可以用 `sizeConstraint` 来表示 `minimumSize` 和 `maximumSize` 了：

``` scala
  val minimumSize: Int => EmailFilter =
	n =>
	  sizeConstraint(_ >= n)
  val maximumSize: Int => EmailFilter =
	n =>
	  sizeConstraint(_ <= n)
```

### 函数组合 ###

为另外两个谓词（`sentByOneOf`、 `notSentByAnyOf`）介绍一个通用的高阶函数，通过它，可以用一个函数去表达另外一个函数。

这个高阶函数就是 `complement` ，给定一个类型为 `A => Boolean` 的谓词，它返回一个新函数，
这个新函数总是得出和谓词相对立的结果：

``` scala
  def complement[A](predicate: A => Boolean) = (a: A) => !predicate(a)
```

现在，对于一个已有的谓词 `p` ，调用 `complement(p)` 可以得到它的补。
然而， `sentByAnyOf` 并不是一个谓词函数，它返回类型为 `EmailFilter` 的谓词。

Scala 函数的可组合能力现在就用的上了：给定两个函数 `f` 、 `g` ， `f.compose(g)` 返回一个新函数，
调用这个新函数时，会首先调用 `g` ，然后应用 `f` 到 `g` 的返回结果上。
类似的， `f.andThen(g)` 返回的新函数会应用 `g` 到 `f` 的返回结果上。

知道了这些，我们就可以重写 `notSentByAnyOf` 了：

``` scala
  val notSentByAnyOf = sentByOneOf andThen (g => complement(g))
```

上面的代码创建了一个新的函数，
这个函数首先应用 `sentByOneOf` 到参数 `Set[String]` 上，产生一个 `EmailFilter` 谓词,
然后，应用 `complement` 到这个谓词上。
使用 Scala 的下划线语法，这短代码还能更精简：

``` scala
  val notSentByAnyOf = sentByOneOf andThen (complement(_))
```

读者可能已经注意到，
给定 `complement` 函数，也可以通过 `minimumSize` 来实现 `maximumSize` 。
不过，先前的实现方式更加灵活，它允许检查邮件内容的任意长度。
谓

#### 谓词组合 ####

邮件过滤器的第二个问题是，当前只能传递一个 `EmailFilter` 给 `newMailsForUser` 函数，而用户必然想设置多个标准。
所以需要可以一种可以创建组合谓词的方法，这个组合谓词可以在任意一个标准满足的情况下返回 `true` ，或者在都不满足时返回 `false` 。

下面的代码是一种实现方式：

``` scala
  def any[A](predicates: (A => Boolean)*): A => Boolean =
	a => predicates.exists(pred => pred(a))
  def none[A](predicates: (A => Boolean)*) = complement(any(predicates: _*))
  def every[A](predicates: (A => Boolean)*) = none(predicates.view.map(complement(_)): _*)
```

`any` 函数返回的新函数会检查是否有一个谓词对于输入 `a` 成真。
`none` 返回的是 `any` 返回函数的补，只要存在一个成真的谓词， `none` 的条件就无法满足。
最后， `every` 利用 `none` 和 `any` 来判定是否每个谓词的补对于输入 `a` 都不成真。

可以使用它们来创建代表用户设置的组合 `EmialFilter` ：

``` scala
  val filter: EmailFilter = every(
	  notSentByAnyOf(Set("johndoe@example.com")),
	  minimumSize(100),
	  maximumSize(10000)
	)
```

#### 流水线组合 ####

再举一个函数组合的例子。回顾下上面的场景，
邮件提供者不仅想让用户可以配置邮件过滤器，还想对用户发送的邮件做一些处理。
这是一些简单的 `Emial => Email` 函数，一些可能的处理函数是：

``` scala
  val addMissingSubject = (email: Email) =>
	if (email.subject.isEmpty) email.copy(subject = "No subject")
	else email
  val checkSpelling = (email: Email) =>
	email.copy(text = email.text.replaceAll("your", "you're"))
  val removeInappropriateLanguage = (email: Email) =>
	email.copy(text = email.text.replaceAll("dynamic typing", "**CENSORED**"))
  val addAdvertismentToFooter = (email: Email) =>
	email.copy(text = email.text + "\nThis mail sent via Super Awesome Free Mail")
```

现在，根据老板的心情，可以按需配置邮件处理的流水线。
通过 `andThen` 调用实现，或者使用 Function 伴生对象上的 `chain` 方法：

``` scala
  val pipeline = Function.chain(Seq(
	addMissingSubject,
	checkSpelling,
	removeInappropriateLanguage,
	addAdvertismentToFooter))
```

### 高阶函数与偏函数 ###

这部分不会关注细节，不过，在知道了这么多通过高阶函数来组合和重用函数的方法之后，你可能想再重新看看偏函数。

#### 链接偏函数 ####

匿名函数那一章提到过，偏函数可以被用来创建责任链：
`PartialFunction` 上的 `orElse` 方法允许链接任意个偏函数，从而组合出一个新的偏函数。
不过，只有在一个偏函数没有为给定输入定义的时候，才会把责任传递给下一个偏函数。
从而可以做下面这样的事情：

``` scala
  val handler = fooHandler orElse barHandler orElse bazHandler
```

#### 再看偏函数 ####

有时候，偏函数并不合适。
仔细想想，一个函数没有为所有的输入值定义操作，这样的事实还可以用一个返回 `Option[A]` 的标准函数代替：
如果函数为一个输入定义了操作，那就返回 `Some[A]` ，否则返回 `None` 。

要这么做的话，可以在给定的偏函数 `pf` 上调用 `lift` 方法得到一个普通的函数，这个函数返回 `Option` 。
反过来，如果有一个返回 `Option` 的普通函数 `f` ，也可以调用 `Function.unlift(f)` 来得到一个偏函数。
总

### 总结 ###

这一章给出了高阶函数的使用，利用它可以在一个新的环境里重用已有函数，并用灵活的方式去组合它们。
在所举的例子中，就代码行数而言，可能看不出太多价值，
这些例子都很简单，只是为了说明而已，在架构层面，组合和重用函数是有很大帮助的。

下一章，我们继续探索函数组合的方式：*函数部分应用和柯里化(Partial Function Application and Currying)*。
