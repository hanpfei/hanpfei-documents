---
title: chromium net到android平台的移植
---

Chromium net是chromium浏览器及ChromeOS中，用于从网络获取资源的模块。这个网络库是用C++编写的，且用了大量的C++11特性。它广泛地支持当前互联网环境中用到的大量的网络协议，如HTTP/1.1,SPDY，HTTP/2，FTP，QUIC，WebSockets等；在安全性方面也有良好的支持，如SSL等；同时，针对性能，它也有诸多的优化，如引入libevent的基于事件驱动的设计。从而，将chromium net库移植到android平台上并用起来，以提升我们的APP的网络访问性能成为了一件非常有价值的事情。

<!--more-->

这里我们就尝试将chromium net移植到android平台上并用起来。但由于庞大的chromium项目有着自己比较特别的开发流程，开发方式，而给我们的移植工作制造了一些障碍。具体地说是：

 - gn/gyp + ninja的构建方式；
 - 为了方便开发，chromium的仓库中包含了构建所需的完整的工具链，包括编译、链接的工具，android的SDK和NDK，甚至是STL。这个工具链与我们惯常用在项目中的工具链多少还是有一点点区别的，这会给我们的编译链接等制造一些困扰。

但好在Google出品比较具有hack风，因而移植工作的难度变得可控。比如gn工具，可以让我们比较方便地对工程做更多的探索，我们可以通过这个工具来了解我们编译一个模块时，整个环境的配置，该模块具体包含了哪些源文件，模块导出的头文件，编译链接时都用了什么样的参数等等。

在具体移植开始之前，先设定我们移植的目标：

 - 让chromium的net模块可以单独地在android平台上跑起来，可以通过JNI的方式进行调用；
 - 将chromium net模块从整个的chromium代码库中抽离，依然基于gn + ninja，建立独立的开发环境，可以进行配置、构建，以方便后续针对chromium net的开发。
 - 针对与移动端上受限的资源，及项目的需要，对chromium net进行一些裁剪瘦身，比如对ftp协议的支持就没有太大的必要，砍掉这些用不到的东西以节省资源，提升效能。

这里主要就基于我们的目标，来探索移植的方法。本文尝试逐步的探索整个的移植过程。本文会记录遇到的每个问题，并给出出错的提示，比如编译器报的error，链接器报的error，运行时崩溃抓到的backtrace等等，然后还会给出对问题的基本分析与定位，及问题的解决方法。
# 编译net模块
首先要做的事情就是下载完整的chromium代码，这可以参考[Chromium Android编译指南](http://www.jianshu.com/p/5fce18cbe016)来完成。然后执行（假设当前目录是chromium代码库的根目录）命令：
```
$ gclient runhooks
$ cd src
$ ./build/install-build-deps.sh
$ ./build/install-build-deps-android.sh
```
下载构建chromium所需的系统工具。

之后需要对编译进行配置。编辑`out/Default/args.gn`文件，并参照[Chromium Android编译指南](http://www.jianshu.com/p/5fce18cbe016)中的说明，输入如下内容：
```
target_os = "android"
target_cpu = "arm"  # (default)
is_debug = true  # (default)

# Other args you may want to set:
is_component_build = true
is_clang = true
symbol_level = 1  # Faster build with fewer symbols. -g1 rather than -g2
enable_incremental_javac = true  # Much faster; experimental
```
保存退出之后，执行：
```
$ gn gen out/Default
```
产生ninja构建所需的各个模块的ninja文件。随后输入如下命令编译net模块：
```
$ ninja -C out/Default net
```
这个命令会编译net模块，及其依赖的所有模块，包括base，crypto，borringssl，protobuf，icu，url等。可以看一下我们编译的成果：
```
$ ls -alh out/Default/ | grep so$
-rwxrwxr-x  1 chrome chrome 1.2M 8月  10 20:55 libbase.cr.so
-rwxrwxr-x  1 chrome chrome  98K 8月  10 20:55 libbase_i18n.cr.so
-rwxrwxr-x  1 chrome chrome 773K 8月  10 20:54 libboringssl.cr.so
-rwxrwxr-x  1 chrome chrome  74K 8月  10 20:55 libcrcrypto.cr.so
-rwxrwxr-x  1 chrome chrome 1.3M 8月  10 20:55 libicui18n.cr.so
-rwxrwxr-x  1 chrome chrome 902K 8月  10 20:55 libicuuc.cr.so
-rwxrwxr-x  1 chrome chrome 4.1M 8月  10 20:59 libnet.cr.so
-rwxrwxr-x  1 chrome chrome 122K 8月  10 20:55 libprefs.cr.so
-rwxrwxr-x  1 chrome chrome 182K 8月  10 20:55 libprotobuf_lite.cr.so
-rwxrwxr-x  1 chrome chrome  90K 8月  10 20:59 liburl.cr.so
```
总共10个共享库文件。

在我们的工程的app模块的jni目录下为chromium创建文件夹`app/src/main/jni/third_party/chromium/libs`和`app/src/main/jni/third_party/chromium/include`，分别用于存放我们编译出来的共享库文件和net等模块导出的头文件及这些头文件include的其它头文件。

这里我们将编译出来的所有so文件拷贝到`app/src/main/jni/third_party/chromium/libs/armeabi`和`app/src/main/jni/third_party/chromium/libs/armeabi-v7a`目录下：
```
cp out/Default/*.so /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/libs/armeabi/
cp out/Default/*.so /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/libs/armeabi-v7a/
```
编译共享库**似乎**挺顺利。
# 提取导出头文件
为了使用net模块提供的API，我们不可避免地要将net导出的头文件引入我们的项目。要做到这些，我们首先就需要从chromium工程中提取net导出的头文件。不像许多其它的C++项目，源代码文件、私有头文件及导出头文件存放的位置被很好地做了区隔，chromium各个模块的所有头文件和源代码文件都是放在一起的。这还是给我们提取导出头文件的工作带来了一点麻烦。

这里我们借助于gn工具提供的desc功能（关于gn工具的用法，可以参考[GN的使用 - GN工具](http://my.oschina.net/wolfcs/blog/726696)一文），输出中如下的这两段：
```
$ gn desc out/Default/ net
Target //net:net
Type: shared_library
Toolchain: //build/toolchain/android:arm

sources
  //net/base/address_family.cc
  //net/base/address_family.h
......

public
  [All headers listed in the sources are public.]
```
编写脚本来实现。

我们可以传入***[chromium代码库的src目录路径]***，***[输出目录的路径]***，***[模块名]***，及***[保存头文件的目标目录路径]***作为参数，来提取模块的所有导出头文件，***[保存头文件的目标目录路径]***参数缺失时默认使用当前目录，比如：
```
$ cd /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include
$ chromium_mod_headers_extracter.py ~/data/chromium_android/src  out/Default net
```
这里一并将该脚本的完整内容贴出来：
```
#!/usr/bin/env python

import os
import shutil
import sys

def print_usage_and_exit():
    print sys.argv[0] + " [chromium_src_root]" + "[out_dir]" + " [target_name]" + " [targetroot]"
    exit(1)

def copy_file(src_file_path, target_file_path):
    if os.path.exists(target_file_path):
        return
    if not os.path.exists(src_file_path):
        return
    target_dir_path = os.path.dirname(target_file_path)
    if not os.path.exists(target_dir_path):
        os.makedirs(target_dir_path)

    shutil.copy(src_file_path, target_dir_path)

def copy_all_files(source_dir, all_files, target_dir):
    for one_file in all_files:
        source_path = source_dir + os.path.sep + one_file
        target_path = target_dir + os.path.sep + one_file
        copy_file(source_path, target_path)

if __name__ == "__main__":
    if len(sys.argv) < 4 or len(sys.argv) > 5:
        print_usage_and_exit()
    chromium_src_root = sys.argv[1]
    out_dir = sys.argv[2]
    target_name = sys.argv[3]
    target_root_path = "."
    if len(sys.argv) == 5:
        target_root_path = sys.argv[4]
    target_root_path = os.path.abspath(target_root_path)

    os.chdir(chromium_src_root)

    cmd = "gn desc " + out_dir + " " + target_name
    outputs = os.popen(cmd).readlines()
    source_start = False
    all_headers = []

    public_start = False
    public_headers = []

    for output_line in outputs:
        output_line = output_line.strip()
        if output_line.startswith("sources"):
            source_start = True
            continue
        elif source_start and len(output_line) == 0:
            source_start = False
            continue
        elif source_start and output_line.endswith(".h"):
            output_line = output_line[1:]
            all_headers.append(output_line)
        elif output_line == "public":
            public_start = True
            continue
        elif public_start and len(output_line) == 0:
            public_start = False
            continue
        elif public_start:
            public_headers.append(output_line)

    if len(public_headers) == 1:
        public_headers = all_headers
    if len(public_headers) > 1:
        copy_all_files(chromium_src_root, public_headers, target_dir=target_root_path)
```
# Chromium net的简单使用
参照`chromium/src/net/tools/get_server_time/get_server_time.cc`的代码，来编写简单的示例程序。首先是JNI（JNI的用法，可以参考[android app中使用JNI](http://my.oschina.net/wolfcs/blog/111309)）的Java层代码：
```
package com.example.hanpfei0306.myapplication;

public class NetUtils {
    static {
        System.loadLibrary("neteasenet");
    }
    private static native void nativeSendRequest(String url);

    public static void sendRequest(String url) {
        nativeSendRequest(url);
    }
}
```
然后是native层的JNI代码，`MyApplication/app/src/main/jni/src/NetJni.cpp`：
```
//
// Created by hanpfei0306 on 16-8-4.
//

#include <stdio.h>
#include <net/base/network_delegate_impl.h>

#include "jni.h"

#include "base/at_exit.h"
#include "base/json/json_writer.h"
#include "base/message_loop/message_loop.h"
#include "base/memory/ptr_util.h"
#include "base/run_loop.h"
#include "base/values.h"
#include "net/http/http_response_headers.h"
#include "net/proxy/proxy_config_service_fixed.h"
#include "net/url_request/url_fetcher.h"
#include "net/url_request/url_fetcher_delegate.h"
#include "net/url_request/url_request_context.h"
#include "net/url_request/url_request_context_builder.h"
#include "net/url_request/url_request_context_getter.h"
#include "net/url_request/url_request.h"

#include "JNIHelper.h"

#define TAG "NetUtils"

// Simply quits the current message loop when finished.  Used to make
// URLFetcher synchronous.
class QuitDelegate : public net::URLFetcherDelegate {
public:
    QuitDelegate() {}

    ~QuitDelegate() override {}

    // net::URLFetcherDelegate implementation.
    void OnURLFetchComplete(const net::URLFetcher* source) override {
        LOGE("OnURLFetchComplete");
        base::MessageLoop::current()->QuitWhenIdle();
        int responseCode = source->GetResponseCode();

        const net::URLRequestStatus status = source->GetStatus();
        if (status.status() != net::URLRequestStatus::SUCCESS) {
            LOGW("Request failed with error code: %s", net::ErrorToString(status.error()).c_str());
            return;
        }

        const net::HttpResponseHeaders* const headers = source->GetResponseHeaders();
        if (!headers) {
            LOGW("Response does not have any headers");
            return;
        }
        size_t iter = 0;
        std::string header_name;
        std::string date_header;
        while (headers->EnumerateHeaderLines(&iter, &header_name, &date_header)) {
            LOGW("Got %s header: %s\n", header_name.c_str(), date_header.c_str());
        }

        std::string responseStr;
        if(!source->GetResponseAsString(&responseStr)) {
            LOGW("Get response as string failed!");
        }

        LOGI("Content len = %lld, response code = %d, response = %s",
             source->GetReceivedResponseContentLength(),
             source->GetResponseCode(),
             responseStr.c_str());
    }

    void OnURLFetchDownloadProgress(const net::URLFetcher* source,
                                    int64_t current,
                                    int64_t total) override {
        LOGE("OnURLFetchDownloadProgress");
    }

    void OnURLFetchUploadProgress(const net::URLFetcher* source,
                                  int64_t current,
                                  int64_t total) override {
        LOGE("OnURLFetchUploadProgress");
    }

private:
    DISALLOW_COPY_AND_ASSIGN(QuitDelegate);
};

// NetLog::ThreadSafeObserver implementation that simply prints events
// to the logs.
class PrintingLogObserver : public net::NetLog::ThreadSafeObserver {
public:
    PrintingLogObserver() {}

    ~PrintingLogObserver() override {
        // This is guaranteed to be safe as this program is single threaded.
        net_log()->DeprecatedRemoveObserver(this);
    }

    // NetLog::ThreadSafeObserver implementation:
    void OnAddEntry(const net::NetLog::Entry& entry) override {
        // The log level of the entry is unknown, so just assume it maps
        // to VLOG(1).
        const char* const source_type = net::NetLog::SourceTypeToString(entry.source().type);
        const char* const event_type = net::NetLog::EventTypeToString(entry.type());
        const char* const event_phase = net::NetLog::EventPhaseToString(entry.phase());
        std::unique_ptr<base::Value> params(entry.ParametersToValue());
        std::string params_str;
        if (params.get()) {
            base::JSONWriter::Write(*params, &params_str);
            params_str.insert(0, ": ");
        }
#ifdef DEBUG_ALL
        LOGI("source_type = %s (id = %u): entry_type = %s : event_phase = %s params_str = %s",
             source_type, entry.source().id, event_type, event_phase, params_str.c_str());
#endif
    }

private:
    DISALLOW_COPY_AND_ASSIGN(PrintingLogObserver);
};

// Builds a URLRequestContext assuming there's only a single loop.
static std::unique_ptr<net::URLRequestContext> BuildURLRequestContext(net::NetLog *net_log) {
    net::URLRequestContextBuilder builder;
    builder.set_net_log(net_log);
//#if defined(OS_LINUX)
    // On Linux, use a fixed ProxyConfigService, since the default one
  // depends on glib.
  //
  // TODO(akalin): Remove this once http://crbug.com/146421 is fixed.
  builder.set_proxy_config_service(
          base::WrapUnique(new net::ProxyConfigServiceFixed(net::ProxyConfig())));
//#endif
    std::unique_ptr<net::URLRequestContext> context(builder.Build());
    context->set_net_log(net_log);
    return context;
}

static void NetUtils_nativeSendRequest(JNIEnv* env, jclass, jstring javaUrl) {
    const char* native_url = env->GetStringUTFChars(javaUrl, NULL);
    LOGW("Url: %s", native_url);
    base::AtExitManager exit_manager;
    LOGW("Url: %s", native_url);

    GURL url(native_url);
    if (!url.is_valid() || (url.scheme() != "http" && url.scheme() != "https")) {
        LOGW("Not valid url: %s", native_url);
        return;
    }
    LOGW("Url: %s", native_url);

    base::MessageLoopForIO main_loop;

    QuitDelegate delegate;
    std::unique_ptr<net::URLFetcher> fetcher =
            net::URLFetcher::Create(url, net::URLFetcher::GET, &delegate);

    net::NetLog *net_log = nullptr;
#ifdef DEBUG_ALL
    net_log = new net::NetLog;
    PrintingLogObserver printing_log_observer;
    net_log->DeprecatedAddObserver(&printing_log_observer,
                                  net::NetLogCaptureMode::IncludeSocketBytes());
#endif

    std::unique_ptr<net::URLRequestContext> url_request_context(BuildURLRequestContext(net_log));
    fetcher->SetRequestContext(
            // Since there's only a single thread, there's no need to worry
            // about when the URLRequestContext gets created.
            // The URLFetcher will take a reference on the object, and hence
            // implicitly take ownership.
            new net::TrivialURLRequestContextGetter(url_request_context.get(),
                                                    main_loop.task_runner()));
    fetcher->Start();
    // |delegate| quits |main_loop| when the request is done.
    main_loop.Run();

    env->ReleaseStringUTFChars(javaUrl, native_url);
}

int jniRegisterNativeMethods(JNIEnv* env, const char *classPathName, JNINativeMethod *nativeMethods, jint nMethods) {
    jclass clazz;
    clazz = env->FindClass(classPathName);
    if (clazz == NULL) {
        return JNI_FALSE;
    }
    if (env->RegisterNatives(clazz, nativeMethods, nMethods) < 0) {
        return JNI_FALSE;
    }
    return JNI_TRUE;
}

static JNINativeMethod gNetUtilsMethods[] = {
        NATIVE_METHOD(NetUtils, nativeSendRequest, "(Ljava/lang/String;)V"),
};

void register_com_netease_volleydemo_NetUtils(JNIEnv* env) {
    jniRegisterNativeMethods(env, "com/example/hanpfei0306/myapplication/NetUtils",
                             gNetUtilsMethods, NELEM(gNetUtilsMethods));
}

// DalvikVM calls this on startup, so we can statically register all our native methods.
jint JNI_OnLoad(JavaVM* vm, void*) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        LOGE("JavaVM::GetEnv() failed");
        abort();
    }

    register_com_netease_volleydemo_NetUtils(env);
    return JNI_VERSION_1_6;
}
```
这个文件里，在nativeSendRequest()函数中调用chromium net做了网络请求，获取响应，并打印出响应的headers及响应的content。

`MyApplication/app/src/main/jni/src/JNIHelper.h`中定义了一些宏，以方便native methods的注册，其具体内容则为：
```
#ifndef CANDYWEBCACHE_JNIHELPER_H
#define CANDYWEBCACHE_JNIHELPER_H

#include <android/log.h>

#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG,TAG ,__VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,TAG ,__VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,TAG ,__VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,TAG ,__VA_ARGS__)
#define LOGF(...) __android_log_print(ANDROID_LOG_FATAL,TAG ,__VA_ARGS__)

#ifndef NELEM
# define NELEM(x) ((int) (sizeof(x) / sizeof((x)[0])))
#endif

#define NATIVE_METHOD(className, functionName, signature) \
    { #functionName, signature, reinterpret_cast<void*>(className ## _ ## functionName) }


#endif //CANDYWEBCACHE_JNIHELPER_H
```
# 配置Gradle
要在Android Studio中使用JNI，还需要对Gralde做一些特别的配置，具体可以参考[在Android Studio中使用NDK/JNI - 实验版插件用户指南](http://my.oschina.net/wolfcs/blog/550677)一文。这里需要对***`MyApplication/build.gradle`***、***`MyApplication/gradle/wrapper/gradle-wrapper.properties`***，和***`MyApplication/app/build.gradle`***这几个文件做修改。

修改***`MyApplication/build.gradle`***文件，最终的内容为：
```
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle-experimental:0.7.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```
在这个文件中配置gradle插件的版本为**`gradle-experimental:0.7.0`**。

修改***`MyApplication/gradle/wrapper/gradle-wrapper.properties`***文件，最终的内容为：
```
#Mon Dec 28 10:00:20 PST 2015
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-2.10-all.zip
```
在这个文件中配置gradle的版本。

修改***`MyApplication/app/build.gradle`***文件，最终的内容为：
```
apply plugin: 'com.android.model.application'

model {
    repositories {
        libs(PrebuiltLibraries) {
            chromium_net {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/libnet.cr.so")
                }
            }
        }
    }

    android {
        compileSdkVersion 23
        buildToolsVersion "23.0.3"

        defaultConfig {
            applicationId "com.example.hanpfei0306.myapplication"
            minSdkVersion.apiLevel 19
            targetSdkVersion.apiLevel 21
            versionCode 1
            versionName "1.0"
        }

        ndk {
            moduleName "neteasenet"

            CFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include/'),])

            cppFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include'),])

            ldLibs.add("android")
            ldLibs.add("log")
            ldLibs.add("z")
        }

        sources {
            main {
                java {
                    source {
                        srcDir "src/main/java"
                    }
                }
                jni {
                    source {
                        srcDirs = ["src/main/jni",]
                    }
                    dependencies {
                        library 'chromium_net' linkage 'shared'
                    }
                }
                jniLibs {
                    source {
                        srcDirs =["src/main/jni/third_party/chromium/libs",]
                    }
                }
            }
        }


        buildTypes {
            debug {
                ndk {
                    abiFilters.add("armeabi")
                    abiFilters.add("armeabi-v7a")
                }
            }
            release {
                minifyEnabled false
                proguardFiles.add(file("proguard-rules.pro"))
                ndk {
                    abiFilters.add("armeabi")
                    abiFilters.add("armeabi-v7a")
                }
            }
        }
    }
    android.packagingOptions {
        exclude 'META-INF/INDEX.LIST'
        exclude 'META-INF/io.netty.versions.properties'
    }
}
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'

    compile 'com.jcraft:jzlib:1.1.2'
    compile 'com.android.support:appcompat-v7:23.4.0'
}
```
我们在这里对native代码的整个编译、链接做配置。这包括，编译要包含哪些源文件，编译时头文件的搜索路径，我们的native代码依赖的要链接的系统共享库/静态库，我们的native代码依赖的要链接的预编译的共享库/静态库，比如我们的chromium net等。配置编译出来的共享库的文件名。同时还要为`buildTypes`配置`abiFilters`，以防止构建系统尝试为其它我们不打算支持的ABI，如arm64-v8a、mips、mips64、x86和x86_64，构建共享库，而发生找不到响应的chromium net的so文件的错误。
# 应用工程编译
做了上面的配置之后，我们怀着激动的心情，小心翼翼地点击“Rebuild Project”菜单项，并期待奇迹发生。编译过程启动之后，很快就终止了，报出了如下的error：
```
:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp
Information:(6) (Unknown) In file included
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h
Error:(10, 35) base/strings/string16.h: No such file or directory
compilation terminated.
Error:Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt
Information:BUILD FAILED
Information:Total time: 4.239 secs
Information:2 errors
Information:0 warnings
Information:See complete output in console
```
提示找不到base模块的头文件**`base/strings/string16.h`**。base模块是chromium项目中大多数模块都会依赖的模块，它提供了许多基本的功能，比如字符串，文件操作，消息队列MessageQueue，内存管理的一些操作等等。net模块的头文件include了base的头文件的地方还不少，而在我们自己的JNI native层代码中，也难免要引用到base定义的一些组件，因而我们需要将base模块导出的头文件一并引入我们的工程。提取base的头文件并引入我们的工程：
```
$ cd /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include
$ chromium_mod_headers_extracter.py ~/data/chromium_android/src  out/Default base
```
## 配置stl：
引入了base模块的头文件之后，再次编译我们的应用工程。又报error了：
```
:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h
Information:(10) (Unknown) In file included
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/strings/string16.h
Error:(33, 22) functional: No such file or directory
compilation terminated.
Error:Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt
Information:BUILD FAILED
Information:Total time: 10.702 secs
Information:2 errors
Information:0 warnings
Information:See complete output in console
```
这里是提示找不到STL的头文件。在MyApplication/app/build.gradle中配置stl：
```
            stl "gnustl_static"
```
## 引入chromium的build配置头文件
配置了stl之后，再次编译我们的应用工程。接着报error：
```
18:52:21: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
In file included from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:10:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/strings/string16.h:37:32: fatal error: build/build_config.h: No such file or directory
 #include "build/build_config.h"
                                ^
compilation terminated.

:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 2.773 secs
C++ compiler failed while compiling NetJni.cpp.
18:52:24: External task execution finished 'assembleDebug'.
```
提示找不到"build/build_config.h"文件。这个文件定义了一些全局性的宏，以对编译做一些全局的控制。这次直接将chromium代码库中的对应文件拷贝过来。

## 引入遗漏的头文件
再次编译我们的应用工程。接着报error：
```
18:55:46: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
In file included from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/completion_callback.h:10:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback.h:8:35: fatal error: base/callback_forward.h: No such file or directory
 #include "base/callback_forward.h"
                                   ^
compilation terminated.

:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 0.663 secs
C++ compiler failed while compiling NetJni.cpp.
18:55:47: External task execution finished 'assembleDebug'.
```
提示找不到base模块的头文件**`base/callback_forward.h`**。直接将chromium代码库中的对应文件拷贝过来。我们似乎是被`gn desc`的输出欺骗了，它似乎并没有将一个模块导出的所有的头文件都告诉我们。像 **`base/callback_forward.h`** 一样，逃过了` gn desc` 的眼睛，同时也逃过了我们提取模块头文件的脚本的眼睛的头文件还有如下的这些：
```
base/message_loop/timer_slack.h
base/files/file.h
net/cert/cert_status_flags_list.h
net/cert/cert_type.h
net/base/privacy_mode.h
net/websockets/websocket_event_interface.h
net/quic/quic_alarm_factory.h
```
这里我们就不将缺少这些文件导致的错误的错误输出列出来了，内容与前面看到的缺少**`base/callback_forward.h`**文件导致的错误的错误输出类似，解决的方法也相同。

算下来，我们提取模块的头文件的脚本遗漏了8个头文件。不能不让人感慨，移植这个事情真是个体力活啊。不过还好遗漏的不是80个文件，或800个，要不然真是要把人逼疯了。

## 引入url模块的导出头文件
引入了那些遗漏的头文件之后，再次编译。继续报错：
```
18:57:46: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
In file included from /home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/atomic:38:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomicops_internals_portable.h:35,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomicops.h:152,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomic_ref_count.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback_internal.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback.h:9,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/completion_callback.h:10,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/c++0x_warning.h:32:2: error: #error This file requires compiler and library support for the ISO C++ 2011 standard. This support is currently experimental, and must be enabled with the -std=c++11 or -std=gnu++11 compiler options.
 #error This file requires compiler and library support for the \
  ^
In file included from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate.h:15:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:12,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/auth.h:13:24: fatal error: url/origin.h: No such file or directory
 #include "url/origin.h"
                        ^
compilation terminated.

:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 0.722 secs
C++ compiler failed while compiling NetJni.cpp.
18:57:47: External task execution finished 'assembleDebug'.
```
这次是提示找不到url模块的头文件**`url/origin.h`**。url模块与我们要用的net联系紧密，同样不可避免地要在我们的native层代码中引用到。因而我们要将url模块导出的头文件一并引入我们的工程。提取url的导出头文件并引入我们的工程：
```
$ cd /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include
$ chromium_mod_headers_extracter.py ~/data/chromium_android/src  out/Default url
```
# 注释掉gtest相关代码
引入了url模块的导出头文件之后，再次编译。继续报错：
```
19:00:48: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
In file included from /home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/atomic:38:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomicops_internals_portable.h:35,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomicops.h:152,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomic_ref_count.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback_internal.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback.h:9,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/completion_callback.h:10,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/c++0x_warning.h:32:2: error: #error This file requires compiler and library support for the ISO C++ 2011 standard. This support is currently experimental, and must be enabled with the -std=c++11 or -std=gnu++11 compiler options.
 #error This file requires compiler and library support for the \
  ^
In file included from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/cookies/canonical_cookie.h:12:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate.h:17,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:12,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/gtest_prod_util.h:8:52: fatal error: testing/gtest/include/gtest/gtest_prod.h: No such file or directory
 #include "testing/gtest/include/gtest/gtest_prod.h"
                                                    ^
compilation terminated.

:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 0.66 secs
C++ compiler failed while compiling NetJni.cpp.
19:00:49: External task execution finished 'assembleDebug'.
```
这一次是提示找不到gtest的头文件。暂时我们还无需借助于gtest来做单元测试，因而这个include显得没有必要。我们将**`MyApplication/app/src/main/jni/third_party/chromium/include/base/gtest_prod_util.h`**文件中对**`"testing/gtest/include/gtest/gtest_prod.h"`**的include注释掉，同时修改**`FRIEND_TEST_ALL_PREFIXES`**宏的定义：
```
#if 0
#define FRIEND_TEST_ALL_PREFIXES(test_case_name, test_name) \
  FRIEND_TEST(test_case_name, test_name); \
  FRIEND_TEST(test_case_name, DISABLED_##test_name); \
  FRIEND_TEST(test_case_name, FLAKY_##test_name)
#else
#define FRIEND_TEST_ALL_PREFIXES(test_case_name, test_name)
#endif
```
这样就可以注释掉类定义中专门为gtest插入的那些代码了。
## 配置C++11
再次编译，继续报错：
```
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:166:5: error: 'unique_ptr' is not a member of 'std'
     std::unique_ptr<net::URLRequestContext> url_request_context(BuildURLRequestContext(net_log));
     ^
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:166:43: error: expected primary-expression before '>' token
     std::unique_ptr<net::URLRequestContext> url_request_context(BuildURLRequestContext(net_log));
                                           ^
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:166:95: error: 'BuildURLRequestContext' was not declared in this scope
     std::unique_ptr<net::URLRequestContext> url_request_context(BuildURLRequestContext(net_log));
                                                                                               ^
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:166:96: error: 'url_request_context' was not declared in this scope
     std::unique_ptr<net::URLRequestContext> url_request_context(BuildURLRequestContext(net_log));
                                                                                                ^
In file included from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/files/file_path.h:113:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/files/file.h:13,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/net_errors.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/proxy/proxy_config_service_fixed.h:9,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:16:
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/containers/hash_tables.h: In instantiation of 'std::size_t base_hash::hash<T>::operator()(const T&) const [with T = std::basic_string<char>; std::size_t = unsigned int]':
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/files/file_path.h:473:56:   required from here
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/containers/hash_tables.h:28:77: error: no match for call to '(std::hash) (const std::basic_string<char>&)'
   std::size_t operator()(const T& value) const { return std::hash<T>()(value); }
                                                                             ^
In file included from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:10:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/strings/string16.h:194:8: note: candidate is:
 struct hash<base::string16> {
        ^
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/strings/string16.h:195:15: note: std::size_t std::hash::operator()(const string16&) const
   std::size_t operator()(const base::string16& s) const {
               ^
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/strings/string16.h:195:15: note:   no known conversion for argument 1 from 'const std::basic_string<char>' to 'const string16& {aka const std::basic_string<short unsigned int, base::string16_char_traits>&}'
In file included from /home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/atomic:41:0,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomicops_internals_portable.h:35,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomicops.h:152,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/atomic_ref_count.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback_internal.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/callback.h:9,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/completion_callback.h:10,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/net/base/network_delegate_impl.h:11,
                 from /media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:6:
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/atomic_base.h: At global scope:
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/atomic_base.h:588:7: warning: inline function 'bool std::__atomic_base<_IntTp>::compare_exchange_strong(std::__atomic_base<_IntTp>::__int_type&, std::__atomic_base<_IntTp>::__int_type, std::memory_order, std::memory_order) volatile [with _ITp = int; std::__atomic_base<_IntTp>::__int_type = int; std::memory_order = std::memory_order]' used but never defined
       compare_exchange_strong(__int_type& __i1, __int_type __i2,
       ^
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/atomic_base.h:525:7: warning: inline function 'std::__atomic_base<_IntTp>::__int_type std::__atomic_base<_IntTp>::exchange(std::__atomic_base<_IntTp>::__int_type, std::memory_order) volatile [with _ITp = int; std::__atomic_base<_IntTp>::__int_type = int; std::memory_order = std::memory_order]' used but never defined
       exchange(__int_type __i,
       ^
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/atomic_base.h:624:7: warning: inline function 'std::__atomic_base<_IntTp>::__int_type std::__atomic_base<_IntTp>::fetch_add(std::__atomic_base<_IntTp>::__int_type, std::memory_order) volatile [with _ITp = int; std::__atomic_base<_IntTp>::__int_type = int; std::memory_order = std::memory_order]' used but never defined
       fetch_add(__int_type __i,
       ^
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/atomic_base.h:485:7: warning: inline function 'void std::__atomic_base<_IntTp>::store(std::__atomic_base<_IntTp>::__int_type, std::memory_order) volatile [with _ITp = int; std::__atomic_base<_IntTp>::__int_type = int; std::memory_order = std::memory_order]' used but never defined
       store(__int_type __i,
       ^
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/atomic_base.h:507:7: warning: inline function 'std::__atomic_base<_IntTp>::__int_type std::__atomic_base<_IntTp>::load(std::memory_order) const volatile [with _ITp = int; std::__atomic_base<_IntTp>::__int_type = int; std::memory_order = std::memory_order]' used but never defined
       load(memory_order __m = memory_order_seq_cst) const volatile noexcept
       ^

:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp'.
> A build operation failed.
      C++ compiler failed while compiling NetJni.cpp.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 1.482 secs
C++ compiler failed while compiling NetJni.cpp.
19:19:25: External task execution finished 'assembleDebug'.
```
这里提示std命名空间中没有**`unique_ptr`**，没有**`hash`**等，前者是C++11中新添加的智能指针模板，而后者则是哈希模板，它定义一个函数对象，实现散列函数。chromium net中，像这样用到了C++11的特性的地方还有很多，这里的这个错误是由于build.gradle中没有进行C++11的配置。我们在MyApplication/app/build.gradle中配置C++11：
```
            cppFlags.addAll(['-std=gnu++11'])
```
不过在前面，我们也有看到如下这样的错误消息：
```
/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/gnu-libstdc++/4.9/include/bits/c++0x_warning.h:32:2: error: #error This file requires compiler and library support for the ISO C++ 2011 standard. This support is currently experimental, and must be enabled with the -std=c++11 or -std=gnu++11 compiler options.
 #error This file requires compiler and library support for the \
  ^
```
如果我们能早些注意到这样的错误消息，做了C++11的配置，大概这里的这个error还是可以避免的。
# 链接
做了C++11的配置之后，再次编译，继续报错：
```
19:29:04: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/message_loop/message_loop.h:650: error: undefined reference to 'base::MessageLoop::MessageLoop(base::MessageLoop::Type)'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:39: error: undefined reference to 'base::MessageLoop::current()'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:39: error: undefined reference to 'base::MessageLoop::QuitWhenIdle()'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:56: error: undefined reference to 'net::HttpResponseHeaders::EnumerateHeaderLines(unsigned int*, std::string*, std::string*) const'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:140: error: undefined reference to 'base::BasicStringPiece<std::string>::BasicStringPiece(char const*)'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:140: error: undefined reference to 'GURL::GURL(base::BasicStringPiece<std::string>)'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:171: error: undefined reference to 'base::MessageLoop::Run()'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:147: error: undefined reference to 'GURL::~GURL()'
/media/data/MyProjects/MyApplication/app/build/intermediates/objectFiles/armeabi-v7aDebugSharedLibrary/neteasenetMainCpp/9a4xqrdajtunvcxs263gsxl2i/NetJni.o:NetJni.cpp:vtable for base::MessageLoopForIO: error: undefined reference to 'base::MessageLoop::DoWork()'
/media/data/MyProjects/MyApplication/app/build/intermediates/objectFiles/armeabi-v7aDebugSharedLibrary/neteasenetMainCpp/9a4xqrdajtunvcxs263gsxl2i/NetJni.o:NetJni.cpp:vtable for base::MessageLoopForIO: error: undefined reference to 'base::MessageLoop::DoDelayedWork(base::TimeTicks*)'
/media/data/MyProjects/MyApplication/app/build/intermediates/objectFiles/armeabi-v7aDebugSharedLibrary/neteasenetMainCpp/9a4xqrdajtunvcxs263gsxl2i/NetJni.o:NetJni.cpp:vtable for base::MessageLoopForIO: error: undefined reference to 'base::MessageLoop::DoIdleWork()'
/media/data/MyProjects/MyApplication/app/build/intermediates/objectFiles/armeabi-v7aDebugSharedLibrary/neteasenetMainCpp/9a4xqrdajtunvcxs263gsxl2i/NetJni.o:NetJni.cpp:vtable for base::MessageLoopForIO: error: undefined reference to 'base::MessageLoop::IsType(base::MessageLoop::Type) const'
/media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include/base/message_loop/message_loop.h:645: error: undefined reference to 'base::MessageLoop::~MessageLoop()'
collect2: error: ld returned 1 exit status

:app:linkNeteasenetArmeabi-v7aDebugSharedLibrary FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:linkNeteasenetArmeabi-v7aDebugSharedLibrary'.
> A build operation failed.
      Linker failed while linking libneteasenet.so.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/linkNeteasenetArmeabi-v7aDebugSharedLibrary/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 4.835 secs
Linker failed while linking libneteasenet.so.
19:29:09: External task execution finished 'assembleDebug'.
```
这次是链接错误，提示找不到base库和url库中的符号**`base::MessageLoop::MessageLoop(base::MessageLoop::Type)`**、**`GURL::~GURL()`**和**`base::MessageLoop::DoIdleWork()`**等。看来经过前面的各种艰难险阻，我们总算是将native层的cpp文件编译为了.o文件。

为了解决这个问题，我们还需要在**`MyApplication/app/build.gradle`**中添加对base和url这两个库的依赖：
```
        chromium_base {
            headers.srcDir "src/main/jni/third_party/chromium/include"
            binaries.withType(SharedLibraryBinary) {
                sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/libbase.cr.so")
            }
        }
        chromium_url {
            headers.srcDir "src/main/jni/third_party/chromium/include"
            binaries.withType(SharedLibraryBinary) {
                sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/liburl.cr.so")
            }
        }
        
        ......
        
                    dependencies {
                        library 'chromium_base' linkage 'shared'
                        library 'chromium_url' linkage 'shared'
                        library 'chromium_net' linkage 'shared'
                    }
```
## 选择正确的STL库
经过了前面的修改，再次编译，然而......继续报错：
```
19:35:16: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp UP-TO-DATE
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:56: error: undefined reference to 'net::HttpResponseHeaders::EnumerateHeaderLines(unsigned int*, std::string*, std::string*) const'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:140: error: undefined reference to 'base::BasicStringPiece<std::string>::BasicStringPiece(char const*)'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:140: error: undefined reference to 'GURL::GURL(base::BasicStringPiece<std::string>)'
collect2: error: ld returned 1 exit status

:app:linkNeteasenetArmeabi-v7aDebugSharedLibrary FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:linkNeteasenetArmeabi-v7aDebugSharedLibrary'.
> A build operation failed.
      Linker failed while linking libneteasenet.so.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/linkNeteasenetArmeabi-v7aDebugSharedLibrary/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 1.171 secs
Linker failed while linking libneteasenet.so.
19:35:17: External task execution finished 'assembleDebug'.
```
这次是提示找不到符号**'net::HttpResponseHeaders::EnumerateHeaderLines(unsigned int*, std::string*, std::string*) const'**， **'base::BasicStringPiece<std::string>::BasicStringPiece(char const*)'**，**'GURL::GURL(base::BasicStringPiece<std::string>)'**。这个问题还真够诡异。我们对相关的这些模块，base，net和url的gn配置文件，gypi配置文件进行反复的检查，可以百分百地确认，相关的这些文件都是有被编译进so的。

可是為什麼能引用不到这些符号呢？那so中到底有没有相关的这些class呢？通过GNU的binutils工具包中的readelf工具可以查看so中的所有符号，无论是函数符号，还是引用的未定义符号。而且编译器在编译C++文件时，符号都是会被修饰的，以便于支持函数重载等C++的一些特性。binutils工具包还提供了工具c++filt，以帮助我们将修饰后的符号，还原回编译前的样子。

我们利用这些工具，来检查那些so中是否真的没有包含未找到的那些符号：
```
$ readelf -s -W libbase.cr.so | grep BasicStringPiece | xargs c++filt | grep BasicStringPiece

base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(char const*)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(char const*, unsigned int)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(std::__1::__wrap_iter<char const*> const&, std::__1::__wrap_iter<char const*> const&)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece()
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(char const*)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(char const*, unsigned int)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(std::__1::__wrap_iter<char const*> const&, std::__1::__wrap_iter<char const*> const&)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)
base::BasicStringPiece<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::BasicStringPiece()
```
so还真的没有我们的代码中要引用的那些符号。那到底是怎么一回事呢？对比编译我们的工程时报的错，实际情况似乎是，在so中包含了我们需要的函数，但两边修饰后的符号不一致，从而导致链接出错。主要是std中的符号，在so中引用的std的符号，其命名空间都增加了一级"__1"。

这似乎与stl有关。NDK可用的[stl库](https://developer.android.com/ndk/guides/cpp-support.html)大体有如下的这些：
```
libstdc++
gabi++_static
gabi++_shared
stlport_static
stlport_shared
gnustl_static
gnustl_shared
c++_static
c++_shared
```
逐个地对这些标准库进行尝试。除了我们前面配置的`gnustl_static`，上面列出的前面的5种，甚至都找不到头文件`<atomic>`，`gnustl_shared`报出了与`gnustl_static`相同的问题。这就只剩下`c++_static`和`c++_shared`可选了。

我们首先尝试c++_static，但依然报了错：
```
20:00:36: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:56: error: undefined reference to 'net::HttpResponseHeaders::EnumerateHeaderLines(unsigned int*, std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> >*, std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> >*) const'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:140: error: undefined reference to 'base::BasicStringPiece<std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> > >::BasicStringPiece(char const*)'
/media/data/MyProjects/MyApplication/app/src/main/jni/src/NetJni.cpp:140: error: undefined reference to 'GURL::GURL(base::BasicStringPiece<std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> > >)'
collect2: error: ld returned 1 exit status

:app:linkNeteasenetArmeabi-v7aDebugSharedLibrary FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:linkNeteasenetArmeabi-v7aDebugSharedLibrary'.
> A build operation failed.
      Linker failed while linking libneteasenet.so.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/linkNeteasenetArmeabi-v7aDebugSharedLibrary/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 4.385 secs
Linker failed while linking libneteasenet.so.
20:00:41: External task execution finished 'assembleDebug'.
```
情况发生了变化。尽管依然报了`undefined reference to`的error，但这次引用不到的符号却已经与我们在so中抓出来的那些符号非常接近了。`c++_static`和`c++_shared`是LLVM libc++ runtime，编译我们的JNI时使用的这个STL的实现似乎与编译chromium代码时使用的那个不太一样，但无疑编译chromium时使用的是LLVM libc++ runtime。

我们随便打开一个std的头文件来一窥究竟，比如声明std::string的头文件android-ndk-r12b/sources/cxx-stl/llvm-libc++/libcxx/include/iosfwd，我们注意到了如下的这几行：
```
_LIBCPP_BEGIN_NAMESPACE_STD

......
_LIBCPP_END_NAMESPACE_STD
```
这个宏的定义在android-ndk-r12b/sources/cxx-stl/llvm-libc++/libcxx/include/__config：
```
#define _LIBCPP_ABI_VERSION 1

#define _LIBCPP_CONCAT1(_LIBCPP_X,_LIBCPP_Y) _LIBCPP_X##_LIBCPP_Y
#define _LIBCPP_CONCAT(_LIBCPP_X,_LIBCPP_Y) _LIBCPP_CONCAT1(_LIBCPP_X,_LIBCPP_Y)

#define _LIBCPP_NAMESPACE _LIBCPP_CONCAT(__ndk,_LIBCPP_ABI_VERSION)

......
#define _LIBCPP_BEGIN_NAMESPACE_STD namespace std { namespace _LIBCPP_NAMESPACE {
#define _LIBCPP_END_NAMESPACE_STD  } }
```
由此我们终于揭开了引用的符号，中间插入的那一级命名空间`__ndk1`的秘密。这是LLVM libc++ runtime加的。

但编译出来的so中的符号又是怎么回事呢？chromium的代码库中，有完整的构建工具链，包括android的NDK和SDK，它们位于`chromium/src/third_party/android_tools`。我们打开文件chromium/src/third_party/android_tools/ndk/sources/cxx-stl/llvm-libc++/libcxx/include/__config，可以看到如下这样的宏定义：
```
#define _LIBCPP_ABI_VERSION 1

#define _LIBCPP_CONCAT1(_LIBCPP_X,_LIBCPP_Y) _LIBCPP_X##_LIBCPP_Y
#define _LIBCPP_CONCAT(_LIBCPP_X,_LIBCPP_Y) _LIBCPP_CONCAT1(_LIBCPP_X,_LIBCPP_Y)

#define _LIBCPP_NAMESPACE _LIBCPP_CONCAT(__,_LIBCPP_ABI_VERSION)
```
至此，这个链接过程中符号找不到的问题终于被厘清——chromium中的NDK不是标准NDK。

这样的话解法也就显而易见了，编译我们的工程时，stl用c++_static，同时定制编译chromium所用的NDK。
# 采用标准NDK编译chromium net
为了解决编译chromium时所用的NDK与编译我们的工程时所用的NDK不一致，导致的链接错误，我们需要定制编译chromium时所用的NDK。这需要重新对chromium的构建进行配置，并再次编译chromium net。首先需要修改out/Default/args.gn，修改后的样子如下：
```
target_os = "android"
target_cpu = "arm"  # (default)
is_debug = true  # (default)

# Other args you may want to set:
is_component_build = true
is_clang = true
symbol_level = 1  # Faster build with fewer symbols. -g1 rather than -g2
enable_incremental_javac = true  # Much faster; experimental
android_ndk_root = "~/data/dev_tools/Android/android-ndk-r12b"
```
然后，通过如下的命令，再次编译chromium net。但出错了：
```
$ gn gen out/Default/
Generating files...
Done. Wrote 11150 targets from 881 files in 2166ms
$ ninja -C out/Default/ net
ninja: Entering directory `out/Default/'
ninja: error: '../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9/libgcc.a', needed by 'obj/base/third_party/dynamic_annotations/libdynamic_annotations.a', missing and no known rule to make it
```
这次是找不到ndk的一个文件**`android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9/libgcc.a`**。我们发现**`android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9`**目录不存在，但是**`android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9.x`**是存在的，我们直接将4.9.x复制到4.9。

再次编译，在链接生成libbase.cr.so时，出错了：
```
$ ninja -C out/Default/ net
ninja: Entering directory `out/Default/'
[1349/2009] SOLINK ./libbase.cr.so
FAILED: libbase.cr.so libbase.cr.so.TOC lib.unstripped/libbase.cr.so 
python "/media/data/chromium_android/src/build/toolchain/gcc_solink_wrapper.py" --readelf="../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-readelf" --nm="../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-nm" --strip=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-strip --sofile="./lib.unstripped/libbase.cr.so" --tocfile="./libbase.cr.so.TOC" --output="./libbase.cr.so" -- ../../third_party/llvm-build/Release+Asserts/bin/clang++ -shared -Wl,--fatal-warnings -fPIC -Wl,-z,noexecstack -Wl,-z,now -Wl,-z,relro -Wl,-z,defs -fuse-ld=gold --gcc-toolchain=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64 -Wl,--icf=all -Wl,--build-id=sha1 -Wl,--no-undefined -Wl,--exclude-libs=libgcc.a -Wl,--exclude-libs=libc++_static.a -Wl,--exclude-libs=libvpx_assembly_arm.a --target=arm-linux-androideabi -Wl,--warn-shared-textrel -Wl,-O1 -Wl,--as-needed -nostdlib -Wl,--warn-shared-textrel --sysroot=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/platforms/android-16/arch-arm  -Wl,-wrap,calloc -Wl,-wrap,free -Wl,-wrap,malloc -Wl,-wrap,memalign -Wl,-wrap,posix_memalign -Wl,-wrap,pvalloc -Wl,-wrap,realloc -Wl,-wrap,valloc -L/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a -o "./lib.unstripped/libbase.cr.so" -Wl,-soname="libbase.cr.so" @"./libbase.cr.so.rsp"
../../base/debug/stack_trace_android.cc:39: error: undefined reference to '_Unwind_GetIP'
clang: error: linker command failed with exit code 1 (use -v to see invocation)
[1358/2009] CXX obj/third_party/protobuf/protobuf_lite/extension_set.o
ninja: build stopped: subcommand failed.
```
这次提示找不到符号_Unwind_GetIP，这个符号是libunwind中的。我们还需要修改`base/BUILD.gn`，在为android编译时，添加对libunwind的依赖：
```
if (is_android) {
  config("android_system_libs") {
    libs = [ "log", "unwind" ]  # Used by logging.cc.
  }
}
```
再次编译时依然出错了：
```
$ gn args out/Default/
Waiting for editor on "/media/data/chromium_android/src/out/Default/args.gn"...
Generating files...
Done. Wrote 11150 targets from 881 files in 2196ms
$ ninja -C out/Default/ net
ninja: Entering directory `out/Default/'
[27/640] SOLINK ./libicuuc.cr.so
FAILED: libicuuc.cr.so libicuuc.cr.so.TOC lib.unstripped/libicuuc.cr.so 
python "/media/data/chromium_android/src/build/toolchain/gcc_solink_wrapper.py" --readelf="../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-readelf" --nm="../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-nm" --strip=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-strip --sofile="./lib.unstripped/libicuuc.cr.so" --tocfile="./libicuuc.cr.so.TOC" --output="./libicuuc.cr.so" -- ../../third_party/llvm-build/Release+Asserts/bin/clang++ -shared -Wl,--fatal-warnings -fPIC -Wl,-z,noexecstack -Wl,-z,now -Wl,-z,relro -Wl,-z,defs -fuse-ld=gold --gcc-toolchain=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64 -Wl,--icf=all -Wl,--build-id=sha1 -Wl,--no-undefined -Wl,--exclude-libs=libgcc.a -Wl,--exclude-libs=libc++_static.a -Wl,--exclude-libs=libvpx_assembly_arm.a --target=arm-linux-androideabi -Wl,--warn-shared-textrel -Wl,-O1 -Wl,--as-needed -nostdlib -Wl,--warn-shared-textrel --sysroot=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/platforms/android-16/arch-arm  -L/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a -o "./lib.unstripped/libicuuc.cr.so" -Wl,-soname="libicuuc.cr.so" @"./libicuuc.cr.so.rsp"
../../third_party/icu/source/common/rbbi.cpp:324: error: undefined reference to '__cxa_bad_typeid'
../../third_party/icu/source/common/schriter.cpp:90: error: undefined reference to '__cxa_bad_typeid'
../../third_party/icu/source/common/stringtriebuilder.cpp:386: error: undefined reference to '__cxa_bad_typeid'
../../third_party/icu/source/common/uchriter.cpp:72: error: undefined reference to '__cxa_bad_typeid'
clang: error: linker command failed with exit code 1 (use -v to see invocation)
[36/640] CXX obj/url/url/url_util.o
ninja: build stopped: subcommand failed.
```
这是在编译icu时，找不到符号__cxa_bad_typeid。这需要重新配置编译环境，更改所用的工具链，不能是clang，修改之后的out/Default/args.gn如下：
```
target_os = "android"
target_cpu = "arm"  # (default)
is_debug = true  # (default)

# Other args you may want to set:
is_component_build = true
is_clang = false
symbol_level = 1  # Faster build with fewer symbols. -g1 rather than -g2
enable_incremental_javac = true  # Much faster; experimental
android_ndk_root = "/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b"
```
重新配置chromium的构建环境，并再次编译net。

这终于编译好了。将编译出来的so文件拷贝进我们的工程。再次编译我们的工程。但依然在报错：
```
21:02:17: Executing external task 'assembleDebug'...
Observed package id 'build-tools;20.0.0' in inconsistent location '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/android-4.4W' (Expected '/home/hanpfei0306/data/dev_tools/Android/sdk/build-tools/20.0.0')
Incremental java compilation is an incubating feature.
:app:compileNeteasenetArmeabi-v7aDebugSharedLibraryNeteasenetMainCpp
:app:linkNeteasenetArmeabi-v7aDebugSharedLibrary
:app:neteasenetArmeabi-v7aDebugSharedLibrary
:app:stripSymbolsArmeabi-v7aDebugSharedLibrary
:app:ndkBuildArmeabi-v7aDebugSharedLibrary
:app:ndkBuildArmeabi-v7aDebugStaticLibrary UP-TO-DATE
:app:compileNeteasenetArmeabiDebugSharedLibraryNeteasenetMainCpp
/usr/local/google/buildbot/src/android/ndk-r12-release/ndk/sources/cxx-stl/llvm-libc++/libcxx/include/atomic:922: error: undefined reference to '__atomic_fetch_add_4'
/usr/local/google/buildbot/src/android/ndk-r12-release/ndk/sources/cxx-stl/llvm-libc++abi/libcxxabi/src/cxa_handlers.cpp:112: error: undefined reference to '__atomic_exchange_4'
/usr/local/google/buildbot/src/android/ndk-r12-release/ndk/sources/cxx-stl/llvm-libc++abi/libcxxabi/src/cxa_default_handlers.cpp:106: error: undefined reference to '__atomic_exchange_4'
/usr/local/google/buildbot/src/android/ndk-r12-release/ndk/sources/cxx-stl/llvm-libc++abi/libcxxabi/src/cxa_default_handlers.cpp:117: error: undefined reference to '__atomic_exchange_4'
collect2: error: ld returned 1 exit status

:app:linkNeteasenetArmeabiDebugSharedLibrary FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:linkNeteasenetArmeabiDebugSharedLibrary'.
> A build operation failed.
      Linker failed while linking libneteasenet.so.
  See the complete log at: file:///media/data/MyProjects/MyApplication/app/build/tmp/linkNeteasenetArmeabiDebugSharedLibrary/output.txt

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 6.648 secs
Linker failed while linking libneteasenet.so.
21:02:24: External task execution finished 'assembleDebug'.
```
不过这次已经不是找不到chromium的库中的符号了，而是标准库中的一些符号。这需要我们修改依赖的stl为`c++_shared`，而不是`c++_static`。再次编译，终于build pass。
# 编译链接的标记设置
chromium中，有许多feature的开关是通过预定义宏来控制的，它的整个的编译链接都有它独特的参数。如果在编译我们的工程时，预定义的宏与编译chromium时预定义的宏不一致，就很容易出现两边的class实际的定义不一样的问题。比如net::URLRequestContextBuilder类的定义：
```
 private:
  std::string accept_language_;
  std::string user_agent_;
  // Include support for data:// requests.
  bool data_enabled_;
#if !defined(DISABLE_FILE_SUPPORT)
  // Include support for file:// requests.
  bool file_enabled_;
#endif
#if !defined(DISABLE_FTP_SUPPORT)
  // Include support for ftp:// requests.
  bool ftp_enabled_;
#endif
  bool http_cache_enabled_;
  bool throttling_enabled_;
  bool backoff_enabled_;
  bool sdch_enabled_;
  bool cookie_store_set_by_client_;

  scoped_refptr<base::SingleThreadTaskRunner> file_task_runner_;
  HttpCacheParams http_cache_params_;
  HttpNetworkSessionParams http_network_session_params_;
  base::FilePath transport_security_persister_path_;
  NetLog* net_log_;
  std::unique_ptr<HostResolver> host_resolver_;
  std::unique_ptr<ChannelIDService> channel_id_service_;
  std::unique_ptr<ProxyConfigService> proxy_config_service_;
  std::unique_ptr<ProxyService> proxy_service_;
  std::unique_ptr<NetworkDelegate> network_delegate_;
  std::unique_ptr<ProxyDelegate> proxy_delegate_;
  std::unique_ptr<CookieStore> cookie_store_;
#if !defined(DISABLE_FTP_SUPPORT)
  std::unique_ptr<FtpTransactionFactory> ftp_transaction_factory_;
#endif
```
会根据宏定义来确定某些字段是否存在，如**`ftp_enabled_`**等。如果两边的宏定义不同，则会导致两边对net::URLRequestContextBuilder对象的处理不同。对于定义了宏DISABLE_FTP_SUPPORT的一边，它在为net::URLRequestContextBuilder对象分配内存空间时，将小于另一边看到的相同类对象的内存空间大小，可想而知，在运行期该会要出现什么样的奇怪问题了。

比如，我们在我们的工程中，定义了宏DISABLE_FTP_SUPPORT，而在编译chromium net时没有定义这个宏。在我们的native代码里，在栈上创建了一个net::URLRequestContextBuilder对象，内存将由我们的工程的编译器分配。而在运行期，分配的这块内存会被传递给类的构造函数，该构造函数则是在chromium net的so中，而它对这块内存空间的预期要大于我们的工程的编译器分配的大小，则在初始化的过程中，难免要踩坏周围的内存的。

为了避免这个问题，最好的方法就是将相关的这些编译、连接，预定义宏等，在两边保持一致。gn工具在这个问题上也可以帮到我们，同样是gn desc，我们需要从输出的如下这些段中提取我们要的参数：
```
$ gn desc out/Default/ net
......
cflags
  -fno-strict-aliasing
  --param=ssp-buffer-size=4
  -fstack-protector
  -funwind-tables
  -fPIC
  -pipe
  -ffunction-sections
  -fno-short-enums
  -finline-limit=64
  -march=armv7-a
  -mfloat-abi=softfp
  -mthumb
  -mthumb-interwork
  -mtune=generic-armv7-a
  -fno-tree-sra
  -fno-caller-saves
  -mfpu=neon
  -Wall
  -Werror
  -Wno-psabi
  -Wno-unused-local-typedefs
  -Wno-maybe-uninitialized
  -Wno-missing-field-initializers
  -Wno-unused-parameter
  -Os
  -fdata-sections
  -ffunction-sections
  -fomit-frame-pointer
  -g1
  --sysroot=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/platforms/android-16/arch-arm
  -fvisibility=hidden

cflags_cc
  -fno-threadsafe-statics
  -fvisibility-inlines-hidden
  -std=gnu++11
  -Wno-narrowing
  -fno-rtti
  -isystem../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libcxx/include
  -isystem../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++abi/libcxxabi/include
  -isystem../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/android/support/include
  -fno-exceptions

......

defines
  V8_DEPRECATION_WARNINGS
  ENABLE_NOTIFICATIONS
  ENABLE_BROWSER_CDMS
  ENABLE_PRINTING=1
  ENABLE_BASIC_PRINTING=1
  ENABLE_SPELLCHECK=1
  USE_BROWSER_SPELLCHECKER=1
  USE_OPENSSL_CERTS=1
  NO_TCMALLOC
  USE_EXTERNAL_POPUP_MENU=1
  ENABLE_WEBRTC=1
  DISABLE_NACL
  ENABLE_SUPERVISED_USERS=1
  VIDEO_HOLE=1
  SAFE_BROWSING_DB_REMOTE
  CHROMIUM_BUILD
  ENABLE_MEDIA_ROUTER=1
  ENABLE_WEBVR
  FIELDTRIAL_TESTING_ENABLED
  _FILE_OFFSET_BITS=64
  ANDROID
  HAVE_SYS_UIO_H
  ANDROID_NDK_VERSION=r10e
  __STDC_CONSTANT_MACROS
  __STDC_FORMAT_MACROS
  COMPONENT_BUILD
  __GNU_SOURCE=1
  _DEBUG
  DYNAMIC_ANNOTATIONS_ENABLED=1
  WTF_USE_DYNAMIC_ANNOTATIONS=1
  DLOPEN_KERBEROS
  NET_IMPLEMENTATION
  USE_KERBEROS
  ENABLE_BUILT_IN_DNS
  POSIX_AVOID_MMAP
  ENABLE_WEBSOCKETS
  GOOGLE_PROTOBUF_NO_RTTI
  GOOGLE_PROTOBUF_NO_STATIC_INITIALIZER
  HAVE_PTHREAD
  PROTOBUF_USE_DLLS
  BORINGSSL_SHARED_LIBRARY
  U_USING_ICU_NAMESPACE=0
  U_ENABLE_DYLOAD=0
  U_NOEXCEPT=
  ICU_UTIL_DATA_IMPL=ICU_UTIL_DATA_FILE

......

ldflags
  -Wl,--fatal-warnings
  -fPIC
  -Wl,-z,noexecstack
  -Wl,-z,now
  -Wl,-z,relro
  -Wl,-z,defs
  -fuse-ld=gold
  -Wl,--icf=all
  -Wl,--build-id=sha1
  -Wl,--no-undefined
  -Wl,--exclude-libs=libgcc.a
  -Wl,--exclude-libs=libc++_static.a
  -Wl,--exclude-libs=libvpx_assembly_arm.a
  -Wl,--warn-shared-textrel
  -Wl,-O1
  -Wl,--as-needed
  -nostdlib
  -Wl,--warn-shared-textrel
  --sysroot=../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/platforms/android-16/arch-arm
  
  -Wl,-wrap,calloc
  -Wl,-wrap,free
  -Wl,-wrap,malloc
  -Wl,-wrap,memalign
  -Wl,-wrap,posix_memalign
  -Wl,-wrap,pvalloc
  -Wl,-wrap,realloc
  -Wl,-wrap,valloc

......

libs
  c++_shared
  /home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9/libgcc.a
  c
  atomic
  dl
  m
  log
  unwind

lib_dirs
  /home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a/
```
最终，我们的***`MyApplication/app/build.gradle`***文件将如下面这样：
```
apply plugin: 'com.android.model.application'

model {
    repositories {
        libs(PrebuiltLibraries) {
            chromium_net {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/libnet.cr.so")
                }
            }
            chromium_base {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/libbase.cr.so")
                }
            }
            chromium_url {
                headers.srcDir "src/main/jni/third_party/chromium/include"
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jni/third_party/chromium/libs/${targetPlatform.getName()}/liburl.cr.so")
                }
            }
        }
    }

    android {
        compileSdkVersion 23
        buildToolsVersion "23.0.3"

        defaultConfig {
            applicationId "com.example.hanpfei0306.myapplication"
            minSdkVersion.apiLevel 19
            targetSdkVersion.apiLevel 21
            versionCode 1
            versionName "1.0"
        }

        ndk {
            moduleName "neteasenet"
            toolchain "clang"



            CFlags.addAll(["-fno-strict-aliasing",
                           "--param=ssp-buffer-size=4",
                           "-fstack-protector",
                           "-funwind-tables",
                           "-fPIC",
                           "-pipe",
                           "-ffunction-sections",
                           "-fno-short-enums",
                           "-finline-limit=64",
                           "-mfloat-abi=softfp",
                           "-mfpu=neon",
                           "-Os",
                           "-fdata-sections",
                           "-ffunction-sections",
                           "-fomit-frame-pointer",
                           "-g1",
                           "-fvisibility=hidden"
            ])
            CFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include/'),])

            cppFlags.addAll(["-fno-threadsafe-statics",
                             "-fvisibility-inlines-hidden",
                             "-std=gnu++11",
                             "-Wno-narrowing",
                             "-fno-rtti",
            ])
            cppFlags.addAll(["-DV8_DEPRECATION_WARNINGS",
                             "-DENABLE_NOTIFICATIONS",
                             "-DENABLE_BROWSER_CDMS",
                             "-DENABLE_PRINTING=1",
                             "-DENABLE_BASIC_PRINTING=1",
                             "-DENABLE_SPELLCHECK=1",
                             "-DUSE_BROWSER_SPELLCHECKER=1",
                             "-DUSE_OPENSSL_CERTS=1",
                             "-DNO_TCMALLOC",
                             "-DUSE_EXTERNAL_POPUP_MENU=1",
                             "-DDISABLE_NACL",
                             "-DENABLE_SUPERVISED_USERS=1",
                             "-DCHROMIUM_BUILD",
                             "-D_FILE_OFFSET_BITS=64",
                             "-DANDROID",
                             "-DHAVE_SYS_UIO_H",
                             "-D__STDC_CONSTANT_MACROS",
                             "-D__STDC_FORMAT_MACROS",
                             "-D_FORTIFY_SOURCE=2",
                             "-DCOMPONENT_BUILD",
                             "-D__GNU_SOURCE=1",
                             "-D_DEBUG",
                             "-DDYNAMIC_ANNOTATIONS_ENABLED=1",
                             "-DWTF_USE_DYNAMIC_ANNOTATIONS=1",
                             "-DDLOPEN_KERBEROS",
                             "-DNET_IMPLEMENTATION",
                             "-DUSE_KERBEROS",
                             "-DENABLE_BUILT_IN_DNS",
                             "-DPOSIX_AVOID_MMAP",
                             "-DENABLE_WEBSOCKETS",
                             "-DGOOGLE_PROTOBUF_NO_RTTI",
                             "-DGOOGLE_PROTOBUF_NO_STATIC_INITIALIZER",
                             "-DHAVE_PTHREAD",
                             "-DPROTOBUF_USE_DLLS",
                             "-DBORINGSSL_SHARED_LIBRARY",
                             "-DU_USING_ICU_NAMESPACE=0",
                             "-DU_ENABLE_DYLOAD=0",
            ])
            cppFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include'), ])

            ldLibs.add("android")
            ldLibs.add("log")
            ldLibs.add("z")
            stl "c++_shared"
        }

        sources {
            main {
                java {
                    source {
                        srcDir "src/main/java"
                    }
                }
                jni {
                    source {
                        srcDirs = ["src/main/jni",]
                    }
                    dependencies {
                        library 'chromium_base' linkage 'shared'
                        library 'chromium_url' linkage 'shared'
                        library 'chromium_net' linkage 'shared'
                    }
                }
                jniLibs {
                    source {
                        srcDirs =["src/main/jni/third_party/chromium/libs",]
                    }
                }
            }
        }

        buildTypes {
            debug {
                ndk {
                    abiFilters.add("armeabi")
                    abiFilters.add("armeabi-v7a")
                }
            }
            release {
                minifyEnabled false
                proguardFiles.add(file("proguard-rules.pro"))
                ndk {
                    abiFilters.add("armeabi")
                    abiFilters.add("armeabi-v7a")
                }
            }
        }
    }
}
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'

    compile 'com.android.support:appcompat-v7:23.4.0'
}
```
