---
title: 在 Android 中使用 OpenGL
date: 2017-09-13 19:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
---

Android 通过 OpenGL 包含了对高性能 2D 和 3D 图形的支持，特别是 OpenGL ES API。OpenGL 是一个跨平台的图形 API，它为 3D 图形处理硬件规定了一个标准的软件接口。OpenGL ES 是一种用于嵌入式设备的 OpenGL 规范。Android 支持多种版本的 OpenGL ES API：
<!--more-->
 * OpenGL ES 1.0 和 1.1 - Android 1.0 及更高版本支持这个 API 规范。
 * OpenGL ES 2.0 - Android 2.0（API level 8）及更高版本支持这个 API 规范。
 * OpenGL ES 3.0 - Android 4.3（API level 18）及更高版本支持这个 API 规范。
 * OpenGL ES 3.1 - Android 5.0（API level 21）及更高版本支持这个 API 规范。

**注意：** 设备上对 OpenGL ES 3.0 API 的支持需要设备生产商提供这个图形管线的实现。运行 Android 4.3 或更高版本的设备 *可能不* 支持 OpenGL ES 3.0 API。在运行时检查支持何种版本的 OpenGL ES 的信息，请参考 [Checking OpenGL ES Version](https://developer.android.com/guide/topics/graphics/opengl.html#version-check)。

**注意：** Android framework 提供的特别的 API 与 J2ME JSR239 OpenGL ES API 类似，但不完全一致。如果你对 J2ME JSR239 比较熟悉，请注意其中的不同。

Android 同时通过它的 framework API 和 NDK 支持 OpenGL。这里主要来看 Android framework 的接口。Android framework 中提供了多个接口让我们可以通过 OpenGL ES 创建并管理图形：GLSurfaceView，TextureView，SurfaceView 等等。如果是要在实际的应用中使用 OpenGL，则 GLSurfaceView 最好用。

为了能够更清晰地厘清，EGL 为 OpenGL 渲染做环境准备的过程，以及 EGL contexts 的管理，这里使用 SurfaceView。示例 OpenGL 应用程序代码（完整代码可以在 [GitHub](https://github.com/hanpfei/OpegGLDemo) 获取）如下：
```
package com.cloudgame.opengldemo;

import javax.microedition.khronos.egl.EGL10;
import javax.microedition.khronos.egl.EGLConfig;
import javax.microedition.khronos.egl.EGLContext;
import javax.microedition.khronos.egl.EGLDisplay;
import javax.microedition.khronos.egl.EGLSurface;
import javax.microedition.khronos.opengles.GL10;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.opengl.GLDebugHelper;
import android.opengl.GLU;
import android.os.Bundle;
import android.util.Log;
import android.view.SurfaceHolder;
import android.view.SurfaceView;

public class BasicGLActivity extends Activity {
    public static final String COLOR_OPTION_EXTRA = "COLORFUL";
    private boolean doColorful = false;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Intent starter = getIntent();
        doColorful = starter.getBooleanExtra(COLOR_OPTION_EXTRA, false);

        mAndroidSurface = new BasicGLSurfaceView(this);

        setContentView(mAndroidSurface);
    }

    private class BasicGLSurfaceView extends SurfaceView implements
            SurfaceHolder.Callback {
        SurfaceHolder mAndroidHolder;

         BasicGLSurfaceView(Context context) {
            super(context);
            mAndroidHolder = getHolder();
            mAndroidHolder.addCallback(this);
            mAndroidHolder.setType(SurfaceHolder.SURFACE_TYPE_GPU);

        }

        public void surfaceChanged(SurfaceHolder holder, int format, int width,
                int height) {
        }

        public void surfaceCreated(SurfaceHolder holder) {
            mGLThread = new BasicGLThread(this);
            
            mGLThread.start();
        }

        public void surfaceDestroyed(SurfaceHolder holder) {
            if (mGLThread != null) {
                mGLThread.requestStop();
            }
        }
    }
    
    BasicGLThread mGLThread;
    private class BasicGLThread extends Thread {
        private static final String DEBUG_TAG = "BasicGLThread";
        SurfaceView sv;
        BasicGLThread(SurfaceView view) {
            sv = view;
        }
        
        private boolean mDone = false;
        public void run() {
            try {
                initEGL();
                initGL();
                
                TriangleSmallGLUT triangle = new TriangleSmallGLUT(3);
                mGL.glMatrixMode(GL10.GL_MODELVIEW);
                mGL.glLoadIdentity();
                GLU.gluLookAt(mGL, 0, 0, 10f, 0, 0, 0, 0, 1, 0f);
                mGL.glColor4f(1f, 0f, 0f, 1f);
                while (!mDone) {
                    mGL.glClear(GL10.GL_COLOR_BUFFER_BIT| GL10.GL_DEPTH_BUFFER_BIT);
                    mGL.glRotatef(1f, 0, 0, 1f);
                    
                    if (doColorful) {
                        triangle.drawColorful(mGL);
                    } else {
                        triangle.draw(mGL);
                    }
                                    
                    mEGL.eglSwapBuffers(mGLDisplay, mGLSurface);
                }
            } catch (Exception e) {
                Log.e(DEBUG_TAG, "GL Failure", e);
            } finally  {
                cleanupGL();
            }
        }
        
        public void requestStop() {
            mDone = true;
            try {
                join();
            } catch (InterruptedException e) {
                Log.e(DEBUG_TAG, "failed to stop gl thread", e);
            }
            
            cleanupGL();
        }
        
        private void cleanupGL() {
            mEGL.eglMakeCurrent(mGLDisplay, EGL10.EGL_NO_SURFACE,
                    EGL10.EGL_NO_SURFACE, EGL10.EGL_NO_CONTEXT);
            mEGL.eglDestroySurface(mGLDisplay, mGLSurface);
            mEGL.eglDestroyContext(mGLDisplay, mGLContext);
            mEGL.eglTerminate(mGLDisplay);

            Log.i(DEBUG_TAG, "GL Cleaned up");
        }
        
        public void initGL( ) {
            int width = sv.getWidth();
            int height = sv.getHeight();
            mGL.glViewport(0, 0, width, height);
            mGL.glMatrixMode(GL10.GL_PROJECTION);
            mGL.glLoadIdentity();
            float aspect = (float) width/height;
            GLU.gluPerspective(mGL, 45.0f, aspect, 1.0f, 30.0f);
             mGL.glClearColor(0.5f,0.5f,0.5f,1);
             
             // the only way to draw primitives with OpenGL ES
             mGL.glEnableClientState(GL10.GL_VERTEX_ARRAY);

            Log.i(DEBUG_TAG, "GL initialized");
        }
        
        public void initEGL() throws Exception {
            mEGL = (EGL10) GLDebugHelper.wrap(EGLContext.getEGL(),
                    GLDebugHelper.CONFIG_CHECK_GL_ERROR
                            | GLDebugHelper.CONFIG_CHECK_THREAD,  null);
            
            if (mEGL == null) {
                throw new Exception("Couldn't get EGL");
            }
            
            mGLDisplay = mEGL.eglGetDisplay(EGL10.EGL_DEFAULT_DISPLAY);
            
            if (mGLDisplay == null) {
                throw new Exception("Couldn't get display for GL");
            }

            int[] curGLVersion = new int[2];
            mEGL.eglInitialize(mGLDisplay, curGLVersion);

            Log.i(DEBUG_TAG, "GL version = " + curGLVersion[0] + "."
                    + curGLVersion[1]);

            EGLConfig[] configs = new EGLConfig[1];
            int[] num_config = new int[1];
            mEGL.eglChooseConfig(mGLDisplay, mConfigSpec, configs, 1,
                    num_config);
            mGLConfig = configs[0];

            mGLSurface = mEGL.eglCreateWindowSurface(mGLDisplay, mGLConfig, sv
                    .getHolder(), null);
            
            if (mGLSurface == null) {
                throw new Exception("Couldn't create new surface");
            }

            mGLContext = mEGL.eglCreateContext(mGLDisplay, mGLConfig,
                    EGL10.EGL_NO_CONTEXT, null);
            
            if (mGLContext == null) {
                throw new Exception("Couldn't create new context");
            }


            if (!mEGL.eglMakeCurrent(mGLDisplay, mGLSurface, mGLSurface, mGLContext)) {
                throw new Exception("Failed to eglMakeCurrent");
            }
                    
            mGL = (GL10) GLDebugHelper.wrap(mGLContext.getGL(),
                    GLDebugHelper.CONFIG_CHECK_GL_ERROR
                            | GLDebugHelper.CONFIG_CHECK_THREAD
                            | GLDebugHelper.CONFIG_LOG_ARGUMENT_NAMES, null);
            
            if (mGL == null) {
                throw new Exception("Failed to get GL");
            }
                
        }
        
        // main OpenGL variables
        GL10 mGL;
        EGL10 mEGL;
        EGLDisplay mGLDisplay;
        EGLConfig mGLConfig;
        EGLSurface mGLSurface;
        EGLContext mGLContext;
        int[] mConfigSpec = { EGL10.EGL_RED_SIZE, 5, 
                EGL10.EGL_GREEN_SIZE, 6, EGL10.EGL_BLUE_SIZE, 5, 
                EGL10.EGL_DEPTH_SIZE, 16, EGL10.EGL_NONE };
    }
    
    @Override
    protected void onResume() {
	super.onResume();
    }

    @Override
    protected void onPause() {
        super.onPause();
    }

    SurfaceView mAndroidSurface;
}
```

`SurfaceView` 的 `SurfaceHolder` 为 OpenGL 的渲染提供画布，即 Surface，它决定图像被实际渲染到什么地方。在上面的代码中，`Activity` 的整个 layout 中就只有一个 `SurfaceView`，它被用于获取 `SurfaceHolder`。

上面的代码中，`SurfaceView` 的 Surface 创建好之后，起了一个单独的线程，这个线程用于处理在该 Surface 上渲染的所有事情。

在能够使用 OpenGL 渲染之前，首先需要为渲染做环境上的准备，即创建 EGL context，并启用它。这通过如下的方法完成：
```
        public void initEGL() throws Exception {
            mEGL = (EGL10) GLDebugHelper.wrap(EGLContext.getEGL(),
                    GLDebugHelper.CONFIG_CHECK_GL_ERROR
                            | GLDebugHelper.CONFIG_CHECK_THREAD,  null);
            
            if (mEGL == null) {
                throw new Exception("Couldn't get EGL");
            }
            
            mGLDisplay = mEGL.eglGetDisplay(EGL10.EGL_DEFAULT_DISPLAY);
            
            if (mGLDisplay == null) {
                throw new Exception("Couldn't get display for GL");
            }

            int[] curGLVersion = new int[2];
            mEGL.eglInitialize(mGLDisplay, curGLVersion);

            Log.i(DEBUG_TAG, "GL version = " + curGLVersion[0] + "."
                    + curGLVersion[1]);

            EGLConfig[] configs = new EGLConfig[1];
            int[] num_config = new int[1];
            mEGL.eglChooseConfig(mGLDisplay, mConfigSpec, configs, 1,
                    num_config);
            mGLConfig = configs[0];

            mGLSurface = mEGL.eglCreateWindowSurface(mGLDisplay, mGLConfig, sv
                    .getHolder(), null);
            
            if (mGLSurface == null) {
                throw new Exception("Couldn't create new surface");
            }

            mGLContext = mEGL.eglCreateContext(mGLDisplay, mGLConfig,
                    EGL10.EGL_NO_CONTEXT, null);
            
            if (mGLContext == null) {
                throw new Exception("Couldn't create new context");
            }


            if (!mEGL.eglMakeCurrent(mGLDisplay, mGLSurface, mGLSurface, mGLContext)) {
                throw new Exception("Failed to eglMakeCurrent");
            }
                    
            mGL = (GL10) GLDebugHelper.wrap(mGLContext.getGL(),
                    GLDebugHelper.CONFIG_CHECK_GL_ERROR
                            | GLDebugHelper.CONFIG_CHECK_THREAD
                            | GLDebugHelper.CONFIG_LOG_ARGUMENT_NAMES, null);
            
            if (mGL == null) {
                throw new Exception("Failed to get GL");
            }
                
        }
        
        // main OpenGL variables
        GL10 mGL;
        EGL10 mEGL;
        EGLDisplay mGLDisplay;
        EGLConfig mGLConfig;
        EGLSurface mGLSurface;
        EGLContext mGLContext;
        int[] mConfigSpec = { EGL10.EGL_RED_SIZE, 5, 
                EGL10.EGL_GREEN_SIZE, 6, EGL10.EGL_BLUE_SIZE, 5, 
                EGL10.EGL_DEPTH_SIZE, 16, EGL10.EGL_NONE };
```

这个过程如下：

1. 获得 `EGLDisplay` 对象。
2. 初始化 `EGLDisplay` 对象。
3. 选择 `EGLConfig`。
4. 基于 `SurfaceHolder` 创建 Windows Surface。
5. 创建 EGL context。
6. 启用前面创建的 EGL context。

随后就可以使用 OpenGL 做渲染了。

每次一个场景渲染完成，都需要通过交换缓冲区来显示渲染结果：
```
                    mEGL.eglSwapBuffers(mGLDisplay, mGLSurface);
```

整个渲染过程结束之后，还需要销毁前面创建的 EGL context：
```
        private void cleanupGL() {
            mEGL.eglMakeCurrent(mGLDisplay, EGL10.EGL_NO_SURFACE,
                    EGL10.EGL_NO_SURFACE, EGL10.EGL_NO_CONTEXT);
            mEGL.eglDestroySurface(mGLDisplay, mGLSurface);
            mEGL.eglDestroyContext(mGLDisplay, mGLContext);
            mEGL.eglTerminate(mGLDisplay);
        }
```

而 framework 的 `GLSurfaceView` 类提供了一些辅助类来管理 EGL contexts，线程间通信，以及与 Activity 生命周期的交互，仅此而已。大多数时候，`GLSurfaceView` 可以简化我们对 OpenGL ES 的使用。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.
