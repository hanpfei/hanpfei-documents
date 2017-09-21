---
title: Android 图形驱动初始化
date: 2017-09-14 13:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
---

从应用程序的角度看 OpenGL 图形系统的接口，主要包括两大部分，一部分是 EGL，它为 OpenGL 渲染准备环境；另一部分是 OpenGL，它执行图形渲染。通过这些接口构造渲染环境，并执行渲染的过程，可以参考 [在 Android 中使用 OpenGL](http://www.jianshu.com/p/b3ab069437c9)。
<!--more-->
对于 Android OpenGL 图形系统的实现的分析，从 EGL context 的创建开始。先来看一下获取 Display 的过程。首先来看 `EGLContext.getEGL()`：
```
public abstract class EGLContext
{
    private static final EGL EGL_INSTANCE = new com.google.android.gles_jni.EGLImpl();
    
    public static EGL getEGL() {
        return EGL_INSTANCE;
    }

    public abstract GL getGL();
}
```

返回的 `EGL` 实例为 `EGLImpl`，即我们在应用中使用的 `EGL` 实际为 `com.google.android.gles_jni.EGLImpl`。

# 获取 Display

然后来看 `eglGetDisplay()`。`EGLImpl` 的定义位于 `frameworks/base/opengl/java/com/google/android/gles_jni/EGLImpl.java`，其中 `eglGetDisplay()` 定义如下：
```
    public synchronized EGLDisplay eglGetDisplay(Object native_display) {
        long value = _eglGetDisplay(native_display);
        if (value == 0) {
            return EGL10.EGL_NO_DISPLAY;
        }
        if (mDisplay.mEGLDisplay != value)
            mDisplay = new EGLDisplayImpl(value);
        return mDisplay;
    }
. . . . . .
    private native long _eglGetDisplay(Object native_display);
```

这里主要通过调用本地层方法 `_eglGetDisplay()` 得到 Display，然后创建 `EGLDisplayImpl`。`_eglGetDisplay()` 返回底层的 Display 对象句柄。

`EGLDisplayImpl` 的实现很简单：
```
package com.google.android.gles_jni;

import javax.microedition.khronos.egl.*;

public class EGLDisplayImpl extends EGLDisplay {
    long mEGLDisplay;

    public EGLDisplayImpl(long dpy) {
        mEGLDisplay = dpy;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EGLDisplayImpl that = (EGLDisplayImpl) o;

        return mEGLDisplay == that.mEGLDisplay;

    }

    @Override
    public int hashCode() {
        /*
         * Based on the algorithm suggested in
         * http://developer.android.com/reference/java/lang/Object.html
         */
        int result = 17;
        result = 31 * result + (int) (mEGLDisplay ^ (mEGLDisplay >>> 32));
        return result;
    }
}
```

`EGLDispaly` 实际是一个标记接口：
```
package javax.microedition.khronos.egl;

public abstract class EGLDisplay
{
}
```

可见 `EGLDisplayImpl` 仅仅包装了本地层返回的 Display 对象句柄。

本地层 `_eglGetDisplay()` 的实现位于 `frameworks/base/core/jni/com_google_android_gles_jni_EGLImpl.cpp`：
```
static jlong jni_eglGetDisplay(JNIEnv *_env, jobject _this, jobject native_display) {
    return reinterpret_cast<jlong>(eglGetDisplay(EGL_DEFAULT_DISPLAY));
}
. . . . . .
{"_eglGetDisplay",   "(" OBJECT ")J", (void*)jni_eglGetDisplay },
```

这里通过调用 EGL 库的 `eglGetDisplay()` 获得 Display。`eglGetDisplay()` 的定义位于 `frameworks/native/opengl/libs/EGL/eglApi.cpp` ：
```
EGLDisplay eglGetDisplay(EGLNativeDisplayType display)
{
    clearError();

    uintptr_t index = reinterpret_cast<uintptr_t>(display);
    if (index >= NUM_DISPLAYS) {
        return setError(EGL_BAD_PARAMETER, EGL_NO_DISPLAY);
    }

    if (egl_init_drivers() == EGL_FALSE) {
        return setError(EGL_BAD_PARAMETER, EGL_NO_DISPLAY);
    }

    EGLDisplay dpy = egl_display_t::getFromNativeDisplay(display);
    return dpy;
}
```

在这个函数中，首先根据需要执行 `egl_init_drivers()` 初始化驱动库。然后通过 `egl_display_t::getFromNativeDisplay(display)` 获得 Dispaly。

`egl_display_t::getFromNativeDisplay(display)` 的定义（位于 `frameworks/native/opengl/libs/EGL/egl_display.cpp`）如下：
```
EGLDisplay egl_display_t::getDisplay(EGLNativeDisplayType display) {

    Mutex::Autolock _l(lock);

    // get our driver loader
    Loader& loader(Loader::getInstance());

    egl_connection_t* const cnx = &gEGLImpl;
    if (cnx->dso && disp.dpy == EGL_NO_DISPLAY) {
        EGLDisplay dpy = cnx->egl.eglGetDisplay(display);
        disp.dpy = dpy;
        if (dpy == EGL_NO_DISPLAY) {
            loader.close(cnx->dso);
            cnx->dso = NULL;
        }
    }

    return EGLDisplay(uintptr_t(display) + 1U);
}
```

在这个函数中，最为关键的就是，在 `disp.dpy` 为 `EGL_NO_DISPLAY` 时，通过 `cnx->egl.eglGetDisplay()` 初始化它了。

`EGLDisplay` 的定义如下：
```
typedef void *EGLDisplay;
```

它仅是 void 指针的 typedef。

总结一下获取 Display 的整个过程
 * 通过 `frameworks/native/opengl/libs/EGL` 初始化图形驱动；
 * 通过厂商提供的设备特有的 EGL 库接口初始化 Display。

# Android 图形驱动初始化
接下来更详细地看一下图形驱动初始化。这通过 `egl_init_drivers()` 完成，该函数定义 (位于`frameworks/native/opengl/libs/EGL/egl.cpp`) 如下：
```
egl_connection_t gEGLImpl;
gl_hooks_t gHooks[2];
. . . . . .
static EGLBoolean egl_init_drivers_locked() {
    if (sEarlyInitState) {
        // initialized by static ctor. should be set here.
        return EGL_FALSE;
    }

    // get our driver loader
    Loader& loader(Loader::getInstance());

    // dynamically load our EGL implementation
    egl_connection_t* cnx = &gEGLImpl;
    if (cnx->dso == 0) {
        cnx->hooks[egl_connection_t::GLESv1_INDEX] =
                &gHooks[egl_connection_t::GLESv1_INDEX];
        cnx->hooks[egl_connection_t::GLESv2_INDEX] =
                &gHooks[egl_connection_t::GLESv2_INDEX];
        cnx->dso = loader.open(cnx);
    }

    return cnx->dso ? EGL_TRUE : EGL_FALSE;
}

static pthread_mutex_t sInitDriverMutex = PTHREAD_MUTEX_INITIALIZER;

EGLBoolean egl_init_drivers() {
    EGLBoolean res;
    pthread_mutex_lock(&sInitDriverMutex);
    res = egl_init_drivers_locked();
    pthread_mutex_unlock(&sInitDriverMutex);
    return res;
}
```

图形驱动初始化通过 `Loader::open(egl_connection_t* cnx)` 完成，初始化的结果将存储于全局结构 `egl_connection_t gEGLImpl` 和 `gl_hooks_t gHooks[2]` 中。

来看一下 `gl_hooks_t` 的定义（位于 `frameworks/native/opengl/libs/hooks.h`）：
```
// maximum number of GL extensions that can be used simultaneously in
// a given process. this limitation exists because we need to have
// a static function for each extension and currently these static functions
// are generated at compile time.
#define MAX_NUMBER_OF_GL_EXTENSIONS 256
. . . . . .
#undef GL_ENTRY
#undef EGL_ENTRY
#define GL_ENTRY(_r, _api, ...) _r (*_api)(__VA_ARGS__);
#define EGL_ENTRY(_r, _api, ...) _r (*_api)(__VA_ARGS__);

struct egl_t {
    #include "EGL/egl_entries.in"
};

struct gl_hooks_t {
    struct gl_t {
        #include "entries.in"
    } gl;
    struct gl_ext_t {
        __eglMustCastToProperFunctionPointerType extensions[MAX_NUMBER_OF_GL_EXTENSIONS];
    } ext;
};
#undef GL_ENTRY
#undef EGL_ENTRY
```

其中 `__eglMustCastToProperFunctionPointerType` 定义 (位于`frameworks/native/opengl/include/EGL/egl.h`) 如下：
```
/* This is a generic function pointer type, whose name indicates it must
 * be cast to the proper type *and calling convention* before use.
 */
typedef void (*__eglMustCastToProperFunctionPointerType)(void);
```

`__eglMustCastToProperFunctionPointerType` 是函数指针类型。`struct gl_hooks_t` 的 `struct gl_ext_t ext` 即为函数指针表，它们用来描述 EGL 扩展接口。

`struct gl_hooks_t` 内的 `struct gl_t` 结构体，其结构体成员在另外一个文件，即 `entries.in` 中定义，该文件位于 `frameworks/native/opengl/libs/entries.in`：
```
GL_ENTRY(void, glActiveShaderProgram, GLuint pipeline, GLuint program)
GL_ENTRY(void, glActiveShaderProgramEXT, GLuint pipeline, GLuint program)
GL_ENTRY(void, glActiveTexture, GLenum texture)
GL_ENTRY(void, glAlphaFunc, GLenum func, GLfloat ref)
GL_ENTRY(void, glAlphaFuncQCOM, GLenum func, GLclampf ref)
. . . . . .
```

配合 `frameworks/native/opengl/libs/hooks.h` 中 `GL_ENTRY` 宏的定义：
```
#define GL_ENTRY(_r, _api, ...) _r (*_api)(__VA_ARGS__);
```

可以看到 `struct gl_hooks_t` 的 `struct gl_t gl` 的所有成员都是函数指针，即它是一个函数表，一个 OpenGL 接口函数的函数表。

上面看到的 `struct egl_t` 与 `struct gl_hooks_t` 的 `struct gl_t gl` 定义类似，只是它的结构体成员来自于另外一个文件 `frameworks/native/opengl/libs/EGL/egl_entries.in`：
```
EGL_ENTRY(EGLDisplay, eglGetDisplay, NativeDisplayType)
EGL_ENTRY(EGLBoolean, eglInitialize, EGLDisplay, EGLint*, EGLint*)
EGL_ENTRY(EGLBoolean, eglTerminate, EGLDisplay)
EGL_ENTRY(EGLBoolean, eglGetConfigs, EGLDisplay, EGLConfig*, EGLint, EGLint*)
EGL_ENTRY(EGLBoolean, eglChooseConfig, EGLDisplay, const EGLint *, EGLConfig *, EGLint, EGLint *)
. . . . . .
```
`EGL_ENTRY` 宏的定义与 `GL_ENTRY` 宏的完全相同。`struct egl_t` 同样为一个函数表，只是它是 EGL 接口的函数表。

再来看 `egl_connection_t` 的定义，位于 `frameworks/native/opengl/libs/EGL/egldefs.h`：
```
struct egl_connection_t {
    enum {
        GLESv1_INDEX = 0,
        GLESv2_INDEX = 1
    };

    inline egl_connection_t() : dso(0) { }
    void *              dso;
    gl_hooks_t *        hooks[2];
    EGLint              major;
    EGLint              minor;
    egl_t               egl;

    void*               libEgl;
    void*               libGles1;
    void*               libGles2;
};
```
这个结构包含了主、次版本号； 4 个指针；两个函数表的指针以及一个函数表。

`Loader::open(egl_connection_t* cnx)` 初始化图形驱动，主要是初始化这些函数表和指针。`Loader::open(egl_connection_t* cnx)` 的定义如下：
```
static void* load_wrapper(const char* path) {
    void* so = dlopen(path, RTLD_NOW | RTLD_LOCAL);
    ALOGE_IF(!so, "dlopen(\"%s\") failed: %s", path, dlerror());
    return so;
}

#ifndef EGL_WRAPPER_DIR
#if defined(__LP64__)
#define EGL_WRAPPER_DIR "/system/lib64"
#else
#define EGL_WRAPPER_DIR "/system/lib"
#endif
#endif

static void setEmulatorGlesValue(void) {
    char prop[PROPERTY_VALUE_MAX];
    property_get("ro.kernel.qemu", prop, "0");
    if (atoi(prop) != 1) return;

    property_get("ro.kernel.qemu.gles",prop,"0");
    if (atoi(prop) == 1) {
        ALOGD("Emulator has host GPU support, qemu.gles is set to 1.");
        property_set("qemu.gles", "1");
        return;
    }

    // for now, checking the following
    // directory is good enough for emulator system images
    const char* vendor_lib_path =
#if defined(__LP64__)
        "/vendor/lib64/egl";
#else
        "/vendor/lib/egl";
#endif

    const bool has_vendor_lib = (access(vendor_lib_path, R_OK) == 0);
    if (has_vendor_lib) {
        ALOGD("Emulator has vendor provided software renderer, qemu.gles is set to 2.");
        property_set("qemu.gles", "2");
    } else {
        ALOGD("Emulator without GPU support detected. "
              "Fallback to legacy software renderer, qemu.gles is set to 0.");
        property_set("qemu.gles", "0");
    }
}

void* Loader::open(egl_connection_t* cnx)
{
    void* dso;
    driver_t* hnd = 0;

    setEmulatorGlesValue();

    dso = load_driver("GLES", cnx, EGL | GLESv1_CM | GLESv2);
    if (dso) {
        hnd = new driver_t(dso);
    } else {
        // Always load EGL first
        dso = load_driver("EGL", cnx, EGL);
        if (dso) {
            hnd = new driver_t(dso);
            hnd->set( load_driver("GLESv1_CM", cnx, GLESv1_CM), GLESv1_CM );
            hnd->set( load_driver("GLESv2",    cnx, GLESv2),    GLESv2 );
        }
    }

    LOG_ALWAYS_FATAL_IF(!hnd, "couldn't find an OpenGL ES implementation");

    cnx->libEgl   = load_wrapper(EGL_WRAPPER_DIR "/libEGL.so");
    cnx->libGles2 = load_wrapper(EGL_WRAPPER_DIR "/libGLESv2.so");
    cnx->libGles1 = load_wrapper(EGL_WRAPPER_DIR "/libGLESv1_CM.so");

    LOG_ALWAYS_FATAL_IF(!cnx->libEgl,
            "couldn't load system EGL wrapper libraries");

    LOG_ALWAYS_FATAL_IF(!cnx->libGles2 || !cnx->libGles1,
            "couldn't load system OpenGL ES wrapper libraries");

    return (void*)hnd;
}
```

这个函数中，执行步骤如下：
1. 为设备是模拟器的情况，设置系统属性 `qemu.gles`。
2. 加载设备特有的图形驱动库，包括 EGL 库，OpenGL ES 1.0 和 2.0 的库。
3. 加载图形驱动 Wrapper，它们都位于 `/system/lib64` 或 `/system/lib`。

`egl_connection_t` 中的几个指针中，`dso` 实际为 `struct driver_t` 指针，该结构定义（位于 `frameworks/native/opengl/libs/EGL/Loader.h`）如下：
```
    struct driver_t {
        driver_t(void* gles);
        ~driver_t();
        status_t set(void* hnd, int32_t api);
        void* dso[3];
    };
```

`struct driver_t` 包含设备生产商提供的设备特有 EGL 和 OpenGL ES 实现库的句柄，如果 EGL 接口和 OpenGL 接口由单独的库实现，它包含一个库的句柄，即这个单独的库，如果 EGL 接口由不同的库实现，它则包含所有这些库的句柄。

`egl_connection_t` 的 `libEgl`、`libGles2` 和 `libGles1` 为动态链接库句柄，其中 `libEgl` 指向 Android 的 EGL Wrapper 库；`libGles1` 指向 Android 的 `GLESv1_CM` Wrapper 库；`libGles2` 指向 Android 的 `GLESv2` Wrapper 库。

然后来看 `Loader::load_driver()`：
```
/* This function is called to check whether we run inside the emulator,
 * and if this is the case whether GLES GPU emulation is supported.
 *
 * Returned values are:
 *  -1   -> not running inside the emulator
 *   0   -> running inside the emulator, but GPU emulation not supported
 *   1   -> running inside the emulator, GPU emulation is supported
 *          through the "emulation" host-side OpenGL ES implementation.
 *   2   -> running inside the emulator, GPU emulation is supported
 *          through a guest-side vendor driver's OpenGL ES implementation.
 */
static int
checkGlesEmulationStatus(void)
{
    /* We're going to check for the following kernel parameters:
     *
     *    qemu=1                      -> tells us that we run inside the emulator
     *    android.qemu.gles=<number>  -> tells us the GLES GPU emulation status
     *
     * Note that we will return <number> if we find it. This let us support
     * more additionnal emulation modes in the future.
     */
    char  prop[PROPERTY_VALUE_MAX];
    int   result = -1;

    /* First, check for qemu=1 */
    property_get("ro.kernel.qemu",prop,"0");
    if (atoi(prop) != 1)
        return -1;

    /* We are in the emulator, get GPU status value */
    property_get("qemu.gles",prop,"0");
    return atoi(prop);
}
. . . . . . 
void Loader::init_api(void* dso,
        char const * const * api,
        __eglMustCastToProperFunctionPointerType* curr,
        getProcAddressType getProcAddress)
{
    const ssize_t SIZE = 256;
    char scrap[SIZE];
    while (*api) {
        char const * name = *api;
        __eglMustCastToProperFunctionPointerType f =
            (__eglMustCastToProperFunctionPointerType)dlsym(dso, name);
        if (f == NULL) {
            // couldn't find the entry-point, use eglGetProcAddress()
            f = getProcAddress(name);
        }
        if (f == NULL) {
            // Try without the OES postfix
            ssize_t index = ssize_t(strlen(name)) - 3;
            if ((index>0 && (index<SIZE-1)) && (!strcmp(name+index, "OES"))) {
                strncpy(scrap, name, index);
                scrap[index] = 0;
                f = (__eglMustCastToProperFunctionPointerType)dlsym(dso, scrap);
                //ALOGD_IF(f, "found <%s> instead", scrap);
            }
        }
        if (f == NULL) {
            // Try with the OES postfix
            ssize_t index = ssize_t(strlen(name)) - 3;
            if (index>0 && strcmp(name+index, "OES")) {
                snprintf(scrap, SIZE, "%sOES", name);
                f = (__eglMustCastToProperFunctionPointerType)dlsym(dso, scrap);
                //ALOGD_IF(f, "found <%s> instead", scrap);
            }
        }
        if (f == NULL) {
            //ALOGD("%s", name);
            f = (__eglMustCastToProperFunctionPointerType)gl_unimplemented;

            /*
             * GL_EXT_debug_label is special, we always report it as
             * supported, it's handled by GLES_trace. If GLES_trace is not
             * enabled, then these are no-ops.
             */
            if (!strcmp(name, "glInsertEventMarkerEXT")) {
                f = (__eglMustCastToProperFunctionPointerType)gl_noop;
            } else if (!strcmp(name, "glPushGroupMarkerEXT")) {
                f = (__eglMustCastToProperFunctionPointerType)gl_noop;
            } else if (!strcmp(name, "glPopGroupMarkerEXT")) {
                f = (__eglMustCastToProperFunctionPointerType)gl_noop;
            }
        }
        *curr++ = f;
        api++;
    }
}

void *Loader::load_driver(const char* kind,
        egl_connection_t* cnx, uint32_t mask)
{
    class MatchFile {
    public:
        static String8 find(const char* kind) {
            String8 result;
            int emulationStatus = checkGlesEmulationStatus();
            switch (emulationStatus) {
                case 0:
#if defined(__LP64__)
                    result.setTo("/system/lib64/egl/libGLES_android.so");
#else
                    result.setTo("/system/lib/egl/libGLES_android.so");
#endif
                    return result;
                case 1:
                    // Use host-side OpenGL through the "emulation" library
#if defined(__LP64__)
                    result.appendFormat("/system/lib64/egl/lib%s_emulation.so", kind);
#else
                    result.appendFormat("/system/lib/egl/lib%s_emulation.so", kind);
#endif
                    return result;
                default:
                    // Not in emulator, or use other guest-side implementation
                    break;
            }

            String8 pattern;
            pattern.appendFormat("lib%s", kind);
            const char* const searchPaths[] = {
#if defined(__LP64__)
                    "/vendor/lib64/egl",
                    "/system/lib64/egl"
#else
                    "/vendor/lib/egl",
                    "/system/lib/egl"
#endif
            };

            // first, we search for the exact name of the GLES userspace
            // driver in both locations.
            // i.e.:
            //      libGLES.so, or:
            //      libEGL.so, libGLESv1_CM.so, libGLESv2.so

            for (size_t i=0 ; i<NELEM(searchPaths) ; i++) {
                if (find(result, pattern, searchPaths[i], true)) {
                    return result;
                }
            }

            // for compatibility with the old "egl.cfg" naming convention
            // we look for files that match:
            //      libGLES_*.so, or:
            //      libEGL_*.so, libGLESv1_CM_*.so, libGLESv2_*.so

            pattern.append("_");
            for (size_t i=0 ; i<NELEM(searchPaths) ; i++) {
                if (find(result, pattern, searchPaths[i], false)) {
                    return result;
                }
            }

            // we didn't find the driver. gah.
            result.clear();
            return result;
        }

    private:
        static bool find(String8& result,
                const String8& pattern, const char* const search, bool exact) {
            if (exact) {
                String8 absolutePath;
                absolutePath.appendFormat("%s/%s.so", search, pattern.string());
                if (!access(absolutePath.string(), R_OK)) {
                    result = absolutePath;
                    return true;
                }
                return false;
            }

            DIR* d = opendir(search);
            if (d != NULL) {
                struct dirent cur;
                struct dirent* e;
                while (readdir_r(d, &cur, &e) == 0 && e) {
                    if (e->d_type == DT_DIR) {
                        continue;
                    }
                    if (!strcmp(e->d_name, "libGLES_android.so")) {
                        // always skip the software renderer
                        continue;
                    }
                    if (strstr(e->d_name, pattern.string()) == e->d_name) {
                        if (!strcmp(e->d_name + strlen(e->d_name) - 3, ".so")) {
                            result.clear();
                            result.appendFormat("%s/%s", search, e->d_name);
                            closedir(d);
                            return true;
                        }
                    }
                }
                closedir(d);
            }
            return false;
        }
    };


    String8 absolutePath = MatchFile::find(kind);
    if (absolutePath.isEmpty()) {
        // this happens often, we don't want to log an error
        return 0;
    }
    const char* const driver_absolute_path = absolutePath.string();

    void* dso = dlopen(driver_absolute_path, RTLD_NOW | RTLD_LOCAL);
    if (dso == 0) {
        const char* err = dlerror();
        ALOGE("load_driver(%s): %s", driver_absolute_path, err?err:"unknown");
        return 0;
    }

    if (mask & EGL) {
        ALOGD("EGL loaded %s", driver_absolute_path);
        getProcAddress = (getProcAddressType)dlsym(dso, "eglGetProcAddress");

        ALOGE_IF(!getProcAddress,
                "can't find eglGetProcAddress() in %s", driver_absolute_path);

        egl_t* egl = &cnx->egl;
        __eglMustCastToProperFunctionPointerType* curr =
            (__eglMustCastToProperFunctionPointerType*)egl;
        char const * const * api = egl_names;
        while (*api) {
            char const * name = *api;
            __eglMustCastToProperFunctionPointerType f =
                (__eglMustCastToProperFunctionPointerType)dlsym(dso, name);
            if (f == NULL) {
                // couldn't find the entry-point, use eglGetProcAddress()
                f = getProcAddress(name);
                if (f == NULL) {
                    f = (__eglMustCastToProperFunctionPointerType)0;
                }
            }
            *curr++ = f;
            api++;
        }
    }

    if (mask & GLESv1_CM) {
        ALOGD("GLESv1_CM loaded %s", driver_absolute_path);
        init_api(dso, gl_names,
            (__eglMustCastToProperFunctionPointerType*)
                &cnx->hooks[egl_connection_t::GLESv1_INDEX]->gl,
            getProcAddress);
    }

    if (mask & GLESv2) {
      ALOGD("GLESv2 loaded %s", driver_absolute_path);
      init_api(dso, gl_names,
            (__eglMustCastToProperFunctionPointerType*)
                &cnx->hooks[egl_connection_t::GLESv2_INDEX]->gl,
            getProcAddress);
    }

    return dso;
}
```

`Loader::load_driver()` 通过三个步骤完成驱动库加载：
第一步，找到驱动库文件的路径。
```
    class MatchFile {
    public:
        static String8 find(const char* kind) {
            String8 result;
            int emulationStatus = checkGlesEmulationStatus();
            switch (emulationStatus) {
                case 0:
#if defined(__LP64__)
                    result.setTo("/system/lib64/egl/libGLES_android.so");
#else
                    result.setTo("/system/lib/egl/libGLES_android.so");
#endif
                    return result;
                case 1:
                    // Use host-side OpenGL through the "emulation" library
#if defined(__LP64__)
                    result.appendFormat("/system/lib64/egl/lib%s_emulation.so", kind);
#else
                    result.appendFormat("/system/lib/egl/lib%s_emulation.so", kind);
#endif
                    return result;
                default:
                    // Not in emulator, or use other guest-side implementation
                    break;
            }

            String8 pattern;
            pattern.appendFormat("lib%s", kind);
            const char* const searchPaths[] = {
#if defined(__LP64__)
                    "/vendor/lib64/egl",
                    "/system/lib64/egl"
#else
                    "/vendor/lib/egl",
                    "/system/lib/egl"
#endif
            };

            // first, we search for the exact name of the GLES userspace
            // driver in both locations.
            // i.e.:
            //      libGLES.so, or:
            //      libEGL.so, libGLESv1_CM.so, libGLESv2.so

            for (size_t i=0 ; i<NELEM(searchPaths) ; i++) {
                if (find(result, pattern, searchPaths[i], true)) {
                    return result;
                }
            }

            // for compatibility with the old "egl.cfg" naming convention
            // we look for files that match:
            //      libGLES_*.so, or:
            //      libEGL_*.so, libGLESv1_CM_*.so, libGLESv2_*.so

            pattern.append("_");
            for (size_t i=0 ; i<NELEM(searchPaths) ; i++) {
                if (find(result, pattern, searchPaths[i], false)) {
                    return result;
                }
            }

            // we didn't find the driver. gah.
            result.clear();
            return result;
        }

    private:
        static bool find(String8& result,
                const String8& pattern, const char* const search, bool exact) {
            if (exact) {
                String8 absolutePath;
                absolutePath.appendFormat("%s/%s.so", search, pattern.string());
                if (!access(absolutePath.string(), R_OK)) {
                    result = absolutePath;
                    return true;
                }
                return false;
            }

            DIR* d = opendir(search);
            if (d != NULL) {
                struct dirent cur;
                struct dirent* e;
                while (readdir_r(d, &cur, &e) == 0 && e) {
                    if (e->d_type == DT_DIR) {
                        continue;
                    }
                    if (!strcmp(e->d_name, "libGLES_android.so")) {
                        // always skip the software renderer
                        continue;
                    }
                    if (strstr(e->d_name, pattern.string()) == e->d_name) {
                        if (!strcmp(e->d_name + strlen(e->d_name) - 3, ".so")) {
                            result.clear();
                            result.appendFormat("%s/%s", search, e->d_name);
                            closedir(d);
                            return true;
                        }
                    }
                }
                closedir(d);
            }
            return false;
        }
    };


    String8 absolutePath = MatchFile::find(kind);
    if (absolutePath.isEmpty()) {
        // this happens often, we don't want to log an error
        return 0;
    }
```

`checkGlesEmulationStatus()` 函数，在不是运行于模拟器中时，返回 -1；在运行于模拟器中，但不支持 GPU 硬件模拟时返回 0；在运行于模拟器中，通过宿主机端的 OpenGL ES 实现来支持 GPU 硬件模拟时，返回 1；在运行于模拟器中，通过 Android 客户系统端的生产商驱动的 OpenGL ES 实现来支持 GPU 硬件模拟时，返回 2。对于运行环境的这种判断，主要依据两个系统属性，即 `ro.kernel.qemu` 和 `qemu.gles`。

只有在 `checkGlesEmulationStatus()` 返回 0，即运行于模拟器，但不支持 OpenGL ES 的 GPU 硬件模拟时，EGL 和 OpenGL ES 库采用软件实现 `/system/lib64/egl/libGLES_android.so` 或 `/system/lib/egl/libGLES_android.so`。

如果是物理设备，或者模拟器开启了 OpenGL ES 的 GPU 硬件模拟，对特定图形驱动库文件——EGL 库文件或 OpenGL ES 库文件——的查找按照如下的顺序进行：
1. `/vendor/lib64/egl` 或 `/vendor/lib/egl` 目录下文件名符合 `lib%s.so` 模式，即 `/vendor` 下文件名完全匹配的库文件，比如 `/vendor/lib64/egl/libGLES.so`
2. `/system/lib64/egl` 或 `/system/lib/egl` 目录下文件名符合 `lib%s.so` 模式，即 `/system` 下文件名完全匹配的库文件，比如 `/system/lib64/egl/libGLES.so`
3. `/vendor/lib64/egl` 或 `/vendor/lib/egl` 目录下文件名符合 `lib%s_*.so` 模式，即 `/vendor` 下文件名前缀匹配的库文件，比如对于 Pixel 设备的 `/vendor/lib64/egl/libEGL_adreno.so`
4. `/system/lib64/egl` 或 `/system/lib/egl` 目录下文件名符合 `lib%s_*.so` 模式，即 `/system` 下文件名前缀匹配的库文件。

Android 会优先采用 `/vendor/` 下设备供应商提供的图形驱动库，优先采用库文件名完全匹配的图形驱动库。

第二步，通过 `dlopen()` 加载库文件。
```
    const char* const driver_absolute_path = absolutePath.string();

    void* dso = dlopen(driver_absolute_path, RTLD_NOW | RTLD_LOCAL);
    if (dso == 0) {
        const char* err = dlerror();
        ALOGE("load_driver(%s): %s", driver_absolute_path, err?err:"unknown");
        return 0;
    }
```

第三步，初始化函数表。
```
    if (mask & EGL) {
        ALOGD("EGL loaded %s", driver_absolute_path);
        getProcAddress = (getProcAddressType)dlsym(dso, "eglGetProcAddress");

        ALOGE_IF(!getProcAddress,
                "can't find eglGetProcAddress() in %s", driver_absolute_path);

        egl_t* egl = &cnx->egl;
        __eglMustCastToProperFunctionPointerType* curr =
            (__eglMustCastToProperFunctionPointerType*)egl;
        char const * const * api = egl_names;
        while (*api) {
            char const * name = *api;
            __eglMustCastToProperFunctionPointerType f =
                (__eglMustCastToProperFunctionPointerType)dlsym(dso, name);
            if (f == NULL) {
                // couldn't find the entry-point, use eglGetProcAddress()
                f = getProcAddress(name);
                if (f == NULL) {
                    f = (__eglMustCastToProperFunctionPointerType)0;
                }
            }
            *curr++ = f;
            api++;
        }
    }

    if (mask & GLESv1_CM) {
        ALOGD("GLESv1_CM loaded %s", driver_absolute_path);
        init_api(dso, gl_names,
            (__eglMustCastToProperFunctionPointerType*)
                &cnx->hooks[egl_connection_t::GLESv1_INDEX]->gl,
            getProcAddress);
    }

    if (mask & GLESv2) {
      ALOGD("GLESv2 loaded %s", driver_absolute_path);
      init_api(dso, gl_names,
            (__eglMustCastToProperFunctionPointerType*)
                &cnx->hooks[egl_connection_t::GLESv2_INDEX]->gl,
            getProcAddress);
    }

    return dso;
}
```

初始化函数表主要通过 `dlsym()` 根据函数名，一个个找到对应的地址，并赋值给函数指针来完成。EGL 接口函数名来自于 `egl_names`，OpenGL ES 接口函数名来自于 `gl_names`，它们的定义位于 `frameworks/native/opengl/libs/EGL/egl.cpp`：
```
#undef GL_ENTRY
#undef EGL_ENTRY
#define GL_ENTRY(_r, _api, ...) #_api,
#define EGL_ENTRY(_r, _api, ...) #_api,

char const * const gl_names[] = {
    #include "../entries.in"
    NULL
};

char const * const egl_names[] = {
    #include "egl_entries.in"
    NULL
};

#undef GL_ENTRY
#undef EGL_ENTRY
```

这是两个字符串数组，数组项同样来自于另外的两个文件，`gl_names` 的数组项来自于 `egl_entries.in`，`gl_names` 的数组项来自于 `entries.in`，这两个文件也正是前面看到的 `struct egl_t` 结构和 `struct gl_hooks_t` 的 `struct gl_t` 结构的结构成员的来源。Android 通过这种方式建立函数表与函数名表中相同函数的项之间的对应关系，即对于相同的函数，它们对应的项在函数表中的偏移，与在函数名表中的偏移相同。

对于特定的进程，Android 图形驱动初始化的过程只执行一次。在 `egl_init_drivers_locked()` 及 `Loader::load_driver()` 中加日志，重新编译 `frameworks/native/opengl/libs`，替换设备中的 `/system/lib64/libEGL.so` 和 `/system/lib/libEGL.so`，重新启动设备，可以看到如下的日志输出：
```
11-27 13:26:19.440   475   475 D libEGL  : EGL loaded /vendor/lib64/egl/libEGL_adreno.so
11-27 13:26:19.628   475   475 D libEGL  : GLESv1_CM loaded /vendor/lib64/egl/libGLESv1_CM_adreno.so
11-27 13:26:19.648   475   475 D libEGL  : GLESv2 loaded /vendor/lib64/egl/libGLESv2_adreno.so
11-27 13:26:21.761   477   547 D libEGL  : EGL loaded /vendor/lib64/egl/libEGL_adreno.so
11-27 13:26:21.776   477   547 D libEGL  : GLESv1_CM loaded /vendor/lib64/egl/libGLESv1_CM_adreno.so
11-27 13:26:21.785   477   547 D libEGL  : GLESv2 loaded /vendor/lib64/egl/libGLESv2_adreno.so
09-14 09:21:47.715   614   614 D libEGL  : EGL loaded /vendor/lib/egl/libEGL_adreno.so
09-14 09:21:47.792   614   614 D libEGL  : GLESv1_CM loaded /vendor/lib/egl/libGLESv1_CM_adreno.so
09-14 09:21:47.819   614   614 D libEGL  : GLESv2 loaded /vendor/lib/egl/libGLESv2_adreno.so
09-14 09:21:48.492   612   612 D libEGL  : EGL loaded /vendor/lib64/egl/libEGL_adreno.so
09-14 09:21:48.496   613   613 D libEGL  : EGL loaded /vendor/lib/egl/libEGL_adreno.so
09-14 09:21:48.503   612   612 D libEGL  : GLESv1_CM loaded /vendor/lib64/egl/libGLESv1_CM_adreno.so
09-14 09:21:48.510   613   613 D libEGL  : GLESv1_CM loaded /vendor/lib/egl/libGLESv1_CM_adreno.so
09-14 09:21:48.513   612   612 D libEGL  : GLESv2 loaded /vendor/lib64/egl/libGLESv2_adreno.so
09-14 09:21:48.523   613   613 D libEGL  : GLESv2 loaded /vendor/lib/egl/libGLESv2_adreno.so
```

根据这些日志的进程号查找响应的进程：
```
system    475   1     151868 22568 SyS_epoll_ 71808e632c S /system/bin/surfaceflinger
root      612   1     2144132 81252 poll_sched 7627a2044c S zygote64
root      613   1     1578176 69960 poll_sched 00f5045700 S zygote
cameraserver 614   1     123228 45404 binder_thr 00efbf4658 S /system/bin/cameraserver
```

可见 Android 中需要用到 OpenGL ES 图形系统的进程主要有 `surfaceflinger`，`cameraserver` 和所有的 Android Java 应用程序。对于普通的 Java 应用程序，这些驱动库会先由 zygote 加载完成，后续 systemserver 指示 `zygote` 为普通的 Java 应用程序 fork 出新的进程，新的进程继承 `zygote` 加载的那些库。

# 图形驱动 Wrapper
Android 中图形驱动 Wrapper 是图形接口的一个中转。Android 中使用 EGL 和 OpenGL ES 接口的应用程序，可能是通过 framework 提供的 Java 接口，也可能通过 NDK 提供的本地层接口，直接调用图形驱动 Wrapper，这些 Wrapper 再将调用转给设备特有的 EGL 和 OpenGL ES 实现库中的函数。

通过 Android.mk 文件可以看到这样的依赖关系。`frameworks/native/opengl/libs/Android.mk` 文件中存在如下的内容：
```
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= 	       \
	EGL/egl_tls.cpp        \
	EGL/egl_cache.cpp      \
	EGL/egl_display.cpp    \
	EGL/egl_object.cpp     \
	EGL/egl.cpp 	       \
	EGL/eglApi.cpp 	       \
	EGL/getProcAddress.cpp.arm \
	EGL/Loader.cpp 	       \
#

LOCAL_SHARED_LIBRARIES += libbinder libcutils libutils liblog libui
LOCAL_MODULE:= libEGL
LOCAL_LDFLAGS += -Wl,--exclude-libs=ALL
LOCAL_SHARED_LIBRARIES += libdl
. . . . . .
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= 		\
	GLES_CM/gl.cpp.arm 	\
#

LOCAL_CLANG := false
LOCAL_SHARED_LIBRARIES += libcutils liblog libEGL
LOCAL_MODULE:= libGLESv1_CM

LOCAL_SHARED_LIBRARIES += libdl
. . . . . .
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
	GLES2/gl2.cpp   \
#

LOCAL_CLANG := false
LOCAL_ARM_MODE := arm
LOCAL_SHARED_LIBRARIES += libcutils libutils liblog libEGL
LOCAL_MODULE:= libGLESv2

LOCAL_SHARED_LIBRARIES += libdl
. . . . . .
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
	GLES2/gl2.cpp   \
#

LOCAL_CLANG := false
LOCAL_ARM_MODE := arm
LOCAL_SHARED_LIBRARIES += libcutils libutils liblog libEGL
LOCAL_MODULE:= libGLESv3
LOCAL_SHARED_LIBRARIES += libdl
. . . . . .

```
即 `frameworks/native/opengl/libs/` 下的源码被编译为了四个动态链接库，分别为 `libEGL.so`、`libGLESv1_CM`、`libGLESv2` 和 `libGLESv3`，其中 `libGLESv2` 和 `libGLESv3` 完全相同。在`frameworks/base/core/jni/Android.mk` 中可以清晰地看到 framework JNI 对这些库的依赖：
```
LOCAL_SHARED_LIBRARIES := \
. . . . . .
    libsqlite \
    libEGL \
    libGLESv1_CM \
    libGLESv2 \
    libvulkan \
    libETC1 \
. . . . . .
```

EGL 的 Wrapper，如我们前面看到的，位于 `frameworks/native/opengl/libs/EGL`，其 EGL API 实现位于 `frameworks/native/opengl/libs/EGL/eglApi.cpp`。

GLESv1_CM 的 Wrapper 位于 `frameworks/native/opengl/libs/GLES_CM`，其接口实现在 `gl.cpp` 文件中：
```
#elif defined(__arm__)

    #define GET_TLS(reg) "mrc p15, 0, " #reg ", c13, c0, 3 \n"

    #define API_ENTRY(_api) __attribute__((noinline)) _api

    #define CALL_GL_API(_api, ...)                              \
         asm volatile(                                          \
            GET_TLS(r12)                                        \
            "ldr   r12, [r12, %[tls]] \n"                       \
            "cmp   r12, #0            \n"                       \
            "ldrne pc,  [r12, %[api]] \n"                       \
            :                                                   \
            : [tls] "J"(TLS_SLOT_OPENGL_API*4),                 \
              [api] "J"(__builtin_offsetof(gl_hooks_t, gl._api))    \
            : "r12"                                             \
            );

#elif defined(__aarch64__)

    #define API_ENTRY(_api) __attribute__((noinline)) _api

    #define CALL_GL_API(_api, ...)                                  \
        asm volatile(                                               \
            "mrs x16, tpidr_el0\n"                                  \
            "ldr x16, [x16, %[tls]]\n"                              \
            "cbz x16, 1f\n"                                         \
            "ldr x16, [x16, %[api]]\n"                              \
            "br  x16\n"                                             \
            "1:\n"                                                  \
            :                                                       \
            : [tls] "i" (TLS_SLOT_OPENGL_API * sizeof(void*)),      \
              [api] "i" (__builtin_offsetof(gl_hooks_t, gl._api))   \
            : "x16"                                                 \
        );

#elif defined(__i386__)
. . . . . .
#define CALL_GL_API_RETURN(_api, ...) \
    CALL_GL_API(_api, __VA_ARGS__) \
    return 0;


extern "C" {
#pragma GCC diagnostic ignored "-Wunused-parameter"
#include "gl_api.in"
#include "glext_api.in"
#pragma GCC diagnostic warning "-Wunused-parameter"
}

#undef API_ENTRY
#undef CALL_GL_API
#undef CALL_GL_API_RETURN
```

这个文件包含了另外两个文件，`gl_api.in` 和 `glext_api.in`。其中 `gl_api.in` 文件位于 `frameworks/native/opengl/libs/GLES_CM/gl_api.in`，内容如下：
```
void API_ENTRY(glAlphaFunc)(GLenum func, GLfloat ref) {
    CALL_GL_API(glAlphaFunc, func, ref);
}
void API_ENTRY(glClearColor)(GLfloat red, GLfloat green, GLfloat blue, GLfloat alpha) {
    CALL_GL_API(glClearColor, red, green, blue, alpha);
}
void API_ENTRY(glClearDepthf)(GLfloat d) {
    CALL_GL_API(glClearDepthf, d);
}
void API_ENTRY(glClipPlanef)(GLenum p, const GLfloat *eqn) {
    CALL_GL_API(glClipPlanef, p, eqn);
}
void API_ENTRY(glColor4f)(GLfloat red, GLfloat green, GLfloat blue, GLfloat alpha) {
    CALL_GL_API(glColor4f, red, green, blue, alpha);
}
. . . . . .
```

`glext_api.in` 文件位于 `frameworks/native/opengl/libs/GLES_CM/glext_api.in`，内容如下：
```
void API_ENTRY(glEGLImageTargetTexture2DOES)(GLenum target, GLeglImageOES image) {
    CALL_GL_API(glEGLImageTargetTexture2DOES, target, image);
}
void API_ENTRY(glEGLImageTargetRenderbufferStorageOES)(GLenum target, GLeglImageOES image) {
    CALL_GL_API(glEGLImageTargetRenderbufferStorageOES, target, image);
}
void API_ENTRY(glBlendEquationSeparateOES)(GLenum modeRGB, GLenum modeAlpha) {
    CALL_GL_API(glBlendEquationSeparateOES, modeRGB, modeAlpha);
}
void API_ENTRY(glBlendFuncSeparateOES)(GLenum srcRGB, GLenum dstRGB, GLenum srcAlpha, GLenum dstAlpha) {
    CALL_GL_API(glBlendFuncSeparateOES, srcRGB, dstRGB, srcAlpha, dstAlpha);
}
```

配和前面的 `API_ENTRY` 和 `CALL_GL_API` 宏定义，可见 `gl.cpp` 中主要是定义了 OpenGL ES 函数接口，这些函数的实现为，借助于前面初始化的函数表，通过汇编代码，跳转到相应的 OpenGL ES 函数实现。

GLESv2 Wrapper 位于 `frameworks/native/opengl/libs/GLES2`，其 OpenGL ES 接口实现在 `gl2.cpp` 文件中：
```
#elif defined(__arm__)

    #define GET_TLS(reg) "mrc p15, 0, " #reg ", c13, c0, 3 \n"

    #define API_ENTRY(_api) __attribute__((noinline)) _api

    #define CALL_GL_API(_api, ...)                              \
         asm volatile(                                          \
            GET_TLS(r12)                                        \
            "ldr   r12, [r12, %[tls]] \n"                       \
            "cmp   r12, #0            \n"                       \
            "ldrne pc,  [r12, %[api]] \n"                       \
            :                                                   \
            : [tls] "J"(TLS_SLOT_OPENGL_API*4),                 \
              [api] "J"(__builtin_offsetof(gl_hooks_t, gl._api))    \
            : "r12"                                             \
            );

#elif defined(__aarch64__)

    #define API_ENTRY(_api) __attribute__((noinline)) _api

    #define CALL_GL_API(_api, ...)                                  \
        asm volatile(                                               \
            "mrs x16, tpidr_el0\n"                                  \
            "ldr x16, [x16, %[tls]]\n"                              \
            "cbz x16, 1f\n"                                         \
            "ldr x16, [x16, %[api]]\n"                              \
            "br  x16\n"                                             \
            "1:\n"                                                  \
            :                                                       \
            : [tls] "i" (TLS_SLOT_OPENGL_API * sizeof(void*)),      \
              [api] "i" (__builtin_offsetof(gl_hooks_t, gl._api))   \
            : "x16"                                                 \
        );

#elif defined(__i386__)
. . . . . .
#define CALL_GL_API_RETURN(_api, ...) \
    CALL_GL_API(_api, __VA_ARGS__) \
    return 0;



extern "C" {
#pragma GCC diagnostic ignored "-Wunused-parameter"
#include "gl2_api.in"
#include "gl2ext_api.in"
#pragma GCC diagnostic warning "-Wunused-parameter"
}

#undef API_ENTRY
#undef CALL_GL_API
#undef CALL_GL_API_RETURN
```

这里的实现与前面看到的 GLESv1_CM 的 Wrapper 的接口实现类似。

# Android OpenGL ES 图形库结构
Android 的 OpenGL ES 图形系统涉及多个库，根据设备类型的不同，这些库有着不同的结构。

对于模拟器，没有开启 OpenGL ES 的 GPU 硬件模拟的情况，Android OpenGL ES 图形库结构如下：

![](http://upload-images.jianshu.io/upload_images/1315506-31279bc299161d67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当为模拟器开启了 OpenGL ES 的 GPU 硬件模拟，实际的 EGL 和 OpenGL ES 实现库会采用由 `android-7.1.1_r22/device/generic/goldfish-opengl` 下的源码编译出来的几个库文件，即 `libGLESv2_emulation.so`、`libGLESv1_CM_emulation.so` 和 `libEGL_emulation.so`。此时，OpenGL ES 图形库结构如下：


![2017-09-16 11-19-05屏幕截图.png](https://www.wolfcstech.com/images/1315506-2f2b3af4a60e835f.png)


对于真实的物理 Android 设备，OpenGL ES 图形库结构如下：

![](http://upload-images.jianshu.io/upload_images/1315506-5e70ae089b8f27f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# Android OpenGL 图形系统分析系列文章
[在 Android 中使用 OpenGL](https://www.wolfcstech.com/2017/09/13/opengl_on_android_with_sv/)
[Android 图形驱动初始化](https://www.wolfcstech.com/2017/09/14/egl_init_drivers/)
[EGL Context 创建](https://www.wolfcstech.com/2017/09/15/egl_context_creation/)
[Android 图形系统之图形缓冲区分配](https://www.wolfcstech.com/2017/09/20/android_graphics_bufferalloc/)
[Android 图形系统之gralloc](https://www.wolfcstech.com/2017/09/21/android_graphics_gralloc/)