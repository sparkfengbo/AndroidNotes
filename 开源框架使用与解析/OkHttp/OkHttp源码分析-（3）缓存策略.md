[TOC]

关于缓存的使用 可以参考 [OkHttp的Wiki](https://github.com/square/okhttp/wiki)



[OkHttp Github](https://github.com/square/okhttp)上的sample中的CacheResponse。

```
  public CacheResponse(File cacheDirectory) throws Exception {
    int cacheSize = 10 * 1024 * 1024; // 10 MiB
    Cache cache = new Cache(cacheDirectory, cacheSize);

    client = new OkHttpClient.Builder()
        .cache(cache)
        .build();
  }
```

之前看了这篇文章[OkHttp3源码分析[缓存策略]](https://www.jianshu.com/p/9cebbbd0eeab),写的其实并不是很全面，不过还是可以参考一下。

### 1.设置缓存

通过OkHttpClient.Builder的cache方法，将OkHttpClient的成员变量cache进行设置。

我们看一下Cache的实现

```
public final class Cache implements Closeable, Flushable {
  final InternalCache internalCache = new InternalCache() {
    @Override public Response get(Request request) throws IOException {
      return Cache.this.get(request);
    }

    @Override public CacheRequest put(Response response) throws IOException {
      return Cache.this.put(response);
    }

    @Override public void remove(Request request) throws IOException {
      Cache.this.remove(request);
    }

    @Override public void update(Response cached, Response network) throws IOException {
      Cache.this.update(cached, network);
    }

    @Override public void trackConditionalCacheHit() {
      Cache.this.trackConditionalCacheHit();
    }

    @Override public void trackResponse(CacheStrategy cacheStrategy) {
      Cache.this.trackResponse(cacheStrategy);
    }
  };
    private final DiskLruCache cache;
    
      Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
  }
  
    private static String urlToKey(Request request) {
    return Util.md5Hex(request.url().toString());
  }
```

可以看到Cache中同样包含了InternalCache的变量，使用了DiskLruCache保存缓存。

看一下 Cache的**put**方法

```

  private static String urlToKey(Request request) {
    return Util.md5Hex(request.url().toString());
  }
  
    private CacheRequest put(Response response) throws IOException {
    String requestMethod = response.request().method();

    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    if (!requestMethod.equals("GET")) {
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }

    if (OkHeaders.hasVaryAll(response)) {
      return null;
    }

    Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(urlToKey(response.request()));
      if (editor == null) {
        return null;
      }
      entry.writeTo(editor);
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
  }
```
可以看到如果请求方法不是get的话是不会缓存的。创建了一个Entry对象，以url的MD5为key写入了DiskLruCache中，返回一个CacheRequestImpl对象。

```
Entry

    public Entry(Response response) {
      this.url = response.request().url().toString();
      this.varyHeaders = OkHeaders.varyHeaders(response);
      this.requestMethod = response.request().method();
      this.protocol = response.protocol();
      this.code = response.code();
      this.message = response.message();
      this.responseHeaders = response.headers();
      this.handshake = response.handshake();
    }
```

*比较好奇的是为什么只缓存了部分内容，没有缓存ResponseBody呢*


再看一下CacheRequestImpl的实现

```
    public CacheRequestImpl(final DiskLruCache.Editor editor) throws IOException {
      this.editor = editor;
      this.cacheOut = editor.newSink(ENTRY_BODY);
      this.body = new ForwardingSink(cacheOut) {
        @Override public void close() throws IOException {
          synchronized (Cache.this) {
            if (done) {
              return;
            }
            done = true;
            writeSuccessCount++;
          }
          super.close();
          editor.commit();
        }
      };
    }
```

再来看看Cache的**get**方法

```
  Response get(Request request) {
    String key = urlToKey(request);
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
      snapshot = cache.get(key);
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
      // Give up because the cache cannot be read.
      return null;
    }

    try {
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }

    Response response = entry.response(snapshot);

    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }

    return response;
  }
  
      public Response response(DiskLruCache.Snapshot snapshot) {
      String contentType = responseHeaders.get("Content-Type");
      String contentLength = responseHeaders.get("Content-Length");
      Request cacheRequest = new Request.Builder()
          .url(url)
          .method(requestMethod, null)
          .headers(varyHeaders)
          .build();
      return new Response.Builder()
          .request(cacheRequest)
          .protocol(protocol)
          .code(code)
          .message(message)
          .headers(responseHeaders)
          .body(new CacheResponseBody(snapshot, contentType, contentLength))
          .handshake(handshake)
          .build();
    }
  }
```

分析代码得知，首先根据key去拿DisLruCache的Snapshot。然后通过缓存的内容重新构建Entry，其实和put操作时保存的Entry的内容一样。

然后根据response方法构造Response。

在最后return的构造的Reponnse的Body实际是CacheResponseBody。看看CacheResponseBody的实现，是不是和CacheRequestImpl很像。

```
CacheResponseBody.java

    public CacheResponseBody(final DiskLruCache.Snapshot snapshot,
        String contentType, String contentLength) {
      this.snapshot = snapshot;
      this.contentType = contentType;
      this.contentLength = contentLength;

      Source source = snapshot.getSource(ENTRY_BODY);
      bodySource = Okio.buffer(new ForwardingSource(source) {
        @Override public void close() throws IOException {
          snapshot.close();
          super.close();
        }
      });
    }
```
实际在存取缓存的过程中不会直接调用OkHttpClient中cache对象的方法。而是直接调用Internal.instance对象的方法。Internal.instance在OkHttpClient的static方法进行了初始化。

```
  static {
    Internal.instance = new Internal() {
      @Override public void addLenient(Headers.Builder builder, String line) {
        builder.addLenient(line);
      }

      @Override public void addLenient(Headers.Builder builder, String name, String value) {
        builder.addLenient(name, value);
      }

      @Override public void setCache(OkHttpClient.Builder builder, InternalCache internalCache) {
        builder.setInternalCache(internalCache);
      }

      @Override public InternalCache internalCache(OkHttpClient client) {
        return client.internalCache();
      }

      @Override public boolean connectionBecameIdle(
          ConnectionPool pool, RealConnection connection) {
        return pool.connectionBecameIdle(connection);
      }

      @Override public RealConnection get(
          ConnectionPool pool, Address address, StreamAllocation streamAllocation) {
        return pool.get(address, streamAllocation);
      }

      @Override public void put(ConnectionPool pool, RealConnection connection) {
        pool.put(connection);
      }

      @Override public RouteDatabase routeDatabase(ConnectionPool connectionPool) {
        return connectionPool.routeDatabase;
      }

      @Override
      public void callEnqueue(Call call, Callback responseCallback, boolean forWebSocket) {
        ((RealCall) call).enqueue(responseCallback, forWebSocket);
      }

      @Override public StreamAllocation callEngineGetStreamAllocation(Call call) {
        return ((RealCall) call).engine.streamAllocation;
      }

      @Override
      public void apply(ConnectionSpec tlsConfiguration, SSLSocket sslSocket, boolean isFallback) {
        tlsConfiguration.apply(sslSocket, isFallback);
      }

      @Override public HttpUrl getHttpUrlChecked(String url)
          throws MalformedURLException, UnknownHostException {
        return HttpUrl.getChecked(url);
      }
    };
  }
```

在使用缓存时调用internalCache方法，实际调用client的internalCache方法。

而OkHttpClient方法

```
  InternalCache internalCache() {
    return cache != null ? cache.internalCache : internalCache;
  }
```

可以看到如果设置了Cache则先使用Cache，如果没有设置则使用设置的internalCache，如果都没设置的话就不能使用缓存了。

那么为什么同时存在两种Cache呢？

**我的理解是：** Cache在对外暴露的使用中还是使用Cache包裹的internalCache，但是Cache的实现时DiskLruCache，而如果想自行实现缓存机制的话就可以实现internalCache

接下来看看缓存的处理


### 2.put缓存


在HttpEngine的readResponse方法中，

```
  public void readResponse() throws IOException {
    Response networkResponse;
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
  
    private static Response stripBody(Response response) {
    return response != null && response.body() != null
        ? response.newBuilder().body(null).build()
        : response;
  }

```
当sendRequest方法执行找到缓存时，并且缓存还有效的情况下（validate方法），执行trackConditionalCacheHit方法增加了缓存的命中数，update更新缓存。

我们再回到Cache方法看看update怎么实现的

```
Cache.java

  private void update(Response cached, Response network) {
    Entry entry = new Entry(network);
    DiskLruCache.Snapshot snapshot = ((CacheResponseBody) cached.body()).snapshot;
    DiskLruCache.Editor editor = null;
    try {
      editor = snapshot.edit(); // Returns null if snapshot is not current.
      if (editor != null) {
        entry.writeTo(editor);
        editor.commit();
      }
    } catch (IOException e) {
      abortQuietly(editor);
    }
}

```

将新构建的userResponse通过stripBody方法去除body，更新缓存。

那么当没有缓存命中时，走到下面的逻辑

```
    if (hasBody(userResponse)) {
      maybeCache();
      userResponse = unzip(cacheWritingResponse(storeRequest, userResponse));
    }
    
      /**
   * Returns true if the response must have a (possibly 0-length) body. See RFC 2616 section 4.3.
   */
  public static boolean hasBody(Response response) {
    // HEAD requests never yield a body regardless of the response headers.
    if (response.request().method().equals("HEAD")) {
      return false;
    }

    int responseCode = response.code();
    if ((responseCode < HTTP_CONTINUE || responseCode >= 200)
        && responseCode != HTTP_NO_CONTENT
        && responseCode != HTTP_NOT_MODIFIED) {
      return true;
    }

    // If the Content-Length or Transfer-Encoding headers disagree with the
    // response code, the response is malformed. For best compatibility, we
    // honor the headers.
    if (OkHeaders.contentLength(response) != -1
        || "chunked".equalsIgnoreCase(response.header("Transfer-Encoding"))) {
      return true;
    }

    return false;
  }
  
    private void maybeCache() throws IOException {
    InternalCache responseCache = Internal.instance.internalCache(client);
    if (responseCache == null) return;

    // Should we cache this response for this request?
    if (!CacheStrategy.isCacheable(userResponse, networkRequest)) {
      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          responseCache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
      return;
    }

    // Offer this request to the cache.
    storeRequest = responseCache.put(stripBody(userResponse));
  }
  
    private Response cacheWritingResponse(final CacheRequest cacheRequest, Response response)
      throws IOException {
    // Some apps return a null body; for compatibility we treat that like a null cache request.
    if (cacheRequest == null) return response;
    Sink cacheBodyUnbuffered = cacheRequest.body();
    if (cacheBodyUnbuffered == null) return response;

    final BufferedSource source = response.body().source();
    final BufferedSink cacheBody = Okio.buffer(cacheBodyUnbuffered);

    Source cacheWritingSource = new Source() {
      boolean cacheRequestClosed;

      @Override public long read(Buffer sink, long byteCount) throws IOException {
        long bytesRead;
        try {
          bytesRead = source.read(sink, byteCount);
        } catch (IOException e) {
          if (!cacheRequestClosed) {
            cacheRequestClosed = true;
            cacheRequest.abort(); // Failed to write a complete cache response.
          }
          throw e;
        }

        if (bytesRead == -1) {
          if (!cacheRequestClosed) {
            cacheRequestClosed = true;
            cacheBody.close(); // The cache response is complete!
          }
          return -1;
        }

        sink.copyTo(cacheBody.buffer(), sink.size() - bytesRead, bytesRead);
        cacheBody.emitCompleteSegments();
        return bytesRead;
      }

      @Override public Timeout timeout() {
        return source.timeout();
      }

      @Override public void close() throws IOException {
        if (!cacheRequestClosed
            && !discard(this, HttpStream.DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
          cacheRequestClosed = true;
          cacheRequest.abort();
        }
        source.close();
      }
    };

    return response.newBuilder()
        .body(new RealResponseBody(response.headers(), Okio.buffer(cacheWritingSource)))
        .build();
  }
```
如果响应Response包含响应体，执行maybeCache

可以看到如果不能进行缓存或者缓存失效就直接remove掉，然后将去除body的userResponse缓存到InternalCache中。返回一个CacheRequest类型的storeRequest。

然后执行 cacheWritingResponse方法。这里得到的storeRequest 实际上是CacheRequestImpl。实际在cacheWritingResponse重写了报文体（RealResponseBody）

```
  private Response unzip(final Response response) throws IOException {
    if (!transparentGzip || !"gzip".equalsIgnoreCase(userResponse.header("Content-Encoding"))) {
      return response;
    }

    if (response.body() == null) {
      return response;
    }

    GzipSource responseBody = new GzipSource(response.body().source());
    Headers strippedHeaders = response.headers().newBuilder()
        .removeAll("Content-Encoding")
        .removeAll("Content-Length")
        .build();
    return response.newBuilder()
        .headers(strippedHeaders)
        .body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)))
        .build();
  }
```

响应报文的主体都是在unzip方法构建的。

具体的写入和OkHttp的DiskLruCache有关，可以参考[OkHttp3源码分析[DiskLruCache]](https://www.jianshu.com/p/23b8aa490a6b)

DiskLruCache会将响应报文的头部写入[Url MD5 Hex 值].0的文件中，将报文主体写入[Url MD5 Hex 值].1的文件中

### 3.get缓存


在HttpEngine中的sendRequest就已经进行了取缓存的操作。



```
InternalCache responseCache = Internal.instance.internalCache(client);
    Response cacheCandidate = responseCache != null
        ? responseCache.get(request)
        : null;
```

通过上面两小节的讲述，此处的内容很好理解了。


### 4.缓存策略

参考CacheStrategy构建过程，此处不再赘述了。