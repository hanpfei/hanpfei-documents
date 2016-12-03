---
title: EventBus设计与实现分析——事件的发布
date: 2016-8-4 11:43:49
tags:
- Android
---

前面在 [EventBus设计与实现分析——特性介绍](https://www.wolfcstech.com/2016/07/03/EventBus%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D/)中介绍了EventBus的基本用法，及其提供的大多数特性的用法；在[EventBus设计与实现分析——订阅者的注册](https://www.wolfcstech.com/2016/07/04/EventBus%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E8%AE%A2%E9%98%85%E8%80%85%E7%9A%84%E6%B3%A8%E5%86%8C/) 中介绍了EventBus中订阅者注册的过程。这里就继续分析EventBus的代码，来了解其事件发布的过程。

<!--more-->

# 事件的发布
如我们前面已经了解到的，在EventBus中，有两种不同类型得事件，一种是普通事件，事件被通知给订阅者之后即被丢弃，另一种是Sticky事件，事件在被通知给订阅者之后会被保存起来，下次有订阅者注册针对这种事件的订阅时，订阅者会直接得到通知。

在EventBus中，会以两个不同的方法来发布这两种不同类型的事件，这两个方法分别是post(Object event)和postSticky(Object event)：
```
    private final Map<Class<?>, Object> stickyEvents;

    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };
......

    /** Posts the given event to the event bus. */
    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
......

    /**
     * Posts the given event to the event bus and holds on to the event (because it is sticky). The most recent sticky
     * event of an event's type is kept in memory for future access by subscribers using {@link Subscribe#sticky()}.
     */
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
......

    /** For ThreadLocal, much faster to set (and get multiple values). */
    final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<Object>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
```
postSticky()仅是在保存了事件之后调用post()来发布事件而已。而在post()中，会借助于PostingThreadState来执行事件发布的过程。PostingThreadState为发布的事件提供了排队功能，同时它还描述一些发布的线程状态。PostingThreadState还是发布过程跟外界交流的一个窗口，外部可通过EventBus类提供的一些方法来控制这个状态，进而影响发布过程，比如取消发布等操作。PostingThreadState对象在ThreadLocal变量中保存，可见发布的事件的队列是每个线程一个的。post()方法会逐个取出事件队列中的每一个事件，调用postSingleEvent()方法来发布。
```
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                Log.d(TAG, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }

    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }

......


    /** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
    private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    addInterfaces(eventTypes, clazz.getInterfaces());
                    clazz = clazz.getSuperclass();
                }
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }

    /** Recurses through super interfaces. */
    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {
                eventTypes.add(interfaceClass);
                addInterfaces(eventTypes, interfaceClass.getInterfaces());
            }
        }
    }
```
postSingleEvent()要发布事件，首先需要找到订阅者，我们前面在 订阅者的注册 中看到，订阅者注册时会在subscriptionsByEventType中保存事件类型和订阅者的映射关系，那要找到订阅者岂不是很容易？

其实不完全是。关键是对于事件类型的处理。要通知的事件类型的订阅者不一定仅仅包含事件对象本身的类型的订阅者，还可能要通知事件类型的父类或实现的接口的类型的订阅者。在eventInheritance被置为true时，就需要通知事件类型的父类或实现的接口的类型的订阅者。lookupAllEventTypes()和addInterfaces()就用于查找所有这样的类型。

postSingleEvent()会逐个事件类型的去通知相应得订阅者，这一任务由postSingleEventForEventType()来完成。而在postSingleEventForEventType()中则是根据subscriptionsByEventType找到所有的订阅者方法，并通过postToSubscription方法来逐个的向这些订阅者方法通知事件。
```
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }

......
    /**
     * Invokes the subscriber if the subscriptions is still active. Skipping subscriptions prevents race conditions
     * between {@link #unregister(Object)} and event delivery. Otherwise the event might be delivered after the
     * subscriber unregistered. This is particularly important for main thread delivery and registrations bound to the
     * live cycle of an Activity or Fragment.
     */
    void invokeSubscriber(PendingPost pendingPost) {
        Object event = pendingPost.event;
        Subscription subscription = pendingPost.subscription;
        PendingPost.releasePendingPost(pendingPost);
        if (subscription.active) {
            invokeSubscriber(subscription, event);
        }
    }

    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
在postToSubscription()中事件的通知又分为同步的通知和异步的通知。同步的通知是直接调用invokeSubscriber(Subscription subscription, Object event)方法，这会将事件对象传递给订阅者方法进行调用。而异步的通知则是将事件及订阅者抛给某个poster就结束。

对于某个订阅者的通知要采用同步通知还是异步通知则需要根据订阅者的ThreadMode及事件发布的线程来定。具体得规则为：
订阅者的线程模式是POSTING  --------------------------------> 同步通知
订阅者的线程模式是MAIN + 事件发布线程是主线程 ---------------> 同步通知
订阅者的线程模式是BACKGROUND + 事件发布线程不是主线程 ------> 同步通知
订阅者的线程模式是BACKGROUND + 事件发布线程是主线程 --------> 异步通知
订阅者的线程模式是MAIN + 事件发布线程不是主线程 --------------> 异步通知
订阅者的线程模式是ASYNC ----------------------------------> 异步通知

同步通知和异步通知各三种。但三种异步通知本身又各不相同，它们分别由三种不同的Poster来处理，订阅者的线程模式是`BACKGROUND` + 事件发布线程是主线程的异步通知由`BackgroundPoster`来处理，订阅者的线程模式是`MAIN` + 事件发布线程不是主线程的异步通知由`HandlerPoster`来处理，而订阅者的线程模式是`ASYNC`的异步通知由`AsyncPoster`来处理。

接着就来看一下这些Poster。首先是HandlerPoster：
```
package org.greenrobot.eventbus;

import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.os.SystemClock;

final class HandlerPoster extends Handler {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}
```
这是一个Handler。其内部有一个PendingPostQueue queue，enqueue()操作即是用描述订阅者方法的Subscription对象和事件对象构造一个PendingPost对象，然后将这个PendingPost对象放入queue中，并在Handler没有在处理事件分发时发送一个消息来唤醒对于事件分发的处理。

而在handleMessage()中，则是逐个从queue中取出PendingPost对象，并通过EventBus的invokeSubscriber(PendingPost pendingPost)来传递事件对象调用订阅者方法。这里调用的invokeSubscriber()方法与前面那个同步版本略有差异，它会将Subscription对象和事件对象从PendingPost对象中提取出来，并调用同步版的方法，同时还会释放PendingPost对象。

这里有一个蛮巧妙得设计，就是那个maxMillisInsideHandleMessage，它用于限制一次事件发布所能消耗的最多的主线程时间。如果事件限制到了的时候订阅者没有通知完，则会发送一个消息，在下一轮中继续处理。

这是一个典型的生产者-消费者模型，生产者是事件的发布者线程，而消费者则是主线程。

PendingPost对象是通过一个链表来组织的。
```

package org.greenrobot.eventbus;

final class PendingPostQueue {
    private PendingPost head;
    private PendingPost tail;

    synchronized void enqueue(PendingPost pendingPost) {
        if (pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        }
        if (tail != null) {
            tail.next = pendingPost;
            tail = pendingPost;
        } else if (head == null) {
            head = tail = pendingPost;
        } else {
            throw new IllegalStateException("Head present, but no tail");
        }
        notifyAll();
    }

    synchronized PendingPost poll() {
        PendingPost pendingPost = head;
        if (head != null) {
            head = head.next;
            if (head == null) {
                tail = null;
            }
        }
        return pendingPost;
    }

    synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
        if (head == null) {
            wait(maxMillisToWait);
        }
        return poll();
    }

}
```
还有PendingPost：
```
package org.greenrobot.eventbus;

import java.util.ArrayList;
import java.util.List;

final class PendingPost {
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();

    Object event;
    Subscription subscription;
    PendingPost next;

    private PendingPost(Object event, Subscription subscription) {
        this.event = event;
        this.subscription = subscription;
    }

    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            if (size > 0) {
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        return new PendingPost(event, subscription);
    }

    static void releasePendingPost(PendingPost pendingPost) {
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            // Don't let the pool grow indefinitely
            if (pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
    }

}
```
PendingPostQueue是一个线程安全的链表，其中链表的节点是PendingPost，它提供了最最基本的入队和出队操作而已。PendingPost再次用了对象池，它提供了获取对象和释放对象的方法。EventBus的作者真的还是蛮喜欢用对象池的嘛。

然后再来看BackgroundPoster：
```
package org.greenrobot.eventbus;

import android.util.Log;

/**
 * Posts events in background.
 * 
 * @author Markus
 */
final class BackgroundPoster implements Runnable {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                executorRunning = true;
                eventBus.getExecutorService().execute(this);
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                while (true) {
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                Log.w("Event", Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }

}
```
BackgroundPoster与HandlerPoster还是挺像的。两者的差别在于BackgroundPoster是一个Runnable，它的enqueue()操作唤醒对于事件分发的处理的方法，是将对象本身放进EventBus的ExecutorService中执行来实现的；另外在处理事件分发的run()方法中，无需像HandlerPoster的handleMessage()方法那样考虑时间限制，它会一次性的将队列中所有的PendingPost处理完才结束。

对于某一个特定事件，一次性的将所有的PendingPost递交给BackgroundPoster，因而大概率的它们会在同一个线程被通知。但如果订阅者对事件的处理过快，在下一个PendingPost还没来得及入队时即执行结束，则还是有可能在不同的线程中被通知。

最后再来看一下AsyncPoster：
```
class AsyncPoster implements Runnable {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }

}
```
它会对每一个通知（订阅者方法 + 订阅者对象 + 事件对象）都起一个不同的task来进行。

用一张图来总结EventBus中事件通知的过程：
![](http://upload-images.jianshu.io/upload_images/1315506-0948fc2f84f92f4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

EventBus发布事件的过程大体如此。
