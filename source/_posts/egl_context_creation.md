---
title: EGL Context 创建
date: 2017-09-15 13:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
---

继续 EGL context 创建的分析。
<!--more-->
# eglInitialize()
来看 `EGL10.eglInitialize()` 的实现。`com.google.android.gles_jni.EGLImpl` 中，这个方法的实现如下：
```
    public native boolean     eglInitialize(EGLDisplay display, int[] major_minor);
```

它是一个本地层方法。其实际实现位于 `frameworks/base/core/jni/com_google_android_gles_jni_EGLImpl.cpp`：
```
static jfieldID gDisplay_EGLDisplayFieldID;
. . . . . .
static void nativeClassInit(JNIEnv *_env, jclass eglImplClass)
{
. . . . . .
    jclass display_class = _env->FindClass("com/google/android/gles_jni/EGLDisplayImpl");
    gDisplay_EGLDisplayFieldID = _env->GetFieldID(display_class, "mEGLDisplay", "J");
. . . . . .
}

. . . . . .
static inline EGLDisplay getDisplay(JNIEnv* env, jobject o) {
    if (!o) return EGL_NO_DISPLAY;
    return (EGLDisplay)env->GetLongField(o, gDisplay_EGLDisplayFieldID);
}
. . . . . .
static jboolean jni_eglInitialize(JNIEnv *_env, jobject _this, jobject display,
        jintArray major_minor) {
    if (display == NULL || (major_minor != NULL &&
            _env->GetArrayLength(major_minor) < 2)) {
        jniThrowException(_env, "java/lang/IllegalArgumentException", NULL);
        return JNI_FALSE;
    }

    EGLDisplay dpy = getDisplay(_env, display);
    EGLBoolean success = eglInitialize(dpy, NULL, NULL);
    if (success && major_minor) {
        int len = _env->GetArrayLength(major_minor);
        if (len) {
            // we're exposing only EGL 1.0
            jint* base = (jint *)_env->GetPrimitiveArrayCritical(major_minor, (jboolean *)0);
            if (len >= 1) base[0] = 1;
            if (len >= 2) base[1] = 0;
            _env->ReleasePrimitiveArrayCritical(major_minor, base, 0);
        }
    }
    return EglBoolToJBool(success);
}
```

`EGL10.eglInitialize()` 以 `EGLDisplay` 对象及一个 `int` 数组为参数，其中 `int` 数组为出参，用于返回版本号，并通过方法返回值表示初始化是否成功。

在 `jni_eglInitialize()` 中，它通过传入的 Java `EGLDisplay` 对象获得本地层 Display 对象的句柄，执行 EGL 库 (EGL wrapper 库) 的 `eglInitialize()` 函数完成初始化，并返回版本号，版本号总是 `1.0`。

EGL 库 (EGL wrapper 库) 的 `eglInitialize()` 的定义如下：
```
EGLBoolean eglInitialize(EGLDisplay dpy, EGLint *major, EGLint *minor)
{
    clearError();

    egl_display_ptr dp = get_display(dpy);
    if (!dp) return setError(EGL_BAD_DISPLAY, EGL_FALSE);

    EGLBoolean res = dp->initialize(major, minor);

    return res;
}
```

在这个函数中先获得本地层 Display 对象的指针对象 `egl_display_ptr`，然后执行 `egl_display_t::initialize(EGLint *major, EGLint *minor)`。

`egl_display_ptr` 定义（位于 `frameworks/native/opengl/libs/EGL/egl_display.h`）如下：
```
class egl_display_ptr {
public:
    explicit egl_display_ptr(egl_display_t* dpy): mDpy(dpy) {
        if (mDpy) {
            if (CC_UNLIKELY(!mDpy->enter())) {
                mDpy = NULL;
            }
        }
    }

    // We only really need a C++11 move constructor, not a copy constructor.
    // A move constructor would save an enter()/leave() pair on every EGL API
    // call. But enabling -std=c++0x causes lots of errors elsewhere, so I
    // can't use a move constructor until those are cleaned up.
    //
    // egl_display_ptr(egl_display_ptr&& other) {
    //     mDpy = other.mDpy;
    //     other.mDpy = NULL;
    // }
    //
    egl_display_ptr(const egl_display_ptr& other): mDpy(other.mDpy) {
        if (mDpy) {
            mDpy->enter();
        }
    }

    ~egl_display_ptr() {
        if (mDpy) {
            mDpy->leave();
        }
    }

    const egl_display_t* operator->() const { return mDpy; }
          egl_display_t* operator->()       { return mDpy; }

    const egl_display_t* get() const { return mDpy; }
          egl_display_t* get()       { return mDpy; }

    operator bool() const { return mDpy != NULL; }

private:
    egl_display_t* mDpy;

    // non-assignable
    egl_display_ptr& operator=(const egl_display_ptr&);
};
```

这是 `egl_display_t` 对象的智能指针，该指针对象创建时执行 `egl_display_t::enter()`，对象销毁时执行 `egl_display_t::leave()`。

`get_display()`定义（位于 `frameworks/native/opengl/libs/EGL/egl_display.h`）如下：
```
inline egl_display_ptr get_display(EGLDisplay dpy) {
    return egl_display_ptr(egl_display_t::get(dpy));
}
```

`egl_display_t::get(dpy)` 定义（位于 `frameworks/native/opengl/libs/EGL/egl_display.cpp`）如下：
```
egl_display_t egl_display_t::sDisplay[NUM_DISPLAYS];
. . . . . .
egl_display_t* egl_display_t::get(EGLDisplay dpy) {
    uintptr_t index = uintptr_t(dpy)-1U;
    if (index >= NUM_DISPLAYS || !sDisplay[index].isValid()) {
        return nullptr;
    }
    return &sDisplay[index];
}
```

本地层 Display 对象句柄为本地层全局静态 `egl_display_t` 对象数组中的索引值加 1。

`egl_display_t::initialize()` 定义如下：
```
EGLBoolean egl_display_t::initialize(EGLint *major, EGLint *minor) {

    {
        Mutex::Autolock _rf(refLock);

        refs++;
        if (refs > 1) {
            if (major != NULL)
                *major = VERSION_MAJOR;
            if (minor != NULL)
                *minor = VERSION_MINOR;
            while(!eglIsInitialized) refCond.wait(refLock);
            return EGL_TRUE;
        }

        while(eglIsInitialized) refCond.wait(refLock);
    }

    {
        Mutex::Autolock _l(lock);

        setGLHooksThreadSpecific(&gHooksNoContext);

        // initialize each EGL and
        // build our own extension string first, based on the extension we know
        // and the extension supported by our client implementation

        egl_connection_t* const cnx = &gEGLImpl;
        cnx->major = -1;
        cnx->minor = -1;
        if (cnx->dso) {
            EGLDisplay idpy = disp.dpy;
            if (cnx->egl.eglInitialize(idpy, &cnx->major, &cnx->minor)) {
                //ALOGD("initialized dpy=%p, ver=%d.%d, cnx=%p",
                //        idpy, cnx->major, cnx->minor, cnx);

                // display is now initialized
                disp.state = egl_display_t::INITIALIZED;

                // get the query-strings for this display for each implementation
                disp.queryString.vendor = cnx->egl.eglQueryString(idpy,
                        EGL_VENDOR);
                disp.queryString.version = cnx->egl.eglQueryString(idpy,
                        EGL_VERSION);
                disp.queryString.extensions = cnx->egl.eglQueryString(idpy,
                        EGL_EXTENSIONS);
                disp.queryString.clientApi = cnx->egl.eglQueryString(idpy,
                        EGL_CLIENT_APIS);

            } else {
                ALOGW("eglInitialize(%p) failed (%s)", idpy,
                        egl_tls_t::egl_strerror(cnx->egl.eglGetError()));
            }
        }

        // the query strings are per-display
        mVendorString.setTo(sVendorString);
        mVersionString.setTo(sVersionString);
        mClientApiString.setTo(sClientApiString);

        mExtensionString.setTo(gBuiltinExtensionString);
        char const* start = gExtensionString;
        do {
            // length of the extension name
            size_t len = strcspn(start, " ");
            if (len) {
                // NOTE: we could avoid the copy if we had strnstr.
                const String8 ext(start, len);
                if (findExtension(disp.queryString.extensions, ext.string(),
                        len)) {
                    mExtensionString.append(ext + " ");
                }
                // advance to the next extension name, skipping the space.
                start += len;
                start += (*start == ' ') ? 1 : 0;
            }
        } while (*start != '\0');

        egl_cache_t::get()->initialize(this);

        char value[PROPERTY_VALUE_MAX];
        property_get("debug.egl.finish", value, "0");
        if (atoi(value)) {
            finishOnSwap = true;
        }

        property_get("debug.egl.traceGpuCompletion", value, "0");
        if (atoi(value)) {
            traceGpuCompletion = true;
        }

        if (major != NULL)
            *major = VERSION_MAJOR;
        if (minor != NULL)
            *minor = VERSION_MINOR;

        mHibernation.setDisplayValid(true);
    }

    {
        Mutex::Autolock _rf(refLock);
        eglIsInitialized = true;
        refCond.broadcast();
    }

    return EGL_TRUE;
}
```

每次执行 `egl_display_t::initialize()` 初始化时，都会递增 `refs`，每次执行 `egl_display_t::terminate()` 终止时，则会递减它。在 `egl_display_t` 对象创建时，`refs` 被初始化为 0。

在 `egl_display_t::initialize()` 中，会首先处理初始化/终止的同步。增加 `refs` 之后，当它大于 1 时，表明对象已经被初始化了，可以直接返回，但如果 `eglIsInitialized` 为 `false` 表明，有另一个线程在初始化，但初始化还没有完成，这需要等待初始化的完成。

增加 `refs` 之后，当它等于 1 时，表明对象还没有初始化过，或者已经被终止，若 `eglIsInitialized` 为 `true` 则表明终止还没有结束，此时则需要等待终止过程结束。

***对于 `refs` 的同步，反正在这里都会同步地等待，为什么不把 `refLock` 锁的锁定范围定位整个函数呢？即锁定整个 `egl_display_t::initialize()` 函数或 `egl_display_t::terminate()`。***

处理初始化/终止的同步之后，才开始了正式的初始化。
1. 首先设置线程特有 GL Hooks 为`gHooksNoContext`。
2. 执行设备特有的实际 EGL 库实现的 `eglInitialize()` 函数来初始化，并从实际 EGL 库实现中查询一些与图形硬件特性有关的字符串，包括图形硬件的生产商，支持的 EGL 版本，支持的扩展和客户端 API。
对于 Google Pixel 设备而言，这些字符串的实际值如下：
```
queryString.vendor = Qualcomm Inc.
queryString.version = 1.4
queryString.extensions = EGL_QUALCOMM_shared_image EGL_KHR_image EGL_KHR_image_base EGL_QCOM_create_image EGL_QCOM_gpu_perf EGL_KHR_lock_surface EGL_KHR_lock_surface2 EGL_KHR_lock_surface3 EGL_KHR_fence_sync EGL_KHR_wait_sync EGL_KHR_cl_event EGL_KHR_cl_event2 EGL_KHR_reusable_sync EGL_IMG_context_priority EGL_KHR_gl_texture_2D_image EGL_KHR_gl_texture_cubemap_image EGL_KHR_gl_texture_3D_image EGL_KHR_gl_renderbuffer_image EGL_EXT_create_context_robustness EGL_EXT_yuv_surface EGL_ANDROID_blob_cache EGL_KHR_create_context EGL_KHR_gl_colorspace EGL_KHR_surfaceless_context EGL_KHR_create_context_no_error EGL_KHR_get_all_proc_addresses EGL_QCOM_lock_image2 EGL_KHR_partial_update EGL_EXT_protected_content EGL_KHR_mutable_render_buffer EGL_ANDROID_recordable EGL_ANDROID_native_fence_sync EGL_ANDROID_image_native_buffer EGL_ANDROID_framebuffer_target EGL_ANDROID_image_crop EGL_IMG_image_plane_attribs 
queryString.clientApi = OpenGL_ES
```
3. 前面查询的字符串为图形硬件的特性。`mVendorString` 等则为 Android 图形系统的特性信息，设备生产商，EGL 版本，客户端版本和支持的 EGL 扩展等信息。这些信息如下。
```
static char const * const sVendorString     = "Android";
static char const * const sVersionString    = "1.4 Android META-EGL";
static char const * const sClientApiString  = "OpenGL_ES";

extern char const * const gBuiltinExtensionString;
extern char const * const gExtensionString;
```

`gBuiltinExtensionString` 和 `gExtensionString` 定义（位于 `frameworks/native/opengl/libs/EGL/eglApi.cpp`）如下：
```
extern char const * const gBuiltinExtensionString =
        "EGL_KHR_get_all_proc_addresses "
        "EGL_ANDROID_presentation_time "
        "EGL_KHR_swap_buffers_with_damage "
        "EGL_ANDROID_create_native_client_buffer "
        "EGL_ANDROID_front_buffer_auto_refresh "
#if ENABLE_EGL_ANDROID_GET_FRAME_TIMESTAMPS
        "EGL_ANDROID_get_frame_timestamps "
#endif
        ;
extern char const * const gExtensionString  =
        "EGL_KHR_image "                        // mandatory
        "EGL_KHR_image_base "                   // mandatory
        "EGL_KHR_image_pixmap "
        "EGL_KHR_lock_surface "
#if (ENABLE_EGL_KHR_GL_COLORSPACE != 0)
        "EGL_KHR_gl_colorspace "
#endif
        "EGL_KHR_gl_texture_2D_image "
        "EGL_KHR_gl_texture_3D_image "
        "EGL_KHR_gl_texture_cubemap_image "
        "EGL_KHR_gl_renderbuffer_image "
        "EGL_KHR_reusable_sync "
        "EGL_KHR_fence_sync "
        "EGL_KHR_create_context "
        "EGL_KHR_config_attribs "
        "EGL_KHR_surfaceless_context "
        "EGL_KHR_stream "
        "EGL_KHR_stream_fifo "
        "EGL_KHR_stream_producer_eglsurface "
        "EGL_KHR_stream_consumer_gltexture "
        "EGL_KHR_stream_cross_process_fd "
        "EGL_EXT_create_context_robustness "
        "EGL_NV_system_time "
        "EGL_ANDROID_image_native_buffer "      // mandatory
        "EGL_KHR_wait_sync "                    // strongly recommended
        "EGL_ANDROID_recordable "               // mandatory
        "EGL_KHR_partial_update "               // strongly recommended
        "EGL_EXT_buffer_age "                   // strongly recommended with partial_update
        "EGL_KHR_create_context_no_error "
        "EGL_KHR_mutable_render_buffer "
        "EGL_EXT_yuv_surface "
        "EGL_EXT_protected_content "
        ;
```
这里初始化获得的 EGL 扩展特性是最终暴露给应用程序的 EGL 扩展特性。(gBuiltinExtensionString + gExtensionString) 为 Android 图形系统可以识别并应用的 EGL 扩展，其中 (gBuiltinExtensionString) 是完全由 Android 的 EGL wrapper 库实现并总是可用的。其余的 (gExtensionString) 则依赖于图形硬件 EGL 驱动中的支持。其中的一些必须得到支持，因为它们由 Android 系统本身使用，它们是上面在注释中标记了 `mandatory` 的那些，CDD 对它们有做要求。系统 *假设* 设备总是支持那些强制的 EGL 扩展，如果那些扩展缺失的话，则设备运行可能会出问题。
实际暴露给应用程序的 EGL 扩展特性将是 Android 图形系统可识别的与图形硬件支持的交集，其中 Android 图形系统可识别的包括由 EGL wrapper 实现的与需要图形硬件支持的。
4. 初始化 `egl_cache_t`。
5. 根据调试有关的一些系统属性设置状态。
6. 返回主、次版本号。
7. 将 `eglIsInitialized` 置为 `true` 并发送广播出去通知其它线程，初始化结束。

# eglChooseConfig()
接下来来看 `eglChooseConfig()`。在 `EGLImpl` 中，它同样为本地层方法：
```
    public native boolean     eglChooseConfig(EGLDisplay display, int[] attrib_list, EGLConfig[] configs, int config_size, int[] num_config);
```

`eglChooseConfig()` 方法的本地层实现（位于 `frameworks/base/core/jni/com_google_android_gles_jni_EGLImpl.cpp`）如下：
```
static jboolean jni_eglChooseConfig(JNIEnv *_env, jobject _this, jobject display,
        jintArray attrib_list, jobjectArray configs, jint config_size, jintArray num_config) {
    if (display == NULL
        || !validAttribList(_env, attrib_list)
        || (configs != NULL && _env->GetArrayLength(configs) < config_size)
        || (num_config != NULL && _env->GetArrayLength(num_config) < 1)) {
        jniThrowException(_env, "java/lang/IllegalArgumentException", NULL);
        return JNI_FALSE;
    }
    EGLDisplay dpy = getDisplay(_env, display);
    EGLBoolean success = EGL_FALSE;

    if (configs == NULL) {
        config_size = 0;
    }
    EGLConfig nativeConfigs[config_size];

    int num = 0;
    jint* attrib_base = beginNativeAttribList(_env, attrib_list);
    success = eglChooseConfig(dpy, attrib_base, configs ? nativeConfigs : 0, config_size, &num);
    endNativeAttributeList(_env, attrib_list, attrib_base);

    if (num_config != NULL) {
        _env->SetIntArrayRegion(num_config, 0, 1, (jint*) &num);
    }

    if (success && configs!=NULL) {
        for (int i=0 ; i<num ; i++) {
            jobject obj = _env->NewObject(gConfig_class, gConfig_ctorID, reinterpret_cast<jlong>(nativeConfigs[i]));
            _env->SetObjectArrayElement(configs, i, obj);
        }
    }
    return EglBoolToJBool(success);
}
```
`eglChooseConfig()` 接收 `EGLDisplay` 和 `int[]` 的 `attrib_list` 作为入参，接收 `EGLConfig[]` 的 `configs` 及 `int[]` 的 `num_config` 接收返回值，而 `int` 型的 `config_size` 则用来表示 `configs` 的长度。
 
`jni_eglChooseConfig()` 做的事情如下：
1. 检查参数有效性。参数无效的时候，抛出异常并返回。
2. 参数转换。将 Java 对象转换为本地层所用的结构。
3. 执行 EGL wrapper 库的 `eglChooseConfig()`。
4. 返回值给 Java 层。对于 `int[]` 的 `num_config`，通过 JNI 函数更新其内容。对于 `EGLConfig[]` 的 `configs`，则会先构造 Java 对象。
```
static jclass gConfig_class;

static jmethodID gConfig_ctorID;
. . . . . .
static void nativeClassInit(JNIEnv *_env, jclass eglImplClass)
{
    jclass config_class = _env->FindClass("com/google/android/gles_jni/EGLConfigImpl");
    gConfig_class = (jclass) _env->NewGlobalRef(config_class);
    gConfig_ctorID = _env->GetMethodID(gConfig_class,  "<init>", "(J)V");
    gConfig_EGLConfigFieldID = _env->GetFieldID(gConfig_class,  "mEGLConfig",  "J");
```

可以看一下这里用到的一些函数的定义：
```
static bool validAttribList(JNIEnv *_env, jintArray attrib_list) {
    if (attrib_list == NULL) {
        return true;
    }
    jsize len = _env->GetArrayLength(attrib_list);
    if (len < 1) {
        return false;
    }
    jint item = 0;
    _env->GetIntArrayRegion(attrib_list, len-1, 1, &item);
    return item == EGL_NONE;
}

static jint* beginNativeAttribList(JNIEnv *_env, jintArray attrib_list) {
    if (attrib_list != NULL) {
        return _env->GetIntArrayElements(attrib_list, (jboolean *)0);
    } else {
        return(jint*) gNull_attrib_base;
    }
}

static void endNativeAttributeList(JNIEnv *_env, jintArray attrib_list, jint* attrib_base) {
    if (attrib_list != NULL) {
        _env->ReleaseIntArrayElements(attrib_list, attrib_base, 0);
    }
}
```

EGL wrapper 库中 `eglChooseConfig()` 定义如下：
```
egl_display_ptr validate_display(EGLDisplay dpy) {
    egl_display_ptr dp = get_display(dpy);
    if (!dp)
        return setError(EGL_BAD_DISPLAY, egl_display_ptr(NULL));
    if (!dp->isReady())
        return setError(EGL_NOT_INITIALIZED, egl_display_ptr(NULL));

    return dp;
}
. . . . . .
EGLBoolean eglChooseConfig( EGLDisplay dpy, const EGLint *attrib_list,
                            EGLConfig *configs, EGLint config_size,
                            EGLint *num_config)
{
    clearError();

    const egl_display_ptr dp = validate_display(dpy);
    if (!dp) return EGL_FALSE;

    if (num_config==0) {
        return setError(EGL_BAD_PARAMETER, EGL_FALSE);
    }

    EGLBoolean res = EGL_FALSE;
    *num_config = 0;

    egl_connection_t* const cnx = &gEGLImpl;
    if (cnx->dso) {
        if (attrib_list) {
            char value[PROPERTY_VALUE_MAX];
            property_get("debug.egl.force_msaa", value, "false");

            if (!strcmp(value, "true")) {
                size_t attribCount = 0;
                EGLint attrib = attrib_list[0];

                // Only enable MSAA if the context is OpenGL ES 2.0 and
                // if no caveat is requested
                const EGLint *attribRendererable = NULL;
                const EGLint *attribCaveat = NULL;

                // Count the number of attributes and look for
                // EGL_RENDERABLE_TYPE and EGL_CONFIG_CAVEAT
                while (attrib != EGL_NONE) {
                    attrib = attrib_list[attribCount];
                    switch (attrib) {
                        case EGL_RENDERABLE_TYPE:
                            attribRendererable = &attrib_list[attribCount];
                            break;
                        case EGL_CONFIG_CAVEAT:
                            attribCaveat = &attrib_list[attribCount];
                            break;
                    }
                    attribCount++;
                }

                if (attribRendererable && attribRendererable[1] == EGL_OPENGL_ES2_BIT &&
                        (!attribCaveat || attribCaveat[1] != EGL_NONE)) {

                    // Insert 2 extra attributes to force-enable MSAA 4x
                    EGLint aaAttribs[attribCount + 4];
                    aaAttribs[0] = EGL_SAMPLE_BUFFERS;
                    aaAttribs[1] = 1;
                    aaAttribs[2] = EGL_SAMPLES;
                    aaAttribs[3] = 4;

                    memcpy(&aaAttribs[4], attrib_list, attribCount * sizeof(EGLint));

                    EGLint numConfigAA;
                    EGLBoolean resAA = cnx->egl.eglChooseConfig(
                            dp->disp.dpy, aaAttribs, configs, config_size, &numConfigAA);

                    if (resAA == EGL_TRUE && numConfigAA > 0) {
                        ALOGD("Enabling MSAA 4x");
                        *num_config = numConfigAA;
                        return resAA;
                    }
                }
            }
        }

        res = cnx->egl.eglChooseConfig(
                dp->disp.dpy, attrib_list, configs, config_size, num_config);
    }
    return res;
}
```

在这里，如果为了调试强制使用 MSAA，即多重采样抗锯齿（MultiSampling Anti-Aliasing，简称MSAA），则会插入 2 个额外的属性来强制启用 MSAA 4x，然后调用图形硬件特有的实际 EGL 实现库的 `eglChooseConfig()` 完成设置。否则直接用设备特有的实际 EGL 实现库的 `eglChooseConfig()` 完成设置。

我们前面 [在 Android 中使用 OpenGL](http://www.jianshu.com/p/b3ab069437c9) 一文中的示例里，`attrib_list` 的实际值如下：
```
        int[] mConfigSpec = { EGL10.EGL_RED_SIZE, 5, 
                EGL10.EGL_GREEN_SIZE, 6, EGL10.EGL_BLUE_SIZE, 5, 
                EGL10.EGL_DEPTH_SIZE, 16, EGL10.EGL_NONE };
```
这个配置大概用于配置 OpenGL ES 渲染所用的颜色模式，深度大小等。

# eglCreateWindowSurface()
然后来看 `eglCreateWindowSurface()`。在 `EGLImpl` 中，它有着如下这样的定义：
```
    public EGLSurface eglCreateWindowSurface(EGLDisplay display, EGLConfig config, Object native_window, int[] attrib_list) {
        Surface sur = null;
        if (native_window instanceof SurfaceView) {
            SurfaceView surfaceView = (SurfaceView)native_window;
            sur = surfaceView.getHolder().getSurface();
        } else if (native_window instanceof SurfaceHolder) {
            SurfaceHolder holder = (SurfaceHolder)native_window;
            sur = holder.getSurface();
        } else if (native_window instanceof Surface) {
            sur = (Surface) native_window;
        }

        long eglSurfaceId;
        if (sur != null) {
            eglSurfaceId = _eglCreateWindowSurface(display, config, sur, attrib_list);
        } else if (native_window instanceof SurfaceTexture) {
            eglSurfaceId = _eglCreateWindowSurfaceTexture(display, config,
                    native_window, attrib_list);
        } else {
            throw new java.lang.UnsupportedOperationException(
                "eglCreateWindowSurface() can only be called with an instance of " +
                "Surface, SurfaceView, SurfaceHolder or SurfaceTexture at the moment.");
        }

        if (eglSurfaceId == 0) {
            return EGL10.EGL_NO_SURFACE;
        }
        return new EGLSurfaceImpl( eglSurfaceId );
    }
. . . . . .
    private native long _eglCreateWindowSurface(EGLDisplay display, EGLConfig config, Object native_window, int[] attrib_list);
    private native long _eglCreateWindowSurfaceTexture(EGLDisplay display, EGLConfig config, Object native_window, int[] attrib_list);
```

`eglCreateWindowSurface()` 根据传入的本地窗口创建 `EGLSurface`。

Android 中可以作为本地窗口传入的有 `SurfaceView` 对象，`SurfaceView` 的 `SurfaceHolder` 或 `Surface` 对象，或者 `TextureView` 的 `SurfaceTexture`。

传入的 `native_window` 如果是 `Surface` 对象，则调用本地层方法 `_eglCreateWindowSurface()` 创建本地层 EGL Surface。如果传入的
 `native_window` 是 `SurfaceView` 或 `SurfaceHolder`，则会先从中获得 `Surface` 对象，然后调用本地层方法 `_eglCreateWindowSurface()` 创建本地层 EGL Surface。

如果传入的 `native_window` 是 `SurfaceTexture`，则会调用本地层方法 `_eglCreateWindowSurfaceTexture()` 创建本地层 EGL Surface。

有了地层 EGL Surface 之后，则创建对象 `EGLSurfaceImpl` 封装本地层 EGL Surface 并返回给调用者。

`EGLSurfaceImpl` 类定义如下：
```
public class EGLSurfaceImpl extends EGLSurface {
    long mEGLSurface;
    private long mNativePixelRef;
    public EGLSurfaceImpl() {
        mEGLSurface = 0;
        mNativePixelRef = 0;
    }
    public EGLSurfaceImpl(long surface) {
        mEGLSurface = surface;
        mNativePixelRef = 0;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EGLSurfaceImpl that = (EGLSurfaceImpl) o;

        return mEGLSurface == that.mEGLSurface;

    }

    @Override
    public int hashCode() {
        /*
         * Based on the algorithm suggested in
         * http://developer.android.com/reference/java/lang/Object.html
         */
        int result = 17;
        result = 31 * result + (int) (mEGLSurface ^ (mEGLSurface >>> 32));
        return result;
    }
}
```

`EGLSurfaceImpl` 仅仅是本地层对象句柄的简单封装。

本地层方法 `_eglCreateWindowSurface()` 的实现如下：
```
static jfieldID gConfig_EGLConfigFieldID;
. . . . . .
static inline EGLConfig getConfig(JNIEnv* env, jobject o) {
    if (!o) return 0;
    return (EGLConfig)env->GetLongField(o, gConfig_EGLConfigFieldID);
}
. . . . . .
static void nativeClassInit(JNIEnv *_env, jclass eglImplClass)
{
    jclass config_class = _env->FindClass("com/google/android/gles_jni/EGLConfigImpl");
    gConfig_class = (jclass) _env->NewGlobalRef(config_class);
    gConfig_ctorID = _env->GetMethodID(gConfig_class,  "<init>", "(J)V");
    gConfig_EGLConfigFieldID = _env->GetFieldID(gConfig_class,  "mEGLConfig",  "J");
. . . . . .
static jlong jni_eglCreateWindowSurface(JNIEnv *_env, jobject _this, jobject display,
        jobject config, jobject native_window, jintArray attrib_list) {
    if (display == NULL || config == NULL
        || !validAttribList(_env, attrib_list)) {
        jniThrowException(_env, "java/lang/IllegalArgumentException", NULL);
        return JNI_FALSE;
    }
    EGLDisplay dpy = getDisplay(_env, display);
    EGLContext cnf = getConfig(_env, config);
    sp<ANativeWindow> window;
    if (native_window == NULL) {
not_valid_surface:
        jniThrowException(_env, "java/lang/IllegalArgumentException",
                "Make sure the SurfaceView or associated SurfaceHolder has a valid Surface");
        return 0;
    }

    window = android_view_Surface_getNativeWindow(_env, native_window);
    if (window == NULL)
        goto not_valid_surface;

    jint* base = beginNativeAttribList(_env, attrib_list);
    EGLSurface sur = eglCreateWindowSurface(dpy, cnf, window.get(), base);
    endNativeAttributeList(_env, attrib_list, base);
    return reinterpret_cast<jlong>(sur);
}
```

在这个函数中，首先从 Display 和 Config 的 Java 对象中获得其相应的本地层对象的句柄。***上面的代码中用 `EGLContext cnf` 来保存 `getConfig(_env, config)` 的返回值。怀疑这是代码作者的 bug，只是由于 `EGLContext` 和 `EGLConfig` 都是 `void *` 的 `typedef`，所以才没有出现实际的问题。***

然后通过 `android_view_Surface_getNativeWindow()` （定义位于 `frameworks/base/core/jni/android_view_Surface.cpp`）从 Java 层的 `Surface` 对象获得本地层的 `ANativeWindow`：
```
sp<ANativeWindow> android_view_Surface_getNativeWindow(JNIEnv* env, jobject surfaceObj) {
    return android_view_Surface_getSurface(env, surfaceObj);
}

sp<Surface> android_view_Surface_getSurface(JNIEnv* env, jobject surfaceObj) {
    sp<Surface> sur;
    jobject lock = env->GetObjectField(surfaceObj,
            gSurfaceClassInfo.mLock);
    if (env->MonitorEnter(lock) == JNI_OK) {
        sur = reinterpret_cast<Surface *>(
                env->GetLongField(surfaceObj, gSurfaceClassInfo.mNativeObject));
        env->MonitorExit(lock);
    }
    env->DeleteLocalRef(lock);
    return sur;
}
```

得到的 `ANativeWindow` 实际为本地层的 `Surface` 类对象。

最后调用 EGL wrapper 库的 `eglCreateWindowSurface()` 创建 `EGLSurface` 并返回给调用者。

EGL wrapper 库的 `eglCreateWindowSurface()` 定义（位于 `frameworks/native/opengl/libs/EGL/eglApi.cpp`）如下：
```
EGLSurface eglCreateWindowSurface(  EGLDisplay dpy, EGLConfig config,
                                    NativeWindowType window,
                                    const EGLint *attrib_list)
{
    clearError();

    egl_connection_t* cnx = NULL;
    egl_display_ptr dp = validate_display_connection(dpy, cnx);
    if (dp) {
        EGLDisplay iDpy = dp->disp.dpy;

        int result = native_window_api_connect(window, NATIVE_WINDOW_API_EGL);
        if (result != OK) {
            ALOGE("eglCreateWindowSurface: native_window_api_connect (win=%p) "
                    "failed (%#x) (already connected to another API?)",
                    window, result);
            return setError(EGL_BAD_ALLOC, EGL_NO_SURFACE);
        }

        // Set the native window's buffers format to match what this config requests.
        // Whether to use sRGB gamma is not part of the EGLconfig, but is part
        // of our native format. So if sRGB gamma is requested, we have to
        // modify the EGLconfig's format before setting the native window's
        // format.

        // by default, just pick RGBA_8888
        EGLint format = HAL_PIXEL_FORMAT_RGBA_8888;
        android_dataspace dataSpace = HAL_DATASPACE_UNKNOWN;

        EGLint a = 0;
        cnx->egl.eglGetConfigAttrib(iDpy, config, EGL_ALPHA_SIZE, &a);
        if (a > 0) {
            // alpha-channel requested, there's really only one suitable format
            format = HAL_PIXEL_FORMAT_RGBA_8888;
        } else {
            EGLint r, g, b;
            r = g = b = 0;
            cnx->egl.eglGetConfigAttrib(iDpy, config, EGL_RED_SIZE,   &r);
            cnx->egl.eglGetConfigAttrib(iDpy, config, EGL_GREEN_SIZE, &g);
            cnx->egl.eglGetConfigAttrib(iDpy, config, EGL_BLUE_SIZE,  &b);
            EGLint colorDepth = r + g + b;
            if (colorDepth <= 16) {
                format = HAL_PIXEL_FORMAT_RGB_565;
            } else {
                format = HAL_PIXEL_FORMAT_RGBX_8888;
            }
        }

        // now select a corresponding sRGB format if needed
        if (attrib_list && dp->haveExtension("EGL_KHR_gl_colorspace")) {
            for (const EGLint* attr = attrib_list; *attr != EGL_NONE; attr += 2) {
                if (*attr == EGL_GL_COLORSPACE_KHR) {
                    if (ENABLE_EGL_KHR_GL_COLORSPACE) {
                        dataSpace = modifyBufferDataspace(dataSpace, *(attr+1));
                    } else {
                        // Normally we'd pass through unhandled attributes to
                        // the driver. But in case the driver implements this
                        // extension but we're disabling it, we want to prevent
                        // it getting through -- support will be broken without
                        // our help.
                        ALOGE("sRGB window surfaces not supported");
                        return setError(EGL_BAD_ATTRIBUTE, EGL_NO_SURFACE);
                    }
                }
            }
        }

        if (format != 0) {
            int err = native_window_set_buffers_format(window, format);
            if (err != 0) {
                ALOGE("error setting native window pixel format: %s (%d)",
                        strerror(-err), err);
                native_window_api_disconnect(window, NATIVE_WINDOW_API_EGL);
                return setError(EGL_BAD_NATIVE_WINDOW, EGL_NO_SURFACE);
            }
        }

        if (dataSpace != 0) {
            int err = native_window_set_buffers_data_space(window, dataSpace);
            if (err != 0) {
                ALOGE("error setting native window pixel dataSpace: %s (%d)",
                        strerror(-err), err);
                native_window_api_disconnect(window, NATIVE_WINDOW_API_EGL);
                return setError(EGL_BAD_NATIVE_WINDOW, EGL_NO_SURFACE);
            }
        }

        // the EGL spec requires that a new EGLSurface default to swap interval
        // 1, so explicitly set that on the window here.
        ANativeWindow* anw = reinterpret_cast<ANativeWindow*>(window);
        anw->setSwapInterval(anw, 1);

        EGLSurface surface = cnx->egl.eglCreateWindowSurface(
                iDpy, config, window, attrib_list);
        if (surface != EGL_NO_SURFACE) {
            egl_surface_t* s = new egl_surface_t(dp.get(), config, window,
                    surface, cnx);
            return s;
        }

        // EGLSurface creation failed
        native_window_set_buffers_format(window, 0);
        native_window_api_disconnect(window, NATIVE_WINDOW_API_EGL);
    }
    return EGL_NO_SURFACE;
}
```

`eglCreateWindowSurface()` 执行步骤如下：
第一步，获得 `egl_connection_t` 和 `egl_display_ptr`：
```
egl_display_ptr validate_display(EGLDisplay dpy) {
    egl_display_ptr dp = get_display(dpy);
    if (!dp)
        return setError(EGL_BAD_DISPLAY, egl_display_ptr(NULL));
    if (!dp->isReady())
        return setError(EGL_NOT_INITIALIZED, egl_display_ptr(NULL));

    return dp;
}

egl_display_ptr validate_display_connection(EGLDisplay dpy,
        egl_connection_t*& cnx) {
    cnx = NULL;
    egl_display_ptr dp = validate_display(dpy);
    if (!dp)
        return dp;
    cnx = &gEGLImpl;
    if (cnx->dso == 0) {
        return setError(EGL_BAD_CONFIG, egl_display_ptr(NULL));
    }
    return dp;
}
```

第二步，通过函数 `native_window_api_connect()` (定义位于 `system/core/include/system/window.h`) 为本地窗口连接 EGL API：
```
/*
 * native_window_api_connect(..., int api)
 * connects an API to this window. only one API can be connected at a time.
 * Returns -EINVAL if for some reason the window cannot be connected, which
 * can happen if it's connected to some other API.
 */
static inline int native_window_api_connect(
        struct ANativeWindow* window, int api)
{
    return window->perform(window, NATIVE_WINDOW_API_CONNECT, api);
}
```

第三步，根据配置计算颜色模式。

第四步，如果 GPU 图形硬件支持 `EGL_KHR_gl_colorspace` 扩展，则计算得到色彩空间。
```
// The EGL_KHR_gl_colorspace spec hasn't been ratified yet, so these haven't
// been added to the Khronos egl.h.
#define EGL_GL_COLORSPACE_KHR           EGL_VG_COLORSPACE
#define EGL_GL_COLORSPACE_SRGB_KHR      EGL_VG_COLORSPACE_sRGB
#define EGL_GL_COLORSPACE_LINEAR_KHR    EGL_VG_COLORSPACE_LINEAR

// Turn linear formats into corresponding sRGB formats when colorspace is
// EGL_GL_COLORSPACE_SRGB_KHR, or turn sRGB formats into corresponding linear
// formats when colorspace is EGL_GL_COLORSPACE_LINEAR_KHR. In any cases where
// the modification isn't possible, the original dataSpace is returned.
static android_dataspace modifyBufferDataspace( android_dataspace dataSpace,
                                                EGLint colorspace) {
    if (colorspace == EGL_GL_COLORSPACE_LINEAR_KHR) {
        return HAL_DATASPACE_SRGB_LINEAR;
    } else if (colorspace == EGL_GL_COLORSPACE_SRGB_KHR) {
        return HAL_DATASPACE_SRGB;
    }
    return dataSpace;
}
. . . . . .
        // now select a corresponding sRGB format if needed
        if (attrib_list && dp->haveExtension("EGL_KHR_gl_colorspace")) {
            for (const EGLint* attr = attrib_list; *attr != EGL_NONE; attr += 2) {
                if (*attr == EGL_GL_COLORSPACE_KHR) {
                    if (ENABLE_EGL_KHR_GL_COLORSPACE) {
                        dataSpace = modifyBufferDataspace(dataSpace, *(attr+1));
                    } else {
                        // Normally we'd pass through unhandled attributes to
                        // the driver. But in case the driver implements this
                        // extension but we're disabling it, we want to prevent
                        // it getting through -- support will be broken without
                        // our help.
                        ALOGE("sRGB window surfaces not supported");
                        return setError(EGL_BAD_ATTRIBUTE, EGL_NO_SURFACE);
                    }
                }
            }
        }
```

第五步，为 window (本地窗口) 设置颜色模式。***比较奇怪，获得色彩模式与设置色彩模式之间，为什么要隔一段计算颜色空间的逻辑呢？***
```
/*
 * native_window_set_buffers_format(..., int format)
 * All buffers dequeued after this call will have the format specified.
 *
 * If the specified format is 0, the default buffer format will be used.
 */
static inline int native_window_set_buffers_format(
        struct ANativeWindow* window,
        int format)
{
    return window->perform(window, NATIVE_WINDOW_SET_BUFFERS_FORMAT, format);
}
```

第六步，为 window 设置色彩空间。
```
/*
 * native_window_set_buffers_data_space(..., int dataSpace)
 * All buffers queued after this call will be associated with the dataSpace
 * parameter specified.
 *
 * dataSpace specifies additional information about the buffer that's dependent
 * on the buffer format and the endpoints. For example, it can be used to convey
 * the color space of the image data in the buffer, or it can be used to
 * indicate that the buffers contain depth measurement data instead of color
 * images.  The default dataSpace is 0, HAL_DATASPACE_UNKNOWN, unless it has been
 * overridden by the consumer.
 */
static inline int native_window_set_buffers_data_space(
        struct ANativeWindow* window,
        android_dataspace_t dataSpace)
{
    return window->perform(window, NATIVE_WINDOW_SET_BUFFERS_DATASPACE,
            dataSpace);
}
```

第七步，设置 Swap interval。
```
        // the EGL spec requires that a new EGLSurface default to swap interval
        // 1, so explicitly set that on the window here.
        ANativeWindow* anw = reinterpret_cast<ANativeWindow*>(window);
        anw->setSwapInterval(anw, 1);
```

第八步，调用实际的设备 EGL 库接口创建 `EGLSurface`，并据此创建 `egl_surface_t` 返回给调用者。

`EGLImpl` 中的本地层方法 `_eglCreateWindowSurfaceTexture()` 的实现如下：
```
static jlong jni_eglCreateWindowSurfaceTexture(JNIEnv *_env, jobject _this, jobject display,
        jobject config, jobject native_window, jintArray attrib_list) {
    if (display == NULL || config == NULL
        || !validAttribList(_env, attrib_list)) {
        jniThrowException(_env, "java/lang/IllegalArgumentException", NULL);
        return 0;
    }
    EGLDisplay dpy = getDisplay(_env, display);
    EGLContext cnf = getConfig(_env, config);
    sp<ANativeWindow> window;
    if (native_window == 0) {
not_valid_surface:
        jniThrowException(_env, "java/lang/IllegalArgumentException",
                "Make sure the SurfaceTexture is valid");
        return 0;
    }

    sp<IGraphicBufferProducer> producer(SurfaceTexture_getProducer(_env, native_window));
    window = new Surface(producer, true);
    if (window == NULL)
        goto not_valid_surface;

    jint* base = beginNativeAttribList(_env, attrib_list);
    EGLSurface sur = eglCreateWindowSurface(dpy, cnf, window.get(), base);
    endNativeAttributeList(_env, attrib_list, base);
    return reinterpret_cast<jlong>(sur);
}
```

这个函数与 `jni_eglCreateWindowSurface()` 函数类似。只是它会从传入的 Java 对象 `SurfaceTexture` 获得本地层的 `IGraphicBufferProducer` 对象引用，然后利用该引用创建本地层 `Surface`。后面的流程就与 `jni_eglCreateWindowSurface()` 的流程就完全一样。

由此不难理解本地层的 `Surface` 是 `IGraphicBufferProducer` 的封装，并提供函数来方便操作 `IGraphicBufferProducer`。

# eglCreateContext()
Android 的 OpenGL 应用，通过 `EGL.eglCreateContext()` 创建EGL context。在 `EGLImpl` 中该方法定义如下：
```
    public EGLContext eglCreateContext(EGLDisplay display, EGLConfig config, EGLContext share_context, int[] attrib_list) {
        long eglContextId = _eglCreateContext(display, config, share_context, attrib_list);
        if (eglContextId == 0) {
            return EGL10.EGL_NO_CONTEXT;
        }
        return new EGLContextImpl( eglContextId );
    }
. . . . . .
    private native long _eglCreateContext(EGLDisplay display, EGLConfig config, EGLContext share_context, int[] attrib_list);
```

这个方方法调用本地层方法 `_eglCreateContext()` 创建本地层 EGL context 并获得其 ID，然后创建 `EGLContextImpl` 对象并返回给调用者。

`EGLContextImpl` 定义如下：
```
public class EGLContextImpl extends EGLContext {
    private GLImpl mGLContext;
    long mEGLContext;

    public EGLContextImpl(long ctx) {
        mEGLContext = ctx;
        mGLContext = new GLImpl();
    }

    @Override
    public GL getGL() {
        return mGLContext;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EGLContextImpl that = (EGLContextImpl) o;

        return mEGLContext == that.mEGLContext;
    }

    @Override
    public int hashCode() {
        /*
         * Based on the algorithm suggested in
         * http://developer.android.com/reference/java/lang/Object.html
         */
        int result = 17;
        result = 31 * result + (int) (mEGLContext ^ (mEGLContext >>> 32));
        return result;
    }
}
```

它是对本地层的 EGL context ID 和 `GLImpl` 的简单封装。

本地层方法 `_eglCreateContext()` 实现如下：
```
static jlong jni_eglCreateContext(JNIEnv *_env, jobject _this, jobject display,
        jobject config, jobject share_context, jintArray attrib_list) {
    if (display == NULL || config == NULL || share_context == NULL
        || !validAttribList(_env, attrib_list)) {
        jniThrowException(_env, "java/lang/IllegalArgumentException", NULL);
        return JNI_FALSE;
    }
    EGLDisplay dpy = getDisplay(_env, display);
    EGLConfig  cnf = getConfig(_env, config);
    EGLContext shr = getContext(_env, share_context);
    jint* base = beginNativeAttribList(_env, attrib_list);
    EGLContext ctx = eglCreateContext(dpy, cnf, shr, base);
    endNativeAttributeList(_env, attrib_list, base);
    return reinterpret_cast<jlong>(ctx);
}
```

这个方法从传入的 Java 对象中获得相应的本地层对象，随后通过 EGL wrapper 库的 `eglCreateContext()` 创建本地层 EGL context 对象，并将其 ID 返回个调用者。

EGL wrapper 库中 `eglCreateContext()` 的定义如下：
```
EGLContext eglCreateContext(EGLDisplay dpy, EGLConfig config,
                            EGLContext share_list, const EGLint *attrib_list)
{
    clearError();

    egl_connection_t* cnx = NULL;
    const egl_display_ptr dp = validate_display_connection(dpy, cnx);
    if (dp) {
        if (share_list != EGL_NO_CONTEXT) {
            if (!ContextRef(dp.get(), share_list).get()) {
                return setError(EGL_BAD_CONTEXT, EGL_NO_CONTEXT);
            }
            egl_context_t* const c = get_context(share_list);
            share_list = c->context;
        }
        EGLContext context = cnx->egl.eglCreateContext(
                dp->disp.dpy, config, share_list, attrib_list);
        if (context != EGL_NO_CONTEXT) {
            // figure out if it's a GLESv1 or GLESv2
            int version = 0;
            if (attrib_list) {
                while (*attrib_list != EGL_NONE) {
                    GLint attr = *attrib_list++;
                    GLint value = *attrib_list++;
                    if (attr == EGL_CONTEXT_CLIENT_VERSION) {
                        if (value == 1) {
                            version = egl_connection_t::GLESv1_INDEX;
                        } else if (value == 2 || value == 3) {
                            version = egl_connection_t::GLESv2_INDEX;
                        }
                    }
                };
            }
            egl_context_t* c = new egl_context_t(dpy, context, config, cnx,
                    version);
            return c;
        }
    }
    return EGL_NO_CONTEXT;
}
```

创建 EGLContext 分为如下几步来完成：
1. 获得共享 Context。（***EGL 的共享 context 是个什么概念？***）
2. 根据共享 context 及传入的 EGLDisplay 等，通过图形硬件特有的实际 EGL 实现库的 `eglCreateContext()` 创建 `EGLContext`。
3. 获取传入的 `attrib_list` 中的 OpenGL ES 版本信息。
4. 根据前面创建的 `EGLContext` 和 OpenGL ES 版本信息创建 `egl_context_t`。

在 EGL Wrapper 库这一级，EGL context 即为 `egl_context_t` 类对象。该类定义如下：
```
class egl_context_t: public egl_object_t {
protected:
    ~egl_context_t() {}
public:
    typedef egl_object_t::LocalRef<egl_context_t, EGLContext> Ref;

    egl_context_t(EGLDisplay dpy, EGLContext context, EGLConfig config,
            egl_connection_t const* cnx, int version);

    void onLooseCurrent();
    void onMakeCurrent(EGLSurface draw, EGLSurface read);

    EGLDisplay dpy;
    EGLContext context;
    EGLConfig config;
    EGLSurface read;
    EGLSurface draw;
    egl_connection_t const* cnx;
    int version;
    String8 gl_extensions;
    Vector<String8> tokenized_gl_extensions;
};
```

此时这个 `egl_context_t` 还无法实际使用，它还没有关联 `EGLSurface`。

# eglMakeCurrent()
`eglMakeCurrent()` 为 `EGLContext` 关联 `EGLSurface`，并为当前线程启用该 `EGLContext`。`EGLImpl` 中 `eglMakeCurrent()` 定义如下：
```
    public native boolean     eglMakeCurrent(EGLDisplay display, EGLSurface draw, EGLSurface read, EGLContext context);
```

这是一个本地层方法，其本地层实现如下：
```
static jboolean jni_eglMakeCurrent(JNIEnv *_env, jobject _this, jobject display, jobject draw, jobject read, jobject context) {
    if (display == NULL || draw == NULL || read == NULL || context == NULL) {
        jniThrowException(_env, "java/lang/IllegalArgumentException", NULL);
        return JNI_FALSE;
    }
    EGLDisplay dpy = getDisplay(_env, display);
    EGLSurface sdr = getSurface(_env, draw);
    EGLSurface srd = getSurface(_env, read);
    EGLContext ctx = getContext(_env, context);
    return EglBoolToJBool(eglMakeCurrent(dpy, sdr, srd, ctx));
}
```

这个实现也很直接：它从传入的 Java 对象参数中获得它们本地层对象，然后调用 EGL wrapper 库的 `eglMakeCurrent()` 并将结果返回给调用者。

EGL wrapper 库中 `eglMakeCurrent()` 定义如下：
```
EGLBoolean eglMakeCurrent(  EGLDisplay dpy, EGLSurface draw,
                            EGLSurface read, EGLContext ctx)
{
    clearError();

    egl_display_ptr dp = validate_display(dpy);
    if (!dp) return setError(EGL_BAD_DISPLAY, EGL_FALSE);

    // If ctx is not EGL_NO_CONTEXT, read is not EGL_NO_SURFACE, or draw is not
    // EGL_NO_SURFACE, then an EGL_NOT_INITIALIZED error is generated if dpy is
    // a valid but uninitialized display.
    if ( (ctx != EGL_NO_CONTEXT) || (read != EGL_NO_SURFACE) ||
         (draw != EGL_NO_SURFACE) ) {
        if (!dp->isReady()) return setError(EGL_NOT_INITIALIZED, EGL_FALSE);
    }

    // get a reference to the object passed in
    ContextRef _c(dp.get(), ctx);
    SurfaceRef _d(dp.get(), draw);
    SurfaceRef _r(dp.get(), read);

    // validate the context (if not EGL_NO_CONTEXT)
    if ((ctx != EGL_NO_CONTEXT) && !_c.get()) {
        // EGL_NO_CONTEXT is valid
        return setError(EGL_BAD_CONTEXT, EGL_FALSE);
    }

    // these are the underlying implementation's object
    EGLContext impl_ctx  = EGL_NO_CONTEXT;
    EGLSurface impl_draw = EGL_NO_SURFACE;
    EGLSurface impl_read = EGL_NO_SURFACE;

    // these are our objects structs passed in
    egl_context_t       * c = NULL;
    egl_surface_t const * d = NULL;
    egl_surface_t const * r = NULL;

    // these are the current objects structs
    egl_context_t * cur_c = get_context(getContext());

    if (ctx != EGL_NO_CONTEXT) {
        c = get_context(ctx);
        impl_ctx = c->context;
    } else {
        // no context given, use the implementation of the current context
        if (draw != EGL_NO_SURFACE || read != EGL_NO_SURFACE) {
            // calling eglMakeCurrent( ..., !=0, !=0, EGL_NO_CONTEXT);
            return setError(EGL_BAD_MATCH, EGL_FALSE);
        }
        if (cur_c == NULL) {
            // no current context
            // not an error, there is just no current context.
            return EGL_TRUE;
        }
    }

    // retrieve the underlying implementation's draw EGLSurface
    if (draw != EGL_NO_SURFACE) {
        if (!_d.get()) return setError(EGL_BAD_SURFACE, EGL_FALSE);
        d = get_surface(draw);
        impl_draw = d->surface;
    }

    // retrieve the underlying implementation's read EGLSurface
    if (read != EGL_NO_SURFACE) {
        if (!_r.get()) return setError(EGL_BAD_SURFACE, EGL_FALSE);
        r = get_surface(read);
        impl_read = r->surface;
    }


    EGLBoolean result = dp->makeCurrent(c, cur_c,
            draw, read, ctx,
            impl_draw, impl_read, impl_ctx);

    if (result == EGL_TRUE) {
        if (c) {
            setGLHooksThreadSpecific(c->cnx->hooks[c->version]);
            egl_tls_t::setContext(ctx);
            _c.acquire();
            _r.acquire();
            _d.acquire();
        } else {
            setGLHooksThreadSpecific(&gHooksNoContext);
            egl_tls_t::setContext(EGL_NO_CONTEXT);
        }
    } else {
        // this will ALOGE the error
        egl_connection_t* const cnx = &gEGLImpl;
        result = setError(cnx->egl.eglGetError(), EGL_FALSE);
    }
    return result;
}
```

`eglMakeCurrent()` 首先获得线程当前关联的 EGL context：
```
static inline EGLContext getContext() { return egl_tls_t::getContext(); }
. . . . . .
    // these are the current objects structs
    egl_context_t * cur_c = get_context(getContext());
```

在 EGL  wrapper 这一级，通过线程局部存储保存当前线程关联的 `egl_context_t`：
```
EGLContext egl_tls_t::getContext() {
    if (sKey == TLS_KEY_NOT_INITIALIZED) {
        return EGL_NO_CONTEXT;
    }
    egl_tls_t* tls = (egl_tls_t *)pthread_getspecific(sKey);
    if (!tls) return EGL_NO_CONTEXT;
    return tls->ctx;
}
```

`frameworks/native/opengl/libs/EGL/egl_object.h` 中 `get_context()` 的定义如下：
```
template<typename NATIVE, typename EGL>
static inline NATIVE* egl_to_native_cast(EGL arg) {
    return reinterpret_cast<NATIVE*>(arg);
}
. . . . . .
static inline
egl_context_t* get_context(EGLContext context) {
    return egl_to_native_cast<egl_context_t>(context);
}
```

`eglMakeCurrent()` 接口有两个主要的功能：一是为一个有效的 EGLContext 关联 Surface，并把该 EGLContext 关联到当前线程，此时对 Surface 没有特别要求，这也就意味着 EGLContext 可以在不关联 Surface 被设置为当前 EGLContext；二是当传入的 EGLContext 为空时，则将当前线程关联的 EGLContext 接触关联，且当 EGLContext 为空时，传入的 Surface 必须为 `EGL_NO_SURFACE`。

`eglMakeCurrent()` 通过 ` egl_display_t::makeCurrent()` 执行底层图形硬件 EGL 库实现级别的 make current：
```
EGLBoolean egl_display_t::makeCurrent(egl_context_t* c, egl_context_t* cur_c,
        EGLSurface draw, EGLSurface read, EGLContext /*ctx*/,
        EGLSurface impl_draw, EGLSurface impl_read, EGLContext impl_ctx)
{
    EGLBoolean result;

    // by construction, these are either 0 or valid (possibly terminated)
    // it should be impossible for these to be invalid
    ContextRef _cur_c(cur_c);
    SurfaceRef _cur_r(cur_c ? get_surface(cur_c->read) : NULL);
    SurfaceRef _cur_d(cur_c ? get_surface(cur_c->draw) : NULL);

    { // scope for the lock
        Mutex::Autolock _l(lock);
        if (c) {
            result = c->cnx->egl.eglMakeCurrent(
                    disp.dpy, impl_draw, impl_read, impl_ctx);
            if (result == EGL_TRUE) {
                c->onMakeCurrent(draw, read);
                if (!cur_c) {
                    mHibernation.incWakeCount(HibernationMachine::STRONG);
                }
            }
        } else {
            result = cur_c->cnx->egl.eglMakeCurrent(
                    disp.dpy, impl_draw, impl_read, impl_ctx);
            if (result == EGL_TRUE) {
                cur_c->onLooseCurrent();
                mHibernation.decWakeCount(HibernationMachine::STRONG);
            }
        }
    }

    if (result == EGL_TRUE) {
        // This cannot be called with the lock held because it might end-up
        // calling back into EGL (in particular when a surface is destroyed
        // it calls ANativeWindow::disconnect
        _cur_c.release();
        _cur_r.release();
        _cur_d.release();
    }

    return result;
}
```

这里调用底层的图形硬件 EGL 库实现的 `eglMakeCurrent()` 完成操作，并根据需要回调 `egl_context_t` 的函数。随后释放当前的 context。

如果传入的 EGLContext 有效，且当前已经关联了一个 EGLContext，则新的替换旧的，但是旧的 `egl_context_t` 的回调 `onLooseCurrent()` 没有被调到。

如果是要为当前线程关联 EGLContext 的话，则设置线程局部的 GL Hooks 为 EGLContext 的 OpenGL ES 版本所对应的 Hooks，并在线程局部存储中保存 `EGLContext`，然后增加 EGLContext 和 Surface 的引用计数：
```
void setGLHooksThreadSpecific(gl_hooks_t const *value) {
    setGlThreadSpecific(value);
}
. . . . . .
void setGlThreadSpecific(gl_hooks_t const *value) {
    gl_hooks_t const * volatile * tls_hooks = get_tls_hooks();
    tls_hooks[TLS_SLOT_OPENGL_API] = value;
}
```

`egl_tls_t::setContext(ctx)` 定义如下：
```
egl_tls_t* egl_tls_t::getTLS() {
    egl_tls_t* tls = (egl_tls_t*)pthread_getspecific(sKey);
    if (tls == 0) {
        tls = new egl_tls_t;
        pthread_setspecific(sKey, tls);
    }
    return tls;
}
. . . . . .
void egl_tls_t::setContext(EGLContext ctx) {
    validateTLSKey();
    getTLS()->ctx = ctx;
}
```

如果是要清除当前线程关联的 EGLContext 的话，则置线程局部的 GL Hooks 为 `gHooksNoContext`，并设置当前线程关联的 `EGLContext` 为 `EGL_NO_CONTEXT`。

***猜测在设备特有的 EGL 库实现一级，无论是软件实现，还是硬件实现，都存在着另外的线程局部存储变量来保存那一级的 EGLContext 数据。***

自此之后，就可以使用 OpenGL ES 的接口来渲染图形了。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# Android OpenGL 图形系统分析系列文章
[在 Android 中使用 OpenGL](https://www.wolfcstech.com/2017/09/13/opengl_on_android_with_sv/)
[Android 图形驱动初始化](https://www.wolfcstech.com/2017/09/14/egl_init_drivers/)
[EGL Context 创建](https://www.wolfcstech.com/2017/09/15/egl_context_creation/)
[Android 图形系统之图形缓冲区分配](https://www.wolfcstech.com/2017/09/20/android_graphics_bufferalloc/)
[Android 图形系统之gralloc](https://www.wolfcstech.com/2017/09/21/android_graphics_gralloc/)