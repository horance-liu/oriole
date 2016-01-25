# The Coding Kata: FizzBuzzWhizz in Java8
 
> Functional programming leads to deep insights into the nature of computation. -- Martin Odersky
 
## 形式化

`FizzBuzzWhizz`详细描述请自行查阅相关资料。此处以`3, 5, 7`为例，形式化地描述一下问题。

```bash
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

```cpp
rule ::= atom | 
         allof(rule(1), rule(2), ..., rule(n)) | 
         anyof(rule(1), rule(2), ..., rule(n))

atom ::= (matcher, action) -> bool
matcher ::= (int) -> bool
action ::= (int) -> string
```

### 测试用例

借助`Java8`增强了的「函数式」能力，可抛弃掉很多重复的「样板代码」，使得设计更加简单、漂亮。此外，`Java8`构造`DSL`的能力也相当值得称赞，非常直接，简单。

```java
public class RuleTest {
  private Rule spec = makeSpec();

  private static Rule makeSpec() {
    Rule r1_3 = atom(times(3), to("Fizz"));
    Rule r1_5 = atom(times(5), to("Buzz"));
    Rule r1_7 = atom(times(7), to("Whizz"));

    Rule r1 = anyof(r1_3, r1_5, r1_7);
    
    Rule r2 = anyof(allof(r1_3, r1_5, r1_7), 
                    allof(r1_3, r1_5), 
                    allof(r1_3, r1_7), 
                    allof(r1_5, r1_7));
    
    Rule r3 = atom(contains(3), to("Fizz"));
    Rule rd = atom(always(true), nop());

    return anyof(r3, r2, r1, rd);
  }
  
  private void rule(int n, String expect) {
    RuleResult result = new RuleResult();
    spec.apply(n, result);
    assertThat(result.toString(), equalTo(expect));
  }
  
  @Test
  public void r1_3() {
    rule(3, "Fizz");
  }

  @Test
  public void r1_5() {
    rule(5, "Buzz");
  }

  @Test
  public void r1_7() {
    rule(7, "Whizz");
  }

  @Test
  public void r2_1() {
    rule(3 * 5 * 7, "FizzBuzzWhizz");
  }

  @Test
  public void r2_2() {
    rule(3 * 5, "FizzBuzz");
  }

  @Test
  public void r2_3() {
    rule(3 * 7, "FizzWhizz");
  }

  @Test
  public void r2_4() {
    rule((5 * 7) * 2, "BuzzWhizz");
  }

  @Test
  public void r3() {
    rule(13, "Fizz");
  }
  
  @Test
  public void priority_of_r3_greater_than_r2_4() {
    rule(35 /* 5*7 */, "Fizz");
  }

  @Test
  public void rd() {
    rule(2, "2");
  }
}
``` 

### 匹配器：`Matcher`

`Matcher`是一个「一元函数」，入参为`Int`，返回值为`Boolean`，是一种典型的「谓词」。从`OO`的角度看，`always`是一种典型的`Null Object`。

```java
@FunctionalInterface
public interface Matcher {
  boolean matches(int n);

  static Matcher times(int times) {
    return n -> n % times == 0;
  }

  static Matcher contains(int num) {
    return n -> Integer.toString(n).contains(Integer.toString(num));
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
    return n -> Integer.toString(n);
  }
}
```

### 规则：`Rule`

> Composition Everywhere

`Rule`是`FizzBuzzWhizz`最核心的抽象，也是设计的灵魂所在。从语义上`Rule`分为`2`种基本类型，并且两者之间形成了优美的、隐式的「树型」结构，体现了「组合式设计」的强大威力。

- `Atomic`
- `Compositions: anyof, allof`

`Rule`是一个「二元函数」，入参为`(Int, RuleResult)`，返回值为`Boolean`。其中，`allof, anyof`使用了`Java8 Stream`的`API`。

```java
@FunctionalInterface
public interface Rule {
  boolean apply(int n, RuleResult rr);
}
```

```java
public final class Rules {
  public static Rule atom(Matcher matcher, Action action) {
    return (n, rr) -> rr.collect(matcher.matches(n), action.to(n));
  }

  public static Rule anyof(Rule... rules) {
    return (n, rr) -> Arrays.asList(rules)
        .stream()
        .anyMatch(rule -> rule.apply(n, rr));
  }
 
  public static Rule allof(Rule... rules) {
    return (n, rr) -> {
      RuleResult result = new RuleResult();
      return rr.collect(allMatch(rules).apply(n, result), result);
    };
  }
  
  private static Rule allMatch(Rule... rules) {
    return (n, rr) -> Arrays.asList(rules)
        .stream()
        .allMatch(rule -> rule.apply(n, rr));
  }
  
  private Rules() {
  }
}
```

### 聚集参数：`RuleResult`

为了取得性能的优势，`RuleResult`充当`Collect Parameter`的角色，是一种常用的实现模式。

```java
package fizz.buzz.whizz;

public class RuleResult {
  public RuleResult() {
    this("");
  }

  public RuleResult(String str) {
    buff = new StringBuilder(str);
  }

  public boolean collect(boolean matched, String str) {
    if (matched) buff.append(str);
    return matched;
  }

  public boolean collect(boolean matched, RuleResult rr) {
    return collect(matched, rr.toString());
  }

  @Override
  public String toString() {
    return buff.toString();
  }

  private StringBuilder buff;
}
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **C++11参考实现：** [https://codingstyle.cn/topics/97](https://codingstyle.cn/topics/97)
- **Scala参考实现：** [https://codingstyle.cn/topics/99](https://codingstyle.cn/topics/99)


