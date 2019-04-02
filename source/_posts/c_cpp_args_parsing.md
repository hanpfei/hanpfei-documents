---
title: C/C++ 命令行参数解析库选型
date: 2019-04-01 11:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

C/C++ 程序可以用的命令行参数解析库主要有如下这些：
<!--more-->
 * cmdline：一个轻量级的 C/C++ 命令行参数解析库。**[GitHub 主页](https://github.com/tanakh/cmdline)**。

 * Boost.Program_options：Boost 程序的标准命令行参数解析库。

 * gflags：Google 的 C/C++命令行参数解析库。**[GitHub 主页](https://github.com/gflags/gflags)**。

 * getopt：Unix-like 系统下 C/C++ 程序的标准命令参数解析库。

 * suboptions：一个用于解析多个层级的复杂参数的库。**[主页](https://www.gnu.org/software/libc/manual/html_node/Suboptions.html#Suboptions)**。

 * argp：GNU 的一个解析 Unix 风格的参数向量的接口。**[主页](https://www.gnu.org/software/libc/manual/html_node/Argp.html)**。

 * Argtable：ANSI C 命令行参数解析库。**[主页](http://argtable.sourceforge.net/)**。

考虑到如下因素：

 * 跨平台；
 * 功能强大；
 * 项目的活跃度，

最终选择 Google 的 ***gflags***。

参考文档：

[Parsing Program Arguments](https://www.gnu.org/software/libc/manual/html_node/Parsing-Program-Arguments.html)

[使用 getopt() 进行命令行处理](https://www.ibm.com/developerworks/cn/aix/library/au-unix-getopt.html)

[C/C++中有哪些简单好用的命令行参数解析工具？](https://segmentfault.com/q/1010000000709952)

[c++ - 在 C++ 中，用于解析 命令行 参数的库](https://ask.helplib.com/others/post_1630372)

[【C++】cmdline —— 轻量级的C++命令行解析库](https://blog.csdn.net/xiaohui_hubei/article/details/40479811)
