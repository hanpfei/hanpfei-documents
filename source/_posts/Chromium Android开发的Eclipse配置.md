---
title: Chromium Android开发的Eclipse配置
date: 2017-01-22 14:05:49
categories: Android开发
tags:
- Android开发
- chromium
- 翻译
---

# 单次的Eclipse配置

这一节包含第一次启动Eclipse时需要的设置步骤。你应该只需浏览这个部分一次，即使你切换了workspaces。
<!--more-->
* 启动Eclipse。它可以通过如下的两种方式启动：
  * 在Ubuntu的主菜单，选择 Applications > Programming > Eclipse 4.5
  * 在命令行中，输入 eclipse45

* 选择你硬盘上的某个位置作为 workspace。 
* 安装CDT - C/C++ Development Tools

  * 从主菜单选择 Help > Install New Software... 
  * 如果用的是Eclipse的mars版的话，选择 mars - [http://download.eclipse.org/releases/mars](http://download.eclipse.org/releases/mars) ( 或任何其它似乎合适的镜像， 如neon http://download.eclipse.org/releases/neon )
  * 选中 Mobile and Device Development > C/C++ Remote Launch
  * 选中 Programming Languages > C/C++ Development Tools
  * 一路点击Next，Finish 等等，来结束向导。
  * 如果你在安装 “C++ Remote Launch” 时遇到了关于 org.eclipse.rse.ui 的missing dependency 错误，则添加一个可用的更新路径http://download.eclipse.org/dsdp/tm/updates/3.2。
  * 当提示重启 Eclipse 时点击 Restart Now。

* 内存
  * 关闭 Eclipse
  * 将如下的几行添加到 `~/.eclipse/init.sh`：
  ```
  ECLIPSE_MEM_START=1024m
  ECLIPSE_MEM_MAX=8192m
  ```

# 通用 Workspace 配置
有一些设置应用于你的 workspace 的所有工程。下面的所有设置都在 Window > Preferences里。

* Android格式化 (formatting)
  * 下载 [android-formatting.xml](https://raw.githubusercontent.com/android/platform_development/master/ide/eclipse/android-formatting.xml)
  * 在左边的配置树中选择 Java > Code Style > Formatter
  * 点击 Import...
  * 选中 android-formatting.xml 文件
  * 确认 Active profile 已经被设置为了Android

* Java import 顺序
  * 下载 [android.importorder](https://raw.githubusercontent.com/android/platform_development/master/ide/eclipse/android.importorder)。
  * 在左边的配置树中选择 Java > Code Style > Organize Imports
  * 点击 Import...
  * 选中 android.importorder 文件

* 禁用自动刷新 (automatic refresh)。否则，Eclipse将时常地去刷新你的工程（那可能会比较慢）。
  * 在左边的配置树中选择 General > Workspace
  * 如果选中了的话就反选 Refresh using native hooks or polling
  * 在左边的配置树中选择 General > Startup and Shutdown
  * 如果选中了的话就反选 Refresh workspace on startup

* 禁用 build before launching
  * 选择 Run/Debug > Launching
  * 反选  Build (if required) before launching

* .gyp 和 .gypi文件类型配置
  * 选择 General > Editors > File Associations
  * 添加 `.gyp` 和 `.gypi` 文件类型，并将它们与 Python Editor关联。
    * 参考 http://pydev.org/index.html 的说明在Eclipse中配置 Python Editor。
  * 通过 Ctrl+Shift+P 和自动匹配 bracket highlight 来享受快乐的生活吧

* Tab ordering
  * If you prefer ordering your tabs by most recently used, go to General > Appearance and check Show most recently used tabs

* 自动完成
  * 选择 Java > Editor > Content Assist
  * 选中 Enable auto activation
  * 修改 Auto activation triggers for Java: 为 ._abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ

* 单行长度
  * 如果你想要修改单行长度指示器，则选择 General > Editors > Text Editors
  * 选中 Show print margin 并将 Print margin column: 修改为 100。

# 工程配置

## 创建工程
* 在主菜单选择 File > New > Project... 
* 在工程树中选择 C/C++ > C++ 并点击 Next。
  * 注意：不是 “Makefile Project with Existing Code”，即使那听起来不错。
* 关于工程名 (Project name)，使用一个对你来说有意义的名字。比如 "chrome-android”
* 反选 Use default location。点击 Browse... 并选择你的 Chromium gclient目录的src 目录。
* 关于工程类型(Project type)，使用 Makefile project > Empty Project。
* 关于 Toolchains 则使用  -- Other Toolchain --
* 点击 Next
* 禁用 默认 CDT builder
  * 点击 Advanced Settings...
  * 在左边的配置树中选择 Builders
  * 反选 CDT Builder
  * 如果出现了 这是一个  ‘advanced feature’ 的警告对话框就点击 OK。
  * 点击 OK 关闭 工程属性对话框 (project properties dialog) 并返回工程创建向导 ( project creation wizard )
  * 点击 Finish 创建工程

## 配置工程
* 在工程上右键点击鼠标并选择 Properties

* 排除 Resources (可选的). 这个可以加快 Eclipse 一点，并可能使indexer更开心。
  * 选择 Resources > Resource Filters
  * 点击 Add...
  * 选择 Exclude all，选择文件夹，并选中 All children (recursive)
  * 键入 .git 作为名字
  * 点击 OK
  * 点击 Apply 提交修改

* C/C++ Indexer (deprecated, seems to be done by default)
  * 在左边的配置树中选择 C/C++ General > Indexer
  * 点击 Restore Defaults
  * 选中 Enable project specific settings
  * 反选 Index source files not included in the build
  * 反选 Allow heuristic resolution of includes
  * 点击 Apply 提交修改

* C/C++ Paths and Symbols. 这将帮助 Eclipse
 为 Chrome构建 symbol。
  * 如果使用gyp构建系统的话，则在命令行中运行`GYP_GENERATORS=eclipse build/gyp_chromium`；如果使用 gn 构建系统的话，则在命令行中运行 `gn gen out/Default/ --ide=eclipse`，然后执行命令 `tools/android/eclipse/generate_cdt_clang_settings.py  out/Default/eclipse-cdt-settings.xml`。
  * 这将生成 `<project root>/out/Release/eclipse-cdt-settings.xml` 或  `<project root>/out/Default/eclipse-cdt-settings.xml` 文件，后面会用到。
  *  在左边的配置树中选择 C/C++ General > Paths and Symbols
  * 点击 Restore Defaults 清楚所有旧设置。
  * 点击 Import Settings... 将显示导入对话框。T
  * 点击 Browse... 将显示 *文件浏览器*。
  * 选择 <project root>/out/Default/eclipse-cdt-settings.xml。
  * 点击 Finish 按钮。整个首选项对话框应该消失。
  * 在工程上右键点击鼠标，并选择Index > Rebuild

* Java
  * 创建一个指向 <project root>/tools/android/eclipse/.classpath 的链接 <project root>/.classpath：
  ```
  ln -s tools/android/eclipse/.classpath .classpath
  ```

* 以如下方式编辑 <project root>/.project ，使工程成为一个Java工程：
  * 在 <buildSpec> 中添加如下几行：
  ```
  <buildCommand>
    <name>org.eclipse.jdt.core.javabuilder</name>
    <arguments></arguments>
  </buildCommand>
  ```
  * 在 <natures> 中添加如下几行：
  ```
  <nature>org.eclipse.jdt.core.javanature</nature>
  ```

参考文档：

[Eclipse Configuration for Android](https://chromium.googlesource.com/chromium/src.git/+/master/docs/eclipse.md)
