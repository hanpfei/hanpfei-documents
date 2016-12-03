---
title: EventBus设计与实现分析——订阅者的注册
date: 2016-7-4 11:43:49
tags:
- Android
---

前面在 [EventBus设计与实现分析——特性介绍](https://www.wolfcstech.com/2016/07/03/EventBus%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D/) 一文中介绍了EventBus的基本用法，及其提供的大多数特性的用法，这让我们对EventBus为用户提供的主要功能有了大体的了解，为我们后续理解EventBus的设计决策提供了良好的基础。这里我们就开始深入到EventBus的实现细节，先来了解其中的订阅者注册的过程。

<!--more-->

# 订阅者的注册
EventBus使用了一种类似于闭包的机制来描述订阅者。即对于EventBus而言，一个订阅者由`一个方法`＋`该方法绑定的上下文（也就是该方法绑定的对象`）＋`订阅者感兴趣的事件的类型`来描述。具体到实现，即是通过`SubscriberMethod`＋`SubscriberMethod`这样几个类来描述。

我们可以看一下这几个类的定义，首先是`Subscription`类：
```
package org.greenrobot.eventbus;

final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

    Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }

    @Override
    public boolean equals(Object other) {
        if (other instanceof Subscription) {
            Subscription otherSubscription = (Subscription) other;
            return subscriber == otherSubscription.subscriber
                    && subscriberMethod.equals(otherSubscription.subscriberMethod);
        } else {
            return false;
        }
    }

    @Override
    public int hashCode() {
        return subscriber.hashCode() + subscriberMethod.methodString.hashCode();
    }
}
```
`subscriberMethod`即是我们前面提到的方法，而`subscriber`则是上下文object。

然后是`SubscriberMethod`类：
```
package org.greenrobot.eventbus;

import java.lang.reflect.Method;

/** Used internally by EventBus and generated subscriber indexes. */
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }

    @Override
    public boolean equals(Object other) {
        if (other == this) {
            return true;
        } else if (other instanceof SubscriberMethod) {
            checkMethodString();
            SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
            otherSubscriberMethod.checkMethodString();
            // Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }

    private synchronized void checkMethodString() {
        if (methodString == null) {
            // Method.toString has more overhead, just take relevant parts of the method
            StringBuilder builder = new StringBuilder(64);
            builder.append(method.getDeclaringClass().getName());
            builder.append('#').append(method.getName());
            builder.append('(').append(eventType.getName());
            methodString = builder.toString();
        }
    }

    @Override
    public int hashCode() {
        return method.hashCode();
    }
}
```
订阅者的注册过程，即是构造`SubscriberMethod`，继而构造`Subscription`，并对这些对象进行存储管理的过程。我们可以通过`EventBus.register()`方法的定义具体看一下在EventBus中是怎么做：
```
    /**
     * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
     * are no longer interested in receiving events.
     * <p/>
     * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
     * The {@link Subscribe} annotation also allows configuration like {@link
     * ThreadMode} and priority.
     */
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
    
    // Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```
通过`EventBus.register()`和`EventBus.subscribe()`可以看到，register的过程主要做了这样一些事情：
1. 查找订阅者对象的所有订阅者方法。方法隶属于类，不以对象的改变而改变，应用程序的一次生命周期中，这些方法总是不变的。
2. 构造`Subscription`来描述每一个订阅者，也就是`订阅者对象`＋`一个订阅者方法`。
3. 建立事件类型与订阅者之间的映射关系，这主要通过`subscriptionsByEventType`数据结构来描述。对于每一个事件类型，其订阅者会依优先级而排序。不难看出，这个结构主要用于事件的发布，在事件发布时，可以通过这个结构快速地找到每一个订阅者。
这个地方查找正确的插入位置是用的顺序查找的方法，估计是考虑到对于一个事件类型，订阅者一般不会太多，而订阅者的优先级一般不会很分散的缘故。
4. 建立订阅者对象与该对象监听的事件类型之间的映射关系，这主要由`typesBySubscriber`数据结构来描述。这个结构则主要用于注销一个对象对事件的监听。注销一个对象对事件得监听时，可以通过这个结构快速地将监听者对象得信息从EventBus中整个的移除出去。
5. 处理对Sticky事件的监听。如我们前面所见，这种类型的事件在发布时会通知到每个监听者，同时还会被保存起来，当后面再次注册对这种事件的监听时，监听者会直接得到通知。

`EventBus`使用`SubscriberMethodFinder`来查找一个类中所有的监听者方法，主要即是`findSubscriberMethods()`方法。

如我们前面所见，要使一个类的某个方法成为监听者方法，需要给该方法添加**Subscribe** annotation，在`SubscriberMethodFinder.findSubscriberMethods()`中会查找加了这个annotation的方法。但老版本的EventBus并不是通过annotation来标识一个监听者方法的，而是通过命名模式等方法，因而这里可以看到一种更为灵活的设计，来给用户提供空间，以支持不同的标识监听者方法的方法：
```
class SubscriberMethodFinder {
    /*
     * In newer class files, compilers may add methods. Those are called bridge or synthetic methods.
     * EventBus must ignore both. There modifiers are not public but defined in the Java class file format:
     * http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1
     */
    private static final int BRIDGE = 0x40;
    private static final int SYNTHETIC = 0x1000;

    private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

    private List<SubscriberInfoIndex> subscriberInfoIndexes;
    private final boolean strictMethodVerification;
    private final boolean ignoreGeneratedIndex;

    private static final int POOL_SIZE = 4;
    private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

    SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {
        this.subscriberInfoIndexes = subscriberInfoIndexes;
        this.strictMethodVerification = strictMethodVerification;
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    }

    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
由于在应用程序的整个生命周期中，一个类的监听者方法总是不会变的，因而可以看到在`SubscriberMethodFinder`中有创建一个缓存*METHOD_CACHE*，以监听者类为key，以监听者方法的列表为值。在`SubscriberMethodFinder.findSubscriberMethods()`方法中会首先检查这个缓存中是否已经有了对应类的监听者方法，如果有的话会直接返回给调用者，没有的时候才会通过反射等机制来查找。

`ignoreGeneratedIndex`用来标记是否要忽略用户定制的监听者方法标识方法。若忽略则直接利用反射机制，也就是**Subscribe** annotation机制来查找监听者方法，即调用`findUsingReflection()`方法；否则会先尝试使用用户定制的机制来查找，查找失败时才会使用反射机制来查找，这也就是调用`findUsingInfo()`方法。

在`findUsingReflection()`方法中实现了如我们前面提到的用annotation标识监听者方法的方式下，查找监听者方法的过程：
```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```
在`findUsingReflection()`中，会借助于`FindState`来查找所有的监听者方法。`FindState`类的角色主要有三个，一是迭代器，用于在监听者类的整个继承体系中遍历每一个类；二是容器，用来存放找到的每一个监听者方法；三是对找到的方法做验证。

`findUsingReflectionInSingleClass()`方法找到一个特定类中所有的监听者方法。这里可以清楚地看到`EventBus`对于监听者方法的声明的完整限制。一个方法被认定为监听者方法的条件为：
1. 方法的可见性被声明为全局可见`public`，同时没有`abstract`、`static`、`bridge`及`synthetic`修饰。
2. 方法的参数只有一个。
3. 有annotation **Subscribe**的声明。
4. `FindState.checkAdd()`会实施一些额外的限制。主要是关于，类中声明了多个方法监听相同事件的处理；及方法override的处理，即在类的继承体系中，有多个类声明了相同的方法，对相同的事件进行监听。

`strictMethodVerification`是一个调试开关，如果开关打开，则当一个类声明了annotation **Subscribe**，但modifier或参数列表不满足要求时，会直接抛出异常。

我们再来详细地看一下FindState的定义：
```
    static class FindState {
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        final StringBuilder methodKeyBuilder = new StringBuilder(128);

        Class<?> subscriberClass;
        Class<?> clazz;
        boolean skipSuperClasses;
        SubscriberInfo subscriberInfo;

        void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }

        void recycle() {
            subscriberMethods.clear();
            anyMethodByEventType.clear();
            subscriberClassByMethodKey.clear();
            methodKeyBuilder.setLength(0);
            subscriberClass = null;
            clazz = null;
            skipSuperClasses = false;
            subscriberInfo = null;
        }

        boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
            Object existing = anyMethodByEventType.put(eventType, method);
            if (existing == null) {
                return true;
            } else {
                if (existing instanceof Method) {
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                return checkAddWithMethodSignature(method, eventType);
            }
        }

        private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());

            String methodKey = methodKeyBuilder.toString();
            Class<?> methodClass = method.getDeclaringClass();
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld);
                return false;
            }
        }

        void moveToSuperclass() {
            if (skipSuperClasses) {
                clazz = null;
            } else {
                clazz = clazz.getSuperclass();
                String clazzName = clazz.getName();
                /** Skip system classes, this just degrades performance. */
                if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                    clazz = null;
                }
            }
        }
    }
```
moveToSuperclass()是迭代器方法，用于帮助SubscriberMethodFinder在订阅者类的整个继承层次中进行遍历。这里可以看到，这个遍历过程一般不会遍历到Object才停止，如果订阅者类直接或间接继承了java标准库或android框架层的类的话，遍历到那些类的时候就会终止了。checkAdd(Method method, Class<?> eventType)方法施加我们前面提到的额外限制，这个过程大体为：
1. 订阅者类的整个继承层次中只有一个方法订阅了某个特定类型的事件，则该方法直接被认定为订阅者方法。
2. 继承层次中对同一个事件类型有多个订阅者方法的，只要不同方法的方法名不同，则都会被认定为订阅者方法。
3. 继承层次中某个被override的方法被声明了订阅者方法，则继承层次中深度最深的类的方法被认定为订阅者方法。子类比父类在继承层次中的深度要深。

SubscriberMethodFinder中FindState对象并不会总是被重新创建，而是有一个缓冲池，需要的时候从这个池子中获取所需得对象，而在对象用完之后，又会被重新归还到池子中，这些操作的详细过程可以在prepareFindState()和getMethodsAndRelease()这两个方法中看到：
```
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }

    private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```
findUsingInfo()和getSubscriberInfo()方法则允许用户插入自定义的订阅者方法声明机制：
```
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
......
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```
所谓的用户自定义订阅者方法声明机制，需要通过实现SubscriberInfoIndex和SubscriberInfo接口来实现，并通过EventBusBuilder在创建EventBus对象时传递进来。SubscriberInfoIndex和SubscriberInfo接口的定义为：
```
package org.greenrobot.eventbus.meta;

/**
 * Interface for generated indexes.
 */
public interface SubscriberInfoIndex {
    SubscriberInfo getSubscriberInfo(Class<?> subscriberClass);
}
```
和
```
package org.greenrobot.eventbus.meta;

import org.greenrobot.eventbus.SubscriberMethod;

/** Base class for generated index classes created by annotation processing. */
public interface SubscriberInfo {
    Class<?> getSubscriberClass();

    SubscriberMethod[] getSubscriberMethods();

    SubscriberInfo getSuperSubscriberInfo();

    boolean shouldCheckSuperclass();
}
```
至此，EventBus中，注册订阅者的过程就分析完了。
