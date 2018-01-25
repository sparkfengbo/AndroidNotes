参考：

- [ Android图片加载框架最全解析（三），深入探究Glide的缓存机制](http://blog.csdn.net/guolin_blog/article/details/54895665)

加之自己屡着源码，写笔记。

------

Glide内存分为两种

- 1.内存缓存：主要作用是防止应用重复将图片数据读取到内存当中
- 2.存储缓存：主要作用是防止应用重复从网络或其他地方重复下载和读取数据

内容包含 1.Glide取图片数据的流程 2.两种缓存的get和put操作

#### 1.Glide取缓存的流程

在Engine的load方法中

```
	public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
      
        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        if (cached != null) {
            cb.onResourceReady(cached);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Loaded resource from cache", startTime, key);
            }
            return null;
        }

        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Loaded resource from active resources", startTime, key);
            }
            return null;
        }

        EngineJob current = jobs.get(key);
        if (current != null) {
            current.addCallback(cb);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Added to existing load", startTime, key);
            }
            return new LoadStatus(cb, current);
        }
        
        
        //分界点，此行代码以上是拿LruCache的内存缓存，此行代码以下是先去磁盘缓存，取不到去网络拿数据，然后保存在磁盘缓存上

        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);
        engineJob.start(runnable);

        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Started new load", startTime, key);
        }
        return new LoadStatus(cb, engineJob);
    }
```

在上面的方法中可以看到Glide取缓存的流程，首先通过loadFromCache方法去内存缓存，然后通过loadFromActiveResources取缓存，如果都没有，会创建从网络上下载图片的任务，下载图片，然后缓存。

可以看到不管是`loadFromCache `还是`loadFromActiveResources `都和key参数有关。

上篇文章分析过  fetcher实际上是HttpUrlFetcher，当我们传入String类型的Uri时，会被层层包装秤GlideUrl，HttpUrlFetcher的getId方法

```
	//HttpUrlFetcher.java
    @Override
    public String getId() {
        return glideUrl.getCacheKey();
    }

```

```
//GlideUrl.java
private final String stringUrl;

    public String getCacheKey() {
      return stringUrl != null ? stringUrl : url.toString();
    }
```

所以id实际上返回的是图片资源的Url，在构建EngineKey时传入了很多参数。EngineKey重写了equals、hashCode、toString方法，保证传入的参数不同，key不一样。得到的EngineKey就是缓存使用的key，改变任何一个参数，得到的key都不同。




####  2.内存缓存分析

##### 2.1内存缓存流程前析

在RequestManager的load方法中会调用loadGeneric方法，方法中会构建StreamStringLoader对象，会调用Glide的buildStreamModelLoader方法

```
Glide.java

    public static <T> ModelLoader<T, InputStream> buildStreamModelLoader(Class<T> modelClass, Context context) {
        return buildModelLoader(modelClass, InputStream.class, context);
    }
    
        public static <T, Y> ModelLoader<T, Y> buildModelLoader(Class<T> modelClass, Class<Y> resourceClass,
            Context context) {
         if (modelClass == null) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Unable to load null model, setting placeholder only");
            }
            return null;
        }
        return Glide.get(context).getLoaderFactory().buildModelLoader(modelClass, resourceClass);
    }
    
        public static Glide get(Context context) {
        if (glide == null) {
            synchronized (Glide.class) {
                if (glide == null) {
                    Context applicationContext = context.getApplicationContext();
                    List<GlideModule> modules = new ManifestParser(applicationContext).parse();

                    GlideBuilder builder = new GlideBuilder(applicationContext);
                    for (GlideModule module : modules) {
                        module.applyOptions(applicationContext, builder);
                    }
                    glide = builder.createGlide();
                    for (GlideModule module : modules) {
                        module.registerComponents(applicationContext, glide);
                    }
                }
            }
        }

        return glide;
    }
```

其实Glide对象是单例模式，在使用时就会通过GlideBuilder构建，下面是构建的具体逻辑。


```
GlideBuilder.java

    Glide createGlide() {
        if (sourceService == null) {
            final int cores = Math.max(1, Runtime.getRuntime().availableProcessors());
            sourceService = new FifoPriorityThreadPoolExecutor(cores);
        }
        if (diskCacheService == null) {
            diskCacheService = new FifoPriorityThreadPoolExecutor(1);
        }

        MemorySizeCalculator calculator = new MemorySizeCalculator(context);
        if (bitmapPool == null) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
                int size = calculator.getBitmapPoolSize();
                bitmapPool = new LruBitmapPool(size);
            } else {
                bitmapPool = new BitmapPoolAdapter();
            }
        }

        if (memoryCache == null) {
            memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
        }

        if (diskCacheFactory == null) {
            diskCacheFactory = new InternalCacheDiskCacheFactory(context);
        }

        if (engine == null) {
            engine = new Engine(memoryCache, diskCacheFactory, diskCacheService, sourceService);
        }

        if (decodeFormat == null) {
            decodeFormat = DecodeFormat.DEFAULT;
        }

        return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
    }
```

可以看到 这里面new出来一个 `LruResourceCache`和`InternalCacheDiskCacheFactory `对象，装进了Engine对象中。关于MemorySizeCalculator的内容请看 4.3小节。

在GlideBuilder的createGlide方法中，会创建一个LruBitmapPool对象，关于LruBitmapPool请看 4.5小结，这里需要知道LruBitmapPool使用LRU算法将Bitmap存储在HashMap中，和LruCache类似，只不过Glide对其进行了一些修改。

**LruResourceCache**继承自LruCache，实际就是内存缓存的类。这里不再重复LruCache的源码。

##### 2.2内存缓存get的流程

再次回到Engine的load方法

```
    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        
       ...
        
        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
       ...
       ...
    }
    
        private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }

        EngineResource<?> cached = getEngineResourceFromCache(key);
        if (cached != null) {
            cached.acquire();
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
        }
        return cached;
    }
    
        private EngineResource<?> getEngineResourceFromCache(Key key) {
        Resource<?> cached = cache.remove(key);

        final EngineResource result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            // Save an object allocation if we've cached an EngineResource (the typical case).
            result = (EngineResource) cached;
        } else {
            result = new EngineResource(cached, true /*isCacheable*/);
        }
        return result;
    }

```

可以看到loadFromCache会调用getEngineResourceFromCache方法，在cache中拿到缓存的资源，并从cache中删除，这个cache实际上是初始化Engine时的LruResourceCache类。

之后对EngineResource的引用自增，然后将这个资源放在类型是`Map<Key, WeakReference<EngineResource<?>>>`的activeResources中。存放的是资源的若引用。

如果在LruResourceCache中没有取到的话，执行loadFromActiveResources方法。

```
Engine.java
    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }

        EngineResource<?> active = null;
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
                active.acquire();
            } else {
                activeResources.remove(key);
            }
        }

        return active;
    }

```

**可以这样理解，内存缓存实际上是放在LruResourceCache中缓存，但是每当使用后就从LruResourceCache弹出，但是将引用保存在了activeResources的HashMap<Key, WeakReference<EngineResource<?>>>中，使用的是弱引用。目的是防止被LruCache回收掉，尽可能保留最久。**


##### 2.3 内存缓存put的流程

当缓存中没有命中图片时，会开启线程进行下载，在Engine的load方法中EngineJob执行EngineRunnable，

EngineRunnable在线程中执行

```
EngineRunnable.java

	private final EngineRunnableManager manager;

    @Override
    public void run() {
        if (isCancelled) {
            return;
        }

        Exception exception = null;
        Resource<?> resource = null;
        try {
            resource = decode();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Exception decoding", e);
            }
            exception = e;
        }

        if (isCancelled) {
            if (resource != null) {
                resource.recycle();
            }
            return;
        }

        if (resource == null) {
            onLoadFailed(exception);
        } else {
            onLoadComplete(resource);
        }
    }
    
    
    private void onLoadComplete(Resource resource) {
        manager.onResourceReady(resource);
    }

```

最终会调用manager的onResourceReady方法，manager其实就是在Engine的load中传递的EngineJob对象，

```
EngineJob.java

    @Override
    public void onResourceReady(final Resource<?> resource) {
        this.resource = resource;
        MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
    }

    private void handleResultOnMainThread() {
        if (isCancelled) {
            resource.recycle();
            return;
        } else if (cbs.isEmpty()) {
            throw new IllegalStateException("Received a resource without any callbacks to notify");
        }
        engineResource = engineResourceFactory.build(resource, isCacheable);
        hasResource = true;

        // Hold on to resource for duration of request so we don't recycle it in the middle of notifying if it
        // synchronously released by one of the callbacks.
        engineResource.acquire();
        listener.onEngineJobComplete(key, engineResource);

        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                cb.onResourceReady(engineResource);
            }
        }
        // Our request is complete, so we can release the resource.
        engineResource.release();
}
```

在handleResultOnMainThread中cb.onResourceReady实际上回调给了GUnericRequest的onResourceReady，会给ImageView设置BackGroundDrawble，正如上篇文章所述。

调用`listener.onEngineJobComplete(key, engineResource);`会走到Engine的`onEngineJobComplete `方法。

```
Engine.java

    @SuppressWarnings("unchecked")
    @Override
    public void onEngineJobComplete(Key key, EngineResource<?> resource) {
        Util.assertMainThread();
        // A null resource indicates that the load failed, usually due to an exception.
        if (resource != null) {
            resource.setResourceListener(key, this);

            if (resource.isCacheable()) {
                activeResources.put(key, new ResourceWeakReference(key, resource, getReferenceQueue()));
            }
        }
        // TODO: should this check that the engine job is still current?
        jobs.remove(key);
    }

    @Override
    public void onEngineJobCancelled(EngineJob engineJob, Key key) {
        Util.assertMainThread();
        EngineJob current = jobs.get(key);
        if (engineJob.equals(current)) {
            jobs.remove(key);
        }
    }

```

可以看到，从网络请求回来的图片都会放在activeResources中。


##### 2.4 activeResources与LruResourceCache cache的区别

此处有个疑问，`Map<Key, WeakReference<EngineResource<?>>>`类型的activeResources和实现MemoryCache接口的`LruResourceCache` 的cache有什么区别呢？

其实从名字可以判断，**activeResources保存的是当前正在使用的资源，而cache保存的是使用过被释放的资源**

在loadFromCache中，从cache取资源并remove，然后放在activeResources中。当资源从网络请求成功后，调用Engine的onEngineJobComplete方法，直接将资源放在activeResources中。但是当资源被释放时，会调用Engine的onResourceReleased方法。

```
Engine.java
    @Override
    public void onResourceReleased(Key cacheKey, EngineResource resource) {
        Util.assertMainThread();
        activeResources.remove(cacheKey);
        if (resource.isCacheable()) {
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
```

此时，会将资源从activeResources中remove，并放在cache中。

就像之前所说的那样

**内存缓存实际上是放在LruResourceCache和持有弱引用的HashMap中共同缓存，但是每当使用后就从LruResourceCache弹出，但是将引用保存在了activeResources的HashMap<Key, WeakReference<EngineResource<?>>>中，使用的是弱引用。目的是防止被LruCache回收掉，尽可能保留最久。因为LruCache是有最大容量的，而activeResources在没有GC前会一致保持资源的引用**，这么想想，咦，有点6诶


####  3.磁盘缓存分析

```
Glide.with(this)
     .load(url)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .into(imageView);
```

v3.7版本接收一下几个参数：

- DiskCacheStrategy.NONE： 表示不缓存任何内容。
- DiskCacheStrategy.SOURCE： 表示只缓存原始图片。
- DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。
- DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。

####  3.1 磁盘缓存的实现

磁盘缓存的实现实际上是使用了DiskLruCache,前面讲述过在GlideBuilder构造Engine对象是传入了InternalCacheDiskCacheFactory对象，InternalCacheDiskCacheFactory实际上是一个工厂类，InternalCacheDiskCacheFactory继承自DiskLruCacheFactory，提供构建DiskCache的方法

```
//InternalCacheDiskCacheFactory.java
public final class InternalCacheDiskCacheFactory extends DiskLruCacheFactory {

    public InternalCacheDiskCacheFactory(Context context) {
        this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR, DiskCache.Factory.DEFAULT_DISK_CACHE_SIZE);
    }

    public InternalCacheDiskCacheFactory(Context context, int diskCacheSize) {
        this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR, diskCacheSize);
    }

    public InternalCacheDiskCacheFactory(final Context context, final String diskCacheName, int diskCacheSize) {
        super(new CacheDirectoryGetter() {
            @Override
            public File getCacheDirectory() {
                File cacheDirectory = context.getCacheDir();
                if (cacheDirectory == null) {
                    return null;
                }
                if (diskCacheName != null) {
                    return new File(cacheDirectory, diskCacheName);
                }
                return cacheDirectory;
            }
        }, diskCacheSize);
    }
}
```

```
public interface DiskCache {

    /**
     * An interface for lazily creating a disk cache.
     */
    interface Factory {

        /** 250 MB of cache. */
        int DEFAULT_DISK_CACHE_SIZE = 250 * 1024 * 1024;
        String DEFAULT_DISK_CACHE_DIR = "image_manager_disk_cache";

        /**
         * Returns a new disk cache, or {@code null} if no disk cache could be created.
         */
        DiskCache build();
    }
}
```

可以看到默认的磁盘缓存大小是250MB，位置是应用的getCacheDir + “image_manager_disk_cache”。

接下来看一下父类DiskLruCacheFactory的方法

```
public class DiskLruCacheFactory implements DiskCache.Factory {
    @Override
    public DiskCache build() {
        File cacheDir = cacheDirectoryGetter.getCacheDirectory();

        if (cacheDir == null) {
            return null;
        }

        if (!cacheDir.mkdirs() && (!cacheDir.exists() || !cacheDir.isDirectory())) {
            return null;
        }

        return DiskLruCacheWrapper.get(cacheDir, diskCacheSize);
    }
}
```

可以看到从DiskLruCacheWrapper得到一个包装,实际上使用的是DiskLruCache

```
DiskLruCacheWrapper.java
    private static final int APP_VERSION = 1;
    private static final int VALUE_COUNT = 1;
    private static DiskLruCacheWrapper wrapper = null;

    private final DiskCacheWriteLocker writeLocker = new DiskCacheWriteLocker();
    private final SafeKeyGenerator safeKeyGenerator;
    private final File directory;
    private final int maxSize;
    private DiskLruCache diskLruCache;
    
        protected DiskLruCacheWrapper(File directory, int maxSize) {
        this.directory = directory;
        this.maxSize = maxSize;
        this.safeKeyGenerator = new SafeKeyGenerator();
    }

    private synchronized DiskLruCache getDiskCache() throws IOException {
        if (diskLruCache == null) {
            diskLruCache = DiskLruCache.open(directory, APP_VERSION, VALUE_COUNT, maxSize);
        }
        return diskLruCache;
    }
```

####  3.2 磁盘缓存的get实现

在EngineRunnale的run方法中会调用decode方法，在decode方法

```
EngineRunnale.java

    private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
            return decodeFromCache();
        } else {
            return decodeFromSource();
        }
    }

    private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Exception decoding result from cache: " + e);
            }
        }

        if (result == null) {
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
        private void onLoadFailed(Exception e) {
        if (isDecodingFromCache()) {
            stage = Stage.SOURCE;
            manager.submitForSource(this);
        } else {
            manager.onException(e);
        }
    }
```

除非load fail，否则都会走到decodeFromCache方法中

```
//EngineRunnable
    private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Exception decoding result from cache: " + e);
            }
        }

        if (result == null) {
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
```

都会调用DecodeJob中的方法

```
DecodeJob.java
    public Resource<Z> decodeResultFromCache() throws Exception {
        if (!diskCacheStrategy.cacheResult()) {
            return null;
        }

        long startTime = LogTime.getLogTime();
        Resource<T> transformed = loadFromCache(resultKey);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded transformed from cache", startTime);
        }
        startTime = LogTime.getLogTime();
        Resource<Z> result = transcode(transformed);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transcoded transformed from cache", startTime);
        }
        return result;
    }
    
        public Resource<Z> decodeSourceFromCache() throws Exception {
        if (!diskCacheStrategy.cacheSource()) {
            return null;
        }

        long startTime = LogTime.getLogTime();
        Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return transformEncodeAndTranscode(decoded);
    }
    
        private Resource<T> loadFromCache(Key key) throws IOException {
        File cacheFile = diskCacheProvider.getDiskCache().get(key);
        if (cacheFile == null) {
            return null;
        }

        Resource<T> result = null;
        try {
            result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
        } finally {
            if (result == null) {
                diskCacheProvider.getDiskCache().delete(key);
            }
        }
        return result;
    }
```

可以看到，在EngineRunnable中先去执行DecodeJob.java的decodeResultFromCache，如果拿不到缓存再执行decodeSourceFromCache方法。而DecodeJob.java的decodeResultFromCache会执行到loadFromCache，实际上是从磁盘缓存中取数据，这是**磁盘缓存的get实现**

####  3.3 磁盘缓存的put实现

接着上面的代码讲，如果磁盘缓存没有的话就会调用decodeSourceFromCache

```
DecodeJob.java
    public Resource<Z> decodeSourceFromCache() throws Exception {
        if (!diskCacheStrategy.cacheSource()) {
            return null;
        }

        long startTime = LogTime.getLogTime();
        Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return transformEncodeAndTranscode(decoded);
    }
```

这里又会调用一次loadFromCache，不过decodeSourceFromCache中使用的key和decodeResultFromCache不一样。

decodeResultFromCache中使用的key是在Engine的load方法中传递了很多参数构建的，而decodeSourceFromCache使用的key是OriginalKey，而这个OriginalKey就是

```
public Key getOriginalKey() {
        if (originalKey == null) {
            originalKey = new OriginalKey(id, signature);
        }
        return originalKey;
    }
```
参数少了很多。继续看decodeSourceFromCache方法的调用，


```
    private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
        long startTime = LogTime.getLogTime();
        Resource<T> transformed = transform(decoded);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transformed resource from source", startTime);
        }

        writeTransformedToCache(transformed);

        startTime = LogTime.getLogTime();
        Resource<Z> result = transcode(transformed);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transcoded transformed from source", startTime);
        }
        return result;
    }
    
        private void writeTransformedToCache(Resource<T> transformed) {
        if (transformed == null || !diskCacheStrategy.cacheResult()) {
            return;
        }
        long startTime = LogTime.getLogTime();
        SourceWriter<Resource<T>> writer = new SourceWriter<Resource<T>>(loadProvider.getEncoder(), transformed);
        diskCacheProvider.getDiskCache().put(resultKey, writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote transformed from source to cache", startTime);
        }
    }
```

可以看到这里虽然传递了一个Writer，但是实际上做的操作时把图片资源存在DiskLruCache中，使用的key是resultKey。

其实当我看到这里，我心中有一个疑问，在EngineRunnable的run方法都是去缓存的操作，什么时候将网络请求提交上去了？

我又分析了一下源码，其实EngineRunnable的run方法在获取不到图片时（缓存中没有资源）会执行两次。

```
    @Override
    public void run() {
        if (isCancelled) {
            return;
        }

        Exception exception = null;
        Resource<?> resource = null;
        try {
            resource = decode();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Exception decoding", e);
            }
            exception = e;
        }

        if (isCancelled) {
            if (resource != null) {
                resource.recycle();
            }
            return;
        }

        if (resource == null) {
            onLoadFailed(exception);
        } else {
            onLoadComplete(resource);
        }
    }
    
        private void onLoadFailed(Exception e) {
        if (isDecodingFromCache()) {
            stage = Stage.SOURCE;
            manager.submitForSource(this);
        } else {
            manager.onException(e);
        }
    }
```
第一次没有得到缓存内容，resource是null，会调用onLoadFailed，此时stage被赋值Stage.SOURCE，这里的manager实际上是EngineJob。

```
EngineJob.java
    @Override
    public void submitForSource(EngineRunnable runnable) {
        future = sourceService.submit(runnable);
    }
```
可以看到将这个runnable提交到了线程池中，这个sourceService实际上是在GlideBuidler中构建的，

```
final int cores = Math.max(1, Runtime.getRuntime().availableProcessors());
            sourceService = new FifoPriorityThreadPoolExecutor(cores);
```

那么在第二次调用run方法时stage由于是Stage.SOURCE，在decode方法会执行decodeFromSource方法。


```
Decoder.java
    public Resource<Z> decodeFromSource() throws Exception {
        Resource<T> decoded = decodeSource();
        return transformEncodeAndTranscode(decoded);
    }
    
        private Resource<T> decodeSource() throws Exception {
        Resource<T> decoded = null;
        try {
            long startTime = LogTime.getLogTime();
            final A data = fetcher.loadData(priority);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Fetched data", startTime);
            }
            if (isCancelled) {
                return null;
            }
            decoded = decodeFromSourceData(data);
        } finally {
            fetcher.cleanup();
        }
        return decoded;
    }
    
       private Resource<T> decodeFromSourceData(A data) throws IOException {
        final Resource<T> decoded;
        if (diskCacheStrategy.cacheSource()) {
            decoded = cacheAndDecodeSourceData(data);
        } else {
            long startTime = LogTime.getLogTime();
            decoded = loadProvider.getSourceDecoder().decode(data, width, height);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Decoded from source", startTime);
            }
        }
        return decoded;
    }
    
        private Resource<T> cacheAndDecodeSourceData(A data) throws IOException {
        long startTime = LogTime.getLogTime();
        SourceWriter<A> writer = new SourceWriter<A>(loadProvider.getSourceEncoder(), data);
        diskCacheProvider.getDiskCache().put(resultKey.getOriginalKey(), writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote source to cache", startTime);
        }

        startTime = LogTime.getLogTime();
        Resource<T> result = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE) && result != null) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return result;
    }
    
        private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
        long startTime = LogTime.getLogTime();
        Resource<T> transformed = transform(decoded);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transformed resource from source", startTime);
        }

        writeTransformedToCache(transformed);

        startTime = LogTime.getLogTime();
        Resource<Z> result = transcode(transformed);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transcoded transformed from source", startTime);
        }
        return result;
    }
    
        private void writeTransformedToCache(Resource<T> transformed) {
        if (transformed == null || !diskCacheStrategy.cacheResult()) {
            return;
        }
        long startTime = LogTime.getLogTime();
        SourceWriter<Resource<T>> writer = new SourceWriter<Resource<T>>(loadProvider.getEncoder(), transformed);
        diskCacheProvider.getDiskCache().put(resultKey, writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote transformed from source to cache", startTime);
        }
    }
```

可以看到

在`decodeFromSource -> decodeSource -> decodeFromSourceData -> cacheAndDecodeSourceData` 这个调用链中，以resultKey.getOriginalKey()为key缓存在磁盘中，但是 ！ `decodeFromSource -> transformEncodeAndTranscode ->writeTransformedToCache `中有会缓存一份同样的数据，只不过此次的key是resultKey，也就是说只要 Url和 signature以外的参数变了，即使是同一张图片，也会缓存一份。



------

#### 4.涉及的数据结构

##### 4.1 LruResourceCache

##### 4.2 InternalCacheDiskCacheFactory

##### 4.3 MemorySizeCalculator

MemorySizeCalculator的作用是计算当前设备可供Glide使用的缓存大小和Bitmap位图池的大小。

```
MemorySizeCalculator.java

    static final int BYTES_PER_ARGB_8888_PIXEL = 4;
    static final int MEMORY_CACHE_TARGET_SCREENS = 2;
    static final int BITMAP_POOL_TARGET_SCREENS = 4;
    static final float MAX_SIZE_MULTIPLIER = 0.4f;
    static final float LOW_MEMORY_MAX_SIZE_MULTIPLIER = 0.33f;

    MemorySizeCalculator(Context context, ActivityManager activityManager, ScreenDimensions screenDimensions) {
        this.context = context;
        final int maxSize = getMaxSize(activityManager);

        final int screenSize = screenDimensions.getWidthPixels() * screenDimensions.getHeightPixels()
                * BYTES_PER_ARGB_8888_PIXEL;

        int targetPoolSize = screenSize * BITMAP_POOL_TARGET_SCREENS;
        int targetMemoryCacheSize = screenSize * MEMORY_CACHE_TARGET_SCREENS;

        if (targetMemoryCacheSize + targetPoolSize <= maxSize) {
            memoryCacheSize = targetMemoryCacheSize;
            bitmapPoolSize = targetPoolSize;
        } else {
            int part = Math.round((float) maxSize / (BITMAP_POOL_TARGET_SCREENS + MEMORY_CACHE_TARGET_SCREENS));
            memoryCacheSize = part * MEMORY_CACHE_TARGET_SCREENS;
            bitmapPoolSize = part * BITMAP_POOL_TARGET_SCREENS;
        }

        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Calculated memory cache size: " + toMb(memoryCacheSize) + " pool size: " + toMb(bitmapPoolSize)
                    + " memory class limited? " + (targetMemoryCacheSize + targetPoolSize > maxSize) + " max size: "
                    + toMb(maxSize) + " memoryClass: " + activityManager.getMemoryClass() + " isLowMemoryDevice: "
                    + isLowMemoryDevice(activityManager));
        }
    }
    
        private static int getMaxSize(ActivityManager activityManager) {
        final int memoryClassBytes = activityManager.getMemoryClass() * 1024 * 1024;
        final boolean isLowMemoryDevice = isLowMemoryDevice(activityManager);
        return Math.round(memoryClassBytes
                * (isLowMemoryDevice ? LOW_MEMORY_MAX_SIZE_MULTIPLIER : MAX_SIZE_MULTIPLIER));
    }
    
    @TargetApi(Build.VERSION_CODES.KITKAT)
    private static boolean isLowMemoryDevice(ActivityManager activityManager) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            return activityManager.isLowRamDevice();
        } else {
            return Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB;
        }
    }
    
    
```

构造方法的第一行调用`getMaxSize`计算可供使用的内存大小，isLowMemoryDevice会判断设备是否大于4.4版本，大于4.4版本会调用ActivityManager的isLowRamDevice方法，如果是小内存的设备，就是可用内存的 0.33f相乘，如果是稍大一点就是与0.4f相乘。

接下来，screenSize是按 屏幕宽高 与ARGB_8888的颜色模式占用大小计算。

可以看到 targetPoolSize 就是4张全屏 argb8888位图大小。缓存池的大小就是2张全屏 argb8888位图的大小。

之后会对`memoryCacheSize`和`bitmapPoolSize`进行校正。


##### 4.4 FifoPriorityThreadPoolExecutor

##### 4.5 LruBitmapPool

LruBitmapPool是使用LRU算法对位图进行管理

```
//LruBitmapPool.java
    LruBitmapPool(int maxSize, LruPoolStrategy strategy, Set<Bitmap.Config> allowedConfigs) {
        this.initialMaxSize = maxSize;
        this.maxSize = maxSize;
        this.strategy = strategy;
        this.allowedConfigs = allowedConfigs;
        this.tracker = new NullBitmapTracker();
    }
    
        private synchronized void trimToSize(int size) {
        while (currentSize > size) {
            final Bitmap removed = strategy.removeLast();
            // TODO: This shouldn't ever happen, see #331.
            if (removed == null) {
                if (Log.isLoggable(TAG, Log.WARN)) {
                    Log.w(TAG, "Size mismatch, resetting");
                    dumpUnchecked();
                }
                currentSize = 0;
                return;
            }

            tracker.remove(removed);
            currentSize -= strategy.getSize(removed);
            removed.recycle();
            evictions++;
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Evicting bitmap=" + strategy.logBitmap(removed));
            }
            dump();
        }
    }
```
可以看到这里面的方法和LruCache很类似，我们注意到这里对图片真正进行操作的是strategy对象，它是LruPoolStrategy类型的接口，默认的实现类是`SizeConfigStrategy`

```
SizeConfigStrategy.java

@TargetApi(Build.VERSION_CODES.KITKAT)
public class SizeConfigStrategy implements LruPoolStrategy {
private final KeyPool keyPool = new KeyPool();
    private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<Key, Bitmap>();
    private final Map<Bitmap.Config, NavigableMap<Integer, Integer>> sortedSizes =
            new HashMap<Bitmap.Config, NavigableMap<Integer, Integer>>();
            
            
    @Override
    public void put(Bitmap bitmap) {
        int size = Util.getBitmapByteSize(bitmap);
        Key key = keyPool.get(size, bitmap.getConfig());

        groupedMap.put(key, bitmap);

        NavigableMap<Integer, Integer> sizes = getSizesForConfig(bitmap.getConfig());
        Integer current = sizes.get(key.size);
        sizes.put(key.size, current == null ? 1 : current + 1);
    }

    @Override
    public Bitmap get(int width, int height, Bitmap.Config config) {
        int size = Util.getBitmapByteSize(width, height, config);
        Key targetKey = keyPool.get(size, config);
        Key bestKey = findBestKey(targetKey, size, config);

        Bitmap result = groupedMap.get(bestKey);
        if (result != null) {
            // Decrement must be called before reconfigure.
            decrementBitmapOfSize(Util.getBitmapByteSize(result), result.getConfig());
            result.reconfigure(width, height,
                    result.getConfig() != null ? result.getConfig() : Bitmap.Config.ARGB_8888);
        }
        return result;
    }
```

可以看到将位图存储在了GroupedLinkedMap对象中。

```
//GroupedLinkedMap.java

class GroupedLinkedMap<K extends Poolable, V> {
    private final LinkedEntry<K, V> head = new LinkedEntry<K, V>();
    private final Map<K, LinkedEntry<K, V>> keyToEntry = new HashMap<K, LinkedEntry<K, V>>();

    public void put(K key, V value) {
        LinkedEntry<K, V> entry = keyToEntry.get(key);

        if (entry == null) {
            entry = new LinkedEntry<K, V>(key);
            makeTail(entry);
            keyToEntry.put(key, entry);
        } else {
            key.offer();
        }

        entry.add(value);
    }

    public V get(K key) {
        LinkedEntry<K, V> entry = keyToEntry.get(key);
        if (entry == null) {
            entry = new LinkedEntry<K, V>(key);
            keyToEntry.put(key, entry);
        } else {
            key.offer();
        }

        makeHead(entry);

        return entry.removeLast();
    }
    ...
    private static class LinkedEntry<K, V> {
        private final K key;
        private List<V> values;
        LinkedEntry<K, V> next;
        LinkedEntry<K, V> prev;

        // Used only for the first item in the list which we will treat specially and which will not contain a value.
        public LinkedEntry() {
            this(null);
        }

        public LinkedEntry(K key) {
            next = prev = this;
            this.key = key;
        }

        public V removeLast() {
            final int valueSize = size();
            return valueSize > 0 ? values.remove(valueSize - 1) : null;
        }

        public int size() {
            return values != null ? values.size() : 0;
        }

        public void add(V value) {
            if (values == null) {
                values = new ArrayList<V>();
            }
            values.add(value);
        }
    }
}    
```

##### 4.6 EngineResource

EngineResource实际上是对Resource<Z>进行封装，通过acquired进行引用计数，并通过listener回调资源是否可以被回收。

```
EngineResource.java
class EngineResource<Z> implements Resource<Z> {
 private final Resource<Z> resource;
    private final boolean isCacheable;
    private ResourceListener listener;
    private Key key;
    private int acquired;
    private boolean isRecycled;
    
    private final Resource<Z> resource;
    void acquire() {
        if (isRecycled) {
            throw new IllegalStateException("Cannot acquire a recycled resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call acquire on the main thread");
        }
        ++acquired;
    }

    /**
     * Decrements the number of consumers using the wrapped resource. Must be called on the main thread.
     *
     * <p>
     *     This must only be called when a consumer that called the {@link #acquire()} method is now done with the
     *     resource. Generally external users should never callthis method, the framework will take care of this for
     *     you.
     * </p>
     */
    void release() {
        if (acquired <= 0) {
            throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call release on the main thread");
        }
        if (--acquired == 0) {
            listener.onResourceReleased(key, this);
        }
    }
```


##### 4.7 Signature







