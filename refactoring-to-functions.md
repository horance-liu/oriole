# Refactoring to Functions：Eliminate Boilerplate Code

> OO makes code understandable by encapsulating moving parting, but FP makes code understandable by minimizing moving parts. －Michael Feathers

## Conditional Deferred Execution

### 日志`Logger`

```java
if (logger.isLoggable(Level.INFO)) {
  logger.info("problem:" + getDiagnostic());
}
```

这个实现存在如下一些坏味道：

- 重复的样板代码，并且散乱到用户的各个角落；
- 在`logger.debug`之前，首先要`logger.isLoggable`，`logger`暴露了太多的状态逻辑，违反了`LoD(Law of Demeter)`

> Eliminate Effects Between Unrelated Things.

### Apply LoD

```java
logger.info("problem:" + getDiagnostic());
```

```java
public void info(String msg) {
  if (isLoggable(Level.INFO)) {
    log(msg)
  }
}
```

这样的设计虽然将状态的查询进行了封装，遵循了`LoD`原则，但依然存在一个严重的性能问题。无论如何，`getDiagnostic`都将得到调用，如果它是一个耗时、昂贵的操作，可能成为系统的瓶颈。 



### Apply Lambda

灵活地应用`Lambda`惰性求值的特性，可以很漂亮地解决这个问题。

```java
public void log(Level level, Supplier<String> supplier) {
  if (isLoggable(level)) {
    log(supplier.get());
  }
}

public void debug(Supplier<String> supplier) {
  log(Level.DEBUG, supplier);
}

public void info(Supplier<String> supplier) {
  log(Level.INFO, supplier);
}

...
```

用户的代码也更加简洁，省略了那些重复的样板代码。

```java
logger.info(() -> "problem:" + getDiagnostic());
```

### Apply Scala: Call by Name

在使用`lambda`时多余的`()`显得有点冗余，可以使用`by-name`参数进一步提高表达力。

```scala
def log(level: Level, msg: => String) {
  if (isLoggable(level)) {
    log(msg)
  }
}

def debug(msg: => String) {
  log(DEBUG, msg)
}

def info(msg: => String) {
  log(INFO, msg)
}
```

```scala
logger.info("problem:" + getDiagnostic());
```

`"problem:" + getDiagnostic()`语句并非在`logger.info`展开计算，它被延迟计算直至被`apply`的时候才真正地被评估和计算。

## Execute Around

我们经常会遇到一个场景，在执行操作之前，先准备环境，之后再拆除环境。例如`XUnit`中的`setUp/tearDown`；操作数据库时，先取得数据库的连接，操作数据后确保释放连接；当操作文件时，先打开文件流，操作文件后确保关闭文件流。

### Apply `try-finally`

为了保证异常安全性，在`Java7`之前，常常使用`try-finally`的实现模式解决这样的问题。

```java
public static String process(File file) throws IOException {
  BufferedReader bf = new BufferedReader(new FileReader(file));
  try {
    return bf.readLine();
  } finally {
    if (bf != null) 
      bf.close();
  }
}
```

这样的设计和实现存在几个问题：

- `if (bf != null)`是必须的，但常常被人遗忘；
- `try-finally`的样板代码遍布在用户程序中，造成大量的重复设计；

### Apply `try-with-resources`

自`Java7`，只要实现了`AutoCloseable`的资源类，可以使用`try-with-resources`的实现模式，进一步简化上例的样板代码。

```java
public String process(File file) throws IOException {
  try(BufferedReader bf = new BufferedReader(new FileReader(file))) {
    return bf.readLine();
  }
}
```

但是，在某些场景下很难最大化地复用代码，这使得实现中存在大量的重复代码。例如遍历文件中所有行，并替换制定模式为其他的字符串。

```java
public String replace(File file, String regex, String i) throws IOException {
  try(BufferedReader bf = new BufferedReader(new FileReader(file))) {
    return bf.readLine().replaceAll(regex, replace);
  }
}
```

### Apply Lambda

为了最大化地复用代码，最小化用户样板代码，将资源操作前后的代码保持封闭，使用`lambda`定制与具体问题相关的处理逻辑。

`process`使用`BufferedProcessor`实现行为的参数化。

```java
public static String process(File file, BufferedProcessor p) throws IOException {
  try(BufferedReader bf = new BufferedReader(new FileReader(file))) {
    return p.process(bf);
  }
}
```

其中，`BufferedProcessor`是一个函数式接口，用于描述`lambda`的原型信息。

```java
@FunctionalInterface
public interface BufferedProcessor {
  String process(BufferedReader bf) throws IOException;
}
```

用户使用`lambda`表达式，使得代码更加简单、漂亮。

```java
process(file, bf -> bf.readLine());
```

如果使用`Method Reference`，可增强表达力。

```java
process(file, BufferedReader::readLine);
```

### Apply Scala: Structural Type, Call by Name, Currying

为了最大化地复用资源释放的实现，使用`Scala`可以神奇地构造一个简单的`DSL`，让用户更好地实现复用。

> Make it Easy to Reuse. 

```scala
import scala.language.reflectiveCalls

object using {
  def apply[R <: { def close(): Unit }, T](resource: => R)(f: R => T) = {
    var res: Option[R] = None
    try {
      res = Some(resource)
      f(res.get)
    } finally {
      if (res != None) res.get.close
    }
  }
}
```

`R <: { def close(): Unit }`中泛型参数`R`是一个拥有`close`方法的类型；`resource: => R`将`resource`声明为`Call by Name`，可延迟计算；`apply`使用了两个参数，并进行了`Currying`化。

受益于`Currying`，用户的定制的函数可以使用大括号来增强表达力，`using`犹如内置的语言特性，得到抽象了的控制结构。

```scala
using(Source.fromFile(file)) { source =>
  source.getLines 
}
```

因为参数`source`仅仅使用了一次，可以通过占位符进一步增强表达力。

```scala
using(Source.fromFile(file)) { _.getLines }
```


