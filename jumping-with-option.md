# Jumping with Option

**刘光聪**，程序员，敏捷教练，开源软件爱好者，具有多年大型遗留系统的重构经验，对`OO`，`FP`，`DSL`等领域具有浓厚的兴趣。

- GitHub: [https://github.com/horance-liu](https://github.com/horance-liu)
- Email: [horance@outlook.com](horance@outlook.com)

## Billion-Dollar Mistake

**Tony Hoare**, `null`的发明者在`2009`年公开道歉，并将此错误称为`Billion-Dollar Mistake`。

> I call it my billion-dollar mistake. It was the invention of the null reference in 1965. At that time, I was designing the first comprehensive type system for references in an object oriented language (ALGOL W). My goal was to ensure that all use of references should be absolutely safe, with checking performed automatically by the compiler. But I couldn't resist the temptation to put in a null reference, simply because it was so easy to implement. This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years.

## Idioms and Patterns

### Preconditions

绝大多数`public`的函数对于传递给它们的参数都需要进行限制。例如，索引值不能为负数，对象引用不能为空等等。良好的设计应该保证“发生错误应尽快检测出来”。为此，常常会在函数入口处进行参数的合法性校验。

为了消除大量参数前置校验的重复代码，可以提取公共的工具类库，例如：

```java
public final class Precoditions {
  private Precoditions() {
  }

  public static void checkArgument(boolean exp, String msg = "") {
    if (!exp) {
      throw new IllegalArgumentException(msg);
    }
  }
  
  public static <T> T requireNonNull(T obj, String msg = "") {
    if (obj == null)
      throw new NullPointerException(msg);
    return obj;
  }

  public static boolean isNull(Object obj) {
    return obj == null;
  }

  public static boolean nonNull(Object obj) {
    return obj != null;
  }
}
```	

使用`requireNonNull`等工具函数时，常常`import static`，使其更具表达力。

```java
import static Precoditions.*;
```

系统中大量存在前置校验的代码，例如：

```java
public BigInteger mod(BigInteger m) {
  if (m.signum() <= 0)
    throw new IllegalArgumentException("must be positive: " + m);
  ...
}
```

可以被重构得更加整洁、紧凑，且富有表现力。

```java
public BigInteger mod(BigInteger m) {
  checkArgument(m.signum() > 0 , "must be positive: " + m);
  ...
}
checkArgument(count > 0, "must be positive: %s", count);</pre>
```

一个常见的**误区**就是：对所有参数都进行限制、约束和检查。我将其称为“缺乏自信”的表现，因为在一些场景下，这样的限制和检查纯属多余。

以`C++`为例，如果`public`接口传递了指针，对该指针做前置校验无可厚非，但仅仅在此做一次校验，其在内部调用链上的所有`private`子函数，如果要传递此指针，应该将其变更为`pass by reference`；特殊地，如果是只读，为了做到编译时的安全，`pass by const-reference`更是明智之举。

可以得到一个推论，对于`private`的函数，你对其调用具有完全的控制，自然保证了其传递参数的有效性；如果非得对其`private`的参数进行前置校验，应该使用`assert`。例如：

```java
private static void <T> sort(T a[], int offset, int length) {
  assert a != null;
  assert offset >= 0 && offset <= a.length;
  assert length >= 0 && length <= a.length - offset;
  
  ...
}
```

### Avoid Pass/Return Null

```java
private final List<Product> stock = new ArrayList<>();

public Product[] filter(Predicate<Product> pred) {
  if (stock.isEmpty()) return null;
  ...
}
```

客户端不得不为此校验返回值，否则将在运行时抛出`NullPointerException`异常。

```java
Product[] fakes = repo.filter(Product::isFake);
if (fakes != null && Arrays.asList(fakes).contains(Product.STILTON)) {
  ...
}
```

经过社区的实践总结出，返回`null`的数组或列表是不明智的，而应该返回零长度的数组或列表。

```java
private final List<Product> stock = new ArrayList<>();

private static final Product[] EMPTY = new Product[0]; 

public Product[] filter(Predicate<Product> pred) {
  if (stock.isEmpty()) return EMPTY;
  ...
}
```

对于返回值是`List`的，则应该使用`Collections.emptyXXX`的静态工厂方法，返回零长度的列表。

```java
private final List<Product> stock = new ArrayList<>();

public Product[] filter(Predicate<Product> pred) {
  if (stock.isEmpty()) return Collections.emptyList();
  ...
}
```

### Null Object

```java
private final List<Product> stock = new ArrayList<>();

public Product[] filter(Predicate<Product> pred) {
  if (stock.isEmpty()) return Collections.emptyList();
  ...
}
```

`Collections.emptyList()`工厂方法返回的就是一个`Null Object`，它的实现大致是这样的。

```java
public final class Collections {
  private Collections() {
  }
 
  private static class EmptyList<E> 
    extends AbstractList<E> 
    implements RandomAccess, Serializable {
  
    private static final long serialVersionUID = 8842843931221139166L;
  
    public Iterator<E> iterator() {
      return emptyIterator();
    }

    public ListIterator<E> listIterator() {
      return emptyListIterator();
    }
  
    public int size() {return 0;}
    public boolean isEmpty() {return true;}
  
    public boolean contains(Object obj) {return false;}
    public boolean containsAll(Collection<?> c) { return c.isEmpty(); }
  
    public Object[] toArray() { return new Object[0]; }
  
    public <T> T[] toArray(T[] a) {
      if (a.length > 0)
        a[0] = null;
      return a;
    }
  
    public E get(int index) {
      throw new IndexOutOfBoundsException("Index: "+index);
    }
  
    public boolean equals(Object o) {
      return (o instanceof List) && ((List<?>)o).isEmpty();
    }
  
    public int hashCode() { return 1; }
    
    private Object readResolve() {
      return EMPTY_LIST;
    }
  }
    
  @SuppressWarnings("rawtypes")
  public static final List EMPTY_LIST = new EmptyList<>();

  @SuppressWarnings("unchecked")
  public static final <T> List<T> emptyList() {
    return (List<T>) EMPTY_LIST;
  }
}    
```

`Null Object`代表了一种例外，并且这样的例外具有特殊性，它是一个有效的对象，对于用户来说是透明的，是感觉不出来的。使用`Null Object`，遵循了"按照接口编程"的良好设计原则，并且让用户处理空和非空的情况得到了统一，使得因缺失`null`检查的错误拒之门外。

## Monadic Option

`Null Object`虽然很优雅地使得空与非空得到和谐，但也存在一些难以忍受的情况。

- 接口发生变化（例如新增加一个方法），代表`Null Object`的类也需要跟着变化；
- `Null Object`在不同的场景下重复这一实现方式，其本质是一种模式的重复；
- 有时候，引入`Null Object`使得设计变得更加复杂，往往得不偿失；

### Option的引入

问题的本质在哪里？`null`代表的是一种空，与其对立的一面便是非空。如果将其放置在一个容器中，问题便得到了很完美的解决。也就是说，如果为空，则该容器为空容器；如果不为空，则该值包含在容器之中。

用`Scala`语言表示，可以建立一个`Option`的容器。如果存在，则用`Some`表示；否则用`None`表示。

```scala
sealed abstract class Option[+A] {
  def isEmpty: Boolean
  def get: A
}

case class Some[+A](x: A) extends Option[A] {
  def isEmpty = false
  def get = x
}

case object None extends Option[Nothing] {
  def isEmpty = true
  def get = throw new NoSuchElementException("None.get")
}
```

这样的表示有如下几个方面的好处：

- 对于存在与不存在的值在类型系统中得以表示；
- 显式地表达了不存在的语义；
- 编译时保证错误的发生；

问题并没有那么简单，如果如下使用，并没有发挥出`Option`的威力。

```scala
def double(num: Option[Int]) = {
  num match {
    Some(n) => Some(n*2)
    None => None
  }
}
```

将`Option`视为容器，让其处理`Some/None`得到统一性和一致性。

```scala
def double(num: Option[Int]) = num.map(_*2)
```

也可以使用`for Comprehension`，在某些场景下将更加简洁、漂亮。

```scala
def double(num: Option[Int]) = for (n <- num) yield(n*2)
```

### Option的本质

通过上例的可以看出来，`Option`本质上是一个`Monad`，它是一种函数式的设计模式。用`Java8`简单地形式化一下，可以如下形式化地描述一个`Monad`。

```java
interface M<A> {
  M<B> flatMap(Function<A, M<B>> f);
  
  default M<B> map(Function<A, B> f) {
    return flatMap(a -> unit(f(a)));
  }
  
  static M<A> unit(A a) {
    ...
  }
}
```

同时满足以下三条规则：

- 右单位元(identity)，既对于任意的`Monad m`，则`m.flatMap(unit) <=> m`；
- 左单位元(unit)，既对于任意的`Monad m`，则`unit(v).flatMap(f) <=> f(v)`；
- 结合律，既对于任意的`Monad m`, 则`m.flatMap(g).flatMap(h) <=> m.flatMap(x => g(x).flatMap(h))`

在这里，我们将`Monad`的数学语义简化，为了更深刻的了解`Monad`的本质，必须深入理解`Cathegory Theory`，这好比你要吃披萨的烹饪精髓，得学习意大利的文化。但这对于大部分的程序员要求优点过高，但不排除部分程序员追求极致。

### Option的实现

`Option`的设计与`List`相似，有如下几个方面需要注意：

- `Option`是一个`Immutablity Container`，或者是一个函数式的数据结构；
- `sealed`保证其类型系统的封闭性；
- `Option[+A]`类型参数是协变的，使得`None`可以成为任意`Option[+A]`的子对象；
- 可以被`for Comprehension`调用；

```scala
sealed abstract class Option[+A] { self =>
  def isEmpty: Boolean
  def get: A
  
  final def isDefined: Boolean = !isEmpty

  final def getOrElse[B >: A](default: => B): B =
    if (isEmpty) default else this.get

  final def map[B](f: A => B): Option[B] =
    if (isEmpty) None else Some(f(this.get))

  final def flatMap[B](f: A => Option[B]): Option[B] =
    if (isEmpty) None else f(this.get)

  final def filter(p: A => Boolean): Option[A] =
    if (isEmpty || p(this.get)) this else None

  final def filterNot(p: A => Boolean): Option[A] =
    if (isEmpty || !p(this.get)) this else None

  final def withFilter(p: A => Boolean): WithFilter = new WithFilter(p)

  class WithFilter(p: A => Boolean) {
    def map[B](f: A => B): Option[B] = self filter p map f
    def flatMap[B](f: A => Option[B]): Option[B] = self filter p flatMap f
    def foreach[U](f: A => U): Unit = self filter p foreach f
    def withFilter(q: A => Boolean): WithFilter = new WithFilter(x => p(x) && q(x))
  }

  final def foreach[U](f: A => U) {
    if (!isEmpty) f(this.get)
  }

  final def collect[B](pf: PartialFunction[A, B]): Option[B] =
    if (!isEmpty) pf.lift(this.get) else None

  final def orElse[B >: A](alternative: => Option[B]): Option[B] =
    if (isEmpty) alternative else this
}

case class Some[+A](x: A) extends Option[A] {
  def isEmpty = false
  def get = x
}

case object None extends Option[Nothing] {
  def isEmpty = true
  def get = throw new NoSuchElementException("None.get")
}
```

### `for Comprehension`的本质

`for Comprehension`其实是对具有`foreach, map, flatMap, withFilter`访问方法的容器的一个语法糖。

首先，`pat <- expr`的生成器被解释为：

```scala
// pat <- expr
pat <- expr.withFilter { case pat => true; case _ => false }
```

如果存在一个生成器和`yield`语句，则解释为：

```scala
// for (pat <- expr1) yield expr2
expr1.map{ case pat => expr2 }
```

如果存在多个生成器，则解释为：

```scala
// for (pat1 <- expr1; pat2 <- expr2) yield exprN
expr.flatMap { case pat1 => for (pat2 <- expr2) yield exprN }
expr.flatMap { case pat1 => expr2.map { case pat2 =>  exprN }}
```

对于`for loop`，可解释为：

```scala
// for (pat1 <- expr1; pat2 <- expr2；...) exprN
expr.foreach { case pat1 => for (pat2 <- expr2; ...) yield exprN }
```

对于包含`guard`的生成器，可解释为：

```scala
// pat1 <- expr1 if guard
pat1 <- expr1.withFilter((arg1, arg2, ...) => guard)
```

## Others

- Stream
- Promise
- Either
- Try
- Validation
- Transaction

后需文章将逐一解开它们的面纱，敬请期待！



