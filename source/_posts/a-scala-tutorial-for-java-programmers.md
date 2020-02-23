---
title: a-scala-tutorial-for-java-programmers
date: 2017-12-10 13:25:42
tags: translation
---
# [译]为Java程序员而做的Scala教程

原文地址：[http://docs.scala-lang.org/tutorials/scala-for-java-programmers.html](http://docs.scala-lang.org/tutorials/scala-for-java-programmers.html)

## 简介
本文对Scala语言及其编译器进行了一个跨素的介绍。旨在向有编程经验的人介绍Scala能被用来做什么，阅读前起码需要了解Java的面向对象的知识。

## 一个简单的例子
在第一个列子里，我们将用到标准的hello world程序。这段程序没什么了不起，但是能在对语言本身并不了解的情况下，描述出如何使用Scala的工具。程序如下：
```scala
object HelloWorld {
    def main(args: Array[String]) {
        println("Hello, World!")
    }
}
```
程序的结构和Java程序很相似：它们由一个叫*main*的方法组成，该方法接收了一个string数组的命令行参数。main的方法体里有一个单独的调用，那是一个预定义的*println*。*main*方法不返回值(它是一个过程方法). 因此, 不需要声明返回值.

和Java程序不太一样的是，声明一个*对象*是可以包含*main*方法的。 这种声明方式，引入了我们常说的单例对象，也就是一个类只有一个实例的情况。上面的那种声明方式同时声明了叫HelloWorld的类和它的一个实例，这个实例也叫HelloWorld. 这个实例会在请求的时候被创建，也就是第一次被调用的时候.

机制的读者会发现main方法并没有被声明为static. 因为Scala里没有静态成员 (方法或变量域). 在相比使用静态成员,Scala会将这些静态成员定义在单例的对象里.

### 编译这个例子

要编译这个例子, 我们要用scalac, Scala编译器. scalac和大部分的编译器一样:接收一个源文件作为参数, 也许有些可选项, 然后产生一个或若干个对象文件. 产生的对象文件都是标准的Java文件.

如果我们将上面的程序保存为HelloWorld.scala, 我们能够通过下面的命令行编译它(大于号 > 只表示这是在shell环境下，不要一起码了):

 &gt; scalac HelloWorld.scala

这将在当前文件夹生成若干文件，其中会有一个HelloWorld.class, 这个文件中的类可以被scala 命令行执行.

### 运行这个例子

编译完成后, Scala程序就可以通过scala命令运行. 它的使用方法和用java命令运行Java程序很类似,接受的参数夜是一样. 上面的例子可以通过下面的命令运行，并得到对应的输出:

 &gt; scala -classpath . HelloWorld
Hello, world!

## 和Java交互

Scala的一个优势是，它和Java代码的交互非常简单. java.lang包里所有的类都是被默认引入的，而其他的类需要显式引入.

让我们通过一个例子来解释这点. 我们希望获取当前时间，并按照某个地区的习惯格式化它,比如中国。

Java定义了许多强大的工具类, 比如 Date和DateFormat. 因为Scala和Java可以无缝操作, 因此在Scala中没有必要实现在Java库中已有的类，而且Scala引入包的方式相比Java而言也更简单：

```scala
import java.util.{Date, Locale}
import java.text.DateFormat
import java.text.DateFormat._

object ChineseDate {
  def main(args: Array[String]) {
    val now = new Date
    val df = getDateInstance(LONG, Locale.SIMPLIFIED_CHINESE)
    println(df format now)
  }
}
```

Scala的引入语句看起来和Java的很相似, 然而实际上前者更强大. 来自同一个包的多个类可以用包括在中括号中的方式，一起被引入，如上面第一行。另一个不同是，当引入整个包，或者整个类时，Scala使用下划线 (_) 取代了星号 (*). 这是因为，在Scala中星号是一个有效的标识符(比如. 方法名), 这在之后也会看到。

第三行的import语句因为引入了DateFormat类. 这就使得静态方法 getDateInstance 和静态域LONG是直接可见的了.

在main方法中我们首先创建了一个Java的Data类的实例，它默认是当前时间。接着我们用之前引入的静态方法getDateInstance定义了一个date format. 最后我们按照地区格式化打印出了当前时间. 最后一行展示出了Scala语法里的一个有趣的特性. 只有一个参数的方法可以通过内嵌的语法调用. 
```scala
df format now 
df.format(now)
```
前一个表达式是后者的一种简洁的写法。


虽然这表面看起来是一个很小的语法细节, 但结果却很重要。这一点会在之后继续深入。

为了结束这一部分和Java的交互的内容, 需要另外指出的是，Scala是可以直接集成Java的类，也可以去实现Java的接口的。

## 所有的东西都是对象

Scala是一个纯面向对象的语言，从这个意义上来讲，所有的东西都是对象，包括数字和函数。从这点上来说，它和Java不一样，因为Java区分了原始类型和引用类型, 也不能将函数作为值来使用.

### 数字是对象

因为数字是对象, 所以它们也有方法. 而且事实上，如下的算数表达式是由多个函数调用组成的:

1 + 2 * 3 / x
这等价于下面的表达式

(1).+(((2).*(3))./(x))
这表示+, *, 等等在Scala里都是有效的标识符。

第二个版本里，数字外的括号是必须的，因为Scala的语法解析使用了词标记的最长匹配规则。因此表达式 1.+(2) 会被分割成 1., +, and 2.，因为1.和1都是有效的，但是1.比后者更长，而1.会被解释成字面量1.0，从而变成一个Double而不是Int。写成 (1).+(2) 可以避免 1 被解释成一个Double。


### 函数是对象

或许更让Java程序员吃惊的是, 在Scala里，函数也是对象. 这就使得函数作为参数传递，作为变量存储，以及从其他函数返回都成为可能。这种将函数当成值来操作的能力是一种非常有趣的编程范式，也就是函数式编程的基石之一。

作为阐释为什么将函数看作值是非常有用的例子，让我们看一个计时器函数，它希望每一秒都有些动作。 我们怎么传递这个需要做的动作呢？ 合理的做法是，作为一个函数. 这种简单的函数传递方式对许多程序员来说都很熟悉， 通常在UI的代码里, 需要注册一个回调函数，它们会在某些事件发生时被调用。

下面的代码片段，有个计时器函数oncePerSecond, 它接受一个回调函数作为参数. 函数的类型被写成 () => Unit 这个类型代表的是一类函数，它们不接收任何的参数，也不返回任何值 (Unit 和C/C++的void类似). 主函数调用计时器函数，传递的是一个在终端上打印一句话的回调函数，换句话说，这段程序每秒都会在控制台上打印出“time flies like an arrow”。

```scala
object Timer {
  def oncePerSecond(callback: () => Unit) {
    while (true) { callback(); Thread sleep 1000 }
  }
  def timeFlies() {
    println("time flies like an arrow...")
  }
  def main(args: Array[String]) {
    oncePerSecond(timeFlies)
  }
}
```

为了打印出这句话，我们使用的时预定义的println，而不是System.out中的那个。

### 匿名函数

尽管上面的程序很好理解，它也能被稍作改进. 第一，注意到函数timeFlies被定义只是为了之后传递给oncePerSecond函数。 为了这个只用一次的函数，对它命名是没有必要的, 事实上，在这个函数被传递给oncePerSecond时再去构造它会更好. 在Scala里使用匿名函数使得这成为可能，匿名函数指的是函数没有名字. 使用匿名函数修正的版本是：

```scala
object TimerAnonymous {
  def oncePerSecond(callback: () => Unit) {
    while (true) { callback(); Thread sleep 1000 }
  }
  def main(args: Array[String]) {
    oncePerSecond(() =>
      println("time flies like an arrow..."))
  }
}
```

和右键头 => 相关的就是一个匿名函数，箭头分隔了函数的参数列表和函数体。这个例子中，参数列表是个空的，因为箭头左侧的括号里是个空的，而函数体则与上面的timeFlies函数相同。

## 类

正如我们所见，Scala是一个面向对象的语言， 因此它是有类这个概念的. (有些面向对象语言是有缺失的，它们没有类的概念，但是Scala是有的) 。在Scala中声明类的语法和Java里的很相近。 一个重要的不同是，Scala中的类可以有参数. 这在下面对复数的定义里可以体现出来。

```scala
class Complex(real: Double, imaginary: Double) {
  def re() = real
  def im() = imaginary
}
```

这个复数类接收了两个参数, 复数的实数部分和虚数部分。当创建这个类的实例时，这两个参数必须被传递, 如： new Complex(1.5, 2.3). 这个类包含两个方法, re and im,可以访问这两个部分。

可以注意到，这两个方法的返回值并没有明显的给出来. 通过查看这些方法右边，编译器能够推倒出这两个方法返回的是Double类型的值。

编译器并不是总能像上面这样推断出值的类型，不幸的是并没有简单的规则能准确的知道，哪些类型是可以被推断出来的，而哪些却不行。在实际使用中，这通常并不是一个严重的问题，因为如果不能推断出那些没有明确指出的类型，编译器会报错。一个简单的规则是，Scala的初级程序员应当在能简单地从上下文推断出类型的时候，不声明变量的类型，然后看编译器是否通过。试过几次后，程序员就能对什么时候略去变量声明，而什么时候应该指定类型有感觉了。

### 没有参数的方法

上面的方法re和im有个小问题，为了去调用它们，必须在它们的名字后面放上空的括号，就像下面这样：

```scala
object ComplexNumbers {
  def main(args: Array[String]) {
    val c = new Complex(1.2, 3.4)
    println("imaginary part: " + c.im())
  }
}
```

如果不带上空的括号，能像访问成员域一样访问实部和虚部就很好了。这在Scala中是可行的，通过用没有参数的方法定义它们，就可以实现简化。这些方法不同于0参数的方法。无论是声明还是使用，它们的名字后面都是没有括号的。因此复数类可以被按如下重写。

```scala
class Complex(real: Double, imaginary: Double) {
  def re = real
  def im = imaginary
}
```

### 继承和重写

Scala中所有的类都继承自一个父类。当像上面例子中的复数一样，没有指定父类时，默认使用的是scala.AnyRef。 

在Scala中，是可以重写父类方法的。 然而，为了避免意外性的重写，当对方法进行重写时，需要强制使用override关键字。对Complex类增加一个继承自Object的重写方法，如下：

```scala
class Complex(real: Double, imaginary: Double) {
  def re = real
  def im = imaginary
  override def toString() =
    "" + re + (if (im < 0) "" else "+") + im + "i"
}
```

### Case Classes 和模式匹配

树是一类经常出现在程序中的数据结构。比如解释器和编译器在内部通常将程序表示成树结构，XML文档是树，一系列的容器，比如RBTree也是树。

我们现在通过一个小的计算程序考察在Scala中，树是如何被表示和操作的。这个程序的目的是操作简单的算数表达式，包括了整数常量和变量的加法，两个简单这种例子是：1+2 and (x+x)+(7+y).

首先我们决定表示这些表达式，最自然的方式就是使用树：节点是运算符（也就是这里的加法），叶子是值（也就是这里的常量和变量）。

在Java中，树节点可以用抽象的父类表示，具体的子类是节点或者叶子。在函数式编程语言里，对所有的情况都可以使用代数数据类型。Scala提供了case class的概念，这有点在前面两者的中间。下面是树的数据类型如何被定义的：

```scala
abstract class Tree
case class Sum(l: Tree, r: Tree) extends Tree
case class Var(n: String) extends Tree
case class Const(v: Int) extends Tree
```

事实上类Sum, Var和 Const 被声明为 case classes 表示它们在若干方面不同于标准的类。 

+ 在创建类的实例时，不再强制使用new关键字（也就是说可以用Const(5)替代 new Const(5)）。
+ getter函数会为构造函数中的参数自动定义（Const实例c可以用c.v的写法获取构造函数中的参数v的值）
+ 提供了equals和hashCode的默认定义，它们不是作用在实例的指针，而是实例内容上。
+ 提供了toString的默认定义，可以以源代码形式打印出实例的值。（如表达式x+1，可以打印出Sum(Var(x),Const(1))）
+ 这些类的实例，可以通过模式匹配被分解，正如下面将讲述的。

既然已经用数据类型表示来算数表达式，我们现在就定义操作符来操作它们。我们以在某些环境里计算表达式的函数开始。环境的目的是讲值赋给变量，比如在｛ x -> 5｝也就是x为5的环境里，表达式x+1结果为6.

因此我们必须找到一个表示环境的方式. 我们当然可以使用关联性的数据结构，比如hash表，但是我们也能直接使用函数。环境可以看做一个将值和变量名关联的函数。环境{ x -> 5 }可以用Scala简写为：

```scala
{ case "x" => 5 }
```

这种记法定义了一个函数，如果函数接受一个字符串"x"作为参数，则返回整数5，否则抛出异常。

在写评价（evaluation）函数前，我们可以给这种环境的类型定一个名字。环境的类型当然能总是使用String => Int来表示，但是若我们为这种类型引入一个名字，程序就会简化，之后的变化也会更简单，这在Scala中已经可以通过下面的记法实现了。

```scala
type Environment = String => Int
```

之后,type Environment就能作为String => Int这类函数类型的别名使用了。

我们现在给出评价函数. 概念上很简单: 
+ 两个表达式求和后的值是两个表达式的值的和; 
+ 变量的值来源于环境; 
+ 常量的值是常量自身

然而用Scala表达上述概念，就会困难一些：

```scala
def eval(t: Tree, env: Environment): Int = t match {
  case Sum(l, r) => eval(l, env) + eval(r, env)
  case Var(n)    => env(n)
  case Const(v)  => v
}
```
当在树t上执行模式匹配时，评价函数开始工作。
直观上来看，上面的定义意境很清晰了：
它首先检查t是不是一个Sum，如果是，将左子树绑定到新变量l上，右子树绑定到变量r上，然后继续处理箭头后面的表达式。表达式能利用绑定在箭头左侧模式上的变量，比如l，r。
如果第一次检查不成功，也就是树不是一个Sum，它会继续检查是不是一个Var。如果是，则将Var节点上的名字和变量n绑定，并处理右手边的表达式。
如果第二次检查还失败，则t不是一个Sum也不是Var，它会检查是不是一个Const，如果是，则将Const中的值和变量v绑定，接着处理右手边。
最终，若所有的检查都失败了，将会抛出一个异常，通知表达式的模式匹配失败，这可能只有当树的声明了更多的子类时会发生。

我们看到模式匹配的基本思想是试图将一个值与一系列模式匹配，一旦模式匹配，提取并命名值的各个部分，最后使用这些命名部分去计算一些代码。

有经验的面向对象编程者可能会奇怪为什么我们在Tree类及其子类里不定义eval方法。我们事实上也可以这样做，因为Scala允许在case classes里像在通常的类里一样定义方法。决定使用模式匹配还是方法是一个代码品味的问题，但在拓展性上也会有影响：
+ 在使用方法时，很容易添加一种新类型的节点，因为只需重新定义一个子类即可；另一方面，若添加一个操作树的新操作是不方便了，因为这需要树的所有子类都进行修改。
+ 在使用模式匹配时，情况就相反了：添加新类型的节点需要修改Tree上所有的模式匹配函数，然后将新节点考虑进去，另一方面，添加新的操作就容易多了，只需要添加一个新的独立的操作就行了。

为了模式匹配更进一步，我们在算数表达式上新定义一个操作：符号求导，读者可能记得这个操作的规则如下：
+ 导数的和是和的导数
+ 若变量v是求导的变量，则v的导数是1，否则导数是0
+ 常数的导数是0

这个规则能被字面的转化成Scala代码，也就是下面的定义：

```scala
def derive(t: Tree, v: String): Tree = t match {
  case Sum(l, r) => Sum(derive(l, v), derive(r, v))
  case Var(n) if (v == n) => Const(1)
  case _ => Const(0)
}
```
这个函数引入了两个与模式匹配相关的新概念。
首先，变量的case表达式有一个守护表达式，也就是if关键字后的表达式。除非该表达式为真，模式匹配才成功。在这里，它是用来确保只有在变量的名字和求导的变量v一致时，才返回常数1的。
第二个新特性是通配符，不需要任何名字，这种情况下会匹配所有的值。

虽然我们还没有深入到模式匹配最厉害的地方，但为了简短起见，我们在这里停住。我们还要看看上面的两个函数是如何在实际的例子里起作用的。为了这个目的，让我们写一个简单的主函数，它表现了几个在表达式 (x+x)+(7+y)上的操作：首先在环境{ x -> 5, y -> 7 }上计算x的导数，然后在是y的导数。

```scala
def main(args: Array[String]) {
  val exp: Tree = Sum(Sum(Var("x"),Var("x")),Sum(Const(7),Var("y")))
  val env: Environment = { case "x" => 5 case "y" => 7 }
  println("Expression: " + exp)
  println("Evaluation with x=5, y=7: " + eval(exp, env))
  println("Derivative relative to x:\n " + derive(exp, "x"))
  println("Derivative relative to y:\n " + derive(exp, "y"))
}
```

执行这段程序，期待的输出应该是：
```shell
Expression: Sum(Sum(Var(x),Var(x)),Sum(Const(7),Var(y)))
Evaluation with x=5, y=7: 24
Derivative relative to x:
 Sum(Sum(Const(1),Const(1)),Sum(Const(0),Const(0)))
Derivative relative to y:
 Sum(Sum(Const(0),Const(0)),Sum(Const(0),Const(1)))
```

检查输出, 我们能看到求导的结果在展示给用户前就已经简化了。使用模式匹配去定义一个基本的简化函数是一个有趣（也很有技巧）的问题，这就留给有经验的读者了。

## 特性（traits）
除从父类继承以外，Scala的类也能从一个或若干个特征中引入代码。

或许对Java程序员来说，最简单的理解traits的方式是将它们看成能包含代码的接口。在Scala中，当一个类从特征中继承了，它就实现了特征的接口，也继承了包含在特征中的所有代码。

为了了解特征多么有用，我们先看看经典的例子，排序对象。比较特定类的对象是很有用的，例如可以对它们进行排序。在java中，有可比性的对象实现了Comparable接口。在Scala里将Comparable定义为一个特征，可以做的比Java更好一些，我们将这称为Ord。

比较对象时，有六个不同的谓词可能有用: 小于，小于等于，等于，不等于，大于等于，大于。然而，这六个如果都定义，则有点过分了,因为如果有其中的两个，六个中剩下的四个也都可以被表示了。比如给定了等于和小于的谓词。则其他的谓词也是可以被表示的。Scala里, 所有的这些通过下面的特征声明，都可以被很好的捕捉到。

```scala
trait Ord {
  def < (that: Any): Boolean
  def <=(that: Any): Boolean =  (this < that) || (this == that)
  def > (that: Any): Boolean = !(this <= that)
  def >=(that: Any): Boolean = !(this < that)
}
```

这个定义既创造了一种新的类型Ord，它的作用类似java的Comparable接口，又在一个抽象的谓词上默认实现了其他的三个。而相等和不相等的谓词不会出现在这里，只是因为它们默认存在于所有对象中。

上面使用的Any类型是Scala中所有其他类型的父类。它可以被看作是java的Object的一个更一般的版本，因为它也是Int，Float这种基本类型的父类。

要使类的对象可以比较，需要定义谓词来测试相等和优先级，而它们被融合在上面的Ord了。举例而言，让我们定义一个日期类，它表示公历中的日期。这样的日期由一天、一个月和一年组成，我们都将其表示为整数。因此，我们定义日期类如下：

```scala
class Date(y: Int, m: Int, d: Int) extends Ord {
  def year = y
  def month = m
  def day = d
  override def toString(): String = year + "-" + month + "-" + day
}
```

重要的是在类名和参数后面的继承ORD的声明。它声明类Date继承了Ord这个特征。

然后，我们重新定义了从Object继承的equals方法，以便通过比较它们各自的字段来正确地比较日期。equals的默认实现是不可用的，因为默认比较的是java对象的指针。定义如下：

```scala
override def equals(that: Any): Boolean =
  that.isInstanceOf[Date] && {
    val o = that.asInstanceOf[Date]
    o.day == day && o.month == month && o.year == year
  }
```

该方法使用了预定义函数isInstanceOf和asInstanceOf。第一个isinstanceof，对应于java的instanceof操作，并返回true当且仅当它的对象是给定类型的一个实例。第二个asInstanceOf，对应于java的转换操作：如果对象是一个给定的类型，则将其视为对应的类型，否则抛出ClassCastException。

最后，定义的一个方法是测试优先级的谓词，如下所示。它使用另一种预定义的方法，即error，它使用给定的错误消息抛出异常。

```scala
def <(that: Any): Boolean = {
  if (!that.isInstanceOf[Date])
    error("cannot compare " + that + " and a Date")

  val o = that.asInstanceOf[Date]
  (year < o.year) ||
  (year == o.year && (month < o.month ||
                     (month == o.month && day < o.day)))
}
```
这就完成了日期类的定义。这个类的实例既是日期，也是可比较对象。此外，他们都定义了六个比较谓词：equals和<直接出现在日期类的定义，其他谓词从Ord特性继承而来。

当然，不止在这里，特性在其他情况下也很有用，但讨论它们的应用超出了本文档的长度范围。

## 范型

我们将在本文中探讨的Scala的最后一个特性是泛型。java程序员应该意识到在他们的语言中，泛型的缺失所带来的问题，而这个问题是java 1.5解决。

泛型是利用类型的参数化来写代码的能力。例如，为链表编写库的程序员面临着决定向列表元素提供哪种类型的问题。由于这个列表可以在许多不同的上下文中使用，所以不可能决定元素的类型必须是什么，例如，int，这将是完全任意的和过度限制的。

java程序员使用的Object，是所有类型的父类。然而，这个解决方案远远不是理想的，因为它对基本类型（int、长、浮点等）不起作用，这意味着程序员必须插入很多动态类型的转换。

Scala使得定义泛型类（和方法）来解决这个问题成为可能。让我们用一个尽可能简单的容器类的例子来检查这个例子：一个引用，它可以是空的，也可以指向某个类型的对象。

```scala
class Reference[T] {
  private var contents: T = _
  def set(value: T) { contents = value }
  def get: T = contents
}
```

类Reference的类型是参数化的，称为T，这是其元素的类型。该类型用于类的主体，如content变量的类型、set方法的参数以及get方法的返回类型。

上面的代码示例引入Scala中的变量，这不需要进一步解释。不过，这个变量的初始值_很有趣，这是一个默认值。此默认值为数值类型的0，布尔类型为false，Unit类型的()，所有对象类型为null。

要使用这个引用类，需要指定用于类型参数T的类型，即cell包含的元素的类型。例如，要创建和使用保存整数的cell，可以编写以下内容：

```scala
object IntegerReference {
  def main(args: Array[String]) {
    val cell = new Reference[Int]
    cell.set(13)
    println("Reference contains the half of " + (cell.get * 2))
  }
}
```

从这个例子中可以看出，在get方法作为整数之前不需要强制返回get方法返回的值。在一个特定的cell中也不可能存储任何整数，因为它被声明为持有整数。

## 结论

本文档简要介绍了Scala语言，并给出了一些基本示例。感兴趣的读者可以通过示例读取Scala文档，其中包含许多更高级的示例，并在需要时参考Scala语言规范。
