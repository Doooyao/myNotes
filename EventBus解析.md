## EventBus源码解析

### 入门逻辑解析

#### 注册

我们平时使用eventbus要先注册,这里看一下注册的逻辑

```java
EventBus.getDefault().register(this)
```

```java
//这里我们可以看到这个EventBus是一个单例
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
接下来我们看一下注册eventbus的代码
```java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();//首先我们获取订阅者的Class
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);//这里通过subscriberMethodFinder来找到所有的订阅方法
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);//依次订阅
            }
        }
    }
```
然后我们看一下订阅方法是如何实现的
```java
 // Must be called in synchronized block 由注释可知,这个必须是被同步调用
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {//两个参数,一个是订阅者,一个是从订阅者中分离出来的订阅方法
        Class<?> eventType = subscriberMethod.eventType;//这有一个eventType 到底是啥呢,我们猜测是我们订阅时自己创建的event类,也就是订阅方法的参数
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);//生成一个订阅 要分清这些东西都是eventbus自己的类,和rxjava可没啥关系
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);//根据我们订阅的事件类型获取缓存的订阅 这个subscriptionsByEventType就是一个hashmap
        if (subscriptions == null) {//之前没有对此类事件订阅
            subscriptions = new CopyOnWriteArrayList<>();//生成对这个事件的订阅list
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {//如果这个订阅之前被注册过了,报错,不能多次对一个订阅注册
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {//根据优先级进行插入,或者插入到最后.这个优先级是啥呢,我们以后再说
                subscriptions.add(i, newSubscription);//注册新的订阅
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);//这个typesBySubscriber也是一个map,持有订阅者所订阅的event的Class
        if (subscribedEvents == null) {//之前没有的话,新建
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);//把eventtype加进去

        if (subscriberMethod.sticky) {//这个属性 sticky 黏性 说实话 我没用过这个属性 所以等下再说
            if (eventInheritance) {
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
#### 取消注册
这里我们接着看一下取消注册的逻辑
```java
EventBus.getDefault().unregister(this)
```
```java
public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);//查到这个订阅者所订阅的所有时间类型
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);//取消注册事件
            }
            typesBySubscriber.remove(subscriber);//从map中删除这个订阅者
        } else {
            Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```
```java
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);//查找所有订阅了eventtype的订阅关系
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {//停止并移除由subscriber产生的订阅关系
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```
我们可以看到取消注册逻辑并不复杂,分别在两个map中移除相应的关系就可以了

#### 分发事件
先看看简单的分发事件
```java
public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();//currentPostingThreadState是一个threadlocal,保存PostingThreadState
        //我们在这里取出当前线程的PostingThreadState
        List<Object> eventQueue = postingState.eventQueue;//获取当前线程的事件队列
        eventQueue.add(event);//把事件添加进队列等待被发送

        if (!postingState.isPosting) {//如果当前线程没有正在发送事件
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();//设置是不是主线程
            postingState.isPosting = true;//发送事件
            if (postingState.canceled) {//不可退出
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);//在本线程循环发送事件,从头发送
                }
            } finally {
                postingState.isPosting = false;//发送结束
                postingState.isMainThread = false;//这个设置为啥呢 暂时不是很理解有什么必要
            }
        }
    }
```
我们先简单的看一下这个PostingThreadState
##### PostingThreadState
```java
final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<Object>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
```
比较简单,不解释连招
然后看一下发送一个事件
##### postSingleEvent
```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {//有可能会抛出异常
        Class<?> eventClass = event.getClass();//拿到事件的class
        boolean subscriptionFound = false;//是否找到了订阅关系
        if (eventInheritance) {//事件继承 暂时先不看他
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);//发送事件
        }
        if (!subscriptionFound) {//没找到
            if (logNoSubscriberMessages) {
                Log.d(TAG, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));//发送没有订阅者的事件
            }
        }
    }
```
##### postSingleEventForEventType
``` java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);//加锁 取出这个事件的订阅关系
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {//循环发送事件
                postingState.event = event; //设置本线程当前正在发送的事件
                postingState.subscription = subscription;//设置本线程当前正在处理的订阅关系
                boolean aborted = false;//是否终止
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);//单独处理一个订阅关系的事件发送 下面会讲
                    aborted = postingState.canceled;//是否终止
                } finally {
                    postingState.event = null;//发送完毕,清空本线程的状态设置
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;//有订阅关系
        }
        return false;//无此事件的订阅关系
    }
```
##### postToSubscription
```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {//根据订阅关系中订阅方法设置的线程模式进行分发 具体怎么设置的我们下面再讲,这里只要先知道有这么几种模式就可以了
            case POSTING://默认模式,在发送事件就在哪里调用订阅者的方法
                invokeSubscriber(subscription, event);
                break;
            case MAIN://在主线程调用订阅者方法
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);//非本线程调用
                }
                break;
            case BACKGROUND://后台调用订阅者方法
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);//非本线程调用
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC://异步调用订阅者方法
                asyncPoster.enqueue(subscription, event);//非本线程调用
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
接下来我们依次进行分析
##### invokeSubscriber
```java
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);//这里可以看到了 直接反射调用
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());//处理异常
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
然后我们看到当需要在非本线程调用的时候,采用的是***Poster.enqueue的方式,我们来看一下这是个啥
##### Poster
在分发事件的时候,我们用到了mainThreadPoster,backgroundPoster 和 asyncPoster,但其实点进去看看实现,根本就没有Poster这么个类或者接口,是我胡咧咧的,只是他们名字长得比较像,在这里的用途也差不多哈哈哈
这三个都是Eventbus实例持有的对象
```java
    private final HandlerPoster mainThreadPoster;
    private final BackgroundPoster backgroundPoster;
    private final AsyncPoster asyncPoster;
    //在Eventbus构造方法里面实例化
    mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);//传入主线程Looper
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);
```
我们分别看一下这三个'Poster'的实现
###### HandlerPoster
看名字猜得出,和handler有关 上源码
```java
final class HandlerPoster extends Handler {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();//构造方法实例化一个队列,用于存储等待发送事件
    }

    void enqueue(Subscription subscription, Object event) {//这就是我们把事件发送给订阅者了
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);//包装成一个等待的发送
        synchronized (this) {//存加锁
            queue.enqueue(pendingPost);//把这个等待入队
            if (!handlerActive) {//这个handler是否正在运行
                handlerActive = true;//设置正在运行
                if (!sendMessage(obtainMessage())) {//发送一个消息 这个消息随便,反正这个handler就这里发送消息
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {//looper所在线程收到并处理消息
        boolean rescheduled = false;//是否推迟
        try {
            long started = SystemClock.uptimeMillis();//开始事件
            while (true) {//循环
                PendingPost pendingPost = queue.poll();//从等待发送事件队列中取出一个
                if (pendingPost == null) {//没了,防止存的数据还没存好取不出来,加锁等存好了再取一遍
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {//彻底没数据了,停止活动,return
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);//在lopper所在线程调用订阅者的方法
                long timeInMethod = SystemClock.uptimeMillis() - started;//方法运行事件
                if (timeInMethod >= maxMillisInsideHandleMessage) {//如果订阅者方法运行的事件超时了
                    if (!sendMessage(obtainMessage())) {//重启一下这个队列 置于原因,现在不知道
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
###### BackgroundPoster
后台运行 直接上代码
```java
final class BackgroundPoster implements Runnable {//这就是一个runnable

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {//入队操作
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);//包装成PendingPost
        synchronized (this) {
            queue.enqueue(pendingPost);//入队
            if (!executorRunning) {//执行器是否正在执行
                executorRunning = true;//执行
                eventBus.getExecutorService().execute(this);//这里看到,我们用eventBus持有的线程池来执行此runnable
            }
        }
    }

    @Override
    public void run() {//runnable方法, 和上面的handlerPoster差不多,有两处不同
        try {
            try {
                while (true) {
                    PendingPost pendingPost = queue.poll(1000);//1这里传参1000 暂时不知道是啥
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
                    //没有限制最大事件,也不知道是啥
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
###### AsyncPoster
异步poster
```java
class AsyncPoster implements Runnable {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);//这里可以看出来,所谓异步就是每一次订阅事件的发送都直接丢给线程池去处理
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
好啦,截止到这里,一个比较简单的eventbus注册订阅发送事件接受事件取消注册的流程就走完了
但是前面的代码里有几个方法和类我们并没有详细的看,只是猜了猜他们是干啥的,下面先详细的看一下

按照顺序,第一个类就是SubscriberMethod了,它是在注册的时候由subscriberMethodFinder.findSubscriberMethods(subscriberClass)生成的并保存的,那我们一起看一下
### SubscriberMethod

##### SubscriberMethod类
```java
public class SubscriberMethod {
    final Method method;//订阅者的方法
    final ThreadMode threadMode;//回调模式,上面讲过了
    final Class<?> eventType;//订阅的事件类型
    final int priority;//优先级
    final boolean sticky;//是否是黏性
    /** Used for efficient comparison */
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {//构造方法
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }

    @Override
    public boolean equals(Object other) {//重写equals方法
        if (other == this) {
            return true;
        } else if (other instanceof SubscriberMethod) {
            checkMethodString();//这个实际上就是使用类名+方法名+eventype类型拼接,也就是说判断这三个是否一样
            SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
            otherSubscriberMethod.checkMethodString();
            // Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }
    
    //使用类名+方法名+eventype类型拼接 并缓存
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
接下来我们看看一个SubscriberMethod是怎么生成的
##### SubscriberMethodFinder
这个类相对来说比较重要啦,我们来看一下源码,比较多
```java
class SubscriberMethodFinder {
    /*
     * In newer class files, compilers may add methods. Those are called bridge or synthetic methods.
     * EventBus must ignore both. There modifiers are not public but defined in the Java class file format:
     * http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1
     */
    private static final int BRIDGE = 0x40;
    private static final int SYNTHETIC = 0x1000;

    private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;//要忽略的方法修饰符
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();//方法缓存hashmap

    private List<SubscriberInfoIndex> subscriberInfoIndexes;
    private final boolean strictMethodVerification;
    private final boolean ignoreGeneratedIndex;

    private static final int POOL_SIZE = 4;
    private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];//是一个findstate的空闲池
    //上面就是参数了,有好多我们都不认识,没关系,一步一步看

    SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {//这是构造方法
        this.subscriberInfoIndexes = subscriberInfoIndexes;//这个我们使用注解处理器生成的,,里面存贮了所有添加了sub注解的可用的方法和他的索引
        this.strictMethodVerification = strictMethodVerification;//是否对方法严格检查模式,这样的话不符合的方法会报错
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;//是否忽略注解处理器生成的索引
    }

    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {//这里就是根据订阅者的Class类生成咱们的SubscriberMethod的方法了
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);//先看看缓存里面有没有
        if (subscriberMethods != null) {//有的话直接返回
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {//是否忽略注解处理器生成的索引
            subscriberMethods = findUsingReflection(subscriberClass);//忽略,直接使用反射查找
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);//通过索引进行查找
        }
        if (subscriberMethods.isEmpty()) {//如果没有 报错
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);//加入缓存
            return subscriberMethods;
        }
    }

    //先看这个,通过预先生成的索引查找
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();//这里有一个FindState内部类,保存了一次查找的状态 这么说不是很清晰,继续看
        //看完了各大概,现在回来看看prepareFindState();
        findState.initForSubscriber(subscriberClass);//通过订阅者Class初始化
        while (findState.clazz != null) {//循环查找
            findState.subscriberInfo = getSubscriberInfo(findState);//查找并设置info给state
            if (findState.subscriberInfo != null) {//订阅者信息不为空
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();//生成所有的SubscriberMethod
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {//循环比对时都需要添加
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);//使用反射查找
            }
            findState.moveToSuperclass();//指针移动到父类
        }
        return getMethodsAndRelease(findState);
    }

    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {//查找完毕,
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);//取出subscriberMethods
        findState.recycle();//回收
        synchronized (FIND_STATE_POOL) {//放进回收池
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
        synchronized (FIND_STATE_POOL) {//从闲置池中拿一个findstate出来
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();//如果没有就搞一个新的
    }

    private SubscriberInfo getSubscriberInfo(FindState findState) {
        //第一次查找的话状态里肯定没有保存subscriberInfo
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        //设置了索引
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {//遍历索引
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }

    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    private void findUsingReflectionInSingleClass(FindState findState) {//使用反射查找
        Method[] methods;//方法
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();//包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。当然也包括它所实现接口的方法
        } catch (Throwable th) {//报错了
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();//这样查找的话连父类的方法都查出来了
            findState.skipSuperClasses = true;//跳过父类
        }
        for (Method method : methods) {//循环
            int modifiers = method.getModifiers();//获取修饰符
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {//不包含上面的几个修饰符
                Class<?>[] parameterTypes = method.getParameterTypes();//获取方法的所有参数
                if (parameterTypes.length == 1) {//只有一个参数
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);//是否添加了@Subscribe注解
                    if (subscribeAnnotation != null) {//添加了
                        Class<?> eventType = parameterTypes[0];//拿到参数Class
                        if (findState.checkAdd(method, eventType)) {//检查添加事件类型和对应的方法 
                            ThreadMode threadMode = subscribeAnnotation.threadMode(); //获取注解生成的线程模式
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));//添加一个SubscriberMethod进去
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {//如果添加了Subscribe注解并且开启了对方法的严格检查
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();//报错
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {//如果添加了Subscribe注解并且开启了对方法的严格检查
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();//报错
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }

    static void clearCaches() {
        METHOD_CACHE.clear();
    }

    static class FindState { //内部类 代表一个查找的状态
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();//SubscriberMethod数组
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();//方法和eventType的对应集合
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();//订阅者的类和MethodKey的对应集合
        final StringBuilder methodKeyBuilder = new StringBuilder(128);//用于构造methodKey

        Class<?> subscriberClass;//本订阅者的类
        Class<?> clazz;//要被查找的类指针
        boolean skipSuperClasses;//是否跳过父类
        SubscriberInfo subscriberInfo;//订阅者信息 这是一个接口

        //初始化
        void initForSubscriber(Class<?> subscriberClass) {//初始化订阅者Class
            this.subscriberClass = clazz = subscriberClass;//当前类指向订阅者类
            skipSuperClasses = false;//设置默认
            subscriberInfo = null;
        }

        //回收
        void recycle() {
            subscriberMethods.clear();//清空各个数组
            anyMethodByEventType.clear();
            subscriberClassByMethodKey.clear();
            methodKeyBuilder.setLength(0);
            subscriberClass = null;
            clazz = null;
            skipSuperClasses = false;
            subscriberInfo = null;
        }

        //这个方法的代码,我对于中间的异常抛出不是很理解,在这里请教一下各位老哥,禁止这种情况的意义是啥呢?比如父类有两个方法订阅相同的event,子类重写了这两个方法并也添加了@Sub注解,就会报异常了,但是和重写或者相同的event关系不大,看下面的代码就知道只有子类父类订阅同event以及重写的方法数量特定的情况下才会触发这个异常
        boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            //判断两次 
            // Usually a subscriber doesn't have methods listening to the same event type.//通常一个订阅者不会有多个方法订阅同一个event
            Object existing = anyMethodByEventType.put(eventType, method);//之前是否有方法订阅此事件
            if (existing == null) {//没有,返回true
                return true;
            } else {
                //之前有方法订阅这个事件了 这就说明在订阅者中出现了多个方法都订阅了此事件
                if (existing instanceof Method) {//这里进行判断 如果是method,说明里面的method
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check//顾明思意,这是一个'妄想'的检查,理论上不会出现这种情况 所以我们先看看看这到底检查的是个啥
                        throw new IllegalStateException();//报错
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);//
                }
                return checkAddWithMethodSignature(method, eventType);
            }
        }
66
        private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {//这个方法的作用是把method和eventtype保存起来,如果之前保存过一样说明一个是父类的方法,
            //一个是子类的方法,这种情况下 我们选择保留子类的方法
            //如果新的是子类或者以前没有保存过,返回true 反之返回false
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());

            String methodKey = methodKeyBuilder.toString();//methodKey = "methodName>eventTypeName"
            Class<?> methodClass = method.getDeclaringClass();//获取声明这个方法的class
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);//看看之前有没有保存这种方法名相同参数也相同的方法?
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                //如果没有,说明是不同名的方法,返回true
                //或者已有的方法是新方法的父类调用的 返回true
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld);//返回false
                return false;
            }
        }

        void moveToSuperclass() {//进去父类继续寻找
            if (skipSuperClasses) {//是否跳过父类
                clazz = null;
            } else {
                clazz = clazz.getSuperclass();//clazz为父类
                String clazzName = clazz.getName();
                /** Skip system classes, this just degrades performance. */
                if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {//父类为系统类,跳过
                    clazz = null;
                }
            }
        }
    }

}
```
### 黏性

前面看到这个就表示没见过,现在先把它怼了
因为我没用过TT,所以直接去百度了 有位老哥是这么解释的
```
所谓的黏性事件，就是指发送了该事件之后再订阅者依然能够接收到的事件。
使用黏性事件的时候有两个地方需要做些修改。一个是订阅事件的地方,
另一个是发布事件的地方，调用EventBus的postSticky方法来发布事件
```
好,这样理解就比较清晰了,因为黏性事件是先发送,后订阅,那我们看源码也按照这个顺序来
##### postSticky
```java
public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```
这一段代码可以说很简单了=,=就是把我们的黏性事件加了入了一个map里面存了起来,这里我们也就知道了,当发送多个类型的黏性事件时,后注册的订阅者只会触发最后一个
下面就可以说一说注册的情况了
##### 注册
上面之前已经上了代码,我们这里就只选和黏性有关的东西
```java
if (subscriberMethod.sticky) {//这里,注册方法的时候判断这是不是粘性的
            if (eventInheritance) {//event事件继承是否开启
                //暂时省略这部分代码
            } else {
                Object stickyEvent = stickyEvents.get(eventType);//取出相应的粘性事件
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);//检查是否发送这个粘性事件给当前订阅
            }
        }
```
```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {//event 存在
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());//这里直接发送这个黏性事件
        }
    }
```
可见这个黏性比较简单,只是在发送的时候存储起来,等待有人注册的时候看看是否需要发送,如果需要就发送
下面我们来看看上面省略的eventInheritance

#### eventInheritance

这个是干什么的呢?
我从网上搜来一段描述

如果post(A),A extends B implements C,D implements C
那么onEvent(A)、onEvent(B)、onEvent(C)会被调用，onEvent(D)不会被调用

很清晰啊,下面我就上相关代码,看看是如何实现的
```java
if (eventInheritance) {//这里是发送非黏性event的代码块,上面有出现过,只不过我们忽略了这个开启了event继承模式情况下里面的逻辑
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);//获取这个event的所有父类及接口
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {//依次发送
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);//发送事件,只要有一次成功就成功
            }
        } else {//不开启event继承模式
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);//发送事件
        }
```
lookupAllEventTypes方法:
```java
/** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
    private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);//这是一个eventClass对应其父类及所有的接口的缓存,第一次查询肯定是空的
            if (eventTypes == null) {//没有缓存
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;//先指向当前clazz
                while (clazz != null) {//不为空
                    eventTypes.add(clazz);//放入list
                    addInterfaces(eventTypes, clazz.getInterfaces());//把所有的接口也放进去
                    clazz = clazz.getSuperclass();//指向父类,继续查找添加
                }
                eventTypesCache.put(eventClass, eventTypes);//加入缓存
            }
            return eventTypes;//返回
        }
    }
```
addInterfaces方法:
```java
 /** Recurses through super interfaces. */
    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {//如果list中不存在
                eventTypes.add(interfaceClass);//加入
                addInterfaces(eventTypes, interfaceClass.getInterfaces());//递归加入
            }
        }
    }
```
下面我们再看看黏性事件的情况
```java
if (subscriberMethod.sticky) {//这是订阅者注册的时候
            if (eventInheritance) {//开启了event继承接收
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {//循环所有的黏性事件
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {//如果注册的事件是黏性事件的父类
                        Object stickyEvent = entry.getValue();//获取黏性事件的对象
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);//发送
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
```

#### 注解处理器
```java
@SupportedAnnotationTypes("org.greenrobot.eventbus.Subscribe")//要处理的注解
@SupportedOptions(value = {"eventBusIndex", "verbose"})//要处理的属性 第一个就是在build中设置的生成的文件名,第二个'冗余模式',是否打印输出信息
public class EventBusAnnotationProcessor extends AbstractProcessor {
    public static final String OPTION_EVENT_BUS_INDEX = "eventBusIndex";
    public static final String OPTION_VERBOSE = "verbose";

    /** Found subscriber methods for a class (without superclasses). */
    private final ListMap<TypeElement, ExecutableElement> methodsByClass = new ListMap<>();//一个map 方法和class的对应
    //
    private final Set<TypeElement> classesToSkip = new HashSet<>();

    private boolean writerRoundDone;
    private int round;
    private boolean verbose;

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latest();//支持的最低版本
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        Messager messager = processingEnv.getMessager();//用于打印log的对象
        try {
            String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);//文件名
            if (index == null) {
                messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
                        " passed to annotation processor");
                return false;
            }
            verbose = Boolean.parseBoolean(processingEnv.getOptions().get(OPTION_VERBOSE));//这个暂时不知道是啥
            int lastPeriod = index.lastIndexOf('.');//截取文件夹名
            String indexPackage = lastPeriod != -1 ? index.substring(0, lastPeriod) : null;

            round++;//计数加一
            if (verbose) {//打印信息
                messager.printMessage(Diagnostic.Kind.NOTE, "Processing round " + round + ", new annotations: " +
                        !annotations.isEmpty() + ", processingOver: " + env.processingOver());
            }
            if (env.processingOver()) {//这个暂时不懂是啥 todo
                if (!annotations.isEmpty()) {
                    messager.printMessage(Diagnostic.Kind.ERROR,
                            "Unexpected processing state: annotations still available after processing over");
                    return false;
                }
            }
            if (annotations.isEmpty()) {//empty
                return false;
            }

            if (writerRoundDone) {//是否已经写完了
                messager.printMessage(Diagnostic.Kind.ERROR,
                        "Unexpected processing state: annotations still available after writing.");
            }
            collectSubscribers(annotations, env, messager);//收集Subscribers
            checkForSubscribersToSkip(messager, indexPackage);//检查Subscriber应不应该被跳过

            if (!methodsByClass.isEmpty()) {//不为空,写入文件
                createInfoIndexFile(index);
            } else {
                messager.printMessage(Diagnostic.Kind.WARNING, "No @Subscribe annotations found");
            }
            writerRoundDone = true;
        } catch (RuntimeException e) {
            // IntelliJ does not handle exceptions nicely, so log and print a message
            e.printStackTrace();
            messager.printMessage(Diagnostic.Kind.ERROR, "Unexpected error in EventBusAnnotationProcessor: " + e);
        }
        return true;
    }
//剩下的方法没有写什么注释了,感兴趣的老哥可以自己看下//
    private void collectSubscribers(Set<? extends TypeElement> annotations, RoundEnvironment env, Messager messager) {
        for (TypeElement annotation : annotations) {//循环注解
            Set<? extends Element> elements = env.getElementsAnnotatedWith(annotation);//获取这个注解对应的元素
            for (Element element : elements) {
                if (element instanceof ExecutableElement) {//如果是方法
                    ExecutableElement method = (ExecutableElement) element;
                    if (checkHasNoErrors(method, messager)) {//检查有没有错误
                        TypeElement classElement = (TypeElement) method.getEnclosingElement();//获取声明这个方法的类
                        methodsByClass.putElement(classElement, method);//放进去
                    }
                } else {//不是方法,报错
                    messager.printMessage(Diagnostic.Kind.ERROR, "@Subscribe is only valid for methods", element);
                }
            }
        }
    }

    private boolean checkHasNoErrors(ExecutableElement element, Messager messager) {
        if (element.getModifiers().contains(Modifier.STATIC)) {//不能是静态方法
            messager.printMessage(Diagnostic.Kind.ERROR, "Subscriber method must not be static", element);
            return false;
        }

        if (!element.getModifiers().contains(Modifier.PUBLIC)) {//必须是public方法
            messager.printMessage(Diagnostic.Kind.ERROR, "Subscriber method must be public", element);
            return false;
        }

        List<? extends VariableElement> parameters = ((ExecutableElement) element).getParameters();//必须有且只有一个参数
        if (parameters.size() != 1) {
            messager.printMessage(Diagnostic.Kind.ERROR, "Subscriber method must have exactly 1 parameter", element);
            return false;
        }
        return true;
    }

    /**
     * 如果索引不可见其类或任何涉及的事件类，则应跳过订阅者类。
     */
    private void checkForSubscribersToSkip(Messager messager, String myPackage) {
        for (TypeElement skipCandidate : methodsByClass.keySet()) {//循环所有的查到的类
            TypeElement subscriberClass = skipCandidate;
            while (subscriberClass != null) {
                if (!isVisible(myPackage, subscriberClass)) {//这个类相对目标包不可见
                    boolean added = classesToSkip.add(skipCandidate);//添加进去
                    if (added) {//打印log
                        String msg;
                        if (subscriberClass.equals(skipCandidate)) {
                            msg = "Falling back to reflection because class is not public";
                        } else {
                            msg = "Falling back to reflection because " + skipCandidate +
                                    " has a non-public super class";
                        }
                        messager.printMessage(Diagnostic.Kind.NOTE, msg, subscriberClass);
                    }
                    break;
                }
                List<ExecutableElement> methods = methodsByClass.get(subscriberClass);//找到所有对应的方法
                if (methods != null) {
                    for (ExecutableElement method : methods) {//循环
                        String skipReason = null;
                        VariableElement param = method.getParameters().get(0);//参数对象
                        TypeMirror typeMirror = getParamTypeMirror(param, messager);//
                        if (!(typeMirror instanceof DeclaredType) ||
                                !(((DeclaredType) typeMirror).asElement() instanceof TypeElement)) {
                            skipReason = "event type cannot be processed";
                        }
                        if (skipReason == null) {
                            TypeElement eventTypeElement = (TypeElement) ((DeclaredType) typeMirror).asElement();
                            if (!isVisible(myPackage, eventTypeElement)) {
                                skipReason = "event type is not public";
                            }
                        }
                        if (skipReason != null) {
                            boolean added = classesToSkip.add(skipCandidate);
                            if (added) {
                                String msg = "Falling back to reflection because " + skipReason;
                                if (!subscriberClass.equals(skipCandidate)) {
                                    msg += " (found in super class for " + skipCandidate + ")";
                                }
                                messager.printMessage(Diagnostic.Kind.NOTE, msg, param);
                            }
                            break;
                        }
                    }
                }
                subscriberClass = getSuperclass(subscriberClass);
            }
        }
    }

    private TypeMirror getParamTypeMirror(VariableElement param, Messager messager) {
        TypeMirror typeMirror = param.asType();//获取参数类型
        // Check for generic type
        if (typeMirror instanceof TypeVariable) {
            TypeMirror upperBound = ((TypeVariable) typeMirror).getUpperBound();
            if (upperBound instanceof DeclaredType) {
                if (messager != null) {
                    messager.printMessage(Diagnostic.Kind.NOTE, "Using upper bound type " + upperBound +
                            " for generic parameter", param);
                }
                typeMirror = upperBound;
            }
        }
        return typeMirror;
    }

    private TypeElement getSuperclass(TypeElement type) {
        if (type.getSuperclass().getKind() == TypeKind.DECLARED) {
            TypeElement superclass = (TypeElement) processingEnv.getTypeUtils().asElement(type.getSuperclass());
            String name = superclass.getQualifiedName().toString();
            if (name.startsWith("java.") || name.startsWith("javax.") || name.startsWith("android.")) {
                // Skip system classes, this just degrades performance
                return null;
            } else {
                return superclass;
            }
        } else {
            return null;
        }
    }

    private String getClassString(TypeElement typeElement, String myPackage) {
        PackageElement packageElement = getPackageElement(typeElement);
        String packageString = packageElement.getQualifiedName().toString();
        String className = typeElement.getQualifiedName().toString();
        if (packageString != null && !packageString.isEmpty()) {
            if (packageString.equals(myPackage)) {
                className = cutPackage(myPackage, className);
            } else if (packageString.equals("java.lang")) {
                className = typeElement.getSimpleName().toString();
            }
        }
        return className;
    }

    private String cutPackage(String paket, String className) {
        if (className.startsWith(paket + '.')) {
            // Don't use TypeElement.getSimpleName, it doesn't work for us with inner classes
            return className.substring(paket.length() + 1);
        } else {
            // Paranoia
            throw new IllegalStateException("Mismatching " + paket + " vs. " + className);
        }
    }

    private PackageElement getPackageElement(TypeElement subscriberClass) {
        Element candidate = subscriberClass.getEnclosingElement();
        while (!(candidate instanceof PackageElement)) {
            candidate = candidate.getEnclosingElement();
        }
        return (PackageElement) candidate;
    }

    private void writeCreateSubscriberMethods(BufferedWriter writer, List<ExecutableElement> methods,
                                              String callPrefix, String myPackage) throws IOException {
        for (ExecutableElement method : methods) {
            List<? extends VariableElement> parameters = method.getParameters();
            TypeMirror paramType = getParamTypeMirror(parameters.get(0), null);
            TypeElement paramElement = (TypeElement) processingEnv.getTypeUtils().asElement(paramType);
            String methodName = method.getSimpleName().toString();
            String eventClass = getClassString(paramElement, myPackage) + ".class";

            Subscribe subscribe = method.getAnnotation(Subscribe.class);
            List<String> parts = new ArrayList<>();
            parts.add(callPrefix + "(\"" + methodName + "\",");
            String lineEnd = "),";
            if (subscribe.priority() == 0 && !subscribe.sticky()) {
                if (subscribe.threadMode() == ThreadMode.POSTING) {
                    parts.add(eventClass + lineEnd);
                } else {
                    parts.add(eventClass + ",");
                    parts.add("ThreadMode." + subscribe.threadMode().name() + lineEnd);
                }
            } else {
                parts.add(eventClass + ",");
                parts.add("ThreadMode." + subscribe.threadMode().name() + ",");
                parts.add(subscribe.priority() + ",");
                parts.add(subscribe.sticky() + lineEnd);
            }
            writeLine(writer, 3, parts.toArray(new String[parts.size()]));

            if (verbose) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "Indexed @Subscribe at " +
                        method.getEnclosingElement().getSimpleName() + "." + methodName +
                        "(" + paramElement.getSimpleName() + ")");
            }

        }
    }

    private void createInfoIndexFile(String index) {
        BufferedWriter writer = null;
        try {
            JavaFileObject sourceFile = processingEnv.getFiler().createSourceFile(index);
            int period = index.lastIndexOf('.');
            String myPackage = period > 0 ? index.substring(0, period) : null;
            String clazz = index.substring(period + 1);
            writer = new BufferedWriter(sourceFile.openWriter());
            if (myPackage != null) {
                writer.write("package " + myPackage + ";\n\n");
            }
            writer.write("import org.greenrobot.eventbus.meta.SimpleSubscriberInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberMethodInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberInfoIndex;\n\n");
            writer.write("import org.greenrobot.eventbus.ThreadMode;\n\n");
            writer.write("import java.util.HashMap;\n");
            writer.write("import java.util.Map;\n\n");
            writer.write("/** This class is generated by EventBus, do not edit. */\n");
            writer.write("public class " + clazz + " implements SubscriberInfoIndex {\n");
            writer.write("    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;\n\n");
            writer.write("    static {\n");
            writer.write("        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();\n\n");
            writeIndexLines(writer, myPackage);
            writer.write("    }\n\n");
            writer.write("    private static void putIndex(SubscriberInfo info) {\n");
            writer.write("        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);\n");
            writer.write("    }\n\n");
            writer.write("    @Override\n");
            writer.write("    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {\n");
            writer.write("        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);\n");
            writer.write("        if (info != null) {\n");
            writer.write("            return info;\n");
            writer.write("        } else {\n");
            writer.write("            return null;\n");
            writer.write("        }\n");
            writer.write("    }\n");
            writer.write("}\n");
        } catch (IOException e) {
            throw new RuntimeException("Could not write source for " + index, e);
        } finally {
            if (writer != null) {
                try {
                    writer.close();
                } catch (IOException e) {
                    //Silent
                }
            }
        }
    }

    private void writeIndexLines(BufferedWriter writer, String myPackage) throws IOException {
        for (TypeElement subscriberTypeElement : methodsByClass.keySet()) {
            if (classesToSkip.contains(subscriberTypeElement)) {
                continue;
            }

            String subscriberClass = getClassString(subscriberTypeElement, myPackage);
            if (isVisible(myPackage, subscriberTypeElement)) {
                writeLine(writer, 2,
                        "putIndex(new SimpleSubscriberInfo(" + subscriberClass + ".class,",
                        "true,", "new SubscriberMethodInfo[] {");
                List<ExecutableElement> methods = methodsByClass.get(subscriberTypeElement);
                writeCreateSubscriberMethods(writer, methods, "new SubscriberMethodInfo", myPackage);
                writer.write("        }));\n\n");
            } else {
                writer.write("        // Subscriber not visible to index: " + subscriberClass + "\n");
            }
        }
    }

    private boolean isVisible(String myPackage, TypeElement typeElement) {
        Set<Modifier> modifiers = typeElement.getModifiers();
        boolean visible;
        if (modifiers.contains(Modifier.PUBLIC)) {
            visible = true;
        } else if (modifiers.contains(Modifier.PRIVATE) || modifiers.contains(Modifier.PROTECTED)) {
            visible = false;
        } else {
            String subscriberPackage = getPackageElement(typeElement).getQualifiedName().toString();
            if (myPackage == null) {
                visible = subscriberPackage.length() == 0;
            } else {
                visible = myPackage.equals(subscriberPackage);
            }
        }
        return visible;
    }

    private void writeLine(BufferedWriter writer, int indentLevel, String... parts) throws IOException {
        writeLine(writer, indentLevel, 2, parts);
    }

    private void writeLine(BufferedWriter writer, int indentLevel, int indentLevelIncrease, String... parts)
            throws IOException {
        writeIndent(writer, indentLevel);
        int len = indentLevel * 4;
        for (int i = 0; i < parts.length; i++) {
            String part = parts[i];
            if (i != 0) {
                if (len + part.length() > 118) {
                    writer.write("\n");
                    if (indentLevel < 12) {
                        indentLevel += indentLevelIncrease;
                    }
                    writeIndent(writer, indentLevel);
                    len = indentLevel * 4;
                } else {
                    writer.write(" ");
                }
            }
            writer.write(part);
            len += part.length();
        }
        writer.write("\n");
    }

    private void writeIndent(BufferedWriter writer, int indentLevel) throws IOException {
        for (int i = 0; i < indentLevel; i++) {
            writer.write("    ");
        }
    }
}
```