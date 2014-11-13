# 柯里化和部分函数应用

上一章重点在于代码重复：提升现有的函数功能、或者将函数进行组合。
这一章，我们来看看另外两种函数重用的机制：**函数的部分应用(Partial Application of Functions)** 、 **柯里化(Currying)** 。

## 部分应用的函数

和其他遵循函数式编程范式的语言一样，Scala 允许*部分*应用一个函数。
调用一个函数时，不是把函数需要的所有参数都传递给它，而是仅仅传递一部分，其他参数留空；
这样会生成一个新的函数，其参数列表由那些被留空的参数组成。（不要把这个概念和偏函数混淆）

为了具体说明这一概念，回到上一章的例子：
假想的免费邮件服务，能够让用户配置筛选器，以使得满足特定条件的邮件显示在收件箱里，其他的被过滤掉。

`Email` 类看起来仍然是这样：

``` scala
case class Email(
  subject: String,
  text: String,
  sender: String,
  recipient: String)
type EmailFilter = Email => Boolean
```

过滤邮件的条件用谓词 `Email => Boolean` 表示， `EmailFilter` 是其别名。
调用适当的工厂方法可以生成这些谓词。

上一章，我们创建了两个这样的工厂方法，它们检查邮件内容长度是否满足给定的最大值或最小值。
这一次，我们使用部分应用函数来实现这些工厂方法，做法是，修改 `sizeConstraint` ，固定某些参数可以创建更具体的限制条件：

其修改后的代码如下：

``` scala
    type IntPairPred = (Int, Int) => Boolean
    def sizeConstraint(pred: IntPairPred, n: Int, email: Email) =
      pred(email.text.size, n)
```

上述代码为一个谓词函数定义了别名 `IntPairPred` ，该函数接受一对整数（值 `n` 和邮件内容长度），检查邮件长度对于 `n` 是否 OK。

请注意，不像上一章的 `sizeConstraint` ，这一个并不返回新的 `EmailFilter`，它只是简单的用参数做计算，返回一个布尔值。
秘诀在于，你可以部分应用这个 `sizeConstraint` 来得到一个 `EmailFilter` 。

遵循 DRY 原则，我们先来定义常用的 `IntPairPred` 实例，这样，在调用 `sizeConstraint` 时，不需要重复的写相同的匿名函数，只需传递下面这些：

``` scala
    val gt: IntPairPred = _ > _
    val ge: IntPairPred = _ >= _
    val lt: IntPairPred = _ < _
    val le: IntPairPred = _ <= _
    val eq: IntPairPred = _ == _
```

最后，调用 `sizeConstraint` 函数，用上面的 `IntPairPred` 传入第一个参数：

``` scala
    val minimumSize: (Int, Email) => Boolean = sizeConstraint(ge, _: Int, _: Email)
    val maximumSize: (Int, Email) => Boolean = sizeConstraint(le, _: Int, _: Email)
```

对所有没有传入值的参数，必须使用占位符 `_` ，还需要指定这些参数的类型，这使得函数的部分应用多少有些繁琐。
Scala 编译器无法推断它们的类型，方法重载使编译器不可能知道你想使用哪个方法。

不过，你可以绑定或漏掉任意个、任意位置的参数。比如，我们可以漏掉第一个值，只传递约束值 `n` ：

``` scala
    val constr20: (IntPairPred, Email) => Boolean =
      sizeConstraint(_: IntPairPred, 20, _: Email)

    val constr30: (IntPairPred, Email) => Boolean =
      sizeConstraint(_: IntPairPred, 30, _: Email)
```

得到的两个函数，接受一个 `IntPairPred` 和一个 `Email` 作为参数，
然后利用谓词函数 `IntPairPred` 把邮件长度和 `20` 、 `30` 比较，
只不过比较方法的逻辑 `IntPairPred` 需要另外指定。

由此可见，虽然函数部分应用看起来比较冗长，但它要比 Clojure 的灵活，在 Clojure 里，必须从左到右的传递参数，不能略掉中间的任何参数。

### 从方法到函数对象

在一个方法上做部分应用时，可以不绑定任何的参数，这样做的效果是产生一个函数对象，并且其参数列表和原方法一模一样。
通过这种方式可以将方法变成一个可赋值、可传递的函数！

``` scala
    val sizeConstraintFn: (IntPairPred, Int, Email) => Boolean = sizeConstraint _
```

## 更有趣的函数

部分函数应用显得太啰嗦，用起来不够优雅，幸好还有其他的替代方法。

也许你已经知道 Scala 里的方法可以有多个参数列表。
下面的代码用多个参数列表重新定义了 `sizeConstraint` ：

``` scala
    def sizeConstraint(pred: IntPairPred)(n: Int)(email: Email): Boolean =
      pred(email.text.size, n)
```

如果把它变成一个可赋值、可传递的函数对象，它的签名看起来会像是这样：

``` scala
    val sizeConstraintFn: IntPairPred => Int => Email => Boolean = sizeConstraint _
```

这种单参数的链式函数称做 **柯里化函数** ，以发明人 Haskell Curry 命名。在 Haskell 编程语言里，所有的函数默认都是柯里化的。

`sizeConstraintFn` 接受一个 `IntPairPred` ，返回一个函数，这个函数又接受 `Int` 类型的参数，返回另一个函数，最终的这个函数接受一个 `Email` ，返回布尔值。

现在，可以把要传入的 `IntPairPred` 传递给 `sizeConstraint` 得到：

``` scala
    val minSize: Int => Email => Boolean = sizeConstraint(ge)
    val maxSize: Int => Email => Boolean = sizeConstraint(le)
```

被留空的参数没必要使用占位符，因为这不是部分函数应用。

现在，可以通过这两个柯里化函数来创建 `EmailFilter` 谓词：

``` scala
    val min20: Email => Boolean = minSize(20)
    val max20: Email => Boolean = maxSize(20)
```

也可以在柯里化的函数上一次性绑定多个参数，直接得到上面的结果。
传入第一个参数得到的函数会立即应用到第二个参数上：

``` scala
    val min20: Email => Boolean = sizeConstraintFn(ge)(20)
    val max20: Email => Boolean = sizeConstraintFn(le)(20)
```

### 函数柯里化

有时候，并不总是能提前知道要不要将一个函数写成柯里化形式，毕竟，和只有单参数列表的函数相比，柯里化函数的使用并不清晰。
而且，偶尔还会想以柯里化的形式去使用第三方的函数，但这些函数的参数都在一个参数列表里。

这就需要一种方法能对函数进行柯里化。
这种的柯里化行为本质上也是一个高阶函数：接受现有的函数，返回新函数。
这个高阶函数就是 `curried` ：`curried` 方法存在于 `Function2` 、 `Function3` 这样的多参数函数类型里。
如果存在一个接受两个参数的 `sum` ，可以通过调用 `curried` 方法得到它的柯里化版本：

``` scala
    val sum: (Int, Int) => Int = _ + _
    val sumCurried: Int => Int => Int = sum.curried
```

使用 `Funtion.uncurried` 进行反向操作，可以将一个柯里化函数转换成非柯里化版本。

### 函数化的依赖注入

在这一章的最后，我们来看看柯里化函数如何发挥其更大的作用。
来自 Java 或者 .NET 世界的人，或多或少都用过依赖注入容器，这些容器为使用者管理对象，以及对象之间的依赖关系。
在 Scala 里，你并不真的需要这样的外部工具，语言已经提供了许多功能，这些功能简化了依赖注入的实现。

函数式编程仍然需要注入依赖：应用程序中上层函数需要调用其他函数。
把要调用的函数硬编码在上层函数里，不利于它们的独立测试。
从而需要把被依赖的函数以参数的形式传递给上层函数。

但是，每次调用都传递相同的依赖，是不符合 DRY 原则的，这时候，柯里化函数就有用了！
柯里化和部分函数应用是函数式编程里依赖注入的几种方式之一。

下面这个简化的例子说明了这项技术：

``` scala
    case class User(name: String)
    trait EmailRepository {
      def getMails(user: User, unread: Boolean): Seq[Email]
    }
    trait FilterRepository {
      def getEmailFilter(user: User): EmailFilter
    }
    trait MailboxService {
      def getNewMails(emailRepo: EmailRepository)(filterRepo: FilterRepository)(user: User) =
        emailRepo.getMails(user, true).filter(filterRepo.getEmailFilter(user))
      val newMails: User => Seq[Email]
    }
```

这个例子有一个依赖两个不同存储库的服务，这些依赖被声明为 `getNewMails` 方法的参数，并且每个依赖都在一个单独的参数列表里。

`MailboxService` 实现了这个方法，留空了字段 `newMails`，这个字段的类型是一个函数： `User => Seq[Email]`，依赖于 `MailboxService` 的组件会调用这个函数。

扩展 `MailboxService` 时，实现 `newMails` 的方法就是应用 `getNewMails` 这个方法，把依赖 `EmailRepository` 、 `FilterRepository` 的具体实现传递给它：

``` scala
    object MockEmailRepository extends EmailRepository {
      def getMails(user: User, unread: Boolean): Seq[Email] = Nil
    }
    object MockFilterRepository extends FilterRepository {
      def getEmailFilter(user: User): EmailFilter = _ => true
    }
    object MailboxServiceWithMockDeps extends MailboxService {
      val newMails: (User) => Seq[Email] =
        getNewMails(MockEmailRepository)(MockFilterRepository) _
    }
```

调用 `MailboxServiceWithMockDeps.newMails(User("daniel")` 无需指定要使用的存储库。
在实际的应用程序中，这个服务也可能是以依赖的方式被使用，而不是直接引用。

这可能不是最强大、可扩展的依赖注入实现方式，但依旧是一个非常不错的选择，对展示部分函数应用和柯里化更广泛的功用来说，这也是一个不错的例子。
如果你想知道更多关于这一点的知识，推荐看 Debasish Ghosh 的幻灯片
“[Dependency Injection in Scala](http://de.slideshare.net/debasishg/dependency-injection-in-scala-beyond-the-cake-pattern)”。

## 总结

这一章讨论了两个附加的可以避免代码重复的函数式编程技术，
并且在这个基础上，得到了很大的灵活性，可以用多种不同的形式重用函数。
部分函数应用和柯里化，这两者或多或少都可以实现同样的效果，只是有时候，其中的某一个会更为优雅。
下一章会继续探讨保持灵活性的方法：类型类（type class）。
