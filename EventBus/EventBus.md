# EventBus源码理解
EventBus是我们在开发中经常使用的开源库，使用起来比较简单，而且源码看起来不是很吃力。受到广大开发者的喜爱~

综述 ![Alt text](./EventBus-Publish-Subscribe.png)
上面这张图片很好的解释了EventBus工作流程，简单来说就是事件被提交到EventBus之后进行查找所有订阅该事件的方法然后执行这些方法.
###获取EventBus实例（单例模式）
#### 使用了双重判断的方式，防止并发的问题，还能极大的提高效率。
``` java
public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null){
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```
#### 构造方法
``` java
public EventBus() {
        this(DEFAULT_BUILDER);
    }
```
### 注册
``` java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
这里面其中参数就是订阅者，也就是我们写的this，register方法主要完成两件事,查找订阅者中所有的订阅方法，然后通过遍利订阅着的订阅方法完成订阅操作。我们首先看下findSubscriberMethods这个方法：
``` java
//从缓存中获取SubscriberMethod集合
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
//ignoreGeneratedIndex是否忽略注解器生成的MyEventBusIndex
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
SubscriberMethod 这个类中主要是用保存订阅方法的Method对象，线程模式，事件类型，优先级，是否粘性事件等属性，主要是两个方法findUsingReflection(subscriberClass)，findUsingInfo(subscriberClass)，这两个方法的区别就是有没有配置subscriberInfo
#### findUsingInfo
``` java
 private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
 //在FindState里面，它保存了一些订阅者的方法以及对订阅方法的校验
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        // 如果我们通过EventBusBuilder配置了MyEventBusIndex，便会获取到subscriberInfo 通常情况下我们下代码的时候并没有配置~
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
                //通过反射来查找订阅方法
             findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```
### # findUsingReflectionInSingleClass
* 没有通过EventBusBuilder配置MyEventBusIndex的情况下就执行这个方法了
``` java
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
                //定于方法中只能有一个参数
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                     //保存到findState对象当中
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
回到register这个方法中，上面我们分析了寻找订阅着方法部分，接下来就是注册了
#### subscribe 
``` java
// Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    //获取订阅者方法中的订阅事件
        Class<?> eventType = subscriberMethod.eventType;
        //创建一个Subscription来保存订阅者和订阅方法
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        //获取当前订阅事件中Subscription的List集合
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
         //该事件对应的Subscription的List集合不存在，则重新创建并保存在subscriptionsByEventType中
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
        //肯定药判断订阅者是否已经被注册啦
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
//将newSubscription按照订阅方法的优先级插入到subscriptions中
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
//通过订阅者获取该订阅者所订阅事件的集合
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        //将当前的订阅事件添加到subscribedEvents中
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
这个方法才是真正的注册，上面我们说的知识寻找订阅者订阅事件的方法。概括来说首先会根据subscriber和subscriberMethod来创建一个Subscription集合subscriptions，然后根据事件类型eventType获取事件集合并把他们添加到typesBySubscriber中，然后把Subscription对象添加到subscriptions中。
### 事件的发送
首先药获取EventBus对象，然后通过Post方法进行事件的发送
```java
   
    public void post(Object event) {
    //PostingThreadState保存着事件队列和线程状态信息
        PostingThreadState postingState = currentPostingThreadState.get();
        //获取事件队列，并将当前事插入到事件队列中
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
        //当前线程是否为主线程
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            // 判断是否取消
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
            //处理队列中的所有事件
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
 ```
 上面在订阅的时候我们以订阅事件为key，将Subscription的List集合作为Value保存到了一个Map中 ，下面这个方法就是通过key来取出集合
```java
   
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
 ```
 > 这就是我们在接收信息的时候所用到的几个方法，具体含义就不再啰嗦了。到这里啊管理EventBus所涉及的源码分析的差不多了，虽然还有好多地方没有分析到位，但大体的思路是有的。
