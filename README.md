# C/C++ ABI 以及如何对动态连接库加载一个C++成员函数符号

## 目录

- [ABI简介](#abi_brief)
- [C++与C语言不同的函数签名](#c_cpp_function_signature)
- [运行时加载动态连接库中的指定C++类的成员函数](#runtime_load_cpp_member_func)

<br />

<a id="abi_brief" name="abi_brief"></a>
## ABI简介

我们熟悉编程的朋友应该知道。一个在 Windows 平台上开发的应用程序如果兼容性做得好，那么在其他装有 Windows 系统的电脑上也能运行。而在 Windows 平台上使用 Windows SDK 开发的一个可执行程序（exe文件），那么在 Linux系统 上是无法运行的。这其中的道理也是显而易见，首先是 Windows 系统与 Linux 系统对可执行文件所支持的文件格式不同。Windows支持的是 **PE** 格式（[PE Format](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)），而 Linux 系统使用的则是 **ELF** 格式（[Executable and Linkable Format](http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf)）。

除此之外，这两个系统默认所使用的 [ABI](https://www.intel.com/content/www/us/en/docs/programmable/683620/current/application-binary-interface.html)（[应用程序二进制接口](https://baike.baidu.com/item/ABI/10912305?fr=aladdin)）也各不相同。Windows系统默认使用的是 **MSVC ABI**（对于x86_64架构可参考：[Overview of x64 ABI conventions](https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions)）。而其他类Unix系统（包括Linux、FreeBSD以及macOS等）则使用 **System V ABI**（对于x86_64架构可参考：[System V Application Binary Interface -- AMD64 Architecture Processor Supplement](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)）。

一般而言，ABI描述了当前处理器架构运行在当前系统上关于数据类型、函数调用约定、堆栈使用等的协定。在一个操作系统上，对于某一特定处理器架构而言，只有拥有一套稳定成熟的ABI约定才能使得各个应用程序之间相互兼容，并且也能方便地开发一些中间件（比如很多应用程序之间可共享的动态连接库）。

另外我们还能发现，上述列出的无论是基于 MSVC 的 ABI 还是基于System V 的 ABI，其实都是基于 **C语言接口** 进行描述的，也就是 C-ABI。正因如此，C-ABI 也是各个编译器、各种不同编程语言，能进行相互调用的基础。比如我们在 Linux 系统上，无论是用 GCC 还是 Clang 编译器，生成的静态库以及动态库都能被相互连接。此外，我们无论是在 Windows 系统还是在 Linux 系统上，可以既可以使用 Pascal、Delphi、Swift、Rust 等编译型编程语言直接使用C语言构建出来的动态连接库，而且大部分需要虚拟机或解释器执行的编程语言（比如 Java、Python、Lua、Node.js等）也均提供了它们自己的接口（比如 Java 提供的是 JNI）来访问C语言构建出来的动态连接库。正因如此，C++ 编程语言直接提供了生成遵循 C-ABI 约定的语法——即 **`extern "C"`**。

阅读以下内容前需要了解《[C语言对目标文件、静态库与动态库的连接及外部符号的可见性](https://github.com/zenny-chen/C_lang_objects_static-libs_dynamic-libs_external-symbols_visibility)》这篇博文的一些知识。各位可以先过目一下~

<br />

<a id="c_cpp_function_signature" name="c_cpp_function_signature"></a>
## C++与C语言不同的函数签名

一般而言，C语言的函数签名十分简单，基本就是其原本的函数名，当然有些系统会在函数名前加一个前导下划线 **`_`** 构成其函数签名。因此，对于一个命名为 **Foo** 的C函数而言，它在目标文件中的符号名就是 **`Foo`** 或是 **`_Foo`**。

不过C++的函数签名就复杂多了。这其中的原因是由于C++编程语言具有函数重载特性，因此其函数签名需要携带参数个数、参数类型以及返回类型的编码。下面举一个简单的例子。

```cpp
// hello.cpp
extern "C" void CPP_CFunc(int a)
{

}

void CPPFunc(int a)
{

}
```

这里笔者新建了一个 **hello.cpp** 源文件，然后里面就定义了两个函数。一个是遵循 C ABI 的 **`CPP_CFunc`**，而另一个是纯 C++ API 的 **`CPPFunc`**。我们先在Linux系统下将该源文件生成目标文件（.o文件）：

```bash
g++ hello.cpp -o hello.cpp.o -c -std=gnu++17
```
随后，我们用 `readelf -s hello.cpp.o` 来查看所生成的 **hello.cpp.o** 的符号。我们可以看到以下导出的符号信息：
>    133: 0000000000000000    14 FUNC    GLOBAL DEFAULT   55 CPP_CFunc  \
   134: 000000000000000e    14 FUNC    GLOBAL DEFAULT   55 _Z7CPPFunci

我们这里可以很明显地看到，基于C ABI的 **`CPP_CFunc`** 这个符号非常干净，与函数名完全一致。而基于C++ ABI的符号则复杂多了，其符号为 **`_Z7CPPFunci`**，而它原先的函数原型为 `void CPPFunc(int a)`。

然后我们在Windows下使用MSVC编译器再试试。我们在Windows下使用MSVC尝试的时候需要使用 **Debug** 模式，要是使用 **Release** 模式，那些没有被引用的符号可能会被优化掉。我们生成好 **hello.obj** 之后，在开始菜单下找到Visual Studio的目录，然后打开 **x64 Native Tools Command Prompt**，进入到当前项目的存放源文件 **hello.cpp** 目录中的 **x64/Debug/** 目录中，使用：`dumpbin /ALL hello.obj`。于是，我们可以搜索到以下符号信息：

> 113 00000000 SECT8  notype ()    External     | CPP_CFunc  \
114 00000000 SECT4  notype ()    External     | ?CPPFunc@@YAXH@Z (void __cdecl CPPFunc(int))

我们可以看到，Windows系统下采用MSVC编译后的 **`CPP_CFunc`** 函数所生成的符号与Linux系统下使用GCC编译出来的一样，与函数名完全一致。而遵循C++ ABI的 **`CPPFunc`** 所导出的符号则与Linux下的大相径庭了，其符号为：**`?CPPFunc@@YAXH@Z`**。

此外，被 **`extern "C"`** 修饰的函数就不能允许被重载（**overload**）了，否则会引发编译时错误。有了这些认识之后，接下来我们将介绍如何在特定操作系统下以及特定处理器架构下，运行时加载动态连接库中的指定C++类中的成员函数。

<br />

## <a id="runtime_load_cpp_member_func"></a> 运行时加载动态连接库中的指定C++类的成员函数

下面我们来看一段看似比较简单的C++代码，该代码在名为 **hello.cpp** 的源文件中。
```cpp
// hello.cpp
#include <cstdio>

#ifdef _WIN32
#define CUSTOM_DLL_EXPORT   __declspec(dllexport)
#else
#define CUSTOM_DLL_EXPORT   __attribute__((visibility("default")))
#endif

struct MyTest
{
    static void CUSTOM_DLL_EXPORT ClassMethod(int a);

    void CUSTOM_DLL_EXPORT MemberMethod(int a);

    void InternalMethod()
    {
        puts("This is the InternalMethod!");
    }

    int m_value = 0;
};

void MyTest::ClassMethod(int a)
{
    printf("ClassMethod value: %d\n", a / 2 - 1);
}

void MyTest::MemberMethod(int a)
{
    printf("MemberMethod value: %d\n", a * 2 + m_value);

    InternalMethod();
}
```
上述代码中我们可以看到，这里定义了一个名为 **MyTest** 的结构体。其中定义了用于外部符号导出的 **`ClassMethod`** 类方法，一个用于外部符号导出的 **`MemberMethod`** 成员方法，以及一个内部使用的 **`InternalMethod`** 成员方法。

这里想必大家应该也已经注意到了，需要被符号导出的 **`ClassMethod`** 类方法以及 **`MemberMethod`** 成员方法被定义到了 **MyTest** 的外部。这是由于倘若定义在结构体类型内部，该类函数或成员函数将被视作为 **内联（`inline`）** 的。而C语言与C++语言标准中均指明了，一个内联函数是不具有连接（**linkage**）的，从而内联函数的符号不会被导出。

接着，我们先在Linux系统下查看这两个导出符号，然后再进行动态连接库的运行时加载。我们先对 **hello.cpp** 进行编译，然后生成一个动态连接库文件 **libcppfunc.so**。
```bash
#! /bin/sh
# build.sh
g++ hello.cpp  -o hello.cpp.o  -c -std=gnu++17 -fvisibility=hidden
g++ hello.cpp.o  -o libcppfunc.so  -shared -s 
```
执行完上述的 `sh build.sh` 命令之后，我们可以执行 `readelf -s -W libcppfunc.so` 来查看刚生成的动态连接库中的符号。我们可以发现以下符号输出：
> 7: 000000000000116e    67 FUNC    GLOBAL DEFAULT   14 _ZN6MyTest12MemberMethodEi
     8: 000000000000113a    52 FUNC    GLOBAL DEFAULT   14 _ZN6MyTest11ClassMethodEi

从符号名上我们能获悉，**`_ZN6MyTest11ClassMethodEi`** 这个符号对应于 **`MyTest::ClassMethod`**，而 **`_ZN6MyTest12MemberMethodEi`** 这个符号对应于 **`MyTest::MemberMethod`**。后面，我们将会利用这些符号做运行时的动态加载。

下面我们看以下新建的C++源文件——**main.cpp**：
```cpp
#include <cstdio>
#include <dlfcn.h>

struct MyTest
{
    int m_value = 0;

    static inline auto (*pClassMethod)(int) -> void = nullptr;

    auto (*pMemberMethod)(MyTest *thisObj, int a) -> void = nullptr;
};

auto main() -> int
{
    void *soHandle = dlopen("libcppfunc.so", RTLD_LAZY);
    if(soHandle == nullptr)
    {
        puts("libcppfunc.so not found!");
        return -1;
    }

    auto pSymbol = dlsym(soHandle, "_ZN6MyTest11ClassMethodEi");
    if(pSymbol == nullptr)
    {
        puts("Symbol MyTest::ClassMethod not Found!");
        dlclose(soHandle);
        return -1;
    }
    MyTest::pClassMethod = reinterpret_cast<decltype(MyTest::pClassMethod)>(pSymbol);

    pSymbol = dlsym(soHandle, "_ZN6MyTest12MemberMethodEi");
    if(pSymbol == nullptr)
    {
        puts("Symbol MyTest::MemberMethod not Found!");
        dlclose(soHandle);
        return -1;
    }

    MyTest test { .m_value = 5 };
    test.pMemberMethod = reinterpret_cast<decltype(test.pMemberMethod)>(pSymbol);

    MyTest::pClassMethod(10);

    test.pMemberMethod(&test, 100);

    dlclose(soHandle);
}
```
一开始，我们同样定义了一个名为 **MyTest** 的结构体，但与之前在 **hello.cpp** 源文件中定义的是有比较大差异的。这是因为我们后面要做的是运行时动态加载原本处于MyTest结构体中的类方法及成员方法，因此这里需要借助函数指针。为了不破坏原本的 **MyTest** 的对象模型，我们需要先确保原来 **MyTest** 中的各个数据成员在内存模型上与当前 **main.cpp** 中的保持一致，然后再往下扩充，将这些函数指针定义到原对象模型的后面，从而看上去呈现一种“继承”的关系。

所以这里先将 `int m_value` 成员对象放在最前面。然后，下面分别定义了一个类成员对象指针 **pClassMethod** 以及一个类实例成员指针 **pMemberMethod**。它们后续将会分别指向从动态连接库中加载出来的 **`MyTest::ClassMethod`** 对应符号以及 **`MyTest::MemberMethod`** 对应符号。

我们这里会感到奇怪的是：为何 **pMemberMethod** 的参数类型为 **`(MyTest *thisObj, int a)`**。由于原本的 **`MyTest::MemberMethod`** 是一个成员实例方法，倘若要用一个指向该成员实例方法的指针指向它，那么其具体类型为 **`auto (MyTest::*)(int) -> void`**。而由于C++在类型系统上对于类型转换的要求非常严格，它不允许一个 **`void*`** 指针转换为一个指向成员实例方法的指针！因此我们这里需要采用成员实例方法相同的函数调用约定去“拟合”它。

对于通常实现而言，一个类的成员实例方法相当于在一开始额外插入一个指向该类类型的指针作为参数。比如这里的 **`void MyTest::MemberMethod(int a)`** 就可看作为：**`void (MyTest *thisObj, int a)`** 函数类型。出于这点考虑，这里将 **pMemberMethod** 指针类型定义为了 **`auto (*)(MyTest *thisObj, int a) -> void`**。

最后我们通过以下命令去编译此 **main.cpp** 并直接生成最终的可执行程序：
```bash
g++ main.cpp  -o test  -std=gnu++17  -L./ -ldl
```
最后，我们在创建一个执行脚本，对 **test** 方便执行。
```bash
#! /bin/sh
# run.sh
export LD_LIBRARY_PATH=./:${LD_LIBRARY_PATH}
./test
```
运行 `sh run.sh` 之后，我们将会得到以下输出：
> ClassMethod value: 4
MemberMethod value: 205
This is the InternalMethod!

<br />

在Windows系统下也类似。我们先通过上述的 **hello.cpp** 生成一个DLL文件，比如名为 **cppfunc.dll**。然后我们通过 `dumpbin /EXPORTS cppfunc.dll` 来观察导出的符号信息，可以看到以下输出：
> 1    0 00001070 ?ClassMethod@MyTest@@SAXH@Z = ?ClassMethod@MyTest@@SAXH@Z (public: static void __cdecl MyTest::ClassMethod(int))
          2    1 00001090 ?MemberMethod@MyTest@@QEAAXH@Z = ?MemberMethod@MyTest@@QEAAXH@Z (public: void __cdecl MyTest::MemberMethod(int))

随后，我们修改一下 **main.cpp**，如下所示。
```cpp
#include <cstdio>
#include <Windows.h>

struct MyTest
{
    int m_value = 0;

    static inline auto (*pClassMethod)(int) -> void = nullptr;

    static inline auto (*pMemberMethod)(MyTest* thisObj, int a) -> void = nullptr;
};

auto main() -> int
{
    HMODULE myTestModule = LoadLibraryA("cppfunc.dll");
    if (myTestModule == nullptr)
    {
        puts("cppfunc.dll not found!");
        return -1;
    }

    FARPROC pSymbol = GetProcAddress(myTestModule, "?ClassMethod@MyTest@@SAXH@Z");
    if (pSymbol == nullptr)
    {
        puts("Symbol MyTest::ClassMethod cannot be located!");
        FreeLibrary(myTestModule);
        return -1;
    }
    MyTest::pClassMethod = reinterpret_cast<decltype(MyTest::pClassMethod)>(pSymbol);

    pSymbol = GetProcAddress(myTestModule, "?MemberMethod@MyTest@@QEAAXH@Z");
    if (pSymbol == nullptr)
    {
        puts("Symbol MyTest::MemberMethod cannot be located!");
        FreeLibrary(myTestModule);
        return -1;
    }
    MyTest::pMemberMethod = reinterpret_cast<decltype(MyTest::pMemberMethod)>(pSymbol);

    MyTest mytest { .m_value = 5 };
    MyTest::pClassMethod(10);
    MyTest::pMemberMethod(&mytest, 100);

    FreeLibrary(myTestModule);

    return 0;
}
```
这里，我们索性将 **pMemberMethod** 也作为类成员对象，这样我们不需要每当创建一个 **MyTest** 对象实例的时候就必须为其 **pMemberMethod** 实例成员做初始化；而且这也不会污染 **MyTest** 原本的对象模型，不会使得其存储空间被无辜地扩充，并且也非常具有便利性。

我们编译运行之后，能得到Linux上相同的输出结果。

进阶参考：[C++如何为标准库添加额外的库或函数](https://www.toutiao.com/article/7211582868970062370/)



