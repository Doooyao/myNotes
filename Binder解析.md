### Binder

#### 手写一个Binder

1. 首先先得有两个进程
```java
唯一方法 在Manifest中添加process属性
android:process=":halou"(app私有)或者android:process="com.example.teststudy.halou"(全局公有)
```
2. 创建接口继承IInterface
```java
//用于声明service的binder需要提供哪些功能
public interface IBookManager extends IInterface {
    static final String DESCRIPTOR = "com.example.teststudy.ipc.IBookManager";//binder的唯一标示
    static final int TRANSACTION_addAndGetBookCount = IBinder.FIRST_CALL_TRANSACTION + 1;//方法标示
    int addAndGetBookCount(int addCount) throws RemoteException;//service要实现的方法
}
```
3. 创建客户端操作的代理
```java
public class BookManagerProxy implements IBookManager{
    private IBinder remote;//持有的远程binder引用
    BookManagerProxy(IBinder iBinder){
        this.remote = iBinder;
    }
    @Override
    public int addAndGetBookCount(int addCount) throws RemoteException {//把参数使用parcel的方式交给远程binder
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();//实例化
        int result;
        try {
            data.writeInterfaceToken(DESCRIPTOR);//写binder标示
            data.writeInt(addCount);//写入要传的参数
            remote.transact(TRANSACTION_addAndGetBookCount,data,reply,0);//调用远程binder处理
            reply.readException();//读取异常情况
            result = reply.readInt();//读取数据
        }finally {
            data.recycle();//回收
            reply.recycle();
        }
        return result;//返回数据
    }

    @Override
    public IBinder asBinder() {//重写asbinder
        return remote;
    }
}
```
4. 服务端Binder
```java
public abstract class BookManager extends Binder implements IBookManager {
    public BookManager(){
        this.attachInterface(this, DESCRIPTOR);//写入binder标示
    }
    public static IBookManager asInterface(IBinder binder) {//使用ibinder生成一个IBookManager
        if (binder == null)
            return null;
        IInterface iin = binder.queryLocalInterface(DESCRIPTOR);//查询本进程是否存在此binder
        if (iin != null && iin instanceof BookManager)//本进程,直接返回,不走transact
            return (BookManager) iin;
        return new BookManagerProxy(binder);//返回代理类,通过Transact调用远程的方法
    }

    @Override
    protected boolean onTransact(int code, @NonNull Parcel data,
                                 @Nullable Parcel reply, int flags) throws RemoteException {//Transact内部会调用此方法
        switch (code){
            case INTERFACE_TRANSACTION:
                reply.writeString(DESCRIPTOR);
                return true;
            case TRANSACTION_addAndGetBookCount://判断方法标示
                data.enforceInterface(DESCRIPTOR);//判断接口匹配
                //此方法将在处理RPC调用时检查客户端和服务器端之间的接口是否相同。
                //细节流程：
                //客户端调用服务器远程方法，在调用之前mRemote.transact，此方法_data.writeInterfaceToken(DESCRIPTOR);将先调用。
                //服务器端onTransact将被调用，然后调用data.enforceInterface(descriptor);以检查接口是否相同，如果它们不匹配，则会抛出错误"Binder invocation to an incorrect interface"。
               int addcount = data.readInt();//从data中读取数据
               int result = this.addAndGetBookCount(addcount);//调用服务端真实方法
               reply.writeNoException();//无异常
               reply.writeInt(result);//写入返回值
               return true;
        }
        return super.onTransact(code, data, reply, flags);
    }
    @Override
    public IBinder asBinder() {//作为bander返回
        return this;
    }
}
```
#### Android中的其他IPC
1. Bundle
2. 文件共享
3. Messenger
服务端:
```java
Messenger messenger = new Messenger(new Handler(Looper.getMainLooper()){//需要一个messenger
        @Override
        public void handleMessage(Message msg) {//处理客户端发来的数据
            Message message = Message.obtain();
            message.arg1 = count+=msg.arg1;
            try {
                msg.replyTo.send(message);//通过这个方式回复客户端
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    });

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();//客户端绑定
    }
```
客户端:
```java
//客户端处理服务端回复数据的messenger
Messenger clientMessenger = new Messenger(new Handler(Looper.getMainLooper()){
        @Override
        public void handleMessage(Message msg) {
            System.out.println(msg.arg1);
        }
    });

serviceMessenger = new Messenger(ibinder);//根据服务端binder获取messenger

//向服务端发送数据
Message message = Message.obtain();
message.arg1 = 6; 
message.replyTo = clientMessenger; //设置回复的messenger及客户端Messenger
try {     
    serviceMessenger.send(message);//发送
} catch (RemoteException e) {     
    e.printStackTrace();
}
```
这个Messenger到底是什么呢,我们看一看源码

```java
public final class Messenger implements Parcelable {
    /**
    * 这个IMessenger是什么呢,我去官网搜了一下,找到了他的Aidl文件,由此可见,应该也是一个Binder
    * oneway interface IMessenger {
    *   void send(in Message msg);//提供一个发送msg的方法
    * }
    */
    private final IMessenger mTarget;

    public Messenger(Handler target) { //这里可以看到从handler中获取的这个binder
        mTarget = target.getIMessenger();
    }
    
    public void send(Message message) throws RemoteException { //调用的内部保存的binder来发送msg
        mTarget.send(message);
    }
    
    public IBinder getBinder() {
        return mTarget.asBinder();//返回持有的binder
    }
    
    public boolean equals(Object otherObj) {
        if (otherObj == null) {
            return false;
        }
        try {
            return mTarget.asBinder().equals(((Messenger)otherObj)//对比持有的binder是否相等
                    .mTarget.asBinder());
        } catch (ClassCastException e) {
        }
        return false;
    }

    public int hashCode() {//重写hashcode
        return mTarget.asBinder().hashCode();
    }
    
    public int describeContents() {
        return 0;
    }
    //序列化部分
    public void writeToParcel(Parcel out, int flags) {
        out.writeStrongBinder(mTarget.asBinder());
    }

    public static final Parcelable.Creator<Messenger> CREATOR
            = new Parcelable.Creator<Messenger>() {
        public Messenger createFromParcel(Parcel in) {
            IBinder target = in.readStrongBinder();
            return target != null ? new Messenger(target) : null;
        }

        public Messenger[] newArray(int size) {
            return new Messenger[size];
        }
    };
    
    //序列化方法
    public static void writeMessengerOrNullToParcel(Messenger messenger,
            Parcel out) {
        out.writeStrongBinder(messenger != null ? messenger.mTarget.asBinder()
                : null);
    }
    
    public static Messenger readMessengerOrNullFromParcel(Parcel in) {
        IBinder b = in.readStrongBinder();
        return b != null ? new Messenger(b) : null;
    }
    
    public Messenger(IBinder target) {//根据IBinder 获取客户端messenger
        mTarget = IMessenger.Stub.asInterface(target);//这里我们可以看到,和之前binder客户端获取一模一样
    }
}
```
可见Messenger是一个持有IMessenger Binder的可序列化对象,无论是sendMessage还是getBinder都是对其内部持有的target操作
客户端也是在获得了ibinder后通过构造方法获取了持有客户端IMessenger的Messenger
下面我们看一下IMessenger和Handler的关系
```java
final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();//这里可以看到,是和上面的binder一样的套路
            return mMessenger;
        }
    }

private final class MessengerImpl extends IMessenger.Stub {     
    public void send(Message msg) {//实现发送msg的方法
        msg.sendingUid = Binder.getCallingUid();//设置Uid
        Handler.this.sendMessage(msg);//发送msg给自己
    }
}
```
关于msg及Handler的其他我们下次看Handler再讲
4. ContentProvider 嘻嘻这个我们下次单独讲
5. socket通信

#### binder 其他细节
##### RemoteCallbackList
```java
public class RemoteCallbackList<E extends IInterface> {//这里我们看到,回调的接口也是要继承IInterface的,
    private static final String TAG = "RemoteCallbackList";
    /*package*/ ArrayMap<IBinder, Callback> mCallbacks
            = new ArrayMap<IBinder, Callback>();
    private Object[] mActiveBroadcast;
    private int mBroadcastCount = -1;
    private boolean mKilled = false;
    private StringBuilder mRecentCallers;

    private final class Callback implements IBinder.DeathRecipient {
        final E mCallback;
        final Object mCookie;

        Callback(E callback, Object cookie) {
            mCallback = callback;
            mCookie = cookie;
        }

        public void binderDied() {
            synchronized (mCallbacks) {//binder死亡的时候移除掉
                mCallbacks.remove(mCallback.asBinder());
            }
            onCallbackDied(mCallback, mCookie);//回调
        }
    }
    
    //这里是注册方法 其他重载的方法我就省略了
    public boolean register(E callback, Object cookie) {
        synchronized (mCallbacks) {
            if (mKilled) { //被杀死
                return false;
            }
            // Flag unusual case that could be caused by a leak. b/36778087
            logExcessiveCallbacks();//
            IBinder binder = callback.asBinder();//先获取ibinder
            try {
                Callback cb = new Callback(callback, cookie);//包装成Callback
                binder.linkToDeath(cb, 0);//添加死亡代理
                mCallbacks.put(binder, cb);//以binder为key存起来
                return true;
            } catch (RemoteException e) {
                return false;
            }
        }
    }

    //取消注册
    public boolean unregister(E callback) {
        synchronized (mCallbacks) {
            Callback cb = mCallbacks.remove(callback.asBinder());//以binder为key,移除
            if (cb != null) {
                cb.mCallback.asBinder().unlinkToDeath(cb, 0);//取消死亡代理
                return true;
            }
            return false;
        }
    }

    //全部清除
    public void kill() {
        synchronized (mCallbacks) {
            for (int cbi=mCallbacks.size()-1; cbi>=0; cbi--) {
                Callback cb = mCallbacks.valueAt(cbi);
                cb.mCallback.asBinder().unlinkToDeath(cb, 0);
            }
            mCallbacks.clear();
            mKilled = true;
        }
    }

    public void onCallbackDied(E callback) {
    }

    public void onCallbackDied(E callback, Object cookie) {
        onCallbackDied(callback);
    }
    
    //开始遍历 推荐使用此方法遍历,线程安全
    public int beginBroadcast() {
        synchronized (mCallbacks) {//加锁
            if (mBroadcastCount > 0) {
                throw new IllegalStateException(
                        "beginBroadcast() called while already in a broadcast");
            }
            
            final int N = mBroadcastCount = mCallbacks.size();//总数
            if (N <= 0) {
                return 0;
            }
            Object[] active = mActiveBroadcast;//正在活动的遍历
            if (active == null || active.length < N) {
                mActiveBroadcast = active = new Object[N];
            }
            for (int i=0; i<N; i++) {
                active[i] = mCallbacks.valueAt(i);//把保存的对象放入数组
            }
            return N;
        }
    }
    
    public E getBroadcastItem(int index) {
        return ((Callback)mActiveBroadcast[index]).mCallback;//获取对象
    }
    
    public Object getBroadcastCookie(int index) {
        return ((Callback)mActiveBroadcast[index]).mCookie;//获取cookie
    }

    //停止操作callback
    public void finishBroadcast() {
        synchronized (mCallbacks) {
            if (mBroadcastCount < 0) {
                throw new IllegalStateException(
                        "finishBroadcast() called outside of a broadcast");
            }

            Object[] active = mActiveBroadcast;
            if (active != null) {
                final int N = mBroadcastCount;
                for (int i=0; i<N; i++) {
                    active[i] = null;//置空数组
                }
            }

            mBroadcastCount = -1;//结束
        }
    }

    //对所有回调进行广播
    public void broadcast(Consumer<E> action) {
        int itemCount = beginBroadcast();
        try {
            for (int i = 0; i < itemCount; i++) {
                action.accept(getBroadcastItem(i));
            }
        } finally {
            finishBroadcast();
        }
    }
    
    public <C> void broadcastForEachCookie(Consumer<C> action) {
        int itemCount = beginBroadcast();
        try {
            for (int i = 0; i < itemCount; i++) {
                action.accept((C) getBroadcastCookie(i));
            }
        } finally {
            finishBroadcast();
        }
    }
    
    //获取数量
    public int getRegisteredCallbackCount() {
        synchronized (mCallbacks) {
            if (mKilled) {
                return 0;
            }
            return mCallbacks.size();
        }
    }
    
    //获取注册回调 非线程安全,如果使用的话需要自己实现线程安全,不推荐
    public E getRegisteredCallbackItem(int index) {
        synchronized (mCallbacks) {
            if (mKilled) {
                return null;
            }
            return mCallbacks.valueAt(index).mCallback;
        }
    }
    
    public Object getRegisteredCallbackCookie(int index) {
        synchronized (mCallbacks) {
            if (mKilled) {
                return null;
            }
            return mCallbacks.valueAt(index).mCookie;
        }
    }
    
    public void dump(PrintWriter pw, String prefix) {
        pw.print(prefix); pw.print("callbacks: "); pw.println(mCallbacks.size());
        pw.print(prefix); pw.print("killed: "); pw.println(mKilled);
        pw.print(prefix); pw.print("broadcasts count: "); pw.println(mBroadcastCount);
    }

    private void logExcessiveCallbacks() {//检测回调是不是太多了,并打印相关信息
        final long size = mCallbacks.size();
        final long TOO_MANY = 3000;
        final long MAX_CHARS = 1000;
        if (size >= TOO_MANY) {
            if (size == TOO_MANY && mRecentCallers == null) {
                mRecentCallers = new StringBuilder();
            }
            if (mRecentCallers != null && mRecentCallers.length() < MAX_CHARS) {
                mRecentCallers.append(Debug.getCallers(5));
                mRecentCallers.append('\n');
                if (mRecentCallers.length() >= MAX_CHARS) {
                    Slog.wtf(TAG, "More than "
                            + TOO_MANY + " remote callbacks registered. Recent callers:\n"
                            + mRecentCallers.toString());
                    mRecentCallers = null;
                }
            }
        }
    }
}
```
##### linkToDeath 死亡代理

linkToDeath（）:为Binder对象设置死亡代理。
unlinkToDeath（）：将设置的死亡代理标志清除。

另一种方式可以在onServiceDisconnected中回调

区别:onServiceDisconnected在UI线程中回调,死亡代理在客户端的Binder线程池中回调

##### Binder线程池 上面提到了binder线程池,这里顺便说一下

##### Binder权限验证
1. 自定义权限并进行验证
2. 在服务端的onTransact方法进行验证,如Uid Pid等

##### Binder连接池
最后,我们来看一下Binder连接池的实现
```java
public class BinderPool {

    private Context context;

    private static volatile BinderPool instance;

    private CountDownLatch mConnectBinderPoolCountDownLatch;

    private IBinderPool iBinderPool;

    private BinderPool(Context context){
        this.context = context.getApplicationContext();
        connectBinderPoolService();
    }

    private static BinderPool getInstance(Context context){//单例
        if (instance==null)
            synchronized (BinderPool.class){
            if (instance==null)
                instance = new BinderPool(context);
            }
        return instance;
    }

    /**
     * 连接远程binder连接池
     */
    private void connectBinderPoolService() {
        mConnectBinderPoolCountDownLatch = new CountDownLatch(1);
        Intent service = new Intent(context,MyService.class);
        context.bindService(service,mBinderPoolConn,Context.BIND_AUTO_CREATE);
        try {
            mConnectBinderPoolCountDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public IBinder queryBinder(int binderCode){
        IBinder iBinder = null;
        if (iBinderPool!=null) {
            try {
                iBinder = iBinderPool.queryBinder(binderCode);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        return iBinder;
    }
    private ServiceConnection mBinderPoolConn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            iBinderPool = IBinderPool.Stub.asInterface(service);//内部持有远程service的binderPool
            try {
                iBinderPool.asBinder().linkToDeath(deathRecipient,0);//死亡监听
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            mConnectBinderPoolCountDownLatch.countDown();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    private IBinder.DeathRecipient deathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            iBinderPool.asBinder().unlinkToDeath(deathRecipient,0);
            iBinderPool = null;
            connectBinderPoolService();//断开连接时重启
        }
    };

    public static class BinderPoolImpl extends IBinderPool.Stub{//这里实现服务端创建ibinder

        @Override
        public IBinder queryBinder(int binderCode) throws RemoteException {
            return null;
        }
    }
}
```

#### 待续... 有其余的以后再补充


