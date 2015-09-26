注：原文当中包含了两部分内容，其中一部分是“在IAR嵌入式工作台上部署单元测试”，另外一部分是[CppUTest google组里面的文章](https://groups.google.com/forum/#!topic/cpputest/WxCfnVZYGHw)。当前只翻译第一部分。

### 在IAR嵌入式工作台上部署单元测试
我刚刚完成了将CppUTest作为我们公司标准单元测试框架的可行性评估，在这个过程当中有时会觉得极为困难，但最终我还是胜利了。因此，我很愿意分享我从中获得的经验并且期望可以：

- 从中发现是否我可以做得更好
- 给其他有志于应用该框架的人提供仅有的一点帮助
- 回馈CppUTest

摆在眼前的一个严格要求便是CppUTest需要与IAR Embedded Workbench 6.4项目工程兼容，我们不想再去维护另外一套专门用来执行测试的构建环境。当时的经验是：直接通过git将代码拉取下来是最好的下载方法，因为这能够很方便的做一些本地化的修改并且将这些修改存储在我们自己的代码仓库中，同时还能够方便的与CppUTest源代码库进行同步。另外一个好处是，我们的用户不必操作git，他们所关注的仅仅是从我们的SVN代码仓库当中下载软件。

为了在IAR上编译CppUTest库，我在cpputest的根目录创建了一个新的IAR工程，并且做了如下设置：

```
Project -> Options -> General Options -> Target -> Core = Cortex-M3.
Project -> Options -> General Options -> Output -> Output file = Library
Project -> Options -> General Options -> Output -> Executables/libraries = Debug (removed exe subdirectory)
Project -> Options -> C/C++ Compiler -> Language 1 -> Language = C++
Project -> Options -> C/C++ Compiler -> Language 1 -> Language conformance =  Standard
Project -> Options -> C/C++ Compiler -> Language 1 -> C++ Dialect = C++ (leave exceptions checked)
Project -> Options -> C/C++ Compiler -> Preprocessor -> Additional include directories = $PROJ_DIR$\include
Project -> Options -> C/C++ Compiler -> Diagnostics -> Suppress these diagnostics = Pa050 (turn off warning about non-standard line endings)
Added all .cpp files in src\CppUTest\
Added src\Platforms\Iar\UtestPlatform.cpp
```

将UtestPlatform.cpp中的第89行“return 1;”修改为“return t”，这样就打开了时间功能。设置完成之后，CppUTest便可以成功编译出Debug和Release版本，最终生成一个CppUTest.a文件。

为了构建CppUTest测试用例，为在cpputest根目录创建了另外一个名为CppUTestTest的IAR工程，并做了如下设置：

```
Project -> Options -> General Options -> Target -> Core = Cortex-M3.
Project -> Options -> General Options -> Library Configuration -> Library low-level interface implementation = Semihosted
Project -> Options -> C/C++ Compiler -> Language 1 -> Language = Auto
Project -> Options -> C/C++ Compiler -> Language 1 -> Language conformance =  Standard
Project -> Options -> C/C++ Compiler -> Language 1 -> C++ Dialect = C++ (leave exceptions checked)
Project -> Options -> C/C++ Compiler -> Preprocessor -> Additional include directories = $PROJ_DIR$\include
Project -> Options -> C/C++ Compiler -> Diagnostics -> Suppress these diagnostics = Pa050 (turn off warning about non-standard line endings)
Project -> Options -> Linker -> Config -> Override default -> Edit -> Stack/Heap Sizes -> CSTACK = 0x600
Project -> Options -> Linker -> Config -> Override default -> Edit -> Stack/Heap Sizes -> HEAP =  0x8000
Added all .cpp and .c files in tests\
Added Debug\CppUTest.a
Changed line 16 of tests\AllocLetTestFree.c to explicit cast to (AllocLetTestFree) to satisfy compiler
Changed line 22 of tests\AllocLetTestFree.c to type AllocLetTestFree instead of void* to satisfy compiler
Changed tests\AllTests.cpp to declare a const char*[] with "-v" as the second element, so it can be passed to RunAllTests to turn on verbose mode in IAR
Built and ran in simulator.
Turned on Debug->C++ Exceptions->Break on uncaught exception to intercept mysterious jumps to abort.
With that out of the way I did the same for CppUTestExt and CppUTestExtTester, with no further dramas.
```

其中的stack/heap分配可能是最具有戏剧性的了。首先，它并没有明确的表明内存耗尽的情况是否发生。当heap空间耗尽时，malloc/new会返回0并且抛出一个异常，然后调试器会中断它（并不会有任何提示，这里原文为with no sign of the offensive statement。）如果打开“Break on uncaught exception”会有帮助。在stack耗尽时，程序的行为将会难以预测，这个时候程序通常正在打印一些比如“stack会崩溃”的错误信息过程当中。我发现0x600和0x8000仅仅够把整个测试跑起来，由于IAR中的Cortex-M3体系架构中在芯片上有64KB的RAM空间，所以实际上它们非常紧张。从map文件当中可以查看到对于RAM区域的内存空间分配：

```
Static: 21056 bytes
Heap: 32768 bytes
iar.dynexit (atexit statics): 8760
Stack:  1536 bytes
Total:  64120 bytes out of 65535 bytes.
```

将“Destory static objects”选项关掉或许可以节省差不多8760字节的空间，但空间仍旧紧张。

前面已经提到过，我还需要在CppUTest源代码上的两个地方做一些修改来使得整个测试正常构建和执行。就我当前所知，这些修改也可以应用到master上：

```
src\Platforms\Iar\UtestPlatform.cpp:89 (return t;)
tests\AllocLetTestFree.c:16 and  tests\AllocLetTestFree.c:22 (explicit types)
```

针对AllTests.cpp的第三个修改或许仅仅对与IAR用户来说是重要的，因为在IAR当中不能为目标执行文件设置命令行参数。

在相应的库文件和测试文件能够正常的构建和运行之后，下一步便是为真实的工程创建测试。一个合理的流程是：在目标工程的源代码层级中创建一个test文件夹，之后再创建AllTests.cpp文件并添加如下内容：

```
#include "CppUTest/CommandLineTestRunner.h"
int main(int ac, char** av)
{
  const char * av_override[] = { "exe", "-v" }; //turn on verbose mode

  //return CommandLineTestRunner::RunAllTests(ac, av);
  return CommandLineTestRunner::RunAllTests(2, av_override);
}

And create MyCodeTest.cpp with this content:
extern "C"
{
#include "..\MyCode.h"
}

#include "CppUTest/TestHarness.h"

TEST_GROUP(FirstTestGroup)
{
  void setup() {}
  void teardown() {}
};

TEST(FirstTestGroup, FirstTest)
{
  FAIL("Fail me!");
}

TEST(FirstTestGroup, SecondTest)
{
  STRCMP_EQUAL("hello", "world");
}
```

之后，在你目标工程所在的目录下面，创建一个名为MyProjectTest的C++ main 工程，并且按照如下设置好Project -> Options：

```
General Options -> change Device to your target device
General Options -> Library Configuration -> check "Use CMSIS" if it used in your target project
C/C++ Compiler -> Language 1 -> Language = Auto
C/C++ Compiler -> Language 1 -> C++ Dialect = C++
C/C++ Compiler -> Preprocessor -> Additional include directories = path\to\cpputest\include
C/C++ Compiler -> Preprocessor -> add any necessary #defines from the target project to Defined symbols
C/C++ Compiler -> Diagnostics -> Suppress these diagnostics = Pa050
Linker -> Config -> Override default with icf file used by target project
```

确保CSTACK空间至少有0x600，HEAP空间至少0X5000。如果你的目标工程已经使用了这些区域你需要做一些适当的扩展，否则只能确保它们可以部署到适合它们的地方。

删除main.cpp, AllTests.cpp, MyCodeTest.cpp和Debug\CppUTest.a（在没有性能/大小限制的时候使用Debug版本，以便可以在代码当中设置断点）。创建一个src组并将目标工程当中的源文件添加进来。之后就可以使用模拟器来调试可执行文件并运行测试了。

打开终端的I/O窗口来观察输出，在non-verbose模式下点号（“.”）代表执行成功的测试用例，而叹号（“！”）代表被忽略的测试用例。

还有其他许多的细节在这里无法一一叙述，比如工作空间，构建配置以及在碰到错误时如何处理。不过希望这些已有的内容可以帮助你迈出第一步。

这些就是我要介绍的全部内容。当时最后一天，CppUTest看起来可以非常好的满足我们的要求，在完成初始化之后，它很好的被集成到了IAR并且IDE上的输出也可读。模拟器可以替代PC上面的执行过程（在PC上面需要维护不同编译器的构建环境）和在硬件上面执行（需要有可以工作的硬件、Flash写入操作以及功能性外围设备，并且非常不易将测试集注入其中）。

