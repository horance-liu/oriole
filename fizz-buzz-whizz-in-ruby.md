# The Coding Kata: FizzBuzzWhizz in Ruby

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

接下来我将使用`Ruby`尝试`FizzBuzzWhizz`问题的设计和实现。

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

```ruby
describe "fizz buzz whizz" do
  include Matcher
  include Action
  include Rule
  
  def spec
    r1_3 = atom(times(3), to("Fizz"))
    r1_5 = atom(times(5), to("Buzz"))
    r1_7 = atom(times(7), to("Whizz"))

    r1 = anyof([r1_3, r1_5, r1_7])

    r2 = anyof([
      allof([r1_3, r1_5, r1_7]),
      allof([r1_3, r1_5]),
      allof([r1_3, r1_7]),
      allof([r1_5, r1_7])])

    r3 = atom(contains(3), to("Fizz"))
    rd = atom(always(true), nop);

    anyof([r3, r2, r1, rd])
  end

  [ 
    [3, 'Fizz' ],
    [5, 'Buzz' ],
    [7, 'Whizz' ],
    [3*5, 'FizzBuzz' ],
    [3*7, 'FizzWhizz' ],
    [5*7*2, 'BuzzWhizz' ],
    [3*5*7, 'FizzBuzzWhizz' ],
    [13, 'Fizz' ],
    [35, 'Fizz' ],
    [2,  '2' ]
  ].each do |n, expect|
  	it "#{n} -> #{expect}" do
      expect(spec.call(n)).to eq expect
    end
  end
end
``` 

### 匹配器：`Matcher`

`Matcher`是一个「一元函数」，入参为`Int`，返回值为`Boolean`，是一种典型的「谓词」。从`OO`的角度看，`always`是一种典型的`Null Object`。

```ruby
module Matcher
  def times(n)
    ->(x) { x % n == 0 }
  end

  def contains(n)
    ->(x) { x.to_s.include?(n.to_s) }
  end

  def always(b)
    ->(x) { b }
  end
end
```

### 执行器：`Action`

`Action`也是一个「一元函数」，入参为`Int`，返回值为`String`，其本质就是定制常见的`map`操作，将定义域映射到值域。

```ruby
module Action
  def to(str)
    ->(n) { str }
  end

  def nop 
    ->(x) { x.to_s }
  end
end
```

### 规则：`Rule`

> Composition Everywhere

`Rule`是`FizzBuzzWhizz`最核心的抽象，也是设计的灵魂所在。从语义上`Rule`分为`2`种基本类型，并且两者之间形成了优美的、隐式的「树型」结构，体现了「组合式设计」的强大威力。

- `Atomic`
- `Compositions: anyof, allof`

`Rule`是一个「一元函数」，入参为`Int`，返回值为`String`。

```ruby
module Rule
  def atom(matcher, action)
    ->(n) { matcher.call(n) ? action.call(n) : '' } 
  end

  def allof(rules)
    ->(n) { strings(rules, n).join }
  end

  def anyof(rules)
    lambda do |n|
      result = strings(rules, n).find { |s| !s.empty? }
      result != nil ? result : ''
    end
  end

  def strings(rules, n)
    rules.map { |rule| rule.call(n) }
  end

  private :strings
end
```

### 源代码

- **Github:** [https://github.com/horance-liu/fizz-buzz-whizz](https://github.com/horance-liu/fizz-buzz-whizz)
- **C++11参考实现：** [https://codingstyle.cn/topics/97](https://codingstyle.cn/topics/97)
- **Java参考实现：** [https://codingstyle.cn/topics/100](https://codingstyle.cn/topics/100)
- **Scala参考实现：** [https://codingstyle.cn/topics/100](https://codingstyle.cn/topics/99)

