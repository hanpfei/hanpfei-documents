---
title: 在C代码调用C++代码
date: 2017-01-24 11:43:49
tags:
- C/C++
categories:
- C/C++
---

由于历史原因，以及不同开发人员的技术偏好，C语言和C++语言都有一些独有的非常有价值的项目，因而两种语言的互操作，充分利用前人造的轮子是一件非常有价值的事情。
<!--more-->
C++代码调用C代码很简单，只要分别在包含的C头文件的开头和结尾加上如下的两个块：
```
#ifdef __cplusplus
extern "C" {
#endif
```
和
```
#ifdef __cplusplus
}
#endif
```
即可。

然而为了支持类、重载等更加高级的特性，在编译C++代码时，C++符号会被修饰。我们dump Linux平台加密库  ***libcrypto++*** 的符号表，可以看到如下的内容：
```
$ readelf -s /usr/lib/libcrypto++.so

Symbol table '.dynsym' contains 9607 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000001daa58     0 SECTION LOCAL  DEFAULT    9 
     2: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTIi@CXXABI_1.3 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __errno_location@GLIBC_2.2.5 (3)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt18uncaught_exceptionv@GLIBCXX_3.4 (4)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt8__detail15_List_node_base7_M_hookEPS0_@GLIBCXX_3.4.15 (5)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND getservbyname@GLIBC_2.2.5 (6)
     7: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND bind@GLIBC_2.2.5 (6)
     8: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt29_Rb_tree_insert_and_rebalancebPSt18_Rb_tree_node_baseS0_RS_@GLIBCXX_3.4 (4)
     9: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __longjmp_chk@GLIBC_2.11 (7)
    10: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTIh@CXXABI_1.3 (2)
    11: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTVSt9basic_iosIcSt11char_traitsIcEE@GLIBCXX_3.4 (4)
    12: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND socket@GLIBC_2.2.5 (6)
    13: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt14basic_ifstreamIcSt11char_traitsIcEED1Ev@GLIBCXX_3.4 (4)
    . . . . . .
    86: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSo5writeEPKcl@GLIBCXX_3.4 (4)
    87: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND malloc@GLIBC_2.2.5 (6)
    88: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt9basic_iosIcSt11char_traitsIcEE4initEPSt15basic_streambufIcS1_E@GLIBCXX_3.4 (4)
    89: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSi5seekgElSt12_Ios_Seekdir@GLIBCXX_3.4 (4)
    90: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND pthread_key_delete@GLIBC_2.2.5 (3)
    91: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND shutdown@GLIBC_2.2.5 (6)
    92: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt15set_new_handlerPFvvE@GLIBCXX_3.4 (4)
    93: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND pthread_getspecific@GLIBC_2.2.5 (3)
    94: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND strcmp@GLIBC_2.2.5 (6)
    95: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND strtol@GLIBC_2.2.5 (6)
    96: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND ioctl@GLIBC_2.2.5 (6)
    . . . . . .
   186: 00000000002c5a80   142 FUNC    GLOBAL DEFAULT   12 _ZN8CryptoPP6xorbufEPhPKhS2_m
   187: 00000000002fd6d0     9 FUNC    WEAK   DEFAULT   12 _ZN8CryptoPP21InvertibleRSAFunction9BERDecodeERNS_22BufferedTransformationE
   188: 00000000001ea840    73 FUNC    GLOBAL DEFAULT   12 _ZN8CryptoPP13Base64Decoder22GetDecodingLookupArrayEv
   189: 0000000000249760     6 FUNC    WEAK   DEFAULT   12 _ZThn8_N8CryptoPP13DL_SignerImplINS_25DL_SignatureSchemeOptionsINS_5DL_SSINS_13DL_Keys_ECDSAINS_4EC2NEEENS_18DL_Algorithm_ECDSAIS4_EENS_37DL_SignatureMessageEncodingMethod_DSAENS_6SHA256EiEES5_S7_S8_S9_EEED0Ev
   190: 0000000000278b60    86 FUNC    WEAK   DEFAULT   12 _ZN8CryptoPP8Rijndael3DecD1Ev
   191: 00000000001fd1f0     2 FUNC    WEAK   DEFAULT   12 _ZN8CryptoPP23DefaultEncryptorWithMAC8FirstPutEPKh
   192: 000000000026a490    51 FUNC    GLOBAL DEFAULT   12 _ZN8CryptoPP23FilterWithBufferedInputC2EPNS_22BufferedTransformationE
   193: 0000000000285180     6 FUNC    WEAK   DEFAULT   12 _ZNK8CryptoPP8GCM_Base6IVSizeEv
   194: 000000000032e830   510 FUNC    WEAK   DEFAULT   12 _ZN8CryptoPP18StandardReallocateItNS_20AllocatorWithCleanupItLb0EEEEENT0_7pointerERS3_PT_NS3_9size_typeES8_b
   195: 00000000002a1790   185 FUNC    WEAK   DEFAULT   12 _ZSt18uninitialized_copyISt15_Deque_iteratorIyRKyPS1_ES0_IyRyPyEET0_T_S9_S8_
   196: 0000000000355610    25 OBJECT  WEAK   DEFAULT   14 _ZTSN8CryptoPP11RSAFunctionE
    . . . . . .
```
这与我们在源文件和头文件里看到的那些函数、类的声明定义都不一样。通过binutils的工具`c++filt` demangle这些符号可以让我们看到它们在代码里的样子：
```
$ c++filt _ZTSN8CryptoPP11RSAFunctionE
typeinfo name for CryptoPP::RSAFunction

$ c++filt _ZN8CryptoPP18StandardReallocateItNS_20AllocatorWithCleanupItLb0EEEEENT0_7pointerERS3_PT_NS3_9size_typeES8_b
CryptoPP::AllocatorWithCleanup<unsigned short, false>::pointer CryptoPP::StandardReallocate<unsigned short, CryptoPP::AllocatorWithCleanup<unsigned short, false> >(CryptoPP::AllocatorWithCleanup<unsigned short, false>&, unsigned short*, CryptoPP::AllocatorWithCleanup<unsigned short, false>::size_type, CryptoPP::AllocatorWithCleanup<unsigned short, false>::size_type, bool)
```
那到底有没有办法在C代码中调用C++代码呢？方法当然是有的，而且还不止一种。

# 通过extern "C"调用

在 .cpp 文件中定义一个函数，声明为`extern "C"`，则该函数可以方便地在C代码中调用。由于该函数在 .cpp 文件中定义，因而在该函数的实现中，可以调用任意的C++代码，包括C++函数，创建C++类等等。

C++头文件：
```
#ifndef CPPFUNCTIONS_H_
#define CPPFUNCTIONS_H_

#ifdef __cplusplus
int cpp_func(int input);
extern "C" {
#endif

int c_func(int input);

#ifdef __cplusplus
}
#endif
#endif /* CPPFUNCTIONS_H_ */
```

C++实现文件如下：
```
#include "CppFunctions.h"

int cpp_func(int input) {
    return 5;
}

int c_func(int input) {
    return cpp_func(input);
}
```

在C代码里调用C++函数：
```
#include <stdio.h>

#include "CppFunctions.h"

int main(int argc, char **argv) {
    printf("%d\n", c_func(10));
    return 0;
}
```
在C++文件里定义的`c_func`函数就像一座桥一样，连接了C代码的世界和C++代码的世界。但 C 函数`c_func`的参数及返回值的类型自然是受到一定的限制的，但在函数实现中可以适配要调用的C++接口，做一些适配。

# 通过dlopen/dlsym调用
借助于在 .cpp 文件中定义的C函数，间接地调用C++接口，固然是能实现在 C 代码中调用C++代码的目标，然而还是有些麻烦。通过libdl提供的接口，可以使我们的目标通过更简便的方式实现。

为dlsym传入经过修饰的符号，可以找到对应的函数的地址。

通过如下命令将上面的CPPFunctions.cpp文件编译为一个动态链接库：
```
$ gcc -shared -fPIC CPPFunctions.cpp -o libCppLibTest.so
```

通过dlopen和dlsym找到对应的C++函数，并将其强制类型转换为适当类型的函数指针，然后通过函数指针调用目标函数，如：
```

#include <dlfcn.h>
#include <stdio.h>

int main(int argc, char **argv) {
    void *libCPPTest = dlopen("/home/hanpfei0306/workspace_java/CppLibTest/Debug/libCppLibTest.so", RTLD_NOW);
    int (*cpp_func)(int) = (int (*)(int))dlsym(libCPPTest, "_Z8cpp_funci");

    printf("cpp_func = %p\n", cpp_func);
    printf("cpp_func output = %d\n", cpp_func(10));

    return 0;
}
```

编译并执行上面的代码，在我的机器上可以看到如下的输出：
```
cpp_func = 0x7f35727a8650
cpp_func output = 5
```

# 参考资料

[C++-dlopen-mini-HOWTO](http://www.isotton.com/devel/docs/C++-dlopen-mini-HOWTO/)
[Using dlsym in c++ without extern “C”](http://stackoverflow.com/questions/18096596/using-dlsym-in-c-without-extern-c)
[关于linker的笔记](http://hqwrong.github.io/2015/04/14/linker.html#)
