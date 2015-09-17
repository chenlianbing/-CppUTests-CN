CppUTest支持与mock一起构建。这篇文档介绍CppUTest对于mock的支持。如下是mock支持的设计目标：

- 与CppUTest相同的设计目的——少量C++特性以便更好的支持嵌入式软件。
- 没有代码生成。
- 没有，或者极少隐藏魔术宏
- 易于使用
- 便于开发

最为主要设计原则是让手动mock更容易，而不是为了自动mock。如果手动mock更容易，使mock自动化是迟早的事，但那本身并不是目的。

### 内容列表
- [简单场景]()
- [使用对象]()
- [使用参数]()
- [将对象作为参数]()
- [输出参数]()
- [返回值]()
- [传递其他数据]()
- [其他的MockSupport]()
- [MockSupport范围]()
- [Mock插件]()
- [C接口]()

### 一个简单场景
一个最简单的场景是检查一个预期的函数是否被调用，如下：

```
#include "CppUTest/TestHarness.h"
#include "CppUTestExt/MockSupport.h"

TEST_GROUP(MockDocumentation)
{
    void teardown()
    {
        mock().clear();
    }
};

void productionCode()
{
    mock().actualCall("productionCode");
}

TEST(MockDocumentation, SimpleScenario)
{
    mock().expectOneCall("productionCode");
    productionCode();
    mock().checkExpectations();
}

```





