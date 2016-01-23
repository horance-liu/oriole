# FizzBuzzWhizz in Modern C++

> Surprisingly, C++11 feels like a new language - the pieces just fit together better. -- Bjarne Stroustrup
## Fizz Buzz Whizz

你是一名体育老师，在某次课距离下课还有五分钟时，你决定搞一个游戏。游戏的规则如下：

1. 首先说出三个不同的特殊数，要求必须是个位数，比如`3、5、7`。 
2. 让所有学生拍成一队，然后按顺序报数。
3. 如果所报数字是第一个特殊数（3）的倍数，那么不能说该数字，而要说Fizz；如果所报数字是第二个特殊数（5）的倍数，那么要说Buzz；如果所报数字是第三个特殊数（7）的倍数，那么要说Whizz。
4. 如果所报数字同时是两个或三个特殊数的倍数情况下，也要特殊处理。例如第一个特殊数和第二个特殊数的倍数，那么不能说该数字，而是要说FizzBuzz。以此类推，如果同时是三个特殊数的倍数，那么要说FizzBuzzWhizz。 
5. 如果所报数字包含了**第一个**特殊数，那么也不能说该数字，而是要说相应的单词。比如本例中第一个特殊数是3，那么要报13的同学应该说Fizz。如果数字中包含了第一个特殊数，那么忽略规则3和规则4，比如要报35的同学只报Fizz，不报BuzzWhizz。

### 测试用例

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

### `Matcher`

```cpp
using Matcher = std::function<bool(int)>;

Matcher times(int times) {
  return [=](auto n) { 
    return n % times == 0; 
  };
}

Matcher contains(int num) {
  return [=](auto n) { 
    return toString(n).find(toString(num)) != std::string::npos; 
  };
}

Matcher always(bool value) {
  return [=](auto) { 
    return value; 
  };
}
```

### `Action`

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

### `Rule`

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

### Github

－ https://

