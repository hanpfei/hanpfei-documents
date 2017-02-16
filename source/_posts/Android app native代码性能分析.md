分析我们app中native层的C/C++代码性能，能够方便我们找出其中的性能瓶颈，并在稍后做有针对性的优化。

# 下载android-ndk-profiler

工欲善其事，必先利其器，我们先要有良好的工具来支持我们做性能分析的愿望。android-ndk-profiler就是目前我们可用的比较好的工具。原来这个项目是托管在google的代码托管服务器的，[地址](https://code.google.com/p/android-ndk-profiler/)，但现在它已经被迁移到gihub。访问原来的地址时，会自动地被重定向到github上，[地址](https://github.com/richq/android-ndk-profiler)。这样也好，倒省掉我们这些天朝子民翻墙的麻烦了。

我们可以到**github**去下载android ndk profiler。可以下载master branch的zip压缩包，也可以把整个项目直接git clone下来，git clone下来可能要更好一点。这个项目的目录结构大体如下(2015-06-25这天的版本)：
```
hanpfei@hanpfei-ThundeRobot:~/android-ndk-profiler_repo$ ls -al
总用量 84
drwxrwxr-x  7 hanpfei hanpfei  4096  6月 25 11:26 .
drwxr-xr-x 54 hanpfei hanpfei  4096  6月 25 11:27 ..
-rw-rw-r--  1 hanpfei hanpfei 35147  6月 24 19:29 COPYING
drwxrwxr-x  2 hanpfei hanpfei  4096  6月 24 19:29 docs
drwxrwxr-x  3 hanpfei hanpfei  4096  6月 25 11:26 example
drwxrwxr-x  8 hanpfei hanpfei  4096  6月 25 11:26 .git
-rw-rw-r--  1 hanpfei hanpfei   365  6月 24 19:29 .gitignore
drwxrwxr-x  2 hanpfei hanpfei  4096  6月 25 11:26 jni
-rw-rw-r--  1 hanpfei hanpfei   974  6月 24 19:29 Makefile
-rw-rw-r--  1 hanpfei hanpfei   122  6月 25 11:26 ndk-excludes.txt
-rw-rw-r--  1 hanpfei hanpfei   791  6月 24 19:29 README.mkd
drwxrwxr-x  2 hanpfei hanpfei  4096  6月 24 19:29 test
-rw-rw-r--  1 hanpfei hanpfei   643  6月 25 11:26 .travis.yml
```

这个项目中提供的例子、文档什么的，可以参考一下。但真正需要被集成到我们项目里的就只有jni目录下面的那些。

我们把jni目录拷贝到另外一个地方，并重命名为android-ndk-profiler，比如：
```
hanpfei@hanpfei-ThundeRobot:~/android-ndk-profiler_repo$ cp -r jni ../android-ndk-profiler
```

后面我们会再来说明为什么要这么做。

# 修改项目jni目录下的Android.mk文件，加载android-ndk-profiler

将android-ndk-profiler集成进我们项目的第一步，就是修改jni目录下的Android.mk文加载android-ndk-profiler了：
```
# compile with profiling
LOCAL_CFLAGS := -pg

LOCAL_STATIC_LIBRARIES := android-ndk-profiler


# at the end of Android.mk
$(call import-module,android-ndk-profiler)
```

如果项目的编译还需要其它的flag，则把应该把"-pg"加在LOCAL_CFLAGS行的最后面，或者在适当的位置加一行，使用+=语法来添加这个flag，比如：
```
LOCAL_CFLAGS += -pg -DP2P_PROFILING
```

***-pg***是gcc的调试选项，它们会将profiling信息加入到最终生成的二进制代码中，profiling信息包含了更多的调试信息。

LOCAL_STATIC_LIBRARIES的值需要与android-ndk-profiler的Android.mk文件中定义的LOCAL_MODULE值对应，$(call import-module,android-ndk-profiler)这一行中，call import-module为Android编译系统的内置命令，而android-ndk-profiler则要与项目的目录名对应，这也就是上面我们为什么要把jni目录copy，重命名为android-ndk-profiler的原因。

# 设置NDK_MODULE_PATH环境变量

到目前为止，我们的项目都还无法编译通过。我们还需要设置NDK_MODULE_PATH环境变量。我们可以用export命令来设置这个环境变量，也可以将这个设置放在ndk-build命令中完成，而这个环境变量的值是android-ndk-profiler的父目录。比如，我们刚刚将android-ndk-profiler的jni目录拷贝到了用户根目录下的android-ndk-profiler目录，那么这个环境变量就应该被设置为～，即我们的用户根目录。

对于Eclipse环境，可以这样来设置：在Package Explorer中，鼠标选中项目，右键单击弹出菜单，Propertities -> C/C++ Build -> Build command，在最后加上NDK_MODULE_PATH=~。比如像下面这样：
```
ndk-build NDK_DEBUG=1 NDK_MODULE_PATH=~
```

如果没有做这样的设置的话，编译时会报错，由Eclipse的Console我们可以看到这样的报错信息：
```
/media/data/dev_tools/android-ndk-r9d/ndk-build NDK_DEBUG=1 
Android NDK: jni/Android.mk: Cannot find module with tag 'android-ndk-profiler' in import path    
jni/Android.mk:101: *** Android NDK: Aborting.    .  Stop.
Android NDK: Are you sure your NDK_MODULE_PATH variable is properly defined ?    
Android NDK: The following directories were searched:    
Android NDK:
```

# ucontext_t类型的定义

很不幸，在我们正确地设置了NDK_MODULE_PATH环境变量之后，还是无法通过编译。Eclipse Console中的报错信息如下：
```
**** Build of configuration Default for project peerTester_udt ****
/media/data/dev_tools/android-ndk-r9d/ndk-build NDK_DEBUG=1 NDK_MODULE_PATH=~ 
[armeabi-v7a] Gdbserver      : [arm-linux-androideabi-4.6] libs/armeabi-v7a/gdbserver
[armeabi-v7a] Gdbsetup       : libs/armeabi-v7a/gdb.setup
[armeabi-v7a] Compile thumb  : android-ndk-profiler <= prof.c
/home/hanpfei/android-ndk-profiler/prof.c: In function 'histogram_bin_incr':
/home/hanpfei/android-ndk-profiler/prof.c:150:2: error: unknown type name 'ucontext_t'
/home/hanpfei/android-ndk-profiler/prof.c:150:26: error: 'ucontext_t' undeclared (first use in this function)
/home/hanpfei/android-ndk-profiler/prof.c:150:26: note: each undeclared identifier is reported only once for each function it appears in
/home/hanpfei/android-ndk-profiler/prof.c:150:38: error: expected expression before ')' token
/home/hanpfei/android-ndk-profiler/prof.c:151:41: error: request for member 'uc_mcontext' in something not a structure or union
make: *** [obj/local/armeabi-v7a/objs-debug/android-ndk-profiler/prof.o] Error 1
```

提示找不到ucontext_t类型的定义。这究竟又是怎么一回事呢？在github上，这个项目的all commits列表中，我们看到有这么几笔commits的comments里提到了ucontext_t：

***b22514a66c0477dd34eadf7039e404bfc6f38c1b***
和
***516eee06cb18429dedabd145c9f00b0b14ff65c8***，
其中前者把jni/ucontext.h头文件中ucontext_t的定义给删了，而后者则干脆直接把这个头文件给彻底移除了。

由***b22514a66c0477dd34eadf7039e404bfc6f38c1b***这笔commit的comments，我们大体可以了解到ucontext_t的定义被从项目中移除的原因。ucontext本是GNU C库提供的一套标准的机制，用来创建、保存、切换用户态执行“上下文”，但无奈早期的NDK不支持这套机制，所以android-ndk-profiler项目就自己加了相关结构的定义。但自r10d版本开始，官方NDK已经是自带了对这套机制的支持，所以原来项目中ucontext_t的定义就显得多余了。

真操蛋，可怜了我们这群还在使用r9版NDK的人儿。如果能方便地把使用的NDK更新到r10d及之后的版本自然更好。如果不能，只有另想他法。

官方把jni/ucontext.h这个文件给删了，那大不了我们把项目的repo reset几笔change，重新找回这个文件就是了。reset到b22514a66c0477dd34eadf7039e404bfc6f38c1b之前的那个状态，大概就是reset 9笔changes：
```
hanpfei@hanpfei-ThundeRobot:~/android-ndk-profiler_repo$ git reset --hard HEAD~9
```

OK，找回了我们要的ucontext.h文件了。此外，还需要修改prof.c文件来包含那个头文件。

至此，我们的项目终于能够顺利通过编译了。

# 添加对监视函数的调用

经过了上面的步骤，android-ndk-profiler的功能终于被顺地编译进了我们的so。但要如何使android-ndk-profiler提供的功能运转起来呢？我们还需要在代码的适当位置，加入对监视函数的调用。

默认产生的profiling结果文件的路径为/sdcard/gmon.out，这个路径在prof.c中定义。因而我们需要让我们的app具有在sdcard上写文件的权限。我们可以在AndroidManifest.xml文件中加入下面一行：
```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

在我们native code的适当位置，加入对监视函数的调用：
```
/* in the start-up code */
monstartup("your_lib.so");

/* in the onPause or shutdown code */
moncleanup();
```

比如在start性质的函数中加入对函数monstartup()的调用，在end性质的函数中加入对moncleanup()函数的调用。要调用这两个函数，其它的一些基本设置必不可少：在调用这些函数的code文件中，inlcude相应的头文件，也就是prof.h；在Android.mk文件的搜索头文件路径列表中加入android-ndk-profiler的路径，也就是变量LOCAL_C_INCLUDES加入~/android-ndk-profiler路径。

# 产生并查看结果

moncleanup()执行结束之后，就在sdcard上产生了结果文件。我们可以使用gprof命令来产生profiling的报表文件。我们需要先用adb pull命令将gmon.out文件拷贝到自己的PC上，然后执行如下的命令产生结果：
```
$/media/data/dev_tools/android-ndk-r9d/toolchains/arm-linux-androideabi-4.8/prebuilt/linux-x86/bin/arm-linux-androideabi-gprof  libmoretvp2p.so  > gprof.txt
```

这个地方的so文件，是从项目的obj/local/armeabi-v7a/下copy出来的，而不是项目的libs/armeabi-v7a/下。使用后者来产生报表时会报错：
```
hanpfei@hanpfei-ThundeRobot:~/p2pclient_prof$ /media/data/dev_tools/android-ndk-r9d/toolchains/arm-linux-androideabi-4.8/prebuilt/linux-x86/bin/arm-linux-androideabi-gprof  libmoretvp2p.so  > gprof.txt
/media/data/dev_tools/android-ndk-r9d/toolchains/arm-linux-androideabi-4.8/prebuilt/linux-x86/bin/arm-linux-androideabi-gprof: file `libmoretvp2p.so' has no symbols
```

提示no symbols。 

我们可以用普通的文本编辑器打开我们在上一步中产生的gprof.txt文件：
```
Flat profile:


Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 21.69      0.82     0.82      299     2.74     2.74  CRcvLossList::CRcvLossList(int)
 20.11      1.58     0.76                             profCount
 12.70      2.06     0.48      299     1.61     1.61  CSndLossList::CSndLossList(int)
  5.29      2.26     0.20                             systemMessage
  4.76      2.44     0.18      301     0.60     0.60  CRcvBuffer::~CRcvBuffer()
  3.44      2.57     0.13      299     0.43     0.43  CRcvBuffer::CRcvBuffer(CUnitQueue*, int)
  2.12      2.65     0.08                             free_maps
  1.32      2.70     0.05     3052     0.02     0.02  CChannel::sendto(sockaddr const*, CPacket&) const
  1.32      2.75     0.05                             std::istream::sentry::sentry(std::istream&, bool)
  1.06      2.79     0.04       31     1.29     3.29  MORETV::HttpDownloadTask::downloadByHttp(std::string const&, Poco::AutoPtr<MORETV::TsDownloadSession>&)
  0.79      2.82     0.03    13956     0.00     0.00  Poco::AutoPtr<MORETV::TransportStreamImpl>::operator->()
  0.79      2.85     0.03     7322     0.00     0.00  MORETV::TransportStreamImpl::write(int, char const*, int)
  0.79      2.88     0.03     4391     0.01     0.01  std::_Rb_tree<int, std::pair<int const, CUDTSocket*>, std::_Select1st<std::pair<int const, CUDTSocket*> >, std::less<int>, std::allocator<std::pair<int const, CUDTSocket*> > >::_M_lower_bound(std::_Rb_tree_node<std::pair<int const, CUDTSocket*> >*, std::_Rb_tree_node<std::pair<int const, CUDTSocket*> >*, int const&)
  0.79      2.91     0.03                             sigemptyset
  0.53      2.93     0.02    33591     0.00     0.00  CChannel::recvfrom(sockaddr*, CPacket&) const
  0.53      2.95     0.02     1323     0.02     0.02  CACKWindow::store(int, int)
  0.53      2.97     0.02                             CRcvQueue::worker(void*)
  0.53      2.99     0.02                             std::string::replace(unsigned int, unsigned int, char const*, unsigned int)
  0.53      3.01     0.02                             std::locale::locale()
...
```

由这个结果，可以看到android-ndk-profiler工具本身的profCount()函数耗费了我们好多的CPU时间唉。

各个字段的具体含义如下：

**% time**
This is the percentage of the total execution time your program spent in this function. These should all add up to 100%.

**cumulative seconds**
This is the cumulative total number of seconds the computer spent executing this functions, plus the time spent in all the functions above this one in this table.

**self seconds**
This is the number of seconds accounted for by this function alone. The flat profile listing is sorted first by this number.

**calls**
This is the total number of times the function was called. If the function was never called, or the number of times it was called cannot be determined (probably because the function was not compiled with profiling enabled), the calls field is blank.

**self ms/call**
This represents the average number of milliseconds spent in this function per call, if this function is profiled. Otherwise, this field is blank for this function.

**total ms/call**
This represents the average number of milliseconds spent in this function and its descendants per call, if this function is profiled. Otherwise, this field is blank for this function. This is the only field in the flat profile that uses call graph analysis.

**name**
This is the name of the function. The flat profile is sorted by this field alphabetically after the self seconds and calls fields are sorted.

Done。