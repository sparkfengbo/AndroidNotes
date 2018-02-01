[TOC]


#### 1.连接池的实现

OkHttp的连接池通过ConnectionPool实现

```
public final class ConnectionPool {
  /**
   * Background threads are used to cleanup expired connections. There will be at most a single
   * thread running per connection pool. The thread pool executor permits the pool itself to be
   * garbage collected.
   */
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

  private final int maxIdleConnections;
  private final long keepAliveDurationNs;
    private final Deque<RealConnection> connections = new ArrayDeque<>();
  final RouteDatabase routeDatabase = new RouteDatabase();
  
    public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }

```

默认支持5个Idle连接，保留的存活时间是5分钟。

在HttpEngine的sendRequest方法中，会调用connect方法，实际上是调用StreamAllocation的newStream方法用来管理连接。

我们来看StreamAllocation的newStream方法

```
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

主要关注findConnection方法的`RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);`

实际上是从连接池中复用的。如果有可以复用的流就复用，没有就重新创建并塞到连接池中。

我们需要关注如何将流放到连接池，从连接池中如何根据Address取对应的连接。

**将流放到连接池中**

```
 synchronized (connectionPool) {
      Internal.instance.put(connectionPool, newConnection);
      this.connection = newConnection;
      if (canceled) throw new IOException("Canceled");
    }
```

调用

```
 Internal.instance = new Internal() {
      ...

	@Override 
	public RealConnection get(ConnectionPool pool, Address address,
          StreamAllocation streamAllocation, Route route) {
        return pool.get(address, streamAllocation, route);
      }
  
      @Override public void put(ConnectionPool pool, RealConnection connection) {
        pool.put(connection);
      }

```

```
ConnectionPool.java

  private final Deque<RealConnection> connections = new ArrayDeque<>();


    @Nullable 
    RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
  
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }

```

可以看到put方法实际上是将创建的连接放在了ArrayDeque中。

而get方法会调用RealConnection的`isEligible`方法判断是否符合Address、route

我们先看看Address的结构

```
public final class Address {
  final HttpUrl url;
  final Dns dns;
  final SocketFactory socketFactory;
  final Authenticator proxyAuthenticator;
  final List<Protocol> protocols;
  final List<ConnectionSpec> connectionSpecs;
  final ProxySelector proxySelector;
  final @Nullable Proxy proxy;
  final @Nullable SSLSocketFactory sslSocketFactory;
  final @Nullable HostnameVerifier hostnameVerifier;
  final @Nullable CertificatePinner certificatePinner;
  }
```
保存了请求的URL、DNS等和连接信息相关。

```
public final class Route {
  final Address address;
  final Proxy proxy;
  final InetSocketAddress inetSocketAddress;
}
```

Route类是具体的用来连接服务器的类。


>The concrete route used by a connection to reach an abstract origin server.
When creating a connection the client has many options:

>HTTP proxy: a proxy server may be explicitly configured for the client. Otherwise the {@linkplain java.net.ProxySelector proxy selector} is used. It may return multiple proxies to attempt.

>IP address: whether connecting directly to an origin server or a proxy, opening a socket requires an IP address. The DNS server may return multiple IP addresses to attempt.

>TLS configuration: which cipher suites and TLS versions to attempt with the HTTPS connection.

>Each route is a specific selection of these options.


```
RealConnection.java
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
  
  
```


```
 if (address.url().host().equals(this.route().address().url().host())) {
      return true; // This connection is a perfect match.
    }

```
关键是上面的代码，如果host相同就可以重用连接。

点到即止，如果太过深入，还得讲述OkIo、HTTP协议等内容。

------

>引用计数
在okhttp中，在高层代码的调用中，使用了类似于引用计数的方式跟踪Socket流的调用，这里的计数对象是StreamAllocation，它被反复执行aquire与release操作，这两个函数其实是在改变RealConnection中的List<Reference<StreamAllocation>> 的大小。（StreamAllocation.java）

```
public void acquire(RealConnection connection) {
  connection.allocations.add(new WeakReference<>(this));
}
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
```
>RealConnection是socket物理连接的包装，它里面维护了List<Reference<StreamAllocation>>的引用。List中StreamAllocation的数量也就是socket被引用的计数，如果计数为0的话，说明此连接没有被使用就是空闲的，需要通过下文的算法实现回收；如果计数不为0，则表示上层代码仍然引用，就不需要关闭连接。



------


### 2.重连机制


在RealCall的getResponse方法中会调用HttpEngine的recover方法进行重试，这个地方就是重连机制的出处，但此处不深入代码细节了，太过深入反而没太大意义。



附带几篇参考文章

- [Android网络编程（八）源码解析OkHttp后篇[复用连接池]](http://liuwangshu.cn/application/network/8-okhttp3-sourcecode2.html)
- [OkHttp3源码分析[复用连接池]](https://www.jianshu.com/p/92a61357164b)




