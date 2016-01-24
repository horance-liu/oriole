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
def times(n: Int) = (x: Int) => x % n == 0
def contains(n: Int) = (x: Int) => x.toString.contains(n.toString)
def always(bool: Boolean) = (x: Int) => bool

def to(str: String) = (x: Int) => str
def nop() = (x: Int) => x.toString

def r1_3 = atom(times(3), to("Fizz"))
def r1_5 = atom(times(5), to("Buzz"))
def r1_7 = atom(times(7), to("Whizz"))

def r1 = r1_3 || r1_5 || r1_7

def r2 = (r1_3 && r1_5) || 
         (r1_3 && r1_7) || 
         (r1_5 && r1_7) || 
         (r1_3 && r1_5 && r1_7)

def r3 = atom(contains(3), to("Fizz"))
def rd = atom(always(true), nop());

def spec = r3 || r2 || r1 || rd 
```

为了简化问题的描述，此处使用`Scala`语言设计的`DSL`来描述，并可作为`C++11`设计的`DSL`提供比较的样本。如有感兴趣的同学，可自行实现`Scala`的版本。

接下来我将使用`C++11`尝试`FizzBuzzWhizz`问题的设计和实现，你会发现其简单程度及其表达力，可与`Scala`不分伯仲。

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
      allof( { r1_3, r1_5, r1_7 } ),
      allof( { r1_3, r1_5 } ),
      allof( { r1_3, r1_7 } ),
      allof( { r1_5, r1_7 } )
    });

    auto r3 = atom(contains(3), to("Fizz"));
    auto rd = atom(always(true), nop());

    return anyof( { r3, r2, r1, rd });
  }
  
  void rule(int n, const std::string& expect) {
    RuleResult result;
    spec(n, result);
    ASSERT_THAT(result.toString(), eq(expect));
  }
  
  TEST("times(3) -> Fizz") {
    rule(3, "Fizz");
  }

  TEST("times(5) -> Buzz") {
    rule(5, "Buzz");
  }

  TEST("times(7) -> Whizz") {
    rule(7, "Whizz");
  }

  TEST("times(3) && times(5) && times(7) -> FizzBuzzWhizz") {
    rule(3 * 5 * 7, "FizzBuzzWhizz");
  }

  TEST("times(3) && times(5) -> FizzBuzz") {
    rule(3 * 5, "FizzBuzz");
  }

  TEST("times(3) && times(7) -> FizzWhizz") {
    rule(3 * 7, "FizzWhizz");
  }

  TEST("times(5) && times(7) -> BuzzWhizz") {
    rule((5 * 7) * 2, "BuzzWhizz");
  }

  TEST("contains(3) -> Fizz") {
    rule(13, "Fizz");
  }

  TEST("the priority of contains(3) is highest") {
    rule(35 /* 5*7 */, "Fizz");
  }

  TEST("others -> others") {
    rule(2, "2");
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

`Rule`是一个「二元函数」，入参为`(int, RuleResult&)`，返回值为`bool`。另外，为了消除`allof, anyof`之间存在重复设计，`combine`自然地被提取出来了。

```cpp
using Rule = std::function<bool(int, RuleResult&)>;

Rule atom(Matcher matcher, Action action) {
  return [=](auto n, auto& rr) { 
    return rr.collect(matcher(n), action(n)); 
  };
}

namespace {
  Rule combine(const std::vector<Rule>& rules, bool shortcut) {
    return [=](auto n, auto& rr) {
      for(auto& rule: rules)
        if (rule(n, rr) == shortcut)
          return shortcut;
      return !shortcut;
    };
  }
}

Rule anyof(const std::vector<Rule>& rules) {
  return combine(rules, true);
}

Rule allof(const std::vector<Rule>& rules) {
  return [=](auto n, auto& rr) {
    RuleResult result;
    return rr.collect(combine(rules, false)(n, result), result);
  };
}
```

### 聚集参数：`RuleResult`

为了取得性能的优势，`RuleResult`充当`Collect Parameter`的角色，是一种常用的实现模式。

```cpp
struct RuleResult {
  RuleResult(const std::string& = "") : result(result) {
  }

  bool collect(bool matched, const RuleResult& rr) {
    return collect(matched, rr.result);
  }

  bool collect(bool matched, const std::string& str) {
    if (matched) result += str;
    return matched;
  }

  const std::string& toString() const {
    return result;
  }

private:
  std::string result;
};
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **Scala参考实现：** [https://codingstyle.cn/topics/99](https://codingstyle.cn/topics/99)
- **Java参考实现：** [https://codingstyle.cn/topics/100](https://codingstyle.cn/topics/100)


