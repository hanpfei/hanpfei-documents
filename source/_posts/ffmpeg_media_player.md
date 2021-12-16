这里的播放器演示程序用于播放一个本地文件，因而不需要关心播放网络上的媒体数据时的网络传输问题。

对于播放本地媒体文件的播放器来说，所要完成的工作主要包括：解封装 -> 音频解码/视频解码 ->  对于音频来说，还需要做格式转换和重采样 -> 音频渲染和视频渲染。本文的代码位于 GitHub [ffmpeg-mediaplayer-android](https://github.com/hanpfei/ffmpeg-mediaplayer-android)。

## 音频渲染

音频播放通过 Android 的 OpenSL ES 接口实现。具体的代码基于 [ndk-samples](https://github.com/android/ndk-samples) 中的 [audio-echo](https://github.com/android/ndk-samples/tree/main/audio-echo) 修改获得。

[ndk-samples](https://github.com/android/ndk-samples) 中的 [audio-echo](https://github.com/android/ndk-samples/tree/main/audio-echo) 在播放或录制的缓冲区 overflow 时，会自动停止播放或录制，demo 中针对此问题做了一些修改。此外，修改 [audio-echo](https://github.com/android/ndk-samples/tree/main/audio-echo) 的代码以支持定制播放的音频内容。

## 为 Android 应用集成 ffmpeg 库

通过 [mobile-ffmpeg](https://github.com/tanersener/mobile-ffmpeg) 编译获得 ffmpeg 的几个库文件，这个工程所基于的 ffmpeg 版本还算比较新，为 FFmpeg 4.4 的。在 Mac 平台，要为 Android 平台编译 ffmpeg 的动态链接库，首先需要安装 Android 的 SDK 和 NDK 包。随后，为 android 平台编译动态链接库的方法为：
```
% export ANDROID_NDK_ROOT=/Users/henryhan/bin/android-ndk-r21e
% export ANDROID_HOME=/Users/henryhan/Library/Android/sdk
% ./android.sh
```

编译方法基本上就是设置环境变量 `ANDROID_NDK_ROOT` 和 `ANDROID_HOME` 分别只想 NDK 和 SDK 的路径，然后执行 [mobile-ffmpeg](https://github.com/tanersener/mobile-ffmpeg) 提供的脚本文件 `android.sh`，`android.sh` 脚本会完成所有的编译动作。

`android.sh` 脚本的执行过程大致为：`mobile-ffmpeg/android.sh` -> `mobile-ffmpeg/build/main-android.sh` -> `mobile-ffmpeg/build/android-ffmpeg.sh`，`mobile-ffmpeg/build/android-ffmpeg.sh` 脚本中可以看到比较详细的编译配置。编译过程中，编译日志都会输出到 `mobile-ffmpeg/build.log`，从这个文件中可以看到比较详细的编译过程的信息。

在 `mobile-ffmpeg/build/android-ffmpeg.sh` 脚本中可以看到：
```
LIB_NAME="ffmpeg"
. . . . . .
./configure \
    --cross-prefix="${BUILD_HOST}-" \
    --sysroot="${ANDROID_NDK_ROOT}/toolchains/llvm/prebuilt/${TOOLCHAIN}/sysroot" \
    --prefix="${BASEDIR}/prebuilt/android-$(get_target_build)/${LIB_NAME}" \
. . . . . .
rm -rf ${BASEDIR}/prebuilt/android-$(get_target_build)/${LIB_NAME}
make install 1>>${BASEDIR}/build.log 2>&1
```

因而编译出来的头文件和动态链接库文件都将被安装到 `mobile-ffmpeg/prebuilt/android-$(get_target_build)/ffmpeg`，如为 x86_64 编译出来的动态链接库被安装在了 `mobile-ffmpeg/prebuilt/android-x86_64/ffmpeg/lib` 目录下，x86_64 平台的头文件被放在了 `mobile-ffmpeg/prebuilt/android-x86_64/ffmpeg/include` 目录下，而为 arm64 编译出来的动态链接库被安装在了 `mobile-ffmpeg/prebuilt/android-arm64/ffmpeg/lib` 目录下，arm64 平台的头文件被放在了 `mobile-ffmpeg/prebuilt/android-arm64/ffmpeg/include` 目录下，其它 ABI 依此类推。

[mobile-ffmpeg](https://github.com/tanersener/mobile-ffmpeg) 的 `android.sh` 还会编译生成 ffmpeg 库的 AAR 文件，位于 `mobile-ffmpeg/prebuilt/android-aar/mobile-ffmpeg/mobile-ffmpeg.aar`。

可以参考文章 [Android studio中NDK开发（二）——使用CMake引入第三方so库及头文件](http://www.4k8k.xyz/article/mjfh095215/90315162) 和 [NDK--CMakeLists配置第三方so库](https://cloud.tencent.com/developer/article/1655210) 在 CMake 中添加预编译的库，大体方法为：
 - 将 ffmpeg 库的几个动态链接库文件和头文件拷贝到适当的位置。
 - 为 CMake 添加库。当需要添加多个 so 库依赖时，也需要为 CMake 添加多个 `add_library()` 项。
 - 设置库二进制文件的搜索路径。需要添加多个 so 库依赖时，也需要为 CMake 添加多个 `set_target_properties()` 项。
 - 设置头文件搜索路径。为 JNI 的 target 添加头文件搜索路径，添加一个 `target_include_directories()`，但其中可以包含一个或多个路径。
 - 设置链接依赖。为 JNI 的 target 添加库依赖，每个库一个 `target_link_libraries()` 的项。

在通过 CMake 的自动打包预构建依赖项时，不再需要在 `build.gradle` 中像下面这样设置预构建项的地址：
```
    sourceSets {
        main {
            jniLibs.srcDirs = ["src/main/libs"]
        }
    }
```

有了 Android Gradle 插件 4.0，上述配置不再是必需的，并且会导致构建失败：
```
-----------
* What went wrong:
Execution failed for task ':app:mergeReleaseNativeLibs'.
> A failure occurred while executing com.android.build.gradle.internal.tasks.MergeJavaResWorkAction
   > 2 files found with path 'lib/x86/libswscale.so' from inputs:
      - /Users/henryhan/Projects/Shopee/henry-entrytask/app/build/intermediates/merged_jni_libs/release/out
      - /Users/henryhan/Projects/Shopee/henry-entrytask/app/build/intermediates/cmake/release/obj
     If you are using jniLibs and CMake IMPORTED targets, see
     https://developer.android.com/r/tools/jniLibs-vs-imported-targets
```

更多信息，可以参考 [Google 的官方文档](https://developer.android.com/r/tools/jniLibs-vs-imported-targets)。

通过 `System.loadLibrary()` 加载 ffmpeg 和 JNI 的 so 文件时，需要注意加载 so 的顺序，被依赖的 so 文件需要先加载，依赖的 so 需要后加载，如 `swscale` 库依赖 `avutil` 库，则需要 `System.loadLibrary("avutil");` 放在 `System.loadLibrary("swscale");` 的前面，否则会报错说符号找不到。

ffmpeg 是一个纯 C 的库，就像在任何 C++ 代码中使用 FFmpeg 的 API 那样，如果 JNI 代码用 C++ 写，ffmpeg 头文件的 include 要放在 `extern "C"` 里。

## 音视频解封装及解码

代码参考了 [HelloFFmpeg](https://github.com/al4fun/HelloFFmpeg) 和 [goffmpeg](https://github.com/liupengh3c/goffmpeg) 这些工程的代码。

视频：解封装获得 H264 编码帧 ->  解码获得裸 YUV 视频流。
音频：解封装获得 AAC 编码音频帧 -> 解码获得裸 PCM 音频流 -> 经过样本格式转换（样本格式由 32 位 float 型转为 16 为有符号整型），重采样（采样率由 48 kHz 转为 44.1 kHz），和通道数转换（从立体声双通道转为单通道）获得适合于送给 Android 平台播放的裸 PCM 音频流。

解码音频直接获得的是所谓 Non-interleaved 格式的裸 PCM 音频流，即内存中保存音频数据的方式为 LLLL...RRRR...，先是所有的左声道的数据，后是右声道的数据。这种格式保存的数据易于做数据处理，如编解码等，一般情况下音频数据处理都是针对某一个声道的连续数据的。但这种格式保存的数据对于播放和传输等场景不是很友好，这会要求播放设备或者是数据的接收端拿到所有数据才能开始播放。因而播放设备一般请求的裸 PCM 音频格式为所谓的 interleaved 的，即以 LRLRLRLR... 这种方式保存的数据。

在 ffmpeg 的术语和概念体系中，用 AVSampleFormat 的 planar 来表示这种音频数据保存格式的差异。对于 planar 的音频 PCM 数据保存格式，其保存音频数据的方式为 LLLL...RRRR...，如 AV_SAMPLE_FMT_S16P，AV_SAMPLE_FMT_FLTP 这些格式，而非 planar 的格式，其保存音频数据的方式则为 LRLRLRLR... 这种。

对于音频重采样，需要注意保持输入样本格式与实际数据格式的一致性，如解码出来的格式为 AV_SAMPLE_FMT_FLTP，但为了后面的处理方便，将解码出来的音频数据做了 interleave，将音频样本数据格式转为了 AV_SAMPLE_FMT_FLT，则也应该通过 `av_opt_set_sample_fmt(swr_ctx_, "in_sample_fmt", format, 0);` 将输入样本格式设置为 AV_SAMPLE_FMT_FLT。ffmpeg 提供了一个 API `enum AVSampleFormat av_get_alt_sample_fmt(enum AVSampleFormat sample_fmt, int planar);` 来做这种格式的转换。

整个解封装和解码过程由播放线程驱动。播放线程中，请求播放数据的回调上来的时候，检查缓冲区里是否有一定量的解码音视频数据，当数据不足时，通过 demuxer 和 decoder 解封装及解码音视频数据，并保存在缓冲区中。

解码和解封装过程的测试，可以将解码出来的数据直接保存在文件中，然后通过 ffplay 之类的工具播放检查。

采用 ffplay 查看YUV数据包括视频或者图片：
```
$ ffplay -f rawvideo -video_size 854x480   trailer.yuv
```
注：
（1）-f rawvideo：这个选项可加可不加。
（2）YUV 文件不包涵宽高数据，所以必须用 -video_size 指定宽和高，格式为：widthxheight
（3）test.yuv可以是一帧（图片）或者多帧（视频）数据

裸 PCM 数据播放命令如下：
```
$ ffplay -f f32le -ac 2 -ar 48000 trailer_.pcm
$ ffplay -f s16le -ac 1 -ar 44100 trailer_441.pcm
```
 上面的命令用于播放样本格式为 32 位 float，采样率为 48 kHz，通道数为 2 的裸 PCM 音频流。下面的命令用于播放样本格式为 16 位有符号整型，采样率为 44.1 kHz，通道数为 1 的裸 PCM 音频流

```
ffmpeg -i trailer.mp4  -ar 44100 -ac 1 output_1.mp4
```

参考了如下文档：
[利用av_read_frame解码h264、mp4多媒体文件为yuv](https://blog.csdn.net/liupenglove/article/details/107325547)
[ffmpeg 查看YUV图片/视频](https://blog.csdn.net/matrix_laboratory/article/details/49470689)
[FFmpeg解码MP4文件为h264和YUV文件](https://blog.csdn.net/hsq1596753614/article/details/81749576)
[FFmpeg —— 9.示例程序（三）：音视频分离（分离为PCM、YUV格式）](https://blog.csdn.net/guoyunfei123/article/details/105522557/?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-0.highlightwordscore&spm=1001.2101.3001.4242.1)
[FFmpeg 重采样示例代码](https://github.com/FFmpeg/FFmpeg/tree/master/doc/examples/resampling_audio.c)

## 音视频播放

视频渲染参考了 [Android 基于FFmpeg的视频播放渲染 CMake + ANativeWindow](https://blog.csdn.net/lakebobo/article/details/79578739)，也参考了其代码 [NDKtest](https://github.com/lakehubo/NDKtest)，这个 demo 在当前的开发环境下基本上没法跑，这个 demo 的 gradle 版本过老，同时它包的 ffmpeg so 只有 armeabi ABI 的，但代码还是有一定参考价值的。不过代码里有非常多的资源泄漏，既有 android 的资源泄漏，如 native window handle，也有 ffmpeg 的对象的泄漏，如 AVFrame 和 SwsContext 等。

如前所述，解封装和解码过程由音频播放线程驱动，音频播放线程会把解码后的音视频数据放进缓冲区中。应用中会启动专门的视频播放线程，视频播放线程定时向视频缓冲区请求 YUV 视频数据，如果视频 YUV 数据存在就拿去播放。

## 播放控制及音视频同步

Android 调用系统文件选择器并拿到文件路径的方法参考了 [Android 选择文件并返回路径](https://blog.csdn.net/tracydragonlxy/article/details/103509238) 及其代码 [ForeverLibrary](https://github.com/StickyTolt/ForeverLibrary/blob/master/alllibrary/src/main/java/com/martin/alllibrary/util/systemUtil/ContentUriUtil.java) 。

对于拖拽的实现，用户在拖拽进度条时，在一个全局状态中设置了进度值，在音频播放线程中请求音频数据的回调中检查进度值，并根据进度值，通过 ffmpeg 的 API 实现 seek。ffmpeg 的 seek API 用法参考了 [How to seek in FFmpeg C/C++](https://stackoverflow.com/questions/5261658/how-to-seek-in-ffmpeg-c-c) 等。

对于暂停的实现，用户在点击暂停键时，同样在全局状态中设置暂停状态，在音频播放线程中请求音频数据的回调中检查暂停状态，并采取一定的措施。

## 测试文件下载

1、地址：http://clips.vorwaerts-gmbh.de/big_buck_bunny.mp4 1分钟
2、地址：http://vjs.zencdn.net/v/oceans.mp4
3、地址：https://media.w3.org/2010/05/sintel/trailer.mp4  52秒
4、地址：http://mirror.aarnet.edu.au/pub/TED-talks/911Mothers_2010W-480p.mp4   10分钟

其他各种格式，MP4, flv, mkv, 3gp 视频下载
[https://www.sample-videos.com/index.php#sample-mp4-video](https://www.sample-videos.com/index.php#sample-mp4-video)


MPlayer 官方提供的各种视频格式的测试文件，载地址：http://samples.mplayerhq.hu/

测试视频的下载地址
http://ultravideo.cs.tut.fi/#testsequences
http://www.tanimoto.nuee.nagoya-u.ac.jp/~fukushima/mpegftv/
http://www.tanimoto.nuee.nagoya-u.ac.jp/~fukushima/mpegftv/Akko.htm


[测试视频的下载地址](https://www.i4k.xyz/article/qq_33212500/80648872)
[测试视频,音频,图片下载](https://www.cnuseful.com/down/)
[测试用视频下载](https://www.cnblogs.com/v5captain/p/12144699.html)
[测试用在线视频地址](https://www.jianshu.com/p/75a7db26d1d7)
[常用各种视频测试文件](https://blog.csdn.net/shadow_zed/article/details/102366755)
[sample-videos](https://sample-videos.com/index.php#sample-mp4-video)


##  开发过程中的问题分析调查

解封装出来的 H264 和 AAC 文件，由于 H264 有编码帧的分割字节串，AAC 有 ADTS header，转储的这些文件可以直接用一些播放器播放，如 VLC player。
对于解码出来的 YUV 和 PCM 数据，ffplay 可以用于播放裸 YUV 和 PCM 的音视频流，但需要正确指定裸流的参数。Audacity 可以用于播放裸的 PCM 音频流并查看音频信号的波形。

我们在开发调试时，也可以通过命令行方式显示的给某一个包授予权限，如下所示
adb shell pm grant com.xxx.xxx  android.permission.READ_EXTERNAL_STORAGE
