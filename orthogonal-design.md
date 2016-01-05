# Orthogonal Design

> Design is there to enable you to keep changing the software easily in the long term.  -- Kent Beck.

## 出处

正交设计的理论、原则、及其方法论出自前`ThoughtWorkers`咨询师袁英杰先生。英杰既是我的老师，也是我的挚友，他高深莫测的软件设计的修为，及其对软件设计独特的哲学思维方式，是我等后辈学习的楷模。

本文所有的论点均出自袁英杰的文章，增加了我的一些思考和理解。

## Why Design?

我常常听到这样的言论，软件设计并不重要，交付功能才是最重要的。当然，交付价值是软件设计的首要任务，正确的软件设计方法与交付并不冲突，相反良好的软件设计过程，就是为了长期地、持久地、更好更快、更容易地实现软件价值的交付。

正如`Kent Beck`所说，软件设计是为了**长期**更加容易地适应未来的变化。

## 软件设计的目标

软件设计就是为了完成如下目标，其可验证性、重要程度依次减低，这就是`Kent Beck`的简单设计原则的表述。

- 实现功能
- 易于重用
- 易于理解
- 没有冗余

### 实现功能

实现功能的目标压倒一起，这也是软件设计的首要标准。如何判定系统功能的完备呢？通过所有测试用例。从`TDD`的角度看，测试用例就是对需求的阐述，是一个闭环的反馈系统，保证其系统的正确性，设计的合理性；当然也是理解系统行为最重要的依据。

### 易于理解

易于理解的软件，使得其他人也能容易地理解系统的行为，业务的规则。什么样的设计才是易于理解的呢？

- `Clean Code`
- `Implement Patterns`
- `Idioms`

### 没有冗余

没有冗余的系统是最简单的系统，恰如其分的系统，不做任何过度设计的系统。

- `Dead Code`
- `YAGNI: You Ain't Gonna Need It`
- `KISS: Keep it Simple, Stupid`

### 易于重用

易于重用的软件结构，使得其应对变化更具弹性，可被容易地修改，具有更加适应变化的能力。最理想的情况下，所有的软件修改都具有局部性。为了做到局部性修改，高内聚、低耦合原则是提高可重用性的最高原则。

为了实现高内聚，低耦合的软件设计，袁英杰提出了正交设计的方法论。

## 正交设计

正交是一个数学概念：两个向量的内积为零，则认为是正交的。简单的说，就是两个向量是垂直的。在一个正交系统里，沿着一个方向的变化，其另外一个方向的值不会发生变化。正交性，意味着更高的内聚，更低的耦合。为此，正交性可以用于衡量系统的可重用性。

如何保证设计的正交性呢？袁英杰提出了正交设计四原则，简明扼要，道破了软件设计的精髓所在。

### 正交设计原则

- 消除重复
- 分离关注点
- 缩小依赖范围
- 向稳定的方向依赖

### 实战

> **需求1** 存在一个学生的列表，查找一个年龄等于`18`岁的学生

#### 快速实现

```java
public static Student findByAge(Student[] students) {
  for (int i=0; i<students.length; i++) {    if (students[i].getAge() == 18) {        return students[i];    }
  }
  return null;
}
```

为了阐述设计的整个思考，及其重构的过程，故意未直接调用`JDK`标准库的`API`。上述实现存在很多代码的坏味道：

- 数组类型
- 容易出错
- 幻数
- 返回`null`

#### 使用`for-each`

按照最小依赖的原则，避免数组下标操作失误，使用`for-each`降低错误发生的可能性。

```java
public static Student findByAge(Student[] students) {
  for (Student s : students) {    if (s.getAge() == 18) {        return student;    }
  }
  return null;
}
```

> **需求2** 查找一个名字为`horance`的学生

#### 比较准则重复

`Copy-Paste`是最快的实现方法，但会产生重复的设计。

```java
public static Student findByName(Student[] students) {
  for (Student s : students) {    if (s.getName().equals("horance")) {        return student;    }
  }
  return null;
}
```

为了消除重复，将查找的算法与比较的准则这两个关注点进行分离。

#### 分离关注点

首先将比较的准则进行抽象化，让其独立变化。

```java
public interface StudentMatcher {
  boolean matches(Student s);
}
```

将各个变化对象化，为此建立了两个简单的算子。

```java
public class AgeMatcher implements StudentMatcher {
  private int age;
  
  public AgeMatcher(int age) {
    this.age = age;
  }
  
  @Override
  public boolean matches(Student s) {
    return s.getAge() == age;
  }
}
```

```java
public class NameMatcher implements StudentMatcher {
  private String name;
  
  public NameMatcher(String name) {
    this.name = name;
  }
  
  @Override
  public boolean matches(Student s) {
    return s.getName().equals(name);
  }
}
```

此刻查找算法的方法名也应该被重命名。

```java
public static Student find(Student[] students, StudentMatcher matcher) {
  for (Student student : students) {    if (matcher.matches(student)) {        return student;    }
  }
  return null;
}
```

客户端的调用变成了：

```java
assertThat(find(students, new AgeMatcher(18)), notNullValue());
assertThat(find(students, new NameMatcher("horance")), notNullValue());
```

#### 结构性重复

`AgeGreater`和`AgeEqualTo`也存在结构型的重复，为此需要进一步消除重复。

经分析两个类的存在无非是为了实现闭包的能力，可以使用`lambda`表达式，直接删除两个类。客户端调用直接定制`lambda`表达式。

```java
assertThat(find(students, s -> s.getAge() == 18), notNullValue());
assertThat(find(students, s -> s.getName().equals("horance")), notNullValue());
```

#### 引入`Iterable`

按照向稳定的方向依赖的原则，为了适应诸如`List, Set`等多种数据结构，甚至包括原生的数组类型，可以将入参重构为重构为更加抽象的`Iterable`类型。

```java
public static Student find(Iterable<Student> students, StudentMatcher matcher) {
  for (Student student : students) {    if (matcher.matches(student)) {        return student;    }
  }
  return null;
}
```

> **需求3** 存在一个老师列表，查找第一个女老师

#### 类型重复

按照既有的代码结构，可以通过`Copy Paste`快速地实现这个功能。

```java
public interface TeacherMatcher {
  boolean matches(Teacher t);
}
```

```java
public static Teacher find(Iterable<Teacher> teachers, TeacherMatcher matcher) {
  for (Teacher t : teachers) {    if (matcher.matches(t)) {        return t;    }
  }
  return null;
}
```

用户接口使用`Method Reference`，增强表达力。

```java
assertThat(find(teachers, Teacher::isFemale), notNullValue());
```

#### 类型参数化

分析`StudentMacher/TeacherMatcher`, `find(Iterable<Student>)/find(Iterable<Teacher>)`的重复，其变化的关注点很明显：集合元素类型。

为此，引入类型参数化设计消除重复。首先消除`StudentMatcher`和`TeacherMatcher`的重复设计，

```java
public interface Matcher<E> {
  boolean matches(E e);
}
```

再对`find`进行类型参数化设计。

```java
public static <E> E find(Iterable<E> c, Matcher<E> m) {
  for (E e : c) {    if (m.matches(e)) {        return e;    }
  }
  return null;
}
```

#### 型变

但`find`的类型参数缺乏型别的能力，为此引入型变能力的支持。

```java
public static <E> E find(Iterable<? extends E> c, Matcher<? super E> m) {
  for (E e : c) {    if (m.matches(e)) {        return e;    }
  }
  return null;
}
```

#### 复用`lambda`

```java
assertThat(find(students, s -> s.getName().equals("Horance")), notNullValue());
assertThat(find(students, s -> s.getName().equals("Tomas")), notNullValue());
```

可以通过`static factory`生产`lambda`表达式，从而消除重复。

```java
public final class StudentMatchers {
  private StudentMatchers() {
  }

  public static StudentMatchers age(int age) {
    return s -> s.getAge() == age;
  } 
  
  public static  StudentMatchers name(String name) {
    return s -> s.getName().equals(name);
  }
  
  public static StudentMatchers female(int age) {
    return Student::female;
  } 
  
  public static StudentMatchers male(int age) {
    return Student::male;
  } 
}
```

```java
import static StudentMatchers.*;

assertThat(find(students, name("horance")), notNullValue());
assertThat(find(students, age(10)), notNullValue());
```

但是，这些`lambda`表达式的复用性很差，例如有如下的需求。

> **需求4** 查找年龄大于18岁的女生

#### 组合查询

从需求看，比较准则增加了众多的语义，再次运用分离关注点的原则，可发现存在两类运算的规则:

- 比较运算：`>, <, >=, <=, ==, !=`
- 逻辑运算：`&&，||，!`

##### 比较语义

先处理比较运算的关注点，为此建立一个`Comparing`的抽象：

```java
public interface Comparing<E> {
  boolean compare(E e1, E e2);
    
  static Comparing<E> eq() {
    return (e1, e2) -> e1.equals(e2);
  }    
  
  static Comparing<E> ne() {
    return (e1, e2) -> !e1.equals(e2);
  }    
  
  static Comparing<E> lt() {
    return (e1, e2) -> e1 < e2;
  }
  
  static Comparing<E> gt() {
    return (e1, e2) -> e1 > e2;
  }

  static Comparing<E> lteq() {
    return (e1, e2) -> e1 <= e2;
  }
  
  static Comparing<E> gteq() {
    return (e1, e2) -> e1 >= e2;
  }
}
```

`age`的静态工厂方法也被重构为：

```java
public final class StudentMatchers {
  ......

  public static StudentMatchers age(Comparing<Integer> c, int age) {
    return s -> c.compare(s.getAge(), age);
  }
}
```

查找年龄大于18岁的学生，可以如此描述。

```java
assertThat(find(students, age(gt(), 18)), notNullValue());
```

##### 逻辑语义

```java
public interface Matcher<E> {
  boolean matches(E e);
  
  default Matcher negate() {
    return e -> !matches(e);
  }

  default Matcher<T> and(Matcher<? super T> other) {
    return e -> matches(e) && other.matches(e);
  }

  default Matcher<T> or(Matcher<? super T> other) {
    return e -> matches(e) || other.matches(e);
  }
}
```

查找年龄大于18岁的女生，可以表述为：

```java
assertThat(find(students, age(gt(), 18).and(female())), notNullValue());
```

#### 再现类型重复

```java
public final class StudentMatchers {
  ......

  public static StudentMatchers age(Comparing<Integer> c, int age) {
    return s -> c.compare(s.getAge(), age);
  } 
}
```

```java
public final class TeacherMatchers {
  ......

  public static TeacherMatchers age(Comparing<Integer> c, int age) {
    return s -> c.compare(s.getAge(), age);
  } 
}
```

#### 鸭子编程

为了消除重复，进行类型参数化，为此提取一个`Student`和`Teacher`的共同接口`HumanLike`，并让它们各自实现此接口。

```java
public interface HumanLike {
  int getAge();
  String getName();
  boolean male();

  default boolean female() {
    return !male();
  }
}
```

此时，`Matchers`依赖于一个更加抽象的`HumanLike`特质(借鉴于`Scala`的特性)。

```java
public final class Matchers {
  ......
  
  public static <E extends HumanLike> Matchers age(Comparing<Integer> c, int age) {
    return s -> c.compare(s.getAge(), age);
  } 
}
```

#### 返回`null`

在最开始，我们遗留了一个问题，`find`返回了`null`。用户调用返回`null`的接口时，常常忘记`null`的检查，导致在运行时发生`NullPointerException`异常。

按照向稳定的方向依赖的原则，`find`的返回值应该设计为`Optional<E>`，从类型系统的角度避免错误的发生。

```java
import java.util.Optional;

public <E> Optional<E> find(Iterable<? extends E> c, Matcher<? super E> m) {
  for (E e : c) {    if (m.matches(e)) {        return Optional.of(e);    }
  }
  return Optional.empty();
}
```

## 思考

用一句最精炼的一句话，或一个词思考如下三个问题：

- 什么样的软件设计才算得上是好的设计？
- 软件设计的本质是什么？
- OO与FP的本区区别是什么？

## About Me

**刘光聪**，程序员，敏捷教练，开源软件爱好者，具有多年大型遗留系统的重构经验，对`OO`，`FP`，`DSL`等领域具有浓厚的兴趣。

- 公众号：lean_programming
- 微信号：guang-yun-sunny
- GitHub：[https://github.com/horance-liu](https://github.com/horance-liu)
- Email：[horance@outlook.com](mailto: horance@outlook.com)

