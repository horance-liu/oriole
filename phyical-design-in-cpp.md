#浅谈C++物理设计

**刘光聪**，程序员，敏捷教练，开源软件爱好者，具有多年大型遗留系统的重构经验，对`OO`，`FP`，`DSL`等领域具有浓厚的兴趣。

- GitHub: [https://github.com/horance-liu](https://github.com/horance-liu)
- Email: [horance@outlook.com](horance@outlook.com)

## 代码规范

### 头文件保护宏

每一个头文件都应该具有独一无二的保护宏，并保持命名规则的一致性，其中命名规则包括两种风格：

- `INCL_<PROJECT>_<MODULE>_<FILE>_H`
- 全局唯一的随机序列码

第一种命名规则问题在于：当文件名重命名或移动目录时，需要同步修改头文件保护宏；推荐使用`IDE`随机自动地生成头文件保护宏，其更加快捷、简单、安全、有效。

**反例：**
```cpp
// thread/Runnable.h
// 因名称太短，存在名字冲突的可能性
#ifndef RUNNABLE_H
#define RUNNABLE_H

#include "base/Role.h"

DEFINE_ROLE(Runnable)
{
    ABSTRACT(void run());
};

#endif
```

**正例：**
```cpp
// cppunit/AutoRegisterSuite.h
#ifndef INCL_CPPUNIT_AUTO_REGISTER_SUITE_H
#define INCL_CPPUNIT_AUTO_REGISTER_SUITE_H

#include "base/Role.h"

struct TestSuite;

DEFINE_ROLE(AtuoRegisterSuite)
{
    ABSTRACT(void add(TestSuite&));
};

#endif
```

**正例：**
```cpp
// cppunit/AutoRegisterSuite.h
// IDE自动生成
#ifndef INCL_ADCM_LLL_3465_DCPOE_ACLDDDE_479_YTEY_H
#define INCL_ADCM_LLL_3465_DCPOE_ACLDDDE_479_YTEY_H

#include "base/Role.h"

struct TestSuite;

DEFINE_ROLE(AtuoRegisterSuite)
{
    ABSTRACT(void add(TestSuite&));
};

#endif
```

### 下划线与驼峰

路径名一律使用小写、下划线或中划线风格的名称；文件名应该与程序主要实体名称相同，可以使用驼峰命名，也可以使用小写、下划线或中划线分割的名字；实现文件的名字必须和头文件保持一致；包含头文件时，必须保持路径名、文件名大小写敏感。

**反例：**
```cpp
// 路径名htmlParser使用了驼峰命名风格
#include "htmlParser/core/Attribute.h"
```

**正例：**
```cpp
// 正确的头文件包含
#include "html-parser/core/Attribute.h"
#include "yaml_parser.h"
```

最后，值得注意的是，团队内必须保持一致的命名风格。

### 大小写敏感

包含头文件时，必须保持路径名、文件名大小写敏感。因为在\ascii{Windows}，其大小写不敏感，编译时检查失效，代码失去了可移植性，所以在包含头文件时必须保持文件名的大小写敏感。

假如存在两个物理文件名分别为`SynchronizedObject.h, yaml_parser.h`的两个文件。

**反例：**
```cpp
// 路径名、文件名大小写与真实物理路径、物理文件名称不符
#include "CppUnit/Core/SynchronizedObject.h"
#include "YAML_Parser.h"
```

**正例：**
```cpp
#include "cppunit/core/SynchronizedObject.h"
#include "yaml_parser.h"
```

最后，值得注意的是，团队内必须保持一致的命名风格。

### 分隔符

包含头文件时，路径分隔符一律使用`Unix`风格，拒绝使用`Windows`风格；即采用`/`而不是使用`\`分割路径。

**反例：**
```cpp
// 使用了Windows风格的路径分割符
#include "cppunit\core\SynchronizedObject.h"
```

**正例：**
```cpp
// 使用了Unix风格的路径分割符
#include "cppunit/core/SynchronizedObject.h"
```

### `extern "C"`

使用`extern "C"`时，不要包括`include`语句。

**反例：**
```cpp
//oss/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
    
#ifdef  __cplusplus
extern "C" {
#endif

// 错误地将include放在了extern "C"中
#include "oss_common.h"

void* oss_alloc(size_t);
void  oss_free(void*);

#ifdef  __cplusplus
}
#endif

#endif
```

**正例：**
```cpp
//oss/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
    
#include "oss_common.h"

#ifdef  __cplusplus
extern "C" {
#endif

void* oss_alloc(size_t);
void  oss_free(void*);

#ifdef  __cplusplus
}
#endif

#endif
```

### 兼容性

当以`C`提供实现时，头文件中必须使用`extern "C"`声明，以便支持`C++`的扩展。

**反例：**
```cpp
// oss/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8    

#include "oss_common.h"

void* oss_alloc(size_t);
void  oss_free(void*);

#endif
```

**正例：**
```cpp
// oss/oss_memery.h
#ifndef HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8
#define HF0916DFB_1CD1_4811_B82B_9B8EB1A007D8    

#include "oss_common.h"

#ifdef  __cplusplus
extern "C" {
#endif

void* oss_alloc(size_t);
void  oss_free(void*);

#ifdef  __cplusplus
}
#endif

#endif
```

### 上帝头文件

拒绝创建巨型头文件，将所有实体声明都放到头文件中，而仅仅将外部依赖的实体声明放到头文件中。


#### 信息隐藏

实现文件也是一种信息隐藏的惯用技术，如果一些程序的实体不对外所依赖，则放在自己的实现文件中，一则可降低依赖关系，二则实现更好的信息隐藏。

对于上帝头文件，其很多声明和定义本来是不应该放到头文件，而应该放会实现文件以便实现更好地信息隐藏。

#### 编译时依赖

巨型头文件必然造成了巨大的编译时依赖，不仅仅带来巨大的编译时开销，更重要的是这样的设计将太多的实现细节暴露给用户，导致后续版本兼容性的问题，阻碍了头文件进一步演进、修改、扩展的可能性，从而失去了软件的可扩展性。

#### `include`顺序依赖

不要认为提供一个大而全的头文件会给你的用户带来方便，用户因此而更加困扰。对于一个巨大的头文件，其依赖关系很难一眼看清楚，其自满足性很难得到保证，用户在包含此头文件时，还要关心头文件之间的依赖关系，甚至关心include语句的顺序，但这样的代码实现是及其脆弱的。

## 物理设计原则

| 原则           | 基本含义                     |
| -------------  |--------------------------- |
| 自满足原则       | 头文件本身是可以编译通过的     |
| 单一职责原则     | 头文件包含的实体的职责是单一的  |
| 最小依赖原则     | 绝不包含不必要的头文件         |
| 最小可见性原则   | 尽量封装隐藏类的成员           |

### 自满足原则

所有头文件都应该自满足的。看一个具体的示例代码，这里定义了一个`TestCase.h`头文件。`TestCase`对父类`TestLeaf, TestFixture`都存在编译时依赖，但没有包含基类的头文件。

**反例：**
```cpp
// cppunit/TestCase.h
#ifndef EOPTIAWE_23908576823_MSLKJDFE_0567925
#define EOPTIAWE_23908576823_MSLKJDFE_0567925    

struct TestCase : TestLeaf, TestFixture
{
    TestCase(const std::string &name="");
    
private:
    OVERRIDE(void run(TestResult *result));
    OVERRIDE(std::string getName() const);

private:
    ABSTRACT(void runTest());
    
private:
    const std::string name;
};

#endif
```

为了满足自满足原则，其自身必须包含其所有父类的头文件。

**正例：**
```cpp
// cppunit/TestCase.h
#ifndef EOPTIAWE_23908576823_MSLKJDFE_0567925
#define EOPTIAWE_23908576823_MSLKJDFE_0567925    

#include "cppunit/core/TestLeaf.h"
#include "cppunit/core/TestFixture.h"

struct TestCase : TestLeaf, TestFixture
{
    TestCase(const std::string &name="");
    
private:
    OVERRIDE(void run(TestResult &result));
    OVERRIDE(std::string getName() const);

private:
    ABSTRACT(void runTest());
    
private:
    const std::string name;
};

#endif
```

即使`TestCase`直接持有`name`的成员变量，但没有必要包含`std::string`的头文件，因为`TestCase`覆写了其父类的`getName`成员函数，父类为了保证自满足原则，自然已经包含了`std::string`的头文件。

同样的原因，也没有必要在此前置声明`TestResult`，因为父类可定已经声明过了。

### 单一职责

这是`SRP(Single Reponsibility
Priciple)`在头文件设计时的一个具体运用。头文件如果包含了其它不相关的元素，则包含该头文件的所有实现文件都将被这些不相关的元素所污染，重编译将成为一件高概率的事件。

如示例代码，将`OutputStream, InputStream`同时定义在一个头文件中，将违背该原则。本来只需只读接口，无意中被只写接口所污染。

**反例：**
```cpp
// io/Stream.h
#ifndef LDGOUIETA_437689Q20_ASIOHKFGP_980341
#define LDGOUIETA_437689Q20_ASIOHKFGP_980341

#include "base/Role.h"

DEFINE_ROLE(OutputStream)
{
    ABSTRACT(void write());
};

DEFINE_ROLE(InputStream)
{
    ABSTRACT(void read());
};

#endif
```

**正例：** 先创建一个`OutputStream.h`文件：
```cpp
// io/OutputStream.h
#ifndef LDGOUIETA_437689Q20_ASIOHKFGP_010234
#define LDGOUIETA_437689Q20_ASIOHKFGP_010234
    
#include "base/Role.h"

DEFINE_ROLE(OutputStream)
{
    ABSTRACT(void write());
};

#endif
```

再创建一个`InputStream.h`文件：

```cpp
// io/InputStream.h
#ifndef LDGOUIETA_437689Q20_ASIOHKFGP_783621
#define LDGOUIETA_437689Q20_ASIOHKFGP_783621

#include "base/Role.h"

DEFINE_ROLE(InputStream)
{
    ABSTRACT(void read());
};

#endif
```

### 最小依赖

一个头文件只应该包含必要的实体，尤其在头文件中仅仅对实体的声明产生依赖，那么前置声明是一种有效的降低编译时依赖的技术。

**反例：**
```cpp
// cppunit/Test.h
#ifndef PERTP_8792346_QQPKSKJ_09472_HAKHKAIE
#define PERTP_8792346_QQPKSKJ_09472_HAKHKAIE

#include <base/Role.h>
#include <cppunit/core/TestResult.h>
#include <string>

DEFINE_ROLE(Test)
{
    ABSTRACT(void run(TestResult& result));
    ABSTRACT(int countTestCases() const);
    ABSTRACT(int getChildTestCount() const);
    ABSTRACT(std::string getName() const);
};

#endif
```

如示例代码，定义了一个`xUnit`框架中的`Test`顶级接口，其对`TestResult`的依赖仅仅是一个声明依赖，并没有必要包含`TestResult.h`，前置声明是解开这类编译依赖的钥匙。

值得注意的是，对标准库`std::string`的依赖，即使它仅作为返回值，但因为它实际上是一个`typedef`，所以必须老实地包含其对应的头文件。事实上，如果产生了对标准库名称的依赖，基本上都需要包含对应的头文件。

另外，对`DEFINE_ROLE`宏定义的依赖则需要包含相应的头文件，以便实现该头文件的自满足。

但是，`TestResult`仅作为成员函数的参数出现在头文件中，所以对`TestResult`的依赖只需前置声明即可。

**正例：**
```cpp
// cppunit/Test.h
#ifndef PERTP_8792346_QQPKSKJ_09472_HAKHKAIE
#define PERTP_8792346_QQPKSKJ_09472_HAKHKAIE

#include <base/Role.h>
#include <string>

struct TestResult;

DEFINE_ROLE(Test)
{
    ABSTRACT(void run(TestResult& result));
    ABSTRACT(int countTestCases() const);
    ABSTRACT(int getChildTestCount() const);
    ABSTRACT(std::string getName() const);
};

#endif
```

在选择包含头文件还是前置声明时，很多程序员感到迷茫。其实规则很简单，在如下场景前置声明即可，无需包含头文件：

- 指针
- 引用
- 返回值
- 函数参数

相反地，如果编译器需要知道实体的真正内容时，则必须包含头文件，此依赖也常常称为强编译时依赖。强编译时依赖主要包括如下几种场景：

- `typedef`定义的实体
- 继承
- 宏
- `inline`
- `template`
- 引用类内部成员时
- `sizeof`运算

### 最小可见性


在头文件中定义一个类时，清晰、准确的`public, protected, private`是传递设计意图的指示灯。其中`private`做为一种实现细节被隐藏起来，为适应未来不明确的变化提供便利的措施。

不要将所有的实体都`public`，这无疑是一种自杀式做法。应该以一种相反的习惯性思维，尽最大可能性将所有实体`private`，直到你被迫不得不这么做为止，依次放开可见性的权限。

如下例代码所示，按照`public-private, function-data`依次排列类的成员，并对具有相同特征的成员归类，将大大改善类的整体布局，给读者留下清晰的设计意图。

**反例：**
```cpp 
//trans-dsl/sched/SimpleAsyncAction.h
#ifndef IUOTIOUQW_NMAKLKLG_984592_KJSDKLJFLK
#define IUOTIOUQW_NMAKLKLG_984592_KJSDKLJFLK

#include "trans-dsl/action/Action.h"
#include "trans-dsl/utils/EventHandlerRegistry.h"

struct SimpleAsyncAction : Action
{
    template<typename T>
    Status waitOn(const EventId eventId, T* thisPointer,
             Status (T::*handler)(const TransactionInfo&, const Event&), 
             bool forever = false)
    {
        return registry.addHandler(eventId, thisPointer, handler, forever);
    }

    Status waitUntouchEvent(const EventId eventId);

    OVERRIDE(Status handleEvent(const TransactionInfo&, const Event&));
    OVERRIDE(void kill(const TransactionInfo&, const Status)); 

    DEFAULT(void, doKill(const TransactionInfo&, const Status));

    EventHandlerRegistry registry;
};

#endif
```

**正例：**
```cpp
// trans-dsl/sched/SimpleAsyncAction.h
#ifndef IUOTIOUQW_NMAKLKLG_984592_KJSDKLJFLK
#define IUOTIOUQW_NMAKLKLG_984592_KJSDKLJFLK

#include "trans-dsl/action/Action.h"
#include "trans-dsl/utils/EventHandlerRegistry.h"

struct SimpleAsyncAction : Action
{
    template<typename T>
    Status waitOn(const EventId eventId, T* thisPointer,
             Status (T::*handler)(const TransactionInfo&, const Event&), 
             bool forever = false)
    {
        return registry.addHandler(eventId, thisPointer, handler, forever);
    }

    Status waitUntouchEvent(const EventId eventId);

private:
    OVERRIDE(Status handleEvent(const TransactionInfo&, const Event&));
    OVERRIDE(void kill(const TransactionInfo&, const Status)); 

private:
    DEFAULT(void, doKill(const TransactionInfo&, const Status));

private:
    EventHandlerRegistry registry;
};

#endif
```

##  最佳实践

### 如何保证自满足

为了验证头文件设计的**自满足原则**，实现文件的第一条语句必然是包含其对应的头文件。

**反例：**
```cpp
// cppunit/TestCase.cpp
#include "cppunit/core/TestResult.h"
#include "cppunit/core/Functor.h"
// 错误：没有放在第一行，无法校验其自满足性
#include "cppunit/core/TestCase.h"

namespace
{
    struct TestCaseMethodFunctor : Functor
    {
        typedef void (TestCase::*Method)();
    
        TestCaseMethodFunctor(TestCase &target, Method method)
           : target(target), method(method)
        {}
    
        bool operator()() const
        {
            target.*method();
            return true;
        }
    
    private:
        TestCase &target;
        Method method;
    };
}

void TestCase::run(TestResult &result)
{
    result.startTest(*this);
  
    if (result.protect(TestCaseMethodFunctor(*this, &TestCase::setUp)))
    {
        result.protect(TestCaseMethodFunctor(*this, &TestCase::runTest)); 
    }

    result.protect(TestCaseMethodFunctor(*this, &TestCase::tearDown));

    result.endTest(*this);
}

...
```

**正例：**
```cpp
// cppunit/TestCase.cpp
#include "cppunit/core/TestCase.h"
#include "cppunit/core/TestResult.h"
#include "cppunit/core/Functor.h"

namespace
{
    struct TestCaseMethodFunctor : Functor
    {
        typedef void (TestCase::*Method)();
    
        TestCaseMethodFunctor(TestCase &target, Method method)
           : target(target), method(method)
        {}
    
        bool operator()() const
        {
            target.*method();
            return true;
        }
    
    private:
        TestCase &target;
        Method method;
    };
}

void TestCase::run(TestResult &result)
{
    result.startTest(*this);
  
    if (result.protect(TestCaseMethodFunctor(*this, &TestCase::setUp))
    {
        result.protect(TestCaseMethodFunctor(*this, &TestCase::runTest)); 
    }

    result.protect(TestCaseMethodFunctor(*this, &TestCase::tearDown));

    result.endTest(*this);
}

...
```

### `override`和`private`

所有`override`的函数(除`override`的`virtual`析构函数之外)都应该是`private`的，以保证按接口编程的良好设计原则。

**反例：**
```cpp
// html-parser/filter/AndFilter.h
#ifndef EOIPWORPIO_06123124_NMVBNSDHJF_497392
#define EOIPWORPIO_06123124_NMVBNSDHJF_497392

#include "html-parser/filter/NodeFilter.h"
#include <list>

struct AndFilter : NodeFilter
{
     void add(NodeFilter*);

     // 设计缺陷：本应该private
     OVERRIDE(bool accept(const Node&) const);

private:
     std::list<NodeFilter*> filters;
};

#endif
```

**正例：**
```cpp
// html-parser/filter/AndFilter.h
#ifndef EOIPWORPIO_06123124_NMVBNSDHJF_497392
#define EOIPWORPIO_06123124_NMVBNSDHJF_497392

#include "html-parser/filter/NodeFilter.h"
#include <list>

struct AndFilter : NodeFilter
{
     void add(NodeFilter*);

private:
     OVERRIDE(bool accept(const Node&) const);

private:
     std::list<NodeFilter*> filters;
};

#endif
```

### `inline`

#### 避免头文件中`inline`

头文件中避免定义`inline`函数，除非性能报告指出此函数是性能的关键瓶颈。

`C++`语言将声明和实现进行分离，程序员为此不得不在头文件和实现文件中重复地对函数进行声明。这是`C/C++`天生给我们的设计带来的重复。这是一件痛苦的事情，驱使部分程序员直接将函数实现为`inline`。

但`inline`函数的代码作为一种不稳定的内部实现细节，被放置在头文件里，其变更所导致的大面积的重新编译是个大概率事件，为改善微乎其微的函数调用性能与其相比将得不偿失。

除非有相关`profiling`性能测试报告，表明这部分关键的热点代码需要被放回头文件中。

但需要注意在特殊的情况，可以将实现`inline`在头文件中，因为为它们创建实现文件过于累赘和麻烦。

- `virtual`析构函数
- 空的`virtual`函数实现
- `C++11`的`default`函数

#### 鼓励实现文件中`inline`

对于在编译单元内部定义的类而言，因为它的客户数量是确定的，就是它本身。另外，由于它本来就定义在源代码文件中，因此并没有增加任何“物理耦合”。所以，对于这样的类，我们大可以将其所有函数都实现为`inline`的，就像写`Java`代码那样，`Once & Only Once`。

以单态类的一种实现技术为例，讲解编译时依赖的解耦与匿名命名空间的使用。(首先，应该抵制单态设计的诱惑，单态其本质是面向对象技术中全局变量的替代品。滥用单态模式，犹如滥用全局变量，是一种典型的设计坏味道。只有确定在系统中唯一存在的概念，才能使用单态模式)。

实现单态，需要对系统中唯一存在的概念进行封装；但这个概念往往具有巨大的数据结构，如果将其声明在头文件中，无疑造成很大的编译时依赖。

**反例：**
```cpp
// ne/NetworkElementRepository.h
#ifndef UIJVASDF_8945873_YUQWTYRDF_85643
#define UIJVASDF_8945873_YUQWTYRDF_85643    

#include "base/Status.h"
#include "base/BaseTypes.h"
#include "transport/ne/NetworkElement.h"
#include <vector>

struct NetworkElementRepository
{
    static NetworkElement& getInstance();

    Status add(const U16 id);
    Status release(const U16 id);
    Status modify(const U16 id);
    
private:
    typedef std::vector<NetworkElement> NetworkElements;
    NetworkElements elements;
};

#endif
```

受文章篇幅的所限，`NetworkElement.h`未列出所有代码实现，但我们知道`NetworkElement`拥有巨大的数据结构，上述设计导致所有包含`NetworkElementRepository`的头文件都被`NetworkElement`所间接污染。

此时，其中可以将依赖置入到实现文件中，解除揭开其严重的编译时依赖。更重要的是，它更好地遵守了按接口编程的原则，改善了软件的扩展性。

**正例：**
```cpp
// ne/NetworkElementRepository.h
#ifndef UIJVASDF_8945873_YUQWTYRDF_85643
#define UIJVASDF_8945873_YUQWTYRDF_85643    

#include "base/Status.h"
#include "base/BaseTypes.h"
#include "base/Role.h"

DEFINE_ROLE(NetworkElementRepository)
{
    static NetworkElementRepository& getInstance();

    ABSTRACT(Status add(const U16 id));
    ABSTRACT(Status release(const U16 id));
    ABSTRACT(Status modify(const U16 id));
};

#endif
```

其实现文件包含`NetworkElement.h`，将对其的依赖控制在本编译单元内部。

```cpp
// ne/NetworkElementRepository.cpp}]
#include "transport/ne/NetworkElementRepository.h"
#include "transport/ne/NetworkElement.h"
#include <vector>

namespace
{
    struct NetworkElementRepositoryImpl : NetworkElementRepository
    {
        OVERRIDE(Status add(const U16 id))
        {
            // inline implements
        }

        OVERRIDE(Status release(const U16 id))
        {
            // inline implements
        }

        OVERRIDE(Status modify(const U16 id))
        {
            // inline implements
        }
    
    private:
        typedef std::vector<NetworkElement> NetworkElements;
        NetworkElements elements;
    };
}

NetworkElementRepository& NetworkElementRepository::getInstance()
{
    static NetworkElementRepositoryImpl inst;
    return inst;
}
```

此处，对`NetworkElementRepositoryImpl`类的依赖是非常明确的，仅本编译单元内，所有可以直接进行`inline`，从而简化了很多实现。

### 匿名`namespace`

匿名`namespace`的存在常常被人遗忘，但它的确是一个利器。匿名`namespace`的存在，使得所有受限于编译单元内的实体拥有了明确的处所。

自此之后，所有`C`风格并局限于编译单元内的`static`函数和变量；以及类似`Java`中常见的`private static`的提取函数将常常被匿名`namespace`替代。

请记住匿名命名空间也是一种重要的信息隐藏技术。在实现文件中提倡使用匿名`namespace`, 以避免潜在的命名冲突。

如上例，`NetworkElementRepository.cpp`通过匿名`namespace`，极大地减低了其头文件的编译时依赖。

### `struct` VS. `class`

除了名字不同之外，`class`和`struct`唯一的差别是：默认可见性。这体现在定义和继承时。`struct`在定义一个成员，或者继承时，如果不指明，则默认为`public`，而`class`则默认为`private`。

但这些都不是重点，重点在于定义接口和继承时，冗余`public`修饰符总让人不舒服。简单设计四原则告诉告诉我们，所有冗余的代码都应该被剔除。

但很多人会认为`struct`是`C`遗留问题，应该避免使用。但这不是问题，我们不应该否认在写`C++`程序时，依然在使用着很多`C`语言遗留的特性。关键在于，我们使用的是`C`语言中能给设计带来好处的特性，何乐而不为呢？

**正例：**
```cpp
// hamcrest/SelfDescribing.h
#ifndef OIWER_NMVCHJKSD_TYT_48457_GSDFUIE
#define OIWER_NMVCHJKSD_TYT_48457_GSDFUIE

struct Description;

struct SelfDescribing
{
    virtual void describeTo(Description& description) const = 0;
    virtual ~SelfDescribing() {}
};

#endif
```

**反例：**
```cpp
// hamcrest/SelfDescribing.h
#ifndef OIWER_NMVCHJKSD_TYT_48457_GSDFUIE
#define OIWER_NMVCHJKSD_TYT_48457_GSDFUIE

class Description;

class SelfDescribing
{
public:
    virtual void describeTo(Description& description) const = 0;
    virtual ~SelfDescribing() {}
};

#endif
```

更重要的是，我们确信“抽象”和“信息隐藏”对于软件的重要性，这促使我将`public`接口总置于类的最前面成为我们的首选，`class`的特性正好与我们的期望背道而驰(`class`的特性正好适合于将数据结构捧为神物的程序员，它们常常将数据结构置于类声明的最前面。)

不管你信仰那一个流派，切忌不能混合使用`class`和`struct`。在大量使用前导声明的情况下，一旦一个使用`struct`的类改为`class`，所有的前置声明都需要修改。

### 万恶的`struct tag`

定义`C`风格的结构体时，`struct tag`彻底抑制了结构体前置声明的可能性，从而阻碍了编译优化的空间。

**反例：**
```cpp
// radio/domain/Cell.h
#ifndef AQTYER_023874_NMHSFHKE_7432378293
#define AQTYER_023874_NMHSFHKE_7432378293

typedef struct tag_Cell
{
    WORD16 wCellId;
    WORD32 dwDlArfcn;
} T_Cell;

#endif
```

```cpp
// radio/domain/Cell.h
#ifndef AQTYER_023874_NMHSFHKE_7432378293
#define AQTYER_023874_NMHSFHKE_7432378293

typedef struct
{
    WORD16 wCellId;
    WORD32 dwDlArfcn;
} T_Cell;

#endif
```

为了兼容`C`并为结构体前置声明提供便利，如下解法是最合适的。

**正例：**
```cpp
// radio/domain/Cell.h
#ifndef AQTYER_023874_NMHSFHKE_7432378293
#define AQTYER_023874_NMHSFHKE_7432378293

typedef struct T_Cell
{
    WORD16 wCellId;
    WORD32 dwDlArfcn;
} T_Cell;

#endif
```

需要注意的是，在`C`语言中，如果没有使用`typedef`，则定义一个结构体的指针，必须显式地加上`struct`关键字：`struct T_Cell *pcell`，而`C++`没有这方面的要求。

### `PIMPL`

如果性能不是关键问题，考虑使用`PIMPL`降低编译时依赖。

**反例：**
```cpp
// mockcpp/ApiHook.h
#ifndef OIWTQNVHD_10945_HDFIUE_23975_HFGA
#define OIWTQNVHD_10945_HDFIUE_23975_HFGA

#include "mockcpp/JmpOnlyApiHook.h"

struct ApiHook
{
    ApiHook(const void* api, const void* stub)
      : stubHook(api, stub)
    {}

private:
    JmpOnlyApiHook stubHook;
};

#endif
```

**正例：**
```cpp
// mockcpp/ApiHook.h
#ifndef OIWTQNVHD_10945_HDFIUE_23975_HFGA
#define OIWTQNVHD_10945_HDFIUE_23975_HFGA

struct ApiHookImpl;

struct ApiHook
{
    ApiHook(const void* api, const void* stub);
    ~ApiHook();

private:
    ApiHookImpl* This;
};

#endif
```

```cpp
// mockcpp/ApiHook.cpp
#include "mockcpp/ApiHook.h"
#include "mockcpp/JmpOnlyApiHook.h"

struct ApiHookImpl
{
   ApiHookImpl(const void* api, const void* stub)
     : stubHook(api, stub)
   {
   }

   JmpOnlyApiHook stubHook;
};

ApiHook::ApiHook( const void* api, const void* stub)
  : This(new ApiHookImpl(api, stub))
{
}

ApiHook::~ApiHook()
{
    delete This;
}
```

通过`ApiHookImpl* This`的桥接，在头文件中解除了对`JmpOnlyApiHook`的依赖，将其依赖控制在本编译单元内部。

### `template`

#### 编译时依赖

当选择模板时，不得不将其实现定义在头文件中。当编译时依赖开销非常大时，编译模板将成为一种负担。设法降低编译时依赖，不仅仅为了缩短编译时间，更重要的是为了得到一个低耦合的实现。

**反例：**
```cpp
// oss/OssSender.h
#ifndef HGGAOO_4611330_NMSDFHW_86794303_HJHASI
#define HGGAOO_4611330_NMSDFHW_86794303_HJHASI

#include "pub_typedef.h"
#include "pub_oss.h"
#include "oss_comm.h"
#include "pub_commdef.h"
#include "base/Assertions.h"
#include "base/Status.h"

struct OssSender
{
    OssSender(const PID& pid, const U8 commType)
      : pid(pid), commType(commType)
    {
    }
    
    template <typename MSG>
    Status send(const U16 eventId, const MSG& msg)
    {
        DCM_ASSERT_TRUE(OSS_SendAsynMsg(eventId, &msg, sizeof(msg), commType,(PID*)&pid) == OSS_SUCCESS); 
        return DCM_SUCCESS;
    }

private:
    PID pid;
    U8 commType;
};

#endif
```

为了实现模板函数`send`，将`OSS`的一些实现细节暴露到了头文件中，包含`OssSender.h`的所有文件将无意识地产生了对`OSS`头文件的依赖。

提取一个私有的`send`函数，并将对`OSS`的依赖移入到`OssSender.cpp`中，对`PID`依赖通过前置声明解除，最终实现如代码所示。

**正例：**
```cpp
// oss/OssSender.h
#ifndef HGGAOO_4611330_NMSDFHW_86794303_HJHASI
#define HGGAOO_4611330_NMSDFHW_86794303_HJHASI

#include "base/Status.h"
#include "base/BaseTypes.h"

struct PID;

struct OssSender
{
    OssSender(const PID& pid, const U16 commType)
      : pid(pid), commType(commType) 
    {
    }
    
    template <typename MSG>
    Status send(const U16 eventId, const MSG& msg)
    {
        return send(eventId, (const void*)&msg, sizeof(MSG));    
    }

private:
    Status send(const U16 eventId, const void* msg, size_t size);

private:
    const PID& pid;
    U8 commType;
};

#endif
```

识别哪些与泛型相关，哪些与泛型无关的知识，并解开此类编译时依赖是`C++`程序员的必备之技。

#### 显式模板实例化

模板的编译时依赖存在两个基本模型：包含模型，`export`模型。`export`模型受编译技术实现的挑战，最终被`C++11`标准放弃。

此时，似乎我们只能选择包含模型。其实，存在一种特殊的场景，适时选择显式模板实例化`(Explicit Template Instantiated)`，降低模板的编译时依赖。是能做到降低模板编译时依赖的。

**反例：**
```cpp
// quantity/Quantity.h
#ifndef HGGQMVJK_892302_NGFSLEU_796YJ_GF5284
#define HGGQMVJK_892302_NGFSLEU_796YJ_GF5284

#include <quantity/Amount.h>

template <typename Unit>
struct Quantity
{
    Quantity(const Amount amount, const Unit& unit)      
      : amountInBaseUnit(unit.toAmountInBaseUnit(amount))
    {}
    
    bool operator==(const Quantity& rhs) const
    {
        return amountInBaseUnit == rhs.amountInBaseUnit;
    }
    
    bool operator!=(const Quantity& rhs) const
    {
        return !(*this == rhs);
    }
    
private:
    const Amount amountInBaseUnit;
};

#endif
```

```cpp
// quantity/Length.h
#ifndef TYIW7364_JG6389457_BVGD7562_VNW12_JFH
#define TYIW7364_JG6389457_BVGD7562_VNW12_JFH
 
#include "quantity/Quantity.h"
#include "quantity/LengthUnit.h"

typedef Quantity<LengthUnit> Length;

#endif
```

```cpp
// quantity/Volume.h
#ifndef HG764MD_NKGJKDSJLD_RY64930_NVHF977E
#define HG764MD_NKGJKDSJLD_RY64930_NVHF977E
 
#include "quantity/Quantity.h"
#include "quantity/VolumeUnit.h"

typedef Quantity<VolumeUnit> Volume;

#endif
```

如上的设计，泛型类`Quantity`的实现都放在了头文件，不稳定的实现细节，例如计算`amountInBaseUnit`的算法变化等因素，将导致包含`Length`或`Volume`的所有源文件都需要重新编译。

更重要的是，因为`LengthUnit, VolumeUnit`头文件的包含，如果因需求变化需要增加支持的单位，将间接导致了包含`Length`或`Volume`的所有源文件也需要重新编译。

如何控制和隔离`Quantity, LengthUnit, VolumeUnit`变化的蔓延，而避免大部分的客户代码重新编译，从而与客户彻底解偶呢？可以通过显式模板实例化将模板实现从头文件中剥离出去，从而避免了不必要的依赖。

**正例：**
```cpp 
// quantity/Quantity.h
#ifndef HGGQMVJK_892302_NGFSLEU_796YJ_GF5284
#define HGGQMVJK_892302_NGFSLEU_796YJ_GF5284

#include <quantity/Amount.h>

template <typename Unit>
struct Quantity
{
    Quantity(const Amount amount, const Unit& unit);
        
    bool operator==(const Quantity& rhs) const;
    bool operator!=(const Quantity& rhs) const;
        
private:
    const Amount amountInBaseUnit;
};

#endif
```

```cpp
// quantity/Quantity.tcc
#ifndef FKJHJT68302_NVGKS97474_YET122_HEIW8565
#define FKJHJT68302_NVGKS97474_YET122_HEIW8565

#include <quantity/Quantity.h>

template <typename Unit>
Quantity<Unit>::Quantity(const Amount amount, const Unit& unit)      
  : amountInBaseUnit(unit.toAmountInBaseUnit(amount))
{}
    
template <typename Unit>
bool Quantity<Unit>::operator==(const Quantity& rhs) const
{
    return amountInBaseUnit == rhs.amountInBaseUnit;
}
    
template <typename Unit>
bool Quantity<Unit>::operator!=(const Quantity& rhs) const
{
    return !(*this == rhs);
}

#endif
```

```cpp
// quantity/Length.h
#ifndef TYIW7364_JG6389457_BVGD7562_VNW12_JFH
#define TYIW7364_JG6389457_BVGD7562_VNW12_JFH

#include "quantity/Quantity.h"

struct LengthUnit;
struct Length : Quantity<LengthUnit> {};

#endif
```

```cpp
// quantity/Length.cpp
#include "quantity/Quantity.tcc"
#include "quantity/LengthUnit.h"

template struct Quantity<LengthUnit>;
```

```cpp
// quantity/Volume.h
#ifndef HG764MD_NKGJKDSJLD_RY64930_NVHF977E
#define HG764MD_NKGJKDSJLD_RY64930_NVHF977E

#include "quantity/Quantity.h"

struct VolumeUnit;
struct Volume : Quantity<VolumeUnit> {};

#endif
```

```cpp 
// quantity/Volume.cpp
#include "quantity/Quantity.tcc"
#include "quantity/VolumeUnit.h"

template struct Quantity<VolumeUnit>;
```

`Length.h`仅仅对`Quantity.h`产生依赖; 特殊地，`Length.cpp`没有产生对`Length.h`的依赖，相反对`Quantity.tcc`产生了依赖。

另外，`Length.h`对`LengthUnit`的依赖关系也简化为声明依赖，而对其真正的编译时依赖，也控制在模板实例化的时刻，即在`Length.cpp`内部。

`LenghtUnit, VolumeUnit`的变化，及其`Quantity.tcc`实现细节的变化，被完全地控制在`Length.cpp, Volume.cpp`内部。

### 子类化优于`typedef/using`

如果使用`typedef`，如果存在对`Length`的依赖，即使是名字的声明依赖，除了包含头文件之外，别无选择。

另外，如果`Quantity`存在`virtual`函数时，`Length`还有进一步扩展`Quantity`的可能性，从而使设计提供了更大的灵活性。

**反例：**
```cpp 
// quantity/Length.h
#ifndef TYIW7364_JG6389457_BVGD7562_VNW12_JFH
#define TYIW7364_JG6389457_BVGD7562_VNW12_JFH

#include "quantity/Quantity.h"

struct LengthUnit;
typedef Quantity<LengthUnit> Length;

#endif
```

**正例：**
```cpp
// quantity/Length.h
#ifndef TYIW7364_JG6389457_BVGD7562_VNW12_JFH
#define TYIW7364_JG6389457_BVGD7562_VNW12_JFH

#include "quantity/Quantity.h"

struct LengthUnit;
struct Length : Quantity<LengthUnit> {};

#endif
```

## 实用宏

为了提高代码的表现力，规范中使用了一部分实用的宏定义。

```cpp
// base/Default.h
#ifndef GKOQWPRT_1038935_NCVBNMZHJS_8909603
#define GKOQWPRT_1038935_NCVBNMZHJS_8909603

namespace details
{
   template <typename T>
   struct DefaultValue
   {
      static T value()
      {
         return T();
      }
   };

   template <typename T>
   struct DefaultValue<T*>
   {
       static T* value()
       {
           return 0;
       }
   };

   template <typename T>
   struct DefaultValue<const T*>
   {
       static T* value()
       {
           return 0;
       }
   };

   template <>
   struct DefaultValue<void>
   {
      static void value()
      {
      }
   };
}

#define DEFAULT(type, method)  \
    virtual type method { return ::details::DefaultValue<type>::value(); }

#endif
```

`DEFAULT`对于定义空实现的`virtual`函数非常方便。需要注意的是，所有计算都是发生在编译时的。

```cpp
// base/Keywords.h
#ifndef H16274882_9153_4DB2_A2E2_F23D4CCB9381
#define H16274882_9153_4DB2_A2E2_F23D4CCB9381

#include "base/Config.h"
#include "base/Default.h"

#define ABSTRACT(...) virtual __VA_ARGS__ = 0

#if __SUPPORT_VIRTUAL_OVERRIDE
#   define OVERRIDE(...) virtual __VA_ARGS__ override
#else
#   define OVERRIDE(...) virtual __VA_ARGS__
#endif

#define EXTENDS(...) , ##__VA_ARGS__
#define IMPLEMENTS(...) EXTENDS(__VA_ARGS__)

#endif
```

`Config.h`提供了编译器支持`C++11`特性的配置信息。`ABSTRACT, OVERRIDE, EXTENDS, IMPLEMENTS`等关键字，使得`Java`程序员也能看懂`C++`的代码，也极大地改善了`C++`的表现力。

```cpp
// base/Role.h
#ifndef HF95EF112_D6C6_4DB0_8C1A_BE5A6CF8E3F1
#define HF95EF112_D6C6_4DB0_8C1A_BE5A6CF8E3F1
#include <base/Keywords.h>

namespace details
{
   template <typename T>
   struct Role
   {
      virtual ~Role() {}
   };
}

#define DEFINE_ROLE(type) struct type : ::details::Role<type>

#endif
```

通过`DEFINE_ROLE`的宏定义来实现对接口的定义，从而可以消除子类对虚拟析构函数的重复实现。


