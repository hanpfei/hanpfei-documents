---
title: 懒人chromium net android移植指南
---

# 编译chromium net模块

首先要做的事情就是下载完整的chromium代码，这可以参考[Chromium Android编译指南](http://www.jianshu.com/p/5fce18cbe016)来完成。然后执行（假设当前目录是chromium代码库的根目录）命令：

<!--more-->

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
is_debug = false  # (default)

# Other args you may want to set:
is_component_build = true
is_clang = false
symbol_level = 1  # Faster build with fewer symbols. -g1 rather than -g2
enable_incremental_javac = false  # Much faster; experimental
android_ndk_root = "/home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b"
```
关键点主要有如下几个：

 - is_debug被置为了false，表示编译非Debug版的。在这种情况下，enable_incremental_javac同样要被置为false。否则在执行`gn gen out/Default`时会报出如下的error：
```
ERROR at //build/config/android/config.gni:136:3: Assertion failed.
  assert(!(enable_incremental_javac && !is_java_debug))
  ^-----
See //build/config/compiler/compiler.gni:5:1: whence it was imported.
import("//build/config/android/config.gni")
^-----------------------------------------
See //BUILD.gn:11:1: whence it was imported.
import("//build/config/compiler/compiler.gni")
^--------------------------------------------
```
 - 配置了android_ndk_root选项，也就是说编译的时候使用我们给的NDK中的工具链，而不是chromium代码库中的工具链。具体的原因，可以参考[chromium net android移植[(http://www.jianshu.com/p/082739b65f03)中的说明。
 - is_clang选项被置为了false。具体的原因可以参考[chromium net android移植[(http://www.jianshu.com/p/082739b65f03)中的说明。

保存退出之后，执行：
```
$ gn gen out/Default
```
产生ninja构建所需的各个模块的ninja文件。

将**`android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9.x`**拷贝到**`android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9`**，利用ninja构建chromium的模块时，需要引用到**`android-ndk-r12b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9/libgcc.a`**。

配置base模块。需要修改修改`base/BUILD.gn`，在为android编译时，添加对libunwind的依赖：
```
if (is_android) {
  config("android_system_libs") {
    libs = [ "log", "unwind" ]  # Used by logging.cc.
  }
}
```
具体的原因可以参考[chromium net android移植[(http://www.jianshu.com/p/082739b65f03)中的说明。

随后输入如下命令编译net模块：
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
# 提取导出头文件
为了使用net模块提供的API，我们不可避免地要将net导出的头文件引入我们的项目。要做到这些，我们首先就需要从chromium工程中提取net导出的头文件。不像许多其它的C++项目，源代码文件、私有头文件及导出头文件存放的位置被很好地做了区隔，chromium各个模块的所有头文件和源代码文件都是放在一起的。这还是给我们提取导出头文件的工作带来了一点麻烦。

这里我们借助于gn工具提供的desc功能（关于gn工具的用法，可以参考[GN的使用 - GN工具](http://my.oschina.net/wolfcs/blog/726696)一文），输出中如下的这两段：
```
$ gn desc out/Default/ net
Target //net:net
Type: shared_library
Toolchain: //build/toolchain/android:arm
......
sources
  //net/base/address_family.cc
  //net/base/address_family.h
......

public
  [All headers listed in the sources are public.]
```
编写脚本来实现。

我们可以传入***[chromium代码库的src目录路径]***，***[输出目录的路径]***，***[模块名]***，及***[保存头文件的目标目录路径]***作为参数，来提取模块的所有导出头文件，***[保存头文件的目标目录路径]***参数缺失时默认使用当前目录，如：
```
$ cd /media/data/MyProjects/MyApplication/app/src/main/jni/third_party/chromium/include
$ chromium_mod_headers_extracter.py ~/data/chromium_android/src  out/Default net
$ chromium_mod_headers_extracter.py ~/data/chromium_android/src  out/Default base
$ chromium_mod_headers_extracter.py ~/data/chromium_android/src  out/Default base
```
我们利用我们的脚本，提取出net、base和url这三个模块导出的头文件。

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
此外，利用脚本提取头文件的方法，会遗漏一些必须的头文件。主要是如下的这几个：
```
base/callback_forward.h
base/message_loop/timer_slack.h
base/files/file.h
net/cert/cert_status_flags_list.h
net/cert/cert_type.h
net/base/privacy_mode.h
net/websockets/websocket_event_interface.h
net/quic/quic_alarm_factory.h
```
对于这些文件，我们直接把它们从chromium的代码库拷贝到我们的工程中的对应位置即可。

我们还需要引入chromium的build配置头文件"build/build_config.h"。直接将chromium代码库中的对应文件拷贝过来，放到对应的位置。

将MyApplication/app/src/main/jni/third_party/chromium/include/base/gtest_prod_util.h文件中对"testing/gtest/include/gtest/gtest_prod.h"的include注释掉，同时修改FRIEND_TEST_ALL_PREFIXES宏的定义：
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

            CFlags.addAll(['-I' + file('src/main/jni/third_party/chromium/include/'),])

            cppFlags.addAll(["-std=gnu++11", ])
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
关键点主要有如下的这些：

 - 为net、base和url这几个模块创建PrebuiltLibraries libs元素，并正确的设置对这些模块的依赖。
 - 配置stl为"c++_shared"。
 - cppFlags的"-std=gnu++11"选项必不可少。
 - buildType下的debug和release，需要给它们ndk的abiFilters添加我们想要支持的ABI，而不是留空，以防止Android Studio为我们编译我们不打算支持的ABI的so，而出现找不到文件的问题。
 - CFlags和cppFlags中除了配置头文件搜索路径的那两行之外，其它的内容，主要是从chromium的构建环境中提取的。方法为：

```
$ gn desc out/Default/ net
Target //net:net
Type: shared_library
Toolchain: //build/toolchain/android:arm

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
  -fomit-frame-pointer
  -fno-ident
  -fdata-sections
  -ffunction-sections
  -g1
  --sysroot=../../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/platforms/android-16/arch-arm
  -fvisibility=hidden

cflags_cc
  -fno-threadsafe-statics
  -fvisibility-inlines-hidden
  -std=gnu++11
  -Wno-narrowing
  -fno-rtti
  -isystem../../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++/libcxx/include
  -isystem../../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/cxx-stl/llvm-libc++abi/libcxxabi/include
  -isystem../../../../../../../home/hanpfei0306/data/dev_tools/Android/android-ndk-r12b/sources/android/support/include
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
  _FORTIFY_SOURCE=2
  COMPONENT_BUILD
  __GNU_SOURCE=1
  NDEBUG
  NVALGRIND
  DYNAMIC_ANNOTATIONS_ENABLED=0
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

主要是build.gradle的cppFlags添加的那些宏定义，它们来自defines。如果这些配置，在编译chromium net so的环境，和构建我们的工程的环境之间存在差异，则很可能会导致运行期一些莫名奇妙的问题，比如意外的缓冲区溢出之类的。
Done。
