#### Handler
这里的逻辑比较简单,直接上源码
##### handler
```java
public class Handler {//会省略部分代码

    private static final boolean FIND_POTENTIAL_LEAKS = false;
    private static final String TAG = "Handler";
    private static Handler MAIN_THREAD_HANDLER = null;
    
    //callback接口
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
    
    //这就是我们处理消息的方法
    public void handleMessage(Message msg) {
    }
    
    //分发消息
    public void dispatchMessage(Message msg) {//在这里我们可以看到对于msg的处理优先级
        if (msg.callback != null) {
            handleCallback(msg);//如果msg自带callback,则用callback处理
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {//如果handler有callback设置,则用callback处理
                    return;
                }
            }
            handleMessage(msg);//如果啥都没有,就用上面的handleMessage处理~
        }
    }
        
    //```省略一些构造方法```
    //在handler的构造方法中,不传Looper的构造方法最后会调用这个,传looper的会调用这个下面的那个~
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {//是否寻找可能的内存泄漏问题 
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                //如果是匿名|内部|本地类,则需要static,否则提示
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();//这里可以看到,因为没有制定looper,默认获取当前线程的looper,如果没有就报错
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;//
        mCallback = callback;
        mAsynchronous = async;
    }

    
    public Handler(Looper looper, Callback callback, boolean async) {//传了looper那就可以直接用
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    /** @hide */
    @NonNull
    public static Handler getMain() { //获取主线程handler
        if (MAIN_THREAD_HANDLER == null) {
            MAIN_THREAD_HANDLER = new Handler(Looper.getMainLooper());
        }
        return MAIN_THREAD_HANDLER;
    }

    //```省略方法```
  
    public final Message obtainMessage()//获取一个持有此handler的msg实例
    {
        return Message.obtain(this);
    }
    //```省略一些上面方法的重载方法,多出的参数可以设定msg的obj what arg等属性```

    //```省略和下面的方法差不多的方法,包括post()
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }
    
    public final boolean postDelayed(Runnable r, Object token, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r, token), delayMillis);
    }
    
    public final boolean postAtFrontOfQueue(Runnable r)
    {
        return sendMessageAtFrontOfQueue(getPostMessage(r));//发送消息并放在最前面
    }
    
    //这个方法不是很安全
    public final boolean runWithScissors(final Runnable r, long timeout) {
        if (Looper.myLooper() == mLooper) {//如果当前线程就是hanler持有的线程
            r.run();//直接运行并返回
            return true;
        }
        BlockingRunnable br = new BlockingRunnable(r);//不然调用这个 这个以后再说
        return br.postAndWait(this, timeout);//一直等待直到r被运行 失败则false
    }
    
    public final void removeCallbacks(Runnable r, Object token)
    {
        mQueue.removeMessages(this, r, token);
    }

    //```省略了一些类似的方法
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }//这个方法可以看到,延迟就是延时时间+当前时间实现的
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    //发送消息
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);//调用此方法
    }   
    //省略方法

    //执行或者发送消息
    public final boolean executeOrSendMessage(Message msg) {
        if (mLooper == Looper.myLooper()) {//当前线程,直接执行
            dispatchMessage(msg);
            return true;
        }
        return sendMessage(msg);//发送消息
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;//设置msg.target
        if (mAsynchronous) {//是不是异步handler
            msg.setAsynchronous(true);//设置属性
        }
        return queue.enqueueMessage(msg, uptimeMillis);//messagequeue发送消息
    }
    
    //```隐藏一些对msgqueue的操作方法 ,关于移除查找消息 ```
    
    public final void dump(Printer pw, String prefix) {//
        pw.println(prefix + this + " @ " + SystemClock.uptimeMillis());
        if (mLooper == null) {
            pw.println(prefix + "looper uninitialized");
        } else {
            mLooper.dump(pw, prefix + "  ");//这个暂时不知道是干了啥
        }
    }

    public final void dumpMine(Printer pw, String prefix) {
        pw.println(prefix + this + " @ " + SystemClock.uptimeMillis());
        if (mLooper == null) {
            pw.println(prefix + "looper uninitialized");
        } else {
            mLooper.dump(pw, prefix + "  ", this);//同上
        }
    }

    //Imessenger,binder 相关 Messenger用的
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }

    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }

    private static Message getPostMessage(Runnable r, Object token) {//生成message
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }

    private static void handleCallback(Message message) {//直接运行message中的callback
        message.callback.run();
    }

    final Looper mLooper;
    final MessageQueue mQueue;
    final Callback mCallback;
    final boolean mAsynchronous;
    IMessenger mMessenger;

    //这里就是前面提到的了 我们分析一下
    private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;//要运行的任务
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;//被运行的时候停止循环,解除锁
                    notifyAll();
                }
            }
        }
        //之前的代码就调用了这个方法
        public boolean postAndWait(Handler handler, long timeout) {
            if (!handler.post(this)) {//发送这个消息
                return false;
            }

            synchronized (this) {//拿锁
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {//在这里等着
                            return false; // timeout
                        }
                        try {
                            wait(delay);
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    while (!mDone) {
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
}
```
##### Message
这就是我们发送的消息 代码比较多,我们省略部分
```java
public final class Message implements Parcelable {
  
    //先看一下参数 
    public int what;
    public int arg1;
    public int arg2;
    public Object obj;
    //消息接受方用于回复的messenger
    public Messenger replyTo;
    //uid
    public int sendingUid = -1;
    
    /*package*/ static final int FLAG_IN_USE = 1 << 0; //正在使用的标记位
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;//异步的标记
    /*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;//
    /*package*/ int flags;
    /*package*/ long when;
    /*package*/ Bundle data;//传输的data数据 因为可能会用于跨进程传输
    /*package*/ Handler target;//目标handler
    /*package*/ Runnable callback;//回调
    /*package*/ Message next;//next 这里可以看出来这是个链表

    public static final Object sPoolSync = new Object();
    private static Message sPool;//回收池头节点
    private static int sPoolSize = 0;//
    private static final int MAX_POOL_SIZE = 50;//
    
    private static boolean gCheckRecycle = true;
    
    //获取一个message实例
    //可以看出来,优先从回收池中获取
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;//从回收池中拿出头结点
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;//返回
            }
        }
        return new Message();//返回一个新的
    }
    
    public static Message obtain(Message orig) {
        //```省略了属性复制的代码``` 这里就是先获取一个message对象,再复制属性
    }
    //```这里省略了大量的obtain重载方法```

    public static void updateCheckRecycle(int targetSdkVersion) {
        if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            gCheckRecycle = false;//使用中被回收是否报错
        }
    }
    //回收本message
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");//报错
            }
            return;
        }
        recycleUnchecked();//回收
    }
    
    void recycleUnchecked() {
        //```省略代码``` 清空数据,设置inuse标记位防止被使用
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {//如果回收池没满
                next = sPool;//放入回收池头部
                sPool = this;
                sPoolSize++;
            }
        }
    }
    
    //从o复制包含的信息属性,其他不复制
    public void copyFrom(Message o) {
        this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM;
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;
        this.sendingUid = o.sendingUid;

        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }
    
    //```省略一些设置和查看信息属性的方法```
    
    //发送自己给target处理
    public void sendToTarget() {
        target.sendMessage(this);
    }
    
    //```省略重写的tostring```
    
    void writeToProto(ProtoOutputStream proto, long fieldId) {
        //```写入输出流 ``` 实现省略了
    }

    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();//序列化方法,即使是序列化也要从obtain拿实例
            msg.readFromParcel(source);
            return msg;
        }

        public Message[] newArray(int size) {
            return new Message[size];
        }
    };
}
```
##### Looper 当然了,也省略了些无关代码
```java
public final class Looper {
    private static final String TAG = "Looper";
    //先看看属性
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();//这就是threadlocal了
    private static Looper sMainLooper;  //主线程looper
    final MessageQueue mQueue;//这里又见到你了message queue
    final Thread mThread;//本looper所在线程

    private Printer mLogging;
    private long mTraceTag;

    /**
     * If set, the looper will show a warning log if a message dispatch takes longer than this.
     */
    private long mSlowDispatchThresholdMs;

    /**
     * If set, the looper will show a warning log if a message delivery (actual delivery time -
     * post time) takes longer than this.
     */
    private long mSlowDeliveryThresholdMs;

    //初始化 这个干了啥呢
    public static void prepare() {
        prepare(true);//可以看出,一般的looper是允许退出的
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");//这里可以看到,一个线程只能有一个looper
        }
        sThreadLocal.set(new Looper(quitAllowed));//向当前线程存了一个Looper
    }

    //初始化主looper 
    public static void prepareMainLooper() {
        prepare(false);//主线程的不允许退出
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();//当前looper设置给主looper
        }
    }

    public static Looper getMainLooper() {//获取主looper
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {//开始当前现成的循环队列
        final Looper me = myLooper();//拿到本线程looper
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;//拿到queue

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {//开始循环
            Message msg = queue.next(); // 从queue中取出数据
            if (msg == null) {//没有东西,循环结束
                return;
            }
            //```省略一些代码```
            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);//依次调用handler分发方法
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
           //省略一些超时处理判断的代码
            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity(); //判断分发代码有没有导致线程被破坏
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            msg.recycleUnchecked();//回收msg
        }
    }
    //```省略超时打印log```

    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();//返回当前线程looper
    }
    
    public static @NonNull MessageQueue myQueue() {//当前线程mq
        return myLooper().mQueue;
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);//这里可以看到了,能不能退出是mq说了算
        mThread = Thread.currentThread();//绑定当前线程
    }
    
    //不安全退出循环
    public void quit() {
        mQueue.quit(false);
    }

    //安全退出循环
    public void quitSafely() {
        mQueue.quit(true);
    }
    //```省略打印及属性获取方法代码```
    
    /** @hide */
    public void writeToProto(ProtoOutputStream proto, long fieldId) {//写入流
        final long looperToken = proto.start(fieldId);
        proto.write(LooperProto.THREAD_NAME, mThread.getName());
        proto.write(LooperProto.THREAD_ID, mThread.getId());
        if (mQueue != null) {
            mQueue.writeToProto(proto, LooperProto.QUEUE);
        }
        proto.end(looperToken);
    }
    
    //省略toString
}
```
##### MessageQueue 用于保存message队列
这个类更是长,我也删除了一部分~~
```java
public final class MessageQueue {

    private final boolean mQuitAllowed;//是否能被退出
    private long mPtr; //

    Message mMessages;//这里持有一个message链表
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();//空闲handlers?县往下看
    private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;//文件记录的数组
    private IdleHandler[] mPendingIdleHandlers;//待空闲
    private boolean mQuitting;//正在退出

    private boolean mBlocked;

    private int mNextBarrierToken;
    
    //下面是一些native层方法
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);

    MessageQueue(boolean quitAllowed) {//构造函数,我们看到同时调用了native层的初始化,返回一个long,可以简单想象成一个id或者指向c层messagequeue的指针
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }

    @Override
    protected void finalize() throws Throwable {//对象被回收时清理数据
        try {
            dispose();
        } finally {
            super.finalize();
        }
    }
    
    private void dispose() {
        if (mPtr != 0) {
            nativeDestroy(mPtr);//native层mq的destory 可以看出来java和c层的mq是共存亡
            mPtr = 0;
        }
    }

    public boolean isIdle() {//判断是否空闲
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            return mMessages == null || now < mMessages.when;//下一个要执行的还没到时间,所以现在没啥事可做
        }
    }

    public void addIdleHandler(@NonNull IdleHandler handler) {//添加一个idlehandler 代码```省略```
    }

    public void removeIdleHandler(@NonNull IdleHandler handler) { //```省略```
    }
    //是否在轮训
    public boolean isPolling() {
        synchronized (this) {
            return isPollingLocked();
        }
    }

    private boolean isPollingLocked() {
        return !mQuitting && nativeIsPolling(mPtr);//底层正在轮训且java层未退出
    }
    
    public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
            @OnFileDescriptorEventListener.Events int events,
            @NonNull OnFileDescriptorEventListener listener) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, events, listener);//添加监听?暂时不知道这是干啥的
        }
    }
    
    public void removeOnFileDescriptorEventListener(@NonNull FileDescriptor fd) {//移除?不知道
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, 0, null);
        }
    }

    private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,//todo
            OnFileDescriptorEventListener listener) {
        final int fdNum = fd.getInt$();

        int index = -1;
        FileDescriptorRecord record = null;
        if (mFileDescriptorRecords != null) {
            index = mFileDescriptorRecords.indexOfKey(fdNum);
            if (index >= 0) {
                record = mFileDescriptorRecords.valueAt(index);
                if (record != null && record.mEvents == events) {
                    return;
                }
            }
        }

        if (events != 0) {
            events |= OnFileDescriptorEventListener.EVENT_ERROR;
            if (record == null) {
                if (mFileDescriptorRecords == null) {
                    mFileDescriptorRecords = new SparseArray<FileDescriptorRecord>();
                }
                record = new FileDescriptorRecord(fd, events, listener);
                mFileDescriptorRecords.put(fdNum, record);
            } else {
                record.mListener = listener;
                record.mEvents = events;
                record.mSeq += 1;
            }
            nativeSetFileDescriptorEvents(mPtr, fdNum, events);
        } else if (record != null) {
            record.mEvents = 0;
            mFileDescriptorRecords.removeAt(index);
            nativeSetFileDescriptorEvents(mPtr, fdNum, 0);
        }
    }

    // Called from native code.
    private int dispatchEvents(int fd, int events) {
        // Get the file descriptor record and any state that might change.
        final FileDescriptorRecord record;
        final int oldWatchedEvents;
        final OnFileDescriptorEventListener listener;
        final int seq;
        synchronized (this) {
            record = mFileDescriptorRecords.get(fd);
            if (record == null) {
                return 0; // spurious, no listener registered
            }

            oldWatchedEvents = record.mEvents;
            events &= oldWatchedEvents; // filter events based on current watched set
            if (events == 0) {
                return oldWatchedEvents; // spurious, watched events changed
            }

            listener = record.mListener;
            seq = record.mSeq;
        }

        // Invoke the listener outside of the lock.
        int newWatchedEvents = listener.onFileDescriptorEvents(
                record.mDescriptor, events);
        if (newWatchedEvents != 0) {
            newWatchedEvents |= OnFileDescriptorEventListener.EVENT_ERROR;
        }

        // Update the file descriptor record if the listener changed the set of
        // events to watch and the listener itself hasn't been updated since.
        if (newWatchedEvents != oldWatchedEvents) {
            synchronized (this) {
                int index = mFileDescriptorRecords.indexOfKey(fd);
                if (index >= 0 && mFileDescriptorRecords.valueAt(index) == record
                        && record.mSeq == seq) {
                    record.mEvents = newWatchedEvents;
                    if (newWatchedEvents == 0) {
                        mFileDescriptorRecords.removeAt(index);
                    }
                }
            }
        }

        // Return the new set of events to watch for native code to take care of.
        return newWatchedEvents;
    }//todo

    Message next() { //我们先看这个方法 此方法在looper.loop中被调用
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // -1 only during first iteration //预备的空闲handler数量
        int nextPollTimeoutMillis = 0; //下一次轮训时间
        for (;;) {//死循环 这个方法只获取一个msg,这个循环的作用是阻塞等待当前要被返回的msg的时间到了
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);//底层同时也进行一次轮训

            synchronized (this) {//尝试取出一个msg并返回
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;//msg头部
                if (msg != null && msg.target == null) {//没有handler来处理它,查找下一个,直到符合条件
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {//找到了可以被处理的msg
                    if (now < msg.when) {//还没到时候呢
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);//可以休眠的时间
                    } else {//现在就可以发送了
                        // Got a message.
                        mBlocked = false;//设置成false 干啥呢
                        //然后从链表中取出这个msg节点
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();//标记这个msg开始被使用啦
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;//没有msg了.....
                }
                
                if (mQuitting) {//需要退出队列
                    dispose();
                    return null;
                }

                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {//获取msg第一次阻塞或者发完了,调用此方法
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;//无事可做,直接循环下一遍
                    continue;
                }

                if (mPendingIdleHandlers == null) {//最少四个
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);//放进数组
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {//运行
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }

    void quit(boolean safe) {
        if (!mQuitAllowed) {//是否允许退出这个循环
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {//已经正在退出了
                return;
            }
            mQuitting = true;//设置标示开始退出

            if (safe) {
                removeAllFutureMessagesLocked();//同步移除msg
            } else {
                removeAllMessagesLocked();//这两个方法有啥区别呢
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);//唤醒底层
        }
    }
    
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {//提交一个同步的障碍 这是个啥?
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();//拿一个message
            msg.markInUse();
            msg.when = when;//设置时间
            msg.arg1 = token;//设置barriertoken

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {//找到适合的插入位置
                    prev = p;
                    p = p.next;
                }
            }
            //把这个barrier插入到适当的位置
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
    
    public void removeSyncBarrier(int token) {//根据token移除同步barrier
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }//查找到符合的barrier
            if (p == null) {//没找到
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;//是否需要唤醒
            if (prev != null) {
                prev.next = p.next;//移除
                needWake = false;//因为还没走到这个,所以不需要唤醒
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;//下一个是空或者不是障碍,需要释放
            }
            p.recycleUnchecked();//回收这个

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);//底层唤醒
            }
        }
    }

    boolean enqueueMessage(Message msg, long when) {//入队
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();//正在被使用
            msg.when = when;//添加事件
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                //新的头部
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();//异步的?
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;//插入合适的位置
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {//唤醒
                nativeWake(mPtr);
            }
        }
        return true;
    }
    
    boolean hasMessages(Handler h, int what, Object object) {//查找
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.what == what && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
    //```省略了上面方法的重载方法 都是查找各种符合条件的第一个```

    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {//移除头部的所有符合的msg
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {//把剩下的全都移除
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
    //```省略了上面方法的重载方法 都是删除各种符合条件的全部```
   
    private void removeAllMessagesLocked() {//直接全部移除 不安全
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }

    private void removeAllFutureMessagesLocked() {
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            if (p.when > now) {//第一个的还没到时间
                removeAllMessagesLocked();//全部移除
            } else {
                Message n;
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
                p.next = null;
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();//移除还没到时间的所有
                } while (n != null);
            }
        }
    }
    
    public static interface IdleHandler {
        boolean queueIdle();
    }
    
    public interface OnFileDescriptorEventListener {
        public static final int EVENT_INPUT = 1 << 0;
        public static final int EVENT_OUTPUT = 1 << 1;
        public static final int EVENT_ERROR = 1 << 2;

        /** @hide */
        @Retention(RetentionPolicy.SOURCE)
        @IntDef(flag = true, prefix = { "EVENT_" }, value = {
                EVENT_INPUT,
                EVENT_OUTPUT,
                EVENT_ERROR
        })
        public @interface Events {}
        @Events int onFileDescriptorEvents(@NonNull FileDescriptor fd, @Events int events);
    }
    
    private static final class FileDescriptorRecord { 
        public final FileDescriptor mDescriptor;
        public int mEvents;
        public OnFileDescriptorEventListener mListener;
        public int mSeq;

        public FileDescriptorRecord(FileDescriptor descriptor,
                int events, OnFileDescriptorEventListener listener) {
            mDescriptor = descriptor;
            mEvents = events;
            mListener = listener;
        }
    }
}
```
java层的代码基本上看完了,可上面还有一些疑问 我们开看看native层的代码都干了啥
```
1. nativeInit
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();//可以看到,在native生成了一个NativeMessageQueue
    if (!nativeMessageQueue) {//报错
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);//引用计数++
    return reinterpret_cast<jlong>(nativeMessageQueue);//把指针转换成jlong并返回给java层
}
下面看看这个NativeMessageQueue的构造函数
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();//从线程获取一个looper
    if (mLooper == NULL) {
        mLooper = new Looper(false);//如果当前线程没有,新建一个
        Looper::setForThread(mLooper);设置给当前线程
    }
}
```
```
2. nativeDestroy
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);//地址long转成指针
    nativeMessageQueue->decStrong(env);//引用--,触发回收
}
```
```
3. nativePollOnce
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);//调用了pollOnce方法
}
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);//调用了looper的pollOnce方法
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```
```
4. nativeWake
void NativeMessageQueue::wake() {
    mLooper->wake();
}
```
```
5. static jboolean android_os_MessageQueue_nativeIsPolling(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->getLooper()->isPolling();
}
```
可以看到上面这几个方法实际都是调用的nativeLooper 
c层先不看了,看不懂=,=哈哈
最后的最后 我们看一下ThreadLocal
##### ThreadLocal 直接上代码
```java
public class ThreadLocal<T> {
    
    private final int threadLocalHashCode = nextHashCode();
    private static AtomicInteger nextHashCode =
        new AtomicInteger();
    private static final int HASH_INCREMENT = 0x61c88647;
    private static int nextHashCode() {//这里可以看出来确保每一个threadLocal对象的hashcode都不同
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    
    protected T initialValue() {
        return null;
    }
    
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }
    
    public ThreadLocal() {//构造函数内并没有做什么事情
    }
    
    //那就直接看Get方法
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//从当前线程中拿出tlmap
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;//取出元素
                return result;
            }
        }
        return setInitialValue();//初始化
    }
    
    private T setInitialValue() {
        T value = initialValue();//先初始化一个T类型的值 默认为null
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//获取本线程的tlmap
        if (map != null)//
            map.set(this, value);//这个对象为key,t为value存进去
        else
            createMap(t, value);//给此线程创建map并把keyvalue存进去
        return value;
    }

    public void set(T value) {//同上
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);//移除本线程存的元素
     }
     
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;//这里可以看到,tlm是存在thread中的对象,所以不同的线程中自然就有不同的对象了
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);//存一个进去
    }
    
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
    
    T childValue(T parentValue) {
        throw new UnsupportedOperationException();
    }

    /**
     * An extension of ThreadLocal that obtains its initial value from
     * the specified {@code Supplier}.
     */
    static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

        private final Supplier<? extends T> supplier;

        SuppliedThreadLocal(Supplier<? extends T> supplier) {
            this.supplier = Objects.requireNonNull(supplier);
        }

        @Override
        protected T initialValue() {
            return supplier.get();
        }
    }
    
    //这就是存在线程中的ThreadLocalMap了 我们看看如何实现的
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> { //这个entry实际上是一个threadlocal的弱引用
            Object value; //使用若引用,当某个threadlocal不再被持有的时候可以被回收掉,省的占着茅坑
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        private static final int INITIAL_CAPACITY = 16;//初始容量
        private Entry[] table;//存储容器 实际是一个数组
        private int size = 0;//大小
        
        private int threshold; //扩容阈值 默认0
        
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {//我们先看看他的构造函数
            table = new Entry[INITIAL_CAPACITY];//初始化数组默认16大小用来存放
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);//根据hashcode的位置进行存放
            table[i] = new Entry(firstKey, firstValue);//放进去
            size = 1;
            setThreshold(INITIAL_CAPACITY);//设置动态扩容的阈值
        }
        private void setThreshold(int len) {//可以看到扩容阈值为2/3 第一次是10 
            threshold = len * 2 / 3;
        }
        //计算下一个索引的位置
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
        //计算上一个索引的位置
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
        //复制 这个方法最后再看
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];//设置大小一样

            for (int j = 0; j < len; j++) {//循环
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);//重新存放
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
        //取出一个
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);//获取他的序号
            Entry e = table[i];
            if (e != null && e.get() == key)//匹配成功,返回
                return e;
            else
                return getEntryAfterMiss(key, i, e);//以为有可能hash冲突,会把后来的放在下一个位置
        }
        
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {//来个循环 以为有可能hash冲突,会把后来的放在下一个位置
                ThreadLocal<?> k = e.get();
                if (k == key)//取出key进行对比
                    return e;
                if (k == null)
                    expungeStaleEntry(i);//清理一下
                else
                    i = nextIndex(i, len);//下一个位置
                e = tab[i];//再进行循环查找
            }
            return null;
        }
        
        private void set(ThreadLocal<?> key, Object value) {//设置
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);//计算hash位置

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) { //从当前位置开始查找直到找到一个空位置
                ThreadLocal<?> k = e.get();

                if (k == key) { //如果找到了符合的key
                    e.value = value; //直接替换value
                    return;
                }

                if (k == null) {//如果key == null,说明旧的已经被回收了,没啥用了
                    replaceStaleEntry(key, value, i);//用此方法替换旧的位置
                    return;
                }
            }

            tab[i] = new Entry(key, value);//找到了空位置,直接放进去
            int sz = ++size;//下一个数放进来的
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        /**
         * Remove the entry for key.
         */
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
        
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len); //前一个
                 (e = tab[i]) != null;//不是空位
                 i = prevIndex(i, len))
                if (e.get() == null)//看看前面还有没有过期的,找到第一个过期的
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == key) {//找到了相同的key
                    e.value = value;//直接设置

                    tab[i] = tab[staleSlot];//拿过去
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
        
        private int expungeStaleEntry(int staleSlot) {//空的,擦除这个位置
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;//把这个位置质控
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);//从当前位置直到一个空位置进行循环
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {//如果又遇到了过期的,置空
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);//重新计算位置
                    if (h != i) {//如果不再当前位置上
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)//从h位置选择一个位置放进去
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;//返回遇到的空位置
        }

        private boolean cleanSomeSlots(int i, int n) {//
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);//从下一个位置开始查找
                Entry e = tab[i];
                if (e != null && e.get() == null) {//这个位置有过期的
                    n = len;//
                    removed = true;//移除
                    i = expungeStaleEntry(i);//从当前位置开始清理
                }
            } while ( (n >>>= 1) != 0);
            return removed;//是否移除掉数据了
        }
        //重新计算hash
        private void rehash() {
            expungeStaleEntries();//先清理旧数据

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4) //需要扩容
                resize();//重新计算大小
        }

        /**
         * Double the capacity of the table.
         */
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;//大小翻倍
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) { //循环旧数据
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC//过期,清空
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);//用新的去计算
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);//放进去
                        newTab[h] = e;
                        count++;//技术++
                    }
                }
            }

            setThreshold(newLen);//设置阈值
            size = count;//大小
            table = newTab;//table
        }

        /**
         * Expunge all stale entries in the table.
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {//全体循环
                Entry e = tab[j];
                if (e != null && e.get() == null)//清理旧数据
                    expungeStaleEntry(j);
            }
        }
    }
}
```
剩下的就先不看了 end~ 