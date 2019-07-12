### OKHttp 解析

##### 1.基本流程

`请求`

```java
//syn 
call.execute();
//asyn
call.enqueue(Callback responseCallback);
```

##### 2.类

```
NamedRunnable=>运行期间按照runnable参数修改运行此runnable的线程名
```

`Dispatcher 分发请求` 

```java
有同步和异步的队列,一个请求运行结束时调用finish

不负责执行同步请求,仅记录
使用内部线程池executorService 运行异步请求

/** 
*	calls 和结束的请求同类型(同步或异步)的running队列
*   promoteCalls 是否进行下一个请求 异步结束true 同步false
*/
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback; //设置的闲置回调
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
  
  
```

`RealInterceptorChain 拦截器链`

```java
 public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

`StreamAllocation`

```

```

`RealConnection`

```java
真正的connection
 public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
      EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");

    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

    if (route.address().sslSocketFactory() == null) {
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    } else {
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        throw new RouteException(new UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
      }
    }

    while (true) {
      try {
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break;
          }
        } else {
          connectSocket(connectTimeout, readTimeout, call, eventListener);
        }
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
        break;
      } catch (IOException e) {
        closeQuietly(socket);
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;
        http2Connection = null;

        eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);

        if (routeException == null) {
          routeException = new RouteException(e);
        } else {
          routeException.addConnectException(e);
        }

        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException;
        }
      }
    }
```

##### 3. 系统拦截器

`基本请求包含的拦截器`

```java
interceptors.add(retryAndFollowUpInterceptor);
interceptors.add(new BridgeInterceptor(client.cookieJar()));
interceptors.add(new CacheInterceptor(client.internalCache()));
interceptors.add(new ConnectInterceptor(client));
if (!forWebSocket) {
   interceptors.addAll(client.networkInterceptors());
}
interceptors.add(new CallServerInterceptor(forWebSocket));
```

`1. RetryAndFollowUpInterceptor `

```java
public Response intercept(Chain chain) throws IOException {
int followUpCount = 0;
Response priorResponse = null;
while (true) {
  if (canceled) {
    streamAllocation.release();
    throw new IOException("Canceled");
  }

  Response response;
  boolean releaseConnection = true;
  try {
    response = realChain.proceed(request, streamAllocation, null, null);
    releaseConnection = false;
  } catch (RouteException e) {
    // The attempt to connect via a route failed. The request will not have been sent.
    if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
      throw e.getFirstConnectException();
    }
    releaseConnection = false;
    continue;
  } catch (IOException e) {
    // An attempt to communicate with a server failed. The request may have been sent.
    boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
    if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
    releaseConnection = false;
    continue;
  } finally {
    // We're throwing an unchecked exception. Release any resources.
    if (releaseConnection) {
      streamAllocation.streamFailed(null);
      streamAllocation.release();
    }
  }

  // Attach the prior response if it exists. Such responses never have a body.
  if (priorResponse != null) {
    response = response.newBuilder()
        .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
        .build();
  }

  Request followUp;
  try {
    followUp = followUpRequest(response, streamAllocation.route());
  } catch (IOException e) {
    streamAllocation.release();
    throw e;
  }

  if (followUp == null) {
    if (!forWebSocket) {
      streamAllocation.release();
    }
    return response;
  }

  closeQuietly(response.body());

  if (++followUpCount > MAX_FOLLOW_UPS) {
    streamAllocation.release();
    throw new ProtocolException("Too many follow-up requests: " + followUpCount);
  }

  if (followUp.body() instanceof UnrepeatableRequestBody) {
    streamAllocation.release();
    throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
  }

  if (!sameConnection(response, followUp.url())) {
    streamAllocation.release();
    streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(followUp.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;
  } else if (streamAllocation.codec() != null) {
    throw new IllegalStateException("Closing the body of " + response
        + " didn't close its backing stream. Bad interceptor?");
  }

  request = followUp;
  priorResponse = response;
}
}
```
```
followUpRequest 重定向方法 todo
```

```java
//恢复方法
private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);

    // The application layer has forbidden retries.
    if (!client.retryOnConnectionFailure()) return false;

    // We can't send the request body again.
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // This exception is fatal.
    if (!isRecoverable(e, requestSendStarted)) return false;

    // No more routes to attempt.
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }
```

`2 BridgeInterceptor`

```
1. 根据原本的请求新建request并填充需要的各种header,cookie等属性,使用此request进行网络请求

2. 从返回的数据中接收cookie,根据返回的response信息构建新的response用于返回上一层处理(okhttp默认没有实现cookie的存取,如果需要要自己实现cookiejar)

3. 如果Content-Encoding == gzip 则解码后返回
```

`3 CacheInterceptor`

```java
/**
* 缓存拦截器
*/
public final class CacheInterceptor implements Interceptor {
  private static final ResponseBody EMPTY_BODY = new ResponseBody() {
    @Override public MediaType contentType() {
      return null;
    }
    @Override public long contentLength() {
      return 0;
    }
    @Override public BufferedSource source() {
      return new Buffer();
    }
  };
  final InternalCache cache;
  public CacheInterceptor(InternalCache cache) {
    this.cache = cache;
  }
    
  /**
  * 拦截方法
  */
  @Override public Response intercept(Chain chain) throws IOException {
      //根据request获取缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    long now = System.currentTimeMillis();
      //获取缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
      //网络请求及缓存response
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    if (cache != null) {
      cache.trackResponse(strategy); //记录缓存命中
    }
      //有缓存,不符合要求
    if (cacheCandidate != null && cacheResponse == null) {
        //关闭缓存流
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }
    // If we're forbidden from using the network and the cache is insufficient, fail.
      //没网没缓存,返回504网关超时
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_BODY)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    Response networkResponse = null;
    try {
      //网络请求数据
        networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (validate(cacheResponse, networkResponse)) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers())) //合并header
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();
        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response); //更新缓存
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    if (HttpHeaders.hasBody(response)) {
      CacheRequest cacheRequest = maybeCache(response, networkResponse.request(), cache);
      response = cacheWritingResponse(cacheRequest, response);
    }
    return response;
  }
```

```java
*******TODO 缓存部分等下再看
```

`4 ConnectInterceptor`

```java
//连接目标服务器并把请求传递给下一拦截器
@Override 
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

`5 CallServerInterceptor`

````java
连接请求数据，下次详细分析//todo
````

##### 4.缓存

`CacheControl`

```
创建或者解析headers中的CacheControl
```

`CacheStrategy`

```
缓存策略 根据请求和缓存的结果中的各项属性判断是否使用缓存
```

`Cache`

```
缓存
```

`DiskLruCache`

```java
1. Entry 缓存入口
private final class Entry { //Entry类，每一个缓存对应一个entry
    final String key; //url相对应

    /** Lengths of this entry's files. */
    final long[] lengths; //todo？？
    final File[] cleanFiles; //写入完成的缓存文件，可读
    final File[] dirtyFiles; //正在写入的缓存

    /** True if this entry has ever been published. */
    boolean readable; //是否可读

    /** The ongoing edit or null if this entry is not being edited. */
    Editor currentEditor;//当前的编辑器，用于编辑本entry的dirtyfile

    /** The sequence number of the most recently committed edit to this entry. */
    long sequenceNumber;//序列号？？todo

    Entry(String key) {
      this.key = key;

      lengths = new long[valueCount];//valuecount：缓存文件的数量，默认为2，一个用来缓存header，一个缓存body
      cleanFiles = new File[valueCount];
      dirtyFiles = new File[valueCount];

      // The names are repetitive so re-use the same builder to avoid allocations.
      StringBuilder fileBuilder = new StringBuilder(key).append('.');//根据key给文件起名
      int truncateTo = fileBuilder.length();
      for (int i = 0; i < valueCount; i++) {
        fileBuilder.append(i);
        cleanFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.append(".tmp");//dirty：临时文件
        dirtyFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.setLength(truncateTo);
      }
    }

    /** Set lengths using decimal numbers like "10123". */
    void setLengths(String[] strings) throws IOException {//设置length ？？todo
      if (strings.length != valueCount) {
        throw invalidLengths(strings);
      }

      try {
        for (int i = 0; i < strings.length; i++) {
          lengths[i] = Long.parseLong(strings[i]);
        }
      } catch (NumberFormatException e) {
        throw invalidLengths(strings);
      }
    }

    /** Append space-prefixed lengths to {@code writer}. */
    void writeLengths(BufferedSink writer) throws IOException {//拼接长度 todo
      for (long length : lengths) {
        writer.writeByte(' ').writeDecimalLong(length);
      }
    }

    private IOException invalidLengths(String[] strings) throws IOException {
      throw new IOException("unexpected journal line: " + Arrays.toString(strings));
    }

    Snapshot snapshot() { //获取缓存的快照
      if (!Thread.holdsLock(DiskLruCache.this)) throw new AssertionError();

      Source[] sources = new Source[valueCount];
      long[] lengths = this.lengths.clone(); // Defensive copy since these can be zeroed out.
      try {
        for (int i = 0; i < valueCount; i++) {
          sources[i] = fileSystem.source(cleanFiles[i]);//必须是已经写入完成的文件
        }
        return new Snapshot(key, sequenceNumber, sources, lengths);
      } catch (FileNotFoundException e) {//文件缺失
        // A file must have been deleted manually!
        for (int i = 0; i < valueCount; i++) {
          if (sources[i] != null) {
            Util.closeQuietly(sources[i]);//关闭已经打开的文件
          } else {
            break;
          }
        }
        try {
          removeEntry(this);//删除本缓存
        } catch (IOException ignored) {
        }
        return null;
      }
    }
  }
```

```java
2.Editor 缓存编辑器
public final class Editor {
    final Entry entry; //要编辑的缓存 一一对应
    final boolean[] written; //是否已经写入
    private boolean done;//编辑完毕

    Editor(Entry entry) { //构造方法
      this.entry = entry;
      this.written = (entry.readable) ? null : new boolean[valueCount];
      //如果可读，证明写入完毕了，written无意义 设null
    }

//解除绑定，用于移除缓存或者报错
    void detach() {
      if (entry.currentEditor == this) {//判断缓存的当前编辑器是否是自己
        for (int i = 0; i < valueCount; i++) {
          try {
            fileSystem.delete(entry.dirtyFiles[i]);//删除所有正在编辑的dirty文件
          } catch (IOException e) {
            // This file is potentially leaked. Not much we can do about that.
          }
        }
        entry.currentEditor = null;//缓存编辑器置空
      }
    }

    //获取缓存的第index个缓存文件输入流用于读取文件 比如0就是缓存head的文件，1是缓存body的
    public Source newSource(int index) {
      synchronized (DiskLruCache.this) {
        if (done) {//状态为done代表编辑器结束了
          throw new IllegalStateException();
        }
        if (!entry.readable || entry.currentEditor != this) {//不可读或者不匹配
          return null;
        }
        try {
          return fileSystem.source(entry.cleanFiles[index]);//获取clean文件的输入流读取clean文件
        } catch (FileNotFoundException e) {
          return null;
        }
      }
    }

    //获取输出流用于向dirty文件中写入缓存数据
    public Sink newSink(int index) {
      synchronized (DiskLruCache.this) {
        if (done) {
          throw new IllegalStateException();
        }
        if (entry.currentEditor != this) {
          return Okio.blackhole();//黑洞流，不会被写入到任何地方
        }
        if (!entry.readable) {//？？todo
          written[index] = true;
        }
        File dirtyFile = entry.dirtyFiles[index];
        Sink sink;
        try {
          sink = fileSystem.sink(dirtyFile);
        } catch (FileNotFoundException e) {
          return Okio.blackhole();
        }
        return new FaultHidingSink(sink) {//自己捕获异常的sink包装
          @Override protected void onException(IOException e) {
            synchronized (DiskLruCache.this) {
              detach();//异常，解除当前绑定并删除正在写的dirty文件
            }
          }
        };
      }
    }

	//提交正在写的信息
    public void commit() throws IOException {
      synchronized (DiskLruCache.this) {
        if (done) {
          throw new IllegalStateException();
        }
        if (entry.currentEditor == this) {
          completeEdit(this, true);
        }
        done = true;//提交完毕
      }
    }

    //终止，提交正在写的
    public void abort() throws IOException {
      synchronized (DiskLruCache.this) {
        if (done) {
          throw new IllegalStateException();
        }
        if (entry.currentEditor == this) {
          completeEdit(this, false);
        }
        done = true;
      }
    }
	
	//除非正在编辑，否则终止
    public void abortUnlessCommitted() {
      synchronized (DiskLruCache.this) {
        if (!done && entry.currentEditor == this) {
          try {
            completeEdit(this, false);
          } catch (IOException ignored) {
          }
        }
      }
    }
  }
```

```java
3。Snapshot 快照
public final class Snapshot implements Closeable {
    private final String key;//url匹配的key
    private final long sequenceNumber; //序列号？？todo
    private final Source[] sources;//source数组
    private final long[] lengths;//

    Snapshot(String key, long sequenceNumber, Source[] sources, long[] lengths) {
      this.key = key;
      this.sequenceNumber = sequenceNumber;
      this.sources = sources;
      this.lengths = lengths;
    }

    public String key() {
      return key;
    }

    /**
     * Returns an editor for this snapshot's entry, or null if either the entry has changed since
     * this snapshot was created or if another edit is in progress.
     */
    public @Nullable Editor edit() throws IOException {
      return DiskLruCache.this.edit(key, sequenceNumber);
    }

    /** Returns the unbuffered stream with the value for {@code index}. */
    public Source getSource(int index) {
      return sources[index];
    }

    /** Returns the byte length of the value for {@code index}. */
    public long getLength(int index) {
      return lengths[index];
    }

    public void close() {
      for (Source in : sources) {
        Util.closeQuietly(in);
      }
    }
  }
```

```java
4.DiskLruCache！！！缓存类本类～～～ 上面都是它的内部类
public final class DiskLruCache implements Closeable, Flushable {
  static final String JOURNAL_FILE = "journal"; //日志文件 （用于读）
  static final String JOURNAL_FILE_TEMP = "journal.tmp"; //临时文件日志文件（用于写）
  static final String JOURNAL_FILE_BACKUP = "journal.bkp"; //日志文件备份
  static final String MAGIC = "libcore.io.DiskLruCache"; //文件头字段
  static final String VERSION_1 = "1"; //版本
  static final long ANY_SEQUENCE_NUMBER = -1 //默认序列号
  static final Pattern LEGAL_KEY_PATTERN = Pattern.compile("[a-z0-9_-]{1,120}");//合法的key的正则
  private static final String CLEAN = "CLEAN";//clean标识，代表编辑完毕
  private static final String DIRTY = "DIRTY";//dirty标识，代表编辑中
  private static final String REMOVE = "REMOVE";//remove表示，删除动作
  private static final String READ = "READ";//read标识 todo

    /*
     *     日志文件类似如下内容
     *     libcore.io.DiskLruCache
     *     1
     *     100
     *     2
     *
     *     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
     *     DIRTY 335c4c6028171cfddfbaae1a9c313c52
     *     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
     *     REMOVE 335c4c6028171cfddfbaae1a9c313c52
     *     DIRTY 1ab96a171faeeee38496d8b330771a7a
     *     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
     *     READ 335c4c6028171cfddfbaae1a9c313c52
     *     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
     */

  final FileSystem fileSystem; //文件系统帮助类
  final File directory; //父文件夹
  private final File journalFile;
  private final File journalFileTmp;
  private final File journalFileBackup;
  private final int appVersion;
  private long maxSize;
  final int valueCount;//一个缓存条目有几个缓存文件，默认两个，一个缓存head，一个缓存body
  private long size = 0;
  BufferedSink journalWriter;
  final LinkedHashMap<String, Entry> lruEntries = new LinkedHashMap<>(0, 0.75f, true);//lru策略的hashmap 
  int redundantOpCount; //多余的行数todo
  boolean hasJournalErrors;//是否有日志错误

  // Must be read and written when synchronized on 'this'.
  boolean initialized; //是否初始化过
  boolean closed;
  boolean mostRecentTrimFailed;
  boolean mostRecentRebuildFailed;

  /**
   * To differentiate between old and current snapshots, each entry is given a sequence number each
   * time an edit is committed. A snapshot is stale if its sequence number is not equal to its
   * entry's sequence number.
   */
  private long nextSequenceNumber = 0;//下一个序列号

  private final Executor executor;//线程池，用于执行cleanupRunnable
  private final Runnable cleanupRunnable = new Runnable() { //清理线程
    public void run() {
      synchronized (DiskLruCache.this) {
        if (!initialized | closed) {
          return; // Nothing to do
        }

        try {
          trimToSize();//清理空间
        } catch (IOException ignored) {
          mostRecentTrimFailed = true;//最近一次清理失败
        }

        try {
          if (journalRebuildRequired()) {//是否需要重新构建日至文件？
            rebuildJournal();//重新构建
            redundantOpCount = 0;//多余行数归零
          }
        } catch (IOException e) {
          mostRecentRebuildFailed = true;//最近一次构建失败
          journalWriter = Okio.buffer(Okio.blackhole());//日至文件输出流使用黑洞，不再记录日至
        }
      }
    }
  };
	//构造方法 不公开
  DiskLruCache(FileSystem fileSystem, File directory, int appVersion, int valueCount, long maxSize,
      Executor executor) {
    this.fileSystem = fileSystem;
    this.directory = directory;
    this.appVersion = appVersion;
    this.journalFile = new File(directory, JOURNAL_FILE);
    this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
    this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
    this.valueCount = valueCount;
    this.maxSize = maxSize;
    this.executor = executor;
  }
	
	//初始化
  public synchronized void initialize() throws IOException {
    assert Thread.holdsLock(this); //断言，当持有自己锁的时候。继续执行，没有持有锁，直接抛异常

    if (initialized) {
      return; // Already initialized.
    }

    // If a bkp file exists, use it instead.没有日志文件时使用备份文件，不然就删除备份
    if (fileSystem.exists(journalFileBackup)) {
      if (fileSystem.exists(journalFile)) {
        fileSystem.delete(journalFileBackup);
      } else {
        fileSystem.rename(journalFileBackup, journalFile);
      }
    }

    // 存在日志文件
    if (fileSystem.exists(journalFile)) {
      try {
        readJournal();//读取日志
        processJournal();//提炼日志
        initialized = true;//已经初始化
        return;
      } catch (IOException journalIsCorrupt) {
        Platform.get().log(WARN, "DiskLruCache " + directory + " is corrupt: "
            + journalIsCorrupt.getMessage() + ", removing", journalIsCorrupt);
      }
      try {
        delete();//读取失败，删除日志
      } finally {
        closed = false;
      }
    }

    rebuildJournal();//重建日志

    initialized = true;
  }

	//公开的构建一个DiskLruCache的方法
  public static DiskLruCache create(FileSystem fileSystem, File directory, int appVersion,
      int valueCount, long maxSize) {
    if (maxSize <= 0) {
      throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
      throw new IllegalArgumentException("valueCount <= 0");
    }

    // Use a single background thread to evict entries.
    Executor executor = new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<Runnable>(), Util.threadFactory("OkHttp DiskLruCache", true));

    return new DiskLruCache(fileSystem, directory, appVersion, valueCount, maxSize, executor);
  }
	//读取日志文件
  private void readJournal() throws IOException {
    BufferedSource source = Okio.buffer(fileSystem.source(journalFile));
    try {//判断文件头部信息
      String magic = source.readUtf8LineStrict();
      String version = source.readUtf8LineStrict();
      String appVersionString = source.readUtf8LineStrict();
      String valueCountString = source.readUtf8LineStrict();
      String blank = source.readUtf8LineStrict();
      if (!MAGIC.equals(magic)
          || !VERSION_1.equals(version)
          || !Integer.toString(appVersion).equals(appVersionString)
          || !Integer.toString(valueCount).equals(valueCountString)
          || !"".equals(blank)) {
        throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
            + valueCountString + ", " + blank + "]");
      }
		//逐行读取
      int lineCount = 0;
      while (true) {
        try {
          readJournalLine(source.readUtf8LineStrict());//读取一行
          lineCount++;//行数++
        } catch (EOFException endOfJournal) {
          break;
        }
      }
      redundantOpCount = lineCount - lruEntries.size();//多余的行数

      // If we ended on a truncated line, rebuild the journal before appending to it.
      if (!source.exhausted()) {//source还有剩余字节没读
        rebuildJournal();//重新构建日志
      } else {
        journalWriter = newJournalWriter();//获取日志文件的输出流
      }
    } finally {
      Util.closeQuietly(source);//关闭输入流
    }
  }
		//获取日志文件的拼接输出流，包装成捕获异常的FaultHidingSink
  private BufferedSink newJournalWriter() throws FileNotFoundException {
    Sink fileSink = fileSystem.appendingSink(journalFile);
    Sink faultHidingSink = new FaultHidingSink(fileSink) {
      @Override protected void onException(IOException e) {
        assert (Thread.holdsLock(DiskLruCache.this));
        hasJournalErrors = true;
      }
    };
    return Okio.buffer(faultHidingSink);
  }
	//读取并处理一行日志
  private void readJournalLine(String line) throws IOException {
    int firstSpace = line.indexOf(' ');//每行空格开始
    if (firstSpace == -1) {
      throw new IOException("unexpected journal line: " + line);
    }

    int keyBegin = firstSpace + 1;//key start
    int secondSpace = line.indexOf(' ', keyBegin);//key end
    final String key;
    if (secondSpace == -1) {
      key = line.substring(keyBegin);//获取key
      if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {//执行日志的remove key动作
        lruEntries.remove(key);
        return;
      }
    } else {
      key = line.substring(keyBegin, secondSpace);//获取key
    }

    Entry entry = lruEntries.get(key);//获取本条日志操作的对应缓存
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);//如果没有，先添加一个空的
    }

    if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {//干净的（已经写入完毕的缓存）
      String[] parts = line.substring(secondSpace + 1).split(" ");
      entry.readable = true;//设置可读
      entry.currentEditor = null;
      entry.setLengths(parts);//设置旧缓存文件长度
    } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {//dirty（正在写的）
      entry.currentEditor = new Editor(entry);//给缓存添加一个新的编辑器
    } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {//读
      // This work was already done by calling lruEntries.get().
    } else {
      throw new IOException("unexpected journal line: " + line);
    }
  }

  /**
   * 计算初始大小并收集垃圾作为打开缓存的一部分。 假设脏条目不一致，将被删除。（谷歌翻译来的。。） 处理缓存
   */
  private void processJournal() throws IOException {
    fileSystem.delete(journalFileTmp);//删除临时文件
    for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
      Entry entry = i.next();
      if (entry.currentEditor == null) { //计算文件大小
        for (int t = 0; t < valueCount; t++) {
          size += entry.lengths[t];
        }
      } else {
        entry.currentEditor = null;
        for (int t = 0; t < valueCount; t++) {//删除dirty缓存及其clean文件
          fileSystem.delete(entry.cleanFiles[t]);
          fileSystem.delete(entry.dirtyFiles[t]);
        }
        i.remove();
      }
    }
  }

  /**
   * 重建日志文件
   */
  synchronized void rebuildJournal() throws IOException {
    if (journalWriter != null) {//关闭日志输出流
      journalWriter.close();
    }

    BufferedSink writer = Okio.buffer(fileSystem.sink(journalFileTmp));//创建对临时文件的输出流
    try {
    //写入规定的文件头
      writer.writeUtf8(MAGIC).writeByte('\n');
      writer.writeUtf8(VERSION_1).writeByte('\n');
      writer.writeDecimalLong(appVersion).writeByte('\n');
      writer.writeDecimalLong(valueCount).writeByte('\n');
      writer.writeByte('\n');
	//遍历目前内存中的缓存目录
      for (Entry entry : lruEntries.values()) {	
        if (entry.currentEditor != null) {//正在编辑中，则写入dirty
          writer.writeUtf8(DIRTY).writeByte(' ');
          writer.writeUtf8(entry.key);
          writer.writeByte('\n');
        } else {//不在编辑中，写入clean
          writer.writeUtf8(CLEAN).writeByte(' ');
          writer.writeUtf8(entry.key);
          entry.writeLengths(writer);
          writer.writeByte('\n');
        }
      }
    } finally {
      writer.close();//关闭输出流
    }
	//替换文件
    if (fileSystem.exists(journalFile)) {
      fileSystem.rename(journalFile, journalFileBackup);
    }
    fileSystem.rename(journalFileTmp, journalFile);
    fileSystem.delete(journalFileBackup);

    journalWriter = newJournalWriter();//获取新的对日志文件的输出流
    hasJournalErrors = false;
    mostRecentRebuildFailed = false;
  }

  /**
   * Returns a snapshot of the entry named {@code key}, or null if it doesn't exist is not currently
   * readable. If a value is returned, it is moved to the head of the LRU queue. 获取缓存快照
   */
  public synchronized Snapshot get(String key) throws IOException {
    initialize();

    checkNotClosed();//检查是否关闭
    validateKey(key);//检查key是否合法
    Entry entry = lruEntries.get(key);
    if (entry == null || !entry.readable) return null;

    Snapshot snapshot = entry.snapshot();//生成快照
    if (snapshot == null) return null;

    redundantOpCount++;//多余的日志+1
    journalWriter.writeUtf8(READ).writeByte(' ').writeUtf8(key).writeByte('\n');//写入读操作日志
    if (journalRebuildRequired()) {//是否需要重新创建日志
      executor.execute(cleanupRunnable);//cleanup
    }

    return snapshot;
  }

  /**
   * Returns an editor for the entry named {@code key}, or null if another edit is in progress.
   */
  public @Nullable Editor edit(String key) throws IOException {
    return edit(key, ANY_SEQUENCE_NUMBER);
  }
	//获取对应key的edit
  synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
    initialize();

    checkNotClosed();
    validateKey(key);
    Entry entry = lruEntries.get(key);
    //？？？todo
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
        || entry.sequenceNumber != expectedSequenceNumber)) {
      return null; // Snapshot is stale.
    }
    if (entry != null && entry.currentEditor != null) {
      return null; // Another edit is in progress.
    }
    if (mostRecentTrimFailed || mostRecentRebuildFailed) {
     //这种状态下存的太多了，清理，不让写入了
      executor.execute(cleanupRunnable);
      return null;
    }

    // Flush the journal before creating files to prevent file leaks.
    //添加dirty日志
    journalWriter.writeUtf8(DIRTY).writeByte(' ').writeUtf8(key).writeByte('\n');
    journalWriter.flush();

    if (hasJournalErrors) {
      return null; // Don't edit; the journal can't be written.
    }
	//新建缓存 添加编辑器
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);
    }
    Editor editor = new Editor(entry);
    entry.currentEditor = editor;
    return editor;//返回编辑器
  }

  /** Returns the directory where this cache stores its data. */
  public File getDirectory() {
    return directory;
  }

  /**
   * Returns the maximum number of bytes that this cache should use to store its data.
   */
  public synchronized long getMaxSize() {
    return maxSize;
  }

  /**
   * Changes the maximum number of bytes the cache can store and queues a job to trim the existing
   * store, if necessary.
   */
  public synchronized void setMaxSize(long maxSize) {
    this.maxSize = maxSize;
    if (initialized) {
      executor.execute(cleanupRunnable);
    }
  }

  /**
   * Returns the number of bytes currently being used to store the values in this cache. This may be
   * greater than the max size if a background deletion is pending.
   */
  public synchronized long size() throws IOException {
    initialize();
    return size;
  }
//编辑完成
  synchronized void completeEdit(Editor editor, boolean success) throws IOException {
    Entry entry = editor.entry;
    if (entry.currentEditor != editor) {
      throw new IllegalStateException();
    }

    // If this edit is creating the entry for the first time, every index must have a value.
    if (success && !entry.readable) {//编辑完成，之前不可读
      for (int i = 0; i < valueCount; i++) {
        if (!editor.written[i]) { //没有被写
          editor.abort(); //终止
          throw new IllegalStateException("Newly created entry didn't create value for index " + i);
        }
        if (!fileSystem.exists(entry.dirtyFiles[i])) { //写文件不存在
          editor.abort();//终止
          return;
        }
      }
    }

    for (int i = 0; i < valueCount; i++) {
      File dirty = entry.dirtyFiles[i];
      if (success) {
        if (fileSystem.exists(dirty)) {//把dirty文件改为clean文件
          File clean = entry.cleanFiles[i];
          fileSystem.rename(dirty, clean);
          long oldLength = entry.lengths[i];//更新长度
          long newLength = fileSystem.size(clean);
          entry.lengths[i] = newLength;
          size = size - oldLength + newLength;//更新缓存的总size
        }
      } else {
        fileSystem.delete(dirty);//失败，删除dirty
      }
    }

    redundantOpCount++;//多余记录行+1（因为上一条记录一定是获取编辑器时写入的dirty操作，目前写完了那条日志就没用了）
    entry.currentEditor = null;//置空编辑器
    if (entry.readable | success) {//成功或之前就是可读的
      entry.readable = true;//可读
      journalWriter.writeUtf8(CLEAN).writeByte(' ');//写入clean日志
      journalWriter.writeUtf8(entry.key);
      entry.writeLengths(journalWriter);//写入缓存文件长度
      journalWriter.writeByte('\n');
      if (success) {
        entry.sequenceNumber = nextSequenceNumber++;//序列号+1	
      }
    } else {
      lruEntries.remove(entry.key);//之前不可读（及没有有效缓存）而且写入失败
      journalWriter.writeUtf8(REMOVE).writeByte(' ');//删除操作
      journalWriter.writeUtf8(entry.key);
      journalWriter.writeByte('\n');
    }
    journalWriter.flush();

    if (size > maxSize || journalRebuildRequired()) {//需要重构日志文件或者缓存大小超出限制
      executor.execute(cleanupRunnable);//cleanup
    }
  }

  /**
   * 是否需要重构日志
   */
  boolean journalRebuildRequired() {
    final int redundantOpCompactThreshold = 2000;//多余的日志超出2000并且超出缓存数
    return redundantOpCount >= redundantOpCompactThreshold
        && redundantOpCount >= lruEntries.size();
  }

  public synchronized boolean remove(String key) throws IOException {
    initialize();

    checkNotClosed();
    validateKey(key);
    Entry entry = lruEntries.get(key);
    if (entry == null) return false;
    boolean removed = removeEntry(entry);
    if (removed && size <= maxSize) mostRecentTrimFailed = false;
    return removed;
  }

  boolean removeEntry(Entry entry) throws IOException {
    if (entry.currentEditor != null) {
      entry.currentEditor.detach(); // Prevent the edit from completing normally.
    }

    for (int i = 0; i < valueCount; i++) {//删除文件
      fileSystem.delete(entry.cleanFiles[i]);
      size -= entry.lengths[i];
      entry.lengths[i] = 0;
    }

    redundantOpCount++;
    //记录remove操作
    journalWriter.writeUtf8(REMOVE).writeByte(' ').writeUtf8(entry.key).writeByte('\n');
    lruEntries.remove(entry.key);

    if (journalRebuildRequired()) {
      executor.execute(cleanupRunnable);
    }

    return true;
  }

  /** Returns true if this cache has been closed. */
  public synchronized boolean isClosed() {
    return closed;
  }

  private synchronized void checkNotClosed() {
    if (isClosed()) {
      throw new IllegalStateException("cache is closed");
    }
  }

  /** Force buffered operations to the filesystem. */
  @Override public synchronized void flush() throws IOException {
    if (!initialized) return;

    checkNotClosed();
    trimToSize();
    journalWriter.flush();
  }

  /** Closes this cache. Stored values will remain on the filesystem. */
  @Override public synchronized void close() throws IOException {
    if (!initialized || closed) {
      closed = true;
      return;
    }
    // Copying for safe iteration.
    for (Entry entry : lruEntries.values().toArray(new Entry[lruEntries.size()])) {
      if (entry.currentEditor != null) {
        entry.currentEditor.abort();//终止所有正在写入的
      }
    }
    trimToSize();//修整缓存空间
    journalWriter.close();//关闭流
    journalWriter = null;
    closed = true;
  }

  void trimToSize() throws IOException {//移除超出大小的缓存
    while (size > maxSize) {
      Entry toEvict = lruEntries.values().iterator().next();
      removeEntry(toEvict);
    }
    mostRecentTrimFailed = false;
  }

  /**
   * Closes the cache and deletes all of its stored values. This will delete all files in the cache
   * directory including files that weren't created by the cache.
   */
  public void delete() throws IOException {//整个删除（包括日志文件）
    close();
    fileSystem.deleteContents(directory);
  }

  /**
   * Deletes all stored values from the cache. In-flight edits will complete normally but their
   * values will not be stored.
   */
  public synchronized void evictAll() throws IOException {//清空
    initialize();
    // Copying for safe iteration.
    for (Entry entry : lruEntries.values().toArray(new Entry[lruEntries.size()])) {
      removeEntry(entry);
    }
    mostRecentTrimFailed = false;
  }

  private void validateKey(String key) {//验证key的格式合法
    Matcher matcher = LEGAL_KEY_PATTERN.matcher(key);
    if (!matcher.matches()) {
      throw new IllegalArgumentException(
          "keys must match regex [a-z0-9_-]{1,120}: \"" + key + "\"");
    }
  }

  /**
   * 快照迭代器
   */
  public synchronized Iterator<Snapshot> snapshots() throws IOException {
    initialize();
    return new Iterator<Snapshot>() {
      /** Iterate a copy of the entries to defend against concurrent modification errors. */
      final Iterator<Entry> delegate = new ArrayList<>(lruEntries.values()).iterator();

      /** The snapshot to return from {@link #next}. Null if we haven't computed that yet. */
      Snapshot nextSnapshot;

      /** The snapshot to remove with {@link #remove}. Null if removal is illegal. */
      Snapshot removeSnapshot;

      @Override public boolean hasNext() {
        if (nextSnapshot != null) return true;

        synchronized (DiskLruCache.this) {
          // If the cache is closed, truncate the iterator.
          if (closed) return false;

          while (delegate.hasNext()) {
            Entry entry = delegate.next();
            Snapshot snapshot = entry.snapshot();
            if (snapshot == null) continue; // Evicted since we copied the entries.
            nextSnapshot = snapshot;
            return true;
          }
        }

        return false;
      }

      @Override public Snapshot next() {
        if (!hasNext()) throw new NoSuchElementException();
        removeSnapshot = nextSnapshot;
        nextSnapshot = null;
        return removeSnapshot;
      }

      @Override public void remove() {
        if (removeSnapshot == null) throw new IllegalStateException("remove() before next()");
        try {
          DiskLruCache.this.remove(removeSnapshot.key);
        } catch (IOException ignored) {
          // Nothing useful to do here. We failed to remove from the cache. Most likely that's
          // because we couldn't update the journal, but the cached entry will still be gone.
        } finally {
          removeSnapshot = null;
        }
      }
    };
  }
  
  
                          ```各种内部类，上文已经展示了，这里省略```
                         
}

```

##### 5 网络！～～

终于到了激动人心的最后一部分！～～

`ConnectionSpec`

```java
是否与提供的sslsocket兼容
public boolean isCompatible(SSLSocket socket) {
    if (!tls) {//不是tls 不兼容
      return false;
    }

    if (tlsVersions != null && !nonEmptyIntersection( //两者的tls版本数组是否有一个相同的
        Util.NATURAL_ORDER, tlsVersions, socket.getEnabledProtocols())) {
      return false;
    }

    if (cipherSuites != null && !nonEmptyIntersection(//两者的密钥算法套件是否有一个相同的
        CipherSuite.ORDER_BY_NAME, cipherSuites, socket.getEnabledCipherSuites())) {
      return false;
    }

    return true;
  }
```

```java
 /** Applies this spec to {@code sslSocket}. */
  void apply(SSLSocket sslSocket, boolean isFallback) {
    ConnectionSpec specToApply = supportedSpec(sslSocket, isFallback);

    if (specToApply.tlsVersions != null) {//把并集设置给
      sslSocket.setEnabledProtocols(specToApply.tlsVersions);
    }
    if (specToApply.cipherSuites != null) {
      sslSocket.setEnabledCipherSuites(specToApply.cipherSuites);
    }
  }

  /**
   * Returns a copy of this that omits cipher suites and TLS versions not enabled by {@code
   * sslSocket}.
   */
  private ConnectionSpec supportedSpec(SSLSocket sslSocket, boolean isFallback) {
    String[] cipherSuitesIntersection = cipherSuites != null
        ? intersect(CipherSuite.ORDER_BY_NAME, sslSocket.getEnabledCipherSuites(), cipherSuites)
        : sslSocket.getEnabledCipherSuites();//求密钥算法套件并集
    String[] tlsVersionsIntersection = tlsVersions != null
        ? intersect(Util.NATURAL_ORDER, sslSocket.getEnabledProtocols(), tlsVersions)
        : sslSocket.getEnabledProtocols();//求版本并集

    String[] supportedCipherSuites = sslSocket.getSupportedCipherSuites();//sslsocket支持的密钥算法
    int indexOfFallbackScsv = indexOf(
        CipherSuite.ORDER_BY_NAME, supportedCipherSuites, "TLS_FALLBACK_SCSV");
    if (isFallback && indexOfFallbackScsv != -1) {//需要的情况，添加对TLS_FALLBACK_SCSV的支持（todo 我不知道这个套件时干啥的=。=TLS_FALLBACK_SCSV）
      cipherSuitesIntersection = concat(
          cipherSuitesIntersection, supportedCipherSuites[indexOfFallbackScsv]);
    }

    return new Builder(this)//构建一个新的配置并返回
        .cipherSuites(cipherSuitesIntersection)
        .tlsVersions(tlsVersionsIntersection)
        .build();
  }
```

`ConnectionSpecSelector`

```java
链接规格选择器

/**
 * Handles the connection spec fallback strategy: When a secure socket connection fails due to a
 * handshake / protocol problem the connection may be retried with different protocols. Instances
 * are stateful and should be created and used for a single connection attempt.
 */
public final class ConnectionSpecSelector {

  private final List<ConnectionSpec> connectionSpecs;
  private int nextModeIndex;
  private boolean isFallbackPossible;
  private boolean isFallback;

  public ConnectionSpecSelector(List<ConnectionSpec> connectionSpecs) {
    this.nextModeIndex = 0;
    this.connectionSpecs = connectionSpecs;
  }

  /**
   * Configures the supplied {@link SSLSocket} to connect to the specified host using an appropriate
   * {@link ConnectionSpec}. Returns the chosen {@link ConnectionSpec}, never {@code null}.
   *
   * @throws IOException if the socket does not support any of the TLS modes available
   1 从OkHttp配置的ConnectionSpec集合中选择一个SSLSocket兼容的一个。
	2 将选择的ConnectionSpec应用到SSLSocket上。
   */
  public ConnectionSpec configureSecureSocket(SSLSocket sslSocket) throws IOException {
    ConnectionSpec tlsConfiguration = null;
    for (int i = nextModeIndex, size = connectionSpecs.size(); i < size; i++) {
      ConnectionSpec connectionSpec = connectionSpecs.get(i);
      if (connectionSpec.isCompatible(sslSocket)) {
        tlsConfiguration = connectionSpec;
        nextModeIndex = i + 1;
        break;
      }
    }

    if (tlsConfiguration == null) {
      // This may be the first time a connection has been attempted and the socket does not support
      // any the required protocols, or it may be a retry (but this socket supports fewer
      // protocols than was suggested by a prior socket).
      throw new UnknownServiceException(
          "Unable to find acceptable protocols. isFallback=" + isFallback
              + ", modes=" + connectionSpecs
              + ", supported protocols=" + Arrays.toString(sslSocket.getEnabledProtocols()));
    }

    isFallbackPossible = isFallbackPossible(sslSocket);

    Internal.instance.apply(tlsConfiguration, sslSocket, isFallback);

    return tlsConfiguration;
  }

  /**
   * Reports a failure to complete a connection. Determines the next {@link ConnectionSpec} to try,
   * if any.
   *
   * @return {@code true} if the connection should be retried using {@link
   * #configureSecureSocket(SSLSocket)} or {@code false} if not
   */
  public boolean connectionFailed(IOException e) {
    // Any future attempt to connect using this strategy will be a fallback attempt.
    isFallback = true;

    if (!isFallbackPossible) {
      return false;
    }

    // If there was a protocol problem, don't recover.
    if (e instanceof ProtocolException) {
      return false;
    }

    // If there was an interruption or timeout (SocketTimeoutException), don't recover.
    // For the socket connect timeout case we do not try the same host with a different
    // ConnectionSpec: we assume it is unreachable.
    if (e instanceof InterruptedIOException) {
      return false;
    }

    // Look for known client-side or negotiation errors that are unlikely to be fixed by trying
    // again with a different connection spec.
    if (e instanceof SSLHandshakeException) {
      // If the problem was a CertificateException from the X509TrustManager,
      // do not retry.
      if (e.getCause() instanceof CertificateException) {
        return false;
      }
    }
    if (e instanceof SSLPeerUnverifiedException) {
      // e.g. a certificate pinning error.
      return false;
    }

    // On Android, SSLProtocolExceptions can be caused by TLS_FALLBACK_SCSV failures, which means we
    // retry those when we probably should not.
    return (e instanceof SSLHandshakeException || e instanceof SSLProtocolException);
  }

  /**
   * Returns {@code true} if any later {@link ConnectionSpec} in the fallback strategy looks
   * possible based on the supplied {@link SSLSocket}. It assumes that a future socket will have the
   * same capabilities as the supplied socket.
   */
  private boolean isFallbackPossible(SSLSocket socket) {
    for (int i = nextModeIndex; i < connectionSpecs.size(); i++) {
      if (connectionSpecs.get(i).isCompatible(socket)) {
        return true;
      }
    }
    return false;
  }
}

```

###### HttpCodec

```java
public interface HttpCodec { //解码器接口
  /**
   * The timeout to use while discarding a stream of input data. Since this is used for connection
   * reuse, this timeout should be significantly less than the time it takes to establish a new
   * connection.
   */
  int DISCARD_STREAM_TIMEOUT_MILLIS = 100;

  /** Returns an output stream where the request body can be streamed. */
  Sink createRequestBody(Request request, long contentLength);//写入请求体

  /** This should update the HTTP engine's sentRequestMillis field. */
  void writeRequestHeaders(Request request) throws IOException;//写入请求头

  /** Flush the request to the underlying socket. */
  void flushRequest() throws IOException;//提交

  /** Flush the request to the underlying socket and signal no more bytes will be transmitted. */
  void finishRequest() throws IOException;

  /**
   * Parses bytes of a response header from an HTTP transport.
   *
   * @param expectContinue true to return null if this is an intermediate response with a "100"
   *     response code. Otherwise this method never returns null.
   */
  Response.Builder readResponseHeaders(boolean expectContinue) throws IOException;//读取响应头

  /** Returns a stream that reads the response body. */
  ResponseBody openResponseBody(Response response) throws IOException;//读取响应体

  /**
   * Cancel this stream. Resources held by this stream will be cleaned up, though not synchronously.
   * That may happen later by the connection pool thread.
   */
  void cancel();//取消请求
}
```

`Http1Codec`

```java
public final class Http1Codec implements HttpCodec {
  private static final int STATE_IDLE = 0; // Idle connections are ready to write request headers.
  private static final int STATE_OPEN_REQUEST_BODY = 1;
  private static final int STATE_WRITING_REQUEST_BODY = 2;
  private static final int STATE_READ_RESPONSE_HEADERS = 3;
  private static final int STATE_OPEN_RESPONSE_BODY = 4;
  private static final int STATE_READING_RESPONSE_BODY = 5;
  private static final int STATE_CLOSED = 6;
    //
  private static final int HEADER_LIMIT = 256 * 1024;//头限制
  
   
  final OkHttpClient client;
  final StreamAllocation streamAllocation;

  final BufferedSource source; //socket source
  final BufferedSink sink;//socket sink
  int state = STATE_IDLE;//默认空闲
  private long headerLimit = HEADER_LIMIT;//请求头限制

  ///构造方法代码省略///
	//获取写入请求体的sink
  @Override public Sink createRequestBody(Request request, long contentLength) {
    if ("chunked".equalsIgnoreCase(request.header("Transfer-Encoding"))) {
      // Stream a request body of unknown length.
      return newChunkedSink();//根据请求头获取相应类型的的sink
    }

    if (contentLength != -1) {
      return newFixedLengthSink(contentLength);//固定长度的sink
    }

    throw new IllegalStateException(
        "Cannot stream a request body without chunked encoding or a known content length!");
  }
//退出
  @Override public void cancel() {
    RealConnection connection = streamAllocation.connection();
    if (connection != null) connection.cancel();//关闭connection
  }

  //设置请求头
  @Override public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());
    writeRequest(request.headers(), requestLine);//给sink写入头
  }

    //解析响应体
  @Override public ResponseBody openResponseBody(Response response) throws IOException {
    streamAllocation.eventListener.responseBodyStart(streamAllocation.call);//event回调
    String contentType = response.header("Content-Type");//获取content-type

    if (!HttpHeaders.hasBody(response)) {//没有body
      Source source = newFixedLengthSource(0);//构建空的source
      return new RealResponseBody(contentType, 0, Okio.buffer(source));//返回
    }

    if ("chunked".equalsIgnoreCase(response.header("Transfer-Encoding"))) {
      Source source = newChunkedSource(response.request().url());//根据url请求响应
      return new RealResponseBody(contentType, -1L, Okio.buffer(source));//返回
    }

    long contentLength = HttpHeaders.contentLength(response);
    if (contentLength != -1) {
      Source source = newFixedLengthSource(contentLength);
      return new RealResponseBody(contentType, contentLength, Okio.buffer(source));//构建定长的source
    }
	//构建未知source
    return new RealResponseBody(contentType, -1L, Okio.buffer(newUnknownLengthSource()));
  }

 /////省略部分代码////

  public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;//头部写入完成，进入下一状态
  }

    //读响应头
  @Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
      throw new IllegalStateException("state: " + state);
    }

    try {
      StatusLine statusLine = StatusLine.parse(readHeaderLine());//解析状态行

      Response.Builder responseBuilder = new Response.Builder()//根据状态行构建response
          .protocol(statusLine.protocol)
          .code(statusLine.code)
          .message(statusLine.message)
          .headers(readHeaders());

      if (expectContinue && statusLine.code == HTTP_CONTINUE) {
        return null;readHeaders
      } else if (statusLine.code == HTTP_CONTINUE) {
        state = STATE_READ_RESPONSE_HEADERS;//进入下一状态
        return responseBuilder;//返回读取的头
      }

      state = STATE_OPEN_RESPONSE_BODY;//
      return responseBuilder;
    } catch (EOFException e) {
     IOException exception = new IOException("unexpected end of stream on " + streamAllocation);
      exception.initCause(e);
      throw exception;
    }
  }

  private String readHeaderLine() throws IOException {//读取一行响应头
    String line = source.readUtf8LineStrict(headerLimit);//读取一行
    headerLimit -= line.length();//
    return line;
  }

  public Headers readHeaders() throws IOException {
    Headers.Builder headers = new Headers.Builder();
    // parse the result headers until the first blank line
    for (String line; (line = readHeaderLine()).length() != 0; ) {//读取剩下的响应头
      Internal.instance.addLenient(headers, line);
    }
    return headers.build();
  }

  public Sink newChunkedSink() {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new ChunkedSink();
  }

  public Sink newFixedLengthSink(long contentLength) {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new FixedLengthSink(contentLength);
  }

  public Source newFixedLengthSource(long length) throws IOException {
    if (state != STATE_OPEN_RESPONSE_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_READING_RESPONSE_BODY;
    return new FixedLengthSource(length);
  }

  public Source newChunkedSource(HttpUrl url) throws IOException {
    if (state != STATE_OPEN_RESPONSE_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_READING_RESPONSE_BODY;
    return new ChunkedSource(url);
  }

  public Source newUnknownLengthSource() throws IOException {
    if (state != STATE_OPEN_RESPONSE_BODY) throw new IllegalStateException("state: " + state);
    if (streamAllocation == null) throw new IllegalStateException("streamAllocation == null");
    state = STATE_READING_RESPONSE_BODY;
    streamAllocation.noNewStreams();
    return new UnknownLengthSource();
  }


  void detachTimeout(ForwardingTimeout timeout) {
    Timeout oldDelegate = timeout.delegate();
    timeout.setDelegate(Timeout.NONE);
    oldDelegate.clearDeadline();
    oldDelegate.clearTimeout();
  }

  /** An HTTP body with a fixed length known in advance. */
  private final class FixedLengthSink implements Sink {
    private final ForwardingTimeout timeout = new ForwardingTimeout(sink.timeout());
    private boolean closed;
    private long bytesRemaining;

    FixedLengthSink(long bytesRemaining) {
      this.bytesRemaining = bytesRemaining;
    }

    @Override public Timeout timeout() {
      return timeout;
    }

    @Override public void write(Buffer source, long byteCount) throws IOException {
      if (closed) throw new IllegalStateException("closed");
      checkOffsetAndCount(source.size(), 0, byteCount);
      if (byteCount > bytesRemaining) {
        throw new ProtocolException("expected " + bytesRemaining
            + " bytes but received " + byteCount);
      }
      sink.write(source, byteCount);
      bytesRemaining -= byteCount;
    }

    @Override public void flush() throws IOException {
      if (closed) return; // Don't throw; this stream might have been closed on the caller's behalf.
      sink.flush();
    }

    @Override public void close() throws IOException {
      if (closed) return;
      closed = true;
      if (bytesRemaining > 0) throw new ProtocolException("unexpected end of stream");
      detachTimeout(timeout);
      state = STATE_READ_RESPONSE_HEADERS;
    }
  }
}

```

```java
以下是几个用到的内部类 
 
  private final class ChunkedSink implements Sink {//实际还是操作的codec持有的sink
    private final ForwardingTimeout timeout = new ForwardingTimeout(sink.timeout());
    private boolean closed;

    ChunkedSink() {
    }

    @Override public Timeout timeout() {
      return timeout;
    }

    @Override public void write(Buffer source, long byteCount) throws IOException {
      if (closed) throw new IllegalStateException("closed");
      if (byteCount == 0) return;

      sink.writeHexadecimalUnsignedLong(byteCount);
      sink.writeUtf8("\r\n");
      sink.write(source, byteCount);
      sink.writeUtf8("\r\n");
    }

    @Override public synchronized void flush() throws IOException {
      if (closed) return; // Don't throw; this stream might have been closed on the caller's behalf.
      sink.flush();
    }

    @Override public synchronized void close() throws IOException {
      if (closed) return;
      closed = true;
      sink.writeUtf8("0\r\n\r\n");
      detachTimeout(timeout);
      state = STATE_READ_RESPONSE_HEADERS;
    }
  }
	
//source
  private abstract class AbstractSource implements Source {
    protected final ForwardingTimeout timeout = new ForwardingTimeout(source.timeout());
    protected boolean closed;
    protected long bytesRead = 0;

    @Override public Timeout timeout() {
      return timeout;
    }

    @Override public long read(Buffer sink, long byteCount) throws IOException {
      try {
        long read = source.read(sink, byteCount);
        if (read > 0) {
          bytesRead += read;
        }
        return read;
      } catch (IOException e) {
        endOfInput(false, e);
        throw e;
      }
    }

    /**
     * Closes the cache entry and makes the socket available for reuse. This should be invoked when
     * the end of the body has been reached.
     */
    protected final void endOfInput(boolean reuseConnection, IOException e) throws IOException {
      if (state == STATE_CLOSED) return;
      if (state != STATE_READING_RESPONSE_BODY) throw new IllegalStateException("state: " + state);

      detachTimeout(timeout);

      state = STATE_CLOSED;
      if (streamAllocation != null) {
        streamAllocation.streamFinished(!reuseConnection, Http1Codec.this, bytesRead, e);
      }
    }
  }

  /** An HTTP body with a fixed length specified in advance. */
  private class FixedLengthSource extends AbstractSource {
    private long bytesRemaining;

    FixedLengthSource(long length) throws IOException {
      bytesRemaining = length;
      if (bytesRemaining == 0) {
        endOfInput(true, null);
      }
    }

    @Override public long read(Buffer sink, long byteCount) throws IOException {
      if (byteCount < 0) throw new IllegalArgumentException("byteCount < 0: " + byteCount);
      if (closed) throw new IllegalStateException("closed");
      if (bytesRemaining == 0) return -1;

      long read = super.read(sink, Math.min(bytesRemaining, byteCount));
      if (read == -1) {
        ProtocolException e = new ProtocolException("unexpected end of stream");
        endOfInput(false, e); // The server didn't supply the promised content length.
        throw e;
      }

      bytesRemaining -= read;
      if (bytesRemaining == 0) {
        endOfInput(true, null);
      }
      return read;
    }

    @Override public void close() throws IOException {
      if (closed) return;

      if (bytesRemaining != 0 && !Util.discard(this, DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
        endOfInput(false, null);
      }

      closed = true;
    }
  }

  /** An HTTP body with alternating chunk sizes and chunk bodies. */
  private class ChunkedSource extends AbstractSource {
    private static final long NO_CHUNK_YET = -1L;
    private final HttpUrl url;
    private long bytesRemainingInChunk = NO_CHUNK_YET;
    private boolean hasMoreChunks = true;

    ChunkedSource(HttpUrl url) {
      this.url = url;
    }

    @Override public long read(Buffer sink, long byteCount) throws IOException {
      if (byteCount < 0) throw new IllegalArgumentException("byteCount < 0: " + byteCount);
      if (closed) throw new IllegalStateException("closed");
      if (!hasMoreChunks) return -1;

      if (bytesRemainingInChunk == 0 || bytesRemainingInChunk == NO_CHUNK_YET) {
        readChunkSize();
        if (!hasMoreChunks) return -1;
      }

      long read = super.read(sink, Math.min(byteCount, bytesRemainingInChunk));
      if (read == -1) {
        ProtocolException e = new ProtocolException("unexpected end of stream");
        endOfInput(false, e); // The server didn't supply the promised chunk length.
        throw e;
      }
      bytesRemainingInChunk -= read;
      return read;
    }

    private void readChunkSize() throws IOException {
      // Read the suffix of the previous chunk.
      if (bytesRemainingInChunk != NO_CHUNK_YET) {
        source.readUtf8LineStrict();
      }
      try {
        bytesRemainingInChunk = source.readHexadecimalUnsignedLong();
        String extensions = source.readUtf8LineStrict().trim();
        if (bytesRemainingInChunk < 0 || (!extensions.isEmpty() && !extensions.startsWith(";"))) {
          throw new ProtocolException("expected chunk size and optional extensions but was \""
              + bytesRemainingInChunk + extensions + "\"");
        }
      } catch (NumberFormatException e) {
        throw new ProtocolException(e.getMessage());
      }
      if (bytesRemainingInChunk == 0L) {
        hasMoreChunks = false;
        HttpHeaders.receiveHeaders(client.cookieJar(), url, readHeaders());
        endOfInput(true, null);
      }
    }

    @Override public void close() throws IOException {
      if (closed) return;
      if (hasMoreChunks && !Util.discard(this, DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
        endOfInput(false, null);
      }
      closed = true;
    }
  }

  /** An HTTP message body terminated by the end of the underlying stream. */
  private class UnknownLengthSource extends AbstractSource {
    private boolean inputExhausted;

    UnknownLengthSource() {
    }

    @Override public long read(Buffer sink, long byteCount)
        throws IOException {
      if (byteCount < 0) throw new IllegalArgumentException("byteCount < 0: " + byteCount);
      if (closed) throw new IllegalStateException("closed");
      if (inputExhausted) return -1;

      long read = super.read(sink, byteCount);
      if (read == -1) {
        inputExhausted = true;
        endOfInput(true, null);
        return -1;
      }
      return read;
    }

    @Override public void close() throws IOException {
      if (closed) return;
      if (!inputExhausted) {
        endOfInput(false, null);
      }
      closed = true;
    }
  }
```

###### Connection

```java
Route route(); //返回一个路由
Socket socket();  //返回一个socket
Handshake handshake();  //如果是一个https,则返回一个TLS握手协议
Protocol protocol(); //返回一个协议类型 比如 http1.1 等或者自定义类型 
```

=====

###### RealConnection

```java
public final class RealConnection extends Http2Connection.Listener implements Connection {
  private static final String NPE_THROW_WITH_NULL = "throw with null exception";
  private static final int MAX_TUNNEL_ATTEMPTS = 21;

  private final ConnectionPool connectionPool;//连接池
  private final Route route;//路由？？不知道是啥 todo

  // 以下属性通过 connect() 初始化 and never reassigned.
  /** The low-level TCP socket. */
  private Socket rawSocket; //底层TCP socket 

  /**
   * The application layer socket. Either an {@link SSLSocket} layered over {@link #rawSocket}, or
   * {@link #rawSocket} itself if this connection does not use SSL.
   */
  private Socket socket;//应用层socket
  private Handshake handshake;//握手
  private Protocol protocol;//协议
  private Http2Connection http2Connection;//http2连接
  private BufferedSource source;//与服务器交互的输入输出流
  private BufferedSink sink;

  // The fields below track connection state and are guarded by connectionPool.

  /** If true, no new streams can be created on this connection. Once true this is always true. */
  public boolean noNewStreams;

  public int successCount;

  /**
   * The maximum number of concurrent streams that can be carried by this connection. If {@code
   * allocations.size() < allocationLimit} then new streams can be created on this connection.
   */
  public int allocationLimit = 1;//最大并发数

  /** Current streams carried by this connection. */
  public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();

  /** Nanotime timestamp when {@code allocations.size()} reached zero. */
  public long idleAtNanos = Long.MAX_VALUE;

  public RealConnection(ConnectionPool connectionPool, Route route) {
    this.connectionPool = connectionPool;
    this.route = route;
  }
	
    //connect 连接 
  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
      EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");//协议不为空，已经连接了

    RouteException routeException = null;
      //获取route支持的链接规格
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

    if (route.address().sslSocketFactory() == null) {//没有ssl
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {//支持的连接规格中是否包含明文
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {//平台不支持明文
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    } else {//ssl
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        throw new RouteException(new UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));//
      }
    }
	//开始连接
    while (true) {//循环
      try {
        if (route.requiresTunnel()) {//需要使用隧道连接
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);//连接隧道
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break;
          }
        } else {
          connectSocket(connectTimeout, readTimeout, call, eventListener);//连接socket
        }
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);//建立协议
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
          //event回调
        break;
      } catch (IOException e) {
        closeQuietly(socket);//关闭socket
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;
        http2Connection = null;
		
        eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);

        if (routeException == null) {
          routeException = new RouteException(e);
        } else {
          routeException.addConnectException(e);
        }

        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException;
        }
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {//尝试创建隧道次数超出最大
      ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
          + MAX_TUNNEL_ATTEMPTS);
      throw new RouteException(exception);
    }

    if (http2Connection != null) {//使用的是http2的方式，则并发数量不为1
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();//获取一个链接支持的最大并发
      }
    }
  }

  /**
   * 通过代理隧道构建https链接
   */
  private void connectTunnel(int connectTimeout, int readTimeout, int writeTimeout, Call call,
      EventListener eventListener) throws IOException {
    Request tunnelRequest = createTunnelRequest();//创建隧道的请求
    HttpUrl url = tunnelRequest.url();//隧道url
    for (int i = 0; i < MAX_TUNNEL_ATTEMPTS; i++) { //在接受的尝试创建隧道次数内
      connectSocket(connectTimeout, readTimeout, call, eventListener);//创建socket链接
      tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url);//创建隧道请求

      if (tunnelRequest == null) break; // Tunnel successfully created.

      // The proxy decided to close the connection after an auth challenge. We need to create a new
      // connection, but this time with the auth credentials.
      closeQuietly(rawSocket);//失败，关闭socket，重来
      rawSocket = null;
      sink = null;
      source = null;
      eventListener.connectEnd(call, route.socketAddress(), route.proxy(), null);
    }
  }

  /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw 
 socket. 创建底层socket链接 */
  private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();//代理
    Address address = route.address();//地址

    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()//根据代理类型创建不一样的socket
        : new Socket(proxy);

    eventListener.connectStart(call, route.socketAddress(), proxy);
    rawSocket.setSoTimeout(readTimeout);
    try {
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);//链接socket
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }

    // The following try/catch block is a pseudo hacky way to get around a crash on Android 7.0
    // More details:
    // https://github.com/square/okhttp/issues/3245
    // https://android-review.googlesource.com/#/c/271775/
    try {
      source = Okio.buffer(Okio.source(rawSocket));//设置输入输出流
      sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
      if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
        throw new IOException(npe);
      }
    }
  }
	
    //建立协议
  private void establishProtocol(ConnectionSpecSelector connectionSpecSelector,
      int pingIntervalMillis, Call call, EventListener eventListener) throws IOException {
    if (route.address().sslSocketFactory() == null) {//http
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {//协议包括h2的明文协议
        socket = rawSocket;
        protocol = Protocol.H2_PRIOR_KNOWLEDGE;
        startHttp2(pingIntervalMillis);//建立h2
        return;
      }

      socket = rawSocket;//h1方案
      protocol = Protocol.HTTP_1_1;
      return;
    }

    eventListener.secureConnectStart(call);
    connectTls(connectionSpecSelector);//https 链接
    eventListener.secureConnectEnd(call, handshake);

    if (protocol == Protocol.HTTP_2) {//http2 协议？
      startHttp2(pingIntervalMillis);//建立h2
    }
  }

  private void startHttp2(int pingIntervalMillis) throws IOException {
    socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
    http2Connection = new Http2Connection.Builder(true)
        .socket(socket, route.address().url().host(), source, sink)
        .listener(this)
        .pingIntervalMillis(pingIntervalMillis)
        .build();
    http2Connection.start();//h2connection
  }

    //tls链接
  private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    Address address = route.address();
    SSLSocketFactory sslSocketFactory = address.sslSocketFactory();
    boolean success = false;
    SSLSocket sslSocket = null;
    try {
      // Create the wrapper over the connected socket.
      sslSocket = (SSLSocket) sslSocketFactory.createSocket(
          rawSocket, address.url().host(), address.url().port(), true /* autoClose */);

      // Configure the socket's ciphers, TLS versions, and extensions.
      ConnectionSpec connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket);
      if (connectionSpec.supportsTlsExtensions()) {
        Platform.get().configureTlsExtensions(
            sslSocket, address.url().host(), address.protocols());
      }

      // Force handshake. This can throw!
      sslSocket.startHandshake();
      // block for session establishment
      SSLSession sslSocketSession = sslSocket.getSession();
      Handshake unverifiedHandshake = Handshake.get(sslSocketSession);

      // Verify that the socket's certificates are acceptable for the target host.
      if (!address.hostnameVerifier().verify(address.url().host(), sslSocketSession)) {
        X509Certificate cert = (X509Certificate) unverifiedHandshake.peerCertificates().get(0);
        throw new SSLPeerUnverifiedException("Hostname " + address.url().host() + " not verified:"
            + "\n    certificate: " + CertificatePinner.pin(cert)
            + "\n    DN: " + cert.getSubjectDN().getName()
            + "\n    subjectAltNames: " + OkHostnameVerifier.allSubjectAltNames(cert));
      }

      // Check that the certificate pinner is satisfied by the certificates presented.
      address.certificatePinner().check(address.url().host(),
          unverifiedHandshake.peerCertificates());

      // Success! Save the handshake and the ALPN protocol.
      String maybeProtocol = connectionSpec.supportsTlsExtensions()
          ? Platform.get().getSelectedProtocol(sslSocket)
          : null;
      socket = sslSocket;
      source = Okio.buffer(Okio.source(socket));
      sink = Okio.buffer(Okio.sink(socket));
      handshake = unverifiedHandshake;
      protocol = maybeProtocol != null
          ? Protocol.get(maybeProtocol)
          : Protocol.HTTP_1_1;
      success = true;
    } catch (AssertionError e) {
      if (Util.isAndroidGetsocknameError(e)) throw new IOException(e);
      throw e;
    } finally {
      if (sslSocket != null) {
        Platform.get().afterHandshake(sslSocket);
      }
      if (!success) {
        closeQuietly(sslSocket);
      }
    }
  }

  /**
   * To make an HTTPS connection over an HTTP proxy, send an unencrypted CONNECT request to create
   * the proxy connection. This may need to be retried if the proxy requires authorization.
   */
  private Request createTunnel(int readTimeout, int writeTimeout, Request tunnelRequest,
      HttpUrl url) throws IOException {
    // Make an SSL Tunnel on the first message pair of each SSL + proxy connection.
    String requestLine = "CONNECT " + Util.hostHeader(url, true) + " HTTP/1.1";
    while (true) {
      Http1Codec tunnelConnection = new Http1Codec(null, null, source, sink);
      source.timeout().timeout(readTimeout, MILLISECONDS);
      sink.timeout().timeout(writeTimeout, MILLISECONDS);
      tunnelConnection.writeRequest(tunnelRequest.headers(), requestLine);
      tunnelConnection.finishRequest();
      Response response = tunnelConnection.readResponseHeaders(false)
          .request(tunnelRequest)
          .build();
      // The response body from a CONNECT should be empty, but if it is not then we should consume
      // it before proceeding.
      long contentLength = HttpHeaders.contentLength(response);
      if (contentLength == -1L) {
        contentLength = 0L;
      }
      Source body = tunnelConnection.newFixedLengthSource(contentLength);
      Util.skipAll(body, Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
      body.close();

      switch (response.code()) {
        case HTTP_OK:
          // Assume the server won't send a TLS ServerHello until we send a TLS ClientHello. If
          // that happens, then we will have buffered bytes that are needed by the SSLSocket!
          // This check is imperfect: it doesn't tell us whether a handshake will succeed, just
          // that it will almost certainly fail because the proxy has sent unexpected data.
          if (!source.buffer().exhausted() || !sink.buffer().exhausted()) {
            throw new IOException("TLS tunnel buffered too many bytes!");
          }
          return null;

        case HTTP_PROXY_AUTH:
          tunnelRequest = route.address().proxyAuthenticator().authenticate(route, response);
          if (tunnelRequest == null) throw new IOException("Failed to authenticate with proxy");

          if ("close".equalsIgnoreCase(response.header("Connection"))) {
            return tunnelRequest;
          }
          break;

        default:
          throw new IOException(
              "Unexpected response code for CONNECT: " + response.code());
      }
    }
  }

  /**
   * Returns a request that creates a TLS tunnel via an HTTP proxy. Everything in the tunnel request
   * is sent unencrypted to the proxy server, so tunnels include only the minimum set of headers.
   * This avoids sending potentially sensitive data like HTTP cookies to the proxy unencrypted.
   */
  private Request createTunnelRequest() {
    return new Request.Builder()
        .url(route.address().url())
        .header("Host", Util.hostHeader(route.address().url(), true))
        .header("Proxy-Connection", "Keep-Alive") // For HTTP/1.0 proxies like Squid.
        .header("User-Agent", Version.userAgent())
        .build();
  }

  /**
   * Returns true if this connection can carry a stream allocation to {@code address}. If non-null
   * {@code route} is the resolved route for a connection.
   */
  public boolean isEligible(Address address, @Nullable Route route) {
    // If this connection is not accepting new streams, we're done.
    if (allocations.size() >= allocationLimit || noNewStreams) return false;

    // If the non-host fields of the address don't overlap, we're done.
    if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;

    // If the host exactly matches, we're done: this connection can carry the address.
    if (address.url().host().equals(this.route().address().url().host())) {
      return true; // This connection is a perfect match.
    }

    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false;

    // 2. The routes must share an IP address. This requires us to have a DNS address for both
    // hosts, which only happens after route planning. We can't coalesce connections that use a
    // proxy, since proxies don't tell us the origin server's IP address.
    if (route == null) return false;
    if (route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (!this.route.socketAddress().equals(route.socketAddress())) return false;

    // 3. This connection's server certificate's must cover the new host.
    if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
    if (!supportsUrl(address.url())) return false;

    // 4. Certificate pinning must match the host.
    try {
      address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
    } catch (SSLPeerUnverifiedException e) {
      return false;
    }

    return true; // The caller's address can be carried by this connection.
  }

  public boolean supportsUrl(HttpUrl url) {
    if (url.port() != route.address().url().port()) {
      return false; // Port mismatch.
    }

    if (!url.host().equals(route.address().url().host())) {
      // We have a host mismatch. But if the certificate matches, we're still good.
      return handshake != null && OkHostnameVerifier.INSTANCE.verify(
          url.host(), (X509Certificate) handshake.peerCertificates().get(0));
    }

    return true; // Success. The URL is supported.
  }

  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }

  public RealWebSocket.Streams newWebSocketStreams(final StreamAllocation streamAllocation) {
    return new RealWebSocket.Streams(true, source, sink) {
      @Override public void close() throws IOException {
        streamAllocation.streamFinished(true, streamAllocation.codec(), -1L, null);
      }
    };
  }

  @Override public Route route() {
    return route;
  }

  public void cancel() {
    // Close the raw socket so we don't end up doing synchronous I/O.
    closeQuietly(rawSocket);
  }

  @Override public Socket socket() {
    return socket;
  }

  /** Returns true if this connection is ready to host new streams. */
  public boolean isHealthy(boolean doExtensiveChecks) {
    if (socket.isClosed() || socket.isInputShutdown() || socket.isOutputShutdown()) {
      return false;
    }

    if (http2Connection != null) {
      return !http2Connection.isShutdown();
    }

    if (doExtensiveChecks) {
      try {
        int readTimeout = socket.getSoTimeout();
        try {
          socket.setSoTimeout(1);
          if (source.exhausted()) {
            return false; // Stream is exhausted; socket is closed.
          }
          return true;
        } finally {
          socket.setSoTimeout(readTimeout);
        }
      } catch (SocketTimeoutException ignored) {
        // Read timed out; socket is good.
      } catch (IOException e) {
        return false; // Couldn't read; socket is closed.
      }
    }

    return true;
  }

  /** Refuse incoming streams. */
  @Override public void onStream(Http2Stream stream) throws IOException {
    stream.close(ErrorCode.REFUSED_STREAM);
  }

  /** When settings are received, adjust the allocation limit. */
  @Override public void onSettings(Http2Connection connection) {
    synchronized (connectionPool) {
      allocationLimit = connection.maxConcurrentStreams();
    }
  }

  @Override public Handshake handshake() {
    return handshake;
  }

  /**
   * Returns true if this is an HTTP/2 connection. Such connections can be used in multiple HTTP
   * requests simultaneously.
   */
  public boolean isMultiplexed() {
    return http2Connection != null;
  }

  @Override public Protocol protocol() {
    return protocol;
  }

  @Override public String toString() {
    return "Connection{"
        + route.address().url().host() + ":" + route.address().url().port()
        + ", proxy="
        + route.proxy()
        + " hostAddress="
        + route.socketAddress()
        + " cipherSuite="
        + (handshake != null ? handshake.cipherSuite() : "none")
        + " protocol="
        + protocol
        + '}';
  }
}

```

###### StreamAllocation

```java
/*
 * Copyright (C) 2015 Square, Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package okhttp3.internal.connection;

import java.io.IOException;
import java.lang.ref.Reference;
import java.lang.ref.WeakReference;
import java.net.Socket;
import java.util.List;
import okhttp3.Address;
import okhttp3.Call;
import okhttp3.Connection;
import okhttp3.ConnectionPool;
import okhttp3.EventListener;
import okhttp3.Interceptor;
import okhttp3.OkHttpClient;
import okhttp3.Route;
import okhttp3.internal.Internal;
import okhttp3.internal.Util;
import okhttp3.internal.http.HttpCodec;
import okhttp3.internal.http2.ConnectionShutdownException;
import okhttp3.internal.http2.ErrorCode;
import okhttp3.internal.http2.StreamResetException;

import static okhttp3.internal.Util.closeQuietly;

/**
 * This class coordinates the relationship between three entities:
 *
 * <ul>
 *     <li><strong>Connections:</strong> physical socket connections to remote servers. These are
 *         potentially slow to establish so it is necessary to be able to cancel a connection
 *         currently being connected.
 *     <li><strong>Streams:</strong> logical HTTP request/response pairs that are layered on
 *         connections. Each connection has its own allocation limit, which defines how many
 *         concurrent streams that connection can carry. HTTP/1.x connections can carry 1 stream
 *         at a time, HTTP/2 typically carry multiple.
 *     <li><strong>Calls:</strong> a logical sequence of streams, typically an initial request and
 *         its follow up requests. We prefer to keep all streams of a single call on the same
 *         connection for better behavior and locality.
 * </ul>
 *
 * <p>Instances of this class act on behalf of the call, using one or more streams over one or more
 * connections. This class has APIs to release each of the above resources:
 *
 * <ul>
 *     <li>{@link #noNewStreams()} prevents the connection from being used for new streams in the
 *         future. Use this after a {@code Connection: close} header, or when the connection may be
 *         inconsistent.
 *     <li>{@link #streamFinished streamFinished()} releases the active stream from this allocation.
 *         Note that only one stream may be active at a given time, so it is necessary to call
 *         {@link #streamFinished streamFinished()} before creating a subsequent stream with {@link
 *         #newStream newStream()}.
 *     <li>{@link #release()} removes the call's hold on the connection. Note that this won't
 *         immediately free the connection if there is a stream still lingering. That happens when a
 *         call is complete but its response body has yet to be fully consumed.
 * </ul>
 *
 * <p>This class supports {@linkplain #cancel asynchronous canceling}. This is intended to have the
 * smallest blast radius possible. If an HTTP/2 stream is active, canceling will cancel that stream
 * but not the other streams sharing its connection. But if the TLS handshake is still in progress
 * then canceling may break the entire connection.
 */
public final class StreamAllocation {
  public final Address address; //地址
  private RouteSelector.Selection routeSelection;//route选择
  private Route route;//luyou
  private final ConnectionPool connectionPool;
  public final Call call;
  public final EventListener eventListener;
  private final Object callStackTrace;

  // State guarded by connectionPool.
  private final RouteSelector routeSelector;
  private int refusedStreamCount;
  private RealConnection connection;
  private boolean reportedAcquired;
  private boolean released;
  private boolean canceled;
  private HttpCodec codec;

  public StreamAllocation(ConnectionPool connectionPool, Address address, Call call,
      EventListener eventListener, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.call = call;
    this.eventListener = eventListener;
    this.routeSelector = new RouteSelector(address, routeDatabase(), call, eventListener);
    this.callStackTrace = callStackTrace;
  }

  public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {//获取一个新的流
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);//获取一个健康的链接
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);//生成链接的输入输出流

      synchronized (connectionPool) {
        codec = resultCodec;//持有此流
        return resultCodec;//返回
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }

  /**
   * 不断查找 zhidao找到一个可用的健康链接
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);//找一个可用链接

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {//新创建的，可以不进行检查，直接返回
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {//不健康的
        noNewStreams();//
        continue;//继续
      }

      return candidate;
    }
  }

  /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();//nonewstream则释放
      if (this.connection != null) { //已经持有链接了
        // We had an already-allocated connection and it's good.
        result = this.connection;//准备返回自己
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {//之前没有绑定
        // Attempt to get a connection from the pool.
        Internal.instance.get(connectionPool, address, this, null);//从连接池中找
        if (connection != null) {//找到
          foundPooledConnection = true;
          result = connection;
        } else {//没找到，标记当前route
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {//找到，返回
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.陆游空
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {//aigezhao
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {//没找到
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();//下一个
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;//新建一个链接
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);//设置链接
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {//在连接池中找到的，直接返回
      eventListener.connectionAcquired(call, result);
      return result;
    }
	//新建的
    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);//建立链接
    routeDatabase().connected(result.route());//村金曲

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;//已经被绑定了

      // Pool the connection.
      Internal.instance.put(connectionPool, result);//把新建的链接放入连接池

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {//可以合并的，http2
        socket = Internal.instance.deduplicate(connectionPool, address, this);//压缩socket
        result = connection;//
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }

  /**
   * Releases the currently held connection and returns a socket to close if the held connection
   * restricts new streams from being created. With HTTP/2 multiple requests share the same
   * connection so it's possible that our connection is restricted from creating new streams during
   * a follow-up request.
   */
  private Socket releaseIfNoNewStreams() {
    assert (Thread.holdsLock(connectionPool));
    RealConnection allocatedConnection = this.connection;
    if (allocatedConnection != null && allocatedConnection.noNewStreams) {
      return deallocate(false, false, true);
    }
    return null;
  }

  public void streamFinished(boolean noNewStreams, HttpCodec codec, long bytesRead, IOException e) {
    eventListener.responseBodyEnd(call, bytesRead);

    Socket socket;
    Connection releasedConnection;
    boolean callEnd;
    synchronized (connectionPool) {
      if (codec == null || codec != this.codec) {
        throw new IllegalStateException("expected " + this.codec + " but was " + codec);
      }
      if (!noNewStreams) {
        connection.successCount++;
      }
      releasedConnection = connection;
      socket = deallocate(noNewStreams, false, true);
      if (connection != null) releasedConnection = null;
      callEnd = this.released;
    }
    closeQuietly(socket);
    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }

    if (e != null) {
      eventListener.callFailed(call, e);
    } else if (callEnd) {
      eventListener.callEnd(call);
    }
  }

  public HttpCodec codec() {
    synchronized (connectionPool) {
      return codec;
    }
  }

  private RouteDatabase routeDatabase() {
    return Internal.instance.routeDatabase(connectionPool);
  }

  public Route route() {
    return route;
  }

  public synchronized RealConnection connection() {
    return connection;
  }

  public void release() {
    Socket socket;
    Connection releasedConnection;
    synchronized (connectionPool) {
      releasedConnection = connection;
      socket = deallocate(false, true, false);
      if (connection != null) releasedConnection = null;
    }
    closeQuietly(socket);
    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
      eventListener.callEnd(call);
    }
  }

  /** Forbid new streams from being created on the connection that hosts this allocation. */
  public void noNewStreams() {
    Socket socket;
    Connection releasedConnection;
    synchronized (connectionPool) {
      releasedConnection = connection;
      socket = deallocate(true, false, false);
      if (connection != null) releasedConnection = null;
    }
    closeQuietly(socket);
    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
  }

  /**
   * Releases resources held by this allocation. If sufficient resources are allocated, the
   * connection will be detached or closed. Callers must be synchronized on the connection pool.
   *
   * <p>Returns a closeable that the caller should pass to {@link Util#closeQuietly} upon completion
   * of the synchronized block. (We don't do I/O while synchronized on the connection pool.)
   */
  private Socket deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {
    assert (Thread.holdsLock(connectionPool));

    if (streamFinished) {
      this.codec = null;
    }
    if (released) {
      this.released = true;
    }
    Socket socket = null;
    if (connection != null) {
      if (noNewStreams) {
        connection.noNewStreams = true;
      }
      if (this.codec == null && (this.released || connection.noNewStreams)) {
        release(connection);
        if (connection.allocations.isEmpty()) {
          connection.idleAtNanos = System.nanoTime();
          if (Internal.instance.connectionBecameIdle(connectionPool, connection)) {
            socket = connection.socket();
          }
        }
        connection = null;
      }
    }
    return socket;
  }

  public void cancel() {
    HttpCodec codecToCancel;
    RealConnection connectionToCancel;
    synchronized (connectionPool) {
      canceled = true;
      codecToCancel = codec;
      connectionToCancel = connection;
    }
    if (codecToCancel != null) {
      codecToCancel.cancel();
    } else if (connectionToCancel != null) {
      connectionToCancel.cancel();
    }
  }

  public void streamFailed(IOException e) {
    Socket socket;
    Connection releasedConnection;
    boolean noNewStreams = false;

    synchronized (connectionPool) {
      if (e instanceof StreamResetException) {
        ErrorCode errorCode = ((StreamResetException) e).errorCode;
        if (errorCode == ErrorCode.REFUSED_STREAM) {
          // Retry REFUSED_STREAM errors once on the same connection.
          refusedStreamCount++;
          if (refusedStreamCount > 1) {
            noNewStreams = true;
            route = null;
          }
        } else if (errorCode != ErrorCode.CANCEL) {
          // Keep the connection for CANCEL errors. Everything else wants a fresh connection.
          noNewStreams = true;
          route = null;
        }
      } else if (connection != null
          && (!connection.isMultiplexed() || e instanceof ConnectionShutdownException)) {
        noNewStreams = true;

        // If this route hasn't completed a call, avoid it for new connections.
        if (connection.successCount == 0) {
          if (route != null && e != null) {
            routeSelector.connectFailed(route, e);
          }
          route = null;
        }
      }
      releasedConnection = connection;
      socket = deallocate(noNewStreams, false, true);
      if (connection != null || !reportedAcquired) releasedConnection = null;
    }

    closeQuietly(socket);
    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
  }

  /**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }

  /** Remove this allocation from the connection's list of allocations. */
  private void release(RealConnection connection) {
    for (int i = 0, size = connection.allocations.size(); i < size; i++) {
      Reference<StreamAllocation> reference = connection.allocations.get(i);
      if (reference.get() == this) {
        connection.allocations.remove(i);
        return;
      }
    }
    throw new IllegalStateException();
  }

  /**
   * Release the connection held by this connection and acquire {@code newConnection} instead. It is
   * only safe to call this if the held connection is newly connected but duplicated by {@code
   * newConnection}. Typically this occurs when concurrently connecting to an HTTP/2 webserver.
   *
   * <p>Returns a closeable that the caller should pass to {@link Util#closeQuietly} upon completion
   * of the synchronized block. (We don't do I/O while synchronized on the connection pool.)
   */
  public Socket releaseAndAcquire(RealConnection newConnection) {
    assert (Thread.holdsLock(connectionPool));
    if (codec != null || connection.allocations.size() != 1) throw new IllegalStateException();

    // Release the old connection.
    Reference<StreamAllocation> onlyAllocation = connection.allocations.get(0);
    Socket socket = deallocate(true, false, false);

    // Acquire the new connection.
    this.connection = newConnection;
    newConnection.allocations.add(onlyAllocation);

    return socket;
  }

  public boolean hasMoreRoutes() {
    return route != null
        || (routeSelection != null && routeSelection.hasNext())
        || routeSelector.hasNext();
  }

  @Override public String toString() {
    RealConnection connection = connection();
    return connection != null ? connection.toString() : address.toString();
  }

  public static final class StreamAllocationReference extends WeakReference<StreamAllocation> {
    /**
     * Captures the stack trace at the time the Call is executed or enqueued. This is helpful for
     * identifying the origin of connection leaks.
     */
    public final Object callStackTrace;

    StreamAllocationReference(StreamAllocation referent, Object callStackTrace) {
      super(referent);
      this.callStackTrace = callStackTrace;
    }
  }
}

```



