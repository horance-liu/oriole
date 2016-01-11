# Programming DSL: Implements JSpec

> There are two ways of constructing a software design. One way is to make it so simple that there are obviously no deficiencies. And the other way is to make it so complicated that there are no obvious deficiencies. -- C.A.R. Hoare

## 前世今生

本文是`《Programming DSL》`系列文章的第`2`篇，如果该主题感兴趣，可以查阅如下文章：

- [Programming DSL: Implements JHamcrest](https://codingstyle.cn/topics/79)
- [正交设计](https://codingstyle.cn/topics/75)

本文通过「`JSpec`」的设计和实现的过程，加深认识「内部`DSL`」设计的基本思路。`JSpec`是使用`Java8`实现的一个简单的「`BDD`」测试框架。

## 动机

在`Java`社区中，`JUnit`是一个广泛被使用的测试框架。不幸的是，`JUnit`的测试用例必须遵循严格的「标识符」命名规则，给程序员带来了很大的不便。

### 命名模式

`Junit`为了做到「自动发现」机制，在运行时完成用例的组织，规定所有的测试用例必须遵循`public void testXXX()`的函数原型。

```java
public void testTrue() {
  Assert.assertTrue(true);
}
```

### 注解

自`Java 1.5`支持「注解」之后，社区逐步意识到了「注解优于命名模式」的最佳实践，`JUnit`使用`@Test`注解，增强了用例的表现力。

```java
@Test
public void alwaysTrue() {
  Assert.assertTrue(true);
}
```

### `Given-When-Then`

经过实践证明，基于场景验收的`Given-When-Then`命名风格具有强大的表现力。但`JUnit`遵循严格的标示符命名规则，程序员需要承受巨大的痛苦。

这种混杂「驼峰」和「下划线」的命名风格，虽然在社区中得到了广泛的应用，但在重命名时，变得非常不方便。

```java
public class GivenAStack {
  @Test
  public void should_be_empty_when_created() { 
  }

  @Test
  public void should_pop_the_last_element_pushed_onto_the_stack() { 
  }
}
```

### 新贵

以`RSpec, Cucumber, Jasmine`等为代表的`[BDD」(Behavior-Driven Development)`测试框架以强大的表现力，迅速得到了社区的广泛应用。其中，`RSpec, Jasmine`就是我较为喜爱的测试框架。例如，`Jasmine`的`JavaScript`测试用例是这样的。

```js
describe("A suite", function() {
  it("contains spec with an expectation", function() {
    expect(true).toBe(true);
  });
});
```

## JSpec

我们将尝试设计和实现一个`Java`版的`BDD`测试框架：**`JSpec`**。它的风格与`Jasmine`基本类似，并与`Junit4`配合得完美无瑕。

```java
@RunWith(JSpec.class)
public class JSpecs {{
  describe("A spec", () -> {
    List<String> items = new ArrayList<>();

    before(() -> {
      items.add("foo");
      items.add("bar");
    });

    after(() -> {
      items.clear();
    });

    it("runs the before() blocks", () -> {
      assertThat(items, contains("foo", "bar"));
    });

    describe("when nested", () -> {
      before(() -> {
        items.add("baz");
      });

      it("runs before and after from inner and outer scopes", () -> {
        assertThat(items, contains("foo", "bar", "baz"));
      });
    });
  });
}}
```

### 初始化块

```java
public class JSpecs {{
  ......
}}
```

嵌套两层`{}`，这是`Java`的一种特殊的初始化方法，常称为`初始化块`。其行为与如下代码类同，但它更加简洁、漂亮。

```java
public class JSpecs {
  public JSpecs() {
    ......
  }
}
```

### 代码块

`describe, it, before, after`都存在一个`() -> {...}`代码块，以便实现行为的定制化，为此先抽象一个`Block`的概念。

```java
@FunctionalInterface
public interface Block {
  void apply() throws Throwable;
}
```

### 雏形

定义如下几个函数，明确`JSpec DSL`的基本雏形。

```java
public class JSpec {
  public static void describe(String desc, Block block) {
    ......
  }

  public static void it(String behavior, Block block) {
    ......
  }

  public static void before(Block block) {
    ......
  }

  public static void after(Block block) {
    ......
  }
```

### 上下文

`describe`可以嵌套`describe, it, before, after`的代码块，并且外层的`describe`给内嵌的代码块建立了「上下文」环境。

例如，`items`在最外层的`describe`中定义，它对`describe`整个内部都可见。

#### 隐式树

`describe`可以嵌套`describe`，并且`describe`为内部的结构建立「上下文」，因此`describe`之间建立了一棵「隐式树」。

#### 领域模型

为此，抽象出了`Context`的概念，用于描述`describe`的运行时。也就是是，`Context`描述了`describe`内部可见的几个重要实体：

- `List<Block> befores：before`代码块集合
- `List<Block> afters：after`代码块集合
- `Description desc：`包含了父子之间的层次关系等上下文描述信息
- `Deque<Executor> executors：`执行器的集合。 

`Executor`在后文介绍，可以将`Executor`理解为`Context`及其`Spec`的运行时行为；其中，`Context`对于于`desribe`子句，`Spec`对于于`it`子句。

因为`describe`之间存在「隐式树」的关系，`Context`及`Spec`之间也就形成了「隐式树」的关系。

#### 参考实现

```java
public class Context {

  private List<Block> befores = new ArrayList<>();
  private List<Block> afters = new ArrayList<>();

  private Deque<Executor> executors = new ArrayDeque<>();
  private Description desc;
  
  public Context(Description desc) {
    this.desc = desc;
  }
  
  public void addChild(Context child) {
    desc.addChild(child.desc);
    executors.add(child);
    
    child.addBefore(collect(befores));
    child.addAfter(collect(afters));
  }

  public void addBefore(Block block) {
    befores.add(block);
  }

  public void addAfter(Block block) {
    afters.add(block);
  }

  public void addSpec(String behavior, Block block) {
    Description spec = createTestDescription(desc.getClassName(), behavior);
    desc.addChild(spec);
    addExecutor(spec, block);
  }

  private void addExecutor(Description desc, Block block) {
    Spec spec = new Spec(desc, blocksInContext(block));
    executors.add(spec);
  }

  private Block blocksInContext(Block block) {
    return collect(collect(befores), block, collect(afters));
  }
}
```

#### 实现`addChild`

`describe`嵌套`describe`时，通过`addChild`完成了两件重要工作：

- 「`子Context`」向「`父Context`」的注册；也就是说，`Context`之间形成了「树」形结构；
- 控制`父Context`中的`before/after`的代码块集合对`子Context`的可见性；

```java
public void addChild(Context child) {
  desc.addChild(child.desc);
  executors.add(child);
    
  child.addBefore(collect(befores));
  child.addAfter(collect(afters));
}
```

其中，`collect`定义于`Block`接口中，完成`before/after`代码块「集合」的迭代处理。这类似于`OO`世界中的「组合模式」，它们代表了一种隐式的「树状结构」。

```java
public interface Block {
  void apply() throws Throwable;

  static Block collect(Iterable<? extends Block> blocks) {
    return () -> {
      for (Block b : blocks) {
        b.apply();
      }
    };
  }
}
```

#### 实现`addExecutor`

其中，`Executor`存在两种情况:

- `Spec:` 使用`it`定义的用例的代码块
- `Context:` 使用`describe`定义上下文。

为此，`addExecutor`被`addSpec, addChild`所调用。`addExecutor`调用时，将`Spec`注册到`Executor`集合中，并定义了`Spec`的「执行规则」。

```java
private void addExecutor(Description desc, Block block) {
    Spec spec = new Spec(desc, blocksInContext(block));
    executors.add(spec);
  }

  private Block blocksInContext(Block block) {
    return collect(collect(befores), block, collect(afters));
  }
```

`blocksInContext`将`it`的「执行序列」行为固化。

- 首先执行`before`代码块集合；
- 然后执行`it`代码块；
- 最后执行`after`代码块集合；

### 抽象`Executor`

之前谈过，`Executor`存在两种情况:

- `Spec:` 使用`it`定义的用例的代码块
- `Context:` 使用`describe`定义上下文。

也就是说，`Executor`构成了一棵「树状」的数据结构；`it`扮演了「叶子节点」的角色；`Context`扮演了「非叶子节点」的角色。为此，`Executor`的设计采用了「组合模式」。

```java
import org.junit.runner.notification.RunNotifier;

@FunctionalInterface
public interface Executor {
  void exec(RunNotifier notifier);
}
```

#### 叶子节点：`Spec`

`Spec`完成对`it`行为的封装，当`exec`时完成`it`代码块`() -> {...}`的调用。

```java
public class Spec implements Executor {

  public Spec(Description desc, Block block) {
    this.desc = desc;
    this.block = block;
  }

  @Override
  public void exec(RunNotifier notifier) {
    notifier.fireTestStarted(desc);
    runSpec(notifier);
    notifier.fireTestFinished(desc);
  }

  private void runSpec(RunNotifier notifier) {
    try {
      block.apply();
    } catch (Throwable t) {
      notifier.fireTestFailure(new Failure(desc, t));
    }
  }

  private Description desc;
  private Block block;
}
```

#### 非叶子节点：`Context`

```java
public class Context implements Executor {
  ......
  
  private Description desc;
  
  @Override
  public void exec(RunNotifier notifier) {
    for (Executor e : executors) {
      e.exec(notifier);
    }
  }
}
```

### 实现`DSL`

有了`Context`的领域模型的基础，`DSL`的实现变得简单了。

```java
public class JSpec {
  private static Deque<Context> ctxts = new ArrayDeque<Context>();

  public static void describe(String desc, Block block) {
    Context ctxt = new Context(createSuiteDescription(desc));
    enterCtxt(ctxt, block);
  }

  public static void it(String behavior, Block block) {
    currentCtxt().addSpec(behavior, block);
  }

  public static void before(Block block) {
    currentCtxt().addBefore(block);
  }

  public static void after(Block block) {
    currentCtxt().addAfter(block);
  }

  private static void enterCtxt(Context ctxt, Block block) {
    currentCtxt().addChild(ctxt);
    applyBlock(ctxt, block);
  }

  private static void applyBlock(Context ctxt, Block block) {
    ctxts.push(ctxt);
    doApplyBlock(block);
    ctxts.pop();
  }

  private static void doApplyBlock(Block block) {
    try {
      block.apply();
    } catch (Throwable e) {
      it("happen to an error", failing(e));
    }
  }

  private static Context currentCtxt() {
    return ctxts.peek();
  }
}
```

#### 上下文切换

但为了控制`Context`之间的「树型关系」(即`describe`的嵌套关系)，为此建立了一个`Stack`的机制，保证运行时在某一个时刻`Context`的唯一性。

只有`describe`的调用会开启「上下文的建立」，并完成上下文「父子关系」的链接。其余操作，例如`it, before, after`都是在当前上下文进行「元信息」的注册。

#### 虚拟的根结点

使用静态初始化块，完成「虚拟根结点」的注册；也就是说，在运行时初始化时，栈中已存在唯一的` Context("JSpec: All Specs")`虚拟根节点。

```java
public class JSpec {
  private static Deque<Context> ctxts = new ArrayDeque<Context>();

  static {
    ctxts.push(new Context(createSuiteDescription("JSpec: All Specs")));
  }
  
  ......
}
```

### 运行器

为了配合`JUnit`框架将`JSpec`运行起来，需要定制一个`JUnit`的`Runner`。

```java
public class JSpec extends Runner {
  private Description desc;
  private Context root;

  public JSpec(Class<?> suite) {
    desc = createSuiteDescription(suite);
    root = new Context(desc);
    enterCtxt(root, reflect(suite));
  }

  @Override
  public Description getDescription() {
    return desc;
  }

  @Override
  public void run(RunNotifier notifier) {
    root.exec(notifier);
  }
  
  ......
}
```

在编写用例时，使用`@RunWith(JSpec.class)`注解，告诉`JUnit`定制化了运行器的行为。

```java
@RunWith(JSpec.class)
public class JSpecs {{
  ......
}}
```

在之前已讨论过，`JSpec`的`run`无非就是将「以树形组织的」`Executor`集合调度起来。

#### 实现`reflect`

`JUnit`在运行时，首先看到了`@RunWith(JSpec.class)`注解，然后反射调用`JSpec`的构造函数。

```java
public JSpec(Class<?> suite) {
  desc = createSuiteDescription(suite);
  root = new Context(desc);
  enterCtxt(root, reflect(suite));
}
```

通过`Block.reflect`的工厂方法，将开始执行测试用例集的「初始化块」。

```java
public interface Block {
  void apply() throws Throwable;
  
  static Block reflect(Class<?> c) {
    return () -> {
      Constructor<?> cons = c.getDeclaredConstructor();
      cons.setAccessible(true);
      cons.newInstance();
    };
  }
}
```

此刻，被`@RunWith(JSpec.class)`注解标注的「初始化块」被执行。

```java
@RunWith(JSpec.class)
public class JSpecs {{
  ......
}}
```

在「初始化块」中顺序完成对`describe, it, before, after`等子句的调用，其中：

- `describe`开辟新的`Context`；
- `describe`可以递归地调用内部嵌套的`describe`；
- `describe`调用`it, before, after`时，将信息注册到了`Context`中；
- 最终`Runner.run`将`Executor`集合按照「树」的组织方式调度起来；

## GitHub

`JSpec`已上传至`GitHub：`[https://github.com/horance-liu/jspec](https://github.com/horance-liu/jspec)，代码细节请参考源代码。


