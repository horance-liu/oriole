# Programming DSL: Matchers

> 软件设计是一个「守破离」的过程。 --袁英杰

## 回顾设计

上次在「软件匠艺小组」上分享了[「正交设计」](https://codingstyle.cn/topics/75)的基本理论，原则和应用，在活动线下收到了很多朋友的反馈。其中有人谈及到了`DSL`的设计，为此我将继续以`find`为例，通过「正交设计」的应用，重点讨论`DSL`的设计过程。

首先回顾一下之前`find`算子重构的成果。

```java
public <E> Optional<E> find(Iterable<? extends E> c, Predicate<? super E> p) {
  for (E e : c) {
    if (p.test(e)) {
      return Optional.of(e);
    }
  }
  return Optional.empty();
}
```

另外根据需求`1~4`，抽象了`2`个变化方向:

- 比较运算：`==, !=`
- 逻辑运算：`&&, ||`

### 比较语义

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

**查找年龄不等于18岁的学生**，可以如此描述。

```java
assertThat(find(students, age(ne(18))).isPresent(), is(true));
```

### 逻辑语义

```java
public interface Predicate<E> {
  boolean test(E e);

  default Predicate<E> and(Predicate<? super E> other) {
    return e -> test(e) && other.test(e);
  }
  
  default Predicate<E> or(Predicate<? super E> other) {
    return e -> test(e) || other.test(e);
  }
}
```

**查找名字叫`horance`的男生**，可以表述为：

```java
assertThat(find(students, name(eq("horance")).and(Human::male)).isPresent(), is(true));
```

## 引入工厂

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

将所有的`static factory`方法都放在接口中，虽然简单，也很自然。但如果方法之间产生重复代码，需要提取函数，设计将变得非常不灵活，因为接口内所有方法都将默认为`public`，这往往不是我们所期望的，为此可以将这些`static factory`方法搬迁到`Matchers`实用类中去。

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

## 完备的比较规则

### 实现大于

> **需求5：** 查找年龄**大于**18岁的学生

```java
assertThat(find(students, age(gt(18)).isPresent(), is(true));
```

```java
public final class Matchers {
  ......
  
  public static <T extends Comparable<T>> Matcher<T> gt(T expected) {
    return actual -> Ordering.<T>natural().compare(actual, expected) > 0;
  }
}
```

其中，`natural`代表了一种自然的比较规则。

```java
public final class Ordering {
  public static <T extends Comparable<T>> Comparator<T> natural() {
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
  
  public static <T extends Comparable<T>> Matcher<T> gt(T expected) {
    return actual -> Ordering.<T>natural().compare(actual, expected) > 0;
  }
  
  public static <T extends Comparable<T>> Matcher<T> lt(T expected) {
    return actual -> Ordering.<T>natural().compare(actual, expected) < 0;
  }
}
```

### 提取函数

设计产生了明显的重复，可以通过「提取函数」来消除重复。

```java
public final class Matchers {
  ......
  
  public static <T extends Comparable<T>> Matcher<T> gt(T expected) {
    return actual -> compare(actual, expected) > 0;
  }
  
  public static <T extends Comparable<T>> Matcher<T> lt(T expected) {
    return actual -> compare(actual, expected) < 0;
  }
  
  private static <T extends Comparable<T>> int compare(T actual, T expected) {
    return Ordering.<T>natural().compare(actual, expected);
  }
}
```

其余比较操作，例如`大于等于，小于等于`的设计和实现依此类推，在此不再重述。

## 完备的字符串操作

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
  
  public static Matcher<String> starts_ignoring_case(String substr) {
    return str -> lower(str).startsWith(lower(substr));
  }
   
  private static String lower(String s) {
    return s.toLowerCase();
  }
}
```

## 修饰语义

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

## 组合语义

### 逻辑或

> **需求13：** 查找名字中包含`horance`，或者以`liu`结尾的学生

```java
assertThat(find(students, name(anyof(contains("horance"), ends("liu")))).isPresent(), is(true));
```

```java
public final class Matchers {    
  ......
  
  @SafeVarargs
  private static <T> Matcher<T> anyof(Matcher<? super T>... matchers) {
    return actual -> {
      for (Matcher<? super T> matcher : matchers) {
        if (matcher.matches(actual)) {
          return true;
        }
      }
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
  private static <T> Matcher<T> allof(Matcher<? super T>... matchers) {
    return actual -> {
      for (Matcher<? super T> matcher : matchers) {
        if (matcher.matches(actual)) {
          return true;
        }
      }
      return false;
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
  private static <T> Matcher<T> combine(boolean shortcut, Matcher<? super T>... matchers) {
    return actual -> {
      for (Matcher<? super T> matcher : matchers) {
        if (matcher.matches(actual) == shortcut) {
          return shortcut;
        }
      }
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

## 占位符

> **需求15：** 查找算法始终失败

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

## 结论

通过一系列的需求的演进和迭代，通过遵循「正交设计」的基本原则，我们得到了一套接口丰富、表达力极强的`DSL`。这一套简单的`DSL`是一个高度可复用的`Matcher`集合，为此我将其称为`Matcher DSL`，其设计既包含了`OO`的方法论，也涉及到了一些`FP`的思维，整体性设计保持高度的一致性和统一性。

