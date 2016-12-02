---
title: 在android中使用OpenGL
date: 2013-2-26 11:43:49
tags:
- Android
- OpenGL
---

﻿在android中使用OpenGL ES需要三个步骤：

1. 创建GLSurfaceView组件，使用Activity来显示GLSurfaceView组建。

2. 为GLSurfaceView组建创建GLSurfaceView.Renderer实例，实现GLSurfaceView.Renderer类时需要实现该接口里的三个方法：
* abstract void onDrawFrame(GL10 gl)：Called to draw the current frame.
* abstract void onSurfaceChanged(GL10 gl, int width, int height)：Called when the surface changed size.
* abstract void onSurfaceCreated(GL10 gl, EGLConfig config)：Called when the surface is created or recreated.
3. 调用GLSurfaceView组建的setRenderer (GLSurfaceView.Renderer renderer) 方法指定Renderer对象，该对象将会完成GLSurfaceView里3D图形的绘制。

<!--more-->

然后来看一个Demo，首先是主Activity：
```
package com.example.androidgldemo;

import android.opengl.GLSurfaceView;
import android.os.Bundle;
import android.app.Activity;
import android.view.Menu;

public class AndroidGLDemo extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        GLSurfaceView glView = new GLSurfaceView(this);
        AndroidGLDemoRenderer renderer = new AndroidGLDemoRenderer();
        glView.setRenderer(renderer);
        setContentView(glView);
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.activity_main, menu);
        return true;
    }
}
```
然后是Renderer的实现：
```
package com.example.androidgldemo;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.FloatBuffer;
import java.nio.IntBuffer;

import javax.microedition.khronos.egl.EGLConfig;
import javax.microedition.khronos.opengles.GL10;

import android.opengl.GLSurfaceView.Renderer;

public class AndroidGLDemoRenderer implements Renderer {
    float[] mTriangleData = new float[] {
            0.1f, 0.6f, 0.0f,
            -0.3f, 0.0f, 0.0f,
            0.3f, 0.1f, 0.0f
    };
    int[] mTriangleColor = new int[] {
            65535, 0, 0, 0, 
            0, 65535, 0, 0,
            0, 0, 65535, 0,
    };
    
    float[] mRectData = new float[] {
            0.4f, 0.4f, 0.0f,
            0.4f, -0.4f, 0.0f,
            -0.4f, 0.4f, 0.0f, 
            -0.4f, -0.4f, 0.0f
    };
    
    int[] mRectColor = new int[] {
            0, 65535, 0, 0, 
            0, 0, 65535, 0, 
            65535, 0, 0, 0,
            65535, 65535, 0, 0,
    };
    
    FloatBuffer mTriangleDataBuffer;
    IntBuffer mTriangleColorBuffer;
    
    FloatBuffer mRectDataBuffer;
    IntBuffer mRectColorBuffer;
    
    public AndroidGLDemoRenderer() {
        mTriangleDataBuffer= bufferUtil(mTriangleData);
        mTriangleColorBuffer = bufferUtil(mTriangleColor);
        
        mRectDataBuffer = bufferUtil(mRectData);
        mRectColorBuffer = bufferUtil(mRectColor);
    }
    
    @Override
    public void onDrawFrame(GL10 gl) {
        gl.glClear(GL10.GL_COLOR_BUFFER_BIT | GL10.GL_DEPTH_BUFFER_BIT);
        gl.glEnableClientState(GL10.GL_VERTEX_ARRAY);
        gl.glEnableClientState(GL10.GL_COLOR_ARRAY);
        gl.glMatrixMode(GL10.GL_MODELVIEW);
        
        gl.glLoadIdentity();
        gl.glTranslatef(-0.6f, 0.0f, -1.5f);
        gl.glVertexPointer(3, GL10.GL_FLOAT, 0, mTriangleDataBuffer);
        gl.glColorPointer(4, GL10.GL_FIXED, 0, mTriangleColorBuffer);
        gl.glDrawArrays(GL10.GL_TRIANGLES, 0, 3);
        
        gl.glLoadIdentity();
        gl.glTranslatef(0.6f, 0.8f, -1.5f);
        gl.glVertexPointer(3, GL10.GL_FLOAT, 0, mRectDataBuffer);
        gl.glColorPointer(4, GL10.GL_FIXED, 0, mRectColorBuffer);
        gl.glDrawArrays(GL10.GL_TRIANGLE_STRIP, 0, 4);
        
        gl.glFinish();
        gl.glDisableClientState(GL10.GL_VERTEX_ARRAY);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        gl.glViewport(0, 0, width, height);
        gl.glMatrixMode(GL10.GL_PROJECTION);
        gl.glLoadIdentity();
        float ratio = (float) width / height;
        gl.glFrustumf(-ratio, ratio, -1, 1, 1, 10);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        gl.glDisable(GL10.GL_DITHER);
        gl.glHint(GL10.GL_PERSPECTIVE_CORRECTION_HINT, GL10.GL_FASTEST);
        gl.glClearColor(0, 0, 0, 0);
        gl.glShadeModel(GL10.GL_SMOOTH);
        gl.glEnable(GL10.GL_DEPTH_TEST);
        gl.glDepthFunc(GL10.GL_LEQUAL);
    }

    public IntBuffer bufferUtil(int []arr){  
        IntBuffer buffer;

        ByteBuffer qbb = ByteBuffer.allocateDirect(arr.length * 4);
        qbb.order(ByteOrder.nativeOrder());

        buffer = qbb.asIntBuffer();
        buffer.put(arr);
        buffer.position(0);

        return buffer;
   }
    
    public FloatBuffer bufferUtil(float []arr){  
        FloatBuffer buffer;

        ByteBuffer qbb = ByteBuffer.allocateDirect(arr.length * 4);
        qbb.order(ByteOrder.nativeOrder());

        buffer = qbb.asFloatBuffer();
        buffer.put(arr);
        buffer.position(0);

        return buffer;
   }
}
```
注意构造函数中那些Buffer的创建方式。在这个地方，不能直接使用FloatBuffer/IntBuffer 的wrap() method。直接用这个method创建出来的buffer会导致JE:
```
02-26 23:12:08.945: E/OpenGLES(2750): Application com.example.androidgldemo (SDK target 17) called a GL11 Pointer method with an indirect Buffer.
02-26 23:12:08.968: W/dalvikvm(2750): threadid=11: thread exiting with uncaught exception (group=0x40d57930)
02-26 23:12:08.984: E/AndroidRuntime(2750): FATAL EXCEPTION: GLThread 16938
02-26 23:12:08.984: E/AndroidRuntime(2750): java.lang.IllegalArgumentException: Must use a native order direct Buffer
02-26 23:12:08.984: E/AndroidRuntime(2750): 	at com.google.android.gles_jni.GLImpl.glVertexPointerBounds(Native Method)
02-26 23:12:08.984: E/AndroidRuntime(2750): 	at com.google.android.gles_jni.GLImpl.glVertexPointer(GLImpl.java:1122)
02-26 23:12:08.984: E/AndroidRuntime(2750): 	at com.example.androidgldemo.AndroidGLDemoRenderer.onDrawFrame(AndroidGLDemoRenderer.java:63)
02-26 23:12:08.984: E/AndroidRuntime(2750): 	at android.opengl.GLSurfaceView$GLThread.guardedRun(GLSurfaceView.java:1516)
02-26 23:12:08.984: E/AndroidRuntime(2750): 	at android.opengl.GLSurfaceView$GLThread.run(GLSurfaceView.java:1240)
```
