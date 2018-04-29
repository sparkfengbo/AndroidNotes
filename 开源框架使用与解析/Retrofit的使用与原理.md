
[TOC]

### 1.如何使用

- [Retrofit Github](https://github.com/square/retrofit)

- [深入浅出 Retrofit，这么牛逼的框架你们还不来看看？](https://segmentfault.com/a/1190000005638577)(文中介绍了Retrofit的使用、Retrofit的原理和Retrofit的小技巧和进阶使用，使得细读)
- [Android：Retrofit 与 RxJava联合使用大合集（含实例教程）！](http://blog.csdn.net/carson_ho/article/details/79125101)
- [这是一份很详细的 Retrofit 2.0 使用教程（含实例讲解）](http://blog.csdn.net/carson_ho/article/details/73732076)


--------

**如何使用的最简单的例子(摘自参考文章)**

举个最简单的例子

```
GitHubService.java

public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

Url 是 https://api.github.com/users/{user}/repos

```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);

Call<List<Repo>> repos = service.listRepos("octocat");

// 同步调用
Response<List<Repo>> data = repos.execute(); 

// 异步调用
repos.enqueue(new Callback<List<Repo>>() {
            @Override
            public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
                List<Repo> data = response.body();
            }

            @Override
            public void onFailure(Call<List<Repo>> call, Throwable t) {
                t.printStackTrace();
            }
        });

```

### 2.最最重要的原理


- 1.Java的动态代理有关
- 2.Java注解

根据上面的代码看流程

Retrofit的create方法

```
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    //这里返回一个 service 的代理对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            //DefaultMethod 是 Java 8 的概念，是定义在 interface 当中的有实现的方法
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //每一个接口最终实例化成一个 ServiceMethod，并且会缓存
            ServiceMethod serviceMethod = loadServiceMethod(method);
            
            //由此可见 Retrofit 与 OkHttp 完全耦合，不可分割
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //下面这一句当中会发起请求，并解析服务端返回的结果
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

validateServiceInterface方法验证service参数是否是interface类型

```
Utils.java

  static <T> void validateServiceInterface(Class<T> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }
    // Prevent API interfaces from extending other interfaces. This not only avoids a bug in
    // Android (http://b.android.com/58753) but it forces composition of API declarations which is
    // the recommended pattern.
    if (service.getInterfaces().length > 0) {
      throw new IllegalArgumentException("API interfaces must not extend other interfaces.");
    }
}
```

之后直接调用了java reflect包下的动态反射方法，出入一个ClassLoader对象，传入一个匿名数组，包含interface类型的数组，这也是为什么一定要验证service是否是interface类型，最后传入一个匿名类InvocationHandler。

我们需要重点关注一下这个匿名的InvocationHandler里面做了什么。

ServiceMethod类和loadServiceMethod处理的工作主要是使用Java的注解和注解处理器，处理GitHubService中使用注解所定义的url和参数等等。稍后会介绍。



封装一个OkHttpCall<T> implements Call<T>

```
public interface Call<T> extends Cloneable {
  //同步发起请求
  Response<T> execute() throws IOException;
  //异步发起请求，结果通过回调返回
  void enqueue(Callback<T> callback);
  boolean isExecuted();
  void cancel();
  boolean isCanceled();
  Call<T> clone();
  //返回原始请求
  Request request();
}
```
Call接口熟悉吗，和OkHttp是不是很像？


当我们调用

```
Call<List<Repo>> repos = service.listRepos("octocat");

// 同步调用
List<Repo> data = repos.execute(); 
```

repos实际就是执行了动态代理InvocationHandler的invoke方法所返回的OkHttpCall。

看一下OkHttpCall的execute方法

```
  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }
```
可以看到，实际上是使用OkHttp的Call去执行相关的请求处理。关于OkHttp的源码分析参考我的笔记。

有一点值得说明的就是parseResponse方法

```
OkHttpCall.java

  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
  
    /** Builds a method return value from an HTTP response body. */
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

最后调用的toResponse方法调用了responseConverter的convert。

这个responseConverter是一个Converter对象。

```
private final Converter<ResponseBody, R> responseConverter;
```

而我们在创建Retrofit所传入的Converter如下例所示，转化成我们想要的格式。

```
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
}
```

我们再回过头关注一下Retrofit的create方法中的最后一句。

```
return serviceMethod.callAdapter.adapt(okHttpCall);
```

默认情况下，如果我们不添加一些参数，在Retorfit进行build时，会加入一个DefaultCallAdapterFactory，而DefaultCallAdapterFactory对Call不会进行任何处理

```
Retrofit.java

public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```
```
Platform.java
  CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }
```

```
DefaultCallAdapterFactory.java

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }

    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return call;
      }
    };
  }
```


通过类似DefaultCallAdapterFactory的实现，实现CallAdapter接口，使得Retrofit与RxJava混合使用，这部分可以参考
[深入浅出 Retrofit，这么牛逼的框架你们还不来看看？](https://segmentfault.com/a/1190000005638577) 不过RxJava学习成本较高，加之我所接触的工程并不使用RxJava，所以这部分暂时忽略。


-----
看看ServiceMethod相关代码

```
Retrofit.java

  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

在Retrofit动态代理的invoke方法中，对调用loadServiceMethod方法，参数method实际上是GitHubService的listRepos方法。可以看到实际上做了缓存的操作，如果没有，通过ServiceMethod.Builder构建，那么传入的参数是什么呢？ 一个是Retrofit对象本身，另一个是method。

看一下Builder

```
	final Annotation[] methodAnnotations;
    final Annotation[][] parameterAnnotationsArray;
    final Type[] parameterTypes;	
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
    
```

如果看不懂这些方法的话可参考[Method详解](http://blog.csdn.net/zhangquanit/article/details/52927216)


看一下build方法

```
    public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      ...
      
      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      ...
      
      ...
      
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

     ...
      return new ServiceMethod<>(this);
    }
    private void parseMethodAnnotation(Annotation annotation) {
      ...
      if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      ...
    }
    
        private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      ...
      
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }

      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // Ensure the query string does not have any named parameters.
        String queryParams = value.substring(question + 1);
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
          throw methodError("URL query string \"%s\" must not have replace block. "
              + "For dynamic query parameters use @Query.", queryParams);
        }
      }

      this.relativeUrl = value;
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```

可以看到针对方法的每个注解都会调用parseMethodAnnotation处理注解内容。

具体的细节这里就不讨论了，都是Java 注解的内容。

需要关注一下请求时参数是怎么传递进去的。

```
Retrofit.java
create

ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
```

当我们调用

```
Call<List<Repo>> repos = service.listRepos("octocat");
```

这行代码时，会把octocat这样的参数传给OkHttpCall。

而调用Call<List<Repo>>的execute实际上调用OkHttpCall的execute

```
OkHttpCall.java

@Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }
  
    private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
  
  
```

```
ServiceMethod.java
  Request toRequest(@Nullable Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.build();
  }
```

实际都是通过ParameterHandler完成参数的填充。具体参考ServiceMethod吧，再细究下去没什么意义了。
