[TOC]

对于OkHttp的使用不再赘述，本篇讲述OkHttp的基本流程。


以下面的代码为例(版本 3.2)

```
        OkHttpClient okHttpClient;
        
        Request request = new Request.Builder()
                .url(url)
                .build();
        //同步
        Response response = okHttpClient.newCall(request).execute();
        
        if(response. isSuccessful()) {
        
        }
        
        //异步
        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                e.printStackTrace();
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (response.isSuccessful()) {
                    String resultBody = response.body().string();
                    JSONObject resultJson = null;
                    ...
                }
            }
        });
```

以GET方法为例，第一种为同步GET方法，第二种为异步GET方法。

### 1.构建Request

当我们构建好Request后调用OkHttpClient的newCall方法，会得到一个RealCall的类，RealCall中封装了Request、HttpEngine和OkHttpClient。


```
final class RealCall implements Call {
  private final OkHttpClient client;

  // Guarded by this.
  private boolean executed;
  volatile boolean canceled;

  /** The application's original request unadulterated by redirects or auth headers. */
  Request originalRequest;
  HttpEngine engine;

  protected RealCall(OkHttpClient client, Request originalRequest) {
    this.client = client;
    this.originalRequest = originalRequest;
  }
  
    @Override public Request request() {
    return originalRequest;
  }

  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain(false);
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
  
    @Override public void enqueue(Callback responseCallback) {
    enqueue(responseCallback, false);
  }

  void enqueue(Callback responseCallback, boolean forWebSocket) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
  }
  ...
```

这是一个很重要的类，里面封装了execute、enqueue、cancel等方法。

### 2.execute与enqueue

**execute方法**

先分析execute方法，可以看到同一个Request不能执行两次，然后通过OkHttpClient的Dispatcher类对象执行，最后通过getResponseWithInterceptorChain方法得到Response。


**enqueue方法**

```
  @Override public void enqueue(Callback responseCallback) {
    enqueue(responseCallback, false);
  }

  void enqueue(Callback responseCallback, boolean forWebSocket) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
  }
```

enque方法也是调用了OkHttpClient的Dispatcher，将request和callback封装成了AsyncCall。

**AsyncCall类**

```
  final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    private final boolean forWebSocket;

    private AsyncCall(Callback responseCallback, boolean forWebSocket) {
      super("OkHttp %s", originalRequest.url().toString());
      this.responseCallback = responseCallback;
      this.forWebSocket = forWebSocket;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    Object tag() {
      return originalRequest.tag();
    }

    void cancel() {
      RealCall.this.cancel();
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain(forWebSocket);
        if (canceled) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

```
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = String.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```

可见AsyncCall是一个Runnable对象，同样调用了getResponseWithInterceptorChain方法。

总而言之，OkHttp的请求都和Dispatcher有关，而getResponseWithInterceptorChain设计到了结果返回前的拦截。

### 3.Dispatcher

```
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  
    public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
  
    private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }

  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }

  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** Used by {@code Call#execute} to signal completion. */
  synchronized void finished(Call call) {
    if (!runningSyncCalls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
  }
  
    /** Returns the number of running calls that share a host with {@code call}. */
  private int runningCallsForHost(AsyncCall call) {
    int result = 0;
    for (AsyncCall c : runningAsyncCalls) {
      if (c.host().equals(call.host())) result++;
    }
    return result;
  }

```

上面的代码中列出了一些比较重要的方法和变量。

我们看到executed方法实际上RealCall添加至runningSyncCalls，enqueue方法将请求添加至readyAsyncCalls，如果超过了最大请求的数量，先放在readyAsyncCalls中。不过额外通过线程池执行了这个AsyncCall，不要忘了AsyncCall实际也是一个Runnable。

值得注意的是，enqueue方法会先判断runningAsyncCalls的大小是否超过maxRequests，通过runningCallsForHost方法超找当前正在执行的相同host的call的总数是否超过maxRequestsPerHost。

Dispatcher的maxRequests默认是64，也就是OkHttp默认最大支持64个请求，maxRequestsPerHost默认是5，同一个host最大有5个请求。当设置这两个值后对调用promoteCalls方法对队列进行整理。这部分不是特别重要。

重要的是runningSyncCalls 、runningAsyncCalls和readyAsyncCalls。他们的类型是ArrayDeque。分别代表了正在执行的同步请求队列，正在执行的异步请求队列，等待执行的异步请求队列。

异步请求队列的执行在核心线程0，最大线程数Integer.MAX_VALUE,队列类型是[SynchronousQueue](https://github.com/sparkfengbo/AndroidNotes/blob/master/Android/%E7%BA%BF%E7%A8%8B%E4%B8%8E%E7%BA%BF%E7%A8%8B%E6%B1%A0/Android%E7%BA%BF%E7%A8%8B%E6%B1%A0.md)的线程池。

讲到这里是不是需要再次补充一下[Java的容器知识](https://github.com/sparkfengbo/AndroidNotes/blob/master/Java/Java%E4%B8%AD%E5%B8%B8%E7%94%A8%E7%9A%84%E5%AE%B9%E5%99%A8%E7%B1%BB.md)呢？ ：）

可以看到Dispatcher其实是一个很轻的类，主要是将请求添加和移除队列或线程池而已。记得上小结的getResponseWithInterceptorChain吗？接下来我们看看getResponseWithInterceptorChain做了什么.

### 4.拦截器Interceptor

通过上面的几个章节我们看到不管是同步的请求（直接调用getResponseWithInterceptorChain方法）还是异步请求（放在线程池中调用run方法而run方法还会调用getResponseWithInterceptorChain方法），都和getResponseWithInterceptorChain方法有关。

```
  private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
    Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
    return chain.proceed(originalRequest);
  }
  

```

可以看到调用了ApplicationInterceptorChain的proceed方法

```
  class ApplicationInterceptorChain implements Interceptor.Chain {
    private final int index;
    private final Request request;
    private final boolean forWebSocket;

    ApplicationInterceptorChain(int index, Request request, boolean forWebSocket) {
      this.index = index;
      this.request = request;
      this.forWebSocket = forWebSocket;
    }

    @Override public Connection connection() {
      return null;
    }

    @Override public Request request() {
      return request;
    }

    @Override public Response proceed(Request request) throws IOException {
      // If there's another interceptor in the chain, call that.
      if (index < client.interceptors().size()) {
        Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
        Interceptor interceptor = client.interceptors().get(index);
        Response interceptedResponse = interceptor.intercept(chain);

        if (interceptedResponse == null) {
          throw new NullPointerException("application interceptor " + interceptor
              + " returned null");
        }

        return interceptedResponse;
      }

      // No more interceptors. Do HTTP.
      return getResponse(request, forWebSocket);
    }
  }
```

proceed方法会先去OkHttpClient中寻找nterceptor并调用intercept方法。默认的OkHttpClient中是不包含Interceptor的，用户需要根据自己的需求添加。

如果拦截了，那么直接返回拦截器对请求处理返回的Response。

------

关于拦截器参考 [OkHttp Wiki](https://github.com/square/okhttp/wiki/Interceptors)

>拦截机制能够监控、修改、重试请求。
>

下面是Wiki上的一个例子，

```
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```

在拦截器的intercept中必须调用chain.proceed(request)，chain.proceed是实际产生Response的地方。

其实通过ApplicationInterceptorChain的proceed方法也可以看出 拦截器是支持链式调用的，也就是有先后顺序。

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/glide/oksttp2.png?raw=true)

拦截器分两类

- 1.Application Interceptors
- 2.Network Interceptors

添加方法不同。

**添加 Application Interceptor**

```
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

  Url `http://www.publicobject.com/helloworld.txt`会重定向到 `https://publicobject.com/helloworld.txt`,OkHttp可以自动重定向，那么添加的LoggingInterceptor会被执行**一次**然后通过chain.proceed()的调用得到Response。LoggingInterceptor打印的log如下：


```
INFO: Sending request http://www.publicobject.com/helloworld.txt on null
User-Agent: OkHttp Example

INFO: Received response for https://publicobject.com/helloworld.txt in 1179.7ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

可以看到log上的url发生了变化。


**添加 Network Interceptor**

使用addNetworkInterceptor代替addInterceptor

```
OkHttpClient client = new OkHttpClient.Builder()
    .addNetworkInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

当我们使用上述代码运行后，打印的log如下

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on Connection{www.publicobject.com:80, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=none protocol=http/1.1}
User-Agent: OkHttp Example
Host: www.publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for http://www.publicobject.com/helloworld.txt in 115.6ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/html
Content-Length: 193
Connection: keep-alive
Location: https://publicobject.com/helloworld.txt

INFO: Sending request https://publicobject.com/helloworld.txt on Connection{publicobject.com:443, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA protocol=http/1.1}
User-Agent: OkHttp Example
Host: publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for https://publicobject.com/helloworld.txt in 80.9ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

可以看到上面的log打印了更多的信息， Network interceptor的chain的connection不是null，可以用来询问（interrogate）IP地址和 TLS 配置。

两种 拦截器各有特点

**Application interceptors**

- 不用担心请求过程中间的重定向和重试所返回的Response
- 总会调用一次
- 关注应用最初的意图，不用关心OkHttp添加的类似`If-None-Match`这样的头部字段
- Permitted to short-circuit and not call Chain.proceed().
- Permitted to retry and make multiple calls to Chain.proceed()

**Network Interceptors**

- 能够操作中间得到的Response
- 不会被cache的Response去调用，而Application interceptor就会
- Observe the data just as it will be transmitted over the network.
- Access to the Connection that carries the request.

-------

主要的作用就是对Request和Response做拦截处理。


### 5. RealCall的getResponse方法

实际的发送请求和得到相应是在RealCall的getResponse方法进行的，getResponse非常长。

```
  /**
   * Performs the request and returns the response. May return null if this call was canceled.
   */
  Response getResponse(Request request, boolean forWebSocket) throws IOException {
    // Copy body metadata to the appropriate request headers.
    RequestBody body = request.body();
    if (body != null) {
      Request.Builder requestBuilder = request.newBuilder();

      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }

      request = requestBuilder.build();
    }

    // Create the initial HTTP engine. Retries and redirects need new engine for each attempt.
    engine = new HttpEngine(client, request, false, false, forWebSocket, null, null, null);

    int followUpCount = 0;
    while (true) {
      if (canceled) {
        engine.releaseStreamAllocation();
        throw new IOException("Canceled");
      }

      boolean releaseConnection = true;
      try {
        engine.sendRequest();
        engine.readResponse();
        releaseConnection = false;
      } catch (RequestException e) {
        // The attempt to interpret the request failed. Give up.
        throw e.getCause();
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        HttpEngine retryEngine = engine.recover(e.getLastConnectException(), null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }
        // Give up; recovery is not possible.
        throw e.getLastConnectException();
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        HttpEngine retryEngine = engine.recover(e, null);
        if (retryEngine != null) {
          releaseConnection = false;
          engine = retryEngine;
          continue;
        }

        // Give up; recovery is not possible.
        throw e;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          StreamAllocation streamAllocation = engine.close();
          streamAllocation.release();
        }
      }

      Response response = engine.getResponse();
      Request followUp = engine.followUpRequest();

      if (followUp == null) {
        if (!forWebSocket) {
          engine.releaseStreamAllocation();
        }
        return response;
      }

      StreamAllocation streamAllocation = engine.close();

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (!engine.sameConnection(followUp.url())) {
        streamAllocation.release();
        streamAllocation = null;
      }

      request = followUp;
      engine = new HttpEngine(client, request, false, false, forWebSocket, streamAllocation, null,
          response);
    }
  }
```
分析代码，getResponse方法会先在Request的基础上添加一些文件头。

然后创建一个HttpEngine（竟然每次都是重新创建），然后进入一个无线循环。

然后调用HttpEngine的sendRequest方法和readResponse。

之后，如果HttpEngine的followUpRequest()返回的Request是null，才会return，否则会试图重用连接，释放资源后构建新的HttpEngine再次发送请求。


### 6.HttpEngine的sendRequest

先看sendRequest方法,sendRequest方法还是比较长的。

```
  /**
   * Figures out what the response source will be, and opens a socket to that source if necessary.
   * Prepares the request headers and gets ready to start writing the request body if it exists.
   *
   * @throws RequestException if there was a problem with request setup. Unrecoverable.
   * @throws RouteException if the was a problem during connection via a specific route. Sometimes
   * recoverable. See {@link #recover}.
   * @throws IOException if there was a problem while making a request. Sometimes recoverable. See
   * {@link #recover(IOException)}.
   */
  public void sendRequest() throws RequestException, RouteException, IOException {
    if (cacheStrategy != null) return; // Already sent.
    if (httpStream != null) throw new IllegalStateException();

    Request request = networkRequest(userRequest);

    InternalCache responseCache = Internal.instance.internalCache(client);
    Response cacheCandidate = responseCache != null
        ? responseCache.get(request)
        : null;

    long now = System.currentTimeMillis();
    cacheStrategy = new CacheStrategy.Factory(now, request, cacheCandidate).get();
    networkRequest = cacheStrategy.networkRequest;
    cacheResponse = cacheStrategy.cacheResponse;

    if (responseCache != null) {
      responseCache.trackResponse(cacheStrategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      userResponse = new Response.Builder()
          .request(userRequest)
          .priorResponse(stripBody(priorResponse))
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_BODY)
          .build();
      return;
    }

    // If we don't need the network, we're done.
    if (networkRequest == null) {
      userResponse = cacheResponse.newBuilder()
          .request(userRequest)
          .priorResponse(stripBody(priorResponse))
          .cacheResponse(stripBody(cacheResponse))
          .build();
      userResponse = unzip(userResponse);
      return;
    }

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean success = false;
    try {
      httpStream = connect();
      httpStream.setHttpEngine(this);

      if (writeRequestHeadersEagerly()) {
        long contentLength = OkHeaders.contentLength(request);
        if (bufferRequestBody) {
          if (contentLength > Integer.MAX_VALUE) {
            throw new IllegalStateException("Use setFixedLengthStreamingMode() or "
                + "setChunkedStreamingMode() for requests larger than 2 GiB.");
          }

          if (contentLength != -1) {
            // Buffer a request body of a known length.
            httpStream.writeRequestHeaders(networkRequest);
            requestBodyOut = new RetryableSink((int) contentLength);
          } else {
            // Buffer a request body of an unknown length. Don't write request headers until the
            // entire body is ready; otherwise we can't set the Content-Length header correctly.
            requestBodyOut = new RetryableSink();
          }
        } else {
          httpStream.writeRequestHeaders(networkRequest);
          requestBodyOut = httpStream.createRequestBody(networkRequest, contentLength);
        }
      }
      success = true;
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (!success && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
  }

```
可以看到首先调用networkRequest方法。

networkRequest又添加了请求首部字段。比如默认的使用gzip压缩首部字段，连接使用Keep-Alive,这样能减少数据传输量，保持TCP连接。为请求添加cookie。不过OkHttp默认的CookieJar是空实现，如果想实现此功能还需要用户自行实现。

```
  private Request networkRequest(Request request) throws IOException {
    Request.Builder result = request.newBuilder();

    if (request.header("Host") == null) {
      result.header("Host", hostHeader(request.url(), false));
    }

    if (request.header("Connection") == null) {
      result.header("Connection", "Keep-Alive");
    }

    if (request.header("Accept-Encoding") == null) {
      transparentGzip = true;
      result.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = client.cookieJar().loadForRequest(request.url());
    if (!cookies.isEmpty()) {
      result.header("Cookie", cookieHeader(cookies));
    }

    if (request.header("User-Agent") == null) {
      result.header("User-Agent", Version.userAgent());
    }

    return result.build();
  }
```

sendRequest接下来调用InternalCache获取缓存的响应。

**InternalCache的初始化操作是在OkHttpClient的static方法中进行**


然后构建CacheStrategy，

通过Factory方法

```
CacheStrategy.java
    public Factory(long nowMillis, Request request, Response cacheResponse) {
      this.nowMillis = nowMillis;
      this.request = request;
      this.cacheResponse = cacheResponse;

      if (cacheResponse != null) {
        Headers headers = cacheResponse.headers();
        for (int i = 0, size = headers.size(); i < size; i++) {
          String fieldName = headers.name(i);
          String value = headers.value(i);
          if ("Date".equalsIgnoreCase(fieldName)) {
            servedDate = HttpDate.parse(value);
            servedDateString = value;
          } else if ("Expires".equalsIgnoreCase(fieldName)) {
            expires = HttpDate.parse(value);
          } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
            lastModified = HttpDate.parse(value);
            lastModifiedString = value;
          } else if ("ETag".equalsIgnoreCase(fieldName)) {
            etag = value;
          } else if ("Age".equalsIgnoreCase(fieldName)) {
            ageSeconds = HeaderParser.parseSeconds(value, -1);
          } else if (OkHeaders.SENT_MILLIS.equalsIgnoreCase(fieldName)) {
            sentRequestMillis = Long.parseLong(value);
          } else if (OkHeaders.RECEIVED_MILLIS.equalsIgnoreCase(fieldName)) {
            receivedResponseMillis = Long.parseLong(value);
          }
        }
      }
    }
```

Factory会根据报文头部记录一些信息，然后调用get方法

```
CacheStrategy.java
    /**
     * Returns a strategy to satisfy {@code request} using the a cached response {@code response}.
     */
    public CacheStrategy get() {
      CacheStrategy candidate = getCandidate();

      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // We're forbidden from using the network and the cache is insufficient.
        return new CacheStrategy(null, null);
      }

      return candidate;
    }
    
        /** Returns a strategy to use assuming the request can use the network. */
    private CacheStrategy getCandidate() {
      // No cached response.
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // Drop the cached response if it's missing a required handshake.
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // If this response shouldn't have been stored, it should never be used
      // as a response source. This check should be redundant as long as the
      // persistence store is well-behaved and the rules are constant.
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }

      CacheControl requestCaching = request.cacheControl();
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }

      long ageMillis = cacheResponseAge();
      long freshMillis = computeFreshnessLifetime();

      if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }

      long minFreshMillis = 0;
      if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }

      long maxStaleMillis = 0;
      CacheControl responseCaching = cacheResponse.cacheControl();
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }

      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());
      }

      Request.Builder conditionalRequestBuilder = request.newBuilder();

      if (etag != null) {
        conditionalRequestBuilder.header("If-None-Match", etag);
      } else if (lastModified != null) {
        conditionalRequestBuilder.header("If-Modified-Since", lastModifiedString);
      } else if (servedDate != null) {
        conditionalRequestBuilder.header("If-Modified-Since", servedDateString);
      }

      Request conditionalRequest = conditionalRequestBuilder.build();
      return hasConditions(conditionalRequest)
          ? new CacheStrategy(conditionalRequest, cacheResponse)
          : new CacheStrategy(conditionalRequest, null);
    }
    
```

总结就是根据OkHttp的策略创建CacheStrategy，CacheStrategy有两个成员。

```
public final class CacheStrategy {
  /** The request to send on the network, or null if this call doesn't use the network. */
  public final Request networkRequest;

  /** The cached response to return or validate; or null if this call doesn't use a cache. */
  public final Response cacheResponse;
}
```

然后根据得到的CacheStrategy去判断是否应该特殊处理。注意到OkHttp是有自动重连和重定位等功能的，所以在用户的一次请求中sendRequest会被调用很多次，priorResponse是将上一次请求得到的响应付给这次的响应中。


接下来看看connect方法

```

HttpEngine.java

  private HttpStream connect() throws RouteException, RequestException, IOException {
    boolean doExtensiveHealthChecks = !networkRequest.method().equals("GET");
    return streamAllocation.newStream(client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis(),
        client.retryOnConnectionFailure(), doExtensiveHealthChecks);
  }
```

```
StreamAllocation.java
  public HttpStream newStream(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws RouteException, IOException {
    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

      HttpStream resultStream;
      if (resultConnection.framedConnection != null) {
        resultStream = new Http2xStream(this, resultConnection.framedConnection);
      } else {
        resultConnection.socket().setSoTimeout(readTimeout);
        resultConnection.source.timeout().timeout(readTimeout, MILLISECONDS);
        resultConnection.sink.timeout().timeout(writeTimeout, MILLISECONDS);
        resultStream = new Http1xStream(this, resultConnection.source, resultConnection.sink);
      }

      synchronized (connectionPool) {
        stream = resultStream;
        return resultStream;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```
实际创建**Http2xStream**或**Http1xStream**。RetryableSink是利用OkIO将Http的请求提放在内存中进行缓存,通过writeRequestHeaders方法的调用，将请求头部写入。

请看一下StreamAllocation的findHealthyConnection和findConnection方法

```
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException, RouteException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Otherwise do a potentially-slow check to confirm that the pooled connection is still good.
      if (candidate.isHealthy(doExtensiveHealthChecks)) {
        return candidate;
      }

      connectionFailed(new IOException());
    }
  }
  
    private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException, RouteException {
    Route selectedRoute;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (stream != null) throw new IllegalStateException("stream != null");
      if (canceled) throw new IOException("Canceled");

      RealConnection allocatedConnection = this.connection;
      if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
        return allocatedConnection;
      }

      // Attempt to get a connection from the pool.
      RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);
      if (pooledConnection != null) {
        this.connection = pooledConnection;
        return pooledConnection;
      }

      selectedRoute = route;
    }

    if (selectedRoute == null) {
      selectedRoute = routeSelector.next();
      synchronized (connectionPool) {
        route = selectedRoute;
      }
    }
    RealConnection newConnection = new RealConnection(selectedRoute);
    acquire(newConnection);

    synchronized (connectionPool) {
      Internal.instance.put(connectionPool, newConnection);
      this.connection = newConnection;
      if (canceled) throw new IOException("Canceled");
    }

    newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.connectionSpecs(),
        connectionRetryEnabled);
    routeDatabase().connected(newConnection.route());

    return newConnection;
  }

```

可以看到也就是在这里调用了connect将连接打开。

```
RealConnection.java

  /** The low-level TCP socket. */
  private Socket rawSocket;
  
  
  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      List<ConnectionSpec> connectionSpecs, boolean connectionRetryEnabled) throws RouteException {
    if (protocol != null) throw new IllegalStateException("already connected");

    RouteException routeException = null;
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);
    Proxy proxy = route.proxy();
    Address address = route.address();

    if (route.address().sslSocketFactory() == null
        && !connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
      throw new RouteException(new UnknownServiceException(
          "CLEARTEXT communication not supported: " + connectionSpecs));
    }

    while (protocol == null) {
      try {
        rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
            ? address.socketFactory().createSocket()
            : new Socket(proxy);
        connectSocket(connectTimeout, readTimeout, writeTimeout, connectionSpecSelector);
      } catch (IOException e) {
        closeQuietly(socket);
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;

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
  }
  
    /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  private void connectSocket(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    rawSocket.setSoTimeout(readTimeout);
    try {
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      throw new ConnectException("Failed to connect to " + route.socketAddress());
    }
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));

    if (route.address().sslSocketFactory() != null) {
      connectTls(readTimeout, writeTimeout, connectionSpecSelector);
    } else {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
    }

    if (protocol == Protocol.SPDY_3 || protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // Framed connection timeouts are set per-stream.

      FramedConnection framedConnection = new FramedConnection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .protocol(protocol)
          .listener(this)
          .build();
      framedConnection.sendConnectionPreface();

      // Only assign the framed connection once the preface has been sent successfully.
      this.allocationLimit = framedConnection.maxConcurrentStreams();
      this.framedConnection = framedConnection;
    } else {
      this.allocationLimit = 1;
    }
  }
```

可以看到source和sink都是在这里创建的，复用了一个Socket连接。


### 7.HttpEngine的readResponse

再看readResponse方法, readResponse方法更长 
```
  /**
   * Flushes the remaining request header and body, parses the HTTP response headers and starts
   * reading the HTTP response body if it exists.
   */
  public void readResponse() throws IOException {
    if (userResponse != null) {
      return; // Already ready.
    }
    if (networkRequest == null && cacheResponse == null) {
      throw new IllegalStateException("call sendRequest() first!");
    }
    if (networkRequest == null) {
      return; // No network response to read.
    }

    Response networkResponse;

    if (forWebSocket) {
      httpStream.writeRequestHeaders(networkRequest);
      networkResponse = readNetworkResponse();
    } else if (!callerWritesRequestBody) {
      networkResponse = new NetworkInterceptorChain(0, networkRequest).proceed(networkRequest);
    } else {
      // Emit the request body's buffer so that everything is in requestBodyOut.
      if (bufferedRequestBody != null && bufferedRequestBody.buffer().size() > 0) {
        bufferedRequestBody.emit();
      }

      // Emit the request headers if we haven't yet. We might have just learned the Content-Length.
      if (sentRequestMillis == -1) {
        if (OkHeaders.contentLength(networkRequest) == -1
            && requestBodyOut instanceof RetryableSink) {
          long contentLength = ((RetryableSink) requestBodyOut).contentLength();
          networkRequest = networkRequest.newBuilder()
              .header("Content-Length", Long.toString(contentLength))
              .build();
        }
        httpStream.writeRequestHeaders(networkRequest);
      }

      // Write the request body to the socket.
      if (requestBodyOut != null) {
        if (bufferedRequestBody != null) {
          // This also closes the wrapped requestBodyOut.
          bufferedRequestBody.close();
        } else {
          requestBodyOut.close();
        }
        if (requestBodyOut instanceof RetryableSink) {
          httpStream.writeRequestBody((RetryableSink) requestBodyOut);
        }
      }

      networkResponse = readNetworkResponse();
    }

    receiveHeaders(networkResponse.headers());

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (validate(cacheResponse, networkResponse)) {
        userResponse = cacheResponse.newBuilder()
            .request(userRequest)
            .priorResponse(stripBody(priorResponse))
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();
        releaseStreamAllocation();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        InternalCache responseCache = Internal.instance.internalCache(client);
        responseCache.trackConditionalCacheHit();
        responseCache.update(cacheResponse, stripBody(userResponse));
        userResponse = unzip(userResponse);
        return;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    userResponse = networkResponse.newBuilder()
        .request(userRequest)
        .priorResponse(stripBody(priorResponse))
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (hasBody(userResponse)) {
      maybeCache();
      userResponse = unzip(cacheWritingResponse(storeRequest, userResponse));
    }
  }
  
    public void receiveHeaders(Headers headers) throws IOException {
    if (client.cookieJar() == CookieJar.NO_COOKIES) return;

    List<Cookie> cookies = Cookie.parseAll(userRequest.url(), headers);
    if (cookies.isEmpty()) return;

    client.cookieJar().saveFromResponse(userRequest.url(), cookies);
  }

```

先讲讲流程，如果sendRequest构建了请求失败或从缓存拿到Response的话会直接return，此时是保存在userResponse中。

可以看到如果为OkHttp设置了NetworkInterceptor会调用NetworkInterceptorChain的proceed方法，会对设置的NetworkInterceptor回调。

可以看到无论哪个分支都会调用readNetworkResponse方法。接下来调用receiveHeaders方法将cookie写入OkHttp的CookieJar中。最后构建userResponse。可见
readNetworkResponse非常重要。


```
HttpEngine.java

  private Response readNetworkResponse() throws IOException {
    httpStream.finishRequest();

    Response networkResponse = httpStream.readResponseHeaders()
        .request(networkRequest)
        .handshake(streamAllocation.connection().handshake())
        .header(OkHeaders.SENT_MILLIS, Long.toString(sentRequestMillis))
        .header(OkHeaders.RECEIVED_MILLIS, Long.toString(System.currentTimeMillis()))
        .build();

    if (!forWebSocket) {
      networkResponse = networkResponse.newBuilder()
          .body(httpStream.openResponseBody(networkResponse))
          .build();
    }

    if ("close".equalsIgnoreCase(networkResponse.request().header("Connection"))
        || "close".equalsIgnoreCase(networkResponse.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    return networkResponse;
  }
```


先调用httpStream.finishRequest方法，根据httpStream的类型，比如`Http2xStream`就会调用`stream.getSink().close();`.

很重要和很隐蔽的一行代码`httpStream.readResponseHeaders()`

我们以Http1xStream为例

```
Http1xStream

  @Override public Response.Builder readResponseHeaders() throws IOException {
    return readResponse();
  }
  
    /** Parses bytes of a response header from an HTTP transport. */
  public Response.Builder readResponse() throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
      throw new IllegalStateException("state: " + state);
    }

    try {
      while (true) {
        StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());

        Response.Builder responseBuilder = new Response.Builder()
            .protocol(statusLine.protocol)
            .code(statusLine.code)
            .message(statusLine.message)
            .headers(readHeaders());

        if (statusLine.code != HTTP_CONTINUE) {
          state = STATE_OPEN_RESPONSE_BODY;
          return responseBuilder;
        }
      }
    } catch (EOFException e) {
      // Provide more context if the server ends the stream before sending a response.
      IOException exception = new IOException("unexpected end of stream on " + streamAllocation);
      exception.initCause(e);
      throw exception;
    }
  }
```

可以看到readResponseHeaders方法实际调用了readResponse方法，看着是读响应报文的首部，其实是读取了响应报文。

看到这行代码，source实际上是在RealConnection创建的OkIO中的类，用来读取打开的流中的内容。

```
StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());
```

在使用后，在OkHttp中都会将流关闭。

至此，OkHttp的一次完整清流流程大致清晰了。