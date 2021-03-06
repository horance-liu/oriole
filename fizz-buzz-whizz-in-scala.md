# The Coding Kata: FizzBuzzWhizz in Scala
 
> Functional programming leads to deep insights into the nature of computation. -- Martin Odersky
 
## 形式化

`FizzBuzzWhizz`详细描述请自行查阅相关资料。此处以`3, 5, 7`为例，形式化地描述一下问题。

```scala
r1
- times(3) -> Fizz
- times(5) -> Buzz
- times(7) -> Whizz
r2
- times(3) && times(5) && times(7) -> FizzBuzzWhizz
- times(3) && times(5) -> FizzBuzz
- times(3) && times(7) -> FizzWhizz
- times(5) && times(7) -> BuzzWhizz
r3
- contains(3) -> Fizz
- the priority of contains(3) is highest
rd
- others -> others
```

接下来我将使用`Scala`尝试`FizzBuzzWhizz`问题的设计和实现。

### 语义模型

从上面的形式化描述，可以很容易地得到`FizzBuzzWhizz`问题的语义模型。

```scala
Rule: (Int) -> String
Matcher: (Int) -> Boolean
Action: (Int) -> String
```

其中，`Rule`存在三种基本的类型：

```scala
Rule ::= atom | allof | anyof
```

三者之间构成了「树型」结构。

```scala
atom: (Matcher, Action) -> String
allof(rule1, rule2, ...): rule1 && rule2 && ... 
anyof(rule1, rule2, ...): rule1 || rule2 || ... 
```

### 测试用例

借助`Scala`强大的「类型系统」能力，可抛弃掉很多重复的「样板代码」，使得设计更加简单、漂亮。此外，`Scala`构造`DSL`的能力也相当值得称赞，非常直接，简单。

```scala
import org.scalatest._
import prop._

class RuleSpec extends PropSpec with TableDrivenPropertyChecks with Matchers {
  val spec = {
    val r1_3 = atom(times(3), to("Fizz"))
    val r1_5 = atom(times(5), to("Buzz"))
    val r1_7 = atom(times(7), to("Whizz"))

    val r1 = anyof(r1_3, r1_5, r1_7)

    val r2 = anyof(
      allof(r1_3, r1_5, r1_7),
      allof(r1_3, r1_5),
      allof(r1_3, r1_7),
      allof(r1_5, r1_7))

    val r3 = atom(contains(3), to("Fizz"))
    val rd = atom(always(true), nop);

    anyof(r3, r2, r1, rd)
  }

  val specs = Table(
    ("n", "expect"),
    (3, "Fizz"),
    (5, "Buzz"),
    (7, "Whizz"),
    (3 * 5, "FizzBuzz"),
    (3 * 7, "FizzWhizz"),
    ((5 * 7) * 2, "BuzzWhizz"),
    (3 * 5 * 7,   "FizzBuzzWhizz"),
    (13, "Fizz"),
    (35, "Fizz"),  // 35 > 5*7
    (2,  "2")
  )

  property("fizz buzz whizz") {
    forAll(specs) { spec(_) should be (_) }
  }
}
``` 

### 匹配器：`Matcher`

`Matcher`是一个「一元函数」，入参为`Int`，返回值为`Boolean`，是一种典型的「谓词」。从`OO`的角度看，`always`是一种典型的`Null Object`。

```scala
object Matchers {
  type Matcher = Int => Boolean

  def times(n: Int): Matcher = _ % n == 0
  def contains(n: Int): Matcher = _.toString.contains(n.toString)
  def always(bool: Boolean): Matcher = _ => bool
}
```

### 执行器：`Action`

`Action`也是一个「一元函数」，入参为`Int`，返回值为`String`，其本质就是定制常见的`map`操作，将定义域映射到值域。

```scala
object Actions {
  type Action = Int => String

  def to(str: String): Action = _ => str
  def nop: Action = _.toString
}
```

### 规则：`Rule`

> Composition Everywhere

`Rule`是`FizzBuzzWhizz`最核心的抽象，也是设计的灵魂所在。从语义上`Rule`分为`2`种基本类型，并且两者之间形成了优美的、隐式的「树型」结构，体现了「组合式设计」的强大威力。

- `Atomic`
- `Compositions: anyof, allof`

`Rule`是一个「一元函数」，入参为`Int`，返回值为`String`。其中，`def atom(matcher: => Matcher, action: => Action)`的入参使用`Pass By Name`的惰性求值的特性。

```scala
object Rules {
  type Rule = (Int) => String

  def atom(matcher: => Matcher, action: => Action): Rule =
    n => if (matcher(n)) action(n) else ""

  def anyof(rules: Rule*): Rule =
    n => rules.map(_(n))
      .filterNot(_.isEmpty)
      .headOption
      .getOrElse("")

  def allof(rules: Rule*): Rule =
    n => rules.foldLeft("") { _ + _(n) }
}
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **C++11参考实现：** [https://codingstyle.cn/topics/97](https://codingstyle.cn/topics/97)
- **Java参考实现：** [https://codingstyle.cn/topics/100](https://codingstyle.cn/topics/100)


