---
title: android的消息队列机制
date: 2013-9-11 11:43:49
categories: Android开发
tags:
- Android开发
---

﻿android下的线程，Looper线程，MessageQueue，Handler，Message等之间的关系，以及Message的send/post及Message dispatch的过程。

<!--more-->

# Looper线程

我们知道，线程是进程中某个单一顺序的控制流，它是内核做CPU调度的单位。那何为Looper线程呢？所谓Looper线程，即是借助于Looper和MessageQueue来管理控制流的一类线程。在android系统中，app的主线程即是借助于Looper和MessageQueue来管理控制流的，因而主线就是一个特殊的Looper线程。其实，不仅仅只有主线程可以用Looper和MessageQueue来管理控制流，其它的线程也一样可以。我们可以先看一下android源代码(Looper类，位置为frameworks/base/core/java/android/os/Looper.java)的注释中给出的一种Looper线程的实现方式：
```
package com.example.messagequeuedemo;

import android.os.Handler;
import android.os.Looper;
import android.util.Log;

public class LooperThread extends Thread {
    public static final String TAG = MainActivity.TAG;
    private static final String CompTAG = "LooperThread";

    public Handler mHandler;

    @Override
    public void run() {
        Log.d(TAG, CompTAG + ": LooperThread=>run");
        Looper.prepare();

        mHandler = new Handler() {
            public void handleMessage(android.os.Message msg) {
                Log.d(TAG, CompTAG + ": LooperThread=>Handler=>handleMessage");
                // process incoming message here
            }
        };

        Looper.loop();
    }
}
```
可以看到，就是在线程的run()方法中，调用Looper.prepare()做一些初始化，然后创建一个Handler对象，最后执行Looper.loop()启动整个的事件循环。就是这么简单的几行代码，一个可以使用消息队列来管理线程执行流程的Looper 线程就创建好了。

那么上面的每一行代码，具体又都做了些什么呢？接着我们就先来看下，神秘的Looper.prepare()到底都干了些什么：
```
     // sThreadLocal.get() will return null unless you've called prepare().
     static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }


    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mRun = true;
        mThread = Thread.currentThread();
    }
```
由这段代码可知，Looper.prepare()是一个静态方法，它做的事情就是简单地为当前的线程创建一个Looper对象，并存储在一个静态的线程局部存储变量中。在Looper的构造函数中又创建了一个MessageQueue对象。同时Looper会引用到当前的线程，并将一个表示执行状态的变量mRun设置为true。对于此处的线程局部存储变量sThreadLocal，可以简单地理解为一个HashMap，该HashMap中存放的数据其类型为Looper，key则为Thread，而每个线程又都只能获得特定于这个线程的key，从而访问到专属于这个线程的数据。

启动Looper线程就和启动普通的线程一样，比如：
```
public class MainActivity extends Activity {
    public static final String TAG = "MessageQueueDemo";
    private static final String CompTAG = "MainActivity";
    private LooperThread mLooperThread;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        Log.d(TAG, CompTAG + ": MainActivity=>onCreate");
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mLooperThread = new LooperThread();
        mLooperThread.start();
    }
```
同样是new一个对象，然后调用该对象的start()方法。

Android SDK本身也提供了一个class来实现LooperThread那样的机制，以方便在app中创建一个执行事件处理循环的后台线程，但提供的功能要完善得多，这个class就是HandlerThread。我们可以看一下这个class的定义：
```
package android.os;
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from 
     * {@link android.os.Process} and not from java.lang.Thread.
     */
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
    /**
     * Quits the handler thread's looper.
     * <p>
     * Causes the handler thread's looper to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     *
     * @see #quitSafely
     */
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    /**
     * Quits the handler thread's looper safely.
     * <p>
     * Causes the handler thread's looper to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * Pending delayed messages with due times in the future will not be delivered.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p>
     * If the thread has not been started or has finished (that is if
     * {@link #getLooper} returns null), then false is returned.
     * Otherwise the looper is asked to quit and true is returned.
     * </p>
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     */
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
    /**
     * Returns the identifier of this thread. See Process.myTid().
     */
    public int getThreadId() {
        return mTid;
    }
}
```
它不仅有事件处理循环，还给app提供了灵活地设置HandlerThread的线程优先级的方法；事件循环实际启动前的回调(onLooperPrepared())；获取HandlerThread的Looper的方法(这个方法对于Handler比较重要)；以及多种停掉HandlerThread的方法——quit()和quitSafely()——以避免资源的leak。

Looper线程有两种，一种是我们上面看到的那种由app自己创建，并在后台运行的类型；另外一种则是android Java应用的主线程。创建前者的Looper对象需要使用Looper.prepare()方法，而创建后者的，则需使用Looper.prepareMainLooper()方法。我们可以看一下Looper.prepareMainLooper()的实现：
```
    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
比较特别的地方即在于，此处调用prepare()方法传进去的quitAllowed参数为false，即表示这个Looper不能够被quit掉。其他倒是基本一样。整个android系统中，调用到prepareMainLooper()方法的地方有两个：
```
/frameworks/base/services/java/com/android/server/
H A D	SystemServer.java	94 Looper.prepareMainLooper();
/frameworks/base/core/java/android/app/
H A D	ActivityThread.java	5087 Looper.prepareMainLooper();
```
一处在SystemServer——system_server的主线程——的run()方法中，用于为system_server主线程初始化消息处理队列；另外一处在ActivityThread的run()方法中，自然即是创建android app主线程的消息处理队列了。

# 通过消息与Looper线程交互

那Looper线程的特别之处究竟在哪里呢？如前所述，这种线程有一个Looper与之关联，会使用消息队列，或者称为事件循环来管理执行的流程。那这种特别之处又如何体现呢？答案即是其它线程可以向此类线程中丢消息进来（当然此类线程本身也可以往自己的消息队列里面丢消息），然后在事件处理循环中，这些事件会得到处理。那究竟要如何往Looper线程的消息队列中发送消息呢？

回忆前面我们创建Looper线程的那段代码，不是有创建一个Handler出来嘛。没错，就是通过Handler来向Looper线程的MessageQueue中发送消息的。可以看一下使用Handler向Looper线程发送消息的方法。LooperThread的代码与上面的一样，向Looper线程发送消息的部分的写法：
```
package com.intel.helloworld;

import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.app.Activity;
import android.util.Log;
import android.view.Menu;

public class MainActivity extends Activity {
    public static final String TAG = "LifecycleDemoApp";
    private static final String CompTAG = "MainActivity";
    
    public static final int MESSAGE_WHAT_CREATE = 1;
    public static final int MESSAGE_WHAT_START = 2;
    public static final int MESSAGE_WHAT_RESUME = 3;
    public static final int MESSAGE_WHAT_PAUSE = 4;
    public static final int MESSAGE_WHAT_STOP = 5;
    public static final int MESSAGE_WHAT_DESTROY = 6;

    LooperThread mThread;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        Log.d(TAG, CompTAG + ": " + "Activity-->onCreate");
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mThread = new LooperThread();
        mThread.start();
    }

    @Override
    protected void onStart() {
        Log.d(TAG, CompTAG + ": " + "Activity-->onStart");
        super.onStart();

        Handler handler = mThread.mHandler;
        Message msg = Message.obtain();
        msg.what = MESSAGE_WHAT_START;
        handler.sendMessage(msg);
    }
    
    @Override
    protected void onResume() {
        Log.d(TAG, CompTAG + ": " + "Activity-->onResume");
        super.onResume();

        Handler handler = mThread.mHandler;
        Message msg = Message.obtain();
        msg.what = MESSAGE_WHAT_RESUME;
        handler.sendMessage(msg);
    }
    
    @Override
    protected void onPause() {
        Log.d(TAG, CompTAG + ": " + "Activity-->onPause");
        super.onPause();

        Handler handler = mThread.mHandler;
        Message msg = Message.obtain();
        msg.what = MESSAGE_WHAT_PAUSE;
        handler.sendMessage(msg);
    }
    
    @Override
    protected void onStop() {
        Log.d(TAG, CompTAG + ": " + "Activity-->onStop");
        super.onStop();

        Handler handler = mThread.mHandler;
        Message msg = Message.obtain();
        msg.what = MESSAGE_WHAT_STOP;
        handler.sendMessage(msg);
    }

    @Override
    protected void onDestroy() {
        Log.d(TAG, CompTAG + ": " + "Activity-->onDestroy");
        super.onDestroy();

        Handler handler = mThread.mHandler;
        Message msg = Message.obtain();
        msg.what = MESSAGE_WHAT_DESTROY;
        handler.sendMessage(msg);
    }

    @Override public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.main, menu);
        return true;
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        Log.d(TAG, CompTAG + ": " + "Activity-->onSaveInstanceState");
        super.onSaveInstanceState(outState);
    }
}
```
使用Handler向一个Looper线程发送消息的过程，基本上即是，调用Message.obtain()或Handler.obtainMessage()获取一个Message对象->设置Message对象->调用 ***在Looper线程中创建的Handler对象的方法发送消息***。

Handler究竟是如何知道要向哪个MessageQueue发送消息的呢？从前面的代码中，我们似乎看不到任何Handler与MessageQueue能关联起来的迹象。这究竟是怎么回事呢？这也是我们特别强调 ***要使用Looper线程中创建的Handler对象*** 来向该Looper线程中发送消息的原因。我们可以看一下Handler对象构造的过程：
```
    /**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }

    /**
     * Constructor associates this handler with the {@link Looper} for the
     * current thread and takes a callback interface in which you can handle
     * messages.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     *
     * @param callback The callback interface in which to handle messages, or null.
     */
    public Handler(Callback callback) {
        this(callback, false);
    }

    /**
     * Use the provided {@link Looper} instead of the default one.
     *
     * @param looper The looper, must not be null.
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }

    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     */
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }

    /**
     * Use the {@link Looper} for the current thread
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with represent to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier long)}.
     *
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(boolean async) {
        this(null, async);
    }

    /**
     * Use the {@link Looper} for the current thread with the specified callback interface
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with represent to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier long)}.
     *
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    
    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.  Also set whether the handler
     * should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
Handler的构造函数共有7个，其中4个不需要传递Looper参数，3个需要。前面我们用的是不需要传递Looper对象的构造函数，因而现在我们主要关注那4个不需要传递Looper参数的。它们都是Handler(Callback callback, boolean async)不同形式的封装，而在这个构造函数中是通过Looper.myLooper()获取到当前线程的Looper对象，并与相关的MessageQueue关联起来的。这也是前面我们在实现Looper线程时，要在其run方法中创建一个public Handler的依据。

当然我们也可以使用那些能够手动传递Looper对象的构造函数，在构造Handler对象时，显式地使其与特定的Looper/消息队列关联起来。比如配合HandlerThread.getLooper()方法来用。

（这个地方我们看到， ***创建Handler对象时，可以传递另外两个参数，一个是Callback，另一个是async，那这两个参数在这套消息队列机制中，又起到一个什么样的作用呢？*** 后面我们在来解答这个问题。）

Handler提供了两组函数用于向一个Looper线程的MessageQueue中发送消息，分别是postXXX()族和sendXXX()族，这两组函数的调用关系大体如下所示：

![此处输入图片的描述](../images/085042_SlZe_919237.jpg)

我们可以先看一下sendXXX()族消息发送方法：
```
    /**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in {@link #handleMessage},
     * in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    /**
     * Sends a Message containing only the what value.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

    /**
     * Sends a Message containing only the what value, to be delivered
     * after the specified amount of time elapses.
     * @see #sendMessageDelayed(android.os.Message, long) 
     * 
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    /**
     * Sends a Message containing only the what value, to be delivered 
     * at a specific time.
     * @see #sendMessageAtTime(android.os.Message, long)
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */

    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    /**
     * Enqueue a message into the message queue after all pending messages
     * before (current time + delayMillis). You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    /**
     * Enqueue a message at the front of the message queue, to be processed on
     * the next iteration of the message loop.  You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     * <b>This method is only for use in very special circumstances -- it
     * can easily starve the message queue, cause ordering problems, or have
     * other unexpected side-effects.</b>
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
这些方法之间大多只有一些细微的差别，它们最终都调用Handler.enqueueMessage()/MessageQueue.enqueueMessage()方法。（**哈哈，这个地方，被我们逮到一个Handler类某贡献者所犯的copy-paste错误：仔细看一下sendMessageAtTime()和sendMessageAtFrontOfQueue()这两个方法，创建RuntimeException的部分，消息中都是"sendMessageAtTime()"。** ）Handler实际的职责，并不完全如它的名称所示，在处理message外，它还要负责发送Message到MessageQueue。

注意，在Handler.enqueueMessage()中，会将 ***Message的target设为this，而且是不加检查，强制覆盖原来的值*** 。后面我们将会看到，Message的target就是Message被dispatched到的Handler，也是将会处理这条Message的Handler。这个地方强制覆盖的逻辑表明，我们用哪个Handler来发送一条消息，那么这条消息就将会被dispatched给哪个Handler来处理，消息的发送者即接收者。这使得 ***Handler那些obtainMessage()方法，及Message需要Handler参数的那些obtain()方法显得非常多余。通过这些方法会在获取Message时设置它的target，但这些设置又总是会在后面被无条件地覆盖掉*** 。

再来看一下MessageQueue.enqueueMessage()方法：
```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        boolean needWake;
        synchronized (this) {
            if (mQuiting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
        }
        if (needWake) {
            nativeWake(mPtr);
        }
        return true;
    }
```
此处我们看到，MessageQueue用一个单向链表来保存所有的Messages，在链表中各个Message按照其请求执行的时间先后来排序。将Message插入MessageQueue的算法也还算清晰简洁，不必赘述。但此处， ***MessageQueue/Message的author为什么要自己实现一个单向链表，而没有用Java标准库提供的容器组件呢？是为了插入一条新的Message方便，还是仅仅为了练习怎么用Java实现单向链表？wake的逻辑后面我们会再来研究*** 。

向MessageQueue中发送消息的postXXX()方法：
```
    /**
     * Causes the Runnable r to be added to the message queue.
     * The runnable will be run on the thread to which this handler is
     * attached.
     *
     * @param r The Runnable that will be executed.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * at a specific time given by <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * The runnable will be run on the thread to which this handler is attached.
     *
     * @param r The Runnable that will be executed.
     * @param uptimeMillis The absolute time at which the callback should run,
     *         using the {@link android.os.SystemClock#uptimeMillis} time-base.
     *  
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean postAtTime(Runnable r, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }

    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * at a specific time given by <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * The runnable will be run on the thread to which this handler is attached.
     *
     * @param r The Runnable that will be executed.
     * @param uptimeMillis The absolute time at which the callback should run,
     *         using the {@link android.os.SystemClock#uptimeMillis} time-base.
     * 
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     *         
     * @see android.os.SystemClock#uptimeMillis
     */
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }

    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * after the specified amount of time elapses.
     * The runnable will be run on the thread to which this handler
     * is attached.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     *  
     * @param r The Runnable that will be executed.
     * @param delayMillis The delay (in milliseconds) until the Runnable
     *        will be executed.
     *        
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed --
     *         if the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean postDelayed(Runnable r, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }

    /**
     * Posts a message to an object that implements Runnable.
     * Causes the Runnable r to executed on the next iteration through the
     * message queue. The runnable will be run on the thread to which this
     * handler is attached.
     * <b>This method is only for use in very special circumstances -- it
     * can easily starve the message queue, cause ordering problems, or have
     * other unexpected side-effects.</b>
     *  
     * @param r The Runnable that will be executed.
     * 
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean postAtFrontOfQueue(Runnable r)
    {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }
```
这组方法相对于前面的sendXXX()族方法而言，其特殊之处在于，它们都需要传入一个Runnable参数，post的消息，其特殊之处也正在于，Message的callback将是传入的Runnable对象。这些特别的地方将影响这些消息在dispatch时的行为。

# 消息队列中消息的处理

消息队列中的消息是在Looper.loop()中被取出处理的：
```
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
           }

            msg.recycle();
        }
    }
```
在取出的msg为NULL之前，消息处理循环都一直运行。为NULL的msg表明消息队列被停掉了。在这个循环中，被取出的消息会被dispatch给一个Handler处理，即是msg.target，执行Handler.dispatchMessage()方法。

注意：Looper.loop()循环体的末尾，调用了从消息队列中取出且已经被dispatched处理的Message的recycle()方法，这表明Looper会帮我们自动回收发送到它的消息队列的消息。在我们将一条消息发送给Looper线程的消息队列之后，我们就不需要再担心消息的回收问题了，Looper自会帮我们很好的处理

接着来看Handler.dispatchMessage()：
```
    /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     *
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }

    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }

    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }
```
可以认为有三种对象可能会实际处理一条消息，分别是消息的Runnable callback，Handler的Callback mCallback和Handler对象本身。但这三种对象在获取消息的处理权方面有一定的优先级，消息的Runnable callback优先级最高，在它不为空时，只执行这个callback；优先级次高的是Handler的Callback mCallback，在它不为空时，Handler会先把消息丢给它处理，如果它不处理返回了false，Handler才会调用自己的handleMessage()来处理。

通常，Handler的handleMessage()方法通常需要override，来实现消息处理的主要逻辑。Handler的mCallback，使得开发者可以比较方便的将消息处理的逻辑和发送消息的Handler完全分开。

此处也可见post消息的特殊之处，此类消息将完全绕过Handler中用于处理消息的handleMessage() 方法，而只会执行消息的sender所实现的Runnable。

# Sleep-Wakeup机制

还有一个问题，当MessageQueue中没有Messages时，Looper线程会做什么呢？它会不停地轮询，并检查消息队列中是否有消息吗？计算机科学发展到现在，闭上眼睛我们都能猜到，Looper线程一定不会去轮询的。Looper线程也确实没有去轮询消息队列。在消息队列为空时，Looper线程会去休眠，然后在消息队列中有了消息之后，再被唤醒。但这样的机制又是如何实现的呢？

## Sleep-Wakeup机制所需设施的建立

我们从Sleep-Wakeup机制所需设施的建立开始。回忆前面的Looper构造函数，它会创建一个MessageQueue对象，而Sleep-Wakeup机制所需设施正是在MessageQueue对象的创建过程中创建出来的。（在android消息队列机制中，消息取出和压入的主要逻辑都在MessageQueue中完成，MessageQueue实现一个定制的阻塞队列，将等待-唤醒的逻辑都放在这个类里想必也没什么让人吃惊的地方吧。）我们来看MessageQueue的构造函数：
```
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
    
    private native static long nativeInit();
```
这个方法调用nativeInit()方法来创建Sleep-Wakeup机制所需设施。来看nativeInit()的实现(在frameworks/base/core/jni/android_os_MessageQueue.cpp)：
```
NativeMessageQueue::NativeMessageQueue() : mInCallback(false), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}

static jint android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jint>(nativeMessageQueue);
}

static JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()I", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "(I)V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(II)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(I)V", (void*)android_os_MessageQueue_nativeWake }
};
```
可以看到，nativeInit()所做的事情，就是创建一个NativeMessageQueue对象，在NativeMessageQueue的构造函数中，会来创建一个Looper对象。与Java层的Looper对象类似，native层的这种Looper对象也是保存在线程局部存储变量中的，每个线程一个。接着我们来看Looper类的构造函数和Looper::getForThread()函数，来了解一下，native层的线程局部存储API的用法(Looper类的实现在frameworks/native/libs/utils/Looper.cpp)：
```
// Hint for number of file descriptors to be associated with the epoll instance.
static const int EPOLL_SIZE_HINT = 8;

// Maximum number of file descriptors for which to retrieve poll events each iteration.
static const int EPOLL_MAX_EVENTS = 16;

static pthread_once_t gTLSOnce = PTHREAD_ONCE_INIT;
static pthread_key_t gTLSKey = 0;

Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    int result = pipe(wakeFds);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];

    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);

    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);

    // Allocate the epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);
}

void Looper::initTLSKey() {
    int result = pthread_key_create(& gTLSKey, threadDestructor);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not allocate TLS key.");
}

void Looper::threadDestructor(void *st) {
    Looper* const self = static_cast<Looper*>(st);
    if (self != NULL) {
        self->decStrong((void*)threadDestructor);
    }
}

void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS

    if (looper != NULL) {
        looper->incStrong((void*)threadDestructor);
    }

    pthread_setspecific(gTLSKey, looper.get());

    if (old != NULL) {
        old->decStrong((void*)threadDestructor);
    }
}

sp<Looper> Looper::getForThread() {
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");

    return (Looper*)pthread_getspecific(gTLSKey);
}
```
关于pthread库提供的线程局部存储API的用法，可以看到，每个线程局部存储对象，都需要一个key，通过pthread_key_create()函数创建，随后各个线程就可以通过这个key并借助于pthread_setspecific()和pthread_getspecific()函数来保存或者获取相应的线程局部存储的变量了。再来看Looper的构造函数。它创建了一个pipe，两个文件描述符。然后设置管道的两个文件描述属性为非阻塞I/O。接着是创建并设置epoll实例。由此我们了解到，android的消息队列是通过epoll机制来实现其Sleep-Wakeup机制的。

## 唤醒

然后来看当其他线程向Looper线程的MessageQueue中插入了消息时，Looper线程是如何被叫醒的。回忆我们前面看到的MessageQueue类的enqueueMessage()方法，它在最后插入消息之后，有调用一个nativeWake()方法。没错，正是这个nativeWake()方法执行了叫醒Looper线程的动作。那它又是如何叫醒Looper线程的呢？来看它的实现：
```
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jint ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->wake();
}

// ----------------------------------------------------------------------------

static JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()I", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "(I)V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(II)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(I)V", (void*)android_os_MessageQueue_nativeWake }
};
```
它只是调用了native层的Looper对象的wake()函数。接着再来看native Looper的wake()函数：
```
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    ssize_t nWrite;
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);

    if (nWrite != 1) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```
它所做的事情，就是向管道的用于写的那个文件中写入一个“W”字符。

## 休眠

Looper线程休眠的过程。我们知道，Looper线程在Looper.loop()方法中，不断地从MessageQueue中取出消息，然后处理，如此循环往复，永不止息。不难想象，休眠的时机应该是在取出消息的时候。Looper.loop()通过MessageQueue.next()从消息队列中取出消息。来看MessageQueue.next()方法：
```
    Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;

        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(mPtr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuiting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
值得注意的是上面那个对于nativePollOnce()的调用。wait机制的实现正在于此。来看这个方法的实现，在native的JNI code里面：
```
class MessageQueue : public RefBase {
public:
    /* Gets the message queue's looper. */
    inline sp<Looper> getLooper() const {
        return mLooper;
    }

    /* Checks whether the JNI environment has a pending exception.
     *
     * If an exception occurred, logs it together with the specified message,
     * and calls raiseException() to ensure the exception will be raised when
     * the callback returns, clears the pending exception from the environment,
     * then returns true.
     *
     * If no exception occurred, returns false.
     */
    bool raiseAndClearException(JNIEnv* env, const char* msg);

    /* Raises an exception from within a callback function.
     * The exception will be rethrown when control returns to the message queue which
     * will typically cause the application to crash.
     *
     * This message can only be called from within a callback function.  If it is called
     * at any other time, the process will simply be killed.
     *
     * Does nothing if exception is NULL.
     *
     * (This method does not take ownership of the exception object reference.
     * The caller is responsible for releasing its reference when it is done.)
     */
    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj) = 0;

protected:
    MessageQueue();
    virtual ~MessageQueue();

protected:
    sp<Looper> mLooper;
};

class NativeMessageQueue : public MessageQueue {
public:
    NativeMessageQueue();
    virtual ~NativeMessageQueue();

    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj);

    void pollOnce(JNIEnv* env, int timeoutMillis);

    void wake();

private:
    bool mInCallback;
    jthrowable mExceptionObj;
};

void NativeMessageQueue::pollOnce(JNIEnv* env, int timeoutMillis) {
    mInCallback = true;
    mLooper->pollOnce(timeoutMillis);
    mInCallback = false;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}

static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jclass clazz,
        jint ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, timeoutMillis);
}
```
继续追Looper::pollOnce()的实现(在frameworks/native/libs/utils/Looper.cpp)：
```
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif

    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %lldns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }

    // Poll.
    int result = ALOOPER_POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // Acquire lock.
    mLock.lock();

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = ALOOPER_POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = ALOOPER_POLL_TIMEOUT;
        goto Done;
    }

    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif

    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= ALOOPER_EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= ALOOPER_EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= ALOOPER_EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= ALOOPER_EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = ALOOPER_POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == ALOOPER_POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd);
            }
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = ALOOPER_POLL_CALLBACK;
        }
    }
    return result;
}
```
它通过调用epoll_wait()函数来等待消息的到来。

# HandlerThread的退出

HandlerThread或使用Looper/MessageQueue自定义的类似东西，必须在不需要时被停掉。如前所见，每次创建MessageQueue，都会占用好几个描述符，但在Linux/Android上，一个进程所能打开的文件描述符的最大个数是有限制的，大多为1024个。如果一个app打开的文件描述符达到了这个上限，则将会出现许多各式各样的古怪问题，像进程间通信，socket，输入系统等很多机制，都会依赖于类文件的东西。同时，HandlerThread也总会占用一个线程。这些资源都是只有在显式地停掉之后才会被释放的。

我们来看一下HandlerThread的退出机制。先来看一个使用了HandlerThread，并适时地退出的例子，代码在frameworks/base/core/java/android/app/IntentService.java：
```
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }
```
在这个例子中，Service的onCreate()方法创建了HandlerThread，保存了HandlerThread的Looper，然后在Service的onDestroy()方法中停掉了Looper，实际上也即是退出了HandlerThread。

来看一下Looper停止方法具体的实现：
```
    /**
     * Quits the looper.
     * <p>
     * Causes the {@link #loop} method to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @see #quitSafely
     */
    public void quit() {
        mQueue.quit(false);
    }

    /**
     * Quits the looper safely.
     * <p>
     * Causes the {@link #loop} method to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * However pending delayed messages with due times in the future will not be
     * delivered before the loop terminates.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p>
     */
    public void quitSafely() {
        mQueue.quit(true);
    }
```
Looper有提供两个退出方法，quit()和quitSafely()，这两个方法都会阻止再向消息队列中发送消息。但它们的主要区别在于，前者不会再处理消息队列中还没有被处理的所有消息，而后者则会在处理完那些已经到期的消息之后才真的退出。

由前面我们对Looper.loop()方法的分析，也不难理解，MessageQueue退出即Looper退出。此处也是直接调用了MessageQueue.quit()方法。那我们就来看一下MessageQueue的quit()方法：
```
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new RuntimeException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```
主要做了几个事情：

 * 第一，设置标记mQuitting为true，以表明HandlerThread要退出了；
 * 第二，根据传入的参数safe，删除队列里面适当类型的消息；
 * 第三，调用nativeWake(mPtr)，将HandlerThread线程唤醒。

我们都知道，Java里面是没有办法直接终止另外一个线程的，同时强制中止一个线程也是很不安全的，所以，这里也是只设置一个标记，然后在MessageQueue.next()中获取消息时检查此标记，以在适当的时候退出。MessageQueue.next()里中止消息处理循环的，主要是下面这几行：
 ```
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
```
然后在 MessageQueue.dispose()方法中完成最终的退出，及资源清理的工作：
```
    private void dispose() {
        if (mPtr != 0) {
            nativeDestroy(mPtr);
            mPtr = 0;
        }
    }
```
android中消息队列机制，大体如此。

Done。
