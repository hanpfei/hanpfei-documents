---
title: C++ lambda 捕获模式与右值引用
date: 2020-03-21 21:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

lambda 表达式和右值引用是 C++11 的两个非常有用的特性。
<!--more-->
lambda 表达式实际上会由编译器创建一个 `std::function` 对象，以值的方式捕获的变量则会由编译器复制一份，在 `std::function` 对象中创建一个对应的类型相同的 const 成员变量，如下面的这段代码：
```
int main() {
  std::string str = "test";
  printf("String address %p in main, str %s\n", &str, str.c_str());
  auto funca = [str]() {
    printf("String address %p (main lambda), str %s\n", &str, str.c_str());
  };

  std::function<void()> funcb = funca;
  std::function<void()> funcc;
  funcc = funca;

  printf("funca\n");
  funca();

  std::function<void()> funcd = std::move(funca);
  printf("funca\n");
  funca();

  printf("funcb\n");
  funcb();

  std::function<void()> funce;
  funce = std::move(funcb);

  printf("funcb\n");
//  funcb();

  printf("funcc\n");
  funcc();

  printf("funcd\n");
  funcd();

  printf("funce\n");
  funce();

//  std::function<void(int)> funcf = funce;

  return 0;
}
```

这段代码的输出如下：
```
String address 0x7ffd9aaab720 in main, str test
funca
String address 0x7ffd9aaab740 (main lambda), str test
funca
String address 0x7ffd9aaab740 (main lambda), str 
funcb
String address 0x55bdd2160280 (main lambda), str test
funcb
funcc
String address 0x55bdd21602b0 (main lambda), str test
funcd
String address 0x55bdd21602e0 (main lambda), str test
funce
String address 0x55bdd2160280 (main lambda), str test
```

由上面调用 `funca` 时的输出，可以看到 lambda 表达式以值的方式捕获的对象 str，其地址在 lambda 表达式内部和外部是不同的。

`std::function` 类对象和普通的魔板类对象一样，可以拷贝构造，如：
```
  std::function<void()> funcb = funca;
```

由调用 `funcb` 时的输出，可以看到拷贝构造时是做了逐成员的拷贝构造。

`std::function` 类对象可以赋值，如：
```
  std::function<void()> funcc;
  funcc = funca;
```
由调用 `funcc` 时的输出，可以看到赋值时是做了逐成员的赋值。

`std::function` 类对象可以移动构造，如：
```
  std::function<void()> funcd = std::move(funca);
```
由移动构造之后，调用 `funca` 和 `funcd` 时的输出，可以看到移动构造时是做了逐成员的移动构造。

`std::function` 类对象可以移动赋值，如：
```
  std::function<void()> funce;
  funce = std::move(funcb);

  printf("funcb\n");
//  funcb();
```
这里把移动赋值之后对 `funcb` 的调用注释掉了，这是因为，作为源的 `funcb` 在移动赋值之后被调用是，会抛出异常，如：
```
String address 0x562334c34280 (main lambda), str test
funcb
terminate called after throwing an instance of 'std::bad_function_call'
  what():  bad_function_call
```

同时，由调用 `funce` 时的输出可以看到，该输出与 `funcb` 在移动赋值之前被调用时的输出完全相同。**即移动赋值是将对象整体 move 走了，这与移动构造时的行为不太一样。**

`std::function` 类对象的拷贝构造或者赋值，也需要满足类型匹配原则，如：
```
  std::function<void(int)> funcf = funce;
```
这行代码会造成编译失败，编译错误信息如下：
```
../src/DemoTest.cpp: In function ‘int main()’:
../src/DemoTest.cpp:64:36: error: conversion from ‘std::function<void()>’ to non-scalar type ‘std::function<void(int)>’ requested
   std::function<void(int)> funcf = funce;
                                    ^~~~~
make: *** [src/DemoTest.o] Error 1
src/subdir.mk:18: recipe for target 'src/DemoTest.o' failed
```

在 lambda 中以值的方式捕获的右值对象，只是在 lambda 的 std::function 对象中做了一份被捕获的右值对象的拷贝，而原来的右值则没有任何改变。

接下来再来看一段示例代码：
```
#include <iostream>
#include <functional>
#include <string>

using namespace std;

void funcd(std::string &&str) {
  printf("String address %p in funcd A, str %s\n", &str, str.c_str());
  string strs = std::move(str);
  printf("String address %p in funcd B, str %s, strs %s\n", &str, str.c_str(), strs.c_str());
}

void funcc(std::string str) {
  printf("String address %p in funcc, str %s\n", &str, str.c_str());
}

void funcb(std::string &str) {
  printf("String address %p in funcb, str %s\n", &str, str.c_str());
}

void funca(std::string &&str) {
  printf("String address %p in funca A, str %s\n", &str, str.c_str());
  std::string stra = str;
  printf("String address %p in funca B, str %s, stra %s\n", &str, str.c_str(), stra.c_str());
}

int main() {
  std::string str = "test";
  printf("String address %p in main A, str %s\n", &str, str.c_str());

  funca(std::move(str));
  printf("String address %p in main B, str %s\n", &str, str.c_str());

//  funcb(std::move(str));
  printf("String address %p in main C, str %s\n", &str, str.c_str());

  funcc(std::move(str));
  printf("String address %p in main D, str %s\n", &str, str.c_str());

  std::string stra = "testa";
  printf("String address %p in main E, stra %s\n", &stra, stra.c_str());

  funcd(std::move(stra));
  printf("String address %p in main F, stra %s\n", &stra, stra.c_str());

  return 0;
}
```

上面这段代码在执行时，输出如下：
```
String address 0x7ffc833f4660 in main A, str test
String address 0x7ffc833f4660 in funca A, str test
String address 0x7ffc833f4660 in funca B, str test, stra test
String address 0x7ffc833f4660 in main B, str test
String address 0x7ffc833f4660 in main C, str test
String address 0x7ffc833f4680 in funcc, str test
String address 0x7ffc833f4660 in main D, str 
String address 0x7ffc833f4680 in main E, stra testa
String address 0x7ffc833f4680 in funcd A, str testa
String address 0x7ffc833f4680 in funcd B, str , strs testa
String address 0x7ffc833f4680 in main F, stra 
```

`funca` 函数接收右值引用作为参数，由 `funca` 函数内部及函数调用前后的输出可以看到，`std::move()` 本身什么都没做，单单调用 `std::move()` 并不会将原来的对象的内容移动到任何地方。`std::move()` 只是一个简单的强制类型转换，将左值转为右值引用。同时可以看到，用右值引用作为参数构造对象，也并没有对右值引用所引用的对象产生任何影响。

`funcb` 函数接收左值引用作为参数，上面的代码中，如下这一行注释掉了：
```
//  funcb(std::move(str));
```
这是因为，`funcb` 不能用一个右值引用作为参数来调用。用右值引用作为参数，调用接收左值引用作为参数的函数 `funcb` 时，会编译失败：
```
g++ -O0 -g3 -Wall -c -fmessage-length=0 -MMD -MP -MF"src/DemoTest.d" -MT"src/DemoTest.o" -o "src/DemoTest.o" "../src/DemoTest.cpp"
../src/DemoTest.cpp: In function ‘int main()’:
../src/DemoTest.cpp:34:18: error: cannot bind non-const lvalue reference of type ‘std::__cxx11::string& {aka std::__cxx11::basic_string<char>&}’ to an rvalue of type ‘std::remove_reference<std::__cxx11::basic_string<char>&>::type {aka std::__cxx11::basic_string<char>}’
   funcb(std::move(str));
         ~~~~~~~~~^~~~~
../src/DemoTest.cpp:17:6: note:   initializing argument 1 of ‘void funcb(std::__cxx11::string&)’
 void funcb(std::string &str) {
      ^~~~~
src/subdir.mk:18: recipe for target 'src/DemoTest.o' failed
make: *** [src/DemoTest.o] Error 1
```
不过，如果 `funcb` 接收 const 左值引用作为参数，如 `void funcb(const std::string &str)`，则在调用该函数时，可以用右值引用作为参数，此时 `funcb` 的行为与 `funca` 基本相同。

`funcc` 函数接收左值作为参数，由 `funcc` 函数内部及函数调用前后的输出可以看到，由于有了左值作为接收者，传入的右值引用所引用的对象的值被 move 走，进入函数的参数栈对象中了。

`funcd` 函数与 `funca` 函数一样，接收右值引用作为参数，但 `funcd` 的特别之处在于，在函数内部，右值构造了一个新的对象，因而右值引用原来引用的对象的值被 move 走，进入了新构造的对象中。

再来看一段示例代码：
```
#include <iostream>
#include <functional>
#include <string>

using namespace std;

void bar(std::string &&str) {
  printf("String address %p in bar A, str %s\n", &str, str.c_str());
  string strs = std::move(str);
  printf("String address %p in bar B, str %s, strs %s\n", &str, str.c_str(), strs.c_str());
}

std::function<void()> bar_bar(std::string &&str) {
  auto funf = [&str]() {
    printf("String address %p (foo lambda) F, stra %s\n", &str, str.c_str());
  };
  return funf;
}

std::function<void()> foo(std::string &&str) {
  printf("String address %p in foo A, str %s\n", &str, str.c_str());

//  auto funa = [str]() {
//    printf("String address %p (foo lambda) A, str %s\n", &str, str.c_str());
//    bar(str);
//  };
//  funa();
//
//  auto funb = [str]() {
//    printf("String address %p (foo lambda) B, str %s\n", &str, str.c_str());
//    bar(std::move(str));
//  };
//  funb();

//  auto func = [str]() mutable {
//    printf("String address %p (foo lambda) C, str %s\n", &str, str.c_str());
//    bar(str);
//  };
//  func();

  auto fund = [str]() mutable {
    printf("String address %p (foo lambda) D, str %s\n", &str, str.c_str());
    bar(std::move(str));
  };
  fund();

  auto fune = [&str]() {
    printf("String address %p (foo lambda) E, str %s\n", &str, str.c_str());
    bar(std::move(str));
  };
  fune();

  std::string stra = "testa";
  return bar_bar(std::move(stra));
}

int main() {
  std::string str = "test";
  printf("String address %p in main A, str %s\n", &str, str.c_str());

  auto funcg = foo(std::move(str));
  printf("String address %p in main B, str %s\n", &str, str.c_str());

  funcg();

  return 0;
}
```

上面这段代码的输出如下：
```
String address 0x7ffc9fe7c5c0 in main A, str test
String address 0x7ffc9fe7c5c0 in foo A, str test
String address 0x7ffc9fe7c540 (foo lambda) D, str test
String address 0x7ffc9fe7c540 in bar A, str test
String address 0x7ffc9fe7c540 in bar B, str , strs test
String address 0x7ffc9fe7c5c0 (foo lambda) E, str test
String address 0x7ffc9fe7c5c0 in bar A, str test
String address 0x7ffc9fe7c5c0 in bar B, str , strs test
String address 0x7ffc9fe7c5c0 in main B, str 
String address 0x7ffc9fe7c560 (foo lambda) F, stra ����
```

在函数 `foo()` 中定义的 `funa` 及对 `funa` 的调用被注释掉了，这是因为这段代码会导致编译失败，具体的错误信息如下：
```
Invoking: GCC C++ Compiler
g++ -O0 -g3 -Wall -c -fmessage-length=0 -MMD -MP -MF"src/DemoTest.d" -MT"src/DemoTest.o" -o "src/DemoTest.o" "../src/DemoTest.cpp"
../src/DemoTest.cpp: In lambda function:
../src/DemoTest.cpp:25:12: error: cannot bind rvalue reference of type ‘std::__cxx11::string&& {aka std::__cxx11::basic_string<char>&&}’ to lvalue of type ‘const string {aka const std::__cxx11::basic_string<char>}’
     bar(str);
            ^
../src/DemoTest.cpp:7:6: note:   initializing argument 1 of ‘void bar(std::__cxx11::string&&)’
 void bar(std::string &&str) {
      ^~~
src/subdir.mk:18: recipe for target 'src/DemoTest.o' failed
make: *** [src/DemoTest.o] Error 1
```
如我们前面提到的，在 lambda 表达式中，以值的方式捕获右值引用时，会在编译器为该 lambda 表达式生成的 `std::function` 类中生成一个 const 对象，const 对象是不能作为右值引用来调用接收右值引用为参数的函数的。

在函数 `foo()` 中定义的 `funb`，相对于 `funa`，在调用 `bar()` 时，为 `str` 裹上了 `std::move()`。不过此时还是会编译失败。错误信息如下：
```
Invoking: GCC C++ Compiler
g++ -O0 -g3 -Wall -c -fmessage-length=0 -MMD -MP -MF"src/DemoTest.d" -MT"src/DemoTest.o" -o "src/DemoTest.o" "../src/DemoTest.cpp"
../src/DemoTest.cpp: In lambda function:
../src/DemoTest.cpp:31:18: error: binding reference of type ‘std::__cxx11::string&& {aka std::__cxx11::basic_string<char>&&}’ to ‘std::remove_reference<const std::__cxx11::basic_string<char>&>::type {aka const std::__cxx11::basic_string<char>}’ discards qualifiers
     bar(std::move(str));
         ~~~~~~~~~^~~~~
../src/DemoTest.cpp:7:6: note:   initializing argument 1 of ‘void bar(std::__cxx11::string&&)’
 void bar(std::string &&str) {
      ^~~
make: *** [src/DemoTest.o] Error 1
src/subdir.mk:18: recipe for target 'src/DemoTest.o' failed
```
在 `funb` 中，`str` 是个 const 对象，因而还是不行。

在函数 `foo()` 中定义的 `func`，相对于 `funa`，加了 `mutable` 修饰。此时还是会编译失败。错误信息如下：
```
Invoking: GCC C++ Compiler
g++ -O0 -g3 -Wall -c -fmessage-length=0 -MMD -MP -MF"src/DemoTest.d" -MT"src/DemoTest.o" -o "src/DemoTest.o" "../src/DemoTest.cpp"
../src/DemoTest.cpp: In lambda function:
../src/DemoTest.cpp:37:12: error: cannot bind rvalue reference of type ‘std::__cxx11::string&& {aka std::__cxx11::basic_string<char>&&}’ to lvalue of type ‘std::__cxx11::string {aka std::__cxx11::basic_string<char>}’
     bar(str);
            ^
../src/DemoTest.cpp:7:6: note:   initializing argument 1 of ‘void bar(std::__cxx11::string&&)’
 void bar(std::string &&str) {
      ^~~
make: *** [src/DemoTest.o] Error 1
src/subdir.mk:18: recipe for target 'src/DemoTest.o' failed
```
无法将左值绑定到一个右值引用上。

在函数 `foo()` 中定义的 `fund`，相对于 `func`，在调用 `bar()` 时，为 `str` 裹上了 `std::move()`。此时终于可以编译成功，可以 move const 的 `str`。

在函数 `foo()` 中定义的 `fune`，相对于 `funb`，以引用的方式捕获了右值引用。在 `fune` 中调用 `bar()`，就如同 `foo()` 直接调用 `bar()` 一样。

在函数 `foo()` 中调用接收一个右值引用作为参数的函数 `bar_bar()` 生成一个函数。在函数 `bar_bar()` 中用 lambda 定义的函数对象 `funf`，以引用的方式捕获一个右值，并在 lambda 中访问改对象。该 lambda 作为 `bar_bar()` 函数生成的函数对象。`foo()` 中调用 `bar_bar()` 时传入函数栈上定义的临时对象 `stra`，并将 `bar_bar()` 返回的函数对象作为返回值返回。在 `main()` 函数中用 `funcg` 接收 `foo()` 函数返回的函数对象，并调用 `funcg`，此时会发生 crash 或能看到乱码。crash 或乱码是因为，在 funf 中，访问的 `str` 对象实际上是 `foo()` 函数中定义的栈上临时对象 `stra`，`foo()` 函数调用结束之后，栈上的临时对象被释放，`main()` 函数中调用 `funcg` 实际在访问一个无效的对象，因而出现问题。

Done。
