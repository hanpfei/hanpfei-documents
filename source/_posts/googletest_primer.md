---
title: Googletest 入门
date: 2019-08-01 21:05:49
categories: C/C++开发
tags:
- C/C++开发
- 翻译
---

# 简介：为什么是 googletest？
`googletest` 可以帮助我们更好地编写 C++ 测试用例。

googletest 是一个由 Google 的测试技术团队开发的测试框架，它考虑到了谷歌的特定需求和限制。无论你使用的是 Linux、Windows 还是 Mac，只要你编写 C++ 代码，googletest 都可以帮到你。它支持任何类型的测试，不只是单元测试。

那么，什么是好的测试，以及 googletest 是如何做到这些的呢？我们相信：
<!--more-->
1. 测试应该是 *独立的* 且 *可重复的*。调试由于其它测试而成功或失败的测试是令人痛苦的，googletest 通过在不同的对象上运行每个测试用例来隔离测试。当测试失败时，googletest 允许你单独运行它，以便快速调试。

2. 测试应该得到良好的*组织*，并反映测试代码的结构。googletest 将相关的测试分组为测试套件，它们可以共享数据和子例程。这种常见的模式很容易接受，并且使测试很容易维护。这样的一致性在人们切换项目并开始工作在一个新的代码库上时尤其有用。

3. 测试应该是 *可移植的* 且 *可复用的*。Google 具有大量的平台无关的代码，它的测试也应该是平台无关的。googletest 适用于不同的操作系统，不同的编译器，有异常或没有异常，所以 googletest 测试可以使用多种配置。

4. 当测试失败时，它们应该提供尽可能多的关于故障的 *信息*。googletest  不会在第一个测试失败时停止。相反，它仅停止当前的测试并继续运行下一个。你还可以设置测试报告非致命故障，在此之后，当前测试将继续运行。这样，你可以在一个运行-编辑-编译周期中探测并解决多个 bug。

5. 测试框架应该将测试编写者从家务活中解放出来，并让他们将精力集中在测试 *内容* 上。googletest 自动追踪所有定义的测试，且无需用户以运行它们的顺序迭代它们。

6. 测试要 *快*。通过 googletest，你可以跨测试用例复用共享资源，且只支付一次 set-up/tear-down 的开销，不使测试相互依赖。

由于 googletest 是基于流行的 xUnit 框架的，如果你以前用过 JUnit 或 PYUnit，你会觉得很自在。如果没有，你需要大约 10 分钟来学习基础知识并开始使用。因此让我们开始吧。

# 术语说明

注意：由于术语 *测试*，*测试用例* 和 *测试套件* 的定义不同，可能会出现一些概念上的混淆，因此要注意不要误解这些术语。

从历史上看，googletest 刚开始使用术语 *测试用例* 来分组相关的测试，然而当前的出版物包括国际软件测试资格委员会([ISTQB](http://www.istqb.org/))及大量关于软件质量的教材使用术语 *[测试套件](http://glossary.istqb.org/search/test%20suite)* 来表示这一含义。

与 googletest 中使用的术语 *测试* 相对应的是 ISTQB 等的术语 *[测试用例](http://glossary.istqb.org/search/test%20case)*。

术语 *测试* 通常具有足够广泛的含义，包括 ISTQB 对 *测试用例* 的定义，所以这里没有太多问题。但 Google Test 中使用的术语 *测试用例* 具有矛盾的含义，因此令人困惑。

googletest 最近开始用 *测试套件* 替换术语 *测试用例*。首选的 API 是 TestSuite*。旧的 TestCase* API 正慢慢地被弃用和重构。

因此请注意术语的不同定义：

含义                                                                              | googletest 术语         | [ISTQB](http://www.istqb.org/) 术语
:----------------------------------------------------------------------------------- | :---------------------- | :----------------------------------
以特定输入值执行一个特定的程序路径并验证结果 | [TEST()](#simple-tests) | [Test Case](http://glossary.istqb.org/search/test%20case)

# 基本概念

使用 googletest 时，从编写 *断言* 开始，这些语句检查条件是否为真。一个断言的结果可以是 *成功*，*非致命失败*，或者*致命失败*。如果发生了致命失败，它终止当前函数；否则程序继续正常运行。

*测试* 使用断言验证被测代码的行为。如果一个测试崩溃或有一个失败的断言，则它 *失败*；否则它 *成功*。

*测试套件* 包含一个或多个测试。你应该将测试分组到反映被测代码结构的测试套件中。当一个测试套件中的多个测试需要共享相同的对象和子例程时，你可以把它们放进一个 *测试夹具* 类中。

一个 *测试程序* 可以包含多个测试套件。

我们将解释如何编写测试程序，从单个断言级别开始，直到测试和测试套件。

# 断言

googletest 断言是像函数调用一样的宏。你通过制造关于类或函数的行为的断言来测试它。当一个断言失败时，googletest 打印断言的源文件和行号的位置，以及一条失败消息。你也可以提供定制的失败消息，它们将被加在 googletest 的消息的后面。

断言成对的出现，它们测试相同的事情但对当前函数具有不同的影响。`ASSERT_*` 版本在它们失败时生成致命失败，并**终止当前函数**。`EXPECT_*` 版本生成非致命失败，它们不终止当前函数。通常 `EXPECT_*` 是首选，它们允许在一个测试中报告更多失败。然而，如果断言的问题失败时继续执行没有意义，则你应该使用 `ASSERT_*`。

由于失败的 `ASSERT_*` 立即从当前函数返回，则可能跳过之后的清理代码，它可能导致空间泄露。依赖于泄露的属性，它可能值得或不值得解决 - 因此，如果在断言错误之外还出现堆检查器错误，请记住这一点。

为了提供定制的失败消息，简单的使用 `<<` 操作符，或一串这种操作符，把它送进宏。一个例子如下：
```
ASSERT_EQ(x.size(), y.size()) << "Vectors x and y are of unequal length";

for (int i = 0; i < x.size(); ++i) {
  EXPECT_EQ(x[i], y[i]) << "Vectors x and y differ at index " << i;
}
```

任何可以被送给 `ostream` 的东西都可以被送进断言的宏 - 特别是，C 字符串和 `string` 对象。如果宽字符串（`wchar_t*`， Windows 上 `UNICODE` 模式的 `TCHAR*`，或 `std::wstring`）被送进断言，它将在打印时被转换为 UTF-8。

## 基本断言

这些断言执行基本的 true/false 条件测试。

致命断言            | 非致命断言         | 验证
-------------------------- | -------------------------- | --------------------
`ASSERT_TRUE(condition);`  | `EXPECT_TRUE(condition);`  | `condition` 是 true
`ASSERT_FALSE(condition);` | `EXPECT_FALSE(condition);` | `condition` 是 false

记住，当它们失败时，`ASSERT_*` 产生一个致命错误，并从当前函数退出，`EXPECT_*` 则产生一个非致命错误，并允许函数继续运行。任何一种情况下，断言失败意味着包含它的测试失败。

**可用性**：Linux，Windows，Mac。

## 二元比较

这个部分描述比较两个值的断言。


致命断言          | 非致命断言       | 验证
------------------------ | ------------------------ | --------------
`ASSERT_EQ(val1, val2);` | `EXPECT_EQ(val1, val2);` | `val1 == val2`
`ASSERT_NE(val1, val2);` | `EXPECT_NE(val1, val2);` | `val1 != val2`
`ASSERT_LT(val1, val2);` | `EXPECT_LT(val1, val2);` | `val1 < val2`
`ASSERT_LE(val1, val2);` | `EXPECT_LE(val1, val2);` | `val1 <= val2`
`ASSERT_GT(val1, val2);` | `EXPECT_GT(val1, val2);` | `val1 > val2`
`ASSERT_GE(val1, val2);` | `EXPECT_GE(val1, val2);` | `val1 >= val2`

值参数必须是断言的比较操作符可比较的，否则将报出编译错误。我们常要求参数支持 `<<` 操作符，以便于把它们送进 `ostream`，但这不再是必须的。如果支持 `<<`，则在断言失败时它将被调用来打印参数；否则 googletest 将尝试以它能找到的最好的方式打印它们。更多细节及如何定制参数的打印的信息，请参考[文档](https://github.com/google/googletest/blob/master/googlemock/docs/cook_book.md#teaching-gmock-how-to-print-your-values)。

这些断言可以使用用户定义的类型，但只有你定义了对应的比较操作符（比如 ==，<，等等）。由于这是 Google [C++ Style Guide](https://google.github.io/styleguide/cppguide.html#Operator_Overloading) 禁止的，你可以使用 `ASSERT_TRUE()` 或 `EXPECT_TRUE()` 断言两个用户定义类型的对象的相等性。

然而，当可能时，`ASSERT_EQ(actual, expected)` 好于 `ASSERT_TRUE(actual == expected)`，因为它在失败时告诉你 `actual` 和 `expected` 的值。

参数总是精确地计算一次。因此，参数有副作用也没关系。然而正如任何普通的 C/C++ 函数那样，参数的求值顺序是未定义的（比如编译器有选择任何顺序的自由），你的代码不应该依赖任何特定的参数求值顺序。

`ASSERT_EQ()` 对指针执行相等性操作。如果使用两个 C 字符串，它测试它们是否位于相同的内存位置，而不是它们是否具有相同的值。因此，如果你想比较 C 字符串的值（比如 `const char*`），使用 `ASSERT_STREQ()`，它将在后面描述。特别地，要断言 C 字符串是 `NULL`，则使用 `ASSERT_STREQ(c_string, NULL)`，如果支持 c++11 则考虑使用 `ASSERT_EQ(c_string, nullptr)`。要比较两个 `string` 对象，你应该使用 `ASSERT_EQ`。

当执行指针比较时使用 `*_EQ(ptr, nullptr)` 和 `*_NE(ptr, nullptr)` 而不是 `*_EQ(ptr, NULL)` 和 `*_NE(ptr, NULL)`。这是因为 `nullptr` 是类型安全的而 `NULL` 不是。参考 [FAQ](https://github.com/google/googletest/blob/master/googletest/docs/faq.md) 获得更多信息。

如果你在使用浮点数，你可能想要使用这些宏的浮点变体以避免四舍五入导致的问题。参考 [高级 googletest 主题](https://github.com/google/googletest/blob/master/googletest/docs/advanced.md) 了解更多信息。

这一节的宏可以同时用于窄的和宽的字符串对象（`string` 和 `wstring`）。

**可用性**：Linux，Windows，Mac。

**历史注释**：在 2016 年 2 月之前 `*_EQ` 习惯上称其为 `ASSERT_EQ(expected, actual)`。，因此大量已有的代码使用这一顺序。现在 `*_EQ` 用同样的方法处理两个参数。

## 字符串比较
这一组断言比较两个 **C 字符串**。如果你想比较两个 `string` 对象，则使用 `EXPECT_EQ`，`EXPECT_NE`，等等。

| 致命断言         | 非致命断言      | 验证               |
| ----------------------- | ----------------------- | ---------------------- |
| `ASSERT_STREQ(str1, str2);`    | `EXPECT_STREQ(str1,  str2);`   | 两个 C 字符串具有相同的内容 |
| `ASSERT_STRNE(str1, str2);`  | `EXPECT_STRNE(str1, str2);`     | 两个 C 字符串具有不同的内容 |
| `ASSERT_STRCASEEQ(str1, str2);`  | `EXPECT_STRCASEEQ(str1, str2);` |两个 C 字符串具有相同的内容，忽略大小写 |
| `ASSERT_STRCASENE(str1, str2);`  | `EXPECT_STRCASENE(str1, str2);` | 两个 C 字符串具有不同的内容，忽略大小写 |

注意断言名字中的 "CASE" 意味着忽略大小写。`NULL` 指针和空字符串被认为是不同的。

`*STREQ*` 和 `*STRNE*` 也接受宽 C 字符串（`wchar_t*`）。如果比较两个宽字符串失败，则它们的值将以 UTF-8 窄字符串的形式打印。

**可用性**：Linux，Windows，Mac。

**另请参阅**：更多字符串比较技巧（子字符串，前缀，后缀，和正则表达式匹配，比如），请参考高级 googletest 指南的[这个部分](https://github.com/google/googletest/blob/master/googletest/docs/advanced.md)。

# 简单的测试

要创建一个测试：

 1. 使用 `TEST()` 宏定义并命名一个测试函数。这些是普通的没有返回值的 C++  函数。
 2. 在这个函数中，可以包含任何你想包含的有效的 C++ 语句，使用各种 googletest 断言检查值。
 3. 测试的结果由断言决定；如果测试中的任何断言失败了（致命的或非致命的），或者如果测试崩溃了，则整个测试失败。否则，测试成功。
```
TEST(TestSuiteName, TestName) {
  ... test body ...
}
```
`TEST()` 的参数从一般到具体。*第一个*参数是测试套件的名字，*第二个*参数是测试用例中测试的名字。这两个名字都必须是有效的 C++ 标识符，且它们都不应该包含下划线（`_`）。测试的 `完整名字` 由包含它的测试套件和它自己的独立名字组成。不同的测试套件中的测试可以具有相同的独立名字。

比如，让我们看一个简单的整数函数：
```
int Factorial(int n);  // Returns the factorial of n
```

这个函数的测试套件看起来可能像下面这样：
```
// Tests factorial of 0.
TEST(FactorialTest, HandlesZeroInput) {
  EXPECT_EQ(Factorial(0), 1);
}

// Tests factorial of positive numbers.
TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(Factorial(1), 1);
  EXPECT_EQ(Factorial(2), 2);
  EXPECT_EQ(Factorial(3), 6);
  EXPECT_EQ(Factorial(8), 40320);
}
```

googletest 根据测试套件来分组结果，因此逻辑上相关的测试应该放进相同的测试套件里；换句话说，它们的 `TEST()` 的第一个参数应该是相同的。在上面的例子中，我们有两个测试，`HandlesZeroInput` 和 `HandlesPositiveInput`，它们属于相同的测试套件 `FactorialTest`。

当命名你的测试套件和测试时，你应该遵循如[命名函数和类](https://google.github.io/styleguide/cppguide.html#Function_Names)相同的规则。

**可用性**：Linux，Windows，Mac。

# 测试夹具：多个测试使用相同的数据配置

如果你发现你写了两个或更多测试操作类似的数据，你可以使用一个 *测试夹具*。它允许你为多个不同的测试复用相同的对象配置。

要创建一个测试夹具：

 1. 创建一个继承自 `::testing::Test` 的类。由于我们想要从子类访问夹具的成员，因此从 `protected:` 开始创建类体。
 2. 在类内部，生命任何你打算使用的对象。
 3. 如果有需要，编写一个默认的构造函数或 `SetUp()` 函数为每个测试准备对象。一个常见的错误是把 `SetUp()` 拼成了 `Setup()`，其中有一个 `u` - 使用 C++11 中的 `override` 确保你正确地拼写了它。
 4. 如果有需要，编写一个析构函数或 `TearDown()` 函数释放你在 `SetUp()` 中分配的所有资源。要学习何时你应该使用构造函数/析构函数以及何时你应该使用`SetUp()`/`TearDown()`，请阅读 [FAQ](https://github.com/google/googletest/blob/master/googletest/docs/faq.md)。

当使用测试夹具时，使用 `TEST_F()` 而不是 `TEST()`，它允许你访问测试夹具中的对象和子例程：
```
TEST_F(TestFixtureName, TestName) {
  ... test body ...
}
```

像 `TEST()` 一样，第一个参数是测试套件的名字，但对于 `TEST_F()`，这必须是测试夹具类的名字。你可能已经猜到：`_F` 指 fixture。

不幸的是，C++ 宏系统不允许我们创建单个能处理这两种类型的测试的宏。使用错误的宏将导致编译器错误。

而且，你必须在使用 `TEST_F()` 之前先定义一个测试夹具类，否则你将得到一个编译器错误 "virtual outside class declaration"。

对于通过 `TEST_F()` 定义的每个测试，googletest 将在运行时创建一个 *全新的* 测试夹具，立即通过 `SetUp()` 初始化它，运行测试，通过调用 `TearDown()` 清理资源，然后删除测试夹具。注意相同测试套件中的不同测试夹具具有不同的测试夹具类，且 googletest 总是在创建下一个之前删除测试夹具。googletest **不** 为多个测试复用相同的测试夹具。一个测试对夹具的任何修改不影响其它的测试。

举个例子，让我们为名为 `Queue` 的 FIFO 队列类编写测试，它具有如下的接口：
```
template <typename E>  // E is the element type.
class Queue {
 public:
  Queue();
  void Enqueue(const E& element);
  E* Dequeue();  // Returns NULL if the queue is empty.
  size_t size() const;
  ...
};
```

首先，定义一个夹具类。按照惯例，当被测试的类是 `Foo` 时你应该把它命名为 `FooTest`。
```
class QueueTest : public ::testing::Test {
 protected:
  void SetUp() override {
     q1_.Enqueue(1);
     q2_.Enqueue(2);
     q2_.Enqueue(3);
  }

  // void TearDown() override {}

  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};
```

在这个例子中，不需要 `TearDown()`，因为在每个测试之后，我们无需清理资源，这些已经由析构函数完成了。

现在我们将编写使用 `TEST_F()` 和这个夹具的测试。
```
TEST_F(QueueTest, IsEmptyInitially) {
  EXPECT_EQ(q0_.size(), 0);
}

TEST_F(QueueTest, DequeueWorks) {
  int* n = q0_.Dequeue();
  EXPECT_EQ(n, nullptr);

  n = q1_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 1);
  EXPECT_EQ(q1_.size(), 0);
  delete n;

  n = q2_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 2);
  EXPECT_EQ(q2_.size(), 1);
  delete n;
}
```

上例同时使用了 `ASSERT_*` 和 `EXPECT_*` 断言。经验法则是当你想要断言失败时测试继续执行揭露更多错误时使用 `EXPECT_*`，当失败之后继续执行没有意义时使用 `ASSERT_*`。比如，`Dequeue` 测试中的第二个断言是 `ASSERT_NE(nullptr, n)`，由于我们后面需要解引用指针 `n`，当 `n` 为 `NULL` 时这将导致段错误。

当运行这些测试时，将发生如下的事情：

 1. googletest 构造一个 `QueueTest` 对象（让我们称它为 `t1`）。
 2. `t1.SetUp()` 初始化 `t1`。
 3. 在 `t1` 上运行第一个测试（`IsEmptyInitially`）。
 4. 测试结束之后 `t1.TearDown()` 清理资源。
 5. `t1` 被销毁。
 6. 上面的步骤在另一个 `QueueTest` 对象上重复，这次运行 `DequeueWorks` 测试。

**可用性**：Linux，Windows，Mac。

# 调用测试

`TEST()` 和 `TEST_F()` 隐式地把它们的测试注册给 googletest。因此，不像许多其它的 C++ 测试框架，因此你无需以运行的顺序把你定义的测试重新列出。

定义你的测试之后，你可以通过 `RUN_ALL_TESTS()` 运行它们，如果所有的测试都成功了它将返回 `0`，否则返回 `1`。注意 `RUN_ALL_TESTS()` 运行你的链接单元中的 *所有测试* -- 它们可以来自于不同的测试套件，甚至是不同的源文件。

当被调用时，`RUN_ALL_TESTS()` 宏：

 - 保存所有的 googletest 标记的状态
 - 为第一个测试创建一个测试夹具对象。
 - 通过 `SetUp()` 初始化它。
 - 在测试夹具对象上运行测试。
 - 通过 `TearDown()` 清理测试夹具。
 - 删除夹具。
 - 恢复所有的 googletest 标记的状态
 - 为下一个测试重复上述步骤，知道所有测试都已经运行完。

如果发生了致命失败则后续的步骤将被跳过。


> 重要：你一定不能忽略 `RUN_ALL_TESTS()` 的返回值，否则你将得到一个编译错误。这种设计的原理是自动化测试服务是基于测试的退出码来决定它是否通过的，而不是它的 stdout/stderr 输出，因此你的 `main()` 函数必须返回 `RUN_ALL_TESTS()` 的值。
>
> 而且，你应该只调用 `RUN_ALL_TESTS()` **一次**。多次调用它将与 googletest 的一些高级功能冲突（比如线程安全的 [death tests](https://github.com/google/googletest/blob/master/googletest/docs/advanced.md#death-tests)）且这是不支持的。

**可用性**：Linux，Windows，Mac。

# 编写 main() 函数

编写你自己的 `main()` 函数，它应该返回 `RUN_ALL_TESTS()` 的值。

你可以从下面的示例代码开始：
```
#include "this/package/foo.h"
#include "gtest/gtest.h"

namespace {

// The fixture for testing class Foo.
class FooTest : public ::testing::Test {
 protected:
  // You can remove any or all of the following functions if its body
  // is empty.

  FooTest() {
     // You can do set-up work for each test here.
  }

  ~FooTest() override {
     // You can do clean-up work that doesn't throw exceptions here.
  }

  // If the constructor and destructor are not enough for setting up
  // and cleaning up each test, you can define the following methods:

  void SetUp() override {
     // Code here will be called immediately after the constructor (right
     // before each test).
  }

  void TearDown() override {
     // Code here will be called immediately after each test (right
     // before the destructor).
  }

  // Objects declared here can be used by all tests in the test suite for Foo.
};

// Tests that the Foo::Bar() method does Abc.
TEST_F(FooTest, MethodBarDoesAbc) {
  const std::string input_filepath = "this/package/testdata/myinputfile.dat";
  const std::string output_filepath = "this/package/testdata/myoutputfile.dat";
  Foo f;
  EXPECT_EQ(f.Bar(input_filepath, output_filepath), 0);
}

// Tests that Foo does Xyz.
TEST_F(FooTest, DoesXyz) {
  // Exercises the Xyz feature of Foo.
}

}  // namespace

int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```

`::testing::InitGoogleTest()` 函数解析 googletest 标记的命令行参数，并移除所有已识别的标记。这允许用户通过不同的标记控制测试程序的行为，我们将在 [AdvancedGuide](https://github.com/google/googletest/blob/master/googletest/docs/advanced.md) 描述相关的内容。你**必须**在调用 `RUN_ALL_TESTS()` 之前调用这个函数，否则标记将无法得到适当的初始化。

在 Windows 上，`InitGoogleTest()` 也可以用宽字符串，因此它也可以被用于以 `UNICODE` 模式编译的程序。

但是正如你可能认为的那样，编写所有这些 `main()` 函数太麻烦了？我们完全同意你的看法，那就是 Google Test 为什么提供一个基本的 `main()` 函数实现的原因。如果它能满足你的需要，则把你的测试与 gtest_main 库链接在一起就好，然后你可以走了。

注意：`ParseGUnitFlags()` 被 `InitGoogleTest()` 废弃了。

# 已知限制

 - Google Test 被设计为线程安全的。`pthreads` 库可用的系统上的实现是线程安全的。当前在其它系统（比如 Windows）上在两个线程中并发地使用 Google Test 断言是*不安全的*。在大多数测试中这通常不是问题，断言在主线程中完成。如果你想提供帮助，你可以志愿在 `gtest-port.h` 中为你的平台实现需要的同步原语。

[原文](https://github.com/google/googletest/blob/master/googletest/docs/primer.md)
