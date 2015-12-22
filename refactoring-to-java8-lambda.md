# Refactoring to Java8 Lambda：Eliminate Boilerplate Code

**刘光聪**，程序员，敏捷教练，开源软件爱好者，具有多年大型遗留系统的重构经验，对`OO`，`FP`，`DSL`等领域具有浓厚的兴趣。

- GitHub: [https://github.com/horance-liu](https://github.com/horance-liu)
- Email: [horance@outlook.com](horance@outlook.com)

## Conditional Deferred Execution

### `Logger`

```java
if (logger.isLoggable(Level.INFO)) {
  logger.info("problem:" + getDiagnostic());
}
```

这个实现存在如下一些坏味道：

- 重复的样板代码，并且散乱到用户的各个角落；
- 在`logger.debug`之前，首先要`logger.isLoggable`，`logger`暴露了太多的状态逻辑，违反了`LoD(Law of Demeter)`

### Apply LoD

```java
logger.info("problem:" + getDiagnostic());
```

这样的设计虽然将状态的查询进行了封装，但依然存在一个严重的性能问题。无论如何，`getDiagnostic`都将得到调用，如果它是一个耗时、昂贵的操作，将严重地影响了系统的性能。 

### Apply Lambda

灵活地应用`Lambda`惰性求值的特性，可以很漂亮地解决这个问题。

```java
public void log(Level level, Supplier<String> supplier) {
  if (isLoggable(level)) {
    log(supplier.get());
  }
}

public void debug(Level level, Supplier<String> supplier) {
  return log(Level.DEBUG, supplier);
}

public void info(Level level, Supplier<String> supplier) {
  return log(Level.INFO, supplier);
}

...
```

用户的代码也更加简洁，省略了那些重复的样板代码。

```java
logger.info(() -> "problem:" + getDiagnostic());
```

## Execute Around

我们经常会遇到一个场景，在执行操作之前，先准备环境，之后再拆除环境。例如`XUnit`中的`setUp/tearDown`；操作数据库时，先取得数据库的连接，操作数据后确保释放连接；当操作文件时，先打开文件流，操作文件后确保关闭文件流。

### `try-finally`

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

### `try-with-resources`

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

```java
@FunctionalInterface
public class interface BufferedProcessor {
  String process(BufferedReader bf) throws IOException;
}
```

`process`使用`BufferedProcessor`实现行为的参数化。

```java
public String process(File file, BufferedProcessor p) throws IOException {
  try(BufferedReader bf = new BufferedReader(new FileReader(file))) {
    return p.process(bf);
  }
}
```

用户直接使用`lambda`表达式，更加简单漂亮。

```java
String oneLine = process(file, bf -> bf.readLine()) ;
```


