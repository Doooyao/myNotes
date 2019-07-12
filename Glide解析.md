#### Glide解析

#####BitmapPool //用于对bitmap的再利用 !!这个缓存的是无用bitmap,用于再利用,不是图片三级缓存中的

实现==> LruBitmapPool //Lru策略

​	|=> BitmapPoolAdapter //空缓存 无用

###### 1. LruBitmapPool介绍

1. 整理空间到指定大小

   ```java
   private synchronized void trimToSize(int size) {
       while (currentSize > size) { //当前内容大于指定大小
           final Bitmap removed = strategy.removeLast();//strategy:缓存策略 有几种 下面会提到
           // TODO: This shouldn't ever happen, see #331.
           if (removed == null) {
               if (Log.isLoggable(TAG, Log.WARN)) {
                   Log.w(TAG, "Size mismatch, resetting");
                   dumpUnchecked();
               }
               currentSize = 0;
               return;
           }

           tracker.remove(removed);//移除
           currentSize -= strategy.getSize(removed);//重新计算当前内容大小
           removed.recycle();//回收移除的bitmap
           evictions++;//移除次数 貌似是用来打log查看的 没啥用
           if (Log.isLoggable(TAG, Log.DEBUG)) {
               Log.d(TAG, "Evicting bitmap=" + strategy.logBitmap(removed));
           }
           dump();
       }
   }

   ====================
   public void trimMemory(int level) {//根据内存报警等级清空缓存或者清除一半缓存
           if (Log.isLoggable(TAG, Log.DEBUG)) {
               Log.d(TAG, "trimMemory, level=" + level);
           }
           if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_MODERATE) {
               clearMemory();
           } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
               trimToSize(maxSize / 2);
           }
       }

   ```

2. bitmap的存储和复用

   ```java
   ======================存放bitmap
   @Override
   public synchronized boolean put(Bitmap bitmap) {
       if (bitmap == null) {
           throw new NullPointerException("Bitmap must not be null");
       }
       if (!bitmap.isMutable() || strategy.getSize(bitmap) > maxSize || !allowedConfigs.contains(bitmap.getConfig())) {//bitmap不可改变或者大小大于最大空间或者不支持格式时不能放进来
           if (Log.isLoggable(TAG, Log.VERBOSE)) {
               Log.v(TAG, "Reject bitmap from pool"
                       + ", bitmap: " + strategy.logBitmap(bitmap)
                       + ", is mutable: " + bitmap.isMutable()
                       + ", is allowed config: " + allowedConfigs.contains(bitmap.getConfig()));
           }
           return false;
       }

       final int size = strategy.getSize(bitmap);
       strategy.put(bitmap);//添加bitmap
       tracker.add(bitmap);//对此bitmap进行追踪

       puts++;//数量+1
       currentSize += size;//当前大小+

       if (Log.isLoggable(TAG, Log.VERBOSE)) {
           Log.v(TAG, "Put bitmap in pool=" + strategy.logBitmap(bitmap));
       }
       dump();

       evict();//重新整理空间 因为有可能加入此bitmap后容量大于最大空间了 需要淘汰一部分需要淘汰的bitmap
       return true;
   }
   ```

   ```java
   //顾名思义 从缓存中拿出一个符合要求的 dirty 的bitmap 及可能有数据
   //get方法就是拿出来以后调用bitmap.eraseColor()涂成透明的

   @TargetApi(Build.VERSION_CODES.HONEYCOMB_MR1)
   @Override
   public synchronized Bitmap getDirty(int width, int height, Bitmap.Config config) {
       // Config will be null for non public config types, which can lead to transformations naively passing in
       // null as the requested config here. See issue #194.
       final Bitmap result = strategy.get(width, height, config != null ? config : DEFAULT_CONFIG);
       if (result == null) {//没有需要的
           if (Log.isLoggable(TAG, Log.DEBUG)) {
               Log.d(TAG, "Missing bitmap=" + strategy.logBitmap(width, height, config));
           }
           misses++;
       } else {//拿出并从缓存中移除
           hits++;
           currentSize -= strategy.getSize(result);
           tracker.remove(result);
           if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB_MR1) {
               result.setHasAlpha(true);
           }
       }
       if (Log.isLoggable(TAG, Log.VERBOSE)) {
           Log.v(TAG, "Get bitmap=" + strategy.logBitmap(width, height, config));
       }
       dump();

       return result;
   }
   ```

###### 2.BitmapTracker和LruPoolStrategy bitmap跟踪器和缓存策略

1. BitmapTracker   追踪调试等 暂时可以不用 

2. LruPoolStrategy //缓存策略=====> SizeConfigStrategy

   ​						|====> AttributeStrategy

   ​						|====> SizeStrategy

   只看SizeConfigStrategy  //glide在api>=19时使用的此策略

   ```java
   //匿名内部类key  
   重写hashcode toString 和equals 使判断相等时只以size和config为条件而不管内存地址是否真的是同一个对象 存储size和config
   static final class Key implements Poolable {
       private final KeyPool pool;//存储key的双向链表 最大化复用次数
       //尽量少new对象 少依靠gc回收

       private int size;
       private Bitmap.Config config;

       public Key(KeyPool pool) {
           this.pool = pool;
       }

       // Visible for testing.
       Key(KeyPool pool, int size, Bitmap.Config config) {
           this(pool);
           init(size, config);
       }

       public void init(int size, Bitmap.Config config) {
           this.size = size;
           this.config = config;
       }

       @Override
       public void offer() {
           pool.offer(this);
       }

       @Override
       public String toString() {
           return getBitmapString(size, config);
       }

       @Override
       public boolean equals(Object o) { //判断两个key是否相等
           if (o instanceof Key) {
               Key other = (Key) o;
               return size == other.size && (config == null ? other.config == null : config.equals(other.config));
           }
           return false;
       }

       @Override
       public int hashCode() { //hash
           int result = size;
           result = 31 * result + (config != null ? config.hashCode() : 0);
           return result;
       }
   }
   ```

   

   ```
   @Override //put方法 放一个bitmap到缓存里面
   public void put(Bitmap bitmap) {
       int size = Util.getBitmapByteSize(bitmap);
       Key key = keyPool.get(size, bitmap.getConfig());//获取key
       groupedMap.put(key, bitmap);//存储
       ==================
       记录符合要求大小的bitmap一共有几个
       NavigableMap<Integer, Integer> sizes = getSizesForConfig(bitmap.getConfig());
       Integer current = sizes.get(key.size);
       sizes.put(key.size, current == null ? 1 : current + 1);
   }
   ```

   ```
   @Override //get方法 获取符合条件的bitmap
   public Bitmap get(int width, int height, Bitmap.Config config) {
       int size = Util.getBitmapByteSize(width, height, config);
       Key targetKey = keyPool.get(size, config);
       Key bestKey = findBestKey(targetKey, size, config);//这个方法等下讲(获取最合适的key)

     	Bitmap result = groupedMap.get(bestKey); //获取bitmap
       if (result != null) {
           // Decrement must be called before reconfigure.
           decrementBitmapOfSize(Util.getBitmapByteSize(result), result.getConfig()); //库存-1
          	//重新配置bitmap
           result.reconfigure(width, height,
                   result.getConfig() != null ? result.getConfig() : Bitmap.Config.ARGB_8888);
       }
       return result;
   }
   ```
   第一个 findBestKey 用于查找最小的大于等于需要size的key

3. 接下来介绍用于实现lru的GroupedLinkedMap和用于记录剩余bitmap库存的NavigableMap

   3.1 NavigableMap

   glide中使用红黑树TreeMap 不多介绍了 可以自行查资料

   3.2 GroupedLinkedMap

   环型链表,每隔节点存储数组

   ====以下选自网路

   groupedMap是GroupedLinkedMap的实例，GroupedLinkedMap内部使用了一个名为head的链表，链表的key是由bitmap size和config构成的Key，value是一个由bitmap构成的链表。这样GroupedLinkedMap中的每个元素就相当于是一个组，这个组中的bitmap具有相同的size和config,对应的存储类实现就是GroupedLinkedMap中的LinkedEntry。同时，为了加快查找速度，GroupedLinkedMap中还有一个keyToEntry的Hashmap，将key和链表中的LinkedEntry对应起来。 在GroupedLinkedMap的Put和get方法中，会将操作元素对应所在的LinkedEntry在head链表中往前移动，由于链表的移动成本很低，因存取效率很高。

   3.3 KeyPool的作用

   为了避免每存取缓存信时候都构造一个新的key，因此这里使用了KeyPool来对最近使用过前20个的由bitmap的size和config构成的key做缓存，增加内存利用率。

#####MemoryCache// 三级缓存之内存缓存~ 

######LruResourceCache

继承自自己实现的LruCache 没有用系统的 LruCache通过linkHashMap实现

下面看一下三个方法

1. put方法添加一个元素(get方法就不写了 直接hashmap中get)

   ```
   public Y put(T key, Y item) {
       final int itemSize = getSize(item);//计算大小
       if (itemSize >= maxSize) { //单个图片大小就大于最大缓存了
           onItemEvicted(key, item); //直接抛弃并回调接口
           return null;
       }

       final Y result = cache.put(key, item); //放入缓存linkhashMap//hashmap.put 如果key重复 会返回key之前对应的value
       if (item != null) {
           currentSize += getSize(item); //size++
       }
       if (result != null) {
           // TODO: should we call onItemEvicted here?
           currentSize -= getSize(result); //size--
       }
       evict();//回收调整内存

       return result;
   }
   ```

2. trimToSize //调整内存size到指定值 ( evict就是直接调用此方法)  //循环移除头节点直到符合

   ```
   protected void trimToSize(int size) {
       Map.Entry<T, Y> last;
       while (currentSize > size) {
           last = cache.entrySet().iterator().next();
           final Y toRemove = last.getValue();
           currentSize -= getSize(toRemove);
           final T key = last.getKey();
           cache.remove(key);
           onItemEvicted(key, toRemove);
       }
   } 
   ```

##### Engine// 流程启动者  暂时不解释了 

#### 启动过程

先看一个简单的Glide的启动过程~

```
Glide.with(context)  //1
        .load(url) //2
        .asBitmap() //3
        .centerCrop() //4
        .into(imageView) //5
```

这里我们选择把图片直接加载到imageview

//分步看以上的操作

##### 1.with(context)

根据不同的context 会调用不同的get方法返回相应的requestmanager

这里选取FagmentActivity 另外 所有后台执行的都会使用applicationcontext调用

```
public RequestManager get(FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) { 
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        FragmentManager fm = activity.getSupportFragmentManager();
        return supportFragmentGet(activity, fm);//根据当前的activity和
    }
}
```

```
RequestManager supportFragmentGet(Context context, FragmentManager fm) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
    //使用fragment放入activity中,再对此fragment实现lifecycle  从而实现对activity的监听
    //保证一个activity只有一个监听用fragment
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
        current.setRequestManager(requestManager);
    }
    //一个监听用fragment只有一个requestManager
    return requestManager;//返回requestManager
}
```

######SupportRequestManagerFragment

添加进activity的fragment 监听生命周期

其中有一个treenode的属性 

```
public interface RequestManagerTreeNode {
    /**
     * Returns all descendant {@link RequestManager}s relative to the context of the current {@link RequestManager}.
     */
    Set<RequestManager> getDescendants();
}
```



###### RequestManager 这里介绍一下requestManager的结构和作用

下面是requestmanager中的属性

```
private final Context context;
    private final Lifecycle lifecycle; //生命周期监听
    private final RequestManagerTreeNode treeNode; //所有的子fragment的requestmanager 用于批量操作,没有用到,暴露给开发者
    private final RequestTracker requestTracker; //请求追踪
    private final Glide glide;
    private final OptionsApplier optionsApplier;
    private DefaultOptions options;
```

##### 2.load() 调用的requestmanager的load方法

根据不同的参数种类最后调用此方法 loadGeneric.load()

分开来看

```
private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
    ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
    ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
            Glide.buildFileDescriptorModelLoader(modelClass, context);
    if (modelClass != null && streamModelLoader == null && fileDescriptorModelLoader == null) {
        throw new IllegalArgumentException("Unknown type " + modelClass + ". You must provide a Model of a type for"
                + " which there is a registered ModelLoader, if you are using a custom model, you must first call"
                + " Glide#register with a ModelLoaderFactory for your custom model class");
    }

    return optionsApplier.apply(
            new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                    glide, requestTracker, lifecycle, optionsApplier));
}


```



