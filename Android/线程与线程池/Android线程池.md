[TOC]

因为创建和销毁线程是需要代价的，所以当任务需要频繁开启线程的时候就需要使用线程池去管理。

### 1.线程池的创建
比较常见的是

```
public static final Executor THREAD_POOL_EXECUTOR        
    = new ThreadPoolExecutor(
            int corePoolSize,                          
            int maximumPoolSize,                          
            long keepAliveTime,                          
            TimeUnit unit,                          
            BlockingQueue<Runnable> workQueue,                          
            ThreadFactory threadFactory
```
这个`THREAD_POOL_EXECUTOR`就是一个线程池，线程池都实现了ExecutorService的接口，定义了execute，shutdown，submit等方法。
下面解释一下参数的含义。

**corePoolSize**
`corePoolSize`表示核心线程数，`corePoolSize`数量的线程会一直存活，除非设置了`allowCoreThreadTimeOut`。

**maximumPoolSize**
`maximumPoolSize`是最大线程数量，`maximumPoolSize`一定是大于`corePoolSize`的，否则会报错。当当前线程数量大于`corePoolSize`小于`maximumPoolSize`，并且阻塞队列满了，则创建新线程。

**keepAliveTime**
`keepAliveTime`代表最大线程数量内，核心线程外的线程在空闲多久内不会被销毁，也就是超过这个时间就会销毁。

**unit**
`TimeUnit.SECONDS`代表`keepAliveTime`的时间单位。

**workQueue**
在任务执行前保存任务的队列，当当前线程池线程数量达到`corePoolSize`并且当前所有线程都处于活动状态，则execute提交的runnable保存到队列中。

**threadFactory**
当executor创建新线程的时候的工厂方法
例如：

```
private static final ThreadFactory sThreadFactory =  new ThreadFactory() {    
        private final AtomicInteger mCount = new AtomicInteger(1);    
        public Thread newThread(Runnable r) {        
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());    
        }
};
```
还有一种：

```
public static final Executor THREAD_POOL_EXECUTOR        
    = new ThreadPoolExecutor(
            int corePoolSize,                          
            int maximumPoolSize,                          
            long keepAliveTime,                          
            TimeUnit unit,                          
            BlockingQueue<Runnable> workQueue,                          
            ThreadFactory threadFactory,
            RejectedExcutionHandler handler
```
这个`handler`是`workQueue`满了之后对新加任务的处理策略，RejectedExcutionHandler是一个接口，包含
`void rejectedExecution(Runnable r, ThreadPoolExecutor executor);`方法，后面会讲到。

### 2.封装的线程池

不过，Java已经为我们封装了一些线程池，在`Executors`中包含很多静态方法。

- **newFixedThreadPool**

```
public static ExecutorService newFixedThreadPool(int nThreads) {    
        return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,  new LinkedBlockingQueue<Runnable>());
}
```
这个方法创建了一个数量固定的线程池，并且工作队列是一个无限制大小的LinkedBlockingQueue。最大线程数量和核心线程数量相同。

- **newSingleThreadExecutor**

```
public static ExecutorService newSingleThreadExecutor() {    
    return new FinalizableDelegatedExecutorService(
          new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS,                                new LinkedBlockingQueue<Runnable>()));
 }
```
`newSingleThreadExecutor`创建一个单线程，无限制的队列。所以队列的任务都是以顺序的方式执行。

- **newCachedThreadPool**

适合执行大量短时间异步任务的线程池。

```
public static ExecutorService newCachedThreadPool() {    
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,  new SynchronousQueue<Runnable>());
}
```
创建一个核心线程数量0，最大线程数量为Integer.MAX_VALUE的线程池，线程存活时间60s，任务队列为SynchronousQueue。

- **ScheduledThreadpoolExecutor**

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {    
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```
最终都会调用到

```
public ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory) {    
      super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS, new DelayedWorkQueue(), threadFactory);}
```
可以看到ScheduledThreadPoolExecutor是一个有固定数量的核心线程，最大线程数量是Integer.MAX_VALUE，如果线程执行完毕则会立即销毁。任务队列是DelayedWorkQueue。

通常`ScheduledThreadPoolExecutor`会定时执行任务，例如：
```
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
```
`initialDelay`是第一次执行延长的时间，`period`是执行周期的次数。

### 3.任务队列

- **ArrayBlockingQueue**

基于数组结构的有界队列，按FIFO规则排序。队列满的话，还有任务想进入队列，则调用拒绝策略。

- **LinkedBlockingQueue**

基于链表的无界队列，按FIFO排序，由于是无界的，所以采用此队列后线程池将忽略拒绝策略参数，还会忽略最大线程数的参数。

- **SynchronousQueue**

`SynchronousQueue`内部没有缓存，每一个 `insert`操作必须对应于另外一个线程的`remove`操作，反之亦然。你不能插入一个元素，除非有另外一个线程尝试着remove。
其实SynchronousQueue的思想就是传递的性质，有人取得时候我才能加，仅此而已。所以SynchronousQueue 执行`put()`的时候不会返回，除非有对应的`take()`, 但是像大小是1的LinkedBlockingQueue，put() 操作会立即返回。SynchronousQueue内部使用了大量的ReentrantLock。

在stackoverflow上的[回答](http://stackoverflow.com/questions/8591610/when-should-i-use-synchronousqueue)有助于理解。


- **PriorityBlockingQueue**

具有优先级队列的有界队列，可自定义优先级。


### 4.拒绝策略RejectedExcutionHandler

`AbortPolicy`、`CallerRunsPolicy`、`DiscardOldestPolicy`、`DiscardPolicy`都是在ThreadPoolExecutor中声明的。

- **AbortPolicy**

拒绝任务，抛出RejectedExecutionException异常。

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      throw new RejectedExecutionException("Task " + r.toString() + " rejected from " + e.toString());}
```

- **CallerRunsPolicy**


拒绝新任务进入，如果线程池没有关闭，那么新任务执行在调用`executor`的线程中直接执行。

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {        
                r.run();    
        }
}
```


- **DiscardOldestPolicy**
当线程池没有关闭时，抛弃最老的还没有执行的请求，然后执行`execute`。

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {   
       if (!e.isShutdown()) {        
          e.getQueue().poll();        
          e.execute(r);    
        }
}
```

- **DiscardPolicy** 

抛弃不能加入的任务，除此之外不做任何处理

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {}
```






-------


在AsyncTask中，核心线程数量是CPU数量+1.


```
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
```
最大线程数量是设置为2*CPU数量+1：


```
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
```

`sPoolWorkQueue`是任务的存储队列，在这里是用一个大小128的链表去存储的：


```
private static final BlockingQueue<Runnable> sPoolWorkQueue =        new LinkedBlockingQueue<Runnable>(128);
```