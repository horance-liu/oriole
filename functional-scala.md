# Functional Scala

> Functional programming leads to deep insights into the nature of computation. -- Martin Odersky.

## What's FP?

「函数式」是一种使用「纯函数」(`Pure Function`)组装程序的编程范式。

## History

### FP

> Object-oriented programming is an exceptionally bad idea which could only have originated in California. -- E.W. Dijkstra.

- Lisp
- Haskell

### OO

> You probably know that arrogance, in computer science, is measured in Nano-Dijkstras. -- Alan Kay.

- Smalltalk
- C++/Java

### What's Scala: Object-Oriented Meets Functional

```scala
// @author: Martin Odersky
def Scala = OO + FP
```

## Pure Functions

> Strive for Purity.

纯函数具有如下`2`个基本特征：

- 无副作用`(Side Effects Free)`
- 引用透明`(Referentially Transparent)`

### Side Effects Free

所谓「副作用」指的是程序运行时修改了全局变量，操作了`IO`等等。「副作用」常常导致错误的发生，并且降低程序的可读性。

- Reassigning a variable

> Prefer `val` to `var`

```scala
var i = 0
i += 1
```
- Modifying a data structure in place```scala
val x = new StringBuilder("hello")
x.append(", ").append("world")
```
- Setting a field on an object

```scala
class Person(var name: String)

val p = new Person("horance")
p.name = "Horance"
```
- Throwing an exception or halting with an error 

```scala
def divide(a: Int, b: Int) = 
  if (b != 0) a/b 
  else throw new ArithmeticException
```

- Operating on the IO

```scala
println("hello, world")
```

### Referential Transparency

所谓「引用透明」，就是指函数本身可以被返回值所替换，而没有影响程序外部行为的特性。`引用透明`的函数非常有利于重构，例如「`Replace Temp with Query`」的重构手法。

例如，`currentCtxt`是引用透明的，`ctxts.peek()`出现的任何地方，均可用`currentCtxt`所代替，以便消除重复。

```scala
object JSpec {
  private val ctxts = new ArrayDeque[Context]

  def before(block: => Unit) {
    currentCtxt.addBefore(block)
  }

  def after(block: => Unit) {
    currentCtxt.addAfter(block)
  }

  def currentCtxt: Context = ctxts.peek()
}
```

### Why Pure Funtions

- Easy to reason, refactor, and debug
- No order dependencies
- Parallelizable
- Cacheable
- Laziness

### How Pure Funtions

```scala
class Rational(var number: Ind, var denom: Int) {
  require(d != 0)
  
  def +(rhs: Rational) {
    number = number * rhs.denom + rhs.number * denom
    denom  = denom * rhs.denom
  }
}
```

如何做才能得到「纯函数」的设计呢？

- Side Effects Free
- Immutable Data

```scala
class Rational(val number: Ind, val denom: Int) {
  require(d != 0)
  
  def +(rhs: Rational) = new Rational( 
    number * rhs.denom + rhs.number * denom,
    denom * rhs.denom)
}
```

- `Rational`是「不可变的」，它使用`val`定义类中的成员；
- `+`是「无副作用」(`Side-Effect Free`)；
- `+`是一个`Pure Function`，对于相同的输入值，永远得到相同的结果；
- `Rational`实例只存在唯一的状态；
- `+`是「引用透明」的，为此`+`的返回值是「上下文自由」的，与时间，顺序等因素无关。

### Mutablity VS. Immutablity

> Minimize mutability.

`Immutablity`相对于`Mutablity`，存在若干方面的优势：

- 易于理解：可变对象随着时间推移，状态空间也随之变得复杂多变；
- 自由：可变对象在传递时，可能需要拷贝；
- 缓存：使用`Flyweights`缓存昂贵的资源、操作；
- 并发：可变对象在多线程的环境下是线程不安全的；
- 安全：如果`hashCode`依赖于可变对象，是及其不安全的；

`Immutablity`相对于`Mutablity`，存在唯一的劣势：

- 性能：在某些场景，不可变性可能会成为性能的瓶颈
 
如何解决这个问题呢？同时提供可变与不可变的实现版本：

- `String` VS. `StringBuilder`
- `scala.collection.mutable` VS. `scala.collection.immutabl`

## Lambda

```scala
List(1, 2, 3).((x: Int) => 2 * x)
```

`(x: Int) => 2 * x`是一个「匿名函数」，也称为「`lambda`」表达式，其类型为：`Function[Int, Int]`。

根据「类型推演」，可以略去参数的类型修饰符：

```scala
List(1, 2, 3).(x => 2 * x)
```

因为`x`在函数体内只出现一次，可以使用「占位符」简化代码：

```scala
List(1, 2, 3).(2*_)
```

## Higher Order Functions

> Functions are things.

函数作为一等公民`(Function as first class)`，可以作为值自由地被传递和存储；接纳函数作为参数，或者返回函数的函数称为「高阶函数」。高阶函数是函数式编程复用的重要基础设施。

### 传递函数

```scala
args.foreach(println)
```

### 返回函数

```scala
def multiply(factor: Int) = 
  (x: Int) => x * factor
  
val triple = multiply(3)
val double = multiply(2)

println(triple(10))  // 30
println(double(10))  // 20
```

## Closure

「闭包」是函数式的一个重要特性，`lambda`表达式将其「上下文」自动地进行传递。

```java

```

## Currying

```scala
@annotation.tailrec
def loop(cond: => Boolean)(body: => Unit) {
  if (cond) {
    body
	 loop(cond, body)
  }
}
```

- 按名传递：`Pass-By-Name`
- 延迟计算：`Lazy Evaluation`
- 无参函数：`Zero Parameter Function`

```scala
var i = 0
loop (i < 10) {
  println(i)
  i += 1
}
```

## Partially Applied Functions

```scala
def sum(a: Int, b: Int, c: Int) = a + b + c

println(sum _)
```

## Composition

> Composition everywhere.

### Function Composition


### Algebraic Types


## Apply FP

> Parameterize all the things

> Every function is a one parameter function

## Understanding FP

> OO makes code understandable by encapsulating moving parting, but FP makes code understandable by minimizing moving parts. －Michael Feathers


## 附录

[Functional Design Patterns](http://www.infoq.com/presentations/fp-design-patterns)

