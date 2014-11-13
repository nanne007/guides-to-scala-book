# 路径依赖类型

上一章介绍了类型类的概念，这种模式使设计出来的程序既拥抱扩展性，又不放弃具体的类型信息。
这一章，我们还将继续探究 Scala 的类型系统，讲讲另一个特性，
这个特性可以将 Scala 与其他主流编程语言区分开：依赖类型，特别是，路径依赖的类型和依赖方法类型。

一个广泛用于反对静态类型的论点是 “the compiler is just in the way”，
最终得到的都是数据，为什么还要建立一个复杂的类型层次结构？

到最后，静态类型的唯一目的就是，让“超级智能”的编译器来定期“羞辱”编程人员，以此来预防程序的 bug，
在事情变得糟糕之前，保证你做出正确的选择。

路径依赖类型是一种强大的工具，它把只有在运行期才知道的逻辑放在了类型里，
编译器可以利用这一点减少甚至防止 bug 的引入。

有时候，意外的引入路径依赖类型可能会导致难堪的局面，尤其是当你从来没有听说过它。
因此，了解和熟悉它绝对是个好主意，不管以后要不要用。

## 问题

先从一个问题开始，这个问题可以由路径依赖类型帮我们解决：
在同人小说中，经常会发生一些骇人听闻的事情。
比如说，两个主角去约会，即使这样的情景有多么的不合常理，甚至还有穿越的同人小说，两个来自不同系列的角色互相约会。

不过，好的同人小说写手对此是不屑一顾的。肯定有什么模式来阻止这样的错误做法。
下面是这种领域模型的初版：

``` scala
    object Franchise {
       case class Character(name: String)
     }
    class Franchise(name: String) {
      import Franchise.Character
      def createFanFiction(
        lovestruck: Character,
        objectOfDesire: Character): (Character, Character) = (lovestruck, objectOfDesire)
    }
```

角色用 `Character` 样例类表示， `Franchise` 类有一个方法，这个方法用来创建有关两个角色的小说。
下面代码创建了两个系列和一些角色：

``` scala
    val starTrek = new Franchise("Star Trek")
    val starWars = new Franchise("Star Wars")

    val quark = Franchise.Character("Quark")
    val jadzia = Franchise.Character("Jadzia Dax")

    val luke = Franchise.Character("Luke Skywalker")
    val yoda = Franchise.Character("Yoda")
```

不幸的是，这一刻，我们无法阻止不好的事情发生：

``` scala
    starTrek.createFanFiction(lovestruck = jadzia, objectOfDesire = luke)
```

多么恐怖的事情！某个人创建了一段同人小说，婕琪戴克斯和天行者卢克竟然在约会！我们不应该容忍这样的事情。

> 婕琪戴克斯：星际迷航中的角色：<http://en.wikipedia.org/wiki/Jadzia_Dax>
> 天行者卢克：星球大战中的角色：<http://en.wikipedia.org/wiki/Luke_Skywalker>

你的第一直觉可能是，在运行期做一些检查，保证约会的两个角色来自同一个特许商。
比如说：

``` scala
    object Franchise {
      case class Character(name: String, franchise: Franchise)
    }
    class Franchise(name: String) {
      import Franchise.Character
      def createFanFiction(
          lovestruck: Character,
          objectOfDesire: Character): (Character, Character) = {
        require(lovestruck.franchise == objectOfDesire.franchise)
        (lovestruck, objectOfDesire)
      }
    }
```

现在，每个角色都有一个指向所属发行商的引用，试图创建包含不同系列角色的小说会引发 `IllegalArgumentException` 异常。

## 路径依赖类型

这挺好，不是吗？毕竟这是被灌输多年的行为方式：快速失败。
然而，有了 Scala，我们能做的更好。
有一种可以更快速失败的方法，不是在运行期，而是在编译期。
为了实现它，我们需要将 `Character` 和它的 `Franchise` 之间的联系编码在类型层面上。

Scala **嵌套类型** 工作的方式允许我们这样做。
一个嵌套类型被绑定在一个外层类型的实例上，而不是外层类型本身。
这意味着，如果将内部类型的一个实例用在包含它的外部类型实例外面，会出现编译错误：

``` scala
    class A {
      class B
      var b: Option[B] = None
    }
    val a1 = new A
    val a2 = new A
    val b1 = new a1.B
    val b2 = new a2.B
    a1.b = Some(b1)
    a2.b = Some(b1) // does not compile
```

不能简单的将绑定在 `a2` 上的类型 `B` 的实例赋值给 `a1` 上的字段：前者的类型是 `a2.B` ，后者的类型是 `a1.B` 。
中间的点语法代表类型的路径，这个路径通往其他类型的具体实例。
因此命名为路径依赖类型。

下面的代码运用了这一技术：

``` scala
    class Franchise(name: String) {
      case class Character(name: String)
      def createFanFictionWith(
        lovestruck: Character,
        objectOfDesire: Character): (Character, Character) = (lovestruck, objectOfDesire)
    }
```

这样，类型 `Character` 嵌套在 `Franchise` 里，它依赖于一个特定的 `Franchise` 实例。

重新创建几个角色和发行商：

``` scala
    val starTrek = new Franchise("Star Trek")
    val starWars = new Franchise("Star Wars")

    val quark = starTrek.Character("Quark")
    val jadzia = starTrek.Character("Jadzia Dax")

    val luke = starWars.Character("Luke Skywalker")
    val yoda = starWars.Character("Yoda")
```

把角色放在一起构成小说：

``` scala
    starTrek.createFanFictionWith(lovestruck = quark, objectOfDesire = jadzia)
    starWars.createFanFictionWith(lovestruck = luke, objectOfDesire = yoda)
```

顺利编译！接下来，试着去把 `jadzia` 和 `luke` 放在一起：

``` scala
    starTrek.createFanFictionWith(lovestruck = jadzia, objectOfDesire = luke)
```

不应该的事情就会编译失败！编译器抱怨类型不匹配：

``` shell
    found   : starWars.Character
    required: starTrek.Character
                   starTrek.createFanFictionWith(lovestruck = jadzia, objectOfDesire = luke)
```

即使这个方法不是在 `Franchise` 中定义的，这项技术同样可用。
这种情况下，可以使用依赖方法类型，一个参数的类型信息依赖于前面的参数。

``` scala
    def createFanFiction(f: Franchise)(lovestruck: f.Character, objectOfDesire: f.Character) =
      (lovestruck, objectOfDesire)
```

可以看到， `lovestruck` 和 `objectOfDesire` 参数的类型依赖于传递给该方法的 `Franchise` 实例。
不过请注意：被依赖的实例只能在一个单独的参数列表里。

## 抽象类型成员

依赖方法类型通常和抽象类型成员一起使用。
假设我们在开发一个键值存储，只支持读取和存放操作，但是类型安全的。
下面是一个简化的实现：

``` scala
    object AwesomeDB {
      abstract class Key(name: String) {
        type Value
      }
    }
    import AwesomeDB.Key
    class AwesomeDB {
      import collection.mutable.Map
      val data = Map.empty[Key, Any]
      def get(key: Key): Option[key.Value] = data.get(key).asInstanceOf[Option[key.Value]]
      def set(key: Key)(value: key.Value): Unit = data.update(key, value)
    }
```

我们定义了一个含有抽象类型成员 `Value` 的类 `Key`。
`AwesomeDB` 中的方法可以引用这个抽象类型，即使不知道也不关心它到底是个什么表现形式。

定义一些想使用的具体的键：

``` scala
    trait IntValued extends Key {
     type Value = Int
    }
    trait StringValued extends Key {
      type Value = String
    }
    object Keys {
      val foo = new Key("foo") with IntValued
      val bar = new Key("bar") with StringValued
    }
```

之后，就可以存放键值对了：

``` scala
    val dataStore = new AwesomeDB
    dataStore.set(Keys.foo)(23)
    val i: Option[Int] = dataStore.get(Keys.foo)
    dataStore.set(Keys.foo)("23") // does not compile
```

## 实践中的路径依赖类型

在典型的 Scala 代码中，路径依赖类型并不是那么无处不在，但它确实是有很大的实践价值的，除了给同人小说建模之外。

最普遍的用法是和 **cake pattern** 一起使用，cake pattern 是一种组件组合和依赖管理的技术。
冠以这一点，可以参考 Debasish Ghosh 的 [文章](http://debasishg.blogspot.ie/2013/02/modular-abstractions-in-scala-with.html) 。


把一些只有在运行期才知道的信息编码到类型里，比如说：异构列表、自然数的类型级别表示，以及在类型中携带大小的集合，
路径依赖类型和依赖方法类型有着至关重要的角色。
Miles Sabin 正在 [Shapeless](https://github.com/milessabin/shapeless) 中探索 Scala 类型系统的极限。
