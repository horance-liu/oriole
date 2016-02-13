# The Coding Kata: FizzBuzzWhizz in Java8
 
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

借助`Java8`增强了的「函数式」能力，可抛弃掉很多重复的「样板代码」，使得设计更加简单、漂亮。此外，`Java8`构造`DSL`的能力也相当值得称赞，非常直接，简单。测试用例此处选择`Spock`(基于`Groovy`语言)，可提高用例的可读性和可维护性。

```groovy
class RuleSpec extends Specification {
  private static def spec() {
    Rule r1_3 = atom(times(3), to("Fizz"))
    Rule r1_5 = atom(times(5), to("Buzz"))
    Rule r1_7 = atom(times(7), to("Whizz"))

    Rule r1 = anyof(r1_3, r1_5, r1_7)

    Rule r2 = anyof(allof(r1_3, r1_5, r1_7),
        allof(r1_3, r1_5),
        allof(r1_3, r1_7),
        allof(r1_5, r1_7))

    Rule r3 = atom(contains(3), to("Fizz"))
    Rule rd = atom(always(true), nop())

    anyof(r3, r2, r1, rd)
  }

  def "fizz buzz whizz"() {
    expect:
    spec().apply(n) == expect

    where:
    n            | expect
    3            | "Fizz"
    5            | "Buzz"
    7            | "Whizz"
    3 * 5 * 7    | "FizzBuzzWhizz"
    3 * 5        | "FizzBuzz"
    3 * 7        | "FizzWhizz"
    (5 * 7) * 2  | "BuzzWhizz"
    13           | "Fizz"
    35 /* 5*7 */ | "Fizz"  /* not "BuzzWhizz" */
    2            | "2"
  }
}
``` 

### 匹配器：`Matcher`

`Matcher`是一个「一元函数」，入参为`Int`，返回值为`Boolean`，是一种典型的「谓词」。从`OO`的角度看，`always`是一种典型的`Null Object`。

```java
import static java.lang.String.valueOf;

@FunctionalInterface
public interface Matcher {
  boolean matches(int n);

  static Matcher times(int n) {
    return x -> x % n == 0;
  }

  static Matcher contains(int n) {
    return x -> valueOf(x).contains(valueOf(n));
  }

  static Matcher always(boolean bool) {
    return n -> bool;
  }
}
```

### 执行器：`Action`

`Action`也是一个「一元函数」，入参为`Int`，返回值为`String`，其本质就是定制常见的`map`操作，将定义域映射到值域。

```java
@FunctionalInterface
public interface Action {
  String to(int n);

  static Action to(String str) {
    return n -> str;
  }

  static Action nop() {
    return n -> String.valueOf(n);
  }
}
```

### 规则：`Rule`

> Composition Everywhere

`Rule`是`FizzBuzzWhizz`最核心的抽象，也是设计的灵魂所在。从语义上`Rule`分为`2`种基本类型，并且两者之间形成了优美的、隐式的「树型」结构，体现了「组合式设计」的强大威力。
              - `Atom`
- `Compositions: anyof, allof`

`Rule`是一个「二元函数」，入参为`(Int)`，返回值为`String`。

```java
@FunctionalInterface
public interface Rule {
  String apply(int n);
}
```

`Rules`为一个工厂类，用于生产各种`Rule`。其中，`allof, anyof`使用了`Java8 Stream`的`API`。

```java
public final class Rules {
  public static Rule atom(Matcher matcher, Action action) {
    return n -> matcher.matches(n) ? action.to(n) : "";
  }

  public static Rule anyof(Rule... rules) {
    return n ->sstream(n, rules)
        .filter(s -> !s.isEmpty())
        .findFirst()
        .orElse("");
  }

  public static Rule allof(Rule... rules) {
    return n -> sstream(n, rules)
        .collect(joining());
  }

  private static Stream<String> sstream(int n, Rule[] rules) {
    return Arrays.asList(rules).stream()
        .map(r -> r.apply(n));
  }

  private Rules() {
  }
}
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **C++11参考实现：** [https://codingstyle.cn/topics/97](https://codingstyle.cn/topics/97)
- **Scala参考实现：** [https://codingstyle.cn/topics/99](https://codingstyle.cn/topics/99)


