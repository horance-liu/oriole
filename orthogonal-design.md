# 正交设计

> Design is there to enable you to keep changing the software easily in the long term.  -- Kent Beck.

## 设计是什么

正如`Kent Beck`所说，软件设计是为了「长期」更加容易地适应未来的变化。正确的软件设计方法是为了长期地、更好更快、更容易地实现软件价值的交付。

## 软件设计的目标

软件设计就是为了完成如下目标，其可验证性、重要程度依次减低。

- 实现功能
- 易于重用
- 易于理解
- 没有冗余

### 实现功能

实现功能的目标压倒一起，这也是软件设计的首要标准。如何判定系统功能的完备性呢？通过所有测试用例。

从`TDD`的角度看，测试用例就是对需求的阐述，是一个闭环的反馈系统，保证其系统的正确性；及其保证设计的合理性，恰如其分，不多不少；当然也是理解系统行为最重要的依据。

### 易于理解

好的设计应该能让其他人也能容易地理解，包括系统的行为，业务的规则。那么，什么样的设计才算得上易于理解的呢？

- `Clean Code`
- `Implement Patterns`
- `Idioms`

### 没有冗余

没有冗余的系统是最简单的系统，恰如其分的系统，不做任何过度设计的系统。

- `Dead Code`
- `YAGNI: You Ain't Gonna Need It`
- `KISS: Keep it Simple, Stupid`

### 易于重用

易于重用的软件结构，使得其应对变化更具弹性；可被容易地修改，具有更加适应变化的能力。

最理想的情况下，所有的软件修改都具有局部性。但现实并非如此，软件设计往往需要花费很大的精力用于依赖的管理，让组件之间的关系变得清晰、一致、漂亮。

那么软件设计的最高准则是什么呢？「高内聚、低耦合」原则是提高可重用性的最高原则。为了实现高内聚，低耦合的软件设计，袁英杰提出了「正交设计」的方法论。

## 正交设计

「正交」是一个数学概念：所谓正交，就是指两个向量的内积为零。简单的说，就是这两个向量是垂直的。在一个正交系统里，沿着一个方向的变化，其另外一个方向不会发生变化。为此，`Bob`大叔将「职责」定义为「变化的原因」。

「正交性」，意味着更高的内聚，更低的耦合。为此，正交性可以用于衡量系统的可重用性。那么，如何保证设计的正交性呢？袁英杰提出了「正交设计的四个基本原则」，简明扼要，道破了软件设计的精髓所在。

### 正交设计原则

- 消除重复
- 分离关注点
- 缩小依赖范围
- 向稳定的方向依赖

### 实战

> **需求1：** 存在一个学生的列表，查找一个年龄等于`18`岁的学生

#### 快速实现

```java
public static Student findByAge(Student[] students) {
  for (int i=0; i<students.length; i++) {
    if (students[i].getAge() == 18) {
        return students[i];
    }
  }
  return null;
}
```

为了阐述设计的整个思考，及其重构的过程，揭示软件设计过程的真实样貌，很多设计是故意地引入坏味道。例如，上述实现存在很多代码的坏味道：

- 缺乏弹性参数类型：只支持数组类型，`List, Set`都被拒之门外；
- 容易出错：操作数组下标，往往引入不经意的错误；
- 幻数：硬编码，将算法与配置高度耦合；
- 返回`null`：再次给用户打开了犯错的大门；

#### 使用`for-each`

按照最小依赖的原则，现屏蔽掉数组下标的细节，使用`for-each`降低错误发生的可能性。

```java
public static Student findByAge(Student[] students) {
  for (Student s : students) {
    if (s.getAge() == 18) {
        return s;
    }
  }
  return null;
}
```

> **需求2：** 查找一个名字为`horance`的学生

#### 重复设计

`Copy-Paste`是最快的实现方法，但会产生重复的设计。

```java
public static Student findByName(Student[] students) {
  for (Student s : students) {
    if (s.getName().equals("horance")) {
        return s;
    }
  }
  return null;
}
```

为了消除重复，可以将查找的算法与比较的准则这两个关注点进行分离。

#### 抽象准则

首先将比较的准则进行抽象化，让其独立变化。

```java
public interface StudentPredicate {
  boolean test(Student s);
}
```

将各个「变化的原因」对象化，为此建立了两个简单的算子。

```java
public class AgePredicate implements StudentPredicate {
  private int age;
  
  public AgePredicate(int age) {
    this.age = age;
  }
  
  @Override
  public boolean test(Student s) {
    return s.getAge() == age;
  }
}
```

```java
public class NamePredicate implements StudentPredicate {
  private String name;
  
  public NamePredicate(String name) {
    this.name = name;
  }
  
  @Override
  public boolean test(Student s) {
    return s.getName().equals(name);
  }
}
```

此刻，查找算法的方法名也应该被重命名，使其保持一致的「抽象层次」。

```java
public static Student find(Student[] students, StudentPredicate p) {
  for (Student s : students) {
    if (p.test(s)) {
        return s;
    }
  }
  return null;
}
```

客户端的调用根据场景，提供算法的配置。

```java
assertThat(find(students, new AgePredicate(18)), notNullValue());
assertThat(find(students, new NamePredicate("horance")), notNullValue());
```

#### 结构性重复

`AgPredicate`和`NamePredicate`存在「结构型重复」，为此需要进一步消除重复。经分析两个类的存在无非是为了实现「闭包」的能力，可以使用`lambda`表达式，「`Code As Data`」，简明扼要。

```java
assertThat(find(students, s -> s.getAge() == 18), notNullValue());
assertThat(find(students, s -> s.getName().equals("horance")), notNullValue());
```

#### 引入`Iterable`

按照向稳定的方向依赖的原则，为了适应诸如`List, Set`等多种数据结构，甚至包括原生的数组类型，可以将入参重构为重构为更加抽象的`Iterable`类型。

```java
public static Student find(Iterable<Student> students, StudentPredicate p) {
  for (Student s : students) {
    if (p.test(s)) {
        return s;
    }
  }
  return null;
}
```

> **需求3：** 存在一个老师列表，查找第一个女老师

#### 类型重复

按照既有的代码结构，可以通过`Copy Paste`快速地实现这个功能。

```java
public interface TeacherPredicate {
  boolean test(Teacher t);
}
```

```java
public static Teacher find(Iterable<Teacher> teachers, TeacherPredicate p) {
  for (Teacher t : teachers) {
    if (p.test(t)) {
        return t;
    }
  }
  return null;
}
```

用户接口依然可以使用`Lambda`表达式。

```java
assertThat(find(teachers, t -> t.female()), notNullValue());
```

如果使用`Method Reference`，可以进一步地改善表达力。

```java
assertThat(find(teachers, Teacher::female), notNullValue());
```

#### 类型参数化

分析`StudentMacher/TeacherPredicate`, `find(Iterable<Student>)/find(Iterable<Teacher>)`的重复，其变化的关注点很明显：*参数类型*。

为此，引入「类型参数化」设计消除重复。首先消除`StudentPredicate`和`TeacherPredicate`的重复设计，

```java
public interface Predicate<E> {
  boolean test(E e);
}
```

再对`find`进行类型参数化设计。

```java
public static <E> E find(Iterable<E> c, Predicate<E> p) {
  for (E e : c) {
    if (p.test(e)) {
        return e;
    }
  }
  return null;
}
```

#### 型变

但`find`的类型参数缺乏「型变」的能力，为此引入「型变」能力的支持，接口更加具有可复用性。

```java
public static <E> E find(Iterable<? extends E> c, Predicate<? super E> p) {
  for (E e : c) {
    if (p.test(e)) {
        return e;
    }
  }
  return null;
}
```

#### 复用`lambda`

观察如下两个测试用例，如果做到极致，可认为两个`lambda`表达式也是重复的。从分离关注点的角度分析，此`lambda`表达式承载的「比较算法」与「参数配置」两个职责，应该对其进行分离。

```java
assertThat(find(students, s -> s.getName().equals("Horance")), notNullValue());
assertThat(find(students, s -> s.getName().equals("Tomas")), notNullValue());
```

可以通过`「Static Factory」`生产`lambda`表达式，将比较算法封装起来；而配置参数通过引入「参数化」设计，将「逻辑」与「配置」分离，从而达到最大化的代码复用。

```java
public final class StudentPredicates {
  private StudentPredicates() {
  }

  public static Predicate<Student> age(int age) {
    return s -> s.getAge() == age;
  } 
  
  public static  Predicate<Student> name(String name) {
    return s -> s.getName().equals(name);
  }
}
```

```java
import static StudentPredicates.*;

assertThat(find(students, name("horance")), notNullValue());
assertThat(find(students, age(10)), notNullValue());
```

#### 组合查询

但是，上述将`lambda`表达式封装在`Factory`的设计是及其脆弱的，例如，增加如下的需求：

> **需求4：** 查找*年龄不等于18岁*的*女生*

最简单的方法就是往`StudentPredicates`不停地增加`「Static Factory」`，但这样的设计严重违反了`「OCP」(开放封闭)`原则。

```java
public final class StudentPredicates {
  ......

  public static Predicate<Student> ageEq(int age) {
    return s -> s.getAge() == age;
  } 
  
  public static Predicate<Student> ageNe(int age) {
    return s -> s.getAge() != age;
  } 
}
```

从需求看，比较准则增加了众多的语义，再次运用分离关注点的原则，可发现存在两类运算的规则:

- 比较运算：`==, !=`
- 逻辑运算：`&&，||`

##### 比较语义

先处理比较运算的关注点，为此建立一个`Equal`的抽象：

```java
public interface Equal<E> {
  boolean test(E e1, E e2);
    
  static <E> Equal<E> eq() {
    return (e1, e2) -> e1.equals(e2);
  }
  
  static <E> Equal<E> ne() {
    return (e1, e2) -> !e1.equals(e2);
  }
}
```

此刻，`age`的设计运用了「函数式」的思维，通过函数的「组合式设计」完成功能的自由拼装，简单、直接、漂亮。

```java
public final class StudentPredicates {
  ......

  public static Predicate<Student> age(Equal<Integer> e, int age) {
    return s -> e.test(s.getAge(), age);
  }
}
```

*查找年龄不等于18岁的学生*，可以如此描述。

```java
assertThat(find(students, age(ne(), 18)), notNullValue());
```

##### 逻辑语义

为了使得逻辑「谓词」变得更加人性化，可以引入「流式接口」的`「DSL」`设计，增强表达力。

```java
public interface Predicate<E> {
  boolean test(E e);

  default Predicate<E> and(Predicate<? super E> other) {
    return e -> test(e) && other.test(e);
  }
}
```

*查找年龄不等于18岁的女生*，可以表述为：

```java
assertThat(find(students, age(ne(), 18).and(Student::female)), notNullValue());
```

#### 重复再现

仔细的读者可能已经发现了，`Student`和`Teacher`两个类也存在「结构型重复」的问题。

```java
public class Student {
  public Student(String name, int age, boolean male) {
    this.name = name;
    this.age = age;
    this.male = male;
  }
  
  ......
  
  private String name;
  private int age;
  private boolean male;
}
```

```java
public class Teacher {
  public Teacher(String name, int age, boolean male) {
    this.name = name;
    this.age = age;
    this.male = male;
  }
  
  ......
  
  private String name;
  private int age;
  private boolean male;
}
```

`Student`与`Teacher`的结构性重复，导致`TeacherPredicates`与`TeacherPredicates`也存在结构性重复，为此需要进一步消除重复。

```java
public final class StudentPredicates {
  ......

  public static Predicate<Student> age(Equals<Integer> e, int age) {
    return s -> e.test(s.getAge(), age);
  } 
}
```

```java
public final class TeacherPredicates {
  ......

  public static Predicate<Teacher> age(Equals<Integer> e, int age) {
    return t -> e.test(t.getAge(), age);
  } 
}
```

#### 提取基类

第一个直觉，通过「提取基类」的重构方法，消除`Student`和`Teacher`的重复设计。

```java
class Human {
  protected Human(String name, int age, boolean male) {
    this.name = name;
    this.age = age;
    this.male = male;
  }
    
  ...
  
  private String name;
  private int age;
  private boolean male;
}
```

从而实现了进一步消除了`Student`和`Teacher`之间的重复设计。

```java
public class Student extends Human {
  public Student(String name, int age, boolean male) {
    super(name, age, male);
  }
}

public class Teacher extends Human {
  public Teacher(String name, int age, boolean male) {
    super(name, age, male);
  }
}
```

此时，可以通过引入「类型界定」的泛型设计，使得`TeacherPredicates`与`TeacherPredicates`合二为一，进一步消除重复设计。

```java
public final class HumanPredicates {
  ......
  
  public static <E extends Human> 
    Predicate<E> age(Equals<Integer> e, int age) {
    return s -> e.test(s.getAge(), age);
  } 
}
```

##### 消灭继承关系

`Student`和`Teacher`依然存在「结构型重复」的问题，可以通过`static factory`的设计方法，并让`Human`的构造函数「私有化」，删除`Student`和`Teacher`两个子类，彻底消除两者之间的重复设计。

```java
public class Human {
  private Human(String name, int age, boolean male) {
    this.name = name;
    this.age = age;
    this.male = male;
  }
  
  public static Human student(String name, int age, boolean male) {
    return new Human(name, age, male);
  }
  
  public static Human teacher(String name, int age, boolean male) {
    return new Human(name, age, male);
  }
  
  ......
}
```

##### 消灭类型界定

`Human`的重构，使得`HumanPredicates`的「类型界定」变得多余，从而进一步简化了设计。

```java
public final class HumanPredicates {
  ......
  
  public static Predicate<Human> age(Equals<Integer> e, int age) {
    return s -> e.test(s.getAge(), age);
  } 
}
```

#### 绝不返回`null`

在最开始，我们遗留了一个问题：*`find`返回了`null`*。用户调用返回`null`的接口时，常常忘记`null`的检查，导致在运行时发生`NullPointerException`异常。

按照「向稳定的方向依赖」的原则，`find`的返回值应该设计为`Optional<E>`，使用「类型系统」的特长，取得如下方面的优势：

- 显式地表达了不存在的语义；
- 编译时保证错误的发生；

```java
import java.util.Optional;

public <E> Optional<E> find(Iterable<? extends E> c, Predicate<? super E> p) {
  for (E e : c) {
    if (p.test(e)) {
        return Optional.of(e);
    }
  }
  return Optional.empty();
}
```

## 思考

用一句最精炼的话，或一个词，思考如下三个问题，并通过微信，公众号，[codingstyle.cn](https://codingstyle.cn/topics/75)，大家互动一下。

- 什么样的软件设计才算得上是好的？
- 软件设计的本质是什么？
- `OO`与`FP`的本质区别是什么？

## 鸣谢

「正交设计」的理论、原则、及其方法论出自前`ThoughtWorks`软件大师「袁英杰」先生。英杰既是我的老师，也是我的挚友；他高深莫测的软件设计的修为，及其对软件设计独特的哲学思维方式，是我等后辈学习的楷模。

## 关于我

**刘光聪**，程序员，敏捷教练，开源软件爱好者，具有多年大型遗留系统的重构经验，对`OO`，`FP`，`DSL`等领域具有浓厚的兴趣。

- 公众号：lean_programming
- 微信号：guang-yun-sunny
- GitHub：[https://github.com/horance-liu](https://github.com/horance-liu)
- Email：[horance@outlook.com](mailto: horance@outlook.com)

