---
title: JNI技巧
date: 2017-06-11 22:17:49
categories: Android开发
tags:
- Android开发
- 翻译
---

JNI 是指 Java 本地层接口（Java Native Interface）。它为用 Java 语言编写的受控代码定义了一种与本地层代码（用 C/C++ 编写）交互的方式。它是厂商无关的，其支持从动态共享库加载代码，尽管有时笨重，但它仍是有效的。

<!--more-->

如果你对它还不熟悉，可以阅读 [JNI规范（Java Native Interface Specification）](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html) 来获得对它的更多了解，了解 JNI 如何工作以及它有哪些功能。规范中有些地方的说明，并不是特别的清晰简洁明了，因而接下来的一些内容也许有点用。

## JavaVM 和 JNIEnv
JNI 定义了两个关键的数据结构，`JavaVM` 和 `JNIEnv`。它们都是指向函数表的指针。(在 C++ 版本中，它们是类，其中包含一个指向函数表的指针，及每个 JNI 函数对应一个的成员函数，这些成员函数则简单地调用函数表中的对应函数。) JavaVM 提供了“调用接口”函数，通过这些函数，可以创建和销毁一个 JavaVM。看一下 JavaVM 结构的定义就一目了然了：
```
#if defined(__cplusplus)
typedef _JavaVM JavaVM;
#else
typedef const struct JNIInvokeInterface* JavaVM;
#endif

/*
 * JNI invocation interface.
 */
struct JNIInvokeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;

    jint        (*DestroyJavaVM)(JavaVM*);
    jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);
    jint        (*DetachCurrentThread)(JavaVM*);
    jint        (*GetEnv)(JavaVM*, void**, jint);
    jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);
};

/*
 * C++ version.
 */
struct _JavaVM {
    const struct JNIInvokeInterface* functions;

#if defined(__cplusplus)
    jint DestroyJavaVM()
    { return functions->DestroyJavaVM(this); }
    jint AttachCurrentThread(JNIEnv** p_env, void* thr_args)
    { return functions->AttachCurrentThread(this, p_env, thr_args); }
    jint DetachCurrentThread()
    { return functions->DetachCurrentThread(this); }
    jint GetEnv(void** env, jint version)
    { return functions->GetEnv(this, env, version); }
    jint AttachCurrentThreadAsDaemon(JNIEnv** p_env, void* thr_args)
    { return functions->AttachCurrentThreadAsDaemon(this, p_env, thr_args); }
#endif /*__cplusplus*/
};
```

理论上，每个进程可以有多个 JavaVM 实例，但 Android 只允许有一个。

JNIEnv 则提供了大多数的 JNI 函数。你的本地层函数都接受 JNIEnv 作为其第一个参数。

JNIEnv 用于线程局部存储。因此，**你不能在线程之间共享同一个 JNIEnv**。如果一段代码没有其它方法获取它的 JNIEnv，你应该共享 JavaVM，并使用 `JavaVM` 结构的 `GetEnv` 函数找到线程的 `JNIEnv`。(假设它有一个；参见下面的 `AttachCurrentThread`。)

`JNIEnv` 和 `JavaVM` 的 C 声明与它们的 C++ 声明不同。依赖于是被 include 进 C 还是 C++ 源文件中，"jni.h" 头文件提供了不同的类型定义。因此，在两个语言都会包含的头文件中，包含 JNIEnv 参数并不是个好主意。(换句话说：如果你的头文件需要`#ifdef __cplusplus`，且该头文件中有任何内容引用了 JNIEnv，那么你可能需要做一些额外的工作。)

比如定义了一个函数，其接受一个 `JNIEnv` 指针作为参数。这个函数在 C 源文件和在 C++ 源文件中实现时，这个参数的类型实际上是不一样的。对这个函数的调用，也要区分是在 C 代码中调用还是在 C++ 代码中调用，并做不同的处理。

## 线程
所有线程都是 Linux 线程，由内核调度。它们通常由 Java 代码启动 (使用 `Thread.start` 方法 )，但它们也可以在其它地方创建，然后附到 JavaVM 上。比如，一个由 `pthread_create` 启动的线程，可以通过 JNI，即 JavaVM 实例的 `AttachCurrentThread` 或 `AttachCurrentThreadAsDaemon` 函数附到 JavaVM。在一个线程被附到 `JavaVM` 之前，它没有 `JNIEnv`，因而也 **不能执行 JNI 调用**。

把一个本底层创建的线程附接到 `JavaVM` 会创建一个 ` java.lang.Thread` 对象，并添加到“main” `ThreadGroup`，使它对于调试器可见。对一个已经附接到 `JavaVM` 的线程调用 `AttachCurrentThread` 是一个空操作。

通过把本地层创建的线程附接到 `JavaVM` 中之后，也就可以在该线程中方便地调用 JNI 函数，访问 Java 对象和结构了。

Android 不会挂起正在执行本地层代码的线程。如果正在进行垃圾回收，或者调试器发出了一个挂起请求，Android 将在线程下一次执行 JNI 调用时暂停它。

通过 JNI 函数附接的线程在它们退出前必须调用 `DetachCurrentThread`。如果直接这样写代码不方便，则在 Android 2.0 (Eclair) 或更高版本上，你可以使用`pthread_key_create` 定义一个将会在线程退出前被调用的析构函数，并在那儿调用 `DetachCurrentThread`。(以该 key 调用 `pthread_setspecific` 来将 `JNIEnv` 保存在线程局部存储中；以此，它将会作为参数被传进你的析构函数。)

## jclass，jmethodID，和 jfieldID
如果你想在本地层代码中访问 Java 对象的成员，你将需要执行以下操作：

* 通过 `FindClass` 获取该类的类对象引用
* 通过 `GetFieldID` 获取成员的成员 ID 对象引用
* 通过适当的方法获取成员的内容，比如 `GetIntField`

类似地，要调用一个方法，你首先要获取类对象的引用，然后获得方法 ID 对象引用。IDs 经常只是指向内部运行时数据结构的指针。查找它们可能需要一些字符串比较，然而一旦有了它们，获取成员或者调用方法的实际调用是非常快的。

如果性能很重要，在你的本底层代码中，进行一次查找操作并将结果缓存起来会很有用。由于有着每个进程一个 `JavaVM` 实例的限制，把这些数据保存在一个静态本地结构中是合理的。

在类被卸载之前，类引用，成员 IDs，和方法 IDs 会保证是有效的。只有当与一个 ClassLoader 相关联的所有类都可以被垃圾回收时，类才会被卸载，这很罕见，但在 Android 中也不是不可能。然而，注意 `jclass` 是一个类引用，且 **必须通过调用 `NewGlobalRef` 来保护** (参见下一节)。

如果你想在类被加载时缓存 IDs，并在类被卸载且重新加载时自动地重新缓存它们，初始化 IDs 的正确方法是，在适当的类中添加一段像下面这样的代码：

```
    /*
     * We use a class initializer to allow the native code to cache some
     * field offsets. This native function looks up and caches interesting
     * class/field/method IDs. Throws on failure.
     */
    private static native void nativeInit();

    static {
        nativeInit();
    }
```

在你的 C/C++ 代码中创建一个 `nativeClassInit` 方法执行 ID 查找。代码将会在类初始化时执行一次。如果类被卸载并重新加载，它将会再次执行。

## 局部和全局引用
传递给本地层方法的每个参数，和由 JNI 函数返回的几乎每个对象均是一个 “局部引用”。这意味着它在当前线程的当前本地层方法运行期间是有效的。**即使对象本身在本地层方法返回之后继续存活，引用依然不是有效的**。

这适用于所有的 `jobject` 子类，包括 `jclass`，`jstring`，和 `jarray`。（当启用扩展 JNI 检查时，运行时将为大多数引用的错误使用发出警告。）

获得非局部引用的仅有的方法是通过函数 `NewGlobalRef` 和 `NewWeakGlobalRef`。

如果你想更长期地持有引用，你必须使用一个“全局的”引用。`NewGlobalRef` 函数接收局部引用作为参数，并返回一个全局引用。全局引用保证是有效的，直到你调用 `DeleteGlobalRef`。

这一模式常被用于缓存 `FindClass` 返回的 jclass，如：
```
jclass localClass = env->FindClass("MyClass");
jclass globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```

如果这样说的话，那 jmethodID 和 jfieldID 呢？

所有的 JNI 方法都接收局部和全局的引用作为参数。可能引用同一个对象的引用具有不同的值。比如，连续地对相同对象调用 `NewGlobalRef` 获得的返回值可能不同。**要查看两个引用是否指向相同对象，你必须使用 `IsSameObject` 函数**。千万不要在本地层代码中用 `==` 比较引用。

这样的结果是在本地层代码中你 **一定不能假设对象引用是常量或唯一的**。在对一个方法的一次调用和下一次调用之间表示对象的 32 位值可能不同， 在连续调用中两个不同的对象可能具有相同的 32 位值。不要使用 `jobject` 值作为键。

程序员需要 “不过分地分配”局部引用。实际上，这意味着如果你正在创建大量的局部引用，也许在遍历一个对象数组，你应该使用 `DeleteLocalRef` 手动释放它们，而不是让 JNI 为你执行。实现只被要求为局部引用保留 16 个槽，因此如果你需要更多，你应该或者在运行过程中删除一些，或者使用 `EnsureLocalCapacity` / `PushLocalFrame` 保留更多。

注意 `jfieldID` 和 `jmethodID` 是不透明类型，而不是对象引用，且不应该被传给 `NewGlobalRef`。像 `GetStringUTFChars` 和 `GetByteArrayElements` 这样的函数返回的原始数据指针也不是对象。（它们可以在线程间传递，且直到对应的 Release 被调用都是有效的。）

一种不常见的情况应该另外提一下。如果你通过 `AttachCurrentThread` 附了一个本地层线程，你执行的代码将从不会自动地释放局部引用，直到线程分离。你创建的任何局部引用将不得不手动删除。通常，在循环中创建局部引用的任何本地层代码可能需要执行一些手动删除。

## UTF-8 和 UTF-16 字符串
Java 编程语言使用 UTF-16。为了方便，JNI 也提供方法使用 [改进的 UTF-8](http://en.wikipedia.org/wiki/UTF-8#Modified_UTF-8)。改进的编码对 C 代码很有用，因为它把 `\u0000` 编码为 `0xc0 0x80` 而不是 `0x00`。关于这一点的好处是，您可以依靠具有 C 风格的以零为终止字符的字符串，适合与标准 libc 的字符串函数一起使用。

缺点是你不能传递任意的 UTF-8 数据给 JNI 并期待它能正确工作。

如果可能，操作 UTF-16 字符串通常更快。当前 Android 在 `GetStringChars` 中不需要拷贝，然而 `GetStringUTFChars` 需要分配并转换为 UTF-8。注意 **UTF-16 字符串不是以 0 结尾的，**且允许 \u0000 ，所以你需要根据字符串的长度来访问 jchar 指针。

**不要忘记  `Release`  你 `Get` 的字符串**。字符串函数返回 `jchar*` 或 `jbyte*`，它们都是指向原始数据类型数据的 C 风格指针，而不是局部引用。它们保证有效，直到调用 `Release`，这意味着当本地层方法返回时它们不会释放。

**传递给 `NewStringUTF` 的数据必须是改进的 UTF-8 格式**。一个常见的错误是，从文件或网络流读取字符数据并在无过滤的情况下交给 `NewStringUTF` 处理。除非你知道数据是 7 位的 ASCII，你需要删除高 ASCII 字符或将其转换为正确的改进 UTF-8 格式。如果你没有，UTF-16 转换可能不是你期待的那样。扩展的 JNI 检查将扫描字符串，并就无效数据向你提出警告，但它们不会捕获所有东西。

## 原始数据类型的数组
JNI 提供了访问数组对象内容的函数。尽管每次只能访问一个数组对象的项，但原始数据类型的数组可以直接读或写，就像它们在 C 中声明的一样。

为了使接口尽可能高效且，`Get<PrimitiveType>ArrayElements` 族调用允许运行时返回指向实际元素的指针，或分配一些内存并拷贝一份。无论哪种方式，返回的原始指针保证有效，直到对应的 `Release` 被调用（这表明，如果数据没有拷贝，则数组对象将被固定，并且不能作为压缩堆的一部分重新定位）。**你必须 `Release` 你 `Get` 的每个数组**。此外，如果 `Get` 调用失败，你必须确保你的代码没有在后面试图 `Release` 一个 NULL 指针。

你可以通过传递一个非空指针作为 `isCopy` 参数决定是否拷贝数据。这很少用到。

`Release` 调用接收一个 `mode` 参数，其可以是三个值中的一个。运行时执行的行为依赖于它是返回一个指向实际数据的指针还是实际数据拷贝的指针：

 * `0`
    * 实际数据：数组对象是未固定的。
    * 拷贝：数据被烤回。拷贝的缓冲区被释放。
 * `JNI_COMMIT`
    * 实际数据：什么也不做。
    * 拷贝：数据被烤回。拷贝的缓冲区 **不释放** 。
 * 0
    * 实际数据：数组对象是未固定的。早期写入 **不会** 中止。
    * 拷贝：拷贝的缓冲区被释放；任何修改丢失。
    
检查 `isCopy` 的一个原因是了解在对数组做了修改后你是否需要以 `JNI_COMMIT` 调用 `Release`—— 如果你在改变数组内容和执行使用数组内容的代码之间进行交替，你可能可以跳过无操作提交。另一个检查标记的可能原因是高效的处理 `JNI_ABORT`。比如，你也许想要获得一个数组，修改它，传递一部分给其它函数，然后丢弃修改。如果你知道 JNI 为你创建了一份拷贝，则无需创建另一份“可编辑的”拷贝。如果 JNI 向你传递了原始的，则你确实需要创建你自己的拷贝。
    
一个常见的错误是（在示例代码中重现）假设你可以在 `* isCopy` 为 false 时跳过调用 `Release`。这不是实际的情况。如果没有分配拷贝缓冲区，则原始的内存必须被固定下来，且不能由垃圾收集器移动。
    
还要注意 `JNI_COMMIT` 标记 **不** 释放数组，在最后你将需要以一个不同的标记再次调用 `Release`。

## 区域调用
当你想做的就只是拷入拷出数据，有另外一些像 `Get<Type>ArrayElements` 和 `GetStringChars` 的调用可能非常有用。考虑下面的代码：
```
    jbyte* data = env->GetByteArrayElements(array, NULL);
    if (data != NULL) {
        memcpy(buffer, data, len);
        env->ReleaseByteArrayElements(array, data, JNI_ABORT);
    }
```

获取数组，拷贝前面的 `len ` 字节的元素，然后释放数组。`Get` 调用是固定还是拷贝数组的内容依赖于实现。代码拷贝数据（也许是第二次），然后调用 `Release`；在这种情况下 `JNI_ABORT` 确保没有第三个副本的机会。

可以完成相同事情的更简单的代码如下：
```
    env->GetByteArrayRegion(array, 0, len, buffer);
```

这有几个优势：

 * 需要一个 JNI 调用而不是 2 个，减少了开销。
 * 不需要固定或额外的数据拷贝。
 * 降低了程序员错误的风险 - 没有了在一些失败后忘记调用 `Release` 的风险。

类似地，你可以使用 `Set<Type>ArrayRegion` 调用把数据复制到一个数组，及 `GetStringRegion` 或 `GetStringUTFRegion` 把字符拷出一个 `String`。

## 异常
**你一定不能在异常挂起时调用大多数 JNI 函数**。你的代码需要注意到异常（通过函数的返回值，`ExceptionCheck`，或 `ExceptionOccurred`）并返回，或清除异常并处理它。

在异常挂起时你能调用的 JNI 函数只有下面这些：

 * `DeleteGlobalRef`
 *`DeleteLocalRef`
 * `DeleteWeakGlobalRef`
 * `ExceptionClear`
 * `ExceptionDescribe`
 * `ExceptionOccurred`
 * `MonitorExit`
 * `PopLocalFrame`
 * `PushLocalFrame`
 * `Release<PrimitiveType>ArrayElements`
 * `ReleasePrimitiveArrayCritical`
 * `ReleaseStringChars`
 * `ReleaseStringCritical`
 * `ReleaseStringUTFChars`

许多 JNI 调用可能抛出异常，但也常提供简单的方式用于失败检查。比如，如果 `NewString` 返回非空值，你不需要检查失败。然而，如果你调用了一个方法（使用像 `CallObjectMethod` 这样的函数），你必须总是检查异常，因为如果抛出了异常，返回值将不是有效的。

注意，由解释器代码抛出的异常无法展开本地层栈帧，且 Android 还不支持 C++ 异常。JNI `Throw` 和 `ThrowNew` 指令只是在当前线程中设置一个异常指针。一旦从本地层代码返回受控代码，异常将被注意到并被适当地处理。

本地层代码可以通过调用 `ExceptionCheck` 或 `ExceptionOccurred` “捕获”异常，并通过 `ExceptionClear` 清除它。通常，丢弃异常而不处理它们可能产生一些问题。

没有内置的函数管理 `Throwable` 对象本身，因此如果你想获取异常字符串，你将需要找到 `Throwable` 类，查找 `getMessage "()Ljava/lang/String;"` 的方法 ID，调用它，如果结果非空，则使用 `GetStringUTFChars` 获得一些你可以交给 `printf(3)` 或等价的函数的东西。

## 扩展检查
JNI 执行非常少的错误检查。错误通常导致崩溃。Android 还提供了一个称为 `CheckJNI` 的模式，其中 JavaVM 和 JNIEnv 函数表指针被切换为，在调用标准实现前执行一系列扩展检查的函数的表。

额外的检查包括：

 * 数组：尝试分配一个负值大小的数组。
 * 坏指针：传递一个坏的 jarray/jclass/jobject/jstring 给 JNI 调用，或传递一个空指针作为不能为空的参数给 JNI 调用。
 * 类名：使用任何 “java/lang/String”形式的类名调用 JNI。
 * 临界调用：在一个 “critical”get 和它对应的 release 之间执行一个 JNI 调用。
 * Direct ByteBuffers：给 `NewDirectByteBuffer` 传递坏的参数。
 * 异常：在异常挂起的时候执行 JNI 调用。
 * JNIEnv*s：在错误的线程中使用 JNIEnv*s。
 * jfieldIDs：使用一个空的 jfieldIDs，或使用一个 jfieldID 给字段设置错误类型的值（比如，试图将一个 String 字段赋值为一个 StringBuilder），或使用静态字段的 jfieldID 来设置一个实例字段，反之亦然，或将一个类的 jfieldID 用于另一个类的实例。
  * jmethodIDs：当执行 `Call*Method` JNI 调用时使用了错误种类的 jmethodID：不正确的返回类型，静态/非静态不匹配，错误类型的 ‘this’（对于非静态调用）或错误的类（对于静态调用）。
  * 引用：在错误的引用种类上使用 `DeleteGlobalRef`/`DeleteLocalRef`。
  * 释放模式：传递一个坏的释放模式给释放调用（`0`，`JNI_ABORT`，或 `JNI_COMMIT` 之外的东西）。
  * 类型安全：在你的本地层方法中返回一个不兼容的类型（比如，在一个声明为返回 String 的方法中返回一个 StringBuilder）。
  * UTF-8：给 JNI 调用传递一个无效的 [改进的 UTF-8](http://en.wikipedia.org/wiki/UTF-8#Modified_UTF-8) 字节序列。

（方法和字段的可访问性依然没有检查：访问限制不适用于本地层代码。）

有多种方法启用 CheckJNI。

如果你正在使用模拟器，CheckJNI 是默认开启的。

如果你有一个经过 root 的设备，你可以使用下面的命令启用 CheckJNI 模式重启运行时：
```
adb shell stop
adb shell setprop dalvik.vm.checkjni true
adb shell start
```
 
在所有这些情况中，你将在 logcat 输出中运行时启动时看到像这样的东西：
 ```
 D AndroidRuntime: CheckJNI is ON
 ```
 
如果你有一个普通的设备，你可以使用下面的命令：
```
adb shell setprop debug.checkjni 1
```

这不影响已经运行的应用，但自那之后启动的应用都将开启 CheckJNI。（将属性修改为其它值或简单地重启将再次禁用 CheckJNI。）在这种情况下，你将在你的 logcat 输出中下次启动一个应用时看到像这样的东西：
```
D Late-enabling CheckJNI
```

你还可以在你的应用的 manifest 中设置 `android:debuggable` 属性来只为你的应用开启 CheckJNI。注意 Android 构建工具将自动地为某一构建类型做这些。

## 本地库
你可以通过标准的 `System.loadLibrary` 调用从共享库加载本地层代码。获取你本地层代码的首选方法是：

* 在一个静态的类初始化器里调用 `System.loadLibrary`。（参考前面的例子，其中用于调用 `nativeClassInit` 的那个。）参数是“未修饰的”库名，比如要加载 "libfubar.so"，你应该传递 "fubar"。
* 提供一个本地层函数： `jint JNI_OnLoad(JavaVM* vm, void* reserved)`
* 在 `JNI_OnLoad`，注册你所有的本地层方法。你应该声明那些方法为 "static"，使那些名称不占用设备的符号表空间。

如果用 C++ 写的话，`JNI_OnLoad` 函数看起来应该像下面这样：
```
jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return -1;
    }

    // Get jclass with env->FindClass.
    // Register methods with env->RegisterNatives.

    return JNI_VERSION_1_6;
}
```

你也可以用共享库的完整路径名调用 `System.load`。对于Android 应用，你也许会发现从 context 对象获取应用程序私有数据存储区的完整路径的方法非常有用。

这是建议采用的方法，但不是唯一的方法。无需显式的注册，你也不是必须提供 `JNI_OnLoad` 函数。你可以使用以特殊的方式命名本地层方法的“发现机制”来代替 (详情参看 [JNI spec](http://java.sun.com/javase/6/docs/technotes/guides/jni/spec/design.html#wp615))，尽管这种方法更不尽如人意。如果方法签名错了的话，在方法第一次被实际调用之前，你都将无法获知这种情况。

关于 `JNI_OnLoad` 另一点需要注意的是：你所作出的任何对于 `FindClass` 的调用发生在用于加载共享库的类加载器的上下文。通常，`FindClass` 使用与解释栈顶端的方法相关联的加载器，或者如果没有（由于线程只是被附接的）它使用“system”类加载器。这使得 `JNI_OnLoad` 成为查找和缓存类对象引用的适当场所。

## 64 位注意事项
Android 当前主要运行于 32 位平台。理论上，可以为 64 位系统构建它，但那不是目前的目标。对于大多数部分来说，这不是你在与本地层代码交互时需要担忧的，但如果你计划将指向本地层结构的指针保存在对象的 integer 字段中，它就变得非常重要了。要支持使用 64 位指针的架构，**你需要在 `long` 字段中保存你的本地层指针而不是 `int` 中**。

## 不支持的功能/向后兼容性
所有的 JNI 1.6 功能都支持，以下这些例外：

 * `DefineClass` 没有实现。 Android 不使用 Java 字节码或类文件，因此传递二进制类数据无法工作。
 
 为了与老版本 Android 保持向后兼容性，你可能需要意识到如下这些：
 
   * **本地层函数的动态查找**
直到 Android 2.0 (Eclair)， 在搜索方法名期间 '$' 字符都不被适当地转为 "_00024"。为了绕过这个问题，需要显式地注册或将本地层方法移出内部类。
   * **分离线程**
直到 Android 2.0 (Eclair)， 都无法使用 `pthread_key_create` 析够函数来避免“线程必须在退出前分离”检查。（运行时也使用 pthread key 析够函数，因此查看谁首先被调用将有一个竞态。）
   * **弱全局引用**
直到 Android 2.0 (Eclair)， 弱全局引用都没有实现。更老的版本将直接地拒绝使用它们的尝试。你可以使用 Android 平台版本常量测试对它的支持。
直到 Android 4.0 (Ice Cream Sandwich)，弱全局引用只能传递给 `NewLocalRef`，`NewGlobalRef` 和 `DeleteWeakGlobalRef`。（规范强烈鼓励程序员在通过它们做任何事之前，创建到弱全局引用的硬引用，因此这不应该是限制。）
对于  Android 4.0 (Ice Cream Sandwich)，弱全局引用可以像任何其它 JNI 引用那样使用。
   * **局部引用**
   直到 Android 4.0 (Ice Cream Sandwich)，局部引用实际上是直接指针。Ice Cream Sandwich添加了间接指针以支持更好的垃圾回收，但这意味着很多 JNI 错误在旧版本上是不可检测的。参考 [ICS 中的 JNI 局部引用变化](http://android-developers.blogspot.com/2011/11/jni-local-reference-changes-in-ics.html) 了解更多详情。
   * **通过 `GetObjectRefType` 确定引用类型**
直到 Android 4.0 (Ice Cream Sandwich)，作为使用直接指针的结果（参考上文），正确地实现 `GetObjectRefType` 都是不可能的。相反我们使用一种启发式的方法，按顺序查找弱全局表，参数，局部表，和全局表。它第一次找到你的直接指针，它将报告你的引用具有它恰巧在检测的类型。这意味着，比如，如果你在一个全局 jclass 上调用 `GetObjectRefType`，碰巧与作为隐式参数传递给你的静态本地方法的 jclass 相同，你将获得 `JNILocalRefType` 而不是 `JNIGlobalRefType`。
   
## FAQ：为什么我遇到了 `UnsatisfiedLinkError`？
当你使用本地层代码时，像下面这样的失败比较常见：
```
java.lang.UnsatisfiedLinkError: Library foo not found
```
在某些情况下它的含义就像它说的那样 - 库找不到。在其它情况中，库存在但是无法被 `dlopen(3)`打开，失败的详情可以在异常的细节消息中找到。

你可能遇到 "library not found" 异常的常见原因如下：

 * 库不存在或应用无法访问。使用 `adb shell ls -l <path>` 检查它是否存在，以及权限。
 * 库不是用 NDK 构建的。这可能导致依赖的函数或库在设备上不存在。
 
 另一种类的 `UnsatisfiedLinkError` 失败看上去像这样：
 ```
 java.lang.UnsatisfiedLinkError: myfunc
        at Foo.myfunc(Native Method)
        at Foo.main(Foo.java:10)
 ```
 在 logcat 中，你将看到：
 ```
 W/dalvikvm(  880): No implementation found for native LFoo;.myfunc ()V
 ```
 这意味着运行时试图找到一个匹配的方法，但未成功。这种情况一些常见的原因如下：
 
  * 库没有加载。检查 logcat 关于库加载的输出消息。
  * 由于名字或签名不匹配，方法没有找到。这通常由于一下原因引起：
     * 对于延迟方法查找，以 `extern "C"` 和适当的可见性 (`JNIEXPORT`) 声明 C++ 函数失败。注意，在 Ice Cream Sandwich 之前，JNIEXPORT 宏是不正确的，因此以老的 `jni.h` 使用新的 GCC 无法工作。你可以使用 `arm-eabi-nm` 查看库中出现的符号；如果它们看起来是修饰过的（比如 `_Z15Java_Foo_myfuncP7_JNIEnvP7_jclass` 而不是 `Java_Foo_myfunc`），或者如果符号类型是小写的 't' 而不是大写的 'T'，则你需要调整声明。
     * 对于显式的注册，输入方法签名时的小错误。确保传递给注册调用的东西与日志文件中的签名匹配。记得 'B' 是 `byte` 且 'Z' 是 `boolean`。签名中的类名组件以 'L' 开头，以 ';' 结尾，使用 '/' 分割包/类名，并使用 '$' 分割内部类名字（比如，`Ljava/util/Map$Entry;`）。

使用 `javah` 自动地生成 JNI 头部也许对避免一些错误有帮助。

## 为什么 FindClass 找不到我的类？
（这个建议的大部分等价地适用于通过 `GetMethodID` 或 `GetStaticMethodID` 查找方法，或通过 `GetFieldID` 或 `GetStaticFieldID` 查找字段的失败。）

确保类名字符串具有正确的格式。JNI 类名以包名开始，且有斜线分割，比如 `java/lang/String`。如果你在查找一个数组类，你需要以适当数量的方括号开始，且还必须以 'L' 和 ';' 包裹类，因此一维的 `String` 数组将是 `[Ljava/lang/String;`。如果你在查找一个内部类，使用 '$' 而不是 ','。通常，在 .class 文件上使用 `javap` 是找到你的类的内部名字的一种好方法。

如果你在使用 ProGuard，请确保 [ProGuard 没有剥去你的类](https://developer.android.com/tools/help/proguard.html#configuring)。这可能在你的类/方法/字段只有 JNI 使用时发生。

如果类名称正确，则可能遇到了类加载器问题。`FindClass` 想要在与你的代码关联的类加载器中开始类搜索。如果检查调用栈，它将看起来像这样：
```
    Foo.myfunc(Native Method)
    Foo.main(Foo.java:10)
```

最上面的方法是 `Foo.myfunc`。`FindClass` 查找与 `Foo` 类关联的 `ClassLoader` 对象并使用它。

这通常执行了你想要的。如果您自己创建一个线程（可能通过调用`pthread_create`，然后使用 `AttachCurrentThread` 连接），您可能会遇到麻烦。现在没有你的应用的栈帧。如果你在这个线程中调用 `FindClass`，JavaVM 将在 "system" 类加载器中启动而不是与你的应用关联的那个，因此尝试查找应用特有的类将失败。

有一些方法绕过这个问题：
  * 在 `JNI_OnLoad` 中执行你的 `FindClass` 一次，并缓存类引用以备后用。任何作为执行 `JNI_OnLoad` 的一部分对 `FindClass` 所做的调用将使用与调用 `System.loadLibrary` 的函数关联的类加载器（这是一个特殊的规则，用来使库初始化更方便）。如果你的应用代码正在加载库，`FindClass` 将使用正确的类加载器。
  * 给需要的函数传递一个类的实例，通过声明你的本地层方法接收一个 Class 参数，并传入 `Foo.class`。
  * 在方便的地方缓存 `ClassLoader` 对象的引用，然后直接触发 `loadClass` 调用。这需要一些力气。

## FAQ：我如何与本地层代码共享原始数据

你可能发现你自己需要同时在 Java 代码和本地层代码中访问一个巨大的原始数据缓冲区。常见的例子包括管理 bitmaps 和声音采样。有两个基本的方法。

你可以把数据存储在 `byte[]` 中。这允许在 Java 代码中非常快速的访问。在本地层代码中，然而，无法保证在不复制数据的情况下能够访问数据。在一些实现中， `GetByteArrayElements` 和 `GetPrimitiveArrayCritical` 将返回指向 Java 堆中的原始数据的实际指针，但在其它实现中，它将在本地层堆上分配一块缓冲区并复制数据。

另一种方法是把数据存储进直接字节缓冲区。这些可以用 `java.nio.ByteBuffer.allocateDirect` ，或JNI `NewDirectByteBuffer` 函数创建。不像普通的字节缓冲区，不在 Java 堆上分配存储，且总是可以在本地层代码中直接访问（通过 `GetDirectBufferAddress` 获得地址）。依赖于直接字节缓冲区访问如何实现，在 Java 代码中访问可能非常慢。

选择使用哪种依赖于两个因素：

  * 大多数的数据访问是发生在 Java 代码中还是 C/C++ 代码中？
  * 如果数据被最终传递给一个系统 API，它的形式必须是什么？（比如，如果数据最终被传递给接收 byte[] 的函数，在一个直接 `ByteBuffer` 中执行处理可能是不明智的。）

如果没有清晰的赢家，使用直接字节缓冲区。对它们的支持是直接内建在 JNI 中的，且在未来的版本中性能应该有提升。

### [打赏](https://www.wolfcstech.com/about/donate.html)

[原文](https://developer.android.com/training/articles/perf-jni.html)