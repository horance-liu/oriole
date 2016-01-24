# The Coding Kata: FizzBuzzWhizz in Scala
 
> Functional programming leads to deep insights into the nature of computation. -- Martin Odersky
 
## 形式化

`FizzBuzzWhizz`详细描述请自行查阅相关资料。此处以`3, 5, 7`为例，形式化地描述一下问题。

```scala
def times(n: Int) = (x: Int) => x % n == 0
def contains(n: Int) = (x: Int) => x.toString.contains(n.toString)
def to(str: String) = (x: Int) => str

val r1_3 = times(3) -> to("Fizz")
val r1_5 = times(5) -> to("Buzz")
val r1_7 = times(7) -> to("Whizz")

val r1 = r1_3 || r1_5 || r1_7

val r2 = (r1_3 && r1_5 && r1_7) ||
         (r1_3 && r1_5) || 
         (r1_3 && r1_7) || 
         (r1_5 && r1_7) || 
         
val r3 = atom(contains(3), to("Fizz"))
val rd = atom(always(true), nop());

val spec = r3 || r2 || r1 || rd 
```

为了简化问题的描述，使用`DSL`形式化地描述问题。接下来我将使用`Scala`尝试`FizzBuzzWhizz`问题的设计和实现。

### 语义模型

从上面的形式化描述，可以很容易地得到`FizzBuzzWhizz`问题的语义模型。

```cpp
rule ::= atom | 
         allof(rule(1), rule(2), ..., rule(n)) | 
         anyof(rule(1), rule(2), ..., rule(n))

atom ::= (matcher, action) -> bool
matcher ::= (int) -> bool
action ::= (int) -> string
```

### 测试用例

借助`Scala`强大的「类型系统」能力，可抛弃掉很多重复的「样板代码」，使得设计更加简单、漂亮。此外，`Scala`构造`DSL`的能力也相当值得称赞，非常直接，简单。

```scala
class RuleSpec extends FunSpec {
  def spec = {
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
    val rd = atom(always(true), nop());

    anyof(r3, r2, r1, rd)
  }

  def rule(n: Int, expected: String) = {
    val result = new RuleResult
    spec(n, result)
    result.toString should be (expected)
  }

  describe("r1") {
    it("times(3) -> Fizz") {
      rule(3, "Fizz")
    }

    it("times(5) -> Buzz") {
      rule(5, "Buzz");
    }

    it("times(7) -> Whizz") {
      rule(7, "Whizz");
    }
  }

  describe("r2") {
    it("times(3) && times(5) && times(7) -> FizzBuzzWhizz") {
      rule(3 * 5 * 7, "FizzBuzzWhizz");
    }

    it("times(3) && times(5) -> FizzBuzz") {
      rule(3 * 5, "FizzBuzz");
    }

    it("times(3) && times(7) -> FizzWhizz") {
      rule(3 * 7, "FizzWhizz");
    }

    it("times(5) && times(7) -> BuzzWhizz") {
      rule((5 * 7) * 2, "BuzzWhizz");
    }
  }

  describe("r3") {
    it("contains(3) -> Fizz") {
      rule(13, "Fizz");
    }

    it("the priority of contains(3) is highest") {
      rule(35 /* 5*7 */ , "Fizz");
    }
  }

  describe("rd") {
    it("others -> others") {
      rule(2, "2");
    }
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
  def nop(): Action = _.toString
}
```

### 规则：`Rule`

> Composition Everywhere

`Rule`是`FizzBuzzWhizz`最核心的抽象，也是设计的灵魂所在。从语义上`Rule`分为`2`种基本类型，并且两者之间形成了优美的、隐式的「树型」结构，体现了「组合式设计」的强大威力。

- `Atomic`
- `Compositions: anyof, allof`

`Rule`是一个「二元函数」，入参为`(Int, RuleResult)`，返回值为`Boolean`。其中，`def atom(matcher: => Matcher, action: => Action)`的入参使用`Pass By Name`的惰性求值的特性。

```scala
object Rules {
  type Rule = (Int, RuleResult) => Boolean

  def atom(matcher: => Matcher, action: => Action): Rule =
    (n, rr) => rr.collect(matcher(n), action(n))

  def anyof(rules: Rule*): Rule =
    (n, rr) => rules.exists(_(n, rr))

  def allof(rules: Rule*): Rule =
    (n, rr) => {
      val result = new RuleResult
      rr.collect(rules.forall(_(n, result)), result)
    }
}
```

### 聚集参数：`RuleResult`

为了取得性能的优势，`RuleResult`充当`Collect Parameter`的角色，是一种常用的实现模式。

```scala
class RuleResult(result: StringBuilder) {
  def this() = this(new StringBuilder(""))

  def collect(matched: Boolean, rr: RuleResult): Boolean =
    collect(matched, rr.toString)

  def collect(matched: Boolean, str: String): Boolean = {
    if (matched) result.append(str)
    matched
  }

  override def toString = result.toString
}
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **C++11参考实现：** [https://codingstyle.cn/topics/97](https://codingstyle.cn/topics/97)
- **Java参考实现：** [https://codingstyle.cn/topics/100](https://codingstyle.cn/topics/100)

