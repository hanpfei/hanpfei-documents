---
title: Googletest 实现简要分析
date: 2019-08-22 21:05:49
categories: C/C++开发
tags:
- C/C++开发
---

借助于 Googletest 测试框架，我们只需编写测试用例代码，并定义简单的 `main()` 函数，编译之后并运行即可以把我们的测试用例跑起来。(更详细的内容可参考 [Googletest 入门](https://www.wolfcstech.com/2019/08/01/googletest_primer/))。但 `main()` 函数调用 `RUN_ALL_TESTS()` 时，是如何找到并运行我们编写的测试用例代码的呢？本文尝试找寻 Googletest 框架背后隐藏的这些秘密。（代码分析基于 `git@github.com:google/googletest.git` 的 commit 2134e3fd857d952e03ce76064fad5ac6e9036104 的版本。）
<!--more-->
# 从 TEST 和 TEST_F 说起

`googletest/googletest/include/gtest/gtest.h` 中 `TEST` 和 `TEST_F` 两个宏的定义如下：
```
// Note that we call GetTestTypeId() instead of GetTypeId<
// ::testing::Test>() here to get the type ID of testing::Test.  This
// is to work around a suspected linker bug when using Google Test as
// a framework on Mac OS X.  The bug causes GetTypeId<
// ::testing::Test>() to return different values depending on whether
// the call is from the Google Test framework itself or from user test
// code.  GetTestTypeId() is guaranteed to always return the same
// value, as it always calls GetTypeId<>() from the Google Test
// framework.
#define GTEST_TEST(test_suite_name, test_name)             \
  GTEST_TEST_(test_suite_name, test_name, ::testing::Test, \
              ::testing::internal::GetTestTypeId())

// Define this macro to 1 to omit the definition of TEST(), which
// is a generic name and clashes with some other libraries.
#if !GTEST_DONT_DEFINE_TEST
#define TEST(test_suite_name, test_name) GTEST_TEST(test_suite_name, test_name)
#endif

// Defines a test that uses a test fixture.
//
// The first parameter is the name of the test fixture class, which
// also doubles as the test suite name.  The second parameter is the
// name of the test within the test suite.
//
// A test fixture class must be declared earlier.  The user should put
// the test code between braces after using this macro.  Example:
//
//   class FooTest : public testing::Test {
//    protected:
//     void SetUp() override { b_.AddElement(3); }
//
//     Foo a_;
//     Foo b_;
//   };
//
//   TEST_F(FooTest, InitializesCorrectly) {
//     EXPECT_TRUE(a_.StatusIsOK());
//   }
//
//   TEST_F(FooTest, ReturnsElementCountCorrectly) {
//     EXPECT_EQ(a_.size(), 0);
//     EXPECT_EQ(b_.size(), 1);
//   }
//
// GOOGLETEST_CM0011 DO NOT DELETE
#define TEST_F(test_fixture, test_name)\
  GTEST_TEST_(test_fixture, test_name, test_fixture, \
              ::testing::internal::GetTypeId<test_fixture>())
```
`TEST` 和 `TEST_F` 两个宏最终都借助于 `GTEST_TEST_` 宏完成其工作。

`googletest/googletest/include/gtest/internal/gtest-internal.h` 中 `GTEST_TEST_` 宏的定义如下：
```
// Expands to the name of the class that implements the given test.
#define GTEST_TEST_CLASS_NAME_(test_suite_name, test_name) \
  test_suite_name##_##test_name##_Test

// Helper macro for defining tests.
#define GTEST_TEST_(test_suite_name, test_name, parent_class, parent_id)      \
  class GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)                    \
      : public parent_class {                                                 \
   public:                                                                    \
    GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)() {}                   \
                                                                              \
   private:                                                                   \
    virtual void TestBody();                                                  \
    static ::testing::TestInfo* const test_info_ GTEST_ATTRIBUTE_UNUSED_;     \
    GTEST_DISALLOW_COPY_AND_ASSIGN_(GTEST_TEST_CLASS_NAME_(test_suite_name,   \
                                                           test_name));       \
  };                                                                          \
                                                                              \
  ::testing::TestInfo* const GTEST_TEST_CLASS_NAME_(test_suite_name,          \
                                                    test_name)::test_info_ =  \
      ::testing::internal::MakeAndRegisterTestInfo(                           \
          #test_suite_name, #test_name, nullptr, nullptr,                     \
          ::testing::internal::CodeLocation(__FILE__, __LINE__), (parent_id), \
          ::testing::internal::SuiteApiResolver<                              \
              parent_class>::GetSetUpCaseOrSuite(__FILE__, __LINE__),         \
          ::testing::internal::SuiteApiResolver<                              \
              parent_class>::GetTearDownCaseOrSuite(__FILE__, __LINE__),      \
          new ::testing::internal::TestFactoryImpl<GTEST_TEST_CLASS_NAME_(    \
              test_suite_name, test_name)>);                                  \
  void GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)::TestBody()
```

`GTEST_TEST_` 宏定义一个类，它为定义的类实现如下特性：

 - 类名：类名由测试套件名、测试用例名和字符串 `Test` 3 部分组成，不同部分用下划线分割。如 `TEST_F(FooTest, InitializesCorrectly)` 和 `TEST(FooTest, InitializesCorrectly)` 定义的类名为 `FooTest_InitializesCorrectly_Test`。
 - 父类：`GTEST_TEST_` 宏定义的类继承自传给它的 `parent_class` 参数类。对于 `TEST_F` 宏而言，就是测试套件名，对于 `TEST` 则是 `::testing::Test` 类。
 - `GTEST_TEST_` 宏为它定义的类定义了一个空的默认构造函数。
 - `GTEST_TEST_` 宏为它定义的类定义并初始化类型为 `::testing::TestInfo* const` 的静态成员变量 `test_info_`，这个指针变量指向由 `::testing::internal::MakeAndRegisterTestInfo()` 函数创建的 `::testing::TestInfo` 类型对象。`::testing::internal::MakeAndRegisterTestInfo()` 函数还会将测试用例的信息注册给 ***Google Test***。
 - `GTEST_TEST_` 宏为它定义的类定义 `TestBody()` 虚函数覆盖父类的对应函数，函数的函数体正是我们编写的测试用例代码，这也就是为什么我们在测试用例代码体中打断点时，看到调用栈，代码是位于名为 `TestBody()` 的函数中的原因。
 - 通过宏 `GTEST_DISALLOW_COPY_AND_ASSIGN_` 实现类对象的禁用拷贝构造和赋值。

`::testing::internal::MakeAndRegisterTestInfo()` 函数定义（`googletest/googletest/src/gtest.cc`）如下：
```
// Creates a new TestInfo object and registers it with Google Test;
// returns the created object.
//
// Arguments:
//
//   test_suite_name:   name of the test suite
//   name:             name of the test
//   type_param:       the name of the test's type parameter, or NULL if
//                     this is not a typed or a type-parameterized test.
//   value_param:      text representation of the test's value parameter,
//                     or NULL if this is not a value-parameterized test.
//   code_location:    code location where the test is defined
//   fixture_class_id: ID of the test fixture class
//   set_up_tc:        pointer to the function that sets up the test suite
//   tear_down_tc:     pointer to the function that tears down the test suite
//   factory:          pointer to the factory that creates a test object.
//                     The newly created TestInfo instance will assume
//                     ownership of the factory object.
TestInfo* MakeAndRegisterTestInfo(
    const char* test_suite_name, const char* name, const char* type_param,
    const char* value_param, CodeLocation code_location,
    TypeId fixture_class_id, SetUpTestSuiteFunc set_up_tc,
    TearDownTestSuiteFunc tear_down_tc, TestFactoryBase* factory) {
  TestInfo* const test_info =
      new TestInfo(test_suite_name, name, type_param, value_param,
                   code_location, fixture_class_id, factory);
  GetUnitTestImpl()->AddTestInfo(set_up_tc, tear_down_tc, test_info);
  return test_info;
}
```

# 测试用例的注册

在通过 `TEST` 和 `TEST_F` 宏定义测试用例时，这些宏实际上将会为测试用例定义一个类，这个类包含一个静态成员变量，在这个静态成员变量初始化时，在 `::testing::internal::MakeAndRegisterTestInfo()` 函数中向 ***Google Test*** 注册测试用例，具体而言，通过 `::testing::internal::UnitTestImpl::AddTestInfo()` 函数注册测试用例，该函数在 `googletest/googletest/src/gtest-internal-inl.h` 中定义，定义如下：
```
  // Adds a TestInfo to the unit test.
  //
  // Arguments:
  //
  //   set_up_tc:    pointer to the function that sets up the test suite
  //   tear_down_tc: pointer to the function that tears down the test suite
  //   test_info:    the TestInfo object
  void AddTestInfo(internal::SetUpTestSuiteFunc set_up_tc,
                   internal::TearDownTestSuiteFunc tear_down_tc,
                   TestInfo* test_info) {
    // In order to support thread-safe death tests, we need to
    // remember the original working directory when the test program
    // was first invoked.  We cannot do this in RUN_ALL_TESTS(), as
    // the user may have changed the current directory before calling
    // RUN_ALL_TESTS().  Therefore we capture the current directory in
    // AddTestInfo(), which is called to register a TEST or TEST_F
    // before main() is reached.
    if (original_working_dir_.IsEmpty()) {
      original_working_dir_.Set(FilePath::GetCurrentDir());
      GTEST_CHECK_(!original_working_dir_.IsEmpty())
          << "Failed to get the current working directory.";
    }

    GetTestSuite(test_info->test_suite_name(), test_info->type_param(),
                 set_up_tc, tear_down_tc)
        ->AddTestInfo(test_info);
  }
```

`::testing::internal::UnitTestImpl::AddTestInfo()` 函数根据测试套件名字等参数获得测试套件；然后将测试用例添加进测试套件中。

`googletest/googletest/src/gtest.cc` 中用于获得测试套件的 `::testing::internal::UnitTestImpl::GetTestSuite()` 函数定义如下：
```
// Finds and returns a TestSuite with the given name.  If one doesn't
// exist, creates one and returns it.  It's the CALLER'S
// RESPONSIBILITY to ensure that this function is only called WHEN THE
// TESTS ARE NOT SHUFFLED.
//
// Arguments:
//
//   test_suite_name: name of the test suite
//   type_param:     the name of the test suite's type parameter, or NULL if
//                   this is not a typed or a type-parameterized test suite.
//   set_up_tc:      pointer to the function that sets up the test suite
//   tear_down_tc:   pointer to the function that tears down the test suite
TestSuite* UnitTestImpl::GetTestSuite(
    const char* test_suite_name, const char* type_param,
    internal::SetUpTestSuiteFunc set_up_tc,
    internal::TearDownTestSuiteFunc tear_down_tc) {
  // Can we find a TestSuite with the given name?
  const auto test_suite =
      std::find_if(test_suites_.rbegin(), test_suites_.rend(),
                   TestSuiteNameIs(test_suite_name));

  if (test_suite != test_suites_.rend()) return *test_suite;

  // No.  Let's create one.
  auto* const new_test_suite =
      new TestSuite(test_suite_name, type_param, set_up_tc, tear_down_tc);

  // Is this a death test suite?
  if (internal::UnitTestOptions::MatchesFilter(test_suite_name,
                                               kDeathTestSuiteFilter)) {
    // Yes.  Inserts the test suite after the last death test suite
    // defined so far.  This only works when the test suites haven't
    // been shuffled.  Otherwise we may end up running a death test
    // after a non-death test.
    ++last_death_test_suite_;
    test_suites_.insert(test_suites_.begin() + last_death_test_suite_,
                        new_test_suite);
  } else {
    // No.  Appends to the end of the list.
    test_suites_.push_back(new_test_suite);
  }

  test_suite_indices_.push_back(static_cast<int>(test_suite_indices_.size()));
  return new_test_suite;
}
```

`::testing::internal::UnitTestImpl::GetTestSuite()` 函数：

 1. 根据传入的测试套件名在测试套件 vector 中查找测试套件，如果找到就返回。
 2. 在测试套件 vector 中没有找到对应的测试套件，创建一个测试套件 `TestSuite` 对象。 
 3. 测试套件名字与 death test 测试套件名规则匹配时，创建的测试套件被放在测试套件 vector 的前面，并在执行时先执行。
 4. 测试套件名字与 death test 测试套件名规则不匹配时，创建的测试套件被放在测试套件 vector 的后面，并在执行时后执行。

向 `TestSuite` 添加测试用例的 `TestSuite::AddTestInfo()` 函数定义（`googletest/googletest/src/gtest.cc`）如下：
```
// Adds a test to this test suite.  Will delete the test upon
// destruction of the TestSuite object.
void TestSuite::AddTestInfo(TestInfo* test_info) {
  test_info_list_.push_back(test_info);
  test_indices_.push_back(static_cast<int>(test_indices_.size()));
}
```

# 测试用例的执行

在 `main()` 函数中，通过调用 `testing::InitGoogleTest(&argc, argv)` 和 `RUN_ALL_TESTS()` 执行所有的测试用例。`testing::InitGoogleTest(&argc, argv)` 主要用于解析命令行参数，如 filter 等，这里不再详细分析。`googletest/googletest/include/gtest/gtest.h` 中的 `RUN_ALL_TESTS()` 函数定义如下：
```
// Use this function in main() to run all tests.  It returns 0 if all
// tests are successful, or 1 otherwise.
//
// RUN_ALL_TESTS() should be invoked after the command line has been
// parsed by InitGoogleTest().
//
// This function was formerly a macro; thus, it is in the global
// namespace and has an all-caps name.
int RUN_ALL_TESTS() GTEST_MUST_USE_RESULT_;

inline int RUN_ALL_TESTS() {
  return ::testing::UnitTest::GetInstance()->Run();
}
```

`RUN_ALL_TESTS()` 函数调用 `googletest/googletest/src/gtest.cc` 中定义的 `UnitTest::Run()` 函数执行所有的测试用例：
```
// Runs all tests in this UnitTest object and prints the result.
// Returns 0 if successful, or 1 otherwise.
//
// We don't protect this under mutex_, as we only support calling it
// from the main thread.
int UnitTest::Run() {
  const bool in_death_test_child_process =
      internal::GTEST_FLAG(internal_run_death_test).length() > 0;

  // Google Test implements this protocol for catching that a test
  // program exits before returning control to Google Test:
  //
  //   1. Upon start, Google Test creates a file whose absolute path
  //      is specified by the environment variable
  //      TEST_PREMATURE_EXIT_FILE.
  //   2. When Google Test has finished its work, it deletes the file.
  //
  // This allows a test runner to set TEST_PREMATURE_EXIT_FILE before
  // running a Google-Test-based test program and check the existence
  // of the file at the end of the test execution to see if it has
  // exited prematurely.

  // If we are in the child process of a death test, don't
  // create/delete the premature exit file, as doing so is unnecessary
  // and will confuse the parent process.  Otherwise, create/delete
  // the file upon entering/leaving this function.  If the program
  // somehow exits before this function has a chance to return, the
  // premature-exit file will be left undeleted, causing a test runner
  // that understands the premature-exit-file protocol to report the
  // test as having failed.
  const internal::ScopedPrematureExitFile premature_exit_file(
      in_death_test_child_process
          ? nullptr
          : internal::posix::GetEnv("TEST_PREMATURE_EXIT_FILE"));

  // Captures the value of GTEST_FLAG(catch_exceptions).  This value will be
  // used for the duration of the program.
  impl()->set_catch_exceptions(GTEST_FLAG(catch_exceptions));

#if GTEST_OS_WINDOWS
  // Either the user wants Google Test to catch exceptions thrown by the
  // tests or this is executing in the context of death test child
  // process. In either case the user does not want to see pop-up dialogs
  // about crashes - they are expected.
  if (impl()->catch_exceptions() || in_death_test_child_process) {
# if !GTEST_OS_WINDOWS_MOBILE && !GTEST_OS_WINDOWS_PHONE && !GTEST_OS_WINDOWS_RT
    // SetErrorMode doesn't exist on CE.
    SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOALIGNMENTFAULTEXCEPT |
                 SEM_NOGPFAULTERRORBOX | SEM_NOOPENFILEERRORBOX);
# endif  // !GTEST_OS_WINDOWS_MOBILE

# if (defined(_MSC_VER) || GTEST_OS_WINDOWS_MINGW) && !GTEST_OS_WINDOWS_MOBILE
    // Death test children can be terminated with _abort().  On Windows,
    // _abort() can show a dialog with a warning message.  This forces the
    // abort message to go to stderr instead.
    _set_error_mode(_OUT_TO_STDERR);
# endif

# if defined(_MSC_VER) && !GTEST_OS_WINDOWS_MOBILE
    // In the debug version, Visual Studio pops up a separate dialog
    // offering a choice to debug the aborted program. We need to suppress
    // this dialog or it will pop up for every EXPECT/ASSERT_DEATH statement
    // executed. Google Test will notify the user of any unexpected
    // failure via stderr.
    if (!GTEST_FLAG(break_on_failure))
      _set_abort_behavior(
          0x0,                                    // Clear the following flags:
          _WRITE_ABORT_MSG | _CALL_REPORTFAULT);  // pop-up window, core dump.
# endif
  }
#endif  // GTEST_OS_WINDOWS

  return internal::HandleExceptionsInMethodIfSupported(
      impl(),
      &internal::UnitTestImpl::RunAllTests,
      "auxiliary test code (environments or event listeners)") ? 0 : 1;
}
```

`internal::HandleExceptionsInMethodIfSupported()` 是 Google test 定义的一个用于执行类的成员函数的函数，这里不再详细分析这个函数的实现。`UnitTest::Run()` 函数实际上通过 `googletest/googletest/src/gtest.cc` 中定义的 `internal::UnitTestImpl::RunAllTests()` 函数执行所有的测试用例：
```
// Runs all tests in this UnitTest object, prints the result, and
// returns true if all tests are successful.  If any exception is
// thrown during a test, the test is considered to be failed, but the
// rest of the tests will still be run.
//
// When parameterized tests are enabled, it expands and registers
// parameterized tests first in RegisterParameterizedTests().
// All other functions called from RunAllTests() may safely assume that
// parameterized tests are ready to be counted and run.
bool UnitTestImpl::RunAllTests() {
  // True iff Google Test is initialized before RUN_ALL_TESTS() is called.
  const bool gtest_is_initialized_before_run_all_tests = GTestIsInitialized();

  // Do not run any test if the --help flag was specified.
  if (g_help_flag)
    return true;

  // Repeats the call to the post-flag parsing initialization in case the
  // user didn't call InitGoogleTest.
  PostFlagParsingInit();

  // Even if sharding is not on, test runners may want to use the
  // GTEST_SHARD_STATUS_FILE to query whether the test supports the sharding
  // protocol.
  internal::WriteToShardStatusFileIfNeeded();

  // True iff we are in a subprocess for running a thread-safe-style
  // death test.
  bool in_subprocess_for_death_test = false;

#if GTEST_HAS_DEATH_TEST
  in_subprocess_for_death_test =
      (internal_run_death_test_flag_.get() != nullptr);
# if defined(GTEST_EXTRA_DEATH_TEST_CHILD_SETUP_)
  if (in_subprocess_for_death_test) {
    GTEST_EXTRA_DEATH_TEST_CHILD_SETUP_();
  }
# endif  // defined(GTEST_EXTRA_DEATH_TEST_CHILD_SETUP_)
#endif  // GTEST_HAS_DEATH_TEST

  const bool should_shard = ShouldShard(kTestTotalShards, kTestShardIndex,
                                        in_subprocess_for_death_test);

  // Compares the full test names with the filter to decide which
  // tests to run.
  const bool has_tests_to_run = FilterTests(should_shard
                                              ? HONOR_SHARDING_PROTOCOL
                                              : IGNORE_SHARDING_PROTOCOL) > 0;

  // Lists the tests and exits if the --gtest_list_tests flag was specified.
  if (GTEST_FLAG(list_tests)) {
    // This must be called *after* FilterTests() has been called.
    ListTestsMatchingFilter();
    return true;
  }

  random_seed_ = GTEST_FLAG(shuffle) ?
      GetRandomSeedFromFlag(GTEST_FLAG(random_seed)) : 0;

  // True iff at least one test has failed.
  bool failed = false;

  TestEventListener* repeater = listeners()->repeater();

  start_timestamp_ = GetTimeInMillis();
  repeater->OnTestProgramStart(*parent_);

  // How many times to repeat the tests?  We don't want to repeat them
  // when we are inside the subprocess of a death test.
  const int repeat = in_subprocess_for_death_test ? 1 : GTEST_FLAG(repeat);
  // Repeats forever if the repeat count is negative.
  const bool gtest_repeat_forever = repeat < 0;
  for (int i = 0; gtest_repeat_forever || i != repeat; i++) {
    // We want to preserve failures generated by ad-hoc test
    // assertions executed before RUN_ALL_TESTS().
    ClearNonAdHocTestResult();

    const TimeInMillis start = GetTimeInMillis();

    // Shuffles test suites and tests if requested.
    if (has_tests_to_run && GTEST_FLAG(shuffle)) {
      random()->Reseed(static_cast<UInt32>(random_seed_));
      // This should be done before calling OnTestIterationStart(),
      // such that a test event listener can see the actual test order
      // in the event.
      ShuffleTests();
    }

    // Tells the unit test event listeners that the tests are about to start.
    repeater->OnTestIterationStart(*parent_, i);

    // Runs each test suite if there is at least one test to run.
    if (has_tests_to_run) {
      // Sets up all environments beforehand.
      repeater->OnEnvironmentsSetUpStart(*parent_);
      ForEach(environments_, SetUpEnvironment);
      repeater->OnEnvironmentsSetUpEnd(*parent_);

      // Runs the tests only if there was no fatal failure or skip triggered
      // during global set-up.
      if (Test::IsSkipped()) {
        // Emit diagnostics when global set-up calls skip, as it will not be
        // emitted by default.
        TestResult& test_result =
            *internal::GetUnitTestImpl()->current_test_result();
        for (int j = 0; j < test_result.total_part_count(); ++j) {
          const TestPartResult& test_part_result =
              test_result.GetTestPartResult(j);
          if (test_part_result.type() == TestPartResult::kSkip) {
            const std::string& result = test_part_result.message();
            printf("%s\n", result.c_str());
          }
        }
        fflush(stdout);
      } else if (!Test::HasFatalFailure()) {
        for (int test_index = 0; test_index < total_test_suite_count();
             test_index++) {
          GetMutableSuiteCase(test_index)->Run();
        }
      }

      // Tears down all environments in reverse order afterwards.
      repeater->OnEnvironmentsTearDownStart(*parent_);
      std::for_each(environments_.rbegin(), environments_.rend(),
                    TearDownEnvironment);
      repeater->OnEnvironmentsTearDownEnd(*parent_);
    }

    elapsed_time_ = GetTimeInMillis() - start;

    // Tells the unit test event listener that the tests have just finished.
    repeater->OnTestIterationEnd(*parent_, i);

    // Gets the result and clears it.
    if (!Passed()) {
      failed = true;
    }

    // Restores the original test order after the iteration.  This
    // allows the user to quickly repro a failure that happens in the
    // N-th iteration without repeating the first (N - 1) iterations.
    // This is not enclosed in "if (GTEST_FLAG(shuffle)) { ... }", in
    // case the user somehow changes the value of the flag somewhere
    // (it's always safe to unshuffle the tests).
    UnshuffleTests();

    if (GTEST_FLAG(shuffle)) {
      // Picks a new random seed for each iteration.
      random_seed_ = GetNextRandomSeed(random_seed_);
    }
  }

  repeater->OnTestProgramEnd(*parent_);

  if (!gtest_is_initialized_before_run_all_tests) {
    ColoredPrintf(
        COLOR_RED,
        "\nIMPORTANT NOTICE - DO NOT IGNORE:\n"
        "This test program did NOT call " GTEST_INIT_GOOGLE_TEST_NAME_
        "() before calling RUN_ALL_TESTS(). This is INVALID. Soon " GTEST_NAME_
        " will start to enforce the valid usage. "
        "Please fix it ASAP, or IT WILL START TO FAIL.\n");  // NOLINT
#if GTEST_FOR_GOOGLE_
    ColoredPrintf(COLOR_RED,
                  "For more details, see http://wiki/Main/ValidGUnitMain.\n");
#endif  // GTEST_FOR_GOOGLE_
  }

  return !failed;
}
```

`internal::UnitTestImpl::RunAllTests()` 函数通过 `internal::UnitTestImpl::GetMutableSuiteCase()` 函数拿到测试套件 `TestSuite` 并执行其 `Run()` 函数：
```
void TestSuite::Run() {
  if (!should_run_) return;

  internal::UnitTestImpl* const impl = internal::GetUnitTestImpl();
  impl->set_current_test_suite(this);

  TestEventListener* repeater = UnitTest::GetInstance()->listeners().repeater();

  // Call both legacy and the new API
  repeater->OnTestSuiteStart(*this);
//  Legacy API is deprecated but still available
#ifndef GTEST_REMOVE_LEGACY_TEST_CASEAPI
  repeater->OnTestCaseStart(*this);
#endif  //  GTEST_REMOVE_LEGACY_TEST_CASEAPI

  impl->os_stack_trace_getter()->UponLeavingGTest();
  internal::HandleExceptionsInMethodIfSupported(
      this, &TestSuite::RunSetUpTestSuite, "SetUpTestSuite()");

  start_timestamp_ = internal::GetTimeInMillis();
  for (int i = 0; i < total_test_count(); i++) {
    GetMutableTestInfo(i)->Run();
  }
  elapsed_time_ = internal::GetTimeInMillis() - start_timestamp_;

  impl->os_stack_trace_getter()->UponLeavingGTest();
  internal::HandleExceptionsInMethodIfSupported(
      this, &TestSuite::RunTearDownTestSuite, "TearDownTestSuite()");

  // Call both legacy and the new API
  repeater->OnTestSuiteEnd(*this);
//  Legacy API is deprecated but still available
#ifndef GTEST_REMOVE_LEGACY_TEST_CASEAPI
  repeater->OnTestCaseEnd(*this);
#endif  //  GTEST_REMOVE_LEGACY_TEST_CASEAPI

  impl->set_current_test_suite(nullptr);
}
```
`TestSuite::Run()` 函数通过 `GetMutableTestInfo()` 函数获得 `TestInfo` 并执行其 `Run()` 函数，`TestInfo::Run()` 函数定义如下：
```
// Creates the test object, runs it, records its result, and then
// deletes it.
void TestInfo::Run() {
  if (!should_run_) return;

  // Tells UnitTest where to store test result.
  internal::UnitTestImpl* const impl = internal::GetUnitTestImpl();
  impl->set_current_test_info(this);

  TestEventListener* repeater = UnitTest::GetInstance()->listeners().repeater();

  // Notifies the unit test event listeners that a test is about to start.
  repeater->OnTestStart(*this);

  const TimeInMillis start = internal::GetTimeInMillis();

  impl->os_stack_trace_getter()->UponLeavingGTest();

  // Creates the test object.
  Test* const test = internal::HandleExceptionsInMethodIfSupported(
      factory_, &internal::TestFactoryBase::CreateTest,
      "the test fixture's constructor");

  // Runs the test if the constructor didn't generate a fatal failure or invoke
  // GTEST_SKIP().
  // Note that the object will not be null
  if (!Test::HasFatalFailure() && !Test::IsSkipped()) {
    // This doesn't throw as all user code that can throw are wrapped into
    // exception handling code.
    test->Run();
  }

  if (test != nullptr) {
    // Deletes the test object.
    impl->os_stack_trace_getter()->UponLeavingGTest();
    internal::HandleExceptionsInMethodIfSupported(
        test, &Test::DeleteSelf_, "the test fixture's destructor");
  }

  result_.set_start_timestamp(start);
  result_.set_elapsed_time(internal::GetTimeInMillis() - start);

  // Notifies the unit test event listener that a test has just finished.
  repeater->OnTestEnd(*this);

  // Tells UnitTest to stop associating assertion results to this
  // test.
  impl->set_current_test_info(nullptr);
}
```

`TestInfo::Run()` 函数创建测试用例类对象，并执行其 `Run()` 函数。`Test::Run()` 执行测试用例定义的 `SetUp()`，测试用例主体 `TestBody()` 函数，和 `TearDown()` 函数：
```
// Runs the test and updates the test result.
void Test::Run() {
  if (!HasSameFixtureClass()) return;

  internal::UnitTestImpl* const impl = internal::GetUnitTestImpl();
  impl->os_stack_trace_getter()->UponLeavingGTest();
  internal::HandleExceptionsInMethodIfSupported(this, &Test::SetUp, "SetUp()");
  // We will run the test only if SetUp() was successful and didn't call
  // GTEST_SKIP().
  if (!HasFatalFailure() && !IsSkipped()) {
    impl->os_stack_trace_getter()->UponLeavingGTest();
    internal::HandleExceptionsInMethodIfSupported(
        this, &Test::TestBody, "the test body");
  }

  // However, we want to clean up as much as possible.  Hence we will
  // always call TearDown(), even if SetUp() or the test body has
  // failed.
  impl->os_stack_trace_getter()->UponLeavingGTest();
  internal::HandleExceptionsInMethodIfSupported(
      this, &Test::TearDown, "TearDown()");
}
```

测试用例的调用执行过程大概是这样的：

`RUN_ALL_TESTS()` -> `UnitTest::Run()` -> `UnitTestImpl::RunAllTests()` -> `TestSuite::Run()` -> `TestInfo::Run()` -> `Test::Run()` -> `Test::TestBody()`。

Done。
