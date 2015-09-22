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

使用mock时你需要做的仅仅是将头文件CppUTestExt/MockSupport.h包含到测试文件当中，该头文件几乎包括了所有CppUTest Mocking需要的内容。而TEST_GROUP的声明与之前保持一致，并不需要任何改变。

TEST（MockDocumentation, SimpleScenario)中包含了如下的期望语句：

```
mock().expectOneCall("productionCode");
```

在调用mock()时会返回被用来记录期望语句的全局MockSupport（后面会较详细的介绍）。在上面这个例子当中，调用了expectOneCall("productionCode")，它记录了一个“仅仅调用一次productionCode函数”的期望。

函数productionCode的调用与被测试代码对应（译注：用来判断被测试代码中是否真的有对应的函数调用）。该测试用例以检测“是否期望被满足”作为结束：

```
mock().checkExpectations();
```

如果在当前的测试工程没有使用MockSupportPlugin，那么你得在每一个测试用例当中均添加上面这一行用来检测期望是否被满足的代码，反之则不需要。同样的，只要你使用了MockSupport插件，对于在teardown中调用的mock().clear()也可以省略。当你即没有使用MockSupportPlugin，也没有添加mock().clear()语句时，内存泄漏检测器就会将mock调用识别为内存泄漏。

一个被mock过的函数看起来像这样：

```
void productionCode()
{
    mock().actualCall("productionCode");
}
```

上面的代码使用MockSupport来记录针对productionCode函数的实际调用（译注：前面提到过，我们是通过调用mock()来使用MockSupport这一全局变量的）。

这种简单的场景经常被使用到，尤其在给项目中用到的第三方库打桩的时候。

如果针对函数productionCode的调用并没有发生，那么该单元测试会失败并且提示如下的错误：

```
ApplicationLib/MockDocumentationTest.cpp:41: error: Failure in TEST(MockDocumentation, SimpleScenario)
    Mock Failure: Expected call did not happen.
    EXPECTED calls that did NOT happen:
        productionCode -> no parameters
    ACTUAL calls that did happen:
        <none>
```

### 使用对象
在mock对象时，它与mock函数并没有什么不同。下面的代码展示了一个使用运行时mock的例子：

```
class ClassFromProductionCodeMock : public ClassFromProductionCode
{
public:
    virtual void importantFunction()
    {
        mock().actualCall("importantFunction");
    }
};

TEST(MockDocumentation, SimpleScenarioObject)
{
    mock().expectOneCall("importantFunction");

    ClassFromProductionCode* object = new ClassFromProductionCodeMock; /* create mock instead of real thing */
    object->importantFunction();
    mock().checkExpectations();

    delete object;
}
```

该代码示例非常清楚明白。真正的对象被手动创建的一个mock对象替代了，并且在替代对象中通过MockSupport记录了真实的调用。

在使用对象的时候，我们可以通过如下的语句来检查函数调用是否发生在特定的对象上面：

```
mock().expectOneCall("importantFunction").onObject(object);
```

同时，在mock调用的地方需要添加如下的改动：

```
mock().actualCall("importantFunction").onObject(this);
```

当一个函数调用并没有发生在它所期望的对象身上时，就会有如下的错误信息出现：

```
MockFailure: Function called on a unexpected object: importantFunction
    Actual object for call has address: <0x1001003e8>
    EXPECTED calls that DID NOT happen related to function: importantFunction
        (object address: 0x1001003e0)::importantFunction -> no parameters
    ACTUAL calls that DID happen related to function: importantFunction
        <none>
```

### 参数
显然，仅仅去检测一个函数是否被调用看起来还是缺乏一定吸引力的，不过像下面这样可以记录一个函数的参数就很有用了：

```
mock().expectOneCall("function").onObject(object).withParameter("p1", 2).withParameter("p2", "hah");
```

对应mock调用的地方应该写成下面这样：

```
mock().actualCall("function").onObject(this).withParameter("p1", p1).withParameter("p2", p2);
```

在执行测试用例的时候，如果其中某个参数检测不通过，会有如下的错误提示：

```
Mock Failure: Expected parameter for function "function" did not happen.
    EXPECTED calls that DID NOT happen related to function: function
        (object address: 0x1)::function -> int p1: <2>, char* p2: <hah>
    ACTUAL calls that DID happen related to function: function
        <none>
    MISSING parameters that didn't happen:
        int p1, char* p2
```

### 将对象做为参数
withParameters有一个不足：它仅仅支持`int`, `double`, `const char *`或者`void*`。然而，除了这些基本类型，很多时候参数会是一些自定义类型的对象。这个时候该如何处理？下面是一个例子：

```
mock().expectOneCall("function").withParameterOfType("myType", "parameterName", object);
```

可见，这个时候会使用到另外一个接口:withParameterOfType。由于mock框架需要知道如何去比较对应的类型，在操作该类型的参数前必须安装好对应的比较器。

```
MyTypeComparator comparator;
mock().installComparator("myType", comparator);
```

其中的MyTypeComparator是一个自定义的比较器，它需要实现MockNamedValueComparator接口，例如：

```
class MyTypeComparator : public MockNamedValueComparator
{
public:
    virtual bool isEqual(void* object1, void* object2)
    {
        return object1 == object2;
    }
    virtual SimpleString valueToString(void* object)
    {
        return StringFrom(object);
    }
};
```

其中的isEqual用来比较两个参数。而valueToString会在用例执行失败的时候用来打印错误信息，也就是搞定如何输出它期望的值与实际的值。如果你更情愿使用C函数，只需要使用接受构造器中函数指针的MockFunctionComparator。
<i style="color:red">怎么理解这句话，源代码中是怎样的呢？是否在框架里面定义了对应的比较器的指针，只需要将比较器作为参数传递进入就好了？</i>

在需要删除比较器的时候，仅仅需要添加如下一行代码即可：

```
mock().removeAllComparators();
```

比较器有时会让人摸不着头脑，这里有一些使用提示：

- <i>提示一：注意比较器变量的作用域！</i>
比较器在传递参数的时候并不会进行值拷贝，而是直接使用从installComparator函数中传递进来的实例。因此，在使用的时候务必确保实例的有效性。比如，如果你在TEST当中调用了installComparator，却在teardown的时候进行checkExpectations，此时比较器已经销毁了，所以这个时候就会使得测试程序崩溃了。

- <i>提示二：在使用MockPlugin的时候也别忽视其作用域</i>
在使用MockPlugin时（推荐），一种可取的方式是通过MockPlugin来安装比较器，或者将比较器置于全局空间下。checkExpectation会在teardown之后调用，如果比较器在之前销毁也将造成崩溃。

### 出参
对于某些参数，它们传递给调用函数时不是直接传递参数对应的值，而是传递引用，如此便可通过修改它所指向的数据来达到“返回值”的效果。

CppUMock支持在expected call时指定这些输出参数：

```
int outputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &outputValue, sizeof(outputValue));
```

同时，在actual call当中需要有下面这样对应的代码：

```
void Foo(int *bar)
{
    mock().actualCall("foo").withOutputParameter("bar", bar);
}
```

一旦在调用了actual call之后之后，传递给Foo函数的参数bar的值将会与expected call当中指定的值一致（在这个例子当中bar的值将为4）。

- <i>提示一：</i>
在使用输出参数的时候，CppUMock对于非法访问内存是束手无策的，它会对在withOutputParameterReturning调用中指定大小的内存执行memcpy操作。因此，如果它拷贝的内存大小超过了actual call中出参指向的数据空间大小，这个时候便极有可能会出现段错误了。

针对withOutputParameterReturning提供了针对char, int, unsigned, long, unsigned long和double类型的支持，这样就可以省略掉size这个参数。（译注：这也在一定程度上避免了不小心指定不合理的空间大小而导致段错误的问题）

```
char charOutputValue = 'a';
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &charOutputValue);

int intOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &intOutputValue);

unsigned unsignedOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &unsignedOutputValue);

long longOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &longOutputValue);

unsigned long unsignedLongOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &unsignedLongOutputValue);

double doubleOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &doubleOutputValue);
```

- <i>提示二：</i>
当有char, int等等数组传递给withOutputParameter的时候，你必须在withOutputParameterReturning的时候指定该数组的确切空间，否则它只会拷贝一个元素。

### 返回值




