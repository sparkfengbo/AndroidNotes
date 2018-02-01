[TOC]

对于OkHttp的使用不再赘述，本篇讲述OkHttp的基本流程。

以下面的代码为例

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

##### 1.构建Request

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

##### 2.execute与enqueue

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

##### 3.Dispatcher