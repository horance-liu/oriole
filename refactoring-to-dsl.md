# Refactoring to DSL

> OO makes code understandable by encapsulating moving parting, but FP makes code understandable by minimizing moving parts. －Michael Feathers
    
## 软件设计的目标
    
- 实现功能
- 易于重用
- 易于理解
- 没有冗余
    
## 正交设计
    
> 软件设计是一个「守破离」的过程。 -- 袁英杰
    
- 消除重复
- 分离变化方向
- 缩小依赖范围
- 向稳定的方向依赖
    
## 实战
    
> **需求1：** 存在一个学生的列表，查找一个年龄等于`18`岁的学生
    
### 快速实现
    
```java
public static Student findByAge(Student[] students) {
 for (int i=0; i<students.length; i++)
   if (students[i].getAge() == 18)
     return students[i];
 return null;
}
```
    
上述实现存在很多设计的「坏味道」：
    
- 缺乏弹性参数类型：只支持数组类型，`List, Set`都被拒之门外；
- 容易出错：操作数组下标，往往引入不经意的错误；
- 幻数：硬编码，将算法与配置高度耦合；
- 返回`null`：再次给用户打开了犯错的大门；
    
### 使用`for-each`
    
按照「最小依赖原则」，先隐藏数组下标的实现细节，使用`for-each`降低错误发生的可能性。
    
```java
public static Student findByAge(Student[] students) {
 for (Student s : students)
   if (s.getAge() == 18)
     return s;
 return null;
}
```
    
> **需求2：** 查找一个名字为`horance`的学生
    
### 重复设计
    
`Copy-Paste`是最快的实现方法，但会产生「重复设计」。
    
```java
public static Student findByName(Student[] students) {
 for (Student s : students)
   if (s.getName().equals("horance"))
     return s;
 return null;
}
```
    
为了消除重复，可以将「查找算法」与「比较准则」这两个「变化方向」进行分离。
    
### 抽象准则
    
首先将比较的准则进行抽象化，让其独立变化。
    
```java
public interface StudentPredicate {
 boolean test(Student s);
}
```
    
将各个「变化原因」对象化，为此建立了两个简单的算子。
    
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
    
此刻，查找算法的方法名也应该被「重命名」，使其保持在同一个「抽象层次」上。
    
```java
public static Student find(Student[] students, StudentPredicate p) {
 for (Student s : students)
   if (p.test(s))
     return s;
 return null;
}
```
    
客户端的调用根据场景，提供算法的配置。
    
```java
assertThat(find(students, new AgePredicate(18)), notNullValue());
assertThat(find(students, new NamePredicate("horance")), notNullValue());
```
    
### 结构性重复
    
`AgePredicate`和`NamePredicate`存在「结构型重复」，需要进一步消除重复。经分析两个类的存在无非是为了实现「闭包」的能力，可以使用`lambda`表达式，「`Code As Data`」，简明扼要。
    
```java
assertThat(find(students, s -> s.getAge() == 18), notNullValue());
assertThat(find(students, s -> s.getName().equals("horance")), notNullValue());
```
    
### 引入`Iterable`
    
按照「向稳定的方向依赖」的原则，为了适应诸如`List, Set`等多种数据结构，甚至包括原生的数组类型，可以将入参重构为重构为更加抽象的`Iterable`类型。
    
```java
public static Student find(Iterable<Student> students, StudentPredicate p) {
 for (Student s : students)
   if (p.test(s))
     return s;
 return null;
}
```
    
> **需求3：** 存在一个老师列表，查找第一个女老师
    
### 类型重复
    
按照既有的代码结构，可以通过`Copy Paste`快速地实现这个功能。
    
```java
public interface TeacherPredicate {
 boolean test(Teacher t);
}
```
    
```java
public static Teacher find(Iterable<Teacher> teachers, TeacherPredicate p) {
 for (Teacher t : teachers)
   if (p.test(t))
     return t;
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
    
### 类型参数化
    
分析`StudentMacher/TeacherPredicate`, `find(Iterable<Student>)/find(Iterable<Teacher>)`的重复，为此引入「类型参数化」的设计。
    
首先消除`StudentPredicate`和`TeacherPredicate`的重复设计。
    
```java
public interface Predicate<E> {
 boolean test(E e);
}
```
    
再对`find`进行类型参数化设计。
    
```java
public static <E> E find(Iterable<E> c, Predicate<E> p) {
 for (E e : c)
   if (p.test(e))
     return e;
 return null;
}
```
    
### 型变
    
但`find`的类型参数缺乏「型变」的能力，为此引入「型变」能力的支持，接口更加具有可复用性。
    
```java
public static <E> E find(Iterable<? extends E> c, Predicate<? super E> p) {
 for (E e : c)
   if (p.test(e))
     return e;
 return null;
}
```
    
### 复用`lambda`
    
> Parameterize all the things.
    
观察如下两个测试用例，如果做到极致，可认为两个`lambda`表达式也是重复的。从「分离变化的方向」的角度分析，此`lambda`表达式承载的「比较算法」与「参数配置」两个职责，应该对其进行分离。
    
```java
assertThat(find(students, s -> s.getName().equals("Horance")), notNullValue());
assertThat(find(students, s -> s.getName().equals("Tomas")), notNullValue());
```
    
可以通过`「Static Factory Method」`生产`lambda`表达式，将比较算法封装起来；而配置参数通过引入「参数化」设计，将「逻辑」与「配置」分离，从而达到最大化的代码复用。
    
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
    
### 组合查询
    
但是，上述将`lambda`表达式封装在`Factory`的设计是及其脆弱的。例如，增加如下的需求：
    
> **需求4：** 查找年龄不等于18岁的女生
    
最简单的方法就是往`StudentPredicates`不停地增加`「Static Factory Method」`，但这样的设计严重违反了`「OCP」(开放封闭)`原则。
    
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
    
从需求看，比较准则增加了众多的语义，再次运用「分离变化方向」的原则，可发现存在两类运算的规则:
    
- 比较运算：`==, !=`
- 逻辑运算：`&&, ||`
    
#### 比较语义
    
先处理比较运算的变化方向，为此建立一个`Matcher`的抽象：
    
```java
public interface Matcher<T> {
 boolean matches(T actual);
   
 static <T> Matcher<T> eq(T expected) {
   return actual -> expected.equals(actual);
 }
 
 static <T> Matcher<T> ne(T expected) {
   return actual -> !expected.equals(actual);
 }
}
```
    
> Composition everywhere.
    
此刻，`age`的设计运用了「函数式」的思维，其行为表现为「高阶函数」的特性，通过函数的「组合式设计」完成功能的自由拼装组合，简单、直接、漂亮。
    
```java
public final class StudentPredicates {
 ......
    
 public static Predicate<Student> age(Matcher<Integer> m) {
   return s -> m.matches(s.getAge());
 }
}
```
    
*查找年龄不等于18岁的学生*，可以如此描述。
    
```java
assertThat(find(students, age(ne(18))), notNullValue());
```
    
#### 逻辑语义
    
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
assertThat(find(students, age(ne(18)).and(Student::female)), notNullValue());
```
    
### 重复再现
    
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
    
#### 级联反应
    
`Student`与`Teacher`的结构性重复，导致`StudentPredicates`与`TeacherPredicates`也存在「结构性重复」。
    
```java
public final class StudentPredicates {
 ......
    
 public static Predicate<Student> age(Matcher<Integer> m) {
   return s -> m.matches(s.getAge());
 }
}
```
    
```java
public final class TeacherPredicates {
 ......
    
 public static Predicate<Teacher> age(Matcher<Integer> m) {
   return t -> m.matches(t.getAge());
 }
}
```
    
为此需要进一步消除重复。
    
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
    
#### 类型界定
    
此时，可以通过引入「类型界定」的泛型设计，使得`StudentPredicates`与`TeacherPredicates`合二为一，进一步消除重复设计。
    
```java
public final class HumanPredicates {
 ......
 
 public static <E extends Human> 
   Predicate<E> age(Matcher<Integer> m) {
   return s -> m.matches(s.getAge());
 } 
}
```
    
#### 消灭继承关系
    
`Student`和`Teacher`依然存在「结构型重复」的问题，可以通过`Static Factory Method`的设计方法，并让`Human`的构造函数「私有化」，删除`Student`和`Teacher`两个子类，彻底消除两者之间的「重复设计」。
    
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
    
#### 消灭类型界定
    
`Human`的重构，使得`HumanPredicates`的「类型界定」变得多余，从而进一步简化了设计。
    
```java
public final class HumanPredicates {
 ......
 
 public static Predicate<Human> age(Matcher<Integer> m) {
   return s -> m.matches(s.getAge());
 } 
}
```
    
### 绝不返回`null`
    
> Billion-Dollar Mistake
    
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
    
### 引入工厂
    
```java
public interface Matcher<T> {
 boolean matches(T actual);
   
 static <T> Matcher<T> eq(T expected) {
   return actual -> expected.equals(actual);
 }
 
 static <T> Matcher<T> ne(T expected) {
   return actual -> !expected.equals(actual);
 }
}
```
    
将所有的`Static Factory`方法都放在接口中，虽然简单，也很自然。但如果方法之间产生重复代码，需要「提取函数」，设计将变得非常不灵活，因为接口内所有方法都将默认为`public`，这往往不是我们所期望的，为此可以将这些`Static Factory`方法搬迁到`Matchers`实用类中去。
    
```java
public final class Matchers {    
 public static <T> Matcher<T> eq(T expected) {
   return actual -> expected.equals(actual);
 }
 
 public static <T> Matcher<T> ne(T expected) {
   return actual -> !expected.equals(actual);
 }
 
 private Matchers() {
 }
}
```
    
### 实现大于
    
> **需求5：** 查找年龄**大于**18岁的学生
    
```java
assertThat(find(students, age(gt(18)).isPresent(), is(true));
```
    
```java
public final class Matchers {
 ......
 
 public static <T extends Comparable<? super T>> Matcher<T> gt(T expected) {
   return actual -> Ordering.<T>natural().compare(actual, expected) > 0;
 }
}
```
    
其中，`natural`代表了一种自然的比较规则。
    
```java
public final class Ordering {
 public static <T extends Comparable<? super T>> Comparator<T> natural() {
   return (t1, t2) -> t1.compareTo(t2);
 }
}
```
    
### 实现小于
    
> **需求6：** 查找年龄**小于**18岁的学生
    
```java
assertThat(find(students, age(lt(18)).isPresent(), is(true));
```
    
依次类推，「小于」的规则实现如下：
    
```java
public final class Matchers {
 ......
 
 public static <T extends Comparable<? super T>> Matcher<T> gt(T expected) {
   return actual -> Ordering.<T>natural().compare(actual, expected) > 0;
 }
 
 public static <T extends Comparable<? super T>> Matcher<T> lt(T expected) {
   return actual -> Ordering.<T>natural().compare(actual, expected) < 0;
 }
}
```
    
### 提取函数
    
设计产生了明显的重复，可以通过「提取函数」来消除重复。
    
```java
public final class Matchers {
 ......
 
 public static <T extends Comparable<? super T>> Matcher<T> gt(T expected) {
   return actual -> compare(actual, expected) > 0;
 }
 
 public static <T extends Comparable<? super T>> Matcher<T> lt(T expected) {
   return actual -> compare(actual, expected) < 0;
 }
 
 private static <T extends Comparable<? super T>> int compare(T actual, T expected) {
   return Ordering.<T>natural().compare(actual, expected);
 }
}
```
    
其余比较操作，例如`大于等于，小于等于`的设计和实现依此类推，在此不再重述。
    
### 包含子串
    
> **需求7：** 查找名字中包含`horance`的学生
    
```java
assertThat(find(students, name(contains("horance")).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
 
 public static Matcher<String> contains(String substr) {
   return str -> str.contains(substr);
 }
}
```
    
### 子串开头
    
> **需求8：** 查找名字以`horance`开头的学生
    
```java
assertThat(find(students, name(starts("horance")).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
 
 public static Matcher<String> starts(String substr) {
   return str -> str.startsWith(substr);
 }
}
```
    
「子串结尾」的逻辑，可以设计`ends`的关键字，实现依此类推，在此不再重述。
    
### 不区分大小写
    
> **需求9：** 查找名字以`horance`开头，但**不区分大小写**的学生
    
```java
assertThat(find(students, name(starts_ignoring_case("horance")).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
    
 public static Matcher<String> starts(String substr) {
   return str -> str.startsWith(substr);
 }
 
 public static Matcher<String> starts_ignoring_case(String substr) {
   return str -> lower(str).startsWith(lower(substr));
 }
    
 private static String lower(String s) {
   return s.toLowerCase();
 }
}
```
    
`starts`与`starts_ignoring_case`之间存在微妙的重复设计，为此需要进一步消除重复。
    
#### 组合式设计
    
```java
assertThat(find(students, name(ignoring_case(Matchers::starts, "Horance"))).isPresent(), is(true));
```
    
运用函数的「组合式设计」，达到代码的最大可复用性。从`OO`的角度看，`ignoring_case`是对`starts, ends, contains`的功能增强，是一种典型的「修饰」关系。
    
```java
public static Matcher<String> ignoring_case(
 Function<String, Matcher<String>> m, String substr) {
 return str -> m.apply(lower(substr)).matches(lower(str));
}
```
    
其中，`Function<String, Matcher<String>>`是一个一元函数，参数为`String`，返回值为`Matcher<String>`。
    
```java
@FunctionalInterface
public interface Function<T, R> {
   R apply(T t);
}
```
    
#### 强迫用户
    
虽然`ignoring_case`的设计高度可复用性，可由用户根据实际情况，自由拼装组合各种算子。但「方法引用」的语法，给用户给造成了不必要的负担。
    
```java
assertThat(find(students, name(ignoring_case(Matchers::starts, "Horance"))).isPresent(), is(true));
```
    
可以提供`starts_ignoring_case`的语法糖，将用户犯错的几率降至最低，但要保证实现不存在重复设计。
    
```java
assertThat(find(students, name(starts_ignoring_case("Horance"))).isPresent(), is(true));
```
    
此时，`ignoring_case`也应该重构为`private`，变为一个「可重用」的函数。
    
```java
public static Matcher<String> starts_ignoring_case(String substr) {
 return ignoring_case(Matchers::starts, substr);
}
     
private static Matcher<String> ignoring_case(
 Function<String, Matcher<String>> m, String substr) {
 return str -> m.apply(lower(substr)).matches(lower(str));
}
```
    
### 修饰语义
    
> **需求13：** 查找名字中不包含`horance`的第一个学生
    
```java
assertThat(find(students, name(not_contains("horance")).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
 
 public static Matcher<String> not_contains(String substr) {
   return str -> !str.contains(substr);
 }
}
```
    
在这之前，也曾遇到过类似的「反义」的操作。例如，**查找年龄不等于18岁的学生**，可以如此描述。
    
```java
assertThat(find(students, age(ne(18))).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
 
 public static <T> Matcher<T> ne(T expected) {
   return actual -> !expected.equals(actual);
 }
}
```
    
两者对「反义」的描述存在两份不同的表示，是一种隐晦的「重复设计」，需要一种巧妙的设计消除重复。
    
### 提取反义
    
为此，应该删除`not_contains, ne`的关键字，并提供统一的`not`关键字。
    
```java
assertThat(find(students, name(not(contains("horance")))).isPresent(), is(true));
```
    
`not`的实现是一种「修饰」的手法，对既有的`Matcher`功能的增强，巧妙地取得了「反义」功能。
    
```java
public final class Matchers {    
 ......
 
 public static <T> Matcher<T> not(Matcher<T> matcher) {
   return actual -> !matcher.matches(actual);
 }
}
```
    
### 语法糖
    
对于`not(eq(18))`可以设计类似于`not(18)`的语法糖，使其更加简单。
    
```java
assertThat(find(students, age(not(18))).isPresent(), is(true));
```
    
其实现就是对`eq`的一种修饰操作。
    
```java
public final class Matchers {    
 ......
 
 public static <T> Matcher<T> not(T expected) {
   return not(eq(expected));
 }
}
```
    
### 逻辑或
    
> **需求13：** 查找名字中包含`horance`，或者以`liu`结尾的学生
    
```java
assertThat(find(students, name(anyof(contains("horance"), ends("liu")))).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
 
 @SafeVarargs
 public static <T> Matcher<T> anyof(Matcher<? super T>... matchers) {
   return actual -> {
     for (Matcher<? super T> matcher : matchers)
       if (matcher.matches(actual)) 
         return true;
     return false;
   };
 }
}
```
    
### 逻辑与
    
> **需求14：** 查找名字中以`horance`开头，并且以`liu`结尾的学生
    
```java
assertThat(find(students, name(allof(starts("horance"), ends("liu")))).isPresent(), is(true));
```
    
```java
public final class Matchers {    
 ......
 
 @SafeVarargs
 public static <T> Matcher<T> allof(Matcher<? super T>... matchers) {
   return actual -> {
     for (Matcher<? super T> matcher : matchers)
       if (!matcher.matches(actual))
         return false;
     return true;
   };
 }
}
```
    
### 短路
    
`allof`与`anyof`之间的实现存在重复设计，可以通过提取函数消除重复。
    
```java
public final class Matchers {    
 ......
 
 @SafeVarargs
 private static <T> Matcher<T> combine(
   boolean shortcut, Matcher<? super T>... matchers) {
   return actual -> {
     for (Matcher<? super T> matcher : matchers)
       if (matcher.matches(actual) == shortcut)
         return shortcut;
     return !shortcut;
   };
 }
 
 @SafeVarargs
 public static <T> Matcher<T> allof(Matcher<? super T>... matchers) {
   return combine(false, matchers);
 }
   
 @SafeVarargs
 public static <T> Matcher<T> anyof(Matcher<? super T>... matchers) {
   return combine(true, matchers);
 }
}
```
    
### 占位符
    
> **需求15：** 查找算法始终失败或成功
    
```java
assertThat(find(students, age(always(false))).isPresent(), is(false));
```
    
```java
public final class Matchers {    
 ......
 
 public static <E> Matcher<E> always(boolean bool) {
   return e -> bool;
 }
}
```
    
### 回顾
    
通过`15`个需求的迭代和演进，通过运用「正交设计」和「组合式设计」的基本思想，得到了一套接口丰富、表达力极强的`DSL`。
    
这一套简单的`DSL`是一个高度可复用的`Matcher`集合，其设计既包含了`OO`的方法论，也涉及到了`FP`的思维，整体性设计保持高度的一致性和统一性。
    
## 鸣谢
    
「正交设计」的理论、原则、及其方法论出自前`ThoughtWorks`软件大师「袁英杰」先生。英杰既是我的老师，也是我的挚友；其高深莫测的软件设计的修为，及其对软件设计独特的哲学思维方式，是我等后辈学习的楷模。
    
## 思考
    
- 软件设计的本质是什么？
- `OO`与`FP`的本质区别是什么？
- 组合式设计的精髓是什么？
    

