# The Coding Kata: FizzBuzzWhizz in Modern C++11

> Surprisingly, C++11 feels like a new language. -- Bjarne Stroustrup

至今，`C++`社区仍具有强大的生命力，尤其自`C++11`出现后获得了新生。`C++`到底存在什么样的魅力，让人如此痴狂呢？本文试图阐述`C++`的设计思维，并揭示`C++`内在的设计本质；最后通过`FizzBuzzWhizz`的设计和实现一展`C++11`的风采。

## 设计思维

### 简单

> Make Simple Tasks Simple.

- Keep Simple Things Simple
- Don't Make Complex Things Unnecessarily Complex
- Don't Make Things Impossible

**Constraint**: Don't Cacrifice Performance.

### 平衡

> Don’t Over Abstract

- Abstraction
- Performance

`C++`试图找到「抽象」和「性能」的平衡点，并将抉择的自由留给了程序员。

### 自由

- No One Size Fits All
- Multi-Paradigm

世界是多样性的，`C++`多范式的设计思维赋予了程序员极大的自由度和灵活性。

### 友好
 
- More and More Expert Friendly

`C++`越来越变得更加友好，这种友好性对于专家感触将更加深刻。
 
## FizzBuzzWhizz

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

借助`C++11`增强了的「类型系统」能力，可抛弃掉很多重复的「样板代码」，使得设计更加简单、漂亮。此外，`C++11`构造`DSL`的能力也相当值得称赞，而且非常直接，简单。

```cpp
FIXTURE(FizzBuzzWhizzSpec) {
  Rule spec = makeSpec();

  Rule makeSpec() {
    auto r1_3 = atom(times(3), to("Fizz"));
    auto r1_5 = atom(times(5), to("Buzz"));
    auto r1_7 = atom(times(7), to("Whizz"));

    auto r1 = anyof( { r1_3, r1_5, r1_7 });

    auto r2 = anyof({
      allof({ r1_3, r1_5, r1_7 }),
      allof({ r1_3, r1_5 }),
      allof({ r1_3, r1_7 }),
      allof({ r1_5, r1_7 })
    });

    auto r3 = atom(contains(3), to("Fizz"));
    auto rd = atom(always(true), nop());

    return anyof({ r3, r2, r1, rd });
  }

  TEST("fizz buzz whizz") {
    rule(3, "Fizz");
    rule(5, "Buzz");
    rule(7, "Whizz");
    rule(3 * 5 * 7, "FizzBuzzWhizz");
    rule(3 * 5, "FizzBuzz");
    rule(3 * 7, "FizzWhizz");
    rule((5 * 7) * 2, "BuzzWhizz");
    rule(13, "Fizz");
    rule(35 /* 5*7 */, "Fizz");
    rule(2, "2");
  }

  void rule(int n, const std::string& expect) {
    ASSERT_THAT(spec(n), eq(expect));
  }
};
``` 

### 匹配器：`Matcher`

`Matcher`是一个「一元函数」，入参为`int`，返回值为`bool`，是一种典型的「谓词」。设计采用了`C++11`函数式的风格，并利用强大的「闭包」能力，让代码更加简洁，并富有表达力。

从`OO`的角度看，`always`是一种典型的`Null Object`。

```cpp
using Matcher = std::function<bool(int)>;

Matcher times(int times) {
  return [=](auto n) { 
    return n % times == 0; 
  };
}

Matcher contains(int num) {
  return [=](auto n) { 
    return toString(n).find(toString(num)) != string::npos; 
  };
}

Matcher always(bool value) {
  return [=](auto) { 
    return value; 
  };
}
```

### 执行器：`Action`

`Action`也是一个「一元函数」，入参为`int`，返回值为`std::string`，其本质就是定制常见的`map`操作，将定义域映射到值域。

```cpp
using Action = std::function<std::string(int)>;

Action to(const std::string& str) {
  return [=](auto) { 
    return str; 
  };
}

Action nop() {
  return [](auto n) { 
    return stdext::toString(n); 
  };
}
```

### 规则：`Rule`

> Composition Everywhere

`Rule`是`FizzBuzzWhizz`最核心的抽象，也是设计的灵魂所在。从语义上`Rule`分为`2`种基本类型，并且两者之间形成了优美的、隐式的「树型」结构，体现了「组合式设计」的强大威力。

- `Atomic`
- `Compositions: anyof, allof`

`Rule`是一个「一元函数」，入参为`int`，返回值为`std::string`。

```cpp
using Rule = std::function<bool(int, RuleResult&)>;

Rule atom(const Matcher& matcher, const Action& action) {
  return [=](auto n) {
    return matcher(n) ? action(n) : "";
  };
}

Rule anyof(const std::vector<Rule>& rules) {
  return [=](auto n) {
    auto r = std::find_if(rules.begin(), rules.end(),
      [&](const auto& r) { return !r(n).empty(); });
    return r != std::end(rules) ? (*r)(n) : "";
  };
}

Rule allof(const std::vector<Rule>& rules) {
  return [=](auto n) {
    return std::accumulate(rules.begin(), rules.end(), std::string(""),
      [=](const auto& result, const auto& rule) {
        return result + rule(n);
    });
  };
}
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **Scala参考实现：** [https://codingstyle.cn/topics/99](https://codingstyle.cn/topics/99)
- **Java参考实现：** [https://codingstyle.cn/topics/100](https://codingstyle.cn/topics/100)


