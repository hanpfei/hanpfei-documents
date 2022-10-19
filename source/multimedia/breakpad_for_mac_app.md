---
title: 在 Mac 客户端应用程序中使用 breakpad
date: 2022-06-29 19:37:49
categories: 音视频开发
tags:
- 音视频开发
---

本文档是使用 Breakpad 构建 Mac 客户端应用程序的分步指南。
<!--more-->
## 准备 Breakpad 的二进制构建以用于你的代码树

你可以通过 Breakpad 工程中的 xcode 工程文件构建 Breakpad 框架和工具的二进制文件，也可以将其构建为项目的依赖项。建议采用前者的方式，这里会对这种方式做详细介绍，因为通过其它工程构建依赖项是有问题的（匹配配置名称），并且 Breakpad 代码几乎不会像你的应用程序那样经常更改。

## 构建必要的目标

所有的目录都是相对于 Breakpad 源码的 `src` 目录的。

 * 以 Release 模式构建 
 `client/mac/Breakpad.xcodeproj` 的 'All' 目标。通过 `xcodebuild` 命令可以看到这个工程的所有目标：

```
breakpad % xcodebuild -list -project src/client/mac/Breakpad.xcodeproj 
Command line invocation:
    /Applications/Xcode.app/Contents/Developer/usr/bin/xcodebuild -list -project src/client/mac/Breakpad.xcodeproj

User defaults from command line:
    IDEPackageSupportUseBuiltinSCM = YES

Information about project "Breakpad":
    Targets:
        Breakpad
        Inspector
        breakpadUtilities
        crash_report_sender
        BreakpadTest
        All
        UnitTests
        generator_test
        minidump_file_writer_unittest
        handler_test
        gtest
        crash_generation_server_test
        minidump_generator_test_helper

    Build Configurations:
        Debug
        Debug With Code Coverage
        Release

    If no build configuration is specified and -scheme is not passed then "Release" is used.

    Schemes:
        All
        all_unittests
        Breakpad
        BreakpadTest
        breakpadUtilities
        byte_cursor_unittest
        bytereader_unittest
        crash_generation_server_test
        crash_report
        crash_report_sender
        dump_syms
        dwarf_cfi_to_module_unittest
        dwarf_cu_to_module_unittest
        dwarf_line_to_module_unittest
        dwarf2diehandler_unittest
        dwarf2reader_cfi_unittest
        generator_test
        gtest
        gtestmockall
        handler_test
        Inspector
        macho_dump
        macho_reader_unittest
        minidump_file_writer_unittest
        minidump_generator_test_helper
        minidump_upload
        module_unittest
        stabs_reader_unittest
        stabs_to_module_unittest
        symupload
        test_assembler_unittest
        UnitTests
```

通过如下命令可以以 Release 模式构建 Breakpad 的 'All' 目标：

```
xcodebuild -configuration Release -target All -project src/client/mac/Breakpad.xcodeproj
```

也可以通过如下命令只构建 Breakpad 框架：

```
xcodebuild -configuration Release -target Breakpad -project src/client/mac/Breakpad.xcodeproj
```

 * 执行 `cp -R client/mac/build/Release/Breakpad.framework <location in your source tree>`
 * 在 `tools/mac/dump_syms` 目录中，构建 `dump_syms.xcodeproj`，并把 `tools/mac/dump_syms/build/Release/dump_syms` 拷贝到一个在构建过程中可以运行的安全的地方。

## 添加 Breakpad.framework

在你的应用程序的框架中，把 Breakpad.Framework 添加进你的工程的框架设置中。当你从文件选择器中选择它时，它将让你选取一个目标来添加；请继续，并检查与你的应用程序关联的那个。

## 将 Breakpad 复制到你的应用程序包中

将 Breakpad 复制到你的应用程序包中，这样它就会在运行时出现。

转到 Xcode Project 窗口的 Targets 部分。点击开合三角以显示应用程序的构建阶段。使用上下文菜单（控制单击，Control Click）添加一个新的复制文件（Copy Files）阶段。在此新阶段的新 'Get Info' 的 General 面板上，将目标设置为 'Frameworks' 关闭 'Info' 面板。使用上下文菜单重命名你的新阶段 'Copy Frameworks'。现在再次将 Breakpad 拖入此 Copy Frameworks 阶段。将其从项目文件树中出现的任何位置拖动。

## 添加一个新的 Run Script 构建阶段

在构建阶段快结束时，添加一个新的 Run Script 构建阶段。这将在 Xcode 对你的项目调用 `/usr/bin/strip` 之前运行。这是你将调用 `dump_sym` 以输出构建的每个体系结构的符号的地方。在我的情况下，相关的代码为：

```
#!/bin/sh
$TOOL_DIR=<location of dump_syms from step 3 above>

"$TOOL_DIR/dump_syms" -a ppc "$PROD" > "$TARGET_NAME ppc.breakpad"

"$TOOL_DIR/dump_syms" -a i386 "$PROD" > "$TARGET_NAME i386.breakpad"
```

## 调整工程设置

 * 打开 Separate Strip，
 * 把 Strip Style 设置为 Non-Global Symbols。

## 编写代码

你需要有一个对象作为 NSApplication 的委托。 在此对象的头文件中，你需要添加

 1. 为 Breakpad 添加一个 ivar，以及
 2. 一个 applicationShouldTerminate:(NSApplication* sender) 消息的声明。

```
#import <Breakpad/Breakpad.h>

@interface BreakpadTest : NSObject {
   .
   .
   .
   BreakpadRef breakpad;
   .
   .
   .
}
.
.
- (NSApplicationTerminateReply)applicationShouldTerminate:(NSApplication *)sender;
.
.
@end
```

在你的对象的实现文件中，

 1. 添加如下的 InitBreakpad 方法
 2. 将你的 awakeFromNib 方法修改为如下所示，
 3. 修改/添加你的应用程序的委托方法，如下所示

```
static BreakpadRef InitBreakpad(void) {
  NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
  BreakpadRef breakpad = 0;
  NSDictionary *plist = [[NSBundle mainBundle] infoDictionary];
  if (plist) {
    // Note: version 1.0.0.4 of the framework changed the type of the argument 
    // from CFDictionaryRef to NSDictionary * on the next line:
    breakpad = BreakpadCreate(plist);
  }
  [pool release];
  return breakpad;
}

- (void)awakeFromNib {
  breakpad = InitBreakpad();
}

- (NSApplicationTerminateReply)applicationShouldTerminate:(NSApplication *)sender {
  BreakpadRelease(breakpad);
  return NSTerminateNow;
}
```

## 配置 Breakpad

为你的应用程序配置 Breakpad。

 1. 在 Breakpad.framework 的 Breakpad.h 文件中查看要传递给 BreakpadCreate() 的键、默认值和描述。
 2. 在传递给 BreakpadCreate() 的字典中添加/编辑特定于 Breakpad 的条目 —— 通常是应用程序的信息 plist。

Notifier Info.plist 中的示例：`<key>BreakpadProduct</key><string>Google_Notifier_Mac</string> <key>BreakpadProductDisplay</key><string>${PRODUCT_NAME}</string>`

## 构建你的应用程序

快完成了！

## 验证

再检查一遍：

你的应用程序应在其包内容中包含：myApp.app/Contents/Frameworks/Breakpad.framework。

符号文件中具有合理的内容（你可以使用文本编辑器查看它们。）

再次查看项目的 Copy Frameworks 阶段。 你泄露了 .h 文件吗？选择它们并删除它们。（如果你将一堆文件拖到你的项目中，Xcode 经常想将你的 .h 文件复制到构建中，这会泄露谷歌的秘密。要警惕！）

## 上传符号文件

你需要配置构建过程以将符号存储在 minidump 处理器可以访问的位置。tools/mac/symupload 中有一个工具，可用于通过 HTTP post 发送符号文件。

 1. 测试

过添加到应用程序的 Info.plist 中，配置 breakpad 以将报告发送到 URL： 

```
<key>BreakpadURL</key>
<string>upload URL</string>
<key>BreakpadReportInterval</key>
<string>30</string>
```

## 最后的说明

Breakpad 会检查它是否在调试器下运行，如果是，通常什么都不做。但是，你可以通过将 Unix shell 变量 BREAKPAD_IGNORE_DEBUGGER 设置为非零值来强制 Breakpad 在调试器下运行。你可以使用 #if DEBUG 将上述编写代码步骤中的源代码括起来，以从调试版本中完全消除它。参考 //depot/googlemac/GoogleNotifier/main.m 的例子。FYI，当你的进程 forks() 是，子进程的异常处理器复位为默认值。因此它们必须重新初始化 Breakpad，否则异常将由 Apple 的 Crash Reporter 处理。

**参考文档**

[mac下利用Breakpad的dump文件进行调试](https://cloud.tencent.com/developer/article/1084368)

[How To Add Breakpad To Your Mac Client Application](https://github.com/hanpfei/OpenRTCClient/blob/m98_4758/webrtc/third_party/breakpad/breakpad/docs/mac_breakpad_starter_guide.md)

[通过Xcode命令行编译](https://www.jianshu.com/p/55b80e746e38)

[探究 Xcode 命令行用法一：Xcode 构建必备认知](https://juejin.cn/post/7024326946406268959)
