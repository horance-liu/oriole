# Jumping with Try

## 场景

以一个简化了的用户登录的鉴权流程，流程大体如下：

- 首先尝试本站鉴权，如果失败，再尝试`twiter`的方式恢复；
- 之后再进行`Two Factor`认证；

## 快速实现

按照流程定义，可以快速实现第一个版本。这段代码充满了很多的坏味道，职责不单一，复杂的异常分支处理，流程的脉络不够清晰等等，接下来我们尝试一种很特别的重构方式来改善设计。

```java
public boolean authenticate(String id, String passwd) {
  User user = null;    
  try {
   user = login(id, passwd);
  } catch (AuthenticationException e) {
    try {
      user = twiterLogin(id, passwd);
    } catch (AuthenticationException et) {
      return false;
    }
  }
    
  return twoFactor(user);
}
```

## 解决思路

`login`或`twiterLogin`生产`User`对象，扮演生产者的角色；而`twoFactor`消费`User`对象，扮演消费者的角色。

最难处理的就是`login`或`twiterLogin`可能会存在异常处理。正常情况下它们生产`User`对象，而异常情况下，则抛出异常。

重构的思路在于将异常处理更加明晰化，让生产者与消费者之间的关系流水化。为此，可以将`User`对象，可能抛出的异常进行容器化，它们形成互斥关系，不能共生于这个世界。

## 容器化

```java
public interface Try<T> {
  static <T, E extends Exception> 
  Try<T> trying(ExceptionalSupplier<T, E> s) {
    try {
      return new Success<>(s.get());
    } catch (Exception e) {
      return new Failure<>(e);
    }
  }
  
  boolean isFailure();
  
  default boolean isSuccess() {
    return !isFailure();
  }
  
  T get();
}
```

其中，`Success`与`Failure`包内私有，对外不公开。

```java
final class Success<T> implements Try<T> {
  private final T value;
  
  Success(T value) {
    this.value = value;
  }
  
  @Override
  public boolean isFailure() {
    return false;
  }

  @Override
  public T get() {
    return value;
  }
}
```

```java
import java.util.NoSuchElementException;

final class Failure<T> implements Try<T> {
  private final Exception e;
  
  Failure(Exception e) {
    this.e = e;
  }
  
  @Override
  public boolean isFailure() {
    return true;
  }

  @Override
  public T get() {
    throw new NoSuchElementException("Failure.get");
  }
}
```

## 生产者

与`JDK8`标准库中不一样，它能处理受检异常。

```java
@FunctionalInterface
public interface ExceptionalSupplier<T, E extends Exception> {
  T get() throws E;
}
```

## 第一次重构

```java
import static Try.trying;

public boolean authenticate(String id, String passwd) {
  Try<User> user = trying(() -> login(id, passwd));
  if (user.isFailure()) {
    user = trying(() -> twiterLogin(id, passwd));
  }
 
  return user.isSuccess() && twoFactor(user.get());
}
```

## 链式DSL

上述`trying`的应用，使用**状态查询**的`Try.isFailure/isSuccess`方法显得有些笨拙，可以通过构造链式的`DSL`改善设计。

```java
public interface Try<T> {
  ......

  <U> Try<U> recover(Function<Exception, Try<U>> f);
}
```

```java
final class Success<T> implements Try<T> {
  ......
   
  @Override
  @SuppressWarnings("unchecked")
  public <U> Try<U> recover(Function<Exception, Try<U>> f) {
    return (Try<U>)this;
  }
}
```

```java
final class Failure<T> implements Try<T> {
  private final Exception e;
  
  Failure(Exception e) {
    this.e = e;
  }
   
  @Override
  public <U> Try<U> recover(Function<Exception, Try<U>> f) {
    try {
      return f.apply(e);
    } catch (Exception e) {
      return new Failure<U>(e);
    }
  }
}
```

## 第二次重构

使用`recover`关键字，进一步地改善表达力。首先试图`login`生产一个`User`，如果失败从`twiterLogin`中恢复；最后由`twoFactor`消费`User`对象。

```java
public boolean authenticate(String id, String passwd) {
  Try<User> user = trying(() -> login(id, passwd))
      .recover(e -> trying(() -> twiterLogin(id, passwd)));    
    
  return user.isSuccess() && twoFactor(user.get());
}
```

### 彻底链化

```java
public interface Try<T> {
  ......

  default T getOrElse(T defaultValue) {
    return isSuccess() ? get() : defaultValue;
  }
    
  <U> Try<U> map(Function<T, U> f);
}
```

```java
final class Success<T> implements Try<T> {
  ......
  
  @Override
  public <U> Try<U> map(Function<T, U> f) {
    try {
      return new Success<U>(f.apply(value));  
    } catch (Exception e) {
      return new Failure<U>(e);
    }
  }
}
```

```java
final class Failure<T> implements Try<T> {
  ......

  @Override
  @SuppressWarnings("unchecked")
  public <U> Try<U> map(Function<T, U> f) {
    return (Try<U>)this;
  }
}
```

## 第三次重构

```java
public boolean authenticate(String id, String passwd) {
  return trying(() -> login(id, passwd))
    .recover(e -> trying(() -> twiterLogin(id, passwd)))
    .map(user -> twoFactor(user))
    .getOrElse(false);
}
```

## 应用Scala

`Java8 Lambda`表达式`() -> login(id, passwd)`中空的参数列表颇让人费解，但这是`Java8`语言本身限制的，应用`Scala`表达力可进一步提高。

```scala
import scala.util.Try

def authenticate(id: String, passwd: String): Boolean = {
  Try(login(id, passwd))
    .recover{ case e: => twiterLogin(id, passwd) }
    .map(twoFactor)
    .getOrElse(false)
}
```

## Try的本质

`Try`是`Monad`的一个应用，使得异常的处理可以在流水线上传递。使用`Scala`的`Try`简化实现版本是这样的。

```scala
sealed abstract class Try[+T] {
  def isSuccess: Boolean
  def get: T
}

final case class Failure[+T](val exception: Throwable) extends Try[T] {
  def isSuccess: Boolean = false
  def get: T = throw exception
}


final case class Success[+T](value: T) extends Try[T] {
  def isSuccess: Boolean = true
  def get = value
}

object Try {
  def apply[T](r: => T): Try[T] = {
    try Success(r) catch {
      case NonFatal(e) => Failure(e)
    }
  }
}
```

## About Me

**刘光聪**，程序员，敏捷教练，开源软件爱好者，具有多年大型遗留系统的重构经验，对`OO`，`FP`，`DSL`等领域具有浓厚的兴趣。

- GitHub: [https://github.com/horance-liu](https://github.com/horance-liu)
- Email: [horance@outlook.com](emailto: horance@outlook.com)

