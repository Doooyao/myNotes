### OKIO

#### 1. 输入输出

```
okio 使用Source / Sink 接口替代java的in/outputstream 


RealBufferedSink => BufferedSink =>Sink
```

``` java
RealBufferedSink 中持有一个Buffer 和Sink ，具体的操作调用Buffer实现
```

#### 2.Segment

存储buffer碎片，双向链表节点

```java
  static final int SIZE = 8192;//大小
  static final int SHARE_MINIMUM = 1024;//分享机制 暂时不懂 todo
  final byte[] data; //
  /** The next byte of application data byte to read in this segment. */
  int pos;
  /** The first byte of available data ready to be written to. */
  int limit;
  /** True if other segments or byte strings use the same byte array. */
  boolean shared;
  /** True if this segment owns the byte array and can append to it, extending {@code limit}. */
  boolean owner;
  /** Next segment in a linked or circularly-linked list. */
  Segment next;
  /** Previous segment in a circularly-linked list. */
  Segment prev;
```

构造方法及创建：

```java
Segment() {
    this.data = new byte[SIZE];
    this.owner = true;
    this.shared = false;
  }

  Segment(byte[] data, int pos, int limit, boolean shared, boolean owner) {
    this.data = data;
    this.pos = pos;
    this.limit = limit;
    this.shared = shared;
    this.owner = owner;
  }

  /**
   * Returns a new segment that shares the underlying byte array with this. Adjusting pos and limit
   * are safe but writes are forbidden. This also marks the current segment as shared, which
   * prevents it from being pooled.
   */
  Segment sharedCopy() {
    shared = true;
    return new Segment(data, pos, limit, true, false);
  }

  /** Returns a new segment that its own private copy of the underlying byte array. */
  Segment unsharedCopy() {
    return new Segment(data.clone(), pos, limit, false, true);
  }
```

```java
写入
/** Moves {@code byteCount} bytes from this segment to {@code sink}. */
  public void writeTo(Segment sink, int byteCount) {
    if (!sink.owner) throw new IllegalArgumentException();
    if (sink.limit + byteCount > SIZE) {
      // We can't fit byteCount bytes at the sink's current position. Shift sink first.
      if (sink.shared) throw new IllegalArgumentException();
      if (sink.limit + byteCount - sink.pos > SIZE) throw new IllegalArgumentException();
      System.arraycopy(sink.data, sink.pos, sink.data, 0, sink.limit - sink.pos);
      sink.limit -= sink.pos;
      sink.pos = 0;
    }

    System.arraycopy(data, pos, sink.data, sink.limit, byteCount);
    sink.limit += byteCount;
    pos += byteCount;
  }
```

```java
压缩优化
/**
   * Call this when the tail and its predecessor may both be less than half
   * full. This will copy data so that segments can be recycled.
   */
  public void compact() {
    if (prev == this) throw new IllegalStateException();//只有一个节点，无法压缩
    if (!prev.owner) return; // Cannot compact: prev isn't writable.//上一个节点没有控制权
    int byteCount = limit - pos;//数据长度
    int availableByteCount = SIZE - prev.limit + (prev.shared ? 0 : prev.pos);//可压缩长度，
    //上一个节点的剩余和索引之前的空间
    if (byteCount > availableByteCount) return; // Cannot compact: not enough writable space.//数据长度大于上一个节点的剩余空间，无法压缩
    writeTo(prev, byteCount);//写入上一个节点
    pop();//出队
    SegmentPool.recycle(this);//回收
  }
```

```java
split方法（共享机制）
public Segment split(int byteCount) {
    if (byteCount <= 0 || byteCount > limit - pos) throw new IllegalArgumentException();
    Segment prefix;
    // We have two competing performance goals:
    //  - Avoid copying data. We accomplish this by sharing segments.
    //  - Avoid short shared segments. These are bad for performance because they are readonly and
    //    may lead to long chains of short segments.
    // To balance these goals we only share segments when the copy will be large.
    if (byteCount >= SHARE_MINIMUM) {
      prefix = new Segment(this);
    } else {
      prefix = SegmentPool.take();
      System.arraycopy(data, pos, prefix.data, 0, byteCount);
    }

    prefix.limit = prefix.pos + byteCount;
    pos += byteCount;
    prev.push(prefix);
    return prefix;
  }
```

#### 3.SegmentPool

```java
参数
/** The maximum number of bytes to pool. */
  // TODO: Is 64 KiB a good maximum size? Do we ever have that many idle segments?
  static final long MAX_SIZE = 64 * 1024; // 64 KiB.最大容量

  /** Singly-linked list of segments. */
  static @Nullable Segment next; //单链表next 用于回收

//该值记录了当前所有Segment的总大小，最大值是为MAX_SIZE
  static long byteCount;//
```

```java
static Segment take() {
    synchronized (SegmentPool.class) {
      if (next != null) { //池中有segment，取出
        Segment result = next; 
        next = result.next;
        result.next = null;
        byteCount -= Segment.SIZE;
        return result;
      }
    }
    return new Segment(); // Pool is empty. Don't zero-fill while holding a lock.
  }
```

```java
static void recycle(Segment segment) {
    if (segment.next != null || segment.prev != null) throw new IllegalArgumentException();
    if (segment.shared) return; // This segment cannot be recycled.//共享的segment不能回收
    synchronized (SegmentPool.class) {
      if (byteCount + Segment.SIZE > MAX_SIZE) return; // Pool is full.
      byteCount += Segment.SIZE;
      segment.next = next;//插入回收链表
      segment.pos = segment.limit = 0;//置空
      next = segment;
    }
  }
```

#### ByteString

```java
是一个不可变比特序列
实际就是一个 final byte[]，提供utf-8等处理方法

final byte[] data;
transient String utf8; // Lazily computed.

内部同时保存byte和utf8 string，节省互相转换性能
```

#### Buffer （okio读写核心）

```
内部存储segment head
```

##### 1.实现接口

clone接口

```java
  /** Returns a deep copy of this buffer. */
  @Override public Buffer clone() {
    Buffer result = new Buffer(); //new buffer
    if (size == 0) return result; 
	
    //复制一个shared true 的sagment
    result.head = head.sharedCopy();
    result.head.next = result.head.prev = result.head; //头尾指针自己指向自己
    for (Segment s = head.next; s != head; s = s.next) { //循环插入                         
      result.head.prev.push(s.sharedCopy());
    }
    result.size = size;
    return result;
  }
```

BufferedSource, BufferedSink接口

#### 超时机制

```java
AsyncTimeout （异步，用于socket）
```

```

```



