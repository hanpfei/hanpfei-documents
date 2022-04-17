---
title: 在 Mac 上为 Android 编译 WebRTC
date: 2022-04-10 23:05:49
categories: 音视频开发
tags:
- 音视频开发
---

在 Mac 上为 Android 编译 WebRTC 的基本流程和在任意平台上编译任何其它目标平台的 WebRTC 大体一致，但在 Mac 上为 Android 编译 WebRTC 不是 WebRTC 官方正式支持的 WebRTC 的构建方式，因而需要针对这种构建方式，对 WebRTC 构建系统中的部分配置和工具做一些适当的调整。本文将描述在 Mac 上为 Android 编译 WebRTC 的流程，流程中各个步骤将会遇到的问题，问题发生原因的简单解释，以及问题的解决方法。
<!--more-->
在 Mac 上为 Android 编译 WebRTC 的过程如下。

## 1. 创建 `args.gn` 文件
为 Android 版 WebRTC 的构建创建 `args.gn` 文件做一些全局的构建配置开关，如：
```
is_component_build = false
rtc_build_tools = false
rtc_include_tests = false
treat_warnings_as_errors = false
use_rtti = false
use_dummy_lastchange = true
is_clang = true
target_cpu = "arm64"
is_debug = true
symbol_level = 2
clang_base_path = "/Users/lisi/Projects/opensource/OpenRTCClient/build_system/llvm-build/mac/mac/Release+Asserts"
android_sdk_build_tools_version = "30.0.1"
target_os = "android"
android_ndk_root = "/Users/lisi/Library/Android/sdk/ndk/23.1.7779620"
android_sdk_root = "/Users/lisi/Library/Android/sdk"
clang_use_chrome_plugins = false
lint_android_sdk_root = "/Users/lisi/Library/Android/sdk"
use_custom_libcxx = false
use_sysroot = false
enable_libaom = false
enable_libaom_decoder = false
rtc_use_h264 = true
rtc_enable_protobuf = true
rtc_include_ilbc = false
rtc_libvpx_build_vp9 = true
ffmpeg_branding = "Chrome"
```

## 2. `gn gen` 生成构建配置文件

通过 `gn gen` 根据创建的 `args.gn` 文件，各个 `BUILD.gn` 和 `*.gni` 构建配置文件，生成 `ninja` 构建所需的配置文件。假设 `args.gn` 文件位于 `build/android/arm64/debug` 目录下，执行如下命令：
```
OpenRTCClient % gn gen build/android/arm64/debug
```

这里就报了错，报错信息如下：
```
OpenRTCClient % gn gen build/android/arm64/debug
ERROR Unresolved dependencies.
//examples/androidapp/third_party/autobanh:autobanh_java__header(//build/toolchain/android:android_clang_arm64)
  needs //third_party/ijar:ijar(//build/toolchain/mac:clang_x64)
//third_party/accessibility_test_framework:accessibility_test_framework_java__header(//build/toolchain/android:android_clang_arm64)
  needs //third_party/ijar:ijar(//build/toolchain/mac:clang_x64)
//third_party/android_deps:android_arch_core_common_java__header(//build/toolchain/android:android_clang_arm64)
  needs //third_party/ijar:ijar(//build/toolchain/mac:clang_x64)
//third_party/android_deps:android_arch_core_runtime_java__classes__header(//build/toolchain/android:android_clang_arm64)
  needs //third_party/ijar:ijar(//build/toolchain/mac:clang_x64)
//third_party/android_deps:android_arch_lifecycle_common_java8_java__header(//build/toolchain/android:android_clang_arm64)
  needs //third_party/ijar:ijar(//build/toolchain/mac:clang_x64)
//third_party/android_deps:android_arch_lifecycle_common_java__header(//build/toolchain/android:android_clang_arm64)
  needs //third_party/ijar:ijar(//build/toolchain/mac:clang_x64)
. . . . . .
```

报错信息提示说，找不到被广泛依赖的一个构建目标 `//third_party/ijar:ijar`。

通过这段报错信息，我们不难了解到，这个构建目标应该是定义在 `webrtc/third_party/ijar/BUILD.gn` 文件中的。我们打开这个文件，来看下这个构建目标的定义：
```
if (is_linux || is_chromeos) {
  config("ijar_compiler_flags") {
    if (is_clang) {
      cflags = [
        "-Wno-shadow",
        "-Wno-unused-but-set-variable",
      ]
    }
  }

  executable("ijar") {
    sources = [
      "classfile.cc",
      "common.h",
      "ijar.cc",
      "mapped_file.h",
      "mapped_file_unix.cc",
      "platform_utils.cc",
      "platform_utils.h",
      "zip.cc",
      "zip.h",
      "zlib_client.cc",
      "zlib_client.h",
    ]

    deps = [ "//third_party/zlib" ]

    configs += [ ":ijar_compiler_flags" ]

    # Always build release since this is a build tool.
    if (is_debug) {
      configs -= [ "//build/config:debug" ]
      configs += [ "//build/config:release" ]
    }
  }
}
```

通过该文件，可以看到，只有在 Linux 和 ChromeOS 上，才会定义构建目标 `//third_party/ijar:ijar`。我们修改上面的 `if` 判断里的条件，同样为 Mac 定义这个构建目标。具体来说，是将如下的条件判断：
```
if (is_linux || is_chromeos) {
```

修改为：
```
if (is_linux || is_chromeos || is_mac) {
```

`webrtc/third_party/ijar/BUILD.gn` 这个文件中的其它内容保持不变。

这样我们可以成功执行 `gn gen` 了：
```
OpenRTCClient % gn gen build/android/arm64/debug
Done. Made 5530 targets from 318 files in 2743ms
```

## 3. 执行 `ninja` 命令构建 WebRTC

#### (1). `-ffile-compilation-dir=.` 导致的编译错误

执行 `ninja` 命令构建 WebRTC：
```
OpenRTCClient % ninja -C build/android/arm64/debug
```

还没编译几行代码，就报了个编译错误：
```
OpenRTCClient % ninja -C build/android/arm64/debug
ninja: Entering directory `/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug'
[1/5234] CC obj/base/third_party/libevent/libevent/event_tagging.o
FAILED: obj/base/third_party/libevent/libevent/event_tagging.o 
../../../../../../../Library/Android/sdk/ndk/23.1.7779620/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang -MMD -MF obj/base/third_party/libevent/libevent/event_tagging.o.d -DHAVE_CONFIG_H -D_GNU_SOURCE -DANDROID -DHAVE_SYS_UIO_H -DANDROID_NDK_VERSION_ROLL=r22_1 -D_DEBUG -DDYNAMIC_ANNOTATIONS_ENABLED=1 -I../../../../webrtc/base/third_party/libevent/android -I../../../../webrtc -Igen -fno-delete-null-pointer-checks -fno-ident -fno-strict-aliasing --param=ssp-buffer-size=4 -fstack-protector -funwind-tables -fPIC -fcolor-diagnostics -fmerge-all-constants -fcrash-diagnostics-dir=../../../../webrtc/tools/clang/crashreports -mllvm -instcombine-lower-dbg-declare=0 -ffp-contract=off -ffunction-sections -fno-short-enums --target=aarch64-linux-android21 -mno-outline-atomics -mno-outline -Wno-builtin-macro-redefined -D__DATE__= -D__TIME__= -D__TIMESTAMP__= -ffile-compilation-dir=. -no-canonical-prefixes -Oz -fdata-sections -ffunction-sections -fno-unique-section-names -fno-omit-frame-pointer -g2 -gdwarf-aranges -ggnu-pubnames -Xclang -fuse-ctor-homing -fvisibility=hidden -Wheader-hygiene -Wstring-conversion -Wtautological-overlap-compare -Wall -Wno-unused-variable -Wno-c++11-narrowing -Wno-unused-but-set-variable -Wno-misleading-indentation -Wno-missing-field-initializers -Wno-unused-parameter -Wloop-analysis -Wno-unneeded-internal-declaration -Wenum-compare-conditional -Wno-psabi -Wno-ignored-pragma-optimize -Wmax-tokens -std=c11 --sysroot=../../../../../../../Library/Android/sdk/ndk/23.1.7779620/toolchains/llvm/prebuilt/darwin-x86_64/sysroot -c ../../../../webrtc/base/third_party/libevent/event_tagging.c -o obj/base/third_party/libevent/libevent/event_tagging.o
clang: error: unknown argument: '-ffile-compilation-dir=.'
[2/5234] CC obj/base/third_party/libevent/libevent/signal.o
. . . . . .
```

这段报错信息是说，`clang` 无法识别一个编译选项 `-ffile-compilation-dir=.`。在 stack overflow 上有个帖子在讨论这个问题，[Can't build Webrtc with Bitcode enabled](https://stackoverflow.com/questions/68498739/cant-build-webrtc-with-bitcode-enabled)。出现这个问题，是因为编译 WebRTC 用的 `clang` 版本一般都比较新，WebRTC 的构建系统为编译设置了较新的 `clang` 版本才支持的一个编译选项，但本地编译时使用的 `clang` 版本不够新，于是出现了编译工具无法识别编译选项的问题。

这个编译选项是在 `webrtc/build/config/compiler/BUILD.gn` 文件中加的：
```
  if (is_clang && strip_absolute_paths_from_debug_symbols) {
    # If debug option is given, clang includes $cwd in debug info by default.
    # For such build, this flag generates reproducible obj files even we use
    # different build directory like "out/feature_a" and "out/feature_b" if
    # we build same files with same compile flag.
    # Other paths are already given in relative, no need to normalize them.
    if (is_nacl) {
      # TODO(https://crbug.com/1231236): Use -ffile-compilation-dir= here.
      cflags += [
        "-Xclang",
        "-fdebug-compilation-dir",
        "-Xclang",
        ".",
      ]
    } else {
      # -ffile-compilation-dir is an alias for both -fdebug-compilation-dir=
      # and -fcoverage-compilation-dir=.
      cflags += [ "-ffile-compilation-dir=." ]
    }
    if (!is_win) {
      # We don't use clang -cc1as on Windows (yet? https://crbug.com/762167)
      asmflags = [ "-Wa,-fdebug-compilation-dir,." ]
    }
```

这里我们将这个编译选项替换掉来解决这个问题。具体来说，是将如下这行代码：
```
      cflags += [ "-ffile-compilation-dir=." ]
```

替换为：
```
      cflags += [ "-fdebug-compilation-dir" ]
```

修改完成后，重新执行 `gn gen` 和 `ninja` 命令。

#### (2). `jni_generator.py` 执行失败导致的错误

随后遇到如下编译报错：
```
[1109/5146] ACTION //sdk/android:generated_external_classes_jni(//build/toolchain/android:android_clang_arm64)
FAILED: gen/sdk/android/generated_external_classes_jni/Integer_jni.h gen/sdk/android/generated_external_classes_jni/Double_jni.h gen/sdk/android/generated_external_classes_jni/Long_jni.h gen/sdk/android/generated_external_classes_jni/Iterable_jni.h gen/sdk/android/generated_external_classes_jni/Iterator_jni.h gen/sdk/android/generated_external_classes_jni/Boolean_jni.h gen/sdk/android/generated_external_classes_jni/BigInteger_jni.h gen/sdk/android/generated_external_classes_jni/Map_jni.h gen/sdk/android/generated_external_classes_jni/LinkedHashMap_jni.h gen/sdk/android/generated_external_classes_jni/ArrayList_jni.h gen/sdk/android/generated_external_classes_jni/Enum_jni.h 
python3 ../../../../webrtc/base/android/jni_generator/jni_generator.py --ptr_type=long --includes ../../../../../../../../webrtc/sdk/android/src/jni/jni_generator_helper.h --jar_file ../../../../../../../Library/Android/sdk/platforms/android-31/android.jar --output_file gen/sdk/android/generated_external_classes_jni/Integer_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Double_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Long_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Iterable_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Iterator_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Boolean_jni.h --output_file gen/sdk/android/generated_external_classes_jni/BigInteger_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Map_jni.h --output_file gen/sdk/android/generated_external_classes_jni/LinkedHashMap_jni.h --output_file gen/sdk/android/generated_external_classes_jni/ArrayList_jni.h --output_file gen/sdk/android/generated_external_classes_jni/Enum_jni.h --input_file=java/lang/Integer.class --input_file=java/lang/Double.class --input_file=java/lang/Long.class --input_file=java/lang/Iterable.class --input_file=java/util/Iterator.class --input_file=java/lang/Boolean.class --input_file=java/math/BigInteger.class --input_file=java/util/Map.class --input_file=java/util/LinkedHashMap.class --input_file=java/util/ArrayList.class --input_file=java/lang/Enum.class
Traceback (most recent call last):
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/base/android/jni_generator/jni_generator.py", line 1630, in <module>
    sys.exit(main())
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/base/android/jni_generator/jni_generator.py", line 1624, in main
    GenerateJNIHeader(java_path, header_path, args)
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/base/android/jni_generator/jni_generator.py", line 1485, in GenerateJNIHeader
    jni_from_javap = JNIFromJavaP.CreateFromClass(input_file, options)
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/base/android/jni_generator/jni_generator.py", line 847, in CreateFromClass
    jni_from_javap = JNIFromJavaP(stdout.split('\n'), options)
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/base/android/jni_generator/jni_generator.py", line 767, in __init__
    self.fully_qualified_class = self.fully_qualified_class.replace('.', '/')
AttributeError: 'JNIFromJavaP' object has no attribute 'fully_qualified_class'
[1110/5146] CXX obj/third_party/abseil-cpp/absl/base/spinlock_wait/spinlock_wait.o
warning: unknown warning option '-Wno-unused-but-set-variable'; did you mean '-Wno-unused-const-variable'? [-Wunknown-warning-option]
```

这次是构建系统在执行脚本文件 `webrtc/base/android/jni_generator/jni_generator.py` 生成 JNI 头文件时出错。根据出错的调用堆栈信息，捞出来相关的主要的代码如下：
```
class JNIFromJavaP(object):
  """Uses 'javap' to parse a .class file and generate the JNI header file."""

  def __init__(self, contents, options):
    self.contents = contents
    self.namespace = options.namespace
    for line in contents:
      class_name = re.match(
          '.*?(public).*?(class|interface) (?P<class_name>\S+?)( |\Z)', line)
      if class_name:
        self.fully_qualified_class = class_name.group('class_name')
        break
    self.fully_qualified_class = self.fully_qualified_class.replace('.', '/')
    # Java 7's javap includes type parameters in output, like HashSet<T>. Strip
    # away the <...> and use the raw class name that Java 6 would've given us.
. . . . . .
  @staticmethod
  def CreateFromClass(class_file, options):
    class_name = os.path.splitext(os.path.basename(class_file))[0]
    javap_path = os.path.abspath(options.javap)
    p = subprocess.Popen(
        args=[javap_path, '-c', '-verbose', '-s', class_name],
        cwd=os.path.dirname(class_file),
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        universal_newlines=True)
    stdout, _ = p.communicate()
    jni_from_javap = JNIFromJavaP(stdout.split('\n'), options)
    return jni_from_javap
. . . . . .
def GenerateJNIHeader(input_file, output_file, options):
  try:
    if os.path.splitext(input_file)[1] == '.class':
      jni_from_javap = JNIFromJavaP.CreateFromClass(input_file, options)
      content = jni_from_javap.GetContent()
    else:
      jni_from_java_source = JNIFromJavaSource.CreateFromFile(
          input_file, options)
      content = jni_from_java_source.GetContent()
  except ParseError as e:
    print(e)
    sys.exit(1)
  if output_file:
    with build_utils.AtomicOutput(output_file, mode='w') as f:
      f.write(content)
  else:
    print(content)
```

相关代码的大致意思是，在 `GenerateJNIHeader()` 函数中调用 `JNIFromJavaP.CreateFromClass(input_file, options)` 方法；在 `JNIFromJavaP.CreateFromClass(input_file, options)` 方法中执行 `javap` 命令，得到命令执行的标准输出的内容，并以此内容为参数创建 `JNIFromJavaP`，在类 `JNIFromJavaP` 构造函数中出错。

我们打印 `JNIFromJavaP` 构造函数接收的 `contents` 参数，可以看到这个参数是空字符串，显然不符合预期。我们提取 `JNIFromJavaP.CreateFromClass(input_file, options)` 方法中执行的 `javap` 命令，这个命令大概像这样：
```
/Library/Java/JavaVirtualMachines/jdk-11.0.13.jdk/Contents/Home/bin/javap  -c  -verbose  -s  Integer 
```

我们在终端命令行中执行这个命令，这个命令执行会报错：
```
OpenRTCClient % /Library/Java/JavaVirtualMachines/jdk-11.0.13.jdk/Contents/Home/bin/javap  -c  -verbose  -s  Integer  
错误: 找不到类: Integer
```

来看一下给 `javap` 传的两个参数是干什么的：
```
OpenRTCClient % javap --help                                                                                                  
用法: javap <options> <classes>
其中, 可能的选项包括:
  -? -h --help -help               输出此帮助消息
  -version                         版本信息
  -v  -verbose                     输出附加信息
  -l                               输出行号和本地变量表
  -public                          仅显示公共类和成员
  -protected                       显示受保护的/公共类和成员
  -package                         显示程序包/受保护的/公共类
                                   和成员 (默认)
  -p  -private                     显示所有类和成员
  -c                               对代码进行反汇编
  -s                               输出内部类型签名
  -sysinfo                         显示正在处理的类的
                                   系统信息 (路径, 大小, 日期, MD5 散列)
  -constants                       显示最终常量
  --module <模块>, -m <模块>       指定包含要反汇编的类的模块
  --module-path <路径>             指定查找应用程序模块的位置
  --system <jdk>                   指定查找系统模块的位置
  --class-path <路径>              指定查找用户类文件的位置
  -classpath <路径>                指定查找用户类文件的位置
  -cp <路径>                       指定查找用户类文件的位置
  -bootclasspath <路径>            覆盖引导类文件的位置

GNU 样式的选项可使用 = (而非空白) 来分隔选项名称
及其值。

每个类可由其文件名, URL 或其
全限定类名指定。示例:
   path/to/MyClass.class
   jar:file:///path/to/MyJar.jar!/mypkg/MyClass.class
   java.lang.Object
```

`-c` 参数用于对代码进行反汇编，`-s` 参数用于输出内部类型签名。不过这里出错的关键不是参数，而是指定类的方式。`javap` 命令接受的指定类的方式如上面的 help 输出中最下面的说明。Mac 平台 `javap` 命令接受的指定类的方式与 Linux 平台的 `javap` 命令接受的不太一样。

WebRTC 的构建系统会通过 `webrtc/base/android/jni_generator/jni_generator.py` 为几个标准库的类生成 JNI 头文件，这些类的类文件打包在 `rt.jar` 文件中。我们从 `rt.jar` 文件中提取所有的类文件，并对类文件执行 `javap` 命令：
```
opensource % mkdir jar_classes
opensource % cd jar_classes
jar_classes % jar xf /Library/Java/JavaVirtualMachines/jdk1.8.0_311.jdk/Contents/Home/jre/lib/rt.jar
jar_classes % cd java/lang
lang % javap -c -s Integer
错误: 找不到类: Integer
lang % javap -c -s Integer.class
Compiled from "Integer.java"
public final class java.lang.Integer extends java.lang.Number implements java.lang.Comparable<java.lang.Integer> {
  public static final int MIN_VALUE;
    descriptor: I

  public static final int MAX_VALUE;
```

在类文件所在的目录，我们仅通过传入类名类指定类时，`javap` 执行失败。而通过 `类名.class` 的形式指定类时，则可以执行成功。在 Linux 平台上，如果位于类文件所在的目录，仅通过传入类名来指定类，而没有 `.class` 后缀名，`javap` 也可以执行成功。

对于上面例子中的 `Integer`，在 Mac 平台上执行 `javap` 时，正确的参数格式有多种，可以是 `java.lang.Integer`，可以是类文件的完整文件路径，如 `/var/folders/1x/5vtfxt410vbf9xhk8vvfz8l40000gn/T/tmp2gbxloem/java/lang/Integer.class`，也可以是类文件的文件名 `Integer.class`，但不能是 `Integer`。

由此，也就可以得到我们对这个问题的解决方案了。

方案一，通过类文件的完整文件路径来给 `javap` 指定类，修改 `JNIFromJavaP.CreateFromClass(input_file, options)` 方法的实现为：
```
  @staticmethod
  def CreateFromClass(class_file, options):
    if sys.platform == "darwin":
      class_name = class_file
    else:
      class_name = os.path.splitext(os.path.basename(class_file))[0]

    javap_path = os.path.abspath(options.javap)
    p = subprocess.Popen(
        args=[javap_path, '-c', '-verbose', '-s', class_name],
        cwd=os.path.dirname(class_file),
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        universal_newlines=True)
    stdout, _ = p.communicate()
    jni_from_javap = JNIFromJavaP(stdout.split('\n'), options)
    return jni_from_javap
```

方案二，由于创建进程执行命令时，设置了工作目录，我们还可以通过类文件的文件名来给 `javap` 指定类，这修改 `JNIFromJavaP.CreateFromClass(input_file, options)` 方法的实现为：
```
  @staticmethod
  def CreateFromClass(class_file, options):
    if sys.platform == "darwin":
      class_name = os.path.basename(class_file)
    else:
      class_name = os.path.splitext(os.path.basename(class_file))[0]

    javap_path = os.path.abspath(options.javap)
    p = subprocess.Popen(
        args=[javap_path, '-c', '-verbose', '-s', class_name],
        cwd=os.path.dirname(class_file),
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        universal_newlines=True)
    stdout, _ = p.communicate()
    jni_from_javap = JNIFromJavaP(stdout.split('\n'), options)
    return jni_from_javap
```

方案三，通过类路径来给 `javap` 指定类，这修改 `JNIFromJavaP.CreateFromClass(input_file, options)` 方法的实现为：
```
  @staticmethod
  def CreateFromClass(class_file, options):
    if sys.platform == "darwin":
      path_items = class_file.split("/")
      i = len(path_items) - 2
      while i >= 0:
        class_name = path_items[i] + "." + class_name
        if path_items[i] == "java":
          break
        i = i -1
    else:
      class_name = os.path.splitext(os.path.basename(class_file))[0]

    javap_path = os.path.abspath(options.javap)
    p = subprocess.Popen(
        args=[javap_path, '-c', '-verbose', '-s', class_name],
        cwd=os.path.dirname(class_file),
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        universal_newlines=True)
    stdout, _ = p.communicate()
    jni_from_javap = JNIFromJavaP(stdout.split('\n'), options)
    return jni_from_javap
```

方案三的这种实现，对于要生成 JNI 的类有一定的限制。

相对来说，综合考虑，方案二无疑是最好的。

#### (3). Android SDK 工具缺失或版本问题导致的编译错误

再次执行 `ninja`，遇到如下编译报错：
```
OpenRTCClient % mv  ~/Library/Android/sdk/cmdline-tools/latest  ~/Library/Android/sdk/cmdline-tools/latest_bak
OpenRTCClient % webrtc_build build android arm64 debug                                                        
/Users/lisi/Projects/opensource/OpenRTCClient/build_system/tools/bin/mac/ninja -C /Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug   
ninja: Entering directory `/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug'
[355/4574] ACTION //examples:AppRTCMobile_test_apk__test_apk__merge_manifests(//build/toolchain/android:android_clang_arm64)
FAILED: gen/examples/AppRTCMobile_test_apk__test_apk_manifest/AndroidManifest.xml 
python3 ../../../../webrtc/build/android/gyp/merge_manifest.py --depfile gen/examples/AppRTCMobile_test_apk__test_apk__merge_manifests.d --android-sdk-cmdline-tools ../../../../../../../Library/Android/sdk/cmdline-tools/latest --root-manifest ../../../../webrtc/examples/androidtests/AndroidManifest.xml --output gen/examples/AppRTCMobile_test_apk__test_apk_manifest/AndroidManifest.xml --extras @FileArg\(gen/examples/AppRTCMobile_test_apk__test_apk.build_config.json:extra_android_manifests\) --min-sdk-version=21 --target-sdk-version=21
Traceback (most recent call last):
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/merge_manifest.py", line 149, in <module>
    main(sys.argv[1:])
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/merge_manifest.py", line 129, in main
    build_utils.CheckOutput(
  File "/Users/lisi/Projects/opensource/OpenRTCClient/webrtc/build/android/gyp/util/build_utils.py", line 276, in CheckOutput
    raise CalledProcessError(cwd, args, stdout + stderr)
util.build_utils.CalledProcessError: Command failed: ( cd /Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug; /Library/Java/JavaVirtualMachines/jdk-11.0.13.jdk/Contents/Home/bin/java -Xmx1G -noverify -cp ../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/build-system/manifest-merger.jar:../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/common/common.jar:../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/sdk-common/sdk-common.jar:../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/sdklib/sdklib.jar:../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/external/com/google/guava/guava/30.1-jre/guava-30.1-jre.jar:../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/external/kotlin-plugin-ij/Kotlin/kotlinc/lib/kotlin-stdlib.jar:../../../../../../../Library/Android/sdk/cmdline-tools/latest/lib/external/com/google/code/gson/gson/2.8.6/gson-2.8.6.jar com.android.manifmerger.Merger --out /Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/gen/examples/AppRTCMobile_test_apk__test_apk_manifest/tmpvg3qg_k3AndroidManifest.xml --property MIN_SDK_VERSION=21 --property TARGET_SDK_VERSION=21 --libs gen/third_party/android_sdk/android_test_base_java/AndroidManifest.xml:gen/third_party/android_sdk/android_test_runner_java/AndroidManifest.xml --main /var/folders/1x/5vtfxt410vbf9xhk8vvfz8l40000gn/T/AndroidManifest.xmlwu610lv1 --property PACKAGE=org.appspot.apprtc.test --remove-tools-declarations )
错误: 找不到或无法加载主类 com.android.manifmerger.Merger
原因: java.lang.ClassNotFoundException: com.android.manifmerger.Merger
```

这个报错是因为，编译 Android 需要生成一些测试 APK 文件，此时需要执行 Android SDK 中的一些 Java 工具，这里提示找不到的主类 `com.android.manifmerger.Merger`，其定义在 `sdk/cmdline-tools/latest/lib/build-system/manifest-merger.jar` 这个 jar 文件中，但在我们的 SDK 中，没有 `sdk/cmdline-tools/latest/` 这个目录。

不难看出 `sdk/cmdline-tools/latest/` 是要引用最新版本的 `cmdline-tools`，但我们的 SDK 包中，只安装了 `3.0` 版。

由此，可以有很多种方案来解决这个问题。

方案一，创建一个符号链接 `sdk/cmdline-tools/latest` 指向 `sdk/cmdline-tools/3.0`，如：
```
tmp % ln -s ~/Library/Android/sdk/cmdline-tools/3.0  ~/Library/Android/sdk/cmdline-tools/lastest
```

方案二，安装最新版的 `cmdline-tools`，既可以通过 Android Studio 来完成，也可以通过 Android SDK 提供的命令行工具来做，如：
```
~/Library/Android/sdk/tools/bin/sdkmanager --install "cmdline-tools;latest"
```

如果发现安装了最新版的 `cmdline-tools` 还是找不到主类 `com.android.manifmerger.Merger`，则可以看下 `cmdline-tools` 中是否有 `~/Library/Android/sdk/cmdline-tools/latest/lib/build-system/manifest-merger.jar` 这个 jar 文件：
```
OpenRTCClient % ls -l ~/Library/Android/sdk/cmdline-tools/latest/lib/build-system/ 
total 424
drwxr-xr-x  3 lisi  staff      96  4 11 14:21 builder-model
drwxr-xr-x  3 lisi  staff      96  4 11 14:21 builder-test-api
-rwxr-xr-x  1 lisi  staff  213592  4 11 14:21 tools.manifest-merger.jar
```

如果是这种情况，可以拷贝 `tools.manifest-merger.jar` 或者为它创建符号链接，如：
```
OpenRTCClient % cp ~/Library/Android/sdk/cmdline-tools/latest/lib/build-system/tools.manifest-merger.jar ~/Library/Android/sdk/cmdline-tools/latest/lib/build-system/manifest-merger.jar
```

在上面的报错信息中，我们可以看到引用了多个 jar 文件，对于其中的每一个 jar 文件，我们都需要以类似这样的方式确保它们的存在。

方案三，修改 WebRTC 的构建配置。
这里的报错，是在执行 `webrtc/build/config/android/internal_rules.gni` 中定义的  `template("merge_manifests")` 时报出的，`cmdline-tools` 的路径是在这里定义的，如：
```
  template("merge_manifests") {
    action_with_pydeps(target_name) {
      forward_variables_from(invoker, TESTONLY_AND_VISIBILITY + [ "deps" ])
      script = "//build/android/gyp/merge_manifest.py"
      depfile = "$target_gen_dir/$target_name.d"

      inputs = [
        invoker.build_config,
        invoker.input_manifest,
      ]

      outputs = [ invoker.output_manifest ]
      _rebased_build_config = rebase_path(invoker.build_config, root_build_dir)

      args = [
        "--depfile",
        rebase_path(depfile, root_build_dir),
        "--android-sdk-cmdline-tools",
        rebase_path("${android_sdk_root}/cmdline-tools/latest",
                    root_build_dir),
        "--root-manifest",
        rebase_path(invoker.input_manifest, root_build_dir),
        "--output",
        rebase_path(invoker.output_manifest, root_build_dir),
        "--extras",
        "@FileArg($_rebased_build_config:extra_android_manifests)",
        "--min-sdk-version=${invoker.min_sdk_version}",
        "--target-sdk-version=${invoker.target_sdk_version}",
      ]
```

可以将这里的 `cmdline-tools` 的路径修改为我们本地已经安装的版本的路径。

#### (4). Clang 缺少 Android NDK 包的库或工具导致编译失败

再次执行 `ninja`，遇到如下编译报错：
```
OpenRTCClient % webrtc_build build android arm64 debug
/Users/lisi/Projects/opensource/OpenRTCClient/build_system/tools/bin/mac/ninja -C /Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug   
ninja: Entering directory `/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug'
[14/1093] LINK ./stun_prober
FAILED: stun_prober exe.unstripped/stun_prober 
python3 "../../../../webrtc/build/toolchain/gcc_link_wrapper.py" --output="./stun_prober" --strip="../../../../build_system/llvm-build/mac/mac/Release+Asserts/bin/llvm-strip" --unstripped-file="./exe.unstripped/stun_prober" -- ../../../../build_system/llvm-build/mac/mac/Release+Asserts/bin/clang++ -fuse-ld=lld -Wl,--fatal-warnings -Wl,--build-id -fPIC -Wl,-z,noexecstack -Wl,-z,relro -Wl,-z,now -Wl,-z,max-page-size=4096 -Wl,--color-diagnostics -Wl,--no-rosegment -Wl,--no-call-graph-profile-sort -Wl,--exclude-libs=libvpx_assembly_arm.a --unwindlib=none --target=aarch64-linux-android21 -Wl,-mllvm,-enable-machine-outliner=never -no-canonical-prefixes -Wl,--gc-sections -Wl,--gdb-index --sysroot=../../../../../../../Library/Android/sdk/ndk/23.1.7779620/toolchains/llvm/prebuilt/darwin-x86_64/sysroot -Wl,--warn-shared-textrel -Wl,-z,defs -Wl,--as-needed -pie -Bdynamic -Wl,-z,nocopyreloc -Wl,--dynamic-linker,/system/bin/linker64 -o "./exe.unstripped/stun_prober" -Wl,--start-group @"./stun_prober.rsp"  -Wl,--end-group  -ldl -lm -llog -lGLESv2 -lOpenSLES
ld.lld: error: cannot open ../../../../build_system/llvm-build/mac/mac/Release+Asserts/lib/clang/14.0.0/lib/linux/libclang_rt.builtins-aarch64-android.a: No such file or directory
ld.lld: error: cannot open ../../../../build_system/llvm-build/mac/mac/Release+Asserts/lib/clang/14.0.0/lib/linux/libclang_rt.builtins-aarch64-android.a: No such file or directory
clang++: error: linker command failed with exit code 1 (use -v to see invocation)
```

这是由于 webrtc 中的 LLVM 工具链中缺少编译链接 Android 二进制文件所需的一些库文件。这可以通过将 NDK 中的库文件拷贝的 LLVM 工具链的对应目录下来解决，如：
```
android-ndk-r23b-darwin % cp -r  toolchains/llvm/prebuilt/darwin-x86_64//lib64/clang/12.0.8/lib/linux  ~/Projects/opensource/OpenRTCClient/build_system/llvm-build/mac/mac/Release+Asserts/lib/clang/14.0.0/lib/
```

再次执行 `ninja`，遇到如下编译报错：
```
OpenRTCClient % webrtc_build build android arm64 debug
/Users/lisi/Projects/opensource/OpenRTCClient/build_system/tools/bin/mac/ninja -C /Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug   
ninja: Entering directory `/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug'
[33/880] LINK ./stun_prober
FAILED: stun_prober exe.unstripped/stun_prober 
python3 "../../../../webrtc/build/toolchain/gcc_link_wrapper.py" --output="./stun_prober" --strip="../../../../build_system/llvm-build/mac/mac/Release+Asserts/bin/llvm-strip" --unstripped-file="./exe.unstripped/stun_prober" -- ../../../../build_system/llvm-build/mac/mac/Release+Asserts/bin/clang++ -fuse-ld=lld -Wl,--fatal-warnings -Wl,--build-id -fPIC -Wl,-z,noexecstack -Wl,-z,relro -Wl,-z,now -Wl,-z,max-page-size=4096 -Wl,--color-diagnostics -Wl,--no-rosegment -Wl,--no-call-graph-profile-sort -Wl,--exclude-libs=libvpx_assembly_arm.a --unwindlib=none --target=aarch64-linux-android21 -Wl,-mllvm,-enable-machine-outliner=never -no-canonical-prefixes -Wl,--gc-sections -Wl,--gdb-index --sysroot=../../../../../../../Library/Android/sdk/ndk/23.1.7779620/toolchains/llvm/prebuilt/darwin-x86_64/sysroot -Wl,--warn-shared-textrel -Wl,-z,defs -Wl,--as-needed -pie -Bdynamic -Wl,-z,nocopyreloc -Wl,--dynamic-linker,/system/bin/linker64 -o "./exe.unstripped/stun_prober" -Wl,--start-group @"./stun_prober.rsp"  -Wl,--end-group  -ldl -lm -llog -lGLESv2 -lOpenSLES
Traceback (most recent call last):
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/toolchain/gcc_link_wrapper.py", line 91, in <module>
    sys.exit(main())
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/toolchain/gcc_link_wrapper.py", line 78, in main
    result = subprocess.call(
  File "/usr/local/Cellar/python@3.9/3.9.9/Frameworks/Python.framework/Versions/3.9/lib/python3.9/subprocess.py", line 349, in call
    with Popen(*popenargs, **kwargs) as p:
  File "/usr/local/Cellar/python@3.9/3.9.9/Frameworks/Python.framework/Versions/3.9/lib/python3.9/subprocess.py", line 951, in __init__
    self._execute_child(args, executable, preexec_fn, close_fds,
  File "/usr/local/Cellar/python@3.9/3.9.9/Frameworks/Python.framework/Versions/3.9/lib/python3.9/subprocess.py", line 1821, in _execute_child
    raise child_exception_type(errno_num, err_msg, err_filename)
FileNotFoundError: [Errno 2] No such file or directory: '../../../../build_system/llvm-build/mac/mac/Release+Asserts/bin/llvm-strip'
```

这个报错是说，在链接的时候，找不到 LLVM 工具链中的 `llvm-strip`。

同样，我们将 NDK 下的工具拷贝过去，同时也要将依赖的库拷贝过去：
```
android-ndk-r23b-darwin % cp toolchains/llvm/prebuilt/darwin-x86_64/bin/llvm-strip /Users/lisi/Projects/opensource/OpenRTCClient/build_system/llvm-build/mac/mac/Release+Asserts/bin/
android-ndk-r23b-darwin % cp -r toolchains/llvm/prebuilt/darwin-x86_64/lib64 /Users/lisi/Projects/opensource/OpenRTCClient/build_system/llvm-build/mac/mac/Release+Asserts/
```

要做同样处理的还有 `llvm-nm` 工具：
```
android-ndk-r23b-darwin % cp toolchains/llvm/prebuilt/darwin-x86_64/bin/llvm-nm /Users/lisi/Projects/opensource/OpenRTCClient/build_system/llvm-build/mac/mac/Release+Asserts/bin/
```

#### (5). Unix 域 Socket 地址错误导致的编译失败

再次执行 `ninja`，遇到如下编译报错：
```
OpenRTCClient % webrtc_build build android arm64 debug                              
/Users/lisi/Projects/opensource/OpenRTCClient/build_system/tools/bin/mac/ninja -C /Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug   
ninja: Entering directory `/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug'
[7/744] ACTION //sdk/android:base_java__errorprone(//build/toolchain/android:android_clang_arm64)
FAILED: obj/sdk/android/base_java__errorprone.errorprone.stamp 
python3 ../../../../webrtc/build/android/gyp/compile_java.py --depfile=gen/sdk/android/base_java__errorprone.d --generated-dir=gen/sdk/android/base_java/generated_java --jar-path=obj/sdk/android/base_java__errorprone.errorprone.stamp --java-srcjars=\[\"gen/sdk/android/base_java.generated.srcjar\"\] --target-name //sdk/android:base_java__errorprone --header-jar obj/sdk/android/base_java.turbine.jar --classpath=\[\"obj/sdk/android/base_java.turbine.jar\"\] --classpath=@FileArg\(gen/sdk/android/base_java.build_config.json:deps_info:javac_full_interface_classpath\) --java-version=1.8 --bootclasspath=@FileArg\(gen/sdk/android/base_java.build_config.json:android:sdk_interface_jars\) --chromium-code=1 --jar-info-exclude-globs=\[\"\*/R.class\",\ \"\*/R\\\$\*.class\",\ \"\*/Manifest.class\",\ \"\*/Manifest\\\$\*.class\",\ \"\*/GEN_JNI.class\"\] --processorpath=@FileArg\(gen/tools/android/errorprone_plugin/errorprone_plugin.build_config.json:deps_info:host_classpath\) --enable-errorprone @gen/sdk/android/base_java.sources --javac-arg=-Werror
Traceback (most recent call last):
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_java.py", line 800, in <module>
    sys.exit(main(sys.argv[1:]))
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_java.py", line 693, in main
    and server_utils.MaybeRunCommand(name=options.target_name,
  File "/Users/lisi/Projects/opensource/OpenRTCClient/webrtc/build/android/gyp/util/server_utils.py", line 40, in MaybeRunCommand
    raise e
  File "/Users/lisi/Projects/opensource/OpenRTCClient/webrtc/build/android/gyp/util/server_utils.py", line 27, in MaybeRunCommand
    sock.connect(SOCKET_ADDRESS)
FileNotFoundError: [Errno 2] No such file or directory
```

在 `webrtc/build/android/gyp/util/server_utils.py` 文件中，我们可以看到 `SOCKET_ADDRESS` 的定义：
```
SOCKET_ADDRESS = '\0chromium_build_server_socket'
```

最前面的 '\0' 导致在 Mac 上创建 unix 域 socket 失败，并设置它为临时目录下的一个路径，如：
```
SOCKET_ADDRESS = '/tmp/chromium_build_server_socket'
```

同时，还有把 Unix 域 socket 的 server 端跑起来，如：
```
OpenRTCClient % python3 webrtc/build/android/fast_local_dev_server.py
```

#### (6). 编译资源失败

再次执行 `ninja`，遇到如下编译报错：
```
[20/692] ACTION //examples:AppRTCMobile__compile_resources(//build/toolchain/android:android_clang_arm64)
FAILED: gen/examples/AppRTCMobile__compile_resources.srcjar obj/examples/AppRTCMobile.ap_ obj/examples/AppRTCMobile.ap_.info gen/examples/AppRTCMobile__compile_resources_R.txt obj/examples/AppRTCMobile/AppRTCMobile.resources.proguard.txt gen/examples/AppRTCMobile__compile_resources.resource_ids 
python3 ../../../../webrtc/build/android/gyp/compile_resources.py --include-resources=@FileArg\(gen/examples/AppRTCMobile.build_config.json:android:sdk_jars\) --aapt2-path ../../../../webrtc/third_party/android_build_tools/aapt2/aapt2 --dependencies-res-zips=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:dependency_zips\) --extra-res-packages=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:extra_package_names\) --extra-main-r-text-files=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:extra_main_r_text_files\) --min-sdk-version=21 --target-sdk-version=29 --webp-cache-dir=obj/android-webp-cache --android-manifest gen/examples/AppRTCMobile_manifest/AndroidManifest.xml --srcjar-out gen/examples/AppRTCMobile__compile_resources.srcjar --version-code 1 --version-name Developer\ Build --arsc-path obj/examples/AppRTCMobile.ap_ --info-path obj/examples/AppRTCMobile.ap_.info --debuggable --r-text-out gen/examples/AppRTCMobile__compile_resources_R.txt --dependencies-res-zip-overlays=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:dependency_zips\) --proguard-file obj/examples/AppRTCMobile/AppRTCMobile.resources.proguard.txt --emit-ids-out=gen/examples/AppRTCMobile__compile_resources.resource_ids --depfile gen/examples/AppRTCMobile__compile_resources.d
E 3273   2236 Subprocess raised an exception:
Traceback (most recent call last):
  File "/Users/lisi/Projects/opensource/OpenRTCClient/webrtc/build/android/gyp/util/parallel.py", line 75, in __call__
    return self._func(*_fork_params[index], **_fork_kwargs)
TypeError: 'NoneType' object is not subscriptable
```

这里我们通过把资源编译的并发编译给关掉来解决这个问题。具体来说，可以修改 `webrtc/build/android/gyp/util/parallel.py` 这个文件，将文件中的 `DISABLE_ASYNC` 设置为 `True`，或者导出环境变量 `export DISABLE_ASYNC=1`。

#### (7). aapt 可执行文件格式错误导致编译失败

再次执行 `ninja`，遇到如下编译报错：
```
[36/579] ACTION //examples:AppRTCMobile__compile_resources(//build/toolchain/android:android_clang_arm64)
FAILED: gen/examples/AppRTCMobile__compile_resources.srcjar obj/examples/AppRTCMobile.ap_ obj/examples/AppRTCMobile.ap_.info gen/examples/AppRTCMobile__compile_resources_R.txt obj/examples/AppRTCMobile/AppRTCMobile.resources.proguard.txt gen/examples/AppRTCMobile__compile_resources.resource_ids 
python3 ../../../../webrtc/build/android/gyp/compile_resources.py --include-resources=@FileArg\(gen/examples/AppRTCMobile.build_config.json:android:sdk_jars\) --aapt2-path ../../../../webrtc/third_party/android_build_tools/aapt2/aapt2 --dependencies-res-zips=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:dependency_zips\) --extra-res-packages=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:extra_package_names\) --extra-main-r-text-files=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:extra_main_r_text_files\) --min-sdk-version=21 --target-sdk-version=29 --webp-cache-dir=obj/android-webp-cache --android-manifest gen/examples/AppRTCMobile_manifest/AndroidManifest.xml --srcjar-out gen/examples/AppRTCMobile__compile_resources.srcjar --version-code 1 --version-name Developer\ Build --arsc-path obj/examples/AppRTCMobile.ap_ --info-path obj/examples/AppRTCMobile.ap_.info --debuggable --r-text-out gen/examples/AppRTCMobile__compile_resources_R.txt --dependencies-res-zip-overlays=@FileArg\(gen/examples/AppRTCMobile.build_config.json:deps_info:dependency_zips\) --proguard-file obj/examples/AppRTCMobile/AppRTCMobile.resources.proguard.txt --emit-ids-out=gen/examples/AppRTCMobile__compile_resources.resource_ids --depfile gen/examples/AppRTCMobile__compile_resources.d
WARNING:root:Running in synchronous mode.
Traceback (most recent call last):
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_resources.py", line 1039, in <module>
    main(sys.argv[1:])
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_resources.py", line 956, in main
    manifest_package_name = _PackageApk(options, build)
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_resources.py", line 759, in _PackageApk
    partials = _CompileDeps(options.aapt2_path, dep_subdirs,
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_resources.py", line 619, in _CompileDeps
    partials = lisi(
  File "/Users/lisi/Projects/opensource/OpenRTCClient/webrtc/build/android/gyp/util/parallel.py", line 207, in BulkForkAndCall
    yield func(*args, **kwargs)
  File "/Users/lisi/Projects/opensource/OpenRTCClient/build/android/arm64/debug/../../../../webrtc/build/android/gyp/compile_resources.py", line 583, in _CompileSingleDep
    build_utils.CheckOutput(
  File "/Users/lisi/Projects/opensource/OpenRTCClient/webrtc/build/android/gyp/util/build_utils.py", line 260, in CheckOutput
    child = subprocess.Popen(args,
  File "/usr/local/Cellar/python@3.9/3.9.9/Frameworks/Python.framework/Versions/3.9/lib/python3.9/subprocess.py", line 951, in __init__
    self._execute_child(args, executable, preexec_fn, close_fds,
  File "/usr/local/Cellar/python@3.9/3.9.9/Frameworks/Python.framework/Versions/3.9/lib/python3.9/subprocess.py", line 1821, in _execute_child
    raise child_exception_type(errno_num, err_msg, err_filename)
OSError: [Errno 8] Exec format error: '../../../../webrtc/third_party/android_build_tools/aapt2/aapt2'
```

这是由于 `webrtc/third_party/android_build_tools/aapt2/aapt2` 文件实际上为 Linux 平台的可执行文件。

编译资源的 template 在 `webrtc/build/config/android/internal_rules.gni` 中定义：
```
  template("compile_resources") {
    forward_variables_from(invoker, TESTONLY_AND_VISIBILITY)

    _deps = [
      invoker.android_sdk_dep,
      invoker.build_config_dep,
    ]
    if (defined(invoker.android_manifest_dep)) {
      _deps += [ invoker.android_manifest_dep ]
    }
    foreach(_dep, invoker.deps) {
      _target_label = get_label_info(_dep, "label_no_toolchain")
      if (filter_exclude([ _target_label ], _java_library_patterns) == [] &&
          filter_exclude([ _target_label ], _java_resource_patterns) != []) {
        # Depend on the java libraries' transitive __assetres target instead.
        _deps += [ "${_target_label}__assetres" ]
      } else {
        _deps += [ _dep ]
      }
    }

    if (defined(invoker.arsc_output)) {
      _arsc_output = invoker.arsc_output
    }
    _final_srcjar_path = "${target_gen_dir}/${target_name}.srcjar"

    _script = "//build/android/gyp/compile_resources.py"

    _inputs = [
      invoker.build_config,
      android_sdk_tools_bundle_aapt2,
    ]

    _rebased_build_config = rebase_path(invoker.build_config, root_build_dir)

    _args = [
      "--include-resources=@FileArg($_rebased_build_config:android:sdk_jars)",
      "--aapt2-path",
      rebase_path(android_sdk_tools_bundle_aapt2, root_build_dir),
      "--dependencies-res-zips=@FileArg($_rebased_build_config:deps_info:dependency_zips)",
      "--extra-res-packages=@FileArg($_rebased_build_config:deps_info:extra_package_names)",
      "--extra-main-r-text-files=@FileArg($_rebased_build_config:deps_info:extra_main_r_text_files)",
      "--min-sdk-version=${invoker.min_sdk_version}",
      "--target-sdk-version=${invoker.target_sdk_version}",
      "--webp-cache-dir=obj/android-webp-cache",
    ]
```

其中会调用脚本文件 `webrtc/build/android/gyp/compile_resources.py` 并传入 aapt2 的路径作为其中的一个参数。可以看到这里引用的 aapt2 的路径为 `android_sdk_tools_bundle_aapt2`。`android_sdk_tools_bundle_aapt2` 是一个编译配置项，其定义在 `webrtc/build/config/android/config.gni` 中，这个配置项有默认值：
```
    android_sdk_tools_bundle_aapt2_dir =
        "//third_party/android_build_tools/aapt2"
 . . . . . .
  android_sdk_tools_bundle_aapt2 = "${android_sdk_tools_bundle_aapt2_dir}/aapt2"
```

我们可以通过在 `args.gn` 中定制 `android_sdk_tools_bundle_aapt2_dir` 这个配置项来解决这个问题。具体来说，需要将这个配置项设置为 Android SDK 中我们选择的 build-tools 的路径，如下：
```
android_sdk_tools_bundle_aapt2_dir = "/Users/lisi/Library/Android/sdk/build-tools/30.0.1"
```

再次再次执行 `ninja`，可以正确编译出各个需要的文件。

笔者在 GitHub 上建了一个仓库，放了 WebRTC 完整的代码以及构建系统，包括 WebRTC 的源码及其依赖的第三方库，编译所需要的各种工具等，其中主要针对 WebRTC 的 m98 版，搭建了整个的构建环境，地址为 [OpenRTCClient](https://github.com/hanpfei/OpenRTCClient)，有需要的朋友可以参考一些。

Done.
