---
title: Cronet android设计与实现分析--库的初始化﻿
date: 2016-10-29 14:05:49
categories: 网络协议
tags:
- 源码分析
- Android
- 网络协议
- chromium
---
# Cronet的基本用法

我们从一段代码来开始我们对Cronet android设计与实现的探索，这段代码向我们展示要如何使用Cronet为android提供的Java接口来做HTTP请求。

<!--more-->

Cronet中主要通过`CronetEngine`来处理网络请求。这里我们专门创建一个class `CronetUtils`来管理`CronetEngine`对象，来处理`CronetEngine`对象的创建，HTTP请求的提交等：
```
public class CronetUtils {
    private static final String TAG = "CronetUtils";

    private static CronetUtils sInstance;

    CronetEngine mCronetEngine;

    private CronetUtils() {
    }

    public static synchronized CronetUtils getsInstance() {
        if (sInstance == null) {
            sInstance = new CronetUtils();
        }
        return sInstance;
    }

    public synchronized void init(Context context) {
        if (mCronetEngine == null) {
            CronetEngine.Builder builder = new CronetEngine.Builder(context);
            builder.enableHttpCache(CronetEngine.Builder.HTTP_CACHE_IN_MEMORY,
                    100 * 1024)
                    .enableHttp2(true)
                    .enableQuic(true)
                    .enableSDCH(true)
                    .setLibraryName("cronet");

            mCronetEngine = builder.build();
        }
    }

    public void getHtml(String url, UrlRequest.Callback callback) {
        startWithURL(url, callback);
    }

    private void startWithURL(String url, UrlRequest.Callback callback) {
        startWithURL(url, callback, null);
    }

    private void startWithURL(String url, UrlRequest.Callback callback, String postData) {
        Executor executor = Executors.newSingleThreadExecutor();
        UrlRequest.Builder builder = new UrlRequest.Builder(url, callback, executor, mCronetEngine);
        applyPostDataToUrlRequestBuilder(builder, executor, postData);
        builder.build().start();
    }

    private void applyPostDataToUrlRequestBuilder(
            UrlRequest.Builder builder, Executor executor, String postData) {
        if (postData != null && postData.length() > 0) {
            builder.setHttpMethod("POST");
            builder.addHeader("Content-Type", "application/x-www-form-urlencoded");
            builder.setUploadDataProvider(
                    UploadDataProviders.create(postData.getBytes()), executor);
        }
    }
}
```
`CronetEngine`是Cronet中资源管理的中心，这些资源包括用于异步执行各种网络请求的线程池，连接池等等。`CronetEngine`本身对于对象的创建没有施加太多的限制，但为了资源的使用效率及性能考虑，我们在这里通过将`CronetUtils`设计为单例，进而控制`CronetEngine`对象的创建为最多一个。

`CronetEngine`的对象需要通过`CronetEngine.Builder`来创建，我们首先创建`CronetEngine.Builder`的对象，然后为Builder设置我们希望Cronet所具有的特性，如开启HTTP2，开启QUIC，开启缓存，设置cornet的so文件的文件名等，然后调用builder.build()来创建对象。

提交网络请求则是，借助于UrlRequest.Builder创建一个UrlRequest，传入url，Executor，callback和CronetEngine等，然后执行UrlRequest.start()向Cronet提交HTTP请求。后续在HTTP请求的执行过程中遇到的事件，会通过callback传递给Cronet的客户端。`CronetEngine`的实现及具体的职责，以及Executor的作用，我们会在后面做详细分析。

而在我们的应用程序中需要实现Callback，用以处理HTTP请求执行过程中遇到的事件，获取执行HTTP请求所得的响应等。一个简单的示例Callback如下：
```
    private static class SimpleUrlRequestCallback extends UrlRequest.Callback {
        private ByteArrayOutputStream mBytesReceived = new ByteArrayOutputStream();
        private WritableByteChannel mReceiveChannel = Channels.newChannel(mBytesReceived);

        private long mRequestStartTime;

        public SimpleUrlRequestCallback(long startTime) {
            mRequestStartTime = startTime;
        }

        @Override
        public void onRedirectReceived(
                UrlRequest request, UrlResponseInfo info, String newLocationUrl) {
            request.followRedirect();
        }

        @Override
        public void onResponseStarted(UrlRequest request, UrlResponseInfo info) {
            request.read(ByteBuffer.allocateDirect(32 * 1024));
        }

        @Override
        public void onReadCompleted(
                UrlRequest request, UrlResponseInfo info, ByteBuffer byteBuffer) {
            byteBuffer.flip();

            try {
                mReceiveChannel.write(byteBuffer);
            } catch (IOException e) {
                org.chromium.base.Log.i(TAG, "IOException during ByteBuffer read. Details: ", e);
            }
            byteBuffer.clear();
            request.read(byteBuffer);
        }

        @Override
        public void onSucceeded(UrlRequest request, UrlResponseInfo info) {
            String receivedData = mBytesReceived.toString();
            receivedData = receivedData.replaceAll("^\"+", "").replaceAll("\"+$", "");
            final String url = info.getUrl();
            org.chromium.base.Log.i(TAG, "ReceivedData = " + receivedData);
            org.chromium.base.Log.i(TAG, "RequestUrl = " + url + " (" + info.getHttpStatusCode() + ")" +
                    "; ReceiveBytes = " + info.getReceivedBytesCount() +
                    "; RequestTime = " + (System.currentTimeMillis() - mRequestStartTime));
        }

        @Override
        public void onFailed(UrlRequest request, UrlResponseInfo info, UrlRequestException error) {
            org.chromium.base.Log.i(TAG, "****** onFailed, error is: %s", error.getMessage());

            final String url = info.getUrl();
            final String text = "Failed " + url + " (" + error.getMessage() + ")";
        }
    }

    View.OnClickListener mBtnClickListener = new View.OnClickListener() {

        @Override
        public void onClick(View v) {
            String url = "http://ip.taobao.com/service/getIpInfo.php?ip=123.58.191.68";

            if (R.id.btn_get_ip_info_with_cronet == v.getId()) {
                CronetUtils.getsInstance().init(MainActivity.this);
                SimpleUrlRequestCallback callback = new SimpleUrlRequestCallback(System.currentTimeMillis());
                CronetUtils.getsInstance().getHtml(url, callback);
```
我们的Callback通过扩展UrlRequest.Callback来实现，它会被传递给Cronet，其中的方法会在适当的时机被调用。回调方法大概有如下这些：

 * onRedirectReceived()：当收到一个重定向的响应时，这个回调方法会被调用。这个方法会在UrlRequest.start()被调用之后，而在UrlRequest.Callback.onResponseStart()之前被调用。重定向的响应的body，如果存在的话，将被忽略。通常在这个回调方法中，我们需要调用request.followRedirect()，以follow重定向。否则Cronet不会自动follow重定向。

 * onResponseStarted()：在所有的重定向都处理完了，且http响应的header都接收完，需要读取response的body时这个方法会被调用。一个请求的整个处理过程中，这个方法只会被调用一次。在这个方法中需要分配ByteBuffer，以用于http response body的读取过程。Response的body内容会首先被读取到这里分配的ByteBuffer中。

 * onReadCompleted()：http response body读取完成，或者ByteBuffer读满的时候，这个回调方法会被调用。在这个回调方法中，需要将ByteBuffer中的数据copy出来，清空ByteBuffer，然后重新启动读取。这里的回调方法实现，是将ByteBuffer的内容copy出来，借助于WritableByteChannel放进ByteArrayOutputStream中。

 * onSucceeded()和onFailed()：这两个回调方法分别在http请求执行完全成功时，和失败时调用。在这两个方法被调用之后，将不会再有其它的回调方法被调用。

 * onCanceled()：这是UrlRequest.Callback的另一个方法，尽管在我们的回调实现中没有实现这个方法。这个方法是在我们取消http请求执行时被调用。这个回调方法被调用之后将不会再有其它方法被调用。

总结一下，Cronet的使用方法如下：
1. 创建CronetEngine对象。
2. 实现UrlRequest.Callback类。
3. 根据Url等http请求相关信息创建UrlRequest。
4. 启动UrlRequest的执行。
 
# CronetEngine对象的创建

CronetEngine对象需要通过CronetEngine.Builder来创建。这里我们来看CronetEngine.Builder.build()方法的实现(chromium/src/components/cronet/android/api/src/org/chromium/net/CronetEngine.java)：

```
        /**
         * Build a {@link CronetEngine} using this builder's configuration.
         * @return constructed {@link CronetEngine}.
         */
        public CronetEngine build() {
            if (getUserAgent() == null) {
                setUserAgent(getDefaultUserAgent());
            }
            CronetEngine cronetEngine = null;
            if (!legacyMode()) {
                cronetEngine = createCronetEngine(this);
            }
            if (cronetEngine == null) {
                cronetEngine = new JavaCronetEngine(getUserAgent());
            }
            Log.i(TAG, "Using network stack: " + cronetEngine.getVersionString());
            // Clear MOCK_CERT_VERIFIER reference if there is any, since
            // the ownership has been transferred to the engine.
            mMockCertVerifier = 0;
            return cronetEngine;
        }
```

在这个方法中，主要做了如下这些事情：
1. 没有为CronetEngine显式地设置UserAgent时，构造一个UserAgent。
2. legacy模式是指，使用系统的HttpUrlConnection实现作为CronetEngine的Http stack，而不是使用chromium来执行网络请求。这个选项默认是关闭的。在不处于legacy模式时，会尝试调用CronetEngine.createCronetEngine()来创建CronetEngine对象，一个以chromium net为http stack的CronetEngine。
3. 因为某些原因CronetEngine.createCronetEngine()调用失败，或处于legacy模式时，创建JavaCronetEngine，也就是以系统的HttpUrlConnection实现作为Http stack的CronetEngine。
4. 将创建的CronetEngine返回给调用者。

CronetEngine.Builder的getDefaultUserAgent()的实现(chromium/src/components/cronet/android/api/src/org/chromium/net/CronetEngine.java)是这样的：
```
        /**
         * Constructs a User-Agent string including application name and version,
         * system build version, model and id, and Cronet version.
         *
         * @return User-Agent string.
         */
        public String getDefaultUserAgent() {
            return UserAgent.from(mContext);
        }
```

在UserAgent.from()定义了一套计算UserAgent的规则(chromium/src/components/cronet/android/api/src/org/chromium/net/UserAgent.java)：
```
    /**
     * Constructs a User-Agent string including application name and version,
     * system build version, model and Id, and Cronet version.
     * @param context the context to fetch the application name and version
     *         from.
     * @return User-Agent string.
     */
    public static String from(Context context) {
        StringBuilder builder = new StringBuilder();

        // Our package name and version.
        builder.append(context.getPackageName());
        builder.append('/');
        builder.append(versionFromContext(context));

        // The platform version.
        builder.append(" (Linux; U; Android ");
        builder.append(Build.VERSION.RELEASE);
        builder.append("; ");
        builder.append(Locale.getDefault().toString());

        String model = Build.MODEL;
        if (model.length() > 0) {
            builder.append("; ");
            builder.append(model);
        }

        String id = Build.ID;
        if (id.length() > 0) {
            builder.append("; Build/");
            builder.append(id);
        }

        builder.append(";");
        appendCronetVersion(builder);

        builder.append(')');

        return builder.toString();
    }
......
    private static int versionFromContext(Context context) {
        synchronized (sLock) {
            if (sVersionCode == VERSION_CODE_UNINITIALIZED) {
                PackageManager packageManager = context.getPackageManager();
                String packageName = context.getPackageName();
                try {
                    PackageInfo packageInfo = packageManager.getPackageInfo(
                            packageName, 0);
                    sVersionCode = packageInfo.versionCode;
                } catch (NameNotFoundException e) {
                    throw new IllegalStateException(
                            "Cannot determine package version");
                }
            }
            return sVersionCode;
        }
    }
......
    private static void appendCronetVersion(StringBuilder builder) {
        builder.append(" Cronet/");
        // TODO(pauljensen): This is the API version not the implementation
        // version. The implementation version may be more appropriate for the
        // UserAgent but is not available until after the CronetEngine is
        // instantiated. Down the road, if the implementation is loaded via
        // other means, this should be replaced with the implementation version.
        builder.append(ApiVersion.CRONET_VERSION);
    }
```
这个过程构造的UserAgent类似于下面这样：
```
com.netease.volleydemo/1 (Linux; U; Android 4.4.2; zh_CN; Galaxy Nexus; Build/KVT49L; Cronet/54.0.2826.0)
```

而CronetEngine.createCronetEngine() 通过反射来创建 org.chromium.net.impl.CronetUrlRequestContext对象，也就是以chromium net为http stack的CronetEngine：
```
    private static final String CRONET_URL_REQUEST_CONTEXT =
            "org.chromium.net.impl.CronetUrlRequestContext";

......

    private static CronetEngine createCronetEngine(Builder builder) {
        CronetEngine cronetEngine = null;
        try {
            Class<? extends CronetEngine> engineClass =
                    builder.getContext()
                            .getClassLoader()
                            .loadClass(CRONET_URL_REQUEST_CONTEXT)
                            .asSubclass(CronetEngine.class);
            Constructor<? extends CronetEngine> constructor =
                    engineClass.getConstructor(Builder.class);
            CronetEngine possibleEngine = constructor.newInstance(builder);
            if (possibleEngine.isEnabled()) {
                cronetEngine = possibleEngine;
            }
        } catch (ClassNotFoundException e) {
            // Leave as null.
        } catch (Exception e) {
            throw new IllegalStateException("Cannot instantiate: " + CRONET_URL_REQUEST_CONTEXT, e);
        }
        return cronetEngine;
    }
```

这里我们来看一下CronetUrlRequestContext的创建过程：
```
    @UsedByReflection("CronetEngine.java")
    public CronetUrlRequestContext(final CronetEngine.Builder builder) {
        CronetLibraryLoader.ensureInitialized(builder.getContext(), builder);
        nativeSetMinLogLevel(getLoggingLevel());
        synchronized (mLock) {
            mUrlRequestContextAdapter = nativeCreateRequestContextAdapter(
                    createNativeUrlRequestContextConfig(builder.getContext(), builder));
            if (mUrlRequestContextAdapter == 0) {
                throw new NullPointerException("Context Adapter creation failed.");
            }
            mNetworkQualityEstimatorEnabled = builder.networkQualityEstimatorEnabled();
        }

        // Init native Chromium URLRequestContext on main UI thread.
        Runnable task = new Runnable() {
            @Override
            public void run() {
                CronetLibraryLoader.ensureInitializedOnMainThread(builder.getContext());
                synchronized (mLock) {
                    // mUrlRequestContextAdapter is guaranteed to exist until
                    // initialization on main and network threads completes and
                    // initNetworkThread is called back on network thread.
                    nativeInitRequestContextOnMainThread(mUrlRequestContextAdapter);
                }
            }
        };
        // Run task immediately or post it to the UI thread.
        if (Looper.getMainLooper() == Looper.myLooper()) {
            task.run();
        } else {
            new Handler(Looper.getMainLooper()).post(task);
        }
    }
```
这个过程大体如下：
1. 加载并初始化cronet库。
2. 根据CronetEngine.Builder的用户设置创建NativeUrlRequestContextConfig。
3. 利用前面创建的NativeUrlRequestContextConfig创建RequestContextAdapter。
4. 在主线程中初始化RequestContextAdapter。

顺带简单地提一下CronetEngine的其它职责。除了创建CronetEngine实现对象外，CronetEngine还可以做这样的一些事情：
1. 创建UrlRequest。
2. 创建BidirectionalStream。
3. 获取一些Metrics，network quality信息及trace信息，如rtt，吞吐量等。
4. 创建URLConnection。
5. 获取Request的一些trace信息。

## 加载并初始化cronet库
CronetLibraryLoader.ensureInitialized() (chromium/src/components/cronet/android/java/src/org/chromium/net/impl/CronetLibraryLoader.java)被用于加载并初始化cronet库：
```
     /**
     * Ensure that native library is loaded and initialized. Can be called from
     * any thread, the load and initialization is performed on main thread.
     */
    public static void ensureInitialized(
            final Context context, final CronetEngine.Builder builder) {
        synchronized (sLoadLock) {
            if (sInitStarted) {
                return;
            }
            sInitStarted = true;
            ContextUtils.initApplicationContext(context.getApplicationContext());
            if (builder.libraryLoader() != null) {
                builder.libraryLoader().loadLibrary(builder.libraryName());
            } else {
                System.loadLibrary(builder.libraryName());
            }
            ContextUtils.initApplicationContextForNative();
            if (!ImplVersion.CRONET_VERSION.equals(nativeGetCronetVersion())) {
                throw new RuntimeException(String.format("Expected Cronet version number %s, "
                                + "actual version number %s.",
                        ImplVersion.CRONET_VERSION, nativeGetCronetVersion()));
            }
            Log.i(TAG, "Cronet version: %s, arch: %s", ImplVersion.CRONET_VERSION,
                    System.getProperty("os.arch"));
            // Init native Chromium CronetEngine on Main UI thread.
            Runnable task = new Runnable() {
                @Override
                public void run() {
                    ensureInitializedOnMainThread(context);
                }
            };
            // Run task immediately or post it to the UI thread.
            if (Looper.getMainLooper() == Looper.myLooper()) {
                task.run();
            } else {
                // The initOnMainThread will complete on the main thread prior
                // to other tasks posted to the main thread.
                new Handler(Looper.getMainLooper()).post(task);
            }
        }
    }

    /**
     * Ensure that the main thread initialization has completed. Can only be called from
     * the main thread. Ensures that the NetworkChangeNotifier is initialzied and the
     * main thread native MessageLoop is initialized.
     */
    static void ensureInitializedOnMainThread(Context context) {
        assert sInitStarted;
        assert Looper.getMainLooper() == Looper.myLooper();
        if (sMainThreadInitDone) {
            return;
        }
        NetworkChangeNotifier.init(context);
        // Registers to always receive network notifications. Note
        // that this call is fine for Cronet because Cronet
        // embedders do not have API access to create network change
        // observers. Existing observers in the net stack do not
        // perform expensive work.
        NetworkChangeNotifier.registerToReceiveNotificationsAlways();
        // registerToReceiveNotificationsAlways() is called before the native
        // NetworkChangeNotifierAndroid is created, so as to avoid receiving
        // the undesired initial network change observer notification, which
        // will cause active requests to fail with ERR_NETWORK_CHANGED.
        nativeCronetInitOnMainThread();
        sMainThreadInitDone = true;
    }

    // Native methods are implemented in cronet_library_loader.cc.
    private static native void nativeCronetInitOnMainThread();
    private static native String nativeGetCronetVersion();
```

加载主要是指动态链库so文件的加载，这个过程通常会在将so文件读入内存，解析引用的其它动态链接库或系统动态连接库的符号外，执行一些诸如native方法的注册等动作。而初始化则主要是在native方法nativeCronetInitOnMainThread()中初始化cronet native层的一些设施。

我们先来看so文件的加载。

### Cronet的JNI_OnLoad

我们知道，通过System.loadLibrary()加载动态链接库so文件时，so库中的JNI_OnLoad方法会被执行，以完成Java的native方法的注册，及其它的一些初始化动作。Cronet的动态链接库so文件的JNI_OnLoad(chromium/src/components/cronet/android/cronet_jni.cc)定义如下：
```
// This is called by the VM when the shared library is first loaded.
extern "C" jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  return cronet::CronetOnLoad(vm, reserved);
}
```
这里是将职责完全委托给了cronet::CronetOnLoad，而后者的定义(chromium/src/components/cronet/android/cronet_library_loader.cc)为：
```
namespace cronet {
namespace {
const base::android::RegistrationMethod kCronetRegisteredMethods[] = {
    {"BaseAndroid", base::android::RegisterJni},
    {"ChromiumUrlRequest", ChromiumUrlRequestRegisterJni},
    {"ChromiumUrlRequestContext", ChromiumUrlRequestContextRegisterJni},
    {"CronetBidirectionalStreamAdapter",
     CronetBidirectionalStreamAdapter::RegisterJni},
    {"CronetLibraryLoader", RegisterNativesImpl},
    {"CronetUploadDataStreamAdapter", CronetUploadDataStreamAdapterRegisterJni},
    {"CronetUrlRequestAdapter", CronetUrlRequestAdapterRegisterJni},
    {"CronetUrlRequestContextAdapter",
     CronetUrlRequestContextAdapterRegisterJni},
    {"NetAndroid", net::android::RegisterJni},
};

// MessageLoop on the main thread, which is where objects that receive Java
// notifications generally live.
base::MessageLoop* g_main_message_loop = nullptr;

net::NetworkChangeNotifier* g_network_change_notifier = nullptr;

bool RegisterJNI(JNIEnv* env) {
  return base::android::RegisterNativeMethods(
      env, kCronetRegisteredMethods, arraysize(kCronetRegisteredMethods));
}

bool Init() {
  url::Initialize();
  return true;
}

}  // namespace

// Checks the available version of JNI. Also, caches Java reflection artifacts.
jint CronetOnLoad(JavaVM* vm, void* reserved) {
  std::vector<base::android::RegisterCallback> register_callbacks;
  register_callbacks.push_back(base::Bind(&RegisterJNI));
  std::vector<base::android::InitCallback> init_callbacks;
  init_callbacks.push_back(base::Bind(&Init));
  if (!base::android::OnJNIOnLoadRegisterJNI(vm, register_callbacks) ||
      !base::android::OnJNIOnLoadInit(init_callbacks)) {
    return -1;
  }
  return JNI_VERSION_1_6;
}
......
}  // namespace cronet
```
这里定义了一个静态的表，表中的每一项都描述了一个包含native方法的类的类名及该类注册native方法的函数。在RegisterJNI()函数中，通过base::android::RegisterNativeMethods函数执行表中每一个类的native方法注册函数来注册native方法。

而RegisterJNI()函数则借助于base::android::OnJNIOnLoadRegisterJNI()来执行。

此外，这里还会执行一些初始化函数。

每个类都通过编译期间自动为它们产生的定义于头文件中的RegisterNativesImpl()函数来注册native方法。如ChromiumUrlRequestContext的ChromiumUrlRequestContextRegisterJni()：
```
// Explicitly register static JNI functions.
bool ChromiumUrlRequestContextRegisterJni(JNIEnv* env) {
  return RegisterNativesImpl(env);
}
```
而在RegisterNativesImpl()函数中，则通过JNIEnv的函数注册native方法，如CronetLibraryLoader的RegisterNativesImpl()函数：
```
static bool RegisterNativesImpl(JNIEnv* env) {
  if (base::android::IsManualJniRegistrationDisabled()) return true;

  const int kMethodsCronetLibraryLoaderSize =
      arraysize(kMethodsCronetLibraryLoader);

  if (env->RegisterNatives(CronetLibraryLoader_clazz(env),
                           kMethodsCronetLibraryLoader,
                           kMethodsCronetLibraryLoaderSize) < 0) {
    jni_generator::HandleRegistrationError(
        env, CronetLibraryLoader_clazz(env), __FILE__);
    return false;
  }

  return true;
}
```

总结一下，cronet动态连接库加载的过程为：
1. 定义native方法注册函数表，其中每一个表项定义了一个类的类名，及该类的native方法注册函数。
2. 通过base模块提供的util函数执行每一个类的native方法注册函数。

这个过程并没有什么特别的地方。比较特殊的是，native方法注册函数的定义方式，在cronet中，所有类的native方法注册函数，java的native方法的native层实现函数，及java native方法与native层的实现函数之间的对应关系表等，都是在构建期间动态产生的。这会给我们的代码阅读带来一点小小的障碍。

### cronet库的初始化

nativeCronetInitOnMainThread()用于初始化cronet库，它是在构建过程中动态产生的CronetLibraryLoader_jni.h文件 (位于chromium_android/src/out/Default/gen/components/cronet/android/cronet_jni_headers/cronet/jni/CronetLibraryLoader_jni.h)中定义的。nativeCronetInitOnMainThread()会将所有的事务委托给CronetInitOnMainThread()(chromium_android/src/components/cronet/android/cronet_library_loader.cc)，这个函数的定义为：
```
void CronetInitOnMainThread(JNIEnv* env, const JavaParamRef<jclass>& jcaller) {
#if !BUILDFLAG(USE_PLATFORM_ICU_ALTERNATIVES)
  base::i18n::InitializeICU();
#endif

  base::FeatureList::InitializeInstance(std::string(), std::string());
  // TODO(bengr): Remove once Data Reduction Proxy no longer needs this for
  // configuration information.
  base::CommandLine::Init(0, nullptr);
  DCHECK(!base::MessageLoop::current());
  DCHECK(!g_main_message_loop);
  g_main_message_loop = new base::MessageLoopForUI();
  base::MessageLoopForUI::current()->Start();
  DCHECK(!g_network_change_notifier);
  net::NetworkChangeNotifier::SetFactory(
      new net::NetworkChangeNotifierFactoryAndroid());
  g_network_change_notifier = net::NetworkChangeNotifier::Create();
}
```
在这里主要是创建了MessageLoop。***不过这里的MessageLoop又是用来做什么的呢？***

## 创建并初始化RequestContextAdapter
回到CronetEngine的创建过程。在加载及初始化cronet动态链接库之后，会创建RequestContextAdapter。

RequestContextAdapter通过native方法nativeCreateRequestContextAdapter()创建，这个方法也是在构建期自动定义的，其实现位于chromium_android/src/out/Default/gen/components/cronet/android/cronet_jni_headers/cronet/jni/CronetUrlRequestContext_jni.h。这个函数将职责委托给CreateRequestContextAdapter()函数(chromium_android/src/components/cronet/android/cronet_url_request_context_adapter.cc)，后者创建CronetURLRequestContextAdapter对象：
```
// Creates RequestContextAdater if config is valid URLRequestContextConfig,
// returns 0 otherwise.
static jlong CreateRequestContextAdapter(JNIEnv* env,
                                         const JavaParamRef<jclass>& jcaller,
                                         jlong jconfig) {
  std::unique_ptr<URLRequestContextConfig> context_config(
      reinterpret_cast<URLRequestContextConfig*>(jconfig));

  CronetURLRequestContextAdapter* context_adapter =
      new CronetURLRequestContextAdapter(std::move(context_config));
  return reinterpret_cast<jlong>(context_adapter);
}

......

CronetURLRequestContextAdapter::CronetURLRequestContextAdapter(
    std::unique_ptr<URLRequestContextConfig> context_config)
    : network_thread_(new base::Thread("network")),
      http_server_properties_manager_(nullptr),
      context_config_(std::move(context_config)),
      is_context_initialized_(false),
      default_load_flags_(net::LOAD_NORMAL) {
  base::Thread::Options options;
  options.message_loop_type = base::MessageLoop::TYPE_IO;
  network_thread_->StartWithOptions(options);
}
```
创建CronetURLRequestContextAdapter的过程主要是起了一个线程。***这个线程又是用来做什么的呢？***

接着继续来看org.chromium.net.impl.CronetUrlRequestContext中初始化的过程，也就是nativeInitRequestContextOnMainThread()方法：
```
extern "C" __attribute__((visibility("default")))
void
    Java_org_chromium_net_impl_CronetUrlRequestContext_nativeInitRequestContextOnMainThread(JNIEnv*
    env,
    jobject jcaller,
    jlong nativePtr) {
  CronetURLRequestContextAdapter* native =
      reinterpret_cast<CronetURLRequestContextAdapter*>(nativePtr);
  CHECK_NATIVE_PTR(env, jcaller, native, "InitRequestContextOnMainThread");
  return native->InitRequestContextOnMainThread(env,
      base::android::JavaParamRef<jobject>(env, jcaller));
}
```
这个方法调用CronetURLRequestContextAdapter的InitRequestContextOnMainThread()函数来执行初始化：
```
void CronetURLRequestContextAdapter::InitRequestContextOnMainThread(
    JNIEnv* env,
    const JavaParamRef<jobject>& jcaller) {
  base::android::ScopedJavaGlobalRef<jobject> jcaller_ref;
  jcaller_ref.Reset(env, jcaller);
  proxy_config_service_ = net::ProxyService::CreateSystemProxyConfigService(
      GetNetworkTaskRunner(), nullptr /* Ignored on Android */);
  net::ProxyConfigServiceAndroid* android_proxy_config_service =
      static_cast<net::ProxyConfigServiceAndroid*>(proxy_config_service_.get());
  // If a PAC URL is present, ignore it and use the address and port of
  // Android system's local HTTP proxy server. See: crbug.com/432539.
  // TODO(csharrison) Architect the wrapper better so we don't need to cast for
  // android ProxyConfigServices.
  android_proxy_config_service->set_exclude_pac_url(true);
  g_net_log.Get().EnsureInitializedOnMainThread();
  GetNetworkTaskRunner()->PostTask(
      FROM_HERE,
      base::Bind(&CronetURLRequestContextAdapter::InitializeOnNetworkThread,
                 base::Unretained(this), base::Passed(&context_config_),
                 jcaller_ref));
}
```
这个函数会创建proxy config service，初始化net log，然后抛一个task给URLRequestContextAdapter的network thread，task是CronetURLRequestContextAdapter::InitializeOnNetworkThread()：
```
void CronetURLRequestContextAdapter::InitializeOnNetworkThread(
    std::unique_ptr<URLRequestContextConfig> config,
    const base::android::ScopedJavaGlobalRef<jobject>&
        jcronet_url_request_context) {
  DCHECK(GetNetworkTaskRunner()->BelongsToCurrentThread());
  DCHECK(!is_context_initialized_);
  DCHECK(proxy_config_service_);
  // TODO(mmenke):  Add method to have the builder enable SPDY.
  net::URLRequestContextBuilder context_builder;

  std::unique_ptr<net::NetworkDelegate> network_delegate(
      new BasicNetworkDelegate());
#if defined(DATA_REDUCTION_PROXY_SUPPORT)
  DCHECK(!data_reduction_proxy_);
  // For now, the choice to enable the data reduction proxy happens once,
  // at initialization. It cannot be disabled thereafter.
  if (!config->data_reduction_proxy_key.empty()) {
    data_reduction_proxy_.reset(new CronetDataReductionProxy(
        config->data_reduction_proxy_key, config->data_reduction_primary_proxy,
        config->data_reduction_fallback_proxy,
        config->data_reduction_secure_proxy_check_url, config->user_agent,
        GetNetworkTaskRunner(), g_net_log.Get().net_log()));
    network_delegate = data_reduction_proxy_->CreateNetworkDelegate(
        std::move(network_delegate));
    context_builder.set_proxy_delegate(
        data_reduction_proxy_->CreateProxyDelegate());
    std::vector<std::unique_ptr<net::URLRequestInterceptor>> interceptors;
    interceptors.push_back(data_reduction_proxy_->CreateInterceptor());
    context_builder.SetInterceptors(std::move(interceptors));
  }
#endif  // defined(DATA_REDUCTION_PROXY_SUPPORT)
  context_builder.set_network_delegate(std::move(network_delegate));
  context_builder.set_net_log(g_net_log.Get().net_log());

  // Android provides a local HTTP proxy server that handles proxying when a PAC
  // URL is present. Create a proxy service without a resolver and rely on this
  // local HTTP proxy. See: crbug.com/432539.
  context_builder.set_proxy_service(
      net::ProxyService::CreateWithoutProxyResolver(
          std::move(proxy_config_service_), g_net_log.Get().net_log()));

  config->ConfigureURLRequestContextBuilder(&context_builder,
                                            g_net_log.Get().net_log(),
                                            GetFileThread()->task_runner());

  // Set up pref file if storage path is specified.
  if (!config->storage_path.empty()) {
    base::FilePath storage_path(config->storage_path);
    // Make sure storage directory has correct version.
    InitializeStorageDirectory(storage_path);
    base::FilePath filepath =
        storage_path.Append(FILE_PATH_LITERAL(kPrefsDirectoryName))
            .Append(FILE_PATH_LITERAL(kPrefsFileName));
    json_pref_store_ =
        new JsonPrefStore(filepath, GetFileThread()->task_runner(),
                          std::unique_ptr<PrefFilter>());
    context_builder.SetFileTaskRunner(GetFileThread()->task_runner());

    // Set up HttpServerPropertiesManager.
    PrefServiceFactory factory;
    factory.set_user_prefs(json_pref_store_);
    scoped_refptr<PrefRegistrySimple> registry(new PrefRegistrySimple());
    registry->RegisterDictionaryPref(kHttpServerProperties,
                                     new base::DictionaryValue());
    pref_service_ = factory.Create(registry.get());

    std::unique_ptr<net::HttpServerPropertiesManager>
        http_server_properties_manager(new net::HttpServerPropertiesManager(
            new PrefServiceAdapter(pref_service_.get()),
            GetNetworkTaskRunner()));
    http_server_properties_manager->InitializeOnNetworkThread();
    http_server_properties_manager_ = http_server_properties_manager.get();
    context_builder.SetHttpServerProperties(
        std::move(http_server_properties_manager));
  }

  // Explicitly disable the persister for Cronet to avoid persistence of dynamic
  // HPKP. This is a safety measure ensuring that nobody enables the persistence
  // of HPKP by specifying transport_security_persister_path in the future.
  context_builder.set_transport_security_persister_path(base::FilePath());

  // Disable net::CookieStore and net::ChannelIDService.
  context_builder.SetCookieAndChannelIdStores(nullptr, nullptr);

  if (config->enable_network_quality_estimator) {
    DCHECK(!network_quality_estimator_);
    network_quality_estimator_.reset(new net::NetworkQualityEstimator(
        std::unique_ptr<net::ExternalEstimateProvider>(),
        std::map<std::string, std::string>(), false, false));
    // Set the socket performance watcher factory so that network quality
    // estimator is notified of socket performance metrics from TCP and QUIC.
    context_builder.set_socket_performance_watcher_factory(
        network_quality_estimator_->GetSocketPerformanceWatcherFactory());
  }

  context_ = context_builder.Build();
  if (network_quality_estimator_)
    context_->set_network_quality_estimator(network_quality_estimator_.get());

  if (config->load_disable_cache)
    default_load_flags_ |= net::LOAD_DISABLE_CACHE;

  if (config->enable_sdch) {
    DCHECK(context_->sdch_manager());
    sdch_owner_.reset(
        new net::SdchOwner(context_->sdch_manager(), context_.get()));
    if (json_pref_store_) {
      sdch_owner_->EnablePersistentStorage(
          base::WrapUnique(new SdchOwnerPrefStorage(json_pref_store_.get())));
    }
  }

  if (config->enable_quic) {
    for (auto hint = config->quic_hints.begin();
         hint != config->quic_hints.end(); ++hint) {
      const URLRequestContextConfig::QuicHint& quic_hint = **hint;
      if (quic_hint.host.empty()) {
        LOG(ERROR) << "Empty QUIC hint host: " << quic_hint.host;
        continue;
      }

      url::CanonHostInfo host_info;
      std::string canon_host(net::CanonicalizeHost(quic_hint.host, &host_info));
      if (!host_info.IsIPAddress() &&
          !net::IsCanonicalizedHostCompliant(canon_host)) {
        LOG(ERROR) << "Invalid QUIC hint host: " << quic_hint.host;
        continue;
      }

      if (quic_hint.port <= std::numeric_limits<uint16_t>::min() ||
          quic_hint.port > std::numeric_limits<uint16_t>::max()) {
        LOG(ERROR) << "Invalid QUIC hint port: "
                   << quic_hint.port;
        continue;
      }

      if (quic_hint.alternate_port <= std::numeric_limits<uint16_t>::min() ||
          quic_hint.alternate_port > std::numeric_limits<uint16_t>::max()) {
        LOG(ERROR) << "Invalid QUIC hint alternate port: "
                   << quic_hint.alternate_port;
        continue;
      }

      url::SchemeHostPort quic_server("https", canon_host, quic_hint.port);
      net::AlternativeService alternative_service(
          net::AlternateProtocol::QUIC, "",
          static_cast<uint16_t>(quic_hint.alternate_port));
      context_->http_server_properties()->SetAlternativeService(
          quic_server, alternative_service, base::Time::Max());
    }
  }

  // If there is a cert_verifier, then populate its cache with
  // |cert_verifier_data|.
  if (!config->cert_verifier_data.empty() && context_->cert_verifier()) {
    std::string data;
    cronet_pb::CertVerificationCache cert_verification_cache;
    if (base::Base64Decode(config->cert_verifier_data, &data) &&
        cert_verification_cache.ParseFromString(data)) {
      DeserializeCertVerifierCache(cert_verification_cache,
                                   reinterpret_cast<net::CachingCertVerifier*>(
                                       context_->cert_verifier()));
    }
  }

  // Iterate through PKP configuration for every host.
  for (const auto& pkp : config->pkp_list) {
    // Add the host pinning.
    context_->transport_security_state()->AddHPKP(
        pkp->host, pkp->expiration_date, pkp->include_subdomains,
        pkp->pin_hashes, GURL::EmptyGURL());
  }

  context_->transport_security_state()
      ->SetEnablePublicKeyPinningBypassForLocalTrustAnchors(
          config->bypass_public_key_pinning_for_local_trust_anchors);

  JNIEnv* env = base::android::AttachCurrentThread();
  jcronet_url_request_context_.Reset(env, jcronet_url_request_context.obj());
  Java_CronetUrlRequestContext_initNetworkThread(
      env, jcronet_url_request_context.obj());

#if defined(DATA_REDUCTION_PROXY_SUPPORT)
  if (data_reduction_proxy_)
    data_reduction_proxy_->Init(true, GetURLRequestContext());
#endif
  is_context_initialized_ = true;
  while (!tasks_waiting_for_context_.empty()) {
    tasks_waiting_for_context_.front().Run();
    tasks_waiting_for_context_.pop();
  }
}
```
这里主要是通net::URLRequestContextBuilder，过创建了一个URLRequestContext对象，这个结构chromium net库的资源管理中心。

这里所做的具体的设置，我们目前先不管。
