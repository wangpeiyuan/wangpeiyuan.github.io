---
title: EventBus 源码浅析
date: 2019-12-21 22:35:52
categories: Android
tags:
---
## 一、前言

EventBus 相信大家已经很熟悉了，一个发布/订阅事件的处理库。

我们这里将会从使用到原理来进一步理解 EventBus 。

<!--more-->

## 二、EventBus 的使用

首先，我们先来回顾一下 EventBus 的使用，一共三个步骤。

第一步：定义事件。

```java
class MessageEvent {
    //添加所需的属性
}
```

第二步：在需要接收事件的地方，即订阅者，定义接收方法以及注册订阅者。

```java
//定义接收方法
@Subscribe(threadMode = ThreadMode.MAIN, priority = 1)
fun onMessageEvent(messageEvent: MessageEvent) {
   //do something
}

@Subscribe(threadMode = ThreadMode.MAIN, priority = 1, sticky = true)
fun onStickyMessageEvent(messageEvent: MessageEvent) {
   //do something
}

//注册订阅者
override fun onStart() {
    super.onStart()
    EventBus.getDefault().register(this)
}

override fun onStop() {
    super.onStop()
    EventBus.getDefault().unregister(this)
}
```

第三步：发送事件。

```java
EventBus.getDefault().post(MessageEvent())
//发送粘性事件
EventBus.getDefault().postSticky(MessageEvent())
```

接下来，我们将深入源码来看看以上的几步操作，内部究竟做了什么处理来实现订阅与发布的。

分析的顺序如下：

1. `EventBus.getDefault().register(this)` 内部做了哪些操作？
2. `EventBus.getDefault().post(MessageEvent())` 和 `EventBus.getDefault().postSticky(MessageEvent())` 如何发送事件？
3. `@Subscribe()...` 接收事件的方法是如何接收到事件的？
4. `EventBus.getDefault().unregister(this)` 又做了哪些处理？

## 三、EventBus 源码浅析

> 注：基于 EventBus 3.1.1 版本进行分析。涉及的代码会做相应的简化处理，只保留关键部分。

### 3.1 注册订阅者 register

首先， EventBus 的实例是如何初始化的。

```java
//EventBus.java
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```

可以发现，EventBus 的实例是通过 `getDefault()` 方法实现的一个单例。

```java
//EventBus.java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}

//SubscriberMethod.java
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;

    //...
}
```

> **注：下面出现的订阅者指的是 `Object subscriber` ，在实际中可以是一个 Activity 之类的。订阅方法指的是 被 `@Subscribe` 注解标识的方法。**

`register` 方法中的代码不多，从这段代码中可以大概知道先是寻找订阅者中的所有订阅方法的集合，然后将这些方法一一记录下来。

接下来我们看看是如何获取订阅方法集合的，可以先大概猜测一下，参数是一个 Class 大概率是通过反射方式获取的，那好，我们跟进代码看看我们猜测是否正确。

```java
//SubscriberMethodFinder.java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    //是否强制使用反射，默认 false
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

可以发现，我们猜测使用了反射这个没错，但不够完整，这里面还可以使用其他的方式来获取订阅方法的集合。先来看看这种方式是什么。

```java
//SubscriberMethodFinder.java
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
```

`subscriberInfo` 在这里面起到至关重要的作用，如果为空就会再次使用反射获取订阅方法，如果不为空则通过 `subscriberInfo.getSubscriberMethods()` 方法来获取。

```java
//SubscriberInfo.java
/** Base class for generated index classes created by annotation processing. */
public interface SubscriberInfo {
    Class<?> getSubscriberClass();

    SubscriberMethod[] getSubscriberMethods();

    SubscriberInfo getSuperSubscriberInfo();

    boolean shouldCheckSuperclass();
}
```

通过注释就可以知道，这是通过注解在编译时期就已经获取了相关订阅方法，因此在这里就可以很快的得到，提高了性能（我们前面使用的时候并没有引入注解相关的，如果想了解 EventBus 注解的使用请查看[Subscriber Index](http://greenrobot.org/eventbus/documentation/subscriber-index/)）。关于注解这里简单说一下，内部也是使用反射获取相关方法，只是将获取的时机提前到了编译时期，具体的话就不再深入，各位如果有兴趣可以去研究一下。

接下来，看一下采用反射获取方法的方式。

```java
//SubscriberMethodFinder.java
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
            }
            //...
        }
        //...
    }
}
```

基本都是反射的基本用法，需要注意的是，父类的相关方法也会进行查找，但会过滤掉系统本身的类。

```java
//SubscriberMethodFinder.java
private static final int POOL_SIZE = 4;
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    //...
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

这里额外说一下 `FIND_STATE_POOL` ，在阅读源码的时候会发现很多地方都使用到了 Pool，比如 Handler 的 Message Pool，Glide 和 Fresco 的 BitmapPool 等等，使用 Pool 的好处显而易见，避免重复的创建对象。

好了，到这里我们通过使用反射或注解的方式获取到了订阅类中订阅方法的集合，然后我们接下来看看如何处理集合中的方法。

```java
//EventBus.java
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    }
    //...

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

//Subscription.java
final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

    //...
}
```

注释上要求 `subscribe` 方法必须运行在同步代码块中，EventBus 很多地方使用了 `synchronized` 来使得在多线程中可以正常的使用。

通过代码可以发现这里主要处理了三件事情：

1. 将事件类型（如 MessageEvent）作为 Key，按照优先级（priority）排序的订阅者含订阅方法列表作为 value 记录在 map `subscriptionsByEventType` 中。
2. 记录订阅者所拥有的所有事件类型。
3. 如果当前的订阅方法是 `sticky` 粘性事件就直接发送该事件（发送的方法后续会分析到）。

到这里我们分析完了 `EventBus.getDefault().register(subscribe)` 里面做了什么处理，然后我们先做个小结。

**小结：EventBus 是一个单例，通过 `getDefault()` 方法获取实例，在 `register(Object subscriber)` 方法中，先通过反射或者注解的方式获取到 `subscriber` 类中所有的订阅方法（被`@Subscribe` 标识的方法）的列表，然后遍历该列表并记录事件和方法（方法按照 `priority` 大小排列），如果当前遍历的方法是一个 `sticky` 就会先发送该事件。**

### 3.2 发送事件 post 和 postSticky 和响应事件

这里先简单看一下当我们调用 post 之后内部调用了哪些方法。

```java
EventBus.post(Object event)
 ->EventBus.postSingleEvent(Object event, PostingThreadState postingState)
   ->EventBus.postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass)
    ->EventBus.postToSubscription(Subscription subscription, Object event, boolean isMainThread)
```

发现最终都到 `postToSubscription` 方法中，虽然 `postSticky` 没在这里列出来但是一样的。接下来我们一个个方法跟过去。

```java
//EventBus.java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
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
    //...
}
```

首先，从当前线程中获取到 `postingState` ，这里 `PostingThreadState` 是从  `ThreadLocal` （线程本地存储区）初始化获取，保证每个线程的 `postingState` 都是独立。然后将事件放入队列，按先进先出顺序发送。

这里需要注意 **`eventInheritance` 这个参数，它主要用来控制是否触发当前事件的父类事件的方法**，什么意思呢。

```java
open inner class MessageEvent {}

inner class SubMessageEvent() : MessageEvent() {
}

@Subscribe
fun onMessageEvent(messageEvent: MessageEvent) {
}

@Subscribe
fun onSubMessageEvent(subMessageEvent: SubMessageEvent) {
}
```

比如，我们有一个 `MessageEvent` 事件，同时还有一个继承于 `MessageEvent` 的 `SubMessageEvent` 的事件，同时订阅这两个事件。

这时如果发送一个 `SubMessageEvent` 事件 `EventBus.getDefault().post(SubMessageEvent())` ，`onMessageEvent()` 和 `onSubMessageEvent()` 这两个方法会被同时触发，即都会接收到事件。如果不想父类事件的订阅方法被触发的话，需要将 `eventInheritance` 设为 `false` ，默认为 `true`。

`eventInheritance` 这个参数在上一节 `register` 中也有使用到，需要注意下。

```java
//EventBus.java
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

```

主要是找到对应事件的订阅类，然后循环发送。

```java
//EventBus.java
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
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
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

void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        }
        //...
    }
```

可以发现 `postToSubscription` 方法主要处理根据订阅方法指定的 `threadMode` 切换到对应的线程，最后在该线程通过 `invoke` 方式调到订阅方法。

`ThreadMode` 线程模式，目前有 5 种：

1. POSTING：订阅方法被调用的线程跟发送事件所在的线程是同一个。如果没有指定的话，这个是默认的线程模式。需要注意由于可能运行在主线程，所以不能进行耗时操作，否则会阻塞主线程。
2. MAIN：订阅方法将在主线程（UI 线程）被调用，不管发送事件在哪个线程。如果发送事件恰好在主线程则直接调用，如果不在主线程则通过 `mainThreadPoster` 来切换线程调用。
3. MAIN_ORDERED：跟 MAIN 方法差不多都是在主线程调用，不同的是，事件是在队列中排队等待调用。此时发送事件的方法不会被阻塞。
4. BACKGROUND：订阅方法将在后台线程被调用。如果发送事件的方法不在主线程，就直接调用订阅方法；如果在主线程，则用一个单一的子线程按顺序调用订阅方法，所以为了避免影响到其他接收事件方法的执行，不能在订阅方法中处理耗时操作，如果要处理耗时操作则使用 `ASYNC`。
5. ASYNC：订阅方法会在另外的一个线程中被调用。不管发送事件所处的线程在后台还是主线程，都是重新启用一个新的线程来调用订阅方法。此模式可以用来处理耗时操作。

上面我们大概了解一下各种线程模式下调用订阅方法的是方式，接下来我们深入了看一下是如何进行线程间切换的呢。

首先，**POSTING** 直接调用，没啥好说的。

接下来 **MAIN**，分为两个条件，如果当前就在主线程，直接调用；否则进入 `mainThreadPoster.enqueue(subscription, event)` 方法。主线程的判断之前没提到，这里简单说一下，主要是通过 Looper 的方式 `Looper.myLooper() == Looper.getMainLooper()`。

先来看看 `mainThreadPoster` 实例时如何生成的。

```java
//EventBus.java
private final MainThreadSupport mainThreadSupport;
private final Poster mainThreadPoster;

mainThreadSupport = builder.getMainThreadSupport();
mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;

//MainThreadSupport.AndroidHandlerMainThreadSupport.java
@Override
public Poster createPoster(EventBus eventBus) {
    return new HandlerPoster(eventBus, looper, 10);
}

//EventBusBuilder.java
MainThreadSupport getMainThreadSupport() {
    //...
    Object looperOrNull = getAndroidMainLooperOrNull();
    return looperOrNull == null ? null :
                new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
    //...
}
Object getAndroidMainLooperOrNull() {
    //...
    return Looper.getMainLooper();
    //...
}
```

可以发现 `mainThreadPoster` 是 HandlerPoster 的实例，需要注意这里传入了一个参数 `MainLooper`。

```java
public class HandlerPoster extends Handler implements Poster {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    //...

    public void enqueue(Subscription subscription, Object event) {
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

//EventBus.java
void invokeSubscriber(PendingPost pendingPost) {
    Object event = pendingPost.event;
    Subscription subscription = pendingPost.subscription;
    PendingPost.releasePendingPost(pendingPost);
    if (subscription.active) {
        invokeSubscriber(subscription, event);
    }
}
```

可以发现 HandlerPoster 就是一个 Handler ，在加上前面实例化 HandlerPoster 的时候指定的 Looper 是一个 MainLooper，那就可以确保 `HandlerPoster.handleMessage()` 运行在主线程了。具体源码比较简单就不一一解析。

接下来是 **MAIN_ORDERED**，跟 MAIN 所调用方法一致。

再然后是 **BACKGROUND**，如果当前在主线程就进入 `backgroundPoster.enqueue(subscription, event)` ，否则直接调用。

```java
final class BackgroundPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;
    private volatile boolean executorRunning;
    //...

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
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }
}
```

BackgroundPoster 是一个 Runnable，通过线程池的方式运行在后台。这里面需要注意的是用了多线程的东西，如 `wait` ， `notifyAll` ，`synchronized` 关键字。

最后看一下 **ASYNC**，直接调用了 `asyncPoster.enqueue(subscription, event)` 方法。

```java
class AsyncPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    //...

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

一看跟  BackgroundPoster 差不多，仅仅是少了一些同步锁等相关的关键字。那么区别仅仅是这样吗，有点懵？

之前说过 BACKGROUND 是运行在单一的子线程中，保证事件按顺序交付。这里一看都是使用 `Executors.newCachedThreadPool()` 线程池（有则用，无则创建，无数量上限），那么怎么保证“单一”呢？可以发现 `ExecutorService().execute(this)` 之前使用了 `synchronized (this)` 同步锁关键字，保证了任一时间内有且仅有一个任务会被线程池执行。

前面解析的是 `post()` 方法，我们来看下 `postSticky()` 方法。

```java
//EventBus.java
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    post(event);
}
```

仅仅多了将 `stickyEvents` 事件保存 Map 中。

好了，到这里我们大致分析完了发送事件 post 和 postSticky 和响应事件，简单做个总结：

**根据 `event` 的类型，在第一步 `register` 中保存的 `subscriptionsByEventType` Map 中获取到对应的订阅类和方法 `Subscription`，将其加入到事件队列中，循环队列按序取出事件，根据订阅方法指定的 `threadMode` 线程模式切换到对应的线程调用订阅方法。**

### 3.3 取消订阅 unregister

最后，如果要退出的话，一般都需要取消订阅，防止泄露。处理的话也比较简单，主要是移除在订阅是所记录的集合。代码如下：

```java
//EventBus.java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}

private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

## 四、总结

通过以上的分析，相信大家对 EventBus 的原理有了进一步的理解。EventBus 当中运用了很多 `synchronized` 在阅读源码的时候可以仔细体会一下 EventBus 是如何保证多线程下运行的正常，还有里面所用到的如 Pool 的运用以及反射等等。

## 五、资源与推荐

1. [EventBus](https://github.com/greenrobot/EventBus)
