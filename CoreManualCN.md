ppUTest是一个基于C/C++的单元测试框架，它可以被用来进行单元测试或者测试驱动你的代码。该框架使用C++编写，但可以同时运用在C和C++项目当中，尤其在嵌入式系统当中运用得更为广泛。

CppUTest核心设计原则：

- 短小精悍，易于使用
- 兼容新旧平台
- 符合“测试驱动”开发方法模式

### 主要内容
- [开始]()
- [测试宏]()
- [断言]()
- [创建与注销]()
- [命令行分支]()
- [内存泄露检测]()
- [测试插件]()
- [脚本]()
- [高级]()
- [C接口]()
- [使用Google Mock]()
- [在CppUTest当中运行Google Tests]()

### 开始
##### 第一个测试

编写第一个测试，您需要做的仅仅是一个新的cpp文件，其中包含了一个TEST_GROUP和TEST。

```
TEST_GROUP(FirstTestGroup)
{
};

TEST(FirstTestGroup, FirstTest)
{
   FAIL("Fail me!");
}
```
如上的测试会失败。对于添加一个新的test_groups来说，这是你需要做的所有工作。如果你需要添加另外一个测试，你只需要按照如下仿照另一个测试用例即可：

```
TEST(FirstTestGroup, SecondTest)
{
    STRCMP_EQUAL("hello", "world");
}
```
总所周知，在采用test-driving编写代码时经常需要进行测试用例的添加和删除。CppUTest设计目标之一让添加和删除测试用例尽可能简单。

##### 写作自己的main函数

显然地，要让整个测试框架运转起来，必须要创建一个main函数作为入口。在CppUTest当中的大部分main函数非常类似，它们一般居于AllTests.cpp文件中并且看起来像这样：

```
int main(int ac, char** av)
{
    return CommandLineTestRunner::RunAllTests(ac, av);
}
```

CppUTest会自动查找你的测试用例。

##### 修改Makefile

你需要一个新的或者修改已有的Makefile来让如上整个测试框架运转起来。

###### CppUTest路径
如果你是通过系统上的安装管理器，比如通过apt-get来安装的，那么或许你就不需要修改路径。否则，你需要在Makefile当中添加CppUTest路径。通常，你可以将CppUTest路径作为系统变量，或者将其包含进Makefile当中，比如：

```
CPPUTEST_HOME = /Users/vodde/workspace/cpputest
```

###### 编译选项
为编译器添加包含路径，并且最好同时添加CppUTest pre-inclue header，因为这可以让内存泄露探测器开启调试信息跟踪并且在C当中提供内存泄露检测。按照如下所示添加包含路径：

```
CPPFLAGS += -I(CPPUTEST_HOME)/include
```

（.p和.cpp文件均兼容CPPFLAGS！）
之后，添加如下支持内存泄露检测：

```
CXXFLAGS += -include $(CPPUTEST_HOME)/include/CppUTest/MemoryLeakDetectorNewMacros.h
    CFLAGS += -include $(CPPUTEST_HOME)/include/CppUTest/MemoryLeakDetectorMallocMacros.h
```

这些flags必须同时添加到测试代码和产品代码当中，它们会使用一个东东来替代malloc和new。

###### 链接选项
在链接flags当中添加CppUTest库，例如：

```
LD_LIBRARIES = -L$(CPPUTEST_HOME)/lib -lCppUTest -lCppUTestExt
```

(上面最后一个flags仅仅在你需要使用诸如mocking等扩展功能的时候才需要添加)

### 常用测试宏

- TEST(gourp, name) - 定义一个测试用例
- IGNORE_TEST(group, name) - 关闭一个测试用例
- TEST_GROUP(group) - 定义一个包含多个测试用例的测试集。
- TEST_GROUP_BASE(group, base) - 作用与TEST_GROUP相同，使用一个与Utest不同的base class
- IMPORT_TEST_GROUP(group) - export一个测试集的名称，以便它可以被其他库链接到

##### 测试创建与注销事项

- 每一个TEST_GROUP应当包含setup和teardown函数
- setup会在测试用例主体之前被调用，而teardown函数会在测试用例之后被调用

### 断言

下面这些宏用来检测一些条件，当它们不满足时会立即退出当前的测试：
- CHECK(boolean condition) - 检测任何boolean结果。
- CHECK_TEXT(boolean condition, text) - 检测任何boolean结果，失败时打印text。
- CHECK_EQUAL(expected, actual) - 使用`==`来检测expected, actual两个实体是否相等。如果你要检测的对象是类的实例，那么该类必须重载了`==`运算符，同时还需要添加一个StringFrom()函数来共检测失败时候打印。
- CHECK_THROWS(expected_exception, expression) - 检测expression表达式是否抛出了期望的异常，该宏仅有当CppUTest使用标准C++库进行编译（默认）时才能时候。
- STRCMP_EQUAL(expected, actual) - 使用strcmp()函数来检测expected与actual是否相等。
- LONGS_EQUAL(expected, actual) - 比较两个数字。
- BYTES_EQUAL(expected, actual) - 比较两个8bit数字。
- POINTERS_EQUAL(expected, actual) - 比较两个指针。
- DOUBLES_EQUAL(expected, actual,tolerance) - 比较两个浮点数，tolerance参数为偏差。
- FAIL(text) - 总是失败，并打印text。

<i>CHECK_EQUAL警告</i>
由于CHECK_EQUAL(expected, actual)会对expected和actual进行多次求值，所以在使用该宏的时候有可能出现一些误导性的错误，尤其在使用mock的时候。比如这种情况：当mock的函数被期望仅调用一次，但实际上使用该宏的时候会对actual表达式进行多次求值，那么便会失败。不过这种问题在使用LONGS_EQUAL()的时候不会出现，应该尽量使用LONGS_EQUAL()。

```
CHECK_EQUAL(10, mock_returning_11())
```
此时会报告：Mock Failure: Unexpected additional call。

```
LONGS_EQUAL(10, mock_returning_11()) // 报告actual与expected不一致。
```

<font style="color:red">Q: 为什么会这样？源代码里面如何实现的呢？</font>
上面的问题可以使用C++高级语言特性来避免，比如说使用模板，但这违背了CppUTest前向兼容的设计目的。

### Setup与Teardown
每一个测试组都可以实现一个setup和teardown方法，setup会在每一个测试用例执行之前被调用，而teardown方法在测试用例执行之后调用。你可以向下面这样定义setup和teardown:

```
TEST_GROUP(FooTestGroup)
{
   void setup()
   {
      // Init stuff
   }

   void teardown()
   {
      // Uninit stuff
   }
};

TEST(FooTestGroup, Foo)
{
   // Test FOO
}

TEST(FooTestGroup, MoreFoo)
{
   // Test more FOO
}

TEST_GROUP(BarTestGroup)
{
   void setup()
   {
      // Init Bar
   }
};

TEST(BarTestGroup, Bar)
{
   // Test Bar
}
```

通常情况下，上面的测试代码会遵照如下顺序执行：
- setup BarTestGroup
- 执行测试用例Bar
- teardown BarTestGroup (原文这里缺少了，提交patch ?)
- setup FooTestGroup
- 执行测试用例MoreFoo
- teardown FooTestGroup
- setup FooTestGroup
- 执行测试用例Foo
- teardown FooTestGroup


### 命令行开关
- -v verbose,在测试用例执行的时候打印它的名字
- -c colorize output，执行成功是打印结果以绿色显示，失败以红色显示
- -r# 重复执行#次，不指定次数默认执行2次。这项功能在调式内存泄露的情况下有用，如果第二次执行没有泄露，也就表明在第一次有人分配了static内存空间并且没有释放它们。
- -g group，仅仅运行测试集名称当中包含有指定字串的那部分测试集
- -sg 仅仅运行指定名称的测试集
- -n 运行测试用例名称当中包含有指定字串的测试用例
- -sn 仅仅运行指定名称的测试用例
- "TEST(group, name)"运行测试集group当中的name测试用例
- -ojunit 输出为JUnit ant插件风格的xml文件（一般用于CI系统之中）
- -k 包名称，子JUnit输出当中添加一个包名称（一般用于CI系统当中的分类）

你可以同时指定多个 `-s|sg`，`-s|sn`以及"TEST(group, name)"参数：

当通过`-s|sg`参数指定测试集，实际上会运行这些测试集中的所有用例，因为没有特定的测试用例名称与之匹配。

当通过`-s|sg`参数指定测试用例时，实际上会运行这些当前所有测试集中名字匹配的所有测试用例，因为并没有特定的特定的测试集合。

当通过`-s|sg`，`-s|sn`（或者使用“TEST(group, name)”共同给出参数时仅仅会运行相匹配的那些测试集或者测试用例。

### 内存泄露检测

CppUTest支持测试用例层面的内存泄露检测，也即是说该框架会检测一个用例在运行前后的内存状态是否相同。

大致上来说，其中的检测过程类似：1. 用例setup前处理->记录使用的内存总数 2. setup 3. 执行测试用例 4. teardown 5. 用例teardown后处理->检查当前的内存总数是否与之前的相等。

内存泄露检测器包括了三部分：* 检测器基础部件（包括new操作符的链接符号）*重载了new操作符的宏（包含额外的文件和行号信息）*重载了的malloc/free的宏，用来兼容C。

为了支持如上提到的宏，你需要在Makefile当中添加如下编译选项（这些选项在你使用CppUTest Makefile helpers时会自动添加）：

```
CXXFLAGS += -include $(CPPUTEST_HOME)/include/CppUTest/MemoryLeakDetectorNewMacros.h
CFLAGS += -include $(CPPUTEST_HOME)/include/CppUTest/MemoryLeakDetectorMallocMacros.h
```

##### 关闭与打开内存泄露检测

有多种方式去关闭内存泄露检测。然而，将内存泄露检测功能打开并且依据它提供的信息去修复对应的内存泄露问题，保证代码质量的可靠是备受推崇的做法。

你可以添加如下代码将内存泄露检测完全关闭：

```
int main(int argc, char** argv)
{
    MemoryLeakWarningPlugin::turnOffNewDeleteOverloads();
    return CommandLineTestRunner::RunAllTests(argc, argv);
}
```

你也可以仅仅像如下这样关闭某一个测试用例的内存检测：

```
void setup()
{
    MemoryLeakWarningPlugin::turnOffNewDeleteOverloads();
}

void teardown()
{
    MemoryLeakWarningPlugin::turnOnNewDeleteOverloads();
}
```

如果想彻底使能内存泄露检测，你也可以在构建CppUTest的时候指定"configure-disable-memory-leak-detection"或者在编译的时候将参数“-DCPPUTEST_MEM_LEAK_DETECTION_DISABLED”传递给编译器。

##### 与STL中的new操作符宏的冲突
