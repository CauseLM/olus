# OKHttp源码解析(一)--初阶

## 一、OKHTTP简介

> - 1.支持HTTP2/SPDY
> - 2.socket自动选择最好路线，并支持自动重连
> - 3.拥有自动维护的socket连接池，减少握手次数
> - 4.拥有队列线程池，轻松写并发
> - 5.拥有Interceptors轻松处理请求与响应（比如透明GZIP压缩）基于Headers的缓存策略

## 二、OKHTTP使用：

#### 1、GET请求



```cpp
  OkHttpClient client = new OkHttpClient();

  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

#### 2、POST请求



```tsx
   public static final MediaType JSON
    = MediaType.parse("application/json; charset=utf-8");

  OkHttpClient client = new OkHttpClient();

  RequestBody body = RequestBody.create(JSON, json);

  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();

  Response response = client.newCall(request).execute();

  return response.body().string();
}
```

## 三、OKHTTP源码流程分析

### (一)、OKHTTP  同步请求debug代码跟踪：



```cpp
  OkHttpClient client = new OkHttpClient();
  Request request = new Request.Builder()
      .url(url)
      .build();
  Response response = client.newCall(request).execute();
```

从上面代码所示，先是new了一个OKHttpClient对象。

**1、OKHttpClient类详解**

[OKHttpClient](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FOkHttpClient.java)类就比较简单了：

> - 1、里面包含了很多对象，其实OKhttp的很多功能模块都包装进这个类，让这个类单独提供对外的API，这种外观模式的设计十分的优雅。[外观模式](https://link.jianshu.com?t=http%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E5%A4%96%E8%A7%82%E6%A8%A1%E5%BC%8F)。
> - 2、而内部模块比较多，就使用了Builder模式(建造器模式)。[Builder模式(建造器模式)](https://link.jianshu.com?t=http%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E5%BB%BA%E9%80%A0%E8%80%85%E6%A8%A1%E5%BC%8F%3Ffromtitle%3D%E7%94%9F%E6%88%90%E5%99%A8%E6%A8%A1%E5%BC%8F%26fromid%3D7488582)
> - 3、它的方法只有一个：newCall.返回一个Call对象(一个准备好了的可以执行和取消的请求)。

而大家仔细读源码又会发现构造了OKHttpClient后又new了一个Rquest对象。那么咱们就来看下Request，说道Request又不得不提Response。所以咱们一起讲了

**2、Request、Response类详解**

[Request](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FRequest.java)      [Response](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FResponse.java)

> - 1、Request、Response分别抽象成请求和相应
> - 2、其中Request包括Headers和RequestBody，而RequestBody是abstract的，他的子类是有FormBody (表单提交的)和 MultipartBody(文件上传)，分别对应了两种不同的MIME类型
>    FormBody ："application/x-www-form-urlencoded"
>    MultipartBody："multipart/"+xxx.
> - 3、其中Response包括Headers和RequestBody，而ResponseBody是abstract的，所以他的子类也是有两个:RealResponseBody和CacheResponseBody,分别代表真实响应和缓存响应。
> - 4、由于RFC协议规定，所以所有的头部信息不是随便写的，request的header与response的header的标准都不同。具体的见 [List of HTTP header fields](https://link.jianshu.com?t=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FList_of_HTTP_header_fields)。OKHttp的封装类Request和Response为了应用程序编程方便，会把一些常用的Header信息专门提取出来，作为局部变量。比如contentType，contentLength，code,message,cacheControl,tag...它们其实都是以name-value对的形势，存储在网络请求的头部信息中。

根据从上面的GET请求，显示用builder构建了Request对象，然后执行了OKHttpClient.java的newCall方法，那么咱们就看看这个newCall里面都做什么操作？



```java
  /**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return new RealCall(this, request, false /* for web socket */);
  }
```

Call是个什么东西，那咱们看下Call这个类

**3、Call类详解**

[Call](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FCall.java): HTTP请求任务封装
 可以说我们能用到的操纵基本上都定义在这个接口里面了，所以也可以说这个类是OKHttp类的核心类了。我们可以通过Call对象来操作请求了。而Call接口内部提供了Factory[工厂方法模式](https://link.jianshu.com?t=http%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95%E6%A8%A1%E5%BC%8F)(将对象的创建延迟到工厂类的子类去进行，从而实现动态配置)
 Call接口提供了内部接口Factory(用于将对象的创建延迟到该工厂类的子类中进行，从而实现动态的配置).



```dart
/**
 * A call is a request that has been prepared for execution. A call can be canceled. As this object
 * represents a single request/response pair (stream), it cannot be executed twice.
 */
public interface Call extends Cloneable {
  /** Returns the original request that initiated this call. */
  Request request();

  /**
   * Invokes the request immediately, and blocks until the response can be processed or is in
   * error.
   *
   * <p>To avoid leaking resources callers should close the {@link Response} which in turn will
   * close the underlying {@link ResponseBody}.
   *
   * <pre>@{code
   *
   *   // ensure the response (and underlying response body) is closed
   *   try (Response response = client.newCall(request).execute()) {
   *     ...
   *   }
   *
   * }</pre>
   *
   * <p>The caller may read the response body with the response's {@link Response#body} method. To
   * avoid leaking resources callers must {@linkplain ResponseBody close the response body} or the
   * Response.
   *
   * <p>Note that transport-layer success (receiving a HTTP response code, headers and body) does
   * not necessarily indicate application-layer success: {@code response} may still indicate an
   * unhappy HTTP response code like 404 or 500.
   *
   * @throws IOException if the request could not be executed due to cancellation, a connectivity
   * problem or timeout. Because networks can fail during an exchange, it is possible that the
   * remote server accepted the request before the failure.
   * @throws IllegalStateException when the call has already been executed.
   */
  Response execute() throws IOException;

  /**
   * Schedules the request to be executed at some point in the future.
   *
   * <p>The {@link OkHttpClient#dispatcher dispatcher} defines when the request will run: usually
   * immediately unless there are several other requests currently being executed.
   *
   * <p>This client will later call back {@code responseCallback} with either an HTTP response or a
   * failure exception.
   *
   * @throws IllegalStateException when the call has already been executed.
   */
  void enqueue(Callback responseCallback);

  /** Cancels the request, if possible. Requests that are already complete cannot be canceled. */
  void cancel();

  /**
   * Returns true if this call has been either {@linkplain #execute() executed} or {@linkplain
   * #enqueue(Callback) enqueued}. It is an error to execute a call more than once.
   */
  boolean isExecuted();

  boolean isCanceled();

  /**
   * Create a new, identical call to this one which can be enqueued or executed even if this call
   * has already been.
   */
  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```

在源码中，OKHttpClient实现了Call.Factory接口，返回了一个RealCall对象。那我们就来看下RealCall这个类

**4、RealCall类详解**

[RealCall](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FRealCall.java)

> - 1、OkHttpClient的newCall方法里面new了RealCall的对象，但是RealCall的构造函数需要传入一个OKHttpClient对象和Request对象(PS：第三个参数false表示不是webSokcet).因此RealCall包装了Request对象。所以RealCall可以很方便地使用这两个对象。
> - 2、RealCall里面的两个关键方法是：execute 和 enqueue。分别用于同步和异步得执行网络请求。
> - 3、RealCall还有一个重要方法是:getResponseWithInterceptorChain，添加拦截器，通过拦截器可以将一个流式工作分解为可配置的分段流程，既增加了灵活性也实现了解耦，关键还可以自有配置，非常完美。

所以client.newCall(request).execute();实际上执行的是RealCall的execute方法，现在咱们再回来看下RealCall的execute的具体实现



```java
    @Override
    public Response execute() throws IOException {
        synchronized (this) {
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
        captureCallStackTrace();
        try {
            client.dispatcher().executed(this);
            Response result = getResponseWithInterceptorChain();
            if (result == null) throw new IOException("Canceled");
            return result;
        } finally {
            client.dispatcher().finished(this);
        }
    }
```

首先是



```java
        synchronized (this) {
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
```

判断call是否执行过，可以看出每个Call对象只能使用一次原则。然后调用了captureCallStackTrace()方法。
 RealCall.java



```csharp
  private void captureCallStackTrace() {
    Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
    retryAndFollowUpInterceptor.setCallStackTrace(callStackTrace);
  }
```

RealCall的captureCallStackTrace() 又调用了Platform.get().getStackTraceForCloseable()



```java
public class Platform {
  public static Platform get() {
    return PLATFORM;
  }
  /**
   * Returns an object that holds a stack trace created at the moment this method is executed. This
   * should be used specifically for {@link java.io.Closeable} objects and in conjunction with
   * {@link #logCloseableLeak(String, Object)}.
   */
  public Object getStackTraceForCloseable(String closer) {
    if (logger.isLoggable(Level.FINE)) {
      return new Throwable(closer); // These are expensive to allocate.
    }
    return null;
  }
}
```

其实是调用AndroidPlatform. getStackTraceForCloseable(String closer)方法。这里就不详细说了，后面详细说。
 然后retryAndFollowUpInterceptor.setCallStackTrace(),在这个方法里面什么都没做就是set一个object进去



```java
public final class RetryAndFollowUpInterceptor implements Interceptor {
  public void setCallStackTrace(Object callStackTrace) {
    this.callStackTrace = callStackTrace;
  }
}
```

综上所示captureCallStackTrace()这个方法其实是捕获了这个请求的StackTrace。
 然后进入了第一个核心类---Dispatcher的的execute方法了，由于下面是进入了关键部分，所以重点讲解下，代码如何：



```kotlin
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
```

看下OKHttpClient的dispatcher()方法的具体内容如下图



```cpp
 //OKHttpClient.java
  public Dispatcher dispatcher() {
    return dispatcher;
  }
```

大家发现client.dispatcher()返回的是Dispatcher对象，那么这个Dispatcher对象是何时创建的那？在OkHttpClient.java里面Build类里面的构造函数里面，如下图



```java
//OkHttpClient.java
public static final class Builder {
   //其它代码先忽略掉
    public Builder() {
      dispatcher = new Dispatcher();
      //其它代码先忽略掉
    }
}
```

所以默认执行Builder()放到时候就创建了一个Dispatcher。那么咱们看下dispatcher里面的execute()是如何处理的



```csharp
  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

里面发现是runningSyncCalls执行了add方法莫非runningSyncCalls是个list，咱们查看dispatcher里面怎么定义runningSyncCalls的。



```php
  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

原来runningSyncCalls是双向队列啊，突然发现Dispatcher里面定义了三个双向队列，看下注释，我们大概能明白readyAsyncCalls 是一个存放了等待执行任务Call的双向队列，runningAsyncCalls是一个存放异步请求任务Call的双向任务队列，runningSyncCalls是一个存放同步请求的双向队列。关于队列咱们在下篇文章里面详细介绍。

执行完client.dispatcher().executed(this);要走到getResponseWithInterceptorChain();方法了里面了，看下这个方法是具体做什么的？



```csharp
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //添加开发者应用层自定义的Interceptor
    interceptors.addAll(client.interceptors());
    //这个Interceptor是处理请求失败的重试，重定向    
    interceptors.add(retryAndFollowUpInterceptor);
    //这个Interceptor工作是添加一些请求的头部或其他信息
    //并对返回的Response做一些友好的处理（有一些信息你可能并不需要）
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //这个Interceptor的职责是判断缓存是否存在，读取缓存，更新缓存等等
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //这个Interceptor的职责是建立客户端和服务器的连接
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      //添加开发者自定义的网络层拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    //一个包裹这request的chain
    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    //把chain传递到第一个Interceptor手中
    return chain.proceed(originalRequest);
  }
```

发现 new了一个ArrayList，然后就是不断的add，后面 new了 RealInterceptorChain对象，最后调用了chain.proceed()方法。先看下RealInterceptorChain的构造函数。



```kotlin
 public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, RealConnection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }
```

发现什么都没做就是做了赋值操作，后面跟踪下chain.proceed()方法
 由于Interceptor是个接口，所以应该是具体实现类RealInterceptorChain的proceed实现



```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```



```java
public final class RealInterceptorChain implements Interceptor.Chain{
  Response intercept(Chain chain) throws IOException;

      @Override 
      public Response proceed(Request request) throws IOException {
        return proceed(request, streamAllocation, httpCodec, connection);
      }

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
        RealInterceptorChain next = new RealInterceptorChain(
                interceptors, streamAllocation, httpCodec, connection, index + 1, request);
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

        return response;
    }

}
```

由于在构造RealInterceptorChain对象时候httpCodec直接赋予了null，所以下面代码直接略过。



```kotlin
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
```

然后看到在proceed方面里面又new了一个RealInterceptorChain类的next对象，温馨提示下，里面的streamAllocation, httpCodec, connection都是null，所以这个next对象和chain最大的区别就是index属性值不同chain是0.而next是1，然后取interceptors下标为1的对象的interceptor。由从上文可知，如果没有开发者自定义的Interceptor时，首先调用的RetryAndFollowUpInterceptor，如果有开发者自己定义的interceptor则调用开发者interceptor。

这里重点说一下，由于后面的interceptor比较多，且涉及的也是重要的部分，而咱们这里主要是讲流程，所以这里就不详细和大家说了，由后面再详细讲解，后面的流程是在每一个interceptor的intercept方法里面都会调用*chain.proceed()*从而调用下一个*interceptor*的*intercept(next)*方法，这样就可以实现遍历*getResponseWithInterceptorChain*里面*interceptors*的item，实现遍历循环，缩减后的代码如下：

```java
  //RetryAndFollowUpInterceptor.java
public Response intercept(Chain chain) throws IOException {
 //忽略部分代码
 response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
 //忽略部分代码
}
```

***

```java
//BridgeInterceptor.java
public Response intercept(Chain chain) throws IOException {
  //忽略部分代码
  Response networkResponse = chain.proceed(requestBuilder.build());
  //忽略部分代码
}
```

***

```java
//CacheInterceptor.java
public Response intercept(Chain chain) throws IOException {
   //忽略部分代码
   networkResponse = chain.proceed(networkRequest);
   //忽略部分代码
}
```

***

```java
//ConnectInterceptor.java
public Response intercept(Chain chain) throws IOException {
     //忽略部分代码
     return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

读过源码我们知道*getResponseWithInterceptorChain*里面*interceptors*的最后一个item是CallServerInterceptor.java，最后一个Interceptor(即CallServerInterceptor)里面是直接返回了response 而不是进行继续递归，具体里面是通过OKio实现的，具体代码，等后面再详细说明，CallServerInterceptor返回response后返回给上一个interceptor,一般是开发者自己定义的networkInterceptor，然后开发者自己的networkInterceptor把他的response返回给前一个interceptor，依次以此类推返回给第一个interceptor，这时候又回到了realCall里面的execute()里面了，代码如下：

```kotlin
  @Override 
  public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }   // 最后把response返回给get请求的返回值。至此整体GET请求的大体流程就已经结束了。(PS:最后别忘记走client.dispatcher().finished(this))
```

大体流程如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-c0e624fe441a861e.png?imageMogr2/auto-orient/strip|imageView2/2/w/892/format/webp)

### (二)、OKHTTP  异步请求debug代码跟踪：

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
        }
        @Override
        public void onResponse(Call call, Response response) throws IOException {
        }
    });
```

前面和同步一样new了一个OKHttp和Request。这块和同步一样就不说了，那么说说和同步不一样的地方，后面异步进入enqueue()方法

```java
   //RealCall.java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

由于executed默认为false，所以先进行判断是否为true，为true则直接跑异常，没有则设置为true，可以看出executed这个是一个标志，标志这个请求是否已经正在请求中，合同步一样先调用了captureCallStackTrace();然后调用 client.dispatcher().enqueue(new AsyncCall(responseCallback));client.dispatcher()返回的是Dispatcher对象所以实际调用的是Dispatcher的enqueue(),那么咱们进入源码看下

```csharp
  //Dispatcher.java
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;

  synchronized void enqueue(AsyncCall call) {
  //如果正在执行的请求小于设定值即64，并且请求同一个主机的request小于设定值即5
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      //添加到执行队列，开始执行请求
      runningAsyncCalls.add(call);
      //获得当前线程池，没有则创建一个
      executorService().execute(call);
    } else {
      //添加到等待队列中
      readyAsyncCalls.add(call);
    }
  }
```

根据源码和注释大家可以看到如果正在执行的异步请求小于64，并且请求同一个主机小于5的时候就先往正在运行的队列里面添加这个call，然后用线程池去执行这个call,否则就把他放到等待队列里面。执行这个call的时候，自然会去走到这个call的run方法，那么咱们看下*AsyncCall.java*这个类,而*AsyncCall.java*又继承*自NamedRunnable.java*咱们就一起看下他们的源码

```java
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
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

final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

上面看到*NamedRunnable*的构造方法设置了name在的run方法里面设定为当前线程的name,而*NamedRunnable*的run方法里面调用了它自己的抽象方法execute，由此可见*NamedRunnable*的作用就是设置了线程的name，然后回调子类的execute方法，那么我们来看下*AsyncCall*的execute方法。貌似好像又回到了之前同步的getResponseWithInterceptorChain()里面，根据返回的response来这只callback回调。所以我们得到了OKHTTP的大体流程，如下图：

![](https:////upload-images.jianshu.io/upload_images/5713484-95420b684a84c849.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

## 三、OKHTTP类详解

大体核心类主要下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-a97d25b97cd5843d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

最后给大家看一下整体的流程图

![img](https:////upload-images.jianshu.io/upload_images/5713484-fd5ed2418c5c03ec.png?imageMogr2/auto-orient/strip|imageView2/2/w/1000/format/webp)



# OKHttp源码解析(二)："前戏"——HTTP的那些事

## 一、TCP

关于TCP的具体，我这里就不细说，如这是我们每个程序员的基本知识，我这里就简单说下，看下OSI的七层模型：

![img](https:////upload-images.jianshu.io/upload_images/5713484-09b59583a5d4165b.png?imageMogr2/auto-orient/strip|imageView2/2/w/986/format/webp)



我们知道TCP工作在第四层Transport层(传输层)，IP在第三层Network层(网络层)，ARP在第二层Data Link层(数据链路层)；在第二层上的数据，我们把她叫做Frame，在第三层上的数据叫Packet，第四层的数据叫Segment。并且，我们需要知道，数据从应用层发下来，会在每一层都加上头部信息，进行封装，然后再发送到数据接收端。这个基本的流程你需要知道，就是每个数据都会经过数据的封装和解封装的过程。在OSI七层模型中，每一层的作用对应的协议如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-40fe1bab7d81a708.png?imageMogr2/auto-orient/strip|imageView2/2/w/714/format/webp)



TCP是一个协议，那这个协议是如何定义的，它的数据格式是什么样子的？那就要进行更深层次的剖析，就需要了解，甚至是熟记TCP协议中的每个字段的含义，如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-3812f5ea2e318821.png?imageMogr2/auto-orient/strip|imageView2/2/w/800/format/webp)



上面就是TCP协议头部的格式，由于它太重要了，所以下面就将每个字段的信息都详细说明下

> - Source Port和Destination Port:分别占用16位，表示源端口号和目的端口号；用于区别主机中的不同进程，IP地址用来区分不同主机的，源端口号和目的端口号配合上IP首部中的源IP地址就能确定为一个TCP连接
> - Sequenece Number:用来标识从TCP发送端向TCP接收端的数据字节流，他表示在这个报文中的第一个数据字节流在数据流中的序号；主要用来解决网络乱序的问题。
> - Acknowledgment Number：32位确认序号包发送确认的一端所期望收到的下一个序号，因此，确认需要应该是上次已成功收到数据字节序号+1，不过只有当标志位中的ACK标志(下面介绍)为1时该确认序列号的字段才有效。主要用来解决不丢包的问题。
> - Offset:给出首部中32bit字的数目，需要这个值是因为任选字段的长度是可变的。这个字段占4bit(最多能表示15个32bit的字，即4*15=60个字节的首部长度)，因此TCP最多有60个字节的首部。然而，没有任选字段，正常的长度是20字节。
> - TCP Flags:TCP首部中有6个比特，它们总的多个可同时被设置为1，主要是用于操控TCP的状态机，依次为URG，ACK，PSH，RST，SYN，FIN。
> - Window:窗口大小，也就是有名的滑动窗口，用来进行流量控制；这是一个复杂的问题

TCP是主机对主机层的传输控制协议，提供可靠的连接服务，采用三次握手确认建立一个连接:位码即tcp标志位,有6种标示:SYN(synchronous建立联机) ACK(acknowledgement 确认) PSH(push传送) FIN(finish结束) RST(reset重置) URG(urgent紧急)Sequence number(顺序号码) Acknowledge number(确认号码)
 现在来介绍一下这些的标志位：

> - URG:次标志表示TCP包的紧急指针域(后面马上就要说到)有效，用来保证TCP连接不被中断，并督促中间层设备要尽快处理这些数据。
> - ACK:此标志表示应答域有效，就是说前面所说的TCP的应答将会包含在TCP数据包中；有两个取值：0和1，为1的时候表示应答域有效，反之为0。
> - PSH:这个标志位表示Push操作。所谓Push操作就是是在数据包到达接受端以后，立即传送给应用程序，而不是在缓冲区排队。
> - RST:这个标志位表示连接复位请求。用来复位哪些产生错误的链接，也被用来拒绝错误和非法的数据包。
> - SYN:表示同步序号，用来建立连接。SYN标志位和ACK标志位搭配使用，当连接请求的时候，SYN=1，ACK=0；连接被响应的时候，SYN=1，ACK=1。这个标志的数据包常常被用来进行端口扫描。扫描者发送一个只有SYN的数据包，如果对方主机响应了一个数据包回来，就表明这台主机存在这个端口，但是由于这种扫描只是进行TCP三次握手的第一次握手，因此这种扫描的成功表明被扫描的机器很不安全，一台安全的主机将会强制要求一个连接严格的进行TCP的三次握手。
> - FIN:表示发送端已经达到数据末尾，也就是说双方数据传送完成，没有数据可以传送了，发送FIN标志位的TCP数据包后，连接将断开。这个表示的数据包也经常被用于进行端口扫描。

讲完标志位后就要开始正式的连接。

## 二、TCP3次握手和4次挥手

### (一)、3次握手

TCP是面向连接的，无论哪一方向另一方发送数据之前，都必须现在双方之间建立一条连接。在TCP/IP协议中，TCP协议提供可靠的连接服务，连接是通过三次握手进行初始化的。三次握手的目的是同步连接双方的序列号和确认号并交换TCP窗口大小的信息。这就是面试中经常被问到的TCP三次握手。那咱们就详细的了解下，看图述说：

![img](https:////upload-images.jianshu.io/upload_images/5713484-4b12153823cc3a0a.png?imageMogr2/auto-orient/strip|imageView2/2/w/875/format/webp)



多么清晰的一张图啊，可惜不是我的，我"借"来的

> - 1、第一次握手:建立连接，客户端先发送连接请求报文，将SYN位置为1，Sequence Number为X;然后，客户端进入SYN+SEND状态，等待服务器的确认；
> - 2、第二次握手：服务器收到SYN报文。服务器收到客户端的SYN报文，需要对这个SYN报文进行确认，设置Acknowledgment Number为x+1(Sequence+1)；同时，自己还要送法SYN消息，将SYN位置为1，Sequence Number为y；服务器将上述所有信息放到一个报文段(即SYN+ACK报文段)中，一并发送给客户端，此时服务器进入SYN+RECV状态。
> - 3、第三次握手；客户端收到服务器的  SYN+ACK报文段。然后将Acknowlegment Number设为y+1,向服务器发送ACK报文段，这个报文段发送完毕后，客户端端服务器都进入ESTABLISHED状态，完成TCP三次握手。

实例:

> - IP 192.168.1.116.3337 > 192.168.1.123.7788: S 3626544836
>    第一次握手：192.168.1.116发送位码syn＝1,随机产生seq number=3626544836的数据包到192.168.1.123,由SYN=1知道192.168.1.116要求建立联机;
> - IP 192.168.1.123.7788 > 192.168.1.116.3337: S 1739326486: ack 3626544837
>    第二次握手：192.168.1.123收到请求后要确认联机信息，向192.168.1.116发送ack number=3626544837,syn=1,ack=1,随机产生seq=1739326486的包;
> - IP 192.168.1.116.3337 > 192.168.1.123.7788: ack 1739326487,ack 1
>    第三次握手：192.168.1.116收到后检查ack number是否正确，即第一次发送的seq number+1,以及位码ack是否为1，若正确，192.168.1.116会再发送ack number=1739326487,ack=1，192.168.1.123收到后确认seq=seq+1,ack=1则连接建立成功。

完成了三次握手，客户端和服务器就可以开始传送数据了，以上就是TCP三次握手的总体介绍。

### (二)、4次回挥手

当客户端和服务器进行三次握手简历TCP连接以后，当数据传送完毕，肯定要断开TCP连接。那对于TCP的断开，就是"4次挥手"。

> - 1、客户端(也可以是服务器)，设置Sequence Number和Acknowledgment Number，向服务器发送一个FIN报文段。此时客户端进入FIN_WAIT_1状态；这表示客户端没有数据发送给主机了。
> - 2、服务器收到客户端发来的FIN报文段，向客户端回一个ACK报文段，Acknowledgement Number为Sequence Number加1；客户端进入FIN_WAIT_2状态，服务器进入CLOSE_WAIT状态；服务器告诉客户端，我同意你的"关闭"请求。
> - 3、服务器向客户端发送FIN报文段，请求关闭连接，同时服务器进入LAST_ACK状态。
> - 4、客户端收到服务器发送的FIN报文段，向主机发送ACK报文段，然后客户端进入TIME_WAIT状态，服务器收到客户端的ACK报文段以后，就关闭连接，此时，客户端等待2MSL后一次没有到收到回复，则证明Server端已正常关闭，那好，客户端也可以关闭连接了。

至此，TCP的四次分手就完成了。

为了让大家更好的理解3次握手和4次挥手，我准备了几道问题，让大家更好的理解3次握手和4次挥手。

**1、为什么握手要“3”次？**

一般同学会认为，握手为什么要3次，感觉2次就可以了。在谢希仁的<计算机网络>中是这样说的

> 为了防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误。

在书中同时举了一个例子，如下:

> “已失效的连接请求报文段”的产生在这样一种情况下：client发出的第一个连接请求报文段并没有丢失，而是在某个网络结点长时间的滞留了，以致延误到连接释放以后的某个时间才到达server。本来这是一个早已失效的报文段。但server收到此失效的连接请求报文段后，就误认为是client再次发出的一个新的连接请求。于是就向client发出确认报文段，同意建立连接。假设不采用“三次握手”，那么只要server发出确认，新的连接就建立了。由于现在client并没有发出建立连接的请求，因此不会理睬server的确认，也不会向server发送数据。但server却以为新的运输连接已经建立，并一直等待client发来数据。这样，server的很多资源就白白浪费掉了。采用“三次握手”的办法可以防止上述现象发生。例如刚才那种情况，client不会向server的确认发出确认。server由于收不到确认，就知道client并没有要求建立连接。”

这就很明白了，防止了服务器端的一直等待而浪费资源

**2、为什么握手要“3”次，而挥手要"4"次?**

TCP协议是一种面向连接的、可靠的、基于字节流的运输层通信协议。TCP是全双工模式，这就意味着，当客户端发起FIN报文段时，只表示客户端已经已经没有数据要发送了，客户端告诉服务器，它的数据已经全部发送完毕了，但是此时客户端还是可以接受服务器的数据；当服务器返回ACK报文段是时，表示它已经知道客户端没有数据发送了，但是服务器还是可以发送数据到客户端；当服务器也发送了FIN报文时，这时候就是表示服务器也没有数据要发送给客户端了，之后就可以愉快地中断这次TCP连接。

为了让大家更好的地理解四次挥手的原理，我们再讲解下四次挥手的状态：

> - 1、FIN_WAIT_1:这个状态要好好解释一下，其实FIN_WAIT_1和FIN_WAIT_2状态的真正含义都是表示等待对方的FIN报文。而这两种状态的区别：FIN_WAIT_1状态实际上是当SOCKET在ESTABLISHED状态时，它想主动关闭连接，向对方发送了FIN报文，此时该SOCKET即进入FIN_WAIT_1状态。而当对方回应ACK报文后，则进入FIN_WAIT_2状态，当然在实际的正常情况下，无论对方何种情况下，都应该马上回应ACK报文，所以FIN_WAIT_1状态一般是比较难见到的，而FIN_WAIT_2状态还有时常常可看到。
> - 2、FIN_WAIT_2:上面已经详细解释了这种状态，实际上 FIN_WAIT_2状态下的SOCKET，表示半连接，既有一方要求close连接，但另外还告诉对方，我暂时还有点数据需要传送给你(ACK信息)，稍后再关闭连接
> - 3、CLOSE_WAIT_2:这种状态的含义其实表示在等待关闭。怎么理解？当对方close一个SOCKET后发送一个FIN报文给自己，服务器系统毫无疑问会回应一个ACK报文给对方，此时则进入到CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是查看你是否还有数据发送给对方，如果没有的话，那么你也就可以close这个这个SOCKET，发送FIN报文给对方，也即关闭连接。所以你在CLOSE_WAIT状态下，需要完成的事情是等待你去关闭连接。
> - 4、LAST_ACK:这个状态还是比较容易好理解的，它是被动关闭一方在发送FIN报文后，最后等待对方的ACK报文。当收到ACK报文后，也可以进入CLOSED状态了。
> - 5、CLOSED:表示连接中断
> - 6、TIME_WAIT:表示收到对方的FIN报文，并发出了ACK报文就等2MSL后即可会到CLOSED可用状态了。如果FIN_WAIT_1下，收到了对方同时带FIN标志和ACK标志的报文时，可以直接进入到TIME_WAIT状态，而无须经过FIN_WAIT_2状态。

**3、为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？**

虽然按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假设网络是不可靠的，一切都可能发生，比如有可能最后一个ACK丢失。所以TIME_WAIT状态是用来重发可能丢失的ACK报文。

## 三、HTTPS

### (一)、什么是HTTPS?

HTTPS(Hypertext Transfer Protocol over Secure Socket Layer/基于安全套接字层的超文本传输协议,或者也可以说HTTP OVER SSL)是网景公司开发的web协议。SSL的版本最高为3.0，后来的版本被称为TLS，现在所用的协议版本一般都是TLS，但是由于SSL出现的时间比较早，所以现在一般指的SSL一般也就是TLS，本文中后面都统一使用SSL来代替TLS。HTTPS是以安全为目标的HTT通道，简单讲就是HTTP的安全版，所以你可以理解HTTPS=HTTP+SSL。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容是SSL。

### (二)、为什么要用HTTPS?

在没有HTTPS之前使用HTTP的协议的，可见HTTPS肯定是弥补的HTTP的安全缺陷，那么咱们来看下HTTP协议的那些安全缺点：
 1、通信使用明文(不加密)，内容可能被窃听(抓包工具可以获取请求和响应内容)如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-ff68832ba0379a24.png?imageMogr2/auto-orient/strip|imageView2/2/w/1038/format/webp)



2、不验证通讯方的身分，任何人都坑你发送请求，不管对方是谁都返回相应，如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-267bcdec2dadada5.png?imageMogr2/auto-orient/strip|imageView2/2/w/798/format/webp)



3、无法证明报文的完整性，可能会遭到篡改，即没有办法确认发出的请求/相应前后一致。

![img](https:////upload-images.jianshu.io/upload_images/5713484-725c91b448beba2b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1003/format/webp)



所以为了解决上述问题，需要在HTTP上加入加密处理和认证机制，我们把添加了加密和认证机制的http称之为http。即HTTP+加密+认证+完成性保护=HTTPS。
 这里先简单的说下HTTPS说下他的优势：

> - 1、内容加密，建立一个信息的安全通道，来保证数据传输过程的安全性。
> - 2、身份认证，确认网站的真是性。
> - 3、数据完整性，防止内容被第三方冒充或者篡改。

是不是 刚好弥补了上面的缺陷。。。。
 补充一下:HTTPS并非应用层的一种新协议，只是HTTP通讯接口部分用SSL(secure socket layer)和TLS(transport layer security)协议代替，通常HTTP直接和TCP通信时，当使用SSL时，则演变成先和SSL通信，再由SSL和TCP通讯，因此所以HTTPS其实就是身披SSL保护外衣的HTTP

### (三)、HTTPS的缺点

上帝给你关上一道门 同时给你打开一扇窗，所以万事万物都是各有利弊，那么HTTPS的缺点是什么那?
 由于要对数据进行加密，认证，所以注定他会比HTTP慢，当然现在也有很多优化。
 对数据进行加解密决定了它比http慢。
 PS:处于安全考虑，浏览器是不会再本地保存HTTPS缓存。实际上，只要在HTTP头中使用特定的命令，HTTPS是可以被缓存的。Firefox默认只在内存中缓存HTTS。但是，只要在请求头中有Cache-Control:Public，缓存就会被写到磁盘上，IE只要http头允许就可以缓存https内容，缓存策略与是否使用HTTPS协议无关。

### (四)、HTTP与HTTPS的相同和异同点

1、HTTP与HTTPS的相同点：
 大多数情况下，HTTP和HTTPS是相同的，因此都采用同一个基础协议，作为HTTP或者HTTPS客户端--浏览器/app等，设置一个连接到web服务器指定的宽口。当服务器接收到请求，它会返回一个状态码以及消息，这个回应可能是请求信息、或者指示某个错误发送的错误信息。系统使用统一资源定位符URI，因此资源可以被唯一指定。在表面上HTTPS和HTTP唯一不同的只是一个协议头HTTPS的说明，其他都一样。

2、HTTP与HTTPS的不同点：

> - 1、HTTPS需要用到CA申请证书。
> - 2、HTTP是超文本传输协议，信息是明文的；HTTPS则是具有安全性的SSL加密传输协议。
> - 3、HTTPS和HTTP使用的是完全不同的连接方式，用的端口也不一样，HTTP是80,HTTPS是443。
> - 4、HTTP的连接很简单，是无状态的，HTTPS是HTTP+SSL协议构建的，可进行加密传输、身份认证的网络协议，比HTTP协议安全。

### (五)、简单说点加密和证书的事

再详细说HTTPS之前先来了解下加密和证书的事情

**1、加密**

加密的2种技术：

> - 1、对称加密(也叫私钥加密)，是指加密和解密使用相同的密钥的加密算法。有时又叫传统加密算法，就是加密密钥能够从解密密钥中推算出来，同时解密密钥也可以从加密密钥中推算出来。而在大多数的对称算法中，加密密钥和解密密钥是相同的，所以也称这种加密算法为秘密密钥或者单密钥算法，常见的对称加密有DES(Data Encryption Standard)、AES(Advanced Encryption Standard)、RC4、IDEA。
> - 2、非对称加密，是指和甲酸算法不同，非对称加密算法有两个密钥：公开密钥(public key）和私有密钥(private key)；并且加密密钥和解密密钥是成对出现的。非对称加密算法在加密和解密过程使用了不同的密钥，非对称加密也称公钥加密，在密钥对中，其中一个密钥是对外公开的，所有都可以获取到，称为公钥，其中一个密钥是不公开的称为私钥。非对称加密算法对加密内容的长度有限制，不能超过公钥长度。常见的非对称加密RSA、DSA/DSS等。

**2、数字摘要**

数字摘要是采用单项Hash函数将需要加密的明文"摘要"成一串固定长度(128位)的密文，这一串密文又称为数字指纹，它有固定的长度，而且不同的明文摘要成密文，其结果总是不同的，而同样的明文摘要必定一定。“数字摘要”是https能确保数据完整性和防篡改的根本原因。常用的摘要主要有MD5、SHA1、SHA256等。

**3、数字签名**

数字签名技术就是对"非对称密钥加解密"和"数字摘要"两项技术的应用，它将摘要信息用发送者的私钥加密，与原文一起传送给接受者。接受者只有用发送者的公钥才能解密被加密的摘要信息，然后用HASH函数对收到的原因产生一个摘要信息，与解密的摘要信息对比。如果相同，则说明收到的信息是完整的，在传输的过程中没有被修改，否则说明信息被修改过，因此数字签名能够验证信息的完整性。
 数字签名的过程：

```bash
明文——>hash运算——>摘要——>私钥加密——>数字签名
```

数字签名的两种功效：

> - 1、能确定消息确实由发送方签名并发出来的，因为别人假冒不了发送方的签名。
> - 2、数字签名能确定消息的完整性。

注意:
 数字签名只能验证数据的完整性，数据本身是否加密不属于数字签名的控制范围

**4、数字证书**

对于请求方来说，它怎么能确定它所得到的公钥一定是从目标主机哪里发送的，
 而且没有被篡改过呢?亦或者请求的目标主机本身就是从事窃取用户信息的不正当行为呢？这时候，我们需要一个权威的值得信赖的第三方机构(一般是由政府机构审核并授权的机构)来统一对外发送主机机构的公钥，只要请求方这种机构获取公钥，就避免了上述问题

(1)数字证书的颁发过程:
 用户首先产生自己的密钥对，并将公共密钥及部分个人身份信息传送给认证中心。认证中心在核实身份后，将执行一些必要的步骤，以确信请求确实由用户发送而来，然后，认证中心将发送给用户一个数字证书，该证书内包含用户的人信息和他的公钥信息，同时还附有认证中心的签名信息(根证书私钥)签名。用户就可以使用自己的数字证书进行相关的各种活动。数字证书由独立的证书发行机构发布，数字证书各不相同，每种证书可提供不同级别的可信度。
 (2)证书包含哪些内容

> - 1、证书颁发机构的名称
> - 2、证书本身的数字签名
> - 3、证书持有者的公钥
> - 4、证书签名用到的Hash算法

(3)验证证书的有效性
 浏览器默认都是会内置CA跟证书，其中根证书包含了CA的公钥

> - 防伪造证书1:如果证书颁发机构是伪造的，浏览器不认识，直接认为是危险证书
> - 防伪造证书2:证书颁发机构是的确存在的，于是根据CA名，找到对应内置的CA根证书、CA的公钥。用CA的公钥，对伪造的证书的摘要进行解密，发现解密不了，认为是危险证书
> - 防篡改:对于篡改的证书，使用CA的公钥对数字签名进行解密得到摘要A，然后再根据签名的Hash算法计算出证书的摘要B，对比A与B，若相等则正常，若不相等则是被篡改过的。
> - 防过期失效验证:正式课在其过期前辈吊销，通常情况是该证书的私钥已经失密。较新的浏览器如果Chrome、Firefox、Opera和Internet Explored 都实现了在线证书的状态协议(OCSP)以排除这种情况:浏览器将网站提供的证书序列号通过OCSP发送给证书颁发机构，后者会告诉浏览器证书是否还是有效的。

![img](https:////upload-images.jianshu.io/upload_images/5713484-4fbe2698dcb58f10.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



### (六)那么咱们来看下HTTPS的结构图

![img](https:////upload-images.jianshu.io/upload_images/5713484-57a544e16d85e486.png?imageMogr2/auto-orient/strip|imageView2/2/w/946/format/webp)





由上图可以看到，HTTP是直接建立在TCP之上的，而HTTPS则又经过了TLS一系列协议。所以，https协议在建立连接之前，首先要和双方握手，身份认证，传输数据之前要进行加密

### (七)SSL和TLS

**1、SSL(Secure Sokcet Layer，安全套接字层)**

SSL是Netscape(网景公司)所研发，用于保障在Internet上数据传输至安全，利用数据加密(Entryption)技术，可确保数据在网络值传输过程中不会被截取，当前版本为3.0。

SSL协议可分为两层:

> - 1、SSL记录协议(SSL Record Protocol):它建立在可靠的传输协议(如TCP)之上，为高层协议提供数据封装、压缩、加密等基本功能的支持。
> - 2、SSL握手协议(SSL Handshake Protocol):它建立在SSL记录协议之上，用于在实际的数据传输开始前，通讯双方进行身份认证、协商加密算法、交换加密密钥等。

**2、TLS(Transport Layer Security,传输层安全协议)**

用于两个应用程序之间提供保密性和数据完整性。
 TLS1.0是IETF(Internet Engineering Task Force,Internet工程任务组)制定的一种新的协议，它建立在SSL3.0协议规范之上，是SSL3.0的后续版本，可以理解为SSL3.1，它是写入RFC的，该协议由两层组成：TLS记录协议(TLS Record)和TLS握手协议(TLS Handshake)。较低的层为TLS记录协议，位于某个可靠的传输协议(例如TCP)上面

TLS 1.0是IETF（Internet Engineering Task Force，Internet工程任务组）制定的一种新的协议，它建立在SSL 3.0协议规范之上，是SSL 3.0的后续版本，可以理解为SSL 3.1，它是写入了 RFC 的。该协议由两层组成： TLS 记录协议（TLS Record）和 TLS 握手协议（TLS Handshake）。较低的层为 TLS 记录协议，位于某个可靠的传输协议（例如 TCP）上面。

**3、SSL/TLS协议作用：**

> - 认证用户和服务器，确保数据发送到正确的客户端和服务器
> - 加密数据以防止数据中途被窃取
> - 维护数据的完整性，确保数据在传输过程中不被改变。

**4、TLS比SSL的优势**

> - 1、对于消息认证使用密钥散列发：TLS使用"消息认证代码的密钥散列法"(HMAC),当记录在开放的网络(如因特网)上传时，该代码确保记录不会被变更。SSLv3.0还提供键控消息认证，但HMAC比SSLv3.0使用的(消息认证代码)MAC功能更安全
> - 2、增强的伪随机功能(PRF):PRF生成密钥数据。在TLS中，HMAC定义PRF。PRF使用两种散列算法保证其安全性，如果任一算法暴露了，只要第二种算法未暴露，则数据仍然是安全的。
> - 3、改进的已完成消息验证：TLS和SSLv3.0都对两个端点提供已完成的消息，该消息认证交换的消息没有被变更。然而，TLS将此已完成消息基于PRF和
> - 4、一致证书处理：与SSLv3.0不同，TLS试图制定必须在TLS之间实现交互的证书类型。
> - 5、特定警报消息：TLS提供更多的特定和附加警报，以指示任一会话端点检测到的问题。TLS还对核实应该发送某些警报进行记录。

### (八)HTTPS握手的过程

HTTP握手是有“3次握手的”，HTTPS是有"8次握手的"，大体分为以下几个步骤：
 1 验证证书的有效性(是否被更改，是否合法)
 2 握手生成会话密钥
 3 利用会话密钥进行内容传输
 先上图

![img](https:////upload-images.jianshu.io/upload_images/5713484-97199e7cd5bbad55.png?imageMogr2/auto-orient/strip|imageView2/2/w/712/format/webp)





我去，都是英文的啊，不过自己看了以下，貌似还能看得懂，咱们就先详细说下：

**1、客户端首次发出请求**

由于客户端(如浏览器)对一些加解密算的支持程度不一样，但是在TLS协议传输过程中必须使用同一套加解密算法才能保证数据能够正常的加解密。在TLS握手阶段，客户端要首先告知服务器，自己支持哪些加密算法，所以客户端需要将本地支持的加密套件(Cipher Suit)的列表传送给服务器。除此之外，客户端还要产生一个随机数，这个随机数一方面需要在客户端保存，另一方面需要传给服务器，客户端的随机数需要跟服务器的随机数结合起来产生后面将要的Master Secret.
 客户端需要提供如下信息:

> - 1、支持的协议版本，比如TLS 1.0版本
> - 2、一个客户端生成的随机数，稍后用于生成"对话密钥"
> - 3、支持的加密方法，比如RSA公钥加密
> - 4、支持的压缩方法
>    PS：客户端发送的信息之中不包括服务器的域名，也就是说，理论上服务器只能包含一个网站，否则会分不清应该向客户端提供哪一个网站的提供的数字证书。这就是为什么通常一台服务器只能由一张数字证书的原因
>    对于虚拟主机的用户来说，这当然很不方便。2006年，TLS协议加入了Server Name Indication扩展，允许客户端向服务器提供它所请求的域名。

**2、服务器的配置**

采用HTTPS协议的服务器必须要有一套数字证书，可以是自己制作或者CA证书。区别就是自己办法的证书需要客户端验证通过，才可以继续访问，而使用CA证书则不会弹出提示页面。这套证书其实就是一堆公钥和私钥。公钥给别人加密使用，私钥给自己解密使用。服务器在接收到客户端的请求后，服务器需要确定加密协议的版本，以及加密的算法，然后也生成一个随机数。

**3、服务器首次回应**

把上个阶段生成的加密协议的版本，加密的算法，随机数以及自己的证书一起发送给客户端。这个随机数是整个过程的第二个随机数。
 返回的信息包括:

> - 1、协议的版本，比如TLS1.0版本，如果浏览器与服务器支持的版本不一致，服务器关闭加密通信
> - 2、加密的算法
> - 3、随机数
> - 4、服务器证书

除了上面这些信息，如果服务器需要确认客户端身份，就会再包含一项请求，要求客户端提供"客户端证书"。比如，金融机构往往只允许认证客户进入自己的网络，就会向正式客户提供USB密钥，里面包含了一张客户端证书。

**4、客户端验证证书**

客户端收到服务器回应以后，首先验证服务器证书。如果证书不是可信任机构颁布、或者证书中的域名和实际域名不一致、或者证书已经过期，就会向访问者显示一个警告，由其选择是否还要继续通信。
 PS:验证证书主要根据服务端发过来的证书名称，在本地寻找其低级证书，并一级一级直到根证书，验证各级证书的合法性。

**5、客户端传送加密信息**

验证证书通过后，客户端再次产生一个随机数(第三个随机数)，然后使用证书中的公钥进行加密，以及放一个ChangCipherSpec消息即编码改变的消息，还有整个前面所有消息的hash值，进行服务器验证，然后用新密钥加密一段数据一并发送到服务器，确保正式通信前无误。
 客户端使用前面的两个随机数以及刚刚新生成的新随机数(又称”pre-master key”)。
 简单的说下ChangeCipherSpec:
 ChangeCipherSpec是一个独立的协议，体现在数据包中就是一个字节的数据，用于告知服务器，客户端已经切换到之前协商好的加密套件(Cipher Suite)的状态，准备使用之前协商好的加密套件加密数据并传输了。
 这部分传送的是用证书加密后的随机值，目的就是让服务器得到这个随机值，以后客户端和服务其的通信就可以通过这个随机值来进行加密解密了。服务器与客户端之前的数据传输过程中是庸才对称加密方式加密的。

注意：
 此外，如果前一步，服务器要求客户端证书，客户端会在这一步发送证书及相关信息。

**6、生成会话密钥**

上面产生的随机数，是整个握手阶段的出现的第三个随机数，又称"pre-master key"。有了它之后，客户端和服务器同时有了三个随机数，接着双方就用事先协商的加密方法，各自生成本地会话所用的同一把"会话密钥"。
 PS:为什么一定要用三个随机数，来生成"会话密钥"？

> 答：不管是客户端还是服务器，都是下需要随机数，这样生成的密钥才不会每次都一样，由于SSL协议中证书是静态的，因此十分有必要引入一种随机因素来保证协商出的密钥的随机性。
>  对于RSA密钥交换算法来说，pre-master-key本身就是一个随机数，再加上第一步、第三步消息中的随机数，三个随机数通过一个密钥导出器最终导出一个对称密钥。pre master 的存在在于SSL协议不信任每一个主机都能产生完全的随机数，如果随机数不随机，那么pre master secret就可能被猜出来，那么仅适用于pre master secret作为密钥就不合适了，因此必须引入新的随机因素，那么客户端和服务器加上pre master secret三个随机数一同生成的密钥就不容易被猜出了，一个伪随机数可能完全不随机，但是三个伪随机就十分接近随机了，每增加一个自由度，随机性增加的可不是一。

**7、服务器最后的回应**

服务器生成"会话密钥"后，向客户端最后发送下面信息：

> - 1、编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送。
> - 2、服务器握手结束通知，表示服务器握手阶段已经结束。这一项同时也是前面所有内容的hash值，用来供客户端校验。

**8 客户端解密**

客户端用之前生成的私钥解密服务器传过来的信息，于是获取了解密后的内容

至此，整个握手阶段全部结束了。接下来，客户端与服务器进入加密通信，就完全是使用普通HTTP协议，只不过用"会话密钥"加密内容。

##### 注意事项

> SSL协议在握手阶段使用的是非对称加密，在传输阶段使用的是对称加密,也就是在说在SSL上传送数据是使用对称加密加密的!因为非对称加密的速度缓慢，耗费资源。其实当客户端和服务器使用非对称加密方式建立连接后，客户端和主机已经决定好了在传输过程使用的对称加密算法和关键的对称加密密钥，由于这个过程本身是安全可靠的，也即对称加密密钥是不可能被窃取盗用的，因此，保证了在传输过程中对数据进行对称加密也是阿安全可靠的！如果有人窃听通信，他可以知道双方选择的加密方法，以及三个随机数中的两个。整个通话的安全，只取决于第三个随机数(pre master secret)能不能被破解。

### (九)HTTPS与代理

2.4 如何使用代理（charles）
 我们知道从HTTPS的整个原理可以知道，客户端和服务器进行通信的成果，客户端是能拿到数据的，代理也一定能拿到，包括公共密钥，证书，算法等，但代理无法获取服务器的私钥，所以无法获取5/6/7/8的会话密钥，也就无法得到数据传输的明文，所以默认的情况下，charles是无法抓https的。那如何让charles转包并获取明文？也就是让charles获取私钥，获取服务器的是不可能的，那只能在通信过程中使用charles自己的证书，并在通信的过程中主动为请求的域名发放证书，流程如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-f9784014cdcadffb.png?imageMogr2/auto-orient/strip|imageView2/2/w/803/format/webp)

从上面可以看出，一旦信任代理的证书，代理在中间就做一个转换的角色，即担任客户端的服务器，又担任服务器的客户端，在中间传输的过程中做了加密解密转换。也就是说一旦信任代理，大力就可以干任何事。

## 四、SPDY

2012年google如一声惊雷提出了SPDY的方案，大家猜开始从正面看待和解决老版本HTTP协议本身的问题，SPDY可以说是综合了HTTPS和HTTP两者有点于一体的传输协议，主要解决：

> - 1、降低延迟:：针对HTTP高延迟的问题，SPDY优雅的采取了多路复用(multiplexing)。多路复用通过多个请求stream共享一个TCP连接的方式，解决了HOL blocking的问题，降低了延迟同事提高了带宽的利用率。
> - 2、请求优先级：多路复用带来的一个新的问题是，在连接共享的基础上有可能会导致关键请求被阻塞。SPDY允许给每个request设置优先级，这样重要的请求就会优先得到相应。比如浏览器加载首页，首页的html内容应该优先展示，之后才是各种静态资源文件，脚本文件等加载，这样保证用户第一时间看到网页的内容。
> - 3、header压缩：HTTP1.x的header很多时候都是重复多余的。选择和是的压缩算法可以减少包的大小和数量。
> - 4、基于HTTPS的加密协议传输，大大提高了传输数据的可靠性。
> - 5、服务端推送(server push)，采用SPDY的网页，例如一个网页有一个style.css请求，客户端在收到style.css数据的同事，服务端会将style.js文件推送给客户端，当客户端再次尝试获取style.js时就可以直接从缓存中获取到，不用再次发送请求了。SPDY构成图:

![img](https:////upload-images.jianshu.io/upload_images/5713484-0646a09f39d3b7cd.png?imageMogr2/auto-orient/strip|imageView2/2/w/238/format/webp)



SPDY位于HTTP之下，TPC和SSL之上，这样可以轻松兼容老版本的HTTP协议，同时也可以使用已有的SSL功能。
 大家来看下SPDY的兼容性：

![img](https:////upload-images.jianshu.io/upload_images/5713484-5ccfffcfe33316ac.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



## 五、HTTP2.0

### (一)、HTTP2.0的前世今生

顾名思义有了HTTPS1.x，那么HTTP2.0也就顺理成章的出现了。
 在HTTP/1.x中，如果客户端想发起多个并行请求必须建立多个TCP连接，这无疑增大了网络开销。另外HTTP/1.x不会压缩请求和响应头，导致了不必要的网络流量，HTTP/1.x不支持资源优先级导致底层TCP连接利用率低下。而这些问题都是HTTP/2要着力解决的。HTTP2.0可以说是SPDY的升级版(其实也是基于SPDY设计的)，但是HTTP2.0跟SPDY仍有不同的地方，主要有以下两点：

> - 1、HTTP2.0支持明文HTTP传输，而SPDY轻质使用HTTPS
> - 2、HTTP2.0消息头的压缩算法采用HPACK，而非SPDY采用的DEFLATE

### (二)、 HTTP2.0的新特性

> - 1、新的二进制格式(Binary Format)：HTTP 1.x的解析是基于文本。基于文本洗衣的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合，基于这种考虑HTTP2.0的协议解析决定采用二进制分帧格式，实现方便且健壮。
> - 2、多路复用(multiPlexing)：即连接共享，即每一个request都是用作连接共享机制的。一个request对应一个id，这样一个连接上可以有多个requst，每个连接的request可以随机的混杂在一起，接受方可以根据request的id将request再归属到各自不同的服务端请求里面，后面有一张多路复用原理图
> - 3、请求优先级：把HTTP消息分解为很多独立的帧后，就可以通过优化这些帧的交错和传输顺序，进一步提供性能。为了做到这一点，每个流都有一个带有31比特的的优先值
> - 4、header压缩：HTTP1.x的header带有大量的信息，而且每次都要重复发送，HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一个header field表，既避免了重复的header传输，又减少了需要传输的大小。
> - 5、服务端推送(server push)：同SPDY一样，HTTP2.0也具有server push功能。

HTTP2.0性能增强的核心，全在于新增的二进制分帧层，它定义了如何封装HTTP消息并在客户端与服务器之间传输

![img](https:////upload-images.jianshu.io/upload_images/5713484-6e3fabb2df5ec707.png?imageMogr2/auto-orient/strip|imageView2/2/w/580/format/webp)

所以HTTP/2引入了三个新的概念：

> - 1、数据流：基于TCP连接之上的逻辑双向字节流，对应一个请求及响应。客户端每发起一个请求就建立一个数据量就，后续该请求及其响应的所有数据都通过该数据流传输。
> - 2、消息：一个请求或者响应对应的一系列数据帧
> - 3、帧”：HTTP/2的最小数据切片单位，每个帧包含帧首部，至少也会标示出当前帧所属的流。

上述概念之间的逻辑关系：

> - 1、所有通信都在一个TCP连接上完成，此连接可以承载任意数量的双向数据流。
> - 2、每个数据流都有一个唯一的标识符和可选的优先级信息，用于承载双向消息。
> - 3、每条消息都是一条逻辑HTTP消息(例如请求或者响应)，包含一个或多个帧。
> - 4、帧是最小的通信单位，承载着特定类型的数据，例如HTTP标头、消息负载等等。来自不同数据流的帧可以交错发送，然后再根据每个帧头的数据流标识符重新组装。
> - 5、每个HTTP消息分分解为多个独立的帧后可以交错发送，从而在宏观上实现了多个请求或者响应并行传输的效果。这类似于多进程环境下的时间分片机制。

所有HTTP 2.0通信都在一个连接上完成，这个连接可以承载任意数量的双向数据流。每个数据流以消息的形式发送。而消息由一或多个帧组成，而这些帧可以乱序发送，然后再根据每个帧首部流标识符重新组装。



![img](https:////upload-images.jianshu.io/upload_images/5713484-838559ff48f41961.png?imageMogr2/auto-orient/strip|imageView2/2/w/479/format/webp)

image.png

简而言之，HTTP2.0把HTTP协议通信的基本单位缩小为一个一个的帧，这些帧对应着逻辑流中的消息。相应地，很多流可以并行地在同一个TCP连接上交换消息。

在HTTP/1.1中，如果客户端想发送多个平行的请求以及改进性能，必须使用多个TCP连接。HTTP2.0的二进制分帧层突破了限制；客户端和服务器可以把HTTP消息分解为互不依赖的帧，然后乱序发送，最后再把另一端把它们重新组合起来。如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-c5fb00296f4c4555.png?imageMogr2/auto-orient/strip|imageView2/2/w/744/format/webp)

HTTP2.png

下面这个图更形些



![img](https:////upload-images.jianshu.io/upload_images/5713484-7cda76527926f93b.png?imageMogr2/auto-orient/strip|imageView2/2/w/932/format/webp)

多路复用图.png

在HTTP/1.x中，头部的数据都是纯文本格式。通常会增加500-800字节的负荷。为了减少开销提高性能，HTTP/2压缩头部数据，由于HTTP2.0连接的两端都知道已经发送了哪些头部，这些头部的值是什么，从而可以针对之前的数据编码发送差异数据。如下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-db697ed5dc5e21a5.png?imageMogr2/auto-orient/strip|imageView2/2/w/372/format/webp)



### (三)HTTP2.0与HTTP1.1的区别

![img](https:////upload-images.jianshu.io/upload_images/5713484-9ec242baba2282e9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1178/format/webp)



## 六、隧道

Web隧道(Web tunnel)是HTTP的另一种用法，可以通过HTTP应用程序访问非HTTP协议的应用程序，Web隧道允许用户允许用户通过HTTP连接发送非HTTP流量，这样就可以在HTTP上捎带其他协议数据了。使用Web隧道最常见的原因就是要在HTTP连接中嵌入非HTTP流量。这样这类流量就可以穿过只允许Web流量通过的防火墙了。

### (一) 建立隧道

Web隧道是用HTTP的CONNECT方法建立起来的。CONNECT方法请求隧道网管创建一条到达任一目的服务器和端口的TCP连接，并对客户端和服务器职期间的后续数据进行盲转发。
 下图显示了CONNECT方法如何建立起一条到达网管的隧道，来自《HTTP 权威指南》



![img](https:////upload-images.jianshu.io/upload_images/5713484-e2af63579ad13e9c.png?imageMogr2/auto-orient/strip|imageView2/2/w/654/format/webp)



> - 1、(a)是客户端相互发送了额一条CONNECT请求给隧道网关。客户端的CONNECT方法请求隧道网关打开一条TCP连接(在这里，打开的是到主机orders.joes-hardware.com的标准SSL端口443的连接)；
> - 2、(b)和(c)中创建了TCP连接，一旦建立了TCP连接，网管就会发送一条HTTP200 Connection Established响应来通知客户端，此时，隧道就建立起来了。客户端通过HTTP隧道发送的所有数据都会被直接转发给输出TCP连接，服务器发送的所有数据都会通过HTTP隧道转发给客户端。

上图中的例子描述了一条SSL隧道，其中的SSL流量是在一条HTTP连接上发送的，但是通过CONNECT方法可以与使用任意协议的任意服务器建立TCP连接。

1、CONNECT请求
 除了起始行之外，CONNECT的语言与其他HTTP方法类似。一个后面跟着冒号和端口号的主机名取代了请求的URL。主机和端口都必须制定:



```undefined
CONNECT home.netscape.com：443 HTTP/1.0
User-agent： Mozilla/4.0
```

和其他HTTP报文一样，起始行之后，有零个或多个HTTP请求首部字段。这些行照例以CRLF结尾，首部列表以一个空的CRLF结束。
 2、CONNECT响应
 发送了请求之后，客户端会等待来网管的响应，和普通HTTP报文一样，响应码表示成功。按照惯例，响应中的原因短语通常设为"Connection Established"。



```jsx
HTTP/1.0 200 Connection Established
Proxy-agent: Netscape-Proxy/1.1
```

与普通HTTP响应不同，这个响应并不需要包含Content-Type首部。此时连接只是对原始字节进行转接，不再是报文的承载者，所以不需要使用内容类型。

管道化数据对网管是不透明的，所以网管不能对分组的顺序和分组流做任何假设。一旦隧道建立起来了，数据就可以在任意时间流向任意方向了。

作为一种性能优化方法，允许客户端在发送了CONNECT请求之后，接受响应之前，发送隧道数据。这样可以更快的将数据发送给服务器，但这就意味着网管必须能够正确处理跟在请求之后的数据。尤其是，网管不能假设网络I/O请求只会返回首部数据，网管必须确保在连接准备就绪时，将与首部一同读进来的数据发送给服务器。在请求之后，或者其他非200但不致命的错误状态，就必须做好重发请求数据的准备。如果在任意时刻，隧道的任意一个端点断开连接，那个端点发出的所有未传输数据都会被传送给另一个端点，之后到另一个端点的链接也会被代理终止。如果还有数据要传输给关闭连接的端点，数据会被丢弃。

### (二)SSL隧道

最初开发Web隧道是为了通过防火墙来传输加密的SSL流量。很多组织都会将所有流量通过分组过滤路由器和代理服务器以隧道方式传输，以提升安全性。但有些协议，比如加密SSL，其信息是加密的的，无法通过传统的代理服务器转发。隧道会通过一条HTTP连接来传输SSL流量，以穿过端口80的HTTP防火墙。

![img](https:////upload-images.jianshu.io/upload_images/5713484-0078e8d5570e81d4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



为了让SSL流量经现存的代理防火墙进行传输，HTTP中添加了一项隧道特性，在此特性中，可以将原始的加密数据放在HTTP报文中，通过普通的HTTP信道传送。

![img](https:////upload-images.jianshu.io/upload_images/5713484-22161576d1a61d88.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



> - a图代表一个SSL连接，SSL流量直接发给了一个(SSL端口443上的)安全Web服务器。
> - b图代表了SSL流量被封装到一条HTTP报文中，并通过HTTP端口80的连接发送，最后解封装为普通的SSL连接。

通常会用隧道将非HTTP流量传过端口过滤防火墙。这一点可以得到很好的利用。比如，通过防火墙传输完全SSL流量。但是，这项特性可能会被滥用，使得恶意协议通过HTTP隧道流入某个组织内部。
 可以像其他协议一样，对HTTPS协议(SSL上的HTTP)进行网管操作：由网关(而不是客户端)初始化与远端HTTPS服务器的SSL会话，然后代表客户端执行HTTPS事务。响应会由代理接受并解密，然后通过(不安全的)HTTP传送给客户端。这是网关处理FTP的方式。但这种方式有几个缺点：

> - 1、客户端到网关之间的链接是普通的非安全HTTP；
> - 2、尽管代理是已认证主体，但客户端无法对远端服务器执行SSL客户端认证(基于X509证书的认证)；网关要支持完整的SSL实现

可以像其他协议一样，对HTTPS协议(SSL上的HTTP)进行网关操作：由网关(而不是客户端)初始化与远端HTTPS服务器的SSL会话，然后代表客户端执行 HTTPS事务。响应会由代理接收并解密，然后通过(不安全的)HTTP传送给客户端。这是网关处理FTP的方式。但这种方式有几个缺点：客户端到网关之间的连接是普通的非安全HTTP；尽管代理是已认证主体，但客户端无法对远端服务器执行SSL客户端认证(基于X509证书的认证)；网关要支持完整的SSL实现。

对于SSL隧道机制来说，无需在代理中实现SSL。SSL会话是建立在产生请求的客户端和目的(安全的)Web服务器之间的，中间的代理服务器只是将加密数据经过隧道传输，并不会在安全事务中扮演其他的角色。

在适当的情况下，也可以将HTTP的其他特性与隧道配合使用。尤其是，可以将代理的认证支持与隧道配合使用，对客户端使用隧道的权利进行认证。

![img](https:////upload-images.jianshu.io/upload_images/5713484-fdbd7489da892a82.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



总的来说，隧道网管无法验证目前使用的协议是否就是它原本打算经过的隧道的协议。因此，比如说，一些修换捣乱的用户可能会通过本打算用于SSL的隧道，越过公司防火墙传递因特网的游戏流量，而恶意用户可能会用隧道打开Telnet会话，或用隧道绕过公司的E-mail扫描器来发送E-mail。为了降低对隧道的滥用，网关应该只为特定的知名端口，比如HTTPS端口443打开隧道。

## 七、代理

Web代理是一种存在于网络中间的实体，提供各式各样的功能。现在网络系统中，Web代理无处不在。

### (一)代理的作用

> - 1、提高访问速度。因为客户端要求的数据存于代理服务器的硬盘中，因此下次这个客户端或者其他要求相同的站点数据时，就会直接从代理服务器的硬盘中读取，代理服务器起到了缓存的作用，对热门站点有很多客户访问时，代理服务器的优势更为明显。
> - 2、Proxy可以起到防火墙的作用，因为所有代理服务器的用户都必须通过代理服务器访问远程站点，因此在代理服务器上就可以设置相应的限制，以过滤或屏蔽掉某些信息。这是局域网网管对局域网用户访问限制最常用的办法，也是局域网用户为什么不能浏览某些网站的原因。拨号用户如果使用代理服务器，同样必须服从代理服务器的访问限制，除非你不使用这个代理服务器。
> - 3、通过代理服务器访问一些不能直接访问的网站，互联网上有许多开放的代理服务器，客户在访问权限受到限制时，而这些代理服务器的访问权限是不受限制的，刚好代理服务器在客户的访问范围之内，那么客户通过代理服务器访问目标网站就能成为可能。国内高校的多使用教育网，不能出国，但通过代理服务器，就能实现访问因特网，这就是高校内代理服务器热的原因。
> - 4、安全性得到提高。无论在上聊天室还是浏览网站，目标网站只知道你来自代理服务器，而你的真实IP就无法预测，这就使得使用者的安全性得以提高

### (二)、代理服务器工作流程

> - 1、当客户端A对Web服务器请求时，A的请求会首先发送到代理服务器。
> - 2、代理服务器接收到客户端请求后，会检查缓存中是否存有客户端所需要的数据。
> - 3、如果代理没有客户端A请求的数据，它将会向Web服务器提交请求
> - 4、Web服务器响应请求的数据
> - 5、代理服务器向客户端A转发Web服务器的数据
> - 6、客户端B访问Web服务器，向代理服务发出请求。
> - 7、代理服务器查找缓存记录，确认已经存在Web服务器相关数据。
> - 8、代理服务器直接回应查询的信息，而不需要再去服务器进行查询，从而达到节约网络流量和提高访问速度的目的

### (三)、代理的形式

HTTP代理存在两种形式，分别如下：

> - 第一种是RFC 7230 -HTTP/1.1：Message Syntax and Routing(即修订后的RFC 2616，HTTP/1.1 协议的第一部分)描述的普通代理。这种代理扮演的是"中间人“的角色，对于连接到它的客户端来说，它是服务端，对于要连接的服务器来说，它是客户端。它就是负责在两端之间来回传送HTTP报文。
> - 第二种是Tunneling TCP basedprotocols through Web proxy servers (通过Web代理服务器用隧道方式传输基于TCP协议)描述的隧道代理。通过HTTP协议正文部分(Body)完成通讯，以HTTP方式实现任意基于TCP应用层协议代理。这种代理使用HTTP的CONNECT方法建立连接，但是CONNECT 最开始并不是RFC 2616 -HTTP/1.1的一部分，直到2014年发布的HTTP/1.1修订版中，才增加了对CONNECT及隧道代理的描述，详见RFC 7231- HTTP/1.1:Semantics andContent。实际上这种代理早就被广泛实现。

PS:事实上，第一种代理，对应<HTTP权威指南>一书中第六章"代理"，第二种代理，对应第八章"集成点：网关、隧道及中继"中的8.5小节"隧道"。

**1、普通代理**

原理：HTTP客户端向代理服务器发送请求报文，代理服务器需要正确地处理请求和连接(例如正确处理(conntion:keep-alive)，同时向目标服务器发送请求，并将受到的响应装发给客户端。

> - 假如我通过代理访问A网站，对A来说，它会把代理当做客户端，完全察觉不到真正的客户端的存在，这实现了隐藏客户端IP的目的，当然代理也可以修改HTTP请求头部，通过X-Forwarded-IP这杨的自定义头部告诉服务器真正的客户端IP。但服务器无法验证这个自定义头部真的是由代理添加，还是客户端修改了请求头，所以从HTTP头部字段取IP时，需要格外小心。

- 给浏览器显示的指定代理，需要手动修改浏览器或相关设置，或者指定PAC(Proxy Auto-Configuration，自动配置代理)文件自动设置，还有些浏览器支持WPAD(Web ProxyAutodiscovery Protocol ,Web代理自动发现协议)。显示指定浏览器代理这种方式一般称之为正向代理，浏览器齐总正向代理后，会对HTTP请求报文做一些修改，来规避老旧代理服务器的一些问题。
- 还有一种情况是访问A网站时，实际上访问的是代理，代理收到请求报文后，再向真正提供服务的服务器发起请求，并将响应转发给浏览器。这种情况一般被称为反向代理，它可以用来隐藏服务器IP及端口，一般使用反向代理后，需要通过修改DNS让域名解析到代理服务器IP，这是浏览器无法察觉真正服务器的存在，当然也就不需要修改配置了。反向代理是Web系统最为常见的部署方式。

**2、隧道代理**

原理：HTTP客户端通过HTTP的CONNECT方法请求隧道代理，创建一条到达任意目的服务器和端口的TCP连接，并对客户端和服务器之间的后继数据进行盲转发。
 具体请查看本片文章第五部分

这里将HTTP隧道分为两种：

> - 1、不使用CONNECT的隧道
>    不实用CONNECT的隧道，实现了数据包的重组和转发。在Proxy收到来自客户端的HTTP请求后，会重新创建Request请求，并发送到目标服务器。当目标服务器返回Response给Proxy之后，Proxy会对Response进行解析，然后重新组装Resposne，发送给客户端。所以在不使用CONNECT方式建立的隧道，Proxy有机会对客户端与目标服务器之间的通信数据进行窥探，而且有机会对数据进行串改。
> - 2、使用CONNECT的隧道:而对于使用CONNECT的隧道则不同。当客户端向Proxy发起HTTP CONNECT Method的时候，就是告诉Proxy，先在Proxy和目标服务器之间先建立起连接，在这个连接建立起来之后，目标服务器会返回一个回复给Proxy，Proxy将这个回复转发给客户端，这个Response是Proxy跟目标服务器连接建立的状态回复，而不是请求数据的Response。在此之后，客户端跟目标服务器所有的通信豆浆使用值前简建立来的连接。这种情况下的HTTP隧道，Proxy仅仅实现转发，而不会关心转发的数据，这也就是为什么在使用Proxy的时候，HTTPS请求必须首先使用HTTP CONNECT建立隧道。因为HTTPS的数据都是经过加密的，Proxy是无法对HTTPS的数据进行解密的，所只能使用CONNECT，仅仅对通信数据进行转发。

**3、与proxy有关的字段**

> - X-Forwarded-For(XFF):是用来识别通过HTTP代理或者负载均衡方法连接到Web服务器的客户端最原始的IP地址的HTTP请求头字段。Squid:缓存代理服务器的开发人员最早引入了这一HTTP头字段，如果该没有XFF或者另外一个种相似的技术，所有通过代理服务器的连接只会显示代理服务器的IP地址(而非连接发起的原始IP地址)，这雅漾的代理服务器实际上充当了匿名服务提供者的角色，如果连接的原始IP地址不可得，恶意访问的检查与预防的难度将大大增加
> - X-Forwarded-Host和X-Forwarded-Proto分别记录客户端最原始的主机和协议
> - Proxy-Authorization:连接到Proxy的身份验证信息
> - Proxy-connection:它不是标准协议的一部分，标准些协议中已经存在一种机制可以完成此协议头的功能，这就是Connection头域，与Proxy-Connection头相比，Connection协议头几乎提供了相同的功能，除了错误部分。而且，Connection协议头可用于任意连接之间，包括HTTP服务器，代理，客户端，而不是像Proxy-Connection一样，只能用于代理服务器和客户端之间。

## 八、InetAddress类和InetSocketAddress类

### (一)、InetAddress(IP地址)

InetAddress是IP地址(也是主机地址)类封装计算机的IP地址和DNS，不包含端口。

**1、如何获取对象:**

创建主机地址对象要靠InetAddress类的几个静态工厂方法:

> - InetAddress getByAddress(byte[] addr):仅根据IP地址的各个字节创建IP地址对象。
> - InetAddress getLocalHost():返回本机IP地址对象
> - InetAddress getByAddress(String host,byte[] addr): 根据主机名和IP地址的各个自己创建IP地址对象。要把IP地址的高字节放在addr的低索引处。
> - InetAddress getByName(String host)：仅根据主机名创建IP地址对象。函数会访问DNS服务器来获取host对象的IP地址。

注：主机名既可以是域名，也可以是IP地址的字符串形式。例如，



```css
InetAddress.getByName("www.163.com")
InetAddress.getByName("192.168.2.23")
```

**2、方法**

> - String toString():返回"主机名/IP地址"字符串
> - String getHostName() :返回此IP地址的主机名
> - byte[] getAddress():返回此IP地址的各个字节，高字节放在地索引处
> - String getHostAddress():返回此IP地址的字符串形式
> - Boolea isReachable(int timeout):测试此IP地址的可达性，最多等待timeout毫秒。
> - Boolean equal(Object obj)：如果此IP地址的getAddress()结果与obj.getAddress()不仅数组长度相同且每个元素也相同，则返回true。

### (二)、InetSocketAddress类

InetSocketAddress是在InetAddress基础上封装了端口号。所以说InetSocketAddress是(IP地址+端口号)类型，也就是端口地址类型。
 它的基类是SocketAddress类，SocketAddress类里面什么都没有。

**1、如何获取对象:**

> - InetSocketAddress(int port):创建IP地址为通配符地址，端口号为port的端口地址。
> - InetSocketAddress(InetAddress addr,int port):创建IP地址addr，端口号为port的端口地址。
> - InetSocketAddress(String host,int port)：根据主机名和端口号创建端口地址。函数会访问DNS服务器来解析出host和IP地址的，如果解析失败，则标记为"未解析"
> - InetSocketAddress createUnresolved(String host,int port):同上，但是不会访问DNS服务器去解析主机名，而是直接标记为"未解析"。

**2、方法**

> - boolean isUnsolved():如果尚未解析出IP地址，则返回true。
> - String toString():返回"主机名/IP地址：端口地址"字符串
> - InetAddress getAddress():返回此端口的IP地址。
> - String getHostName():返回此端口的主机名。
> - int getPort():返回此端口的端口号。
> - boolean equals(Object obj):如果此端口与obj端口的IP地址和端口号都相等，则返回true。

### (三)、两者的却别

关键就是在InetSocketAddress不基于任何协议，一般用于socket编程。

> - 表面看InetSocketAddress多了一个端口号，端口的作用：一台拥有IP地址的主机可以提供许多服务，比如Web服务、FTP服务、SMTP服务等，这些服务完全可以通过1个IP地址来实现。
> - 那么主机怎么区分不同的网络服务？显然不能只靠IP地址，因此IP地址与网络服务的关系是一对多的关系。
> - 实际上是通过"IP地址+端口号"来区分不同的服务的。



# OKHttp源码解析(三)--中阶之线程池和消息队列

## 一、线程池的理解

**(一)android中的异步任务**

android的异步任务一般都是用Thread+Handler或者AsyncTask来实现，其中笔者当初经历过各种各样坑，特别是内存泄漏，当初笔者可是相当的欲死欲仙啊！所以现在很少有开发者还在用这一套来做异步任务，现在一般都是Rxjava为主，当然还有自己自定义的异步任务框架(比如笔者)，像RxJava都帮我们写好了对应场景的线程池，这是为什么？

**1、线程池的理解**

我对线程池的理解是有两个层次，一种是狭隘的，一种是广义的，那么咱们各自都说下

**(1)狭义上的线程池：**

> 线程池是一种多线程处理形式，处理过程中将任务添加到队列中，后面再创建线程去处理这些任务，线程池里面的线程都是后台线程，每个线程都是默认的优先级下运。如果某个线程处于空闲中，将添加一个任务进来，让空闲线程去处理任务。如果所有线程都很繁忙，消息队列会挂起，等待某个线程池空闲后再处理任务。这样可以保证线程数量不能超多最大数量。

**(2)广义上的线程池：**

> 多线程技术主要是解决处理器单元内多个线程执行的问题，它可以显著减少处理的单元闲置时间，增加处理器单元的吞吐能力。如果对多线程应用不当，会增加对单个任务的的处理时间。

举例说明：
 假如一个服务器完成一项任务的时间为T：

> T1 创建线程的时间
>  T2 在线程中执行任务的时间，包括线程同步所需要的时间
>  T3 线程销毁的时间

显然 T= T1+T2+T3. 注意:这是一个理想化的情况

> 可以看出，T1，T3是多线程自身带来的开销(在Java中，通过映射pThread，并进一步通过SystemCall实现native线程)，我们渴望减少T1和T3的时间，从而减少T的时间。但是一些线程的使用者并没有注意到这一点，所以在线程中频繁的创建或者销毁线程，这导致T1和T3在T中占有相当比例。这显然突出的线程池的弱点(T1，T3),而不是有点(并发性)。
>  [取自IBM知识库](https://link.jianshu.com?t=https://www.ibm.com/developerworks/cn/java/l-threadPool/)

所以线程池的技术正是如何关注缩短或调整T1，T3时间的技术，从而提高服务器程序的性能。

> - 1、通过对线程进行缓存，减少创建和销毁时间的损失
> - 2、通过控制线程数量的阀值，减少线程过少带来的CPU闲置(比如长时间卡在I/O上了)与线程过多给JVM内存与线程切换时系统调用的压力。

在平时我们可以通过*线程工厂*来创建线程池来尽量避免上述的问题。

## 二 Dispatcher 类详解

[Dispatcher](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/Dispatcher.java)负责请求的分发

### 1、线程池executeService



```csharp
/** Executes calls. Created lazily. */
private ExecutorService executorService;

public synchronized ExecutorService executorService() {
   if (executorService == null) {
     executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS, new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
   }
   return executorService;
 }
```

由上面代码可以得出Dispatcher内部实现了懒加载的无边界限制的线程池。参数解析

> - 1、0：核心线程数量，保持在线程池中的线程数量(即使已经空闲)，为0代表线程空闲后不会保留，等待一段时间后停止。
> - 2、Integer.MAX_VALUE:表示线程池可以容纳最大线程数量
> - 3、TimeUnit.SECOND:当线程池中的线程数量大于核心线程时，空闲的线程就会等待60s才会被终止，如果小于，则会立刻停止。
> - 4、new SynchronousQueue<Runnable>()：线程等待队列。同步队列，按序排队，先来先服务
> - 5、Util.threadFactory("OkHttp Dispatcher", false):线程工厂，直接创建一个名为OkHttp Dispatcher的非守护线程。

(1)SynchronousQueue每个插入操作必须等待另一个线程的移除操作，同样任何一个移除操作都等待另一个线程的插入操作。因此队列内部其实没有任何一个元素，或者说容量为0，严格说并不是一种容器，由于队列没有容量，因此不能调用peek等操作，因此只有移除元素才有元素，显然这是一种快速传递元素的方式，也就是说在这种情况下元素总是以最快的方式从插入者(生产者)传递给移除者(消费者),这在多任务队列中最快的处理任务方式。对于高频请求场景，无疑是最合适的。

(2)在OKHttp中，创建了一个阀值是Integer.MAX_VALUE的线程池，它不保留任何最小线程，随时创建更多的线程数，而且如果线程空闲后，只能多活60秒。所以也就说如果收到20个并发请求，线程池会创建20个线程，当完成后的60秒后会自动关闭所有20个线程。他这样设计成不设上限的线程，以保证I/O任务中高阻塞低占用的过程，不会长时间卡在阻塞上。

### 2、发起请求

整个框架主要通过Call来封装每一次的请求。同时Call持有OkHttpClient和一份Request。而每一次的同步或者异步请求都会有Dispatcher的参与。

**(1)、同步**

Dispatcher在执行同步的Call：直接加入到runningSyncCall队列中，实际上并没有执行该Call，而是交给外部执行



```csharp
  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

**(2)、异步**

将Call加入队列：如果当前正在执行的call的数量大于maxRequest(64),或者该call的Host上的call超过maxRequestsPerHos(5)，则加入readyAsyncCall排队等待，否则加入runningAsyncCalls并执行



```csharp
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

### 3、结束请求

从ready到running,在每个call结束的时候都会调用finished

```csharp
 private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      //每次remove完后，执行promoteCalls来轮转。
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }
    //线程池为空时，执行回调
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }

  private void promoteCalls() {
     //如果当前执行的线程大于maxRequests(64)，则不操作
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
```

通过上面代码，大家可以知道finished先执行calls.remove(call)删除call，然后执行promoteCalls()，在promoteCalls()方法里面：如果当前线程大于maxRequest则不操作，如果小于maxRequest则遍历readyAsyncCalls，取出一个call，并把这个call放入runningAsyncCalls，然后执行execute。在遍历过程中如果runningAsyncCalls超过maxRequest则不再添加，否则一直添加。所以可以这样说：

promoteCalls()负责ready的Call到running的Call的转化

具体的执行请求则在RealCall里面实现的，同步的在RealCall的execute里面实现的，而异步的则在AsyncCall的execute里面实现的。里面都是调用RealCall的getResponseWithInterceptorChain的方法来实现责任链的调用。

## 三、OKHttp的任务调度

在说调度任务之前先说下

### 1、Dispatcher任务调度

在OKHttp中，它使用[Dispatcher](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/7826bcb2fb1facb697a4c512776756c05d8c9deb/okhttp/src/main/java/okhttp3/Dispatcher.java#L38-L38)作为任务的调度器。

如下图所示



![img](https:////upload-images.jianshu.io/upload_images/5713484-8b70426f2370e277.png?imageMogr2/auto-orient/strip|imageView2/2/w/1044/format/webp)

Dispatcher.png

在整个调度流程中涉及的成员如下：
 其中

> - *Dispatcher* 对象是分发者，也是生产者(默认在主线程中)
> - *AsyncCall* 对象其实是一个任务即Runnable(内部做了包装异步接口)



```cpp
// Dispatcher.java 
maxRequests = 64   // 最大并发请求数为64
maxRequestsPerHost = 5 //每个主机最大请求数为5
ExecutorService executorService  //消费者池（也就是线程池）
Deque<AsyncCall> readyAsyncCalls： // 异步的缓存，正在准备被消费的（用数组实现，可自动扩容，无大小限制）
Deque<AsyncCall> runningAsyncCalls //正在运行的 异步的任务集合，仅仅是用来引用正在运行的任务以判断并发量，注意它并不是消费者缓存
Deque<RealCall> runningSyncCalls  //正在运行的，同步的任务集合。仅仅是用来引用正在运行的同步任务以判断并发量
```

通过将请求任务分发给多个线程，可以显著的减少I/O等待时间

### 2、OKHttp调度的具体流程分析

#### (1)同步调度分析

第一步是：是调用了RealCall的execute()方法里面调用executed(this);

```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

**第二步：在Dispatcher里面的executed执行入队操作**

```csharp
 /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

**第三步：执行getResponseWithInterceptorChain();进入拦截器链流程，然后进行请求，获取Response，并返回Response result 。**

**第四步：执行client.dispatcher().finished(this)操作**

```cpp
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }
```

这里其实做的是出队操作。至此同步的调度就已经结束了

#### (2)异步调度分析

AsyncCall类简介

在讲解异步调度之前不得不提到AsyncCall这个类，AsyncCall，他其实是RealCall的内部类

```kotlin
 //RealCall.java
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        //执行耗时任务
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          //retryAndFollowUpInterceptor取消了 执行失败
          signalledCallback = true;
         //回调，注意这里回调是在线程池中，不是主线程
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          //一切正常走入正常流程
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
       //最后执行出队
        client.dispatcher().finished(this);
      }
    }
  }
```

**第一步 是调用了RealCall的enqueue()方法**

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

在enqueue里面调用了client.dispatcher().enqueue(new AsyncCall(responseCallback));方法

**第二步：在Dispatcher里面的enqueue执行入队操作**

```csharp
  synchronized void enqueue(AsyncCall call) {
    //判断是否满足入队的条件(立即执行)
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      //正在运行的异步集合添加call
      runningAsyncCalls.add(call);
      //执行这个call
      executorService().execute(call);
    } else {
      //不满足入队(立即执行)条件,则添加到等待集合中
      readyAsyncCalls.add(call);
    }
  }
```

上述代码发现想要入队需要满足下面的条件

```css
(runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost)
```

如果满足条件，那么就直接把AsyncCall直接加到runningCalls的队列中，并在线程池中执行（线程池会根据当前负载自动创建，销毁，缓存相应的线程）。反之就放入readyAsyncCalls进行缓存等待。
 runningAsyncCalls.size() < maxRequests  表示当前正在运行的AsyncCall是否小于maxRequests = 64
 runningCallsForHost(call) < maxRequestsPerHos 表示同一个地址访问的AsyncCall是否小于maxRequestsPerHost = 5;
 即 当前正在并发的请求不能超过64且同一个地址的访问不能超过5个

第三步：这里分两种情况

**情况1 第三步 可以直接入队**

```csharp
runningAsyncCalls.add(call);
```

**第四步：线程池executorService执行execute()方法**

```css
executorService().execute(call);
```

由于AsyncCall继承于NamedRunnable类，而NamedRunnable类又是Runnable类的实现类，所以走到了AsyncCall的execute()方法里面

**第五步：执行AsyncCall的execute()方法**

```kotlin
     @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```

**第六步：执行getResponseWithInterceptorChain();进入拦截器链流程，然后进行请求，获取Response。**

**第七步：如果是正常的获取到Response，则执行responseCallback.onResponse()**

**第八步：执行client.dispatcher().finished(this)操作 进行出队操作**

```cpp
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }
```

PS:注意这里面第三个参数 同步是false，异步是true，如果是异步则需要进行是否添加继续入队的情景

**情况2  第三步 不能直接入队，需要等待**

```csharp
readyAsyncCalls.add(call);
```

**第四步 触发条件**

能进入等待则说明当前要么有64条正在进行的并发，要么同一个地址有5个请求，所以要等待。

当有如下条件被满足或者触发的时候则执行promoteCalls操作

- 1 Dispatcher的setMaxRequestsPerHost()方法被调用时

```java
   public synchronized void setMaxRequestsPerHost(int maxRequestsPerHost) {
    //设置的maxRequestsPerHost不能小于1
    if (maxRequestsPerHost < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequestsPerHost);
    }
    this.maxRequestsPerHost = maxRequestsPerHost;
    promoteCalls();
  }
```

- 2 Dispatcher的setMaxRequests()被调用时



```java
public synchronized void setMaxRequests(int maxRequests) {
     //设置的maxRequests不能小于1
    if (maxRequests < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequests);
    }
    this.maxRequests = maxRequests;
    promoteCalls();
  }
```

- 3当有一条请求结束了，执行了finish()的出队操作，这时候会触发promoteCalls()进行调整



```undefined
 if (promoteCalls) 
    promoteCalls();
```

**第五步 执行Dispatcher的promoteCalls()方法**

```csharp
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
```

**第六步 先判断是否满足 初步入队条件**

```kotlin
 if (runningAsyncCalls.size() >= maxRequests) 
    return;
 if (readyAsyncCalls.isEmpty()) 
    return; // No ready calls to promote.
```

如果此时 并发的数量还是大于maxRequests=64则return并继续等待
 如果此时，没有等待的任务，则直接return并继续等待

**第七步 满足初步的入队条件，进行遍历，然后进行第二轮入队判断**

```csharp
  for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
```

进行同一个host是否已经有5请求在了，如果在了，则return返回并继续等待

**第八步 此时已经全部满足条件,则从等待队列面移除这个call，然后添加到正在运行的队列中**

```csharp
        i.remove();
        runningAsyncCalls.add(call);
```

**第九步 线程池executorService执行execute()方法**

```css
executorService().execute(call);
```

**第十步：执行AsyncCall的execute()方法**

```kotlin
     @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```

**第十一步：执行getResponseWithInterceptorChain();进入拦截器链流程，然后进行请求，获取Response。**

**第十二步：如果是正常的获取到Response，则执行responseCallback.onResponse()**

**第十三步：执行client.dispatcher().finished(this)操作 进行出队操作**

```cpp
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }
```

至此 异步任务调度已经结束了

#### 总结：

> - 1、异步流程总结
>    所以简单的描述下异步调度为：如果当前还能可以执行异步任务，则入队，并立即执行，否则加入readyAsyncCalls队列，当一个请求执行完毕后，会调用promoteCalls()，来把readyAsyncCalls队列中的Async移出来并加入到runningAsyncCalls，并开始执行。然后在当前线程中去执行Call的getResponseWithInterceptorChain（）方法，直接获取当前的返回数据Response
> - 2、对比同步和异步任务，我们会发现:同步请求和异步请求原理都是一样的，都是在getResponseWithInterceptorChain()函数通过Interceptor链条来实现网络请求逻辑，而异步任务则通过ExecutorService来实现的。PS:在Dispatcher中添加一个封装了Callback的Call的匿名内部类AsyncCall来执行当前 的Call。这个AsyncCall是Call的匿名内部类。AsyncCall的execute方法仍然会回调到Call的 getResponseWithInterceptorChain方法来完成请求，同时将返回数据或者状态通过Callback来完成。

## 四、OKHttp调度的"优雅'之处：

1、采用Dispacher作为调度，与线程池配合实现了高并发，低阻塞的的运行
 2、采用Deque作为集合，按照入队的顺序先进先出
 3、最精彩的就是在try/catch/finally中调用finished函数，可以主动控制队列的移动。避免了使用锁而wait/notify操作。

![](https:////upload-images.jianshu.io/upload_images/5713484-4e3db972fc792ba6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

# OKHttp源码解析(四)--中阶之拦截器及调用链

## 一、interceptor调用链的入口

那我们书接上文。上篇文章已经说明了OKHttp有两种调用方式，一种是阻塞的同步请求，一种是异步的非阻塞的请求。但是无论同步还是异步都会调用下RealCall的 getResponseWithInterceptorChain方法来完成请求，同时将返回数据或者状态通过Callback来完成。源代码如下：

```csharp
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //添加 在配置 OkHttpClient 时设置的 interceptors
    interceptors.addAll(client.interceptors());
    //添加 负责失败重试以及重定向的 RetryAndFollowUpInterceptor；
    interceptors.add(retryAndFollowUpInterceptor);
    //添加 负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的 响应转换为用户友好的响应的 BridgeInterceptor；
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //添加 负责读取缓存直接返回、更新缓存的 CacheInterceptor；
    interceptors.add(new CacheInterceptor(client.internalCache()));
     //添加 负责和服务器建立连接的 ConnectInterceptor；
    interceptors.add(new ConnectInterceptor(client));
    //如果不是webSocket
    if (!forWebSocket) {
      //添加 OkHttpClient 时设置的 networkInterceptors；
      interceptors.addAll(client.networkInterceptors());
    }
    //最后 添加 负责向服务器发送请求数据、从服务器读取响应数据的 CallServerInterceptor。
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```

从上面可知，无论同步还是异步，他们的入口都是从RealCall的getResponseWithInterceptorChain进来的。

## 二、interceptor接口和RealInterceptorChain类

**interceptor接口详解**

[Interceptor](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FInterceptor.java) 负责拦截和分发

> - 1.1 先来看看Intercepor本身文档的含义：观察，修改以及可能短路的请求输出和响应请求的回来。通常情况下拦截器用来添加，移除或者转换请求或者回应的头部信息
> - 1.2 拦截器，就像水管一样，把一节一节的水管(拦截器)连起来，形成一个回路，实际上client到server也是如此，通过一个又一个的interceptor串起来，然后把数据发送到服务器，又能接受返回的数据，每一个拦截器(水管)都有自己的作用，分别处理不同东西，比如消毒，净化，去杂质，就像一层层过滤网一样。

```java
/**
 * Observes, modifies, and potentially short-circuits requests going out and the corresponding
 * responses coming back in. Typically interceptors add, remove, or transform headers on the request
 * or response.
 */
public interface Interceptor {
   //负责拦截
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();
     //负责分发、前行
    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```

> - 2.1 Interceptor是一个接口，主要是对请求和相应的过滤处理，其中有一个抽象方法即*Response intercept(Chain chain) throws IOException*负责具体的过滤。
> - 2.2 而在他的子类里面又调用了Chain，从而实现拦截器调用链(chain)，所以真正实现拦截作用的是其内部接口Chain

Interceptor.Chain的实现类都是RealInterceptorChain，也就说说处理调用过程的实现是RealInterceptorChain。所以RealInterceptorChain持有一个List的Interceptor，通过对这个List的Interceptor进行迭代和递归推进。让我们看看源码实现。



```java
/**
 * A concrete interceptor chain that carries the entire interceptor chain: all application
 * interceptors, the OkHttp core, all network interceptors, and finally the network caller.
 */
public final class RealInterceptorChain implements Interceptor.Chain {
  private final List<Interceptor> interceptors;
  private final Request request;   
 // 下面属性会在执行各个拦截器的过程中一步一步赋值
  private final StreamAllocation streamAllocation;//在RetryAndFollowUpInterceptor中new的
  private final HttpCodec httpCodec; //在ConnectInterceptor中new的
  private final RealConnection connection; //在ConnectInterceptor中new的
  private final int index;  //通过index + 1
  private int calls;  //通过call++

  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, RealConnection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }

  @Override public Connection connection() {
    return connection;
  }

  public StreamAllocation streamAllocation() {
    return streamAllocation;
  }

  public HttpCodec httpStream() {
    return httpCodec;
  }

  @Override public Request request() {
    return request;
  }
   // 实现了父类proceed方法
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  //处理调用
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    // 1、迭代拦截器集合
    if (index >= interceptors.size()) throw new AssertionError();
     //2、创建一次实例，call+1
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
    // 3、创建一个RealInterceptorChain实例
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    //4、取出下一个 interceptor
    Interceptor interceptor = interceptors.get(index);
    //5、执行intercept方法，拦截器又会调用proceed()方法
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

    return response;
  }
}
```

看到了上述源码，给大家分析一下就是：

> - 第一步，先判断是否超过list的size，如果超过则遍历结束，如果没有超过则继续执行
> - 第二步calls+1
> - 第三步new了一个RealInterceptorChain，其中然后下标index+1
> - 第四步 从list取出下一个interceptor对象
> - 第五步 执行interceptor的intercept方法

总结一下就是每一个RealInterceptorChain对应一个interceptor,然后每一个interceptor再产生下一个RealInterceptorChain，直到List迭代完成。所以上面基本上就是迭代+递归,找了一些图片有助于大家理解如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-ffe0f2bb3c9c91d1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

拦截器1.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-a2568b4d0d2a9712.png?imageMogr2/auto-orient/strip|imageView2/2/w/560/format/webp)

拦截器2.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-018254742cda4d7b.png?imageMogr2/auto-orient/strip|imageView2/2/w/831/format/webp)

拦截器3.png

> PS:
>  这里的拦截器有点像安卓里面的触控反馈的Interceptor。既一个网络请求，按一定的顺序，经由多个拦截器进行处理，该拦截器可以决定自己处理并且返回我的结果，也可以选择向下继续传递，让后面的拦截器处理返回它的结果。这个设计模式叫做[责任链模式](https://link.jianshu.com?t=https%3A%2F%2Fzh.wikipedia.org%2Fwiki%2F%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F)。
>  与Android中的触控反馈interceptor的设计略有不同的是，后者通过返回true 或者 false 来决定是否已经拦截。而OkHttp这里的拦截器通过函数调用的方式，讲参数传递给后面的拦截器的方式进行传递。这样做的好处是拦截器的逻辑比较灵活，可以在后面的拦截器处理完并返回结果后仍然执行自己的逻辑；缺点是逻辑没有前者清晰。

## 三、[Address](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FAddress.java)类详解

老规矩，先来看他的类注释



```dart
/**
 * A specification for a connection to an origin server. For simple connections, this is the
 * server's hostname and port. If an explicit proxy is requested (or {@linkplain Proxy#NO_PROXY no
 * proxy} is explicitly requested), this also includes that proxy information. For secure
 * connections the address also includes the SSL socket factory, hostname verifier, and certificate
 * pinner.
 *
 * <p>HTTP requests that share the same {@code Address} may also share the same {@link Connection}.
 */
```

翻译如下:与服务器连接的格式，对于简单的链接，这里是服务器的主机名和端口号。如果是通过代理(Proxy)的链接，则包含代理信息(Proxy)。如果是安全链接，则还包括SSL socket Factory、hostname验证器，证书等。
 通过翻译大家可以理解为一个地址的包装类，封装了地址的所有可能，说白了address描述了建立连接的所有配置信息
 再来看下它的字段构造函数



```dart
  final HttpUrl url;
  final Dns dns;
  final SocketFactory socketFactory;
  final Authenticator proxyAuthenticator;
  final List<Protocol> protocols;
  final List<ConnectionSpec> connectionSpecs;
  final ProxySelector proxySelector;
  final Proxy proxy;
  final SSLSocketFactory sslSocketFactory;
  final HostnameVerifier hostnameVerifier;
  final CertificatePinner certificatePinner;

  public Address(String uriHost, int uriPort, Dns dns, SocketFactory socketFactory,
      SSLSocketFactory sslSocketFactory, HostnameVerifier hostnameVerifier,
      CertificatePinner certificatePinner, Authenticator proxyAuthenticator, Proxy proxy,
      List<Protocol> protocols, List<ConnectionSpec> connectionSpecs, ProxySelector proxySelector) {
    this.url = new HttpUrl.Builder()
        .scheme(sslSocketFactory != null ? "https" : "http")
        .host(uriHost)
        .port(uriPort)
        .build();

    if (dns == null) throw new NullPointerException("dns == null");
    this.dns = dns;

    if (socketFactory == null) throw new NullPointerException("socketFactory == null");
    this.socketFactory = socketFactory;

    if (proxyAuthenticator == null) {
      throw new NullPointerException("proxyAuthenticator == null");
    }
    this.proxyAuthenticator = proxyAuthenticator;

    if (protocols == null) throw new NullPointerException("protocols == null");
    this.protocols = Util.immutableList(protocols);

    if (connectionSpecs == null) throw new NullPointerException("connectionSpecs == null");
    this.connectionSpecs = Util.immutableList(connectionSpecs);

    if (proxySelector == null) throw new NullPointerException("proxySelector == null");
    this.proxySelector = proxySelector;

    this.proxy = proxy;
    this.sslSocketFactory = sslSocketFactory;
    this.hostnameVerifier = hostnameVerifier;
    this.certificatePinner = certificatePinner;
  }
```

果然和咱们想的一样，就是一个地址的包装类，包含了三种请求类型的封装1直连，2走代理，3ssl 。

这里先简单的说下Address的构造函数，咱们回想一下什么时候new的Address对象，记忆好的同学可能想起来了，在RetryAndFollowUpInterceptor类里面，我们曾经创建过StreamAllocation类，在构造这个StreamAllocation的对象的时候，需要传入一个Addres对象，而在RetryAndFollowUpInterceptor类中则是通过createAddress来创建的Address对象的



```csharp
  private Address createAddress(HttpUrl url) {
    SSLSocketFactory sslSocketFactory = null;
    HostnameVerifier hostnameVerifier = null;
    CertificatePinner certificatePinner = null;
    if (url.isHttps()) {
      sslSocketFactory = client.sslSocketFactory();
      hostnameVerifier = client.hostnameVerifier();
      certificatePinner = client.certificatePinner();
    }

    return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
        sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
        client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
  }
```

通过构createAddress方法，我们发现除了uriHost和uriPort外的所有构造函数的参数均来自OkHttpClient，而Address的url字段正式根据这两个字段构造的，由此可见，Address的url字段仅仅包含HTTP请求的url的schema+host+port三部分的信息，而不包含path和query等信息。

让我们再来看下它的一个重要方法equalsNonHost



```kotlin
  boolean equalsNonHost(Address that) {
    return this.dns.equals(that.dns)
        && this.proxyAuthenticator.equals(that.proxyAuthenticator)
        && this.protocols.equals(that.protocols)
        && this.connectionSpecs.equals(that.connectionSpecs)
        && this.proxySelector.equals(that.proxySelector)
        && equal(this.proxy, that.proxy)
        && equal(this.sslSocketFactory, that.sslSocketFactory)
        && equal(this.hostnameVerifier, that.hostnameVerifier)
        && equal(this.certificatePinner, that.certificatePinner)
        && this.url().port() == that.url().port();
  }
```

用来返回两个Address是否是同一个地址，这个方法什么时候会被调用那，会在连接池复用的时候调用，因为只有两个Address相同才能说明这两个连接的配置信息是一直的，才能使用RealConnection的复用。看到方法内部大家发现，这个"相同"的要求还是很严格的，必须所有配置信息都一直才可以。

至此这个类基本已经讲解完毕，后续流程涉及到再提及。

## 四、[Route](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2FRoute.java)

老规矩先看类的注释



```dart
/**
 * The concrete route used by a connection to reach an abstract origin server. When creating a
 * connection the client has many options:
 *
 * <ul>
 *     <li><strong>HTTP proxy:</strong> a proxy server may be explicitly configured for the client.
 *         Otherwise the {@linkplain java.net.ProxySelector proxy selector} is used. It may return
 *         multiple proxies to attempt.
 *     <li><strong>IP address:</strong> whether connecting directly to an origin server or a proxy,
 *         opening a socket requires an IP address. The DNS server may return multiple IP addresses
 *         to attempt.
 * </ul>
 *
 * <p>Each route is a specific selection of these options.
 */
```

简单翻译下就是：
 连接使用的路由到抽象服务器。创建连接时，客户端有很多选择
 1、HTTP proxy(http代理)：已经为客户端配置了一个专门的代理服务器，否则会通过net.ProxySelector proxy selector尝试多个代理
 2、IP address(ip地址)：无论是通过直连还是通过代理，DNS服务器可能会尝试多个ip地址。
 每一个路由都是上述路由的一种格式
 所以我的理解就是OkHttp3中抽象出来的Route是描述网络数据包传输的路径，最主要还是描述直接与其建立TCP连接的目标端点。

现在看下他的字段和构造函数



```java
  final Address address;
  final Proxy proxy;
  final InetSocketAddress inetSocketAddress;

  public Route(Address address, Proxy proxy, InetSocketAddress inetSocketAddress) {
    if (address == null) {
      throw new NullPointerException("address == null");
    }
    if (proxy == null) {
      throw new NullPointerException("proxy == null");
    }
    if (inetSocketAddress == null) {
      throw new NullPointerException("inetSocketAddress == null");
    }
    this.address = address;
    this.proxy = proxy;
    this.inetSocketAddress = inetSocketAddress;
  }
```

所以咱咱们知道Route通过代理服务器的信息proxy,及链接的目标地址Address来描述路由即Route，连接的目标地址inetSocketAddress根据代理类型的不同而有着不同的含义，这主要是通过不同代理协议的差异而造成的。对于无需代理的情况，连接的目标地址inetSocketAddress中包含HTTP服务器经过DNS域名解析的IP地址以及协议端口号；对于SOCKET代理其中包含HTTP服务器的域名及协议端口号；对于HTTP代理，其中则包含代理服务器经过域名解析的IP地址及端口号。

这里面说一个后面能用到的requiresTunnel()方法



```csharp
  /**
   * Returns true if this route tunnels HTTPS through an HTTP proxy. See <a
   * href="http://www.ietf.org/rfc/rfc2817.txt">RFC 2817, Section 5.2</a>.
   */
  public boolean requiresTunnel() {
    //是HTTP请求，但是还有SSL
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
  }
```

即对于设置了HTTP代理，且安全的连接(SSL)需要请求代理服务器，建立一个到目标HTTP服务器的隧道连接，客户端与HTTP代理建立TCP连接，以此请求HTTP代理服务器在客户端与HTTP服务器之间进行数据的盲目转发
 即对于设置了HTTP代理，且安全的连接 (SSL) 需要请求代理服务器建立一个到目标HTTP服务器的隧道连接，客户端与HTTP代理建立TCP连接，以此请求HTTP代理服务在客户端与HTTP服务器之间进行数据的盲转发。

## 五、[RouteDatabase](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2Finternal%2Fconnection%2FRouteDatabase.java)

先来看看类注释



```dart
/**
 * A blacklist of failed routes to avoid when creating a new connection to a target address. This is
 * used so that OkHttp can learn from its mistakes: if there was a failure attempting to connect to
 * a specific IP address or proxy server, that failure is remembered and alternate routes are
 * preferred.
 */
```

简单翻译一下就是：
 当在创建与目标地址的链接时，为了避免重复出现路由故障而创建的黑名单，如果尝试链接特定的IP或者代理服务器最后失败了，将记住这些故障。
 这个类很简单， 大家看下



```java
  private final Set<Route> failedRoutes = new LinkedHashSet<>();

  /** Records a failure connecting to {@code failedRoute}. */
  public synchronized void failed(Route failedRoute) {
    failedRoutes.add(failedRoute);
  }

  /** Records success connecting to {@code failedRoute}. */
  public synchronized void connected(Route route) {
    failedRoutes.remove(route);
  }

  /** Returns true if {@code route} has failed recently and should be avoided. */
  public synchronized boolean shouldPostpone(Route route) {
    return failedRoutes.contains(route);
  }
```

看了代码，相信大家都知道了，这个类，内部维护了一个LinkHashSet().如果链接失败了，就放进去，如果成功就删除，还提供了一个判断是否包含路由的方法，用来判断该rount是否存在于LinkedHashSet()里面

## 六、[RouteSelector](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2Finternal%2Fconnection%2FRouteSelector.java)

**(1)先来看注释：**

```dart
/**
 * Selects routes to connect to an origin server. Each connection requires a choice of proxy server,
 * IP address, and TLS mode. Connections may also be recycled.
 */
```

翻译一下就是：
 这个类主要是选择连接到服务器的路由，每个连接应该是代理服务器/IP地址/TLS模式 三者中的一种。连接月可以被回收
 所以可以把RouteSelector理解为路由选择器.

**(2)那为什么需要RouteSelector那？**

因为HTTP请求处理过程中所需的TCP连接建立过程，主要是找到一个Route，然后依据代理协议规则与特定目标建立TCP连接。对于无代理的情况，是与HTTP服务器建立TCP连接，对于SOCKS代理和http代理，是与代理服务器建立tcp连接，虽然都是与代理服务器建立tcp连接，但是SOCKS代理协议和http代理协议又有一定的区别。
 而且借助于域名做负均衡已经是网络中非常常见的手法了，因而，常常会有域名对应不同IP地址的情况。同时相同系统也可以设置多个代理，这使Route的选择变得非常复杂。
 在OKHTTP中，对Route连接有一定的错误处理机制。OKHTTP会逐个尝试找到Route建立TCP连接，直到找到可用的哪一个。这样对Route信息有良好的管理。OKHTTP中借助RouteSelector类管理所有路由信息，并帮助选择路由。

**(3)看一下它的字段和构造函数**

```java
  private final Address address;
  private final RouteDatabase routeDatabase;

  /* The most recently attempted route. */
  private Proxy lastProxy;
  private InetSocketAddress lastInetSocketAddress;

  /* State for negotiating the next proxy to use. */
  private List<Proxy> proxies = Collections.emptyList();
  private int nextProxyIndex;

  /* State for negotiating the next socket address to use. */
  private List<InetSocketAddress> inetSocketAddresses = Collections.emptyList();
  private int nextInetSocketAddressIndex;

  /* State for negotiating failed routes */
  private final List<Route> postponedRoutes = new ArrayList<>();

  public RouteSelector(Address address, RouteDatabase routeDatabase) {
    this.address = address;
    this.routeDatabase = routeDatabase;

    resetNextProxy(address.url(), address.proxy());
  }

  /** Prepares the proxy servers to try. */
  private void resetNextProxy(HttpUrl url, Proxy proxy) {
    if (proxy != null) {
     //第一种方式
      // If the user specifies a proxy, try that and only that.
      proxies = Collections.singletonList(proxy);
    } else {
     //第二种方式
      // Try each of the ProxySelector choices until one connection succeeds.
      List<Proxy> proxiesOrNull = address.proxySelector().select(url.uri());
      proxies = proxiesOrNull != null && !proxiesOrNull.isEmpty()
          ? Util.immutableList(proxiesOrNull)
          : Util.immutableList(Proxy.NO_PROXY);
    }
    nextProxyIndex = 0;
  }
```

RouteSelector这个类的字段和构造函数比较简单，但是在构造函数里面调用了resetNextProxy()方法
 收集路由主要分为两个步骤：第一步收集所有的代理；第二步则是收集特定的代理服务器选择所有的*连接目标的地址*。
 收集代理的过程正如上面的这段代码所示，有两种方式：
 一是外部通过address传入代理，此时代理集合将包含这唯一的代理。address的代理最终来源于OkHttpClient，我们可以在构造OkHttpClient时设置代理，来指定该client执行的所有请求特定的代理。
 二是，借助于ProxySelectory获得多个代理。ProxySelector最终也来源于OkHttpClient用户当然也可以对此进行配置。但通常情况下，使用系统默认收集的所有代理保存在列表proxies中
 为OkHttpClient配置Proxy或ProxySelector的场景大概是，需要让连接使用代理，但不使用系统的代理配置情况。
 PS：proxies是在这时候被初始化的。inetSocketAddresses也是在这里被初始化，并且添加的第一个元素

**(4)hasNext()方法**

```cpp
  /**
   * Returns true if there's another route to attempt. Every address has at least one route.
   */
  public boolean hasNext() {
    return hasNextInetSocketAddress()
        || hasNextProxy()
        || hasNextPostponed();
  }

  /** Returns true if there's another proxy to try. */
 //是否还有代理
  private boolean hasNextProxy() {
    return nextProxyIndex < proxies.size();
  }

  /** Returns true if there's another socket address to try. */ 
  //是否还有socket地址
  private boolean hasNextInetSocketAddress() {
    return nextInetSocketAddressIndex < inetSocketAddresses.size();
  }

  /** Returns true if there is another postponed route to try. */
  //是否还有延迟路由
  private boolean hasNextPostponed() {
    return !postponedRoutes.isEmpty();
  }
```

hasNext()表明是否有可以使用的路由
 里面做了三个判断，如果满足一条就可以表明有可以使用的路由 ：1是否还有代理、2是否还有Socket、3是否还有延迟路由，如果三者都没有，则认为没有了。

**(5)next()方法**

下面介绍一下他的一个重要方法next()方法。
 收集 一个特定代理服务器选择下的 *连接目标地址* ，因代理类型的不同而不同，这里主要分3种情况：

> - 1、对于没有配置代理的情况，会对HTTP服务器的域名进行DNS域名解析，并为每个解析到的IP地址创建 *连接的目标地址*
> - 2、对于SOCKS代理，直接以HTTP的服务器的域名以及协议端口创建 *连接目标地址*
> - 3、对于HTTP代理，则会对HTTP代理服务器的域名进行DNS域名解析，并为每个解析到的IP地址创建 *连接的目标地址*

这里其实就是OkHttp发生DNS域名解析的场所。对于使用代理的场景，没有对HTTP服务器的域名做DNS域名解析，也就意味着HTTP服务器的域名解析要由代理服务器完成。
 代理服务器的收集是在创建RouteSelector完成的；而一个特定的代理服务器选择下，*连接目标地址* 收集则是在选择Route时根据需要完成的。
 代码如下:

```csharp
  public Route next() throws IOException {
    // Compute the next route to attempt.
    if (!hasNextInetSocketAddress()) {
      if (!hasNextProxy()) {
        if (!hasNextPostponed()) {
          throw new NoSuchElementException();
        }
        return nextPostponed();
      }
      lastProxy = nextProxy();
    }
    lastInetSocketAddress = nextInetSocketAddress();

    Route route = new Route(address, lastProxy, lastInetSocketAddress);
    if (routeDatabase.shouldPostpone(route)) {
      postponedRoutes.add(route);
      // We will only recurse in order to skip previously failed routes. They will be tried last.
      return next();
    }

    return route;
  }
```

这个方法主要是通过收集路由来选择路由。里面分了三种情况

> - 1、如果hasNextPostponed()，则return nextPostponed()。
> - 2、如果hasNextProxy()，则通过nextProxy()获取上一个代理，并用他去构造一个route，如果在失败链接的数据库里面有这个route，则最后通过递归调用next()，否则返回route
> - 3、如果hasNextInetSocketAddress()，则通过nextInetSocketAddress()获取上一个InetSocketAddress，并用他去构造一个route，如果在这个失败里面数据中有这个路由，然后继续通过递归调用next()方法，或者直接返回route。

那么首先我们来看下nextPostponed()这个方法



```csharp
  /** Returns the next postponed route to try. */
  private Route nextPostponed() {
    return postponedRoutes.remove(0);
  }
```

postponedRoutes是一个list，里面存放的是之前失败链接的路由，目的是在前所有不符合的情况，把之前失败的路由再试一次。
 再来看一下nextProxy()方法



```csharp
  /** Returns the next proxy to try. May be PROXY.NO_PROXY but never null. */
  private Proxy nextProxy() throws IOException {
    if (!hasNextProxy()) {
      throw new SocketException("No route to " + address.url().host()
          + "; exhausted proxy configurations: " + proxies);
    }
    Proxy result = proxies.get(nextProxyIndex++);
    resetNextInetSocketAddress(result);
    return result;
  }
```

这个就是从proxies里面去一个出来，proxies是在构造函数里面方法resetNextProxy()来赋值的。
 咱们再来看下nextInetSocketAddress()方法



```csharp
  /** Returns the next socket address to try. */
  private InetSocketAddress nextInetSocketAddress() throws IOException {
    if (!hasNextInetSocketAddress()) {
      throw new SocketException("No route to " + address.url().host()
          + "; exhausted inet socket addresses: " + inetSocketAddresses);
    }
    return inetSocketAddresses.get(nextInetSocketAddressIndex++);
  }
```

这个就是从inetSocketAddresses里面取一个出来，proxies是在构造函数里面方法resetNextProxy()来赋值的。

**(6)connectFailed()方法**

通过维护失败的路由信息，以避免浪费时间去连接一切不可用的路由。RouteSelector借助于RouteDatabase维护失败的路由信息。
 综上所述RouteSelector在OkHttp里面主要负责三件事，1收集路由信息，2选择路由，3维护失败路由。

## 七、 RetryAndFollowUpInterceptor 类详解

[RetryAndFollowUpInterceptor](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2Finternal%2Fhttp%2FRetryAndFollowUpInterceptor.java) 负责失败重连以及重定向

```java
/**
 * This interceptor recovers from failures and follows redirects as necessary. It may throw an
 * {@link IOException} if the call was canceled.
 */
public final class RetryAndFollowUpInterceptor implements Interceptor {
  /**
   * How many redirects and auth challenges should we attempt? Chrome follows 21 redirects; Firefox,
   * curl, and wget follow 20; Safari follows 16; and HTTP/1.0 recommends 5.
   */
  //最大恢复追逐次数：
  private static final int MAX_FOLLOW_UPS = 20;

  public RetryAndFollowUpInterceptor(OkHttpClient client, boolean forWebSocket) {
    this.client = client;
    this.forWebSocket = forWebSocket;
  }

@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    // 三个参数分别对应：(1)全局的连接池，(2)连接线路Address, (3)堆栈对象
    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response = null;
      boolean releaseConnection = true;
      try {
        //  执行下一个拦截器，即BridgeInterceptor
       // 这里有个很重的信息，即会将初始化好的连接对象传递给下一个拦截器，也是贯穿整个请求的连击对象，上面我们说过，在拦截器执行过程中，RealInterceptorChain的几个属性字段会一步一步赋值
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        //  如果有异常，判断是否要恢复
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
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
       // 检查是否符合要求
      Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        // 返回结果
        return response;
      }
       //不符合，关闭响应流
      closeQuietly(response.body());
       // 是否超过最大限制
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }
       // 是否有相同的连接
      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(
            client.connectionPool(), createAddress(followUp.url()), callStackTrace);
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```

我们知道每个拦截器都实现了接口interceptor,interceptor.intercept()方法就是子类用来处理自己的业务逻辑，所以我们仅仅需要分析这个方法即可。看源码我们得出了如下流程
 1 根据url创建一个Address对象，初始化一个Socket连接对象，基于Okio



```csharp
  private Address createAddress(HttpUrl url) {
    SSLSocketFactory sslSocketFactory = null;
    HostnameVerifier hostnameVerifier = null;
    CertificatePinner certificatePinner = null;
    if (url.isHttps()) {
      sslSocketFactory = client.sslSocketFactory();
      hostnameVerifier = client.hostnameVerifier();
      certificatePinner = client.certificatePinner();
    }

    return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
        sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
        client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
  }
```

2 用前面创建的address作为参数去实例化StreamAllocation
 PS：此处还没有真正的去建立连接，只是初始化一个连接对象
 3 开启一个while(true)循环
 4 如果取消，释放资源并抛出异常，结束流程
 5 执行下一个拦截器，一般是BridgeInterceptor
 6  如果发生异常，走到catch里面，判断是否继续请求，不继续请求则退出
 7  如果priorResponse不为空，则说明前面已经获取到了响应，这里会结合当前获取的Response和先前的Response
 8 调用followUpRequest查看响应是否需要重定向，如果不需要重定向则返回当前请求
 9 重定向次数+1，同时判断是否达到最大限制数量。是：退出
 10 检查是否有相同的链接，是：释放，重建创建
 11 重新设置request，并把当前的Response保存到priorResponse，继续while循环

我们来看下重定向的判断followUpRequest



```java
/**
   * Figures out the HTTP request to make in response to receiving {@code userResponse}. This will
   * either add authentication headers, follow redirects or handle a client request timeout. If a
   * follow-up is either unnecessary or not applicable, this returns null.
   */
  private Request followUpRequest(Response userResponse) throws IOException {
    if (userResponse == null) throw new IllegalStateException();
    Connection connection = streamAllocation.connection();
    Route route = connection != null
        ? connection.route()
        : null;
    int responseCode = userResponse.code();

    final String method = userResponse.request().method();
    switch (responseCode) {
      case HTTP_PROXY_AUTH:
        Proxy selectedProxy = route != null
            ? route.proxy()
            : client.proxy();
        if (selectedProxy.type() != Proxy.Type.HTTP) {
          throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
        }
        return client.proxyAuthenticator().authenticate(route, userResponse);

      case HTTP_UNAUTHORIZED:
        return client.authenticator().authenticate(route, userResponse);

      case HTTP_PERM_REDIRECT:
      case HTTP_TEMP_REDIRECT:
        // "If the 307 or 308 status code is received in response to a request other than GET
        // or HEAD, the user agent MUST NOT automatically redirect the request"
        if (!method.equals("GET") && !method.equals("HEAD")) {
          return null;
        }
        // fall-through
      case HTTP_MULT_CHOICE:
      case HTTP_MOVED_PERM:
      case HTTP_MOVED_TEMP:
      case HTTP_SEE_OTHER:
        // Does the client allow redirects?
        if (!client.followRedirects()) return null;

        String location = userResponse.header("Location");
        if (location == null) return null;
        HttpUrl url = userResponse.request().url().resolve(location);

        // Don't follow redirects to unsupported protocols.
        if (url == null) return null;

        // If configured, don't follow redirects between SSL and non-SSL.
        boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
        if (!sameScheme && !client.followSslRedirects()) return null;

        // Most redirects don't include a request body.
        Request.Builder requestBuilder = userResponse.request().newBuilder();
        if (HttpMethod.permitsRequestBody(method)) {
          final boolean maintainBody = HttpMethod.redirectsWithBody(method);
          if (HttpMethod.redirectsToGet(method)) {
            requestBuilder.method("GET", null);
          } else {
            RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
            requestBuilder.method(method, requestBody);
          }
          if (!maintainBody) {
            requestBuilder.removeHeader("Transfer-Encoding");
            requestBuilder.removeHeader("Content-Length");
            requestBuilder.removeHeader("Content-Type");
          }
        }

        // When redirecting across hosts, drop all authentication headers. This
        // is potentially annoying to the application layer since they have no
        // way to retain them.
        if (!sameConnection(userResponse, url)) {
          requestBuilder.removeHeader("Authorization");
        }

        return requestBuilder.url(url).build();

      case HTTP_CLIENT_TIMEOUT:
        // 408's are rare in practice, but some servers like HAProxy use this response code. The
        // spec says that we may repeat the request without modifications. Modern browsers also
        // repeat the request (even non-idempotent ones.)
        if (userResponse.request().body() instanceof UnrepeatableRequestBody) {
          return null;
        }

        return userResponse.request();

      default:
        return null;
    }
  }
```

这里主要是根据响应码(code)和响应头(header)，查看是否需要重定向，并重新设置请求，当然，如果是正常响应则直接返回Response停止循环



```java
/**
   * Report and attempt to recover from a failure to communicate with a server. Returns true if
   * {@code e} is recoverable, or false if the failure is permanent. Requests with a body can only
   * be recovered if the body is buffered or if the failure occurred before the request has been
   * sent.
   */
  private boolean recover(IOException e, boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);
     // 1. 应用层配置不在连接，默认为true
    // The application layer has forbidden retries.
    if (!client.retryOnConnectionFailure()) return false;
     // 2. 请求Request出错不能继续使用
    // We can't send the request body again.
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;
    //  是否可以恢复的
    // This exception is fatal.
    if (!isRecoverable(e, requestSendStarted)) return false;
    // 4. 没用更多线路可供选择
    // No more routes to attempt.
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }

  private boolean isRecoverable(IOException e, boolean requestSendStarted) {
    // If there was a protocol problem, don't recover.
    if (e instanceof ProtocolException) {
      return false;
    }

    // If there was an interruption don't recover, but if there was a timeout connecting to a route
    // we should try the next route (if there is one).
    if (e instanceof InterruptedIOException) {
      return e instanceof SocketTimeoutException && !requestSendStarted;
    }

    // Look for known client-side or negotiation errors that are unlikely to be fixed by trying
    // again with a different route.
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

    // An example of one we might want to retry with a different route is a problem connecting to a
    // proxy and would manifest as a standard IOException. Unless it is one we know we should not
    // retry, we return true and try a new route.
    return true;
  }
```

看上面代码可以这样理解:判断是否可以恢复如果下面几种条件符合，则返回true，代表可以恢复，如果返回false，代表不可恢复。

1. 应用层配置不在连接(默认为true)，则不可恢复
2. 请求Request是不可重复使用的Request，则不可恢复
3. 根据Exception的类型判断是否可以恢复的 (isRecoverable()方法)
    3.1、如果是协议错误（ProtocolException）则不可恢复
    3.2、如果是中断异常（InterruptedIOException）则不可恢复
    3.3、如果是SSL握手错误（SSLHandshakeException && CertificateException）则不可恢复
    3.4、certificate pinning错误（SSLPeerUnverifiedException）则不可恢复
4. 没用更多线路可供选择 则不可恢复
    如果上述条件都不满足，则这个request可以恢复

综上所述：
 一个循环来不停的获取response。每循环一次都会获取下一个request，如果没有，则返回response，退出循环。而获取下一个request的逻辑，是根据上一个response返回的状态码，分别作处理。

## 四 BridgeInterceptor 类详解

[BridgeInterceptor](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2Finternal%2Fhttp%2FBridgeInterceptor.java) :负责对Request和Response报文进行加工，具体如下：

> - 1、请求从应用层数据类型类型转化为网络调用层的数据类型。
> - 2、将网络层返回的数据类型 转化为 应用层数据类型。
> - 3、补充：Keep－Alive 连接：

区别如下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-50b9e0b33cad8183.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Paste_Image.png

看下源码：



```java
   @Override
   // 主要方法，其他略
   // 此拦截器较为简单，其中有两点比较重要，1、cookie的处理 2Gzip
    public Response intercept(Interceptor.Chain chain) throws IOException {
        Request userRequest = chain.request();
        Request.Builder requestBuilder = userRequest.newBuilder();
        RequestBody body = userRequest.body();
        if (body != null) {
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
        }
        if (userRequest.header("Host") == null) {
            requestBuilder.header("Host", hostHeader(userRequest.url(), false));
        }
        if (userRequest.header("Connection") == null) {
            requestBuilder.header("Connection", "Keep-Alive");
        }
        // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
        // the transfer stream.
        boolean transparentGzip = false;
        if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
            transparentGzip = true;
            requestBuilder.header("Accept-Encoding", "gzip");
        }
         // 所以返回的cookies不能为空，否则这里会报空指针
        List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
        if (!cookies.isEmpty()) {
            // 创建Okhpptclitent时候配置的cookieJar，
            requestBuilder.header("Cookie", cookieHeader(cookies));
        }
        if (userRequest.header("User-Agent") == null) {
            requestBuilder.header("User-Agent", Version.userAgent());
        }

        //  以上为请求前的头处理
        Response networkResponse = chain.proceed(requestBuilder.build());
         // 以下是请求完成，拿到返回后的头处理
         // 响应header， 如果没有自定义配置cookie不会解析
        HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
        Response.Builder responseBuilder = networkResponse.newBuilder()
                .request(userRequest);
         // 前面解析完header后，判断服务器是否支持gzip压缩格式，如果支持将交给Okio处理
        if (transparentGzip
                && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
                && HttpHeaders.hasBody(networkResponse)) {
            GzipSource responseBody = new GzipSource(networkResponse.body().source());
            Headers strippedHeaders = networkResponse.headers().newBuilder()
                    .removeAll("Content-Encoding")
                    .removeAll("Content-Length")
                    .build();
            responseBuilder.headers(strippedHeaders);
            // 处理完成后，重新生成一个response
            responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
        }
        return responseBuilder.build();
    }
```

读了源码发现这个interceptor比较简单，可以分为发送请求和响应两个阶段来看:
 1.在发送阶段BridgeInterceptor补全了一些header包括*Content-Type*、*Content-Length*、*Transfer-Encoding*、*Host*、*Connection*、*Accept-Encoding*、*User-Agent*。
 2.如果需要gzip压缩则进行gzip压缩
 3.加载*Cookie*
 4.随后创建新的request并交付给后续的interceptor来处理，以获取响应。
 5.首先保存*Cookie*
 6.如果服务器返回的响应content是以gzip压缩过的，则会先进行解压缩，移除响应中的header Content-Encoding和Content-Length，构造新的响应返回。
 7 否则直接返回response

其中* CookieJar*来自* OkHttpClient*,他是OKHttp的Cookie管理类，负责Cookie的存取。



```dart
public interface CookieJar {
  /** A cookie jar that never accepts any cookies. */
  CookieJar NO_COOKIES = new CookieJar() {
    @Override public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
    }

    @Override public List<Cookie> loadForRequest(HttpUrl url) {
      return Collections.emptyList();
    }
  };

  /**
   * Saves {@code cookies} from an HTTP response to this store according to this jar's policy.
   *
   * <p>Note that this method may be called a second time for a single HTTP response if the response
   * includes a trailer. For this obscure HTTP feature, {@code cookies} contains only the trailer's
   * cookies.
   */
  void saveFromResponse(HttpUrl url, List<Cookie> cookies);

  /**
   * Load cookies from the jar for an HTTP request to {@code url}. This method returns a possibly
   * empty list of cookies for the network request.
   *
   * <p>Simple implementations will return the accepted cookies that have not yet expired and that
   * {@linkplain Cookie#matches match} {@code url}.
   */
  List<Cookie> loadForRequest(HttpUrl url);
}
```

由于OKHttpClient默认的构造过程可以看到，OKHttp默认是没有提供Cookie管理功能的，所以如果想增加Cookie管理需要重写里面的方法,PS:如果重写CookieJar()需要注意loadForRequest()方法的返回值不能为null



```cpp
  public static void receiveHeaders(CookieJar cookieJar, HttpUrl url, Headers headers) {
      // 没有配置，不解析
    if (cookieJar == CookieJar.NO_COOKIES) return;
    // 此处遍历，解析Set-Cookie的值，比如max-age
    List<Cookie> cookies = Cookie.parseAll(url, headers);
    if (cookies.isEmpty()) return;
     // 然后保存，即自定义
    cookieJar.saveFromResponse(url, cookies);
  }
```



作者：隔壁老李头
链接：https://www.jianshu.com/p/e3b6f821acb8
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# OKHttp源码解析(五)--OKIO简介及FileSystem

## 一、[okio](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio)

说道okio就不能不提JDK里面io，那么咱们先简单说下JDK里面的io。

### (一)、javaNIO和阻塞I/O：

> - 1、阻塞I/O通信模式：调用InputStream.read()方法时是阻塞的，它会一直等到数据到来时才返回
> - 2、NIO通信模式：在JDK1.4开始，是一种非阻塞I/O，在Java NIO的服务端由一个专门的线程来处理所有I/O事件，并负责分发；线程之间通讯通过wait和notify等方式。

### (二)、okio

okio是由square公司开发的，它补充了java.io和java.nio的不足，以便能够更加方便，快速的访问、存储和处理你的数据。OKHttp底层也是用该库作为支持。而且okio使用起来很简单，减少了很多io操作的基本代码，并且对内存和CPU使用做了优化，他的主要功能封装在ByteString和Buffer这两个类中。

okio的作者认为，javad的JDK对字节流和字符流进行分开定义这一世情，并不是那么优雅，所以在okio并不进行这样划分。具体的做法就是把比特数据都交给Buffer管理，然后Buff实现BufferedSource和BufferedSink这两个接口，最后通过调用buffer相应的方法对数据进行编码。

## 二、okio的使用

假设我有一个hello.txt文件，内容是hello jianshu，现在我用okio把它读出来。
 那我们先理一下思路：读取文件的步骤是首先要拿到一个输入流，okio封装了许多输入流，统一使用的方法重载source方法转换成一个source，然后使用buffer方法包装成BufferedSource，这个里面提供了流的各种操作，读String,读byte，short等，甚至是16进制数。这里直接读出文件的内容，十分简单，代码如下：

```csharp
    public static void main(String[] args) {
        File file = new File("hello.txt");
        try {
            readString(new FileInputStream(file));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void readString(InputStream in) throws IOException {
      BufferedSource source = Okio.buffer(Okio.source(in));  //创建BufferedSource
      String s = source.readUtf8();  //以UTF-8读
      System.out.println(s);     //打印
      source.close();
    }
```

--------------------------------------输出-----------------------------------------
 hello jianshu
 Okio是对Java底层io的封装，所以底层io能做的Okio都能做。

上面的大体流程如下：
 第一步，首先是调用okio的source(InputStream in)方法获取Source对象
 第二步，调用okio的buffer(Source source)方法获取BufferedSource对象
 第三步，调用BufferedSource的readUtf8()方法读取String对象
 第四步，关闭BufferedSource

总结下大体流程如下：

> - 1.构建缓冲池，缓冲源对象
> - 2.读写操作
> - 3.关闭缓冲池

## 三、[Sink](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FSink.java)和[Source](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FSource.java)及其实现

### (一)、Sink和Source

在JDK里面有InputStream和OutputStream两个接口，由于okio作者认为Inputsream和OutputStream中的available()和读写单字节的方法纯属鸡肋，所以怼出Source和Sink接口。Source和Sink类似于InputStream和OutputStream，是io操作的顶级接口类,这两个接口均实现了Closeable接口。所以可以把Source简单的看成InputStream，Sink简单看成OutputStream。结构图如下图:



![img](https:////upload-images.jianshu.io/upload_images/5713484-cf40fe1898aed549.png?imageMogr2/auto-orient/strip|imageView2/2/w/638/format/webp)

okio1.png



其中Sink代表的是输出流，Source代表的是输入流，这两个基本上都是对称的。那我们先来分析下Sink，代码如下：



```java
public interface Sink extends Closeable, Flushable {
  /** Removes {@code byteCount} bytes from {@code source} and appends them to this. */
  // 定义基础的write操作,该方法将字节写入Buffer
  void write(Buffer source, long byteCount) throws IOException;

  /** Pushes all buffered bytes to their final destination. */
  @Override void flush() throws IOException;

  /** Returns the timeout for this sink. */
  Timeout timeout();

  /**
   * Pushes all buffered bytes to their final destination and releases the
   * resources held by this sink. It is an error to write a closed sink. It is
   * safe to close a sink more than once.
   */
  @Override void close() throws IOException;
}
```



```java
public interface Source extends Closeable {
  /**
   * Removes at least 1, and up to {@code byteCount} bytes from this and appends
   * them to {@code sink}. Returns the number of bytes read, or -1 if this
   * source is exhausted.
   */
    // 定义基础的read操作,该方法将字节写入Buffer
  long read(Buffer sink, long byteCount) throws IOException;

  /** Returns the timeout for this source. */
  Timeout timeout();

  /**
   * Closes this source and releases the resources held by this source. It is an
   * error to read a closed source. It is safe to close a source more than once.
   */
  @Override void close() throws IOException;
}
```

虽然Sink和Source只定义了很少的方法，这也是为何说它容易实现的原因，但是我们一般在使用过程中，不直接拿它使用，而是使用BufferedSink和BufferedSouce对接口的封装，因为在BufferedSinke和BufferedSource接口定义了一系列好用的方法。

### (二)、 [BufferedSinke](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FBufferedSink.java)和 [BufferedSource](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FBufferedSource.java)

看源码可知BufferedSink和BufferedSource定义了很多方便的方法如下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-dd1220041c348b31.png?imageMogr2/auto-orient/strip|imageView2/2/w/384/format/webp)

BufferedSinke.png

但是发现BufferedSink和BufferedSource两个都是接口 ，那么他的具体具体实现类是什么那？

### (三)、 [RealBufferedSink](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FRealBufferedSink.java)和 [RealBufferedSource](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FRealBufferedSource.java)

因为RealBufferedSink和RealBufferedSource是一一对应的，我就讲解RealBufferedSInk了，RealBufferedSource这里就不仔细讲解了。



```java
final class RealBufferedSink implements BufferedSink {
    public final Buffer buffer = new Buffer();
    public final Sink sink;

    public RealBufferedSink(Sink sink){
        if (sink == null)
            throw new NullPointerException("sink == null");
        this.sink = sink;
    }
}
```

大家看源代码可知RealBufferedSink类中有两个主要参数，一个是新建的Buffer对象，一个是Sink对象。虽然这个类叫做RealBufferedSinke，但是实际上这个只是一个保存Buffer对象的代理而已，真生的实现都是在Buffer中实现的。



```java
  @Override public BufferedSink write(byte[] source) throws IOException {
    if (closed) throw new IllegalStateException("closed");
    buffer.write(source);
    return emitCompleteSegments();
  }

  @Override public BufferedSink write(byte[] source, int offset, int byteCount) throws IOException {
    if (closed) throw new IllegalStateException("closed");
    buffer.write(source, offset, byteCount);
    return emitCompleteSegments();
  }
```

看上面代码可知，RealBufferedSink实现BufferedSink的方法实际上都是调用buffer对应的方法，对应的RealBufferedSource也是这样调用的buffer的read方法。

可以看到这个实现了BufferedSink接口的两个方法实际上都是调用了buffer的对应方法，对应的RealBufferedSource也是同样的调用buffer中的read方法，关于Buffer这个类会在下面详述，刚才我们看到Sink接口中有一个Timeout的类，这个就是Okio所实现的超时机制，保证了IO操作的稳定性。
 所以关于读写操作这块的类图如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-7404e49cbd97138f.png?imageMogr2/auto-orient/strip|imageView2/2/w/464/format/webp)

okio4.png



关于Buffer这个类，我们后面再说

### (四)其它实现类

1、Sink和Source它们还有各自的支持gzip压缩的实现类GzipSink和GzipSource
 2、具有委托功能的抽象类ForwardingSink和ForwardingSource，它们的具体实现类是InflaterSource和DeflaterSink，这两个类主要用于压缩，为GzipSink和GzipSource服务。
 综上所述关于类的结构图如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-7039ead52b1ae609.png?imageMogr2/auto-orient/strip|imageView2/2/w/873/format/webp)

okio2.png

刚刚我们看到在Sink接口中有一个Time类，这个就是okio中实现超时机制的接口，用于保证IO操作的稳定性。

## 四、Segment和SegmentPool解析

### (一)、[Segment](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FSegment.java)

#### 1、Segment字面的意思就是片段，okio将数据也就是Buffer分割成一块块的片段，内部维护者固定长度的byte[]数组，同时segment拥有前面节点和后面节点，构成一个双向循环链表，就像下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-5521b9c5571587a5.png?imageMogr2/auto-orient/strip|imageView2/2/w/791/format/webp)

这样采取分片使用链表连接，片中使用数组存储，兼具读的连续性，以及写的可插入性，对比单一使用链表或者数组，是一种折中的方案，读写更快，而且有个好处根据需求改动分片的大小来权衡读写的业务操作，另外，segment也有一些内置的优化操作，综合这些okio才能大放异彩，后面解析Buffer时会讲解什么时候形成双向循环链表。



```java
      // 每一个segment所含数据的大小，固定的
    static final int SIZE = 8192;
     // Segments 用分享的方式避免复制数组
    static final int SHARE_MINIMUM = 1024;
  
    final byte[] data;
     // data[]中第一个可读的位置
    int pos;
     // data[]中第一个可写的位 
     //所以一个Segment的可读数据数量为pos~limit-1=limit-pos；limit和pos的有效值为0~SIZE-1
    int limit;
    //当前存储的data数据是其它对象共享的则为真  
    boolean shared;
    //是否是自己是操作者 
    boolean owner;
    ///前一个Segment
    Segment pre;
    ///下一个Segment
    Segment next;
```

总结一下就是：
 SIZE就是一个segment的最大字节数，其中还有一个SHARE_MINIMUM，这个涉及到segment优化的另一个技巧，共享内存，然后data就是保存的字节数组，pos，limit就是开始和结束点的index,shared和owner用来设置状态判断是否可读写，一个有共享内存的sement是不能写入的，pre，next就是前置后置节点。

#### 2、Segment的构造方法



```kotlin
Segment() {  
  this.data = new byte[SIZE];  
  this.owner = true;   
  this.shared = false; 
}  
Segment(Segment shareFrom) {  
  this(shareFrom.data, shareFrom.pos, shareFrom.limit);  
  shareFrom.shared = true;
}  
Segment(byte[] data, int pos, int limit) {  
  this.data = data;  
  this.pos = pos;  
  this.limit = limit;  
  this.owner = false; 
  this.shared = true;  
}  
```

> - 1、无参的构造函数，表明采用该构造器表明该数据data的所有者就是该Segment自己，所以owner为真，shared为假
> - 2、一个参数的构造函数，看代码我们知道这个数据来自其他Segment，所以表明这个Segment是被别人共享了，所以shared为真，owner为假
> - 3、三个参数构造函数表明数据直接使用外面的，所以share为真，owner为假

由上面代码可知，owner和shared这两个状态是互斥的，且赋值都是同步赋值的。

#### 3、Segment的几个方法分析

既然是双向循环链表，其中也会有一些操作的方法:

##### (1)pop()方法：

将当前Segment从Segment链中移除去将，返回下一个Segment代码如下



```kotlin
    public Segment pop(){
        Segment result = next != this ? next : null;
        pre.next = next;
        next.pre = pre;
        next = null;
        pre = null;
        return result;
    }
```

pop方法移除自己，首先将自己的前后两个节点连接起来，然后将自己的前后置空，这昂就脱离了整个双向链表，然后返回next。说道pop就一定会联想到push。

##### (2)push方法：

Segment中的push表示将一个Segment压入该Segment节点的后面，返回刚刚压入的Segment



```kotlin
    public Segment push(Segment segment){
        segment.pre = this;
        segment.next = next;
        next.pre = segment;
        next = segment;
        return segment;
    }
```

push方法就是在当前和next引用中间插入一个segment进来，并且返回插入的segment，这两个都是寻常的双向链表的操作，我们再来看看如何写入数据。

##### (3)writeTo方法



```java
  /** Moves {@code byteCount} bytes from this segment to {@code sink}. */
  public void writeTo(Segment sink, int byteCount) {
     //只能对自己操作
    if (!sink.owner) throw new IllegalArgumentException();
     //写的起始位置加上需要写的byteCount大于SIZE
    if (sink.limit + byteCount > SIZE) {
      // We can't fit byteCount bytes at the sink's current position. Shift sink first.
       //如果是共享内存，不能写，则抛异常
      if (sink.shared) throw new IllegalArgumentException();
      //如果不是共享内存，且写的起始位置加上byteCount减去头还大于SIZE则抛异常
      if (sink.limit + byteCount - sink.pos > SIZE) throw new IllegalArgumentException();
      //上述条件都不满足，我们需要先触动要写文件的地址，把sink.data从sink.pos这个位置移动到 0这个位置，移动的长度是limit - sink.pos，这样的好处就是sink初始位置是0
      System.arraycopy(sink.data, sink.pos, sink.data, 0, sink.limit - sink.pos);
      sink.limit -= sink.pos;
      sink.pos = 0;
    }
    //开始尾部写入 写完置limit地址
    System.arraycopy(data, pos, sink.data, sink.limit, byteCount);
    sink.limit += byteCount;
    pos += byteCount;//索引后移
  }
```

> PS:这里我们来复习一下arraycopy方法：
>  public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos, int length);
>  src:源数组；    srcPos:源数组要复制的起始位置；dest:目的数组；   destPos:目的数组放置的起始位置；length:复制的长度。
>  实现过程是这样的，先生成一个长度为length的临时数组,将圆数组中srcPos 到scrPos+lengh-1之间的数据拷贝到临时数组中。然后将这个临时数组复制到dest中。

读到上面的代码，我们发现owner和shared这两个状态是互斥的，赋值都是同步赋值的，这里我们有点明白这两个参数的意义了，如果是共享的就是无法写，以免污染数据，会抛出异常。
 如果要写的字节加上原有的字节大于单个segment的最大值也会抛出异常。
 还有一种情况就是，由于前面的pos索引可能因为read方法取出数据，pos所以后移这样导致可以容纳数据，这样可以先执行移动操作，使用JDK提供的System.arraycopy方法来移动到pos为0的状态，更改pos和limit索引后再在尾部写入byteCount数的数据，写完之后实际上原segment读了byteCount的数据，所以pos需要后移这么多。

##### (4) compact()方法(压缩机制)

除了写入数据之外，segment还有一个优化的技巧，因为每个segment的片段size是固定的，为了防止经过长时间的使用后，每个segment中的数据被分割的十分严重，可能一个很小的数据却占据了整个segment，所以有了一个压缩机制。



```cpp
  public void compact() {
    //上一个节点就是自己，意味着就一个节点，无法压缩，抛异常
    if (prev == this) throw new IllegalStateException();
     //如果上一个节点不是自己的，所以你是没有权利压缩的
    if (!prev.owner) return; // Cannot compact: prev isn't writable.
    //能进来说明，存在上一个节点，且上一个节点是自己的，可以压缩
    //记录当前Segment具有的数据，数据大小为limit-pos
    int byteCount = limit - pos;
    统计前结点是否被共享，如果共享则只记录Size-limit大小，如果没有被共享，则加上pre.pos之前的空位置；
    //本行代码主要是获取前一个segment的可用空间。先判断prev是否是共享的，如果是共享的，则只记录SIZE-limit,如果没有共享则记录SIZE-limit加上prev.pos之前的空位置
    int availableByteCount = SIZE - prev.limit + (prev.shared ? 0 : prev.pos);
   //判断prev的空余空间是否能够容纳Segment的全部数据，不能容纳则返回
    if (byteCount > availableByteCount) return; // Cannot compact: not enough writable space.
    //能容纳则将自己的这个部分数据写入上一个Segment
    writeTo(prev, byteCount);
    //讲当前Segment从Segment链表中移除
    pop();
    //回收该Segment
    SegmentPool.recycle(this);
  }
```

总结下上述代码：如果前面的Segment是共享的，那么不可写，也不能压缩，然后判断前一个的剩余大小是否比当前打，如果有足够的空间来容纳数据，调用前面的writeTo方法写入数据，写完以后，移除当前segment，然后回收segment。

##### (5)split()方法(共享机制)

还有一种机制是共享机制，为了减少数据复制带来的性能开销。
 本方法用于将Segment一分为二，将pos+1pos+btyeCount-1的内容给新的Segment,将pos+byteCountlimit-1的内容留给自己，然后将前面的新Segment插入到自己的前面。这里需要注意的是，虽然这里变成了两个Segment，但是实际上byte[]数据并没有被拷贝，两个Segment都引用了改Segment。



```swift
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

这个方法实际上经过很多次的改变，在回顾okio1.6时，发现有一个重要的差异就是多了一个SHARE_MININUM参数，同时也多了一个注释，为了防止一个很小的片段就进行共享，我们知道共享之后为了防止数据污染就无法写了，如果存在大量的共享片段，实际上是浪费资源的，所以通过这个对比可以看出这个最小数的的意义，而且这个方法在1.6的版本中检索实际上只有一个地方使用了这个方法，就是Buffer中的write方法，为了效率在移动大数据的时候直接移动了segment而不是data，这样在写数据上能达到很高的效率，具体的write细节会在后面讲解。
 在上面的方法中出现的 SegmentPool ，这是什么东西，那我们下面就来讲解一下

### (二) [SegmentPool](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FSegmentPool.java)

SegmentPool是一个Segment池，由一个单项链表构成。该池负责Segment的回收和闲置Segment管理，也就是说Buffer使用的Segment是从Segment单项链表中取出的，这样有效的避免了GC频率
 SegmentPool这个类比较简单，先来看下这个类的属性
 先看下



```java
//一个Segment记录的最大长度是8192，因此SegmentPool只能存储8个Segment
static final long MAX_SIZE = 64 * 1024;
//该SegmentPool存储了一个回收Segment的链表
static Segment next;
//该值记录了当前所有Segment的总大小，最大值是为MAX_SIZE
static long byteCount;  
```

上面代码表明了一个池子的上线也就是64K，相当于8个Segment，next这个属性表明这个SegmentPool是按照单链表的方式进行存储的，byteCount这个字段表明已经使用的大小。

SegmentPool的方法十分简单，就两个一个是"取"，一个是"收"



```java
  static Segment take() {
    synchronized (SegmentPool.class) {
      if (next != null) {
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

为了防止多线程同时操作出现问题，这里加了锁。注意：这里的next表示整个对象池的头，而不是"下一个"。如果next为null，则这个，池子里面没有Segment。否则就从里面拿出一个next出来，并将next的下一个节点赋值为next(即为头)，然后设置下一下byteCount。如果链表为空，则创建一个Segment对象返回。
 下面咱们来看下回收的方法



```java
  static void recycle(Segment segment) {
    //如果这个要回收的Segment被前后引用，则无法回收
    if (segment.next != null || segment.prev != null) throw new IllegalArgumentException();
    //如果这个要回收的Segment的数据是分享的，则无法回收
    if (segment.shared) return; // This segment cannot be recycled.
    //加锁
    synchronized (SegmentPool.class) {
      //如果 这个空间已经不足以再放入一个空的Segment，则不回收
      if (byteCount + Segment.SIZE > MAX_SIZE) return; // Pool is full.
      //设置SegmentPool的池大小
      byteCount += Segment.SIZE;
      //segment的下一个指向头
      segment.next = next;
      //设置segment的可读写位置为0
      segment.pos = segment.limit = 0;
      //设置当前segment为头
      next = segment;
    }
  }
```

上面代码很简单，就是尝试将参数Segment对象加入到自身的Segment链表中。其中做了一些条件判断，具体看注释。
 SegmentPool总结：
 1.大家注意到没有SegmentPool的作用就是管理多余的Segment，不直接丢弃废弃的Segment,等客户需要Segment的时候直接从池中获取一个对象，避免了重复创建新兑现，提高资源利用率。
 2.大家仔细读取源码会发现SegmentPool的所有操作都是基于对表头的操作。

## 五、[ByteString](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FByteString.java)

ByteString存储的是不可变比特序列，可能你不太理解这句话，如果给出final btye[] data 你是不是就秒懂了。官方文档说可以把它看成string的远方亲戚，且这个亲戚符合人工工学设计，逼格是不是很高。不过简单的讲，他就是一个byte序列(数组)，以制定的编码格式进行解码。目前支持的解码规则有hex，base64和UTF-8等，机智如你可能会说String也是如此。是的，你说的没错，ByteString 只是把这些方法进行了封装。免去了我们直接输入类似的"utf-8"这样的错误，直接通过调用utf-8格式进行解码，还做了优化，在第一次调用uft8()方法的时候得到了一个该解码的String，同时在ByteString内部还保留了这个引用，当再次调用utf-8()的时候，则直接返回这个引用。
 刚刚说了不可变对象。Effective Java 一书中有一条给了不可变对象的需要遵循的的几条原则：

> - 1.不要提供任何会修改对象状态的方法
> - 2.保证类不会被扩展
> - 3.使所有的域都是final的
> - 4.使所有的域都是private的
> - 5.确保对于任何可变组件的互斥访问

不可变对象有许多好处，首先本质是线程安全，不要求同步处理，也就是没有锁之类的性能问题，而且可以被自由的共享内部信息，当然坏处就是要创建大量类的对象,咱们看看ByteString的属性

### 1、ByteString属性

```java
//明显给16进制数准备的
static final char[] HEX_DIGITS = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f' };
  //一个空的对象 
  /** A singleton empty {@code ByteString}. */
  public static final ByteString EMPTY = ByteString.of();
  
  final byte[] data;
  //hashcode的值
  transient int hashCode; // Lazily computed; 0 if unknown.
  transient String utf8; // Lazily computed.
```

这里要重点说明下 byte[] data和transient String utf8，ByteString不仅是不可变的，同时在内部有两个filed，分别是byte数据，以及String数据，这样能够让这个类Byte和String转换上基本没有开销。同样需要保存这两份引用，这明显的空间换时间的方式，为了性能okio做了很多事情。这是这个String前面有transient关键字标记，也就是说不会进入序列化和反序列化，所以readObject和writeObject()中没有utf8属性

### 2、ByteString构造函数

```kotlin
  ByteString(byte[] data) {
    this.data = data; // Trusted internal constructor doesn't clone data.
  }
```

构造函数的参数为一个byte[]类型，不过构造函数只能被同包的类使用，因此我们创建ByteString对象并不是通过该方法。我们是如何构造一个ByteString对象？其实是通过 of()方法构造对象的

### 3、of()方法创建ByteString对象

```java
  /**
   * Returns a new byte string containing a clone of the bytes of {@code data}.
   */
  public static ByteString of(byte... data) {
    if (data == null) throw new IllegalArgumentException("data == null");
    return new ByteString(data.clone());
  }

  /**
   * Returns a new byte string containing a copy of {@code byteCount} bytes of {@code data} starting
   * at {@code offset}.
   */
  public static ByteString of(byte[] data, int offset, int byteCount) {
    if (data == null) throw new IllegalArgumentException("data == null");
    checkOffsetAndCount(data.length, offset, byteCount);

    byte[] copy = new byte[byteCount];
    System.arraycopy(data, offset, copy, 0, byteCount);
    return new ByteString(copy);
  }

  public static ByteString of(ByteBuffer data) {
    if (data == null) throw new IllegalArgumentException("data == null");

    byte[] copy = new byte[data.remaining()];
    data.get(copy);
    return new ByteString(copy);
```

ByteString 有三个of方法，都是return一个ByteString，咱们就一个一个的分析

> - 1.第一个of方法，就是调用clone方法，重新创建一个byte数组。clone一个数组的原因很简单，我们确保ByteString的data指向byte[]没有被其他对象所引用，否则就容易破坏ByteString中存储的是一个不可变化的的比特流数据这一约束。
> - 2.第二个of方法，首先进行checkOffsetAndCount()方法进行边界检查，然后，将data中指定的数据拷贝到数组中去。
> - 3.第三个of方法，同理，和第二个of方法差不多，只不过一个是byte[]，一个是ByteBuffer。

### 4、toString()

```kotlin
/**
   * Returns a human-readable string that describes the contents of this byte string. Typically this
   * is a string like {@code [text=Hello]} or {@code [hex=0000ffff]}.
   */
  @Override public String toString() {
    if (data.length == 0) {
      return "[size=0]";
    }

    String text = utf8();
    int i = codePointIndexToCharIndex(text, 64);

    if (i == -1) {
      return data.length <= 64
          ? "[hex=" + hex() + "]"
          : "[size=" + data.length + " hex=" + substring(0, 64).hex() + "…]";
    }

    String safeText = text.substring(0, i)
        .replace("\\", "\\\\")
        .replace("\n", "\\n")
        .replace("\r", "\\r");
    return i < text.length()
        ? "[size=" + data.length + " text=" + safeText + "…]"
        : "[text=" + safeText + "]";
  }
```

### 5、utf8()

utf8格式的String

```tsx
  /** Constructs a new {@code String} by decoding the bytes as {@code UTF-8}. */
  public String utf8() {
    String result = utf8;
    // We don't care if we double-allocate in racy code.
    return result != null ? result : (utf8 = new String(data, Util.UTF_8));
  }
```

这里面的一个判断语句，实现ByteString性能优化，看来优化这个东西还是很容易实现的。第一次创建UTF-8对象的方法是调用new String(data,Util.UTF_8),后面就不再调用该方法而是直接返回result；发现uft8是对String 的方法进一步封装。

下面是ByteString的方法



![img](https:////upload-images.jianshu.io/upload_images/5713484-2a117c8cb66f8450.png?imageMogr2/auto-orient/strip|imageView2/2/w/347/format/webp)

## 六、[Buffer](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FBuffer.java)

### 1、Buffer简介

> - 1、Buffer存储的是可变比特序列，需要注意的是Buffer内部对比特数据的存储不是直接使用一个byte数组那么简单，它使用了一种新的数据类型Segment进行存储。不过我们先不去管Segment是什么东西，可以先直接将Buffer想象成一个LinkedList集合就知道了，之所以做这样，因为Buffer的容量可以动态扩展，从序列的尾部存入数据，从序列的头部读取数据。其实Buffer的底层实现了远比LinkedList复杂的多，他使用双向链表的形式存储数组，链表节点的数据类型就是前面说的Segment，Segment存储有一个不可变比特序列，即final byte[] data。使用的Buffer的好处在于从一个Buffer移动到另一个Buffer的时候，实际上并没有进行拷贝，只是改变了它对应的Segment的所有者，其实这也采用链表存储数据的好处，这样的特点在多线程网络通信中带来很大的好处。最后使用Buffer还有另一个好处那就是它实现了BufferSource和BufferSink接口。
> - 2、前面讲到Buffer这个类实际上就是整个读写的核心，包括RealBufferSource和RealBufferedSink实际上都只是一个代理，里面操作全部都是通过Buffer来完成。

### 2、Buffer的属性及实现的接口

```java
public final class Buffer implements BufferedSource, BufferedSink, Cloneable {
  //明显是给16进制准备的
  private static final byte[] DIGITS =
      { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f' };
  static final int REPLACEMENT_CHARACTER = '\ufffd';
  //Buffer存储了一个这样的head节点，这就是Buffer对数据的存储结构。字节数组都是交给Segment进行管理
  Segment head;
  //当前存储的数据大小
  long size;
```

因为Buffer持有一个Segment的引用，所以通过这个引用能拿到整个链表中的所有数据。
 同时Buffer实现了三个接口，读，写以及clone。

**1、clone接口**

那咱们先说clone吧，这个比较简答，大家都能看懂，Java拷贝，主要分为深拷贝还是浅拷贝。Buffer这里采用的是浅拷贝。

```cpp
  //看到注释的第一句话，我就知道了是深拷贝，哈哈！
  /** Returns a deep copy of this buffer. */
  @Override public Buffer clone() {
    //先new了一个Buffer对象
    Buffer result = new Buffer();
    //如果size==0，说明这个Buffer是空的，所以直接返回即可
    if (size == 0) return result;
    //如果size!=0，说明这个Buffer是有数据的，然后吧head指向这个copy的head,PS大家回想下Segment的这个构造函数，里面是怎么操作的？
    result.head = new Segment(head);
    //然后设置copy的head的next和prev的值
    result.head.next = result.head.prev = result.head;
    //开始遍历这个Buffer持有的Segment链了
    for (Segment s = head.next; s != head; s = s.next) {
      result.head.prev.push(new Segment(s));
    }
    result.size = size;
    return result;
  }
```

> - 1对应了实现的clone方法，如果整个Buffer的size为0，也就是没有数据，那么返回一个新建的Buffer对象，如果不为空就是遍历所有segment并且都创建一个对应的segment，这样clone出来的对象就是一个全新的毫无关系的对象。
> - 2前面分析segment的时候有讲到是一个双向链表，但是segment自身构造的时候却没有形成闭环，其实就是在Buffer中产生的。
>    result.head.next = result.head.prev = result.head;
> - 3关于for循环有同学表示看不懂，我这里就详细解释下:
>    先取出head的next对象的引用，然后压入该节点，比如 目前顺序是 A B C，A是head，A的pre是C，所以在C节点的后面加入新的segmentD，压后的顺序就是 A B C D,然后执行s.next，如果s应是最后了，则跳出循环

**2、BufferedSource, BufferedSink接口**

Buffer实现了这两个接口的所有方法，既有读，也有写，由于篇幅的限制，我就不全部讲解，只挑几个讲解。其他的都大同小异

```kotlin
 @Override public int readInt() {
    //第一步
    if (size < 4) throw new IllegalStateException("size < 4: " + size);
    //第二步
    Segment segment = head;
    int pos = segment.pos;
    int limit = segment.limit;
    // If the int is split across multiple segments, delegate to readByte().
    if (limit - pos < 4) {
      return (readByte() & 0xff) << 24
          |  (readByte() & 0xff) << 16
          |  (readByte() & 0xff) <<  8
          |  (readByte() & 0xff);
    }
     //第三步
    byte[] data = segment.data;
    int i = (data[pos++] & 0xff) << 24
        |   (data[pos++] & 0xff) << 16
        |   (data[pos++] & 0xff) <<  8
        |   (data[pos++] & 0xff);
    size -= 4;
    //第四步
    if (pos == limit) {
      head = segment.pop();
      SegmentPool.recycle(segment);
    } else {
      segment.pos = pos;
    }
    //第五步
    return i;
  }
```

第一步，做了一次验证，因为一个int数据的字节数是4，所以必须保证当前Buffer的size大于4。
 第二步，如果当前的Segement所包含的字节数小于4,因此还需要去一个Segment中获取一分部数据，因此通过调用readByte()方法一个字节一个字节的读取，该方法我们后介绍
 第三步，如果当前Segment的数据够用，因此直接从pos位置开始读取4个字节的数据，然后将其转换为int数据，转换方法很简单就是位和或运算。
 第四步，如果pos==limit证明当前head对应的Segment没有可读数据，因此将该Segment从双向链表移除，并回收该Segment。如果还有数据则设置新的pos值。
 第五步，返回解析得到int值
 这里面提到了readByte()那么咱们就来研究下这个readByte()方法



```java
 @Override public byte readByte() {
     //第一步
    if (size == 0) throw new IllegalStateException("size == 0");
     //第二步
    Segment segment = head;
    int pos = segment.pos;
    int limit = segment.limit;
    //第三步
    byte[] data = segment.data;
    //第四步
    byte b = data[pos++];
    size -= 1;
    //第五步
    if (pos == limit) {
      head = segment.pop();
      SegmentPool.recycle(segment);
    } else {
      segment.pos = pos;
    }
    //第六步
    return b;
  }
```

第一步，做size判断，如果size==0表示没有什么东西好读取的。
 第二步，获取Segment，并取得这个segment的pos位置和limit的位置
 第三步，获取byte[] data
 第四步，获取一个字节的byte
 第五步，判断这个segment有没有可读数据了，如果没有可读数据则回收Segment。如果有数据则更新pos的值
 第六步，返回b

那咱们再来看看写的方法，那换一个就不写int，选写short吧



```java
  @Override public Buffer writeShort(int s) {
    //第一步
    Segment tail = writableSegment(2);
    //第二步
    byte[] data = tail.data;
    int limit = tail.limit;
    data[limit++] = (byte) ((s >>> 8) & 0xff);
    data[limit++] = (byte)  (s        & 0xff);
    tail.limit = limit;
    //第三步
    size += 2;
    //第四步
    return this;
  }

  /**
   * Returns a tail segment that we can write at least {@code minimumCapacity}
   * bytes to, creating it if necessary.
   */
  Segment writableSegment(int minimumCapacity) {
    if (minimumCapacity < 1 || minimumCapacity > Segment.SIZE) throw new IllegalArgumentException();
    //如果hea=null表明Buffer里面一个Segment都没有
    if (head == null) {
      //从SegmentPool取出一个Segment 
      head = SegmentPool.take(); // Acquire a first segment.
      return head.next = head.prev = head;
    }
    //如果head不为null.
    Segment tail = head.prev;
    //如果已经写入的数据+最小可以写入的空间超过限制，则在SegmentPool里面取一个
    if (tail.limit + minimumCapacity > Segment.SIZE || !tail.owner) {
      tail = tail.push(SegmentPool.take()); // Append a new empty segment to fill up.
    }
    return tail;
  }
```

writeShort用来给Buffer写入一个short数据
 第一步,通过writeableSegment拿到一个能有2个字节的空间的segment，writeableSegment方法我里面写了注释，大家自己去看，我这里就不详细说明了。
 第二步，tail中的data就是字节数组，limit则是数据的尾部索引，写数据就是在尾部继续写，直接设置在data通过limit自增后的index,然后重置尾部索引.
 第三步，size+2
 第四步，返回tail

## 七、Okio中的超时机制

### 1、[TimeOut](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FTimeout.java)

okio的超时机制让I/O操作不会因为异常阻塞在某个未知的错误上，okio的基础超时机制是采取同步超时。
 1以输出流Sink为例子，当我们用下面的方法包装流时：

```java
//Okio.java

 //实际上调用的两个参数的sink方法，第二个参数是new的TimeOut对象，即同步超时
public static Sink sink(OutputStream out) {
    return sink(out, new Timeout());
  }

  private static Sink sink(final OutputStream out, final Timeout timeout) {
    if (out == null) throw new IllegalArgumentException("out == null");
    if (timeout == null) throw new IllegalArgumentException("timeout == null");

    return new Sink() {
        @Override 
        public void write(Buffer source, long byteCount) throws IOException {
        checkOffsetAndCount(source.size, 0, byteCount);
        while (byteCount > 0) {
          timeout.throwIfReached();
          Segment head = source.head;
          int toCopy = (int) Math.min(byteCount, head.limit - head.pos);
          out.write(head.data, head.pos, toCopy);

          head.pos += toCopy;
          byteCount -= toCopy;
          source.size -= toCopy;

          if (head.pos == head.limit) {
            source.head = head.pop();
            SegmentPool.recycle(head);
          }
        }
      }

      @Override public void flush() throws IOException {
        out.flush();
      }

      @Override public void close() throws IOException {
        out.close();
      }

      @Override public Timeout timeout() {
        return timeout;
      }

      @Override public String toString() {
        return "sink(" + out + ")";
      }
    };
  }
```

可以看到write方法里中实际上有一个while循环，在每个开始写的时候就调用了timeout.throwIfReached()方法，我们推断这个方法里面做了时间是否超时的判断，如果超时了，应该throw一个exception出来。这很明显是一个同步超时机制，按序执行。同理Source也是一样，那么咱们看下他里面到底是怎么执行的



```java
  /**
   * Throws an {@link InterruptedIOException} if the deadline has been reached or if the current
   * thread has been interrupted. This method doesn't detect timeouts; that should be implemented to
   * asynchronously abort an in-progress operation.
   */
  public void throwIfReached() throws IOException {
    if (Thread.interrupted()) {
      throw new InterruptedIOException("thread interrupted");
    }

    if (hasDeadline && deadlineNanoTime - System.nanoTime() <= 0) {
      throw new InterruptedIOException("deadline reached");
    }
  }
```

大家先看下上面的注释，我英语不是很好，大家如果有更好的翻译，请提示我，我翻译如下：
 如果到达了最后的时间，会抛出InterruptedIOException，或者线程被interrupted了也会抛出InterruptedIOException。该方法是不会检查超时的，应该是是一个异步进度操作单元来实现这个类，进行检查超时。
 但是当我们看okio对于socket的封装时



```java
  /**
   * Returns a sink that writes to {@code socket}. Prefer this over {@link
   * #sink(OutputStream)} because this method honors timeouts. When the socket
   * write times out, the socket is asynchronously closed by a watchdog thread.
   */
  public static Sink sink(Socket socket) throws IOException {
    if (socket == null) throw new IllegalArgumentException("socket == null");
    AsyncTimeout timeout = timeout(socket);
    Sink sink = sink(socket.getOutputStream(), timeout);
    return timeout.sink(sink);
  }
```

### 2、[AsyncTimeout](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokio%2Fblob%2Fmaster%2Fokio%2Fsrc%2Fmain%2Fjava%2Fokio%2FAsyncTimeout.java)

这里出现一个AsyncTimeout类，这个实际上继承于TimeOut所实现的一个异步超时类，这个异步类比同步要复杂的多，它使用了人一个WatchDog线程在后台进行监听超时。
 这里面用到了Watchdog,Watchdog是AsyncTimeout的静态内部类，那么我们来下看Watchdog是个什么东西

```java
  private static final class Watchdog extends Thread {
    public Watchdog() {
      super("Okio Watchdog");
      setDaemon(true);
    }

    public void run() {
      while (true) {
        try {
          AsyncTimeout timedOut;
          synchronized (AsyncTimeout.class) {
            timedOut = awaitTimeout();

            // Didn't find a node to interrupt. Try again.
            if (timedOut == null) continue;

            // The queue is completely empty. Let this thread exit and let another watchdog thread
            // get created on the next call to scheduleTimeout().
            if (timedOut == head) {
              head = null;
              return;
            }
          }

          // Close the timed out node.
          timedOut.timedOut();
        } catch (InterruptedException ignored) {
        }
      }
    }
  }
```

这里的WatchDog并不是Linux中的那个，只是一个继承于Thread的一类，里面的run方法执行的就是超时的判断，之所以在socket写时采取异步超时，这完全是由socket自身的性质决定的，socket经常会阻塞自己，导致下面的事情执行不了。AsyncTimeout继承Timeout类，可以复写里面的timeout方法，这个方法会在watchdao的线程中调用，所以不能执行长时间的操作，否则就会引起其他的超时。
 下面详细分析AsyncTimeout这个类



```java
    //不要一次写超过64k的数据否则可能会在慢连接中导致超时
    private static final int TIMEOUT_WRITE_SIZE = 64 * 1024;
    private static AsyncTimeout head;
    private boolean inQueue;
    private AsyncTimeout next;
    private long timeoutAt;
```

首先就是一个最大的写值，定义为64K，刚好和一个Buffer大小一样。官方的解释是如果连续读写超过这个数字的字节，那么及其容易导致超时，所以为了限制这个操作，直接给出了一个能写的最大数。
 下面两个参数head和next，很明显表明这是一个单链表，timeoutAt则是超时时间。使用者在操作之前首先要调用enter()方法，这样相当于注册了这个超时监听，然后配对的实现exit()方法。这样exit()有一个返回值会表明超时是否出发，注意：这个timeout是异步的，可能会在exit()后才调用



```java
   public final void enter() {
    if (inQueue) throw new IllegalStateException("Unbalanced enter/exit");
    long timeoutNanos = timeoutNanos();
    boolean hasDeadline = hasDeadline();
    if (timeoutNanos == 0 && !hasDeadline) {
      return; // No timeout and no deadline? Don't bother with the queue.
    }
    inQueue = true;
    scheduleTimeout(this, timeoutNanos, hasDeadline);
  }
```

这里只是判断了inQueue的状态，然后设置inQueue的状态，真正的调用是在scheduleTimeout()里面，那我们进去看下



```java
private static synchronized void scheduleTimeout(
      AsyncTimeout node, long timeoutNanos, boolean hasDeadline) {
    // Start the watchdog thread and create the head node when the first timeout is scheduled.
    //head==null，表明之前没有，本次是第一次操作，开启Watchdog守护线程
    if (head == null) {
      head = new AsyncTimeout();
      new Watchdog().start();
    }

    long now = System.nanoTime();
    //如果有最长限制(hasDeadline我翻译为最长限制)，并且超时时长不为0
    if (timeoutNanos != 0 && hasDeadline) {
      // Compute the earliest event; either timeout or deadline. Because nanoTime can wrap around,
      // Math.min() is undefined for absolute values, but meaningful for relative ones.
      //对比最长限制和超时时长，选择最小的那个值
      node.timeoutAt = now + Math.min(timeoutNanos, node.deadlineNanoTime() - now);
    } else if (timeoutNanos != 0) {
      //如果没有最长限制，但是超时时长不为0，则使用超时时长
      node.timeoutAt = now + timeoutNanos;
    } else if (hasDeadline) {
      //如果有最长限制，但是超时时长为0，则使用最长限制
      node.timeoutAt = node.deadlineNanoTime();
    } else {
     //如果既没有最长限制，和超时时长，则抛异常
      throw new AssertionError();
    }

    // Insert the node in sorted order.
    long remainingNanos = node.remainingNanos(now);
    for (AsyncTimeout prev = head; true; prev = prev.next) {
      //如果下一个为null或者剩余时间比下一个短 就插入node
      if (prev.next == null || remainingNanos < prev.next.remainingNanos(now)) {
        node.next = prev.next;
        prev.next = node;
        if (prev == head) {
          AsyncTimeout.class.notify(); // Wake up the watchdog when inserting at the front.
        }
        break;
      }
    }
  }
```

上面可以看出这个链表实际上是按照剩余的超时时间来进行排序的，快到超时的节点排在表头，一次往后递增。我们以一个read的代码来砍整个超时的绑定过程。



```java
     @Override
      public long read(Buffer sink, long byteCount) throws IOException {
            boolean throwOnTimeout = false;
            enter();
             try {
                 long result = source.read(sink,byteCount);
                 throwOnTimeout = true;
                 return result;
             }catch (IOException e){
                 throw exit(e);
             }finally {
                 exit(throwOnTimeout);
             }
      }
```

首先调用enterf方法，然后去做读的操作，这里可以砍到不仅在catch上而且是在finally中也做了操作，这样一场和正常的情况都考虑到了，在exit中调用了真正的exit方法，exit中会判断这个异步超时对象是否在链表中



```java
    final void exit(boolean throwOnTimeout) throws IOException {
        boolean timeOut =  exit();
        if (timeOut && throwOnTimeout)
            throw newTimeoutException(null);
    }

    public final boolean exit(){
        if (!inQueue)
            return false;
        inQueue = false;
        return cancelScheduledTimeout(this);
    }
```

回到前面说的的WatchDog，内部的run方法是一个while(true)的一个死循环，由于在while(true)里面锁住了内部的awaitTimeout的操作，这个await正是判断是否超时的真正地方。



```java
static AsyncTimeout awaitTimeout() throws InterruptedException {
        //拿到下一个节点
        AsyncTimeout node = head.next;
        //如果queue为空，等待直到有node进队，或者触发IDLE_TIMEOUT_MILLS
        if (node == null) {
            long startNanos = System.nanoTime();
            AsyncTimeout.class.wait(IDLE_TIMEOUT_MILLS);
            return head.next == null && (System.nanoTime() - startNanos) >= IDLE_TIMEOUT_NANOS ? head : null;
        }
        long waitNanos = node.remainingNanos(System.nanoTime());
        //这个head依然还没有超时，继续等待
        if (waitNanos > 0) {
            long waitMills = waitNanos / 1000000L;
            waitNanos -= (waitMills * 1000000L);
            AsyncTimeout.class.wait(waitMills, (int) waitNanos);
            return null;
        }
        head.next = node.next;
        node.next = null;
        return node;
    }
```

这里就比较清晰了，主要就是通过这个remainNanos来判断预定的超时时间减去当前时间是否大于0，如果比0大就说明还没超时，于是wait剩余的时间，然后表示没有超时，如果小于0，就会把这个从链表中移除，根据前面的exit方法中的判断就能触发整个超时的方法。

所以Buffer的写操作，实际上就是不断增加Segment的一个过程，读操作，就是不断消耗Segment中的数据，如果数据读取完，则使用SegmentPool进行回收。Buffer更多的逻辑主要是跨Segment读取数据，需要把前一个Segment的前端拼接在一起，因此看起来代码相对很多，但是其实开销非常低

## 八、okio的优雅之处

### (一)、okio类

在说okio的设计模式之前，先说下okio这这个类，该类是一个大工厂，为我们创建出各种各样的Sink、Source 对象，提供了三种数据源InputStream/OutputStream、Socket、File，我们可以把本该对这个三类数据源的IO操作通过okio库来实现，更方便，更高效。

### (二) 图演示

1、okio的类图

![img](https:////upload-images.jianshu.io/upload_images/5713484-e2af4e60b71bf432.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png



2、okio读写流程图

![img](https:////upload-images.jianshu.io/upload_images/5713484-42bef72907339ff5.png?imageMogr2/auto-orient/strip|imageView2/2/w/368/format/webp)

okio操作图.png

### (三) okio高效方便之处

> - 1、它对数据进行了*分块处理(Segment)*，这样在大数据IO的时候可以以块为单位进行IO，这可以提高IO的吞吐率
> - 2、它对这些数据块使用*链表*来进行管理，这可以仅通过移动*指针*就进行数据的管理，而不用真正的处理数据，而且对扩容来说十分方便.
> - 3、闲置的块进行管理，通过一个块池(SegmentPool)的管理，避免系统GC和申请byte时的zero-fill。其他的还有一些小细节上的优化，比如如果你把一个UTF-8的String转化为ByteString，ByteString会保留一份对原来String的引用，这样当你下次需要decode这个String时，程序通过保留的引用直接返回对应的String，从而避免了转码过程。
> - 4、他为所有的Source、Sink提供了超时操作，这是在Java原生IO操作是没有的。
> - 5、okio它对数据的读写都进行了封装，调用者可以十分方便的进行各种值(Stringg,short,int,hex,utf-8,base64等)的转化。

## 九、[FileSystem](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2Finternal%2Fio%2FFileSystem.java)

```java
public interface FileSystem {
  /** The host machine's local file system. */
  FileSystem SYSTEM = new FileSystem() {
    @Override public Source source(File file) throws FileNotFoundException {
      return Okio.source(file);
    }

    @Override public Sink sink(File file) throws FileNotFoundException {
      try {
        return Okio.sink(file);
      } catch (FileNotFoundException e) {
        // Maybe the parent directory doesn't exist? Try creating it first.
        file.getParentFile().mkdirs();
        return Okio.sink(file);
      }
    }

    @Override public Sink appendingSink(File file) throws FileNotFoundException {
      try {
        return Okio.appendingSink(file);
      } catch (FileNotFoundException e) {
        // Maybe the parent directory doesn't exist? Try creating it first.
        file.getParentFile().mkdirs();
        return Okio.appendingSink(file);
      }
    }

    @Override public void delete(File file) throws IOException {
      // If delete() fails, make sure it's because the file didn't exist!
      if (!file.delete() && file.exists()) {
        throw new IOException("failed to delete " + file);
      }
    }

    @Override public boolean exists(File file) {
      return file.exists();
    }

    @Override public long size(File file) {
      return file.length();
    }

    @Override public void rename(File from, File to) throws IOException {
      delete(to);
      if (!from.renameTo(to)) {
        throw new IOException("failed to rename " + from + " to " + to);
      }
    }

    @Override public void deleteContents(File directory) throws IOException {
      File[] files = directory.listFiles();
      if (files == null) {
        throw new IOException("not a readable directory: " + directory);
      }
      for (File file : files) {
        if (file.isDirectory()) {
          deleteContents(file);
        }
        if (!file.delete()) {
          throw new IOException("failed to delete " + file);
        }
      }
    }
  };

  /** Reads from {@code file}. */
  Source source(File file) throws FileNotFoundException;

  /**
   * Writes to {@code file}, discarding any data already present. Creates parent directories if
   * necessary.
   */
  Sink sink(File file) throws FileNotFoundException;

  /**
   * Writes to {@code file}, appending if data is already present. Creates parent directories if
   * necessary.
   */
  Sink appendingSink(File file) throws FileNotFoundException;

  /** Deletes {@code file} if it exists. Throws if the file exists and cannot be deleted. */
  void delete(File file) throws IOException;

  /** Returns true if {@code file} exists on the file system. */
  boolean exists(File file);

  /** Returns the number of bytes stored in {@code file}, or 0 if it does not exist. */
  long size(File file);

  /** Renames {@code from} to {@code to}. Throws if the file cannot be renamed. */
  void rename(File from, File to) throws IOException;

  /**
   * Recursively delete the contents of {@code directory}. Throws an IOException if any file could
   * not be deleted, or if {@code dir} is not a readable directory.
   */
  void deleteContents(File directory) throws IOException;
}
```

看完这段代码大家就会知道，FileSystem是一个接口，里面有一个他的实现类SYSTEM.所以可以FileSystem看成okhttp中文件系统对okio的桥接管理类。
 看下他的所有方法：

![img](https:////upload-images.jianshu.io/upload_images/5713484-66f1335d03c4eec5.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

> - 1、Source source(File): 获取File的source (用于读)
> - 2、Sinke  sink(File)：获取File的Sink (用于写)
> - 3、Sink appending(File): 获取File的Sink,拼接用的(用于写)
> - 4、delete(File)： void 删除文件
> - 5、exists(File)：文件是否存在
> - 6、size(Flie)：获取文件的大小
> - 7、rename(File，File): 文件改名
> - 8、deleteContents(File)：删除文件夹



# OKHttp源码解析(六)--中阶之缓存基础

## 一、Cache缓存的简介

缓存，顾名思义，也就是方便用户快速获取值的一种存储方式。小到CPU同频的昂贵的缓存颗粒，内缓存，硬盘，网络，CDN反缓存，DNS递归查询，OS页面置换，Redis数据库，都可以看作缓存。它有如下的特点：

> 

- 1.缓存载体与持久载体总是相对的，体量远远小于持久载体，成本高于之久体量，但是速度却高速持久体量。
- 2.需要实现排序依据，比如在java中可以使用Comparable<T>作为排序的接口
- 3.需要一种页面置换算法(page replacement algorithm)将旧页面去掉换成新页面，如最久未使用算法(LRU)、先进先出算法(FIFO)、最紧最小使用算法(LFU)、非最紧使用算法(NMRU)等
- 4.可溯源，如果没有命中缓存，就需要从原始地址获取，这个步骤叫做"回源头"，CDN厂商会标注"回源率"作为卖点

PS:在OKHTTP中，使用FileSystem作为缓存载体(磁盘相对于网络缓存)，使用LRU作为页面置换算法(封装了LinkedHashMap)。

HTTP作为客户端与服务器沟通的重要协议，对从事android开发的同学来说是一个非常重要的环节，其中网络层优化又是重中之重。今天主要是讲解OKHTTP中的缓存处理，那么首先先简单介绍下为什么要用缓存

## 二、为什么要用缓存



```undefined
  缓存对移动端非常重要，使用缓存可以提高用户体验，用缓存的主要在于：
```

> 

- 1 减少请求次数，较少服务器压力

- 2本地数据读取更快，让页面不会空白几百毫秒

- 3在无网络的情况下提供数据
   HTTP缓存是最好的减少客户端服务器往返次数的方案，缓存提供了一种机制来保证客户端或者代理能够存储一些东西，而这些东西将会在稍后的HTTP响应中用到的。(即第一次请求了，到了客户端，缓存起来，下次如果请求还需要一些资源，就不用到服务器去取了)这样，就不用让一些资源再次跨越整个网络了。

  ![img](https:////upload-images.jianshu.io/upload_images/5713484-0807ade7cc14a335.png?imageMogr2/auto-orient/strip|imageView2/2/w/403/format/webp)

  请求与缓存.png

## 三、HTTP缓存机制

### 1、HTTP报文

HTTP报文就是客户端和服务器之间通信时发送及其响应的数据块。客户端向服务器请求数据，发送请求(request)报文；服务器向客户端下发返回数据，返回响应(response)报文，报文信息主要分为两部分。

> 

- 1包含属性的头部(header)-------------附加信息(cookie，缓存信息等)，与缓存相关的规则信息，均包含在header中
- 2包含数据的主体部分(body)--------------HTTP请求真正想要传输的部分

### 2、缓存分类

**(1)按照"端“”分类**

缓存可以分为

- 1、服务器缓存，其中服务器缓存又可以分为服务器缓存和反向代理服务器缓存(也叫网关缓存，比如Nginx反向代理，Squid等)，其实广泛使用的CSN也是一种服务端缓存，目的都是让用户的请求走"捷径"，并且都是缓存图片、文件等静态资源。
- 2、客户端缓存
   客户端缓存则一般是只浏览器缓存，目的就是加速各种静态资源的访问，想想淘宝，京东，百度随便一个网页都是上百请求，每天PV都是上亿的，如果没有缓存，用户体验会急剧下降，同时服务器压力巨大。

**(2) 按照"是否想服务器发起请求，进行对比"分类**

> 可以分为:

- 1 强制缓存(不对比缓存)
- 2 对比缓存

已存在缓存数据时,仅基于强制缓存，请求数据流程如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-814445ae42439999.png?imageMogr2/auto-orient/strip|imageView2/2/w/865/format/webp)

强制缓存.png

已存在缓存数据时，仅基于对比缓存，请求数据的流程如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-80c95e1c181d3c63.png?imageMogr2/auto-orient/strip|imageView2/2/w/947/format/webp)

对比缓存.png

我们可以看到两类缓存规则的不同，强制缓存如果生效，则不再和服务器交互了，而对比缓存不惯是否生效，都需要和服务器发生交互。
 通过上面了解到，在缓存数据未失效的情况下，可以直接使用缓存数据，那么客户端是怎么判断数据是否失效的？同理，什么时候采用强制缓存，而什么时候又采用对比缓存，这里面客户端是怎么和服务器进行交互的？上面也说道，缓存规则是包含在响应header里面的。莫非所有的交互在header里面？

### 3、请求头header中有关缓存的设置

#### 3.1 expires

在HTTP/1.0中expires的值围服务器端返回的到期时间，即下一次请求时，请求时间小于服务器返回的到期时间，直接使用缓存数据，这里面有个问题，由于到期时间是服务器生成的，但是客户端的时间可能和服务器有误差，所以这就会导致误差，所以到了HTTP1.1基本上不适用expires了，使用Cache-Control替代了expires

#### 3.2 Cache-Control

Cache-Control 是最重要的规则。常见的取值有private、public
 、no-cache、max-age、no-store、默认是private。

响应头部                                                             意义
 Cache-Control：public                                       响应被公有缓存，移动端无用

| 响应头部                  |            意义            |
| ------------------------- | :------------------------: |
| Cache-Control：public     | 响应被共有缓存，移动端无用 |
| Cache-Control：private    | 响应被私有缓存，移动端无用 |
| Cache-Control：no-cache   |           不缓存           |
| Cache-Control：no-store   |           不缓存           |
| Cache-Control：max-age=60 |      60秒之后缓存过期      |

(PS：在浏览器里面，private 表示客户端可以缓存，public表示客户端和服务器都可以缓存)
 举个例子。入下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-226007a643454d4f.png?imageMogr2/auto-orient/strip|imageView2/2/w/393/format/webp)

缓存header.png



图中Cache-Control仅指定了max-age所以默认是private。缓存时间是31536000，也就是说365内的再次请求这条数据，都会直接获取缓存数据库中的数据，直接使用。

#### 3.3 Last-Modified/If-Modified-Since

上面提到了对比缓存，顾名思义，需要进行比较判断是否可以使用缓存，客户端第一次发起请求时，服务器会将缓存标志和数据一起返回给客户端，客户端当二者缓存至缓存数据库中。再次其你去数据时，客户端将备份的缓存标志发送给服务器，服务器根据标志来进行判断，判断成功后，返回304状态码，通知客户端比较成功，可以使用缓存数据。
 上面说到了对比缓存的流程，那么具体又是怎么实现的那？

**Last-Modified**

是通过Last-Modified/If-Modified-Since来实现的，服务器在响应请求时，告诉浏览器资源的最后修改时间。

![img](https:////upload-images.jianshu.io/upload_images/5713484-928fd3696949a556.png?imageMogr2/auto-orient/strip|imageView2/2/w/591/format/webp)

Last-Modified.png

**If-Modified-Since**

再次请求服务器时，通过此字段通知服务器上次请求时，服务器返回最远的最后修改时间。服务器收到请求后发现有If-Modified-Since则与被请求资源的最后修改时间进行对比。若资源的最后修改时间大于If-Modified-Since，说明资源又被改动过，则响应整个内容，返回状态码是200.如果资源的最后修改时间小于或者等于If-Modified-Since，说明资源没有修改，则响应状态码为304，告诉客户端继续使用cache.

![img](https:////upload-images.jianshu.io/upload_images/5713484-dab332d139587fd6.png?imageMogr2/auto-orient/strip|imageView2/2/w/597/format/webp)

If-Modified-Since.png

#### 3.4 ETag/If-None-Match(优先级高于Last-Modified/If-Modified-Since)

Etag:
 服务响应请求时，告诉客户端当前资源在服务器的唯一标识(生成规则由服务器决定)



![img](https:////upload-images.jianshu.io/upload_images/5713484-3a3096ae0a2eb725.png?imageMogr2/auto-orient/strip|imageView2/2/w/592/format/webp)

If-None-Match:
 再次请求服务器时，通过此字段通知服务器客户端缓存数据的唯一标识。服务器收到请求后发现有头部If-None-Match则与被请求的资源的唯一标识进行对比，不同则说明资源被改过，则响应整个内容，返回状态码是200，相同则说明资源没有被改动过，则响应状态码304，告知客户端可以使用缓存



![img](https:////upload-images.jianshu.io/upload_images/5713484-886273a315e32e3a.png?imageMogr2/auto-orient/strip|imageView2/2/w/592/format/webp)

正式使用时按需求也许只包含其中部分字段，客户端要根据这些信息存储这次请求信息，然后在客户端发起的时间内检查缓存，遵循下面的步骤

![img](https:////upload-images.jianshu.io/upload_images/5713484-b59ad898cbd27a14.png?imageMogr2/auto-orient/strip|imageView2/2/w/554/format/webp)

## 四、Cache-Control类详解

[CacheControl](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/CacheControl.java) 对应HTTP里面的CacheControl

```java
public final class CacheControl {

  private final boolean noCache;
  private final boolean noStore;
  private final int maxAgeSeconds;
  private final int sMaxAgeSeconds;
  private final boolean isPrivate;
  private final boolean isPublic;
  private final boolean mustRevalidate;
  private final int maxStaleSeconds;
  private final int minFreshSeconds;
  private final boolean onlyIfCached;
  private final boolean noTransform;

  /**
   * Cache control request directives that require network validation of responses. Note that such
   * requests may be assisted by the cache via conditional GET requests.
   */
  public static final CacheControl FORCE_NETWORK = new Builder().noCache().build();

  /**
   * Cache control request directives that uses the cache only, even if the cached response is
   * stale. If the response isn't available in the cache or requires server validation, the call
   * will fail with a {@code 504 Unsatisfiable Request}.
   */
  public static final CacheControl FORCE_CACHE = new Builder()
      .onlyIfCached()
      .maxStale(Integer.MAX_VALUE, TimeUnit.SECONDS)
      .build();
}
```

CacheControl类是对HTTP的Cache-Control头部的描述。CacheControl没有公共的构造方法，内部通过一个Build进行设置值，获取值可以通过CacheControl对象进行获取。
 Builder具体有如下设置方法：

> 

- 1、noCache()
   对应于“no-cache”，如果出现在 *响应* 的头部，不是表示不允许对响应进行缓存，而是表示客户端需要与服务器进行再次验证，进行一个额外的GET请求得到最新的响应；如果出现请求头部，则表示不适用缓存响应，即记性网络请求获取响应。
- 2、noStore()
   对应于"no-store"，如果出现在响应头部，则表明该响应不能被缓存
- 3、maxAge(int maxAge,TimeUnit timeUnit)
   对应"max-age"，设置缓存响应的最大存货时间。如果缓存响满足了到了最大存活时间，那么将不会再进行网络请求
- 4、maxStale(int maxStale,TimeUnit timeUnit)
   对应“max-stale”，缓存响应可以接受的最大过期时间，如果没有指定该参数，那么过期缓存响应将不会被使用
- 5、minFresh(int minFresh,TimeUnit timeUnit)
   对应"min-fresh"，设置一个响应将会持续刷新最小秒数，如果一个响应当minFresh过去后过期了，那么缓存响应不能被使用，需要重新进行网络请求
- 6、onlyIfCached()
   对应“onlyIfCached”，用于请求头部，表明该请求只接受缓存中的响应。如果缓存中没有响应，那么返回一个状态码为504的响应。

CacheControl类中还有其他方法，这里就不一一介绍了。想了解的可以去API文档查看。
 对于常用的缓存控制，CacheControl中提供了两个常用的于修饰请求，FORCE_CACHE表示只使用缓存中的响应，哪怕这个缓存过期了，FORCE_NETWORK这个表示只能使用网络响应

## 五、CacheStrategy类详解

[CacheStrategy](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/db9c2db40b0b89a1853715fd52e2748463d9cc9c/okhttp/src/main/java/okhttp3/internal/http/CacheStrategy.java#L31-L31) 缓存策略类
 OKHTTP使用了CacheStrategy实现了上面的流程图，它根据之前缓存的结果与当前将要发送Request的header进行策略，并得出是否进行请求的结果。

### (一)、策略原理

根据输出的networkRequest和cacheResponse的值是否为null给出不同的策略，如下：

| networkRequest | cacheResponse |                         result 结果                          |
| -------------- | :-----------: | :----------------------------------------------------------: |
| null           |     null      | only-if-cached (表明不进行网络请求，且缓存不存在或者过期，一定会返回503错误) |
| null           |   non-null    |           不进行网络请求，直接返回缓存，不请求网络           |
| non-null       |     null      |    需要进行网络请求，而且缓存不存在或者过去，直接访问网络    |
| non-null       |   non-null    | Header中包含ETag/Last-Modified标签，需要在满足条件下请求，还是需要访问网络 |

以上是对networkRequest/cacheResponse进行的switch查询得出的，下面我们就详细讲解下

### (二)、CacheStrategy类的构造

CacheStrategy使用Factory模式进行构造，通过Factory的get()方法获取CacheStrategy的对象，参数如下：



```csharp
    public Factory(long nowMillis, Request request, Response cacheResponse) {
      this.nowMillis = nowMillis;
      this.request = request;
      this.cacheResponse = cacheResponse;

      if (cacheResponse != null) {
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
        Headers headers = cacheResponse.headers();
        //获取cacheReposne中的header中值
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
            ageSeconds = HttpHeaders.parseSeconds(value, -1);
          }
        }
      }
    }

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
    /**
     * Returns a strategy to satisfy {@code request} using the a cached response {@code response}.
     */
    public CacheStrategy get() {
      //获取当前的缓存策略
      CacheStrategy candidate = getCandidate();
     //如果是网络请求不为null并且请求里面的cacheControl是只用缓存
      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // We're forbidden from using the network and the cache is insufficient.
        //使用只用缓存的策略
        return new CacheStrategy(null, null);
      }
      return candidate;
    }

    /** Returns a strategy to use assuming the request can use the network. */
    private CacheStrategy getCandidate() {
      // No cached response.
      //如果没有缓存响应，返回一个没有响应的策略
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }
       //如果是https，丢失了握手，返回一个没有响应的策略
      // Drop the cached response if it's missing a required handshake.
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }
     
      // 响应不能被缓存
      // If this response shouldn't have been stored, it should never be used
      // as a response source. This check should be redundant as long as the
      // persistence store is well-behaved and the rules are constant.
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }
     
      //获取请求头里面的CacheControl
      CacheControl requestCaching = request.cacheControl();
      //如果请求里面设置了不缓存，则不缓存
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }
      //获取响应的年龄
      long ageMillis = cacheResponseAge();
      //获取上次响应刷新的时间
      long freshMillis = computeFreshnessLifetime();
      //如果请求里面有最大持久时间要求，则两者选择最短时间的要求
      if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }

      long minFreshMillis = 0;
      //如果请求里面有最小刷新时间的限制
      if (requestCaching.minFreshSeconds() != -1) {
         //用请求中的最小更新时间来更新最小时间限制
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }
      //最大验证时间
      long maxStaleMillis = 0;
      //响应缓存控制器
      CacheControl responseCaching = cacheResponse.cacheControl();
      //如果响应(服务器)那边不是必须验证并且存在最大验证秒数
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        //更新最大验证时间
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }
     //响应支持缓存
       //持续时间+最短刷新时间<上次刷新时间+最大验证时间 则可以缓存
      //现在时间(now)-已经过去的时间（sent）+可以存活的时间<最大存活时间(max-age)
      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
       //缓存响应
        return new CacheStrategy(null, builder.build());
      }
    
      //如果想缓存request，必须要满足一定的条件
      // Find a condition to add to the request. If the condition is satisfied, the response body
      // will not be transmitted.
      String conditionName;
      String conditionValue;
      if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
      } else if (lastModified != null) {
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
      } else if (servedDate != null) {
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
      } else {
        //没有条件则返回一个定期的request
        return new CacheStrategy(request, null); // No condition! Make a regular request.
      }
      
      Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
      Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

      Request conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build();
      //返回有条件的缓存request策略
      return new CacheStrategy(conditionalRequest, cacheResponse);
    }
```

通过上面分析，我们可以发现，OKHTTP实现的缓存策略实质上就是大量的if/else判断，这些其实都是和RFC标准文档里面写死的。
 上面说了这么多，那么咱们要开始今天的主题了----CacheInterceptor类

## 六、 CacheInterceptor 类详解

[BridgeInterceptor](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/cache/CacheInterceptor.java) :负责将*请求*和*返回* 关联的保存到缓存中。客户端和服务器根据一定的机制(策略CacheStrategy )，在需要的时候使用缓存的数据作为网络响应，节省了时间和宽带。

老规矩上源码：



```kotlin
  //CacheInterceptor.java
 @Override 
 public Response intercept(Chain chain) throws IOException {
    //如果存在缓存，则从缓存中取出，有可能为null
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //获取缓存策略对象
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    //策略中的请求
    Request networkRequest = strategy.networkRequest;
     //策略中的响应
    Response cacheResponse = strategy.cacheResponse;
     //缓存非空判断，
    if (cache != null) {
      cache.trackResponse(strategy);
    }
    //缓存策略不为null并且缓存响应是null
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }
     //禁止使用网络(根据缓存策略)，缓存又无效，直接返回
    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
     //缓存有效，不使用网络
    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    //缓存无效，执行下一个拦截器
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
     //本地有缓存，根据条件选择使用哪个响应
    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
     //使用网络响应
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
       //缓存到本地
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

简单的说下上述流程:
 1、如果配置缓存，则从缓存中取一次，不保证存在
 2、缓存策略
 3、缓存监测
 4、禁止使用网络（根据缓存策略），缓存又无效，直接返回
 5、缓存有效，不使用网络
 6、缓存无效，执行下一个拦截器
 7、本地有缓存，根具条件选择使用哪个响应
 8、使用网络响应
 9、 缓存到本地
 大体流程分析完，那么咱们再详细分析下。
 首先说到了缓存就不得不提下OKHttp里面的Cache.java类和InternalCache.java那么咱们就简单的聊下这两个类

### (一)、Cache.java类

[Cache](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/Cache.java)

#### 1、基本特征



```java
private final DiskLruCache cache; 

public Cache(File directory, long maxSize) {  
    this(directory, maxSize, FileSystem.SYSTEM);  
  }  
Cache(File directory, long maxSize, FileSystem fileSystem) {  
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);  
}  
```

通过上面代码可知

> 

- 1、Cache对象拥有一个DiskLruCache引用。
- 2、Cache构造器接受两个参数，意味着如果我们想要创建一个缓存必须指定缓存文件存储的目录和缓存文件的最大值

#### 2、既然是Cache，那么一定会有"增"、"删"、"改"、"查"。

**(1) ”增“操作——put()方法**

```kotlin
CacheRequest put(Response response) {
    String requestMethod = response.request().method();
    //判断请求如果是"POST"、"PATCH"、"PUT"、"DELETE"、"MOVE"中的任何一个则调用DiskLruCache.remove(urlToKey(request));将这个请求从缓存中移除出去。
    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    //判断请求如果不是Get则不进行缓存，直接返回null。官方给的解释是缓存get方法得到的Response效率高，其它方法的Response没有缓存效率低。通常通过get方法获取到的数据都是固定不变的的，因此缓存效率自然就高了。其它方法会根据请求报文参数的不同得到不同的Response，因此缓存效率自然而然就低了。
    if (!requestMethod.equals("GET")) {
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }
     //判断请求中的http数据包中headers是否有符号"*"的通配符，有则不缓存直接返回null
    if (HttpHeaders.hasVaryAll(response)) {
      return null;
    }
    //由Response对象构建一个Entry对象,Entry是Cache的一个内部类
    Entry entry = new Entry(response);
    //通过调用DiskLruCache.edit();方法得到一个DiskLruCache.Editor对象。
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      //把这个entry写入
      //方法内部是通过Okio.buffer(editor.newSink(ENTRY_METADATA));获取到一个BufferedSink对象，随后将Entry中存储的Http报头数据写入到sink流中。
      entry.writeTo(editor);
      //构建一个CacheRequestImpl对象，构造器中通过editor.newSink(ENTRY_BODY)方法获得Sink对象
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
  }
```

总结一下上述步骤
 第一步，先判断是不是一个正常的请求(get,post等)
 第二步，由于只支持get请求，非get请求直接返回
 第三步，通配符过滤
 第四步，通过上述检验后开始真正的缓存流程，new一个Entry
 第五步，获取一个DiskLruCache.Editor对象
 第六步，通过DiskLruCache.Edito写入数据
 第七步，返回数据
 PS：关于key()方法在remove里面详解

上面使用到了remove方法，莫非就是"删"的操作，那咱们来看下

**(2) ”删“操作——remove()方法**

```csharp
  void remove(Request request) throws IOException {
    cache.remove(key(request.url()));
  }
  public static String key(HttpUrl url) {
    return ByteString.encodeUtf8(url.toString()).md5().hex();
  }
```

果然remove就是传说中的"删除"操作，
 key()这个方法原来就说获取url的MD5和hex生成的key

**(3) ”改“操作——update()方法**

```csharp
void update(Response cached, Response network) {
    //用response构造一个Entry对象
    Entry entry = new Entry(network);
    //从命中缓存中获取到的DiskLruCache.Snapshot
    DiskLruCache.Snapshot snapshot = ((CacheResponseBody) cached.body()).snapshot;
    //从DiskLruCache.Snapshot获取DiskLruCache.Editor()对象
    DiskLruCache.Editor editor = null;
    try {
      editor = snapshot.edit(); // Returns null if snapshot is not current.
      if (editor != null) {
        //将entry写入editor中
        entry.writeTo(editor);
        editor.commit();
      }
    } catch (IOException e) {
      abortQuietly(editor);
    }
  }
```

根据上述代码大体流程如下：
 第一步，首先要获取entry对象
 第二步，获取DiskLruCache.Editor对象
 第三步，写入entry对象

**(4) ”查“操作——get()方法**

```kotlin
 Response get(Request request) {
    //获取url经过MD5和HEX的key
    String key = key(request.url());
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
     //根据key来获取一个snapshot，由此可知我们的key-value里面的value对应的是snapshot
      snapshot = cache.get(key);
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
      // Give up because the cache cannot be read.
      return null;
    }
    //利用前面的Snapshot创建一个Entry对象。存储的内容是响应的Http数据包Header部分的数据。snapshot.getSource得到的是一个Source对象 (source是okio里面的一个接口)
    try {
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }

    //利用entry和snapshot得到Response对象，该方法内部会利用前面的Entry和Snapshot得到响应的Http数据包Body（body的获取方式通过snapshot.getSource(ENTRY_BODY)得到）创建一个CacheResponseBody对象；再利用该CacheResponseBody对象和第三步得到的Entry对象构建一个Response的对象，这样该对象就包含了一个网络响应的全部数据了。
    Response response = entry.response(snapshot);
    //对request和Response进行比配检查，成功则返回该Response。匹配方法就是url.equals(request.url().toString()) && requestMethod.equals(request.method()) && OkHeaders.varyMatches(response, varyHeaders, request);其中Entry.url和Entry.requestMethod两个值在构建的时候就被初始化好了，初始化值从命中的缓存中获取。因此该匹配方法就是将缓存的请求url和请求方法跟新的客户请求进行对比。最后OkHeaders.varyMatches(response, varyHeaders, request)是检查命中的缓存Http报头跟新的客户请求的Http报头中的键值对是否一样。如果全部结果为真，则返回命中的Response。
    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }

    return response;
  }
```

总结上面流程大体是：
 第一步 获取key
 第一步 获取DiskLruCache.Snapshot对象
 第三步 获取Entry对象
 第四步 获取response
 第五步 检查是response

通过对上述增删改查的分析，我们可以得出如下结论

| 方法                                 |               返回值                |
| ------------------------------------ | :---------------------------------: |
| DiskLruCache.get(String)             |    可以获取DiskLruCache.Snapshot    |
| DiskLruCache.remove(String)          |            可以移除请求             |
| DiskLruCache.edit(String)            | 可以获得一个DiskLruCache.Editor对象 |
| DiskLruCache.Editor.newSink(int)     |         可以获得一个sink流          |
| DiskLruCache.Snapshot.getSource(int) |      可以获取一个Source对象。       |
| DiskLruCache.Snapshot.edit()         | 可以获得一个DiskLruCache.Editor对象 |

DiskLruCache是OKHTTP的缓存的精髓，由于篇幅限制，在下一章讲解



# OKHttp源码解析(七)--中阶之缓存机制

上一章主要讲解了HTTP中的缓存以及OKHTTP中的缓存，今天我们主要讲解OKHTTP中缓存体系的精髓---DiskLruCache，由于篇幅限制，今天内容看似不多，大概分为两个部分
 1.DiskLruCache内部类详解
 2.DiskLruCache类详解
 3.OKHTTP的缓存的实现---CacheInterceptor的具体执行流程

## 一、[DiskLruCache](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/cache/DiskLruCache.java)

在看DiskLruCache前先看下他的几个内部类

### 1、Entry.class(DiskLruCache的内部类)

Entry内部类是实际用于存储的缓存数据的实体类，每一个url对应一个Entry实体

```dart
 private final class Entry {
    final String key;
    /** 实体对应的缓存文件 */ 
    /** Lengths of this entry's files. */
    final long[] lengths; //文件比特数 
    final File[] cleanFiles;
    final File[] dirtyFiles;
    /** 实体是否可读，可读为true，不可读为false*/  
    /** True if this entry has ever been published. */
    boolean readable;

     /** 编辑器，如果实体没有被编辑过，则为null*/  
    /** The ongoing edit or null if this entry is not being edited. */
    Editor currentEditor;
    /** 最近提交的Entry的序列号 */  
    /** The sequence number of the most recently committed edit to this entry. */
    long sequenceNumber;
    //构造器 就一个入参 key，而key又是url，所以，一个url对应一个Entry
    Entry(String key) {
     
      this.key = key;
      //valueCount在构造DiskLruCache时传入的参数默认大小为2
      //具体请看Cache类的构造函数，里面通过DiskLruCache.create()方法创建了DiskLruCache，并且传入一个值为2的ENTRY_COUNT常量
      lengths = new long[valueCount];
      cleanFiles = new File[valueCount];
      dirtyFiles = new File[valueCount];

      // The names are repetitive so re-use the same builder to avoid allocations.
      StringBuilder fileBuilder = new StringBuilder(key).append('.');
      int truncateTo = fileBuilder.length();
      //由于valueCount为2,所以循环了2次，一共创建了4份文件
      //分别为key.1文件和key.1.tmp文件
      //           key.2文件和key.2.tmp文件
      for (int i = 0; i < valueCount; i++) {
        fileBuilder.append(i);
        cleanFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.append(".tmp");
        dirtyFiles[i] = new File(directory, fileBuilder.toString());
        fileBuilder.setLength(truncateTo);
      }
    }
```

通过上述代码咱们知道了，一个url对应一个Entry对象，同时，每个Entry对应两个文件，key.1存储的是Response的headers，key.2文件存储的是Response的body

### 2、Snapshot (DiskLruCache的内部类)



```java
  /** A snapshot of the values for an entry. */
  public final class Snapshot implements Closeable {
    private final String key;  //也有一个key
    private final long sequenceNumber; //序列号
    private final Source[] sources; //可以读入数据的流   这么多的流主要是从cleanFile中读取数据
    private final long[] lengths; //与上面的流一一对应  

    //构造器就是对上面这些属性进行赋值
    Snapshot(String key, long sequenceNumber, Source[] sources, long[] lengths) {
      this.key = key;
      this.sequenceNumber = sequenceNumber;
      this.sources = sources;
      this.lengths = lengths;
    }

    public String key() {
      return key;
    }
   //edit方法主要就是调用DiskLruCache的edit方法了，入参是该Snapshot对象的两个属性key和sequenceNumber.
    /**
     * Returns an editor for this snapshot's entry, or null if either the entry has changed since
     * this snapshot was created or if another edit is in progress.
     */
    public Editor edit() throws IOException {
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

这时候再回来看下Entry里面的snapshot()方法



```csharp
    /**
     * Returns a snapshot of this entry. This opens all streams eagerly to guarantee that we see a
     * single published snapshot. If we opened streams lazily then the streams could come from
     * different edits.
     */
    Snapshot snapshot() {
      //首先判断 线程是否有DiskLruCache对象的锁
      if (!Thread.holdsLock(DiskLruCache.this)) throw new AssertionError();
      //new了一个Souce类型数组，容量为2
      Source[] sources = new Source[valueCount];
      //clone一个long类型的数组，容量为2
      long[] lengths = this.lengths.clone(); // Defensive copy since these can be zeroed out.
       //获取cleanFile的Source，用于读取cleanFile中的数据，并用得到的souce、Entry.key、Entry.length、sequenceNumber数据构造一个Snapshot对象
      try {
        for (int i = 0; i < valueCount; i++) {
          sources[i] = fileSystem.source(cleanFiles[i]);
        }
        return new Snapshot(key, sequenceNumber, sources, lengths);
      } catch (FileNotFoundException e) {
        // A file must have been deleted manually!
        for (int i = 0; i < valueCount; i++) {
          if (sources[i] != null) {
            Util.closeQuietly(sources[i]);
          } else {
            break;
          }
        }
        // Since the entry is no longer valid, remove it so the metadata is accurate (i.e. the cache
        // size.)
        try {
          removeEntry(this);
        } catch (IOException ignored) {
        }
        return null;
      }
    }
```

由上面代码可知Spapshot里面的key，sequenceNumber，sources，lenths都是一个entry,其实也就可以说一个Entry对象一一对应一个Snapshot对象

### 3、Editor.class(DiskLruCache的内部类)

Editro类的属性和构造器貌似看不到什么东西，不过通过构造器，我们知道，在构造一个Editor的时候必须传入一个Entry，莫非Editor是对这个Entry操作类。



```java
/** Edits the values for an entry. */
  public final class Editor {
    final Entry entry;
    final boolean[] written;
    private boolean done;

    Editor(Entry entry) {
      this.entry = entry;
      this.written = (entry.readable) ? null : new boolean[valueCount];
    }

    /**
     * Prevents this editor from completing normally. This is necessary either when the edit causes
     * an I/O error, or if the target entry is evicted while this editor is active. In either case
     * we delete the editor's created files and prevent new files from being created. Note that once
     * an editor has been detached it is possible for another editor to edit the entry.
     *这里说一下detach方法，当编辑器(Editor)处于io操作的error的时候，或者editor正在被调用的时候而被清
     *除的，为了防止编辑器可以正常的完成。我们需要删除编辑器创建的文件，并防止创建新的文件。如果编
     *辑器被分离，其他的编辑器可以编辑这个Entry
     */
    void detach() {
      if (entry.currentEditor == this) {
        for (int i = 0; i < valueCount; i++) {
          try {
            fileSystem.delete(entry.dirtyFiles[i]);
          } catch (IOException e) {
            // This file is potentially leaked. Not much we can do about that.
          }
        }
        entry.currentEditor = null;
      }
    }

    /**
     * Returns an unbuffered input stream to read the last committed value, or null if no value has
     * been committed.
     * 获取cleanFile的输入流 在commit的时候把done设为true
     */
    public Source newSource(int index) {
      synchronized (DiskLruCache.this) {
       //如果已经commit了，不能读取了
        if (done) {
          throw new IllegalStateException();
        }
        //如果entry不可读，并且已经有编辑器了(其实就是dirty)
        if (!entry.readable || entry.currentEditor != this) {
          return null;
        }
        try {
         //通过filesystem获取cleanFile的输入流
          return fileSystem.source(entry.cleanFiles[index]);
        } catch (FileNotFoundException e) {
          return null;
        }
      }
    }

    /**
     * Returns a new unbuffered output stream to write the value at {@code index}. If the underlying
     * output stream encounters errors when writing to the filesystem, this edit will be aborted
     * when {@link #commit} is called. The returned output stream does not throw IOExceptions.
    * 获取dirty文件的输出流，如果在写入数据的时候出现错误，会立即停止。返回的输出流不会抛IO异常
     */
    public Sink newSink(int index) {
      synchronized (DiskLruCache.this) {
       //已经提交，不能操作
        if (done) {
          throw new IllegalStateException();
        }
       //如果编辑器是不自己的，不能操作
        if (entry.currentEditor != this) {
          return Okio.blackhole();
        }
       //如果entry不可读，把对应的written设为true
        if (!entry.readable) {
          written[index] = true;
        }
         //如果文件
        File dirtyFile = entry.dirtyFiles[index];
        Sink sink;
        try {
          //如果fileSystem获取文件的输出流
          sink = fileSystem.sink(dirtyFile);
        } catch (FileNotFoundException e) {
          return Okio.blackhole();
        }
        return new FaultHidingSink(sink) {
          @Override protected void onException(IOException e) {
            synchronized (DiskLruCache.this) {
              detach();
            }
          }
        };
      }
    }

    /**
     * Commits this edit so it is visible to readers.  This releases the edit lock so another edit
     * may be started on the same key.
     * 写好数据，一定不要忘记commit操作对数据进行提交，我们要把dirtyFiles里面的内容移动到cleanFiles里才能够让别的editor访问到
     */
    public void commit() throws IOException {
      synchronized (DiskLruCache.this) {
        if (done) {
          throw new IllegalStateException();
        }
        if (entry.currentEditor == this) {
          completeEdit(this, true);
        }
        done = true;
      }
    }

    /**
     * Aborts this edit. This releases the edit lock so another edit may be started on the same
     * key.
     */
    public void abort() throws IOException {
      synchronized (DiskLruCache.this) {
        if (done) {
          throw new IllegalStateException();
        }
        if (entry.currentEditor == this) {
         //这个方法是DiskLruCache的方法在后面讲解
          completeEdit(this, false);
        }
        done = true;
      }
    }

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

哎，看到这个了类的注释，发现Editor的确就是编辑entry类的。
 Editor里面的几个方法Source newSource(int index) ，Sink newSink(int index)，commit()，abort()，abortUnlessCommitted() ，既然是编辑器，我们看到上面的方法应该可以猜到，上面的方法一次对应如下

| 方法                        |                             意义                             |
| --------------------------- | :----------------------------------------------------------: |
| Source newSource(int index) |               返回指定index的cleanFile的读入流               |
| Sink newSink(int index)     |             向指定index的dirtyFiles文件写入数据              |
| commit()                    | 这里执行的工作是提交数据，并释放锁，最后通知DiskLruCache刷新相关数据 |
| abort()                     |                      终止编辑，并释放锁                      |
| abortUnlessCommitted()      |                    除非正在编辑，否则终止                    |

abort()和abortUnlessCommitted()最后都会执行completeEdit(Editor, boolean) 这个方法这里简单说下：
 success情况提交：dirty文件会被更名为clean文件，entry.lengths[i]值会被更新，DiskLruCache,size会更新（DiskLruCache,size代表的是所有整个缓存文件加起来的总大小），redundantOpCount++，在日志中写入一条Clean信息
 failed情况：dirty文件被删除，redundantOpCount++，日志中写入一条REMOVE信息

至此DiskLruCache的内部类就全部介绍结束了。现在咱们正式关注下DiskLruCache类

## 二、[DiskLruCache](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/cache/DiskLruCache.java)类详解

### (一)、重要属性

DiskLruCache里面有一个属性是lruEntries如下：



```php
private final LinkedHashMap<String, Entry> lruEntries = new LinkedHashMap<>(0, 0.75f, true);

  /** Used to run 'cleanupRunnable' for journal rebuilds. */
  private final Executor executor;
```

LinkedHashMap自带Lru算法的光环属性，详情请看LinkedHashMap源码说明
 DiskLruCache也有一个线程池属性 executor，不过该池最多有一个线程工作，用于清理，维护缓存数据。创建一个DiskLruCache对象的方法是调用该方法，而不是直接调用构造器。

### (二)、构造函数和创建对象

DiskLruCache有一个构造函数，但是不是public的所以DiskLruCache只能被包内中类调用，不能在外面直接new。不过DiskLruCache提供了一个静态方法create，对外提供DiskLruCache对象



```dart
//DiskLruCache.java
  /**
   * Create a cache which will reside in {@code directory}. This cache is lazily initialized on
   * first access and will be created if it does not exist.
   *
   * @param directory a writable directory
   * @param valueCount the number of values per cache entry. Must be positive.
   * @param maxSize the maximum number of bytes this cache should use to store
   */
  public static DiskLruCache create(FileSystem fileSystem, File directory, int appVersion,
      int valueCount, long maxSize) {
    if (maxSize <= 0) {
      throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
      throw new IllegalArgumentException("valueCount <= 0");
    }
    //这个executor其实就是DiskLruCache里面的executor
    // Use a single background thread to evict entries.
    Executor executor = new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<Runnable>(), Util.threadFactory("OkHttp DiskLruCache", true));

    return new DiskLruCache(fileSystem, directory, appVersion, valueCount, maxSize, executor);
  }

  static final String JOURNAL_FILE = "journal";  
  static final String JOURNAL_FILE_TEMP = "journal.tmp";  
  static final String JOURNAL_FILE_BACKUP = "journal.bkp"  

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
```

该构造器会在制定的目录下创建三份文件，这三个文件是DiskLruCache的工作日志文件。在执行DiskLruCache的任何方法之前都会执行initialize()方法来完成DiskLruCache的初始化，有人会想为什么不在DiskLruCache的构造器中完成对该方法的调用，其实是为了延迟初始化，因为初始化会创建一系列的文件和对象，所以做了延迟初始化。

### (三)、初始化

那么来看下initialize里面的代码



```java
  public synchronized void initialize() throws IOException {
 
    //断言，当持有自己锁的时候。继续执行，没有持有锁，直接抛异常
    assert Thread.holdsLock(this);
    //如果已经初始化过，则不需要再初始化，直接rerturn
    if (initialized) {
      return; // Already initialized.
    }

    // If a bkp file exists, use it instead.
     //如果有journalFileBackup文件
    if (fileSystem.exists(journalFileBackup)) {
      // If journal file also exists just delete backup file.
      //如果有journalFile文件
      if (fileSystem.exists(journalFile)) {
        //有journalFile文件 则删除journalFileBackup文件
        fileSystem.delete(journalFileBackup);
      } else {
         //没有journalFile，则将journalFileBackUp更名为journalFile
        fileSystem.rename(journalFileBackup, journalFile);
      }
    }

    // Prefer to pick up where we left off.
    if (fileSystem.exists(journalFile)) {
       //如果有journalFile文件，则对该文件，则分别调用readJournal()方法和processJournal()方法
      try {
        readJournal();
        processJournal();
        //设置初始化过标志
        initialized = true;
        return;
      } catch (IOException journalIsCorrupt) {
        Platform.get().log(WARN, "DiskLruCache " + directory + " is corrupt: "
            + journalIsCorrupt.getMessage() + ", removing", journalIsCorrupt);
      }

      // The cache is corrupted, attempt to delete the contents of the directory. This can throw and
      // we'll let that propagate out as it likely means there is a severe filesystem problem.
      try {
        //如果没有journalFile则删除
        delete();
      } finally {
        closed = false;
      }
    }
     //重新建立journal文件
    rebuildJournal();
    initialized = true;
  }
```

大家发现没有，如论是否有journal文件，最后都会将initialized设为true,该值不会再被设置为false，除非DiskLruCache对象呗销毁。这表明initialize()放啊在DiskLruCache对象的整个生命周期中只会执行一次。该动作完成日志的写入和lruEntries集合的初始化。
 这里面分别调用了readJournal()方法和processJournal()方法，那咱们依次分析下这两个方法,这里面有大量的okio里面的代码，如果大家对okio不熟悉能读上一篇文章。



```csharp
private void readJournal() throws IOException {
     //获取journalFile的source即输入流
    BufferedSource source = Okio.buffer(fileSystem.source(journalFile));
    try {
     //读取相关数据
      String magic = source.readUtf8LineStrict();
      String version = source.readUtf8LineStrict();
      String appVersionString = source.readUtf8LineStrict();
      String valueCountString = source.readUtf8LineStrict();
      String blank = source.readUtf8LineStrict();
      //做校验
      if (!MAGIC.equals(magic)
          || !VERSION_1.equals(version)
          || !Integer.toString(appVersion).equals(appVersionString)
          || !Integer.toString(valueCount).equals(valueCountString)
          || !"".equals(blank)) {
        throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
            + valueCountString + ", " + blank + "]");
      }

      int lineCount = 0;
     //校验通过，开始逐行读取数据
      while (true) {
        try {
          readJournalLine(source.readUtf8LineStrict());
          lineCount++;
        } catch (EOFException endOfJournal) {
          break;
        }
      }
     //读取出来的行数减去lruEntriest的集合的差值，即日志多出的"冗余"记录
      redundantOpCount = lineCount - lruEntries.size();
      // If we ended on a truncated line, rebuild the journal before appending to it.
      //source.exhausted()表示是否还多余字节，如果没有多余字节，返回true，有多月字节返回false
      if (!source.exhausted()) {
       //如果有多余字节，则重新构建下journal文件，主要是写入头文件，以便下次读的时候，根据头文件进行校验
        rebuildJournal();
      } else {
        //获取这个文件的Sink
        journalWriter = newJournalWriter();
      }
    } finally {
      Util.closeQuietly(source);
    }
  }
```

这里说一下ource.readUtf8LineStrict()方法，这个方法是BufferedSource接口的方法，具体实现是RealBufferedSource，所以大家要去RealBufferedSource里面去找具体实现。我这里简单说下，就是从source里面按照utf-8编码取出一行的数据。这里面读取了magic，version，appVersionString，valueCountString，blank，然后进行校验，这个数据是在"写"的时候，写入的，具体情况看DiskLruCache的rebuildJournal()方法。随后记录redundantOpCount的值，该值的含义就是判断当前日志中记录的行数和lruEntries集合容量的差值，即日志中多出来的"冗余"记录。
 读取的时候又调用了readJournalLine()方法，咱们来研究下这个方法



```dart
private void readJournalLine(String line) throws IOException {
    获取空串的position，表示头
    int firstSpace = line.indexOf(' ');
    //空串的校验
    if (firstSpace == -1) {
      throw new IOException("unexpected journal line: " + line);
    }
    //第一个字符的位置
    int keyBegin = firstSpace + 1;
    // 方法返回第一个空字符在此字符串中第一次出现，在指定的索引即keyBegin开始搜索，所以secondSpace是爱这个字符串中的空字符(不包括这一行最左侧的那个空字符)
    int secondSpace = line.indexOf(' ', keyBegin);
    final String key;
    //如果没有中间的空字符
    if (secondSpace == -1) {
     //截取剩下的全部字符串构成key
      key = line.substring(keyBegin);
      if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
         //如果解析的是REMOVE信息，则在lruEntries里面删除这个key
        lruEntries.remove(key);
        return;
      }
    } else {
     //如果含有中间间隔的空字符，则截取这个中间间隔到左侧空字符之间的字符串，构成key
      key = line.substring(keyBegin, secondSpace);
    }
    //获取key后，根据key取出Entry对象
    Entry entry = lruEntries.get(key);
   //如果Entry为null，则表明内存中没有，则new一个，并把它放到内存中。
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);
    }
    //如果是CLEAN开头
    if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
     //line.substring(secondSpace + 1) 为获取中间空格后面的内容，然后按照空字符分割，设置entry的属性，表明是干净的数据，不能编辑。
      String[] parts = line.substring(secondSpace + 1).split(" ");
      entry.readable = true;
      entry.currentEditor = null;
      entry.setLengths(parts);
    } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
      //如果是以DIRTY开头，则设置一个新的Editor，表明可编辑
      entry.currentEditor = new Editor(entry);
    } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
      // This work was already done by calling lruEntries.get().
    } else {
      throw new IOException("unexpected journal line: " + line);
    }
  }
```

这里面主要是具体的解析，如果每次解析的是非REMOVE信息，利用该key创建一个entry，如果是判断信息是CLEAN则设置ENTRY为可读，并设置entry.currentEditor表明当前Entry不可编辑，调用entry.setLengths(String[])，设置该entry.lengths的初始值。如果判断是Dirty则设置enry.currentEdtor=new Editor(entry)；表明当前Entry处于被编辑状态。

**通过上面我得到了如下的结论：**

- 1、如果是CLEAN的话，对这个entry的文件长度进行更新
- 2、如果是DIRTY，说明这个值正在被操作，还没有commit，于是给entry分配一个Editor。
- 3、如果是READ，说明这个值被读过了，什么也不做。

看下journal文件你就知道了



```objectivec
 1 *     libcore.io.DiskLruCache
 2 *     1
 3 *     100
 4 *     2
 5 *
 6 *     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
 7 *     DIRTY 335c4c6028171cfddfbaae1a9c313c52
 8 *     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
 9 *     REMOVE 335c4c6028171cfddfbaae1a9c313c52
10 *     DIRTY 1ab96a171faeeee38496d8b330771a7a
11 *     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
12 *     READ 335c4c6028171cfddfbaae1a9c313c52
13 *     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

然后又调用了processJournal()方法，那我们来看下：



```cpp
  /**
   * Computes the initial size and collects garbage as a part of opening the cache. Dirty entries
   * are assumed to be inconsistent and will be deleted.
   */
  private void processJournal() throws IOException {
    fileSystem.delete(journalFileTmp);
    for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
      Entry entry = i.next();
      if (entry.currentEditor == null) {
        for (int t = 0; t < valueCount; t++) {
          size += entry.lengths[t];
        }
      } else {
        entry.currentEditor = null;
        for (int t = 0; t < valueCount; t++) {
          fileSystem.delete(entry.cleanFiles[t]);
          fileSystem.delete(entry.dirtyFiles[t]);
        }
        i.remove();
      }
    }
  }
```

先是删除了journalFileTmp文件
 然后调用for循环获取链表中的所有Entry，如果Entry的中Editor!=null，则表明Entry数据时脏的DIRTY，所以不能读，进而删除Entry下的缓存文件，并且将Entry从lruEntries中移除。如果Entry的Editor==null，则证明该Entry下的缓存文件可用，记录它所有缓存文件的缓存数量，结果赋值给size。
 readJournal()方法里面调用了rebuildJournal()，initialize()方法同样会readJourna，但是这里说明下：readJournal里面调用的rebuildJournal()是有条件限制的，initialize()是一定会调用的。那我们来研究下readJournal()



```java
 /**
   * Creates a new journal that omits redundant information. This replaces the current journal if it
   * exists.
   */
  synchronized void rebuildJournal() throws IOException {
    //如果写入流不为空
    if (journalWriter != null) {
      //关闭写入流
      journalWriter.close();
    }
   //通过okio获取一个写入BufferedSinke
    BufferedSink writer = Okio.buffer(fileSystem.sink(journalFileTmp));
    try {
     //写入相关信息和读取向对应，这时候大家想下readJournal
      writer.writeUtf8(MAGIC).writeByte('\n');
      writer.writeUtf8(VERSION_1).writeByte('\n');
      writer.writeDecimalLong(appVersion).writeByte('\n');
      writer.writeDecimalLong(valueCount).writeByte('\n');
      writer.writeByte('\n');
    
      //遍历lruEntries里面的值
      for (Entry entry : lruEntries.values()) {
        //如果editor不为null，则为DIRTY数据
        if (entry.currentEditor != null) {
           在开头写上 DIRTY，然后写上 空字符
          writer.writeUtf8(DIRTY).writeByte(' ');
           //把entry的key写上
          writer.writeUtf8(entry.key);
          //换行
          writer.writeByte('\n');
        } else {
          //如果editor为null，则为CLEAN数据,  在开头写上 CLEAN，然后写上 空字符
          writer.writeUtf8(CLEAN).writeByte(' ');
           //把entry的key写上
          writer.writeUtf8(entry.key);
          //结尾接上两个十进制的数字，表示长度
          entry.writeLengths(writer);
          //换行
          writer.writeByte('\n');
        }
      }
    } finally {
      //最后关闭写入流
      writer.close();
    }
   //如果存在journalFile
    if (fileSystem.exists(journalFile)) {
      //把journalFile文件重命名为journalFileBackup
      fileSystem.rename(journalFile, journalFileBackup);
    }
    然后又把临时文件，重命名为journalFile
    fileSystem.rename(journalFileTmp, journalFile);
    //删除备份文件
    fileSystem.delete(journalFileBackup);
    //拼接一个新的写入流
    journalWriter = newJournalWriter();
    //设置没有error标志
    hasJournalErrors = false;
    //设置最近重新创建journal文件成功
    mostRecentRebuildFailed = false;
  }
```

总结下：
 获取一个写入流，将lruEntries集合中的Entry对象写入tmp文件中，根据Entry的currentEditor的值判断是CLEAN还是DIRTY,写入该Entry的key，如果是CLEAN还要写入文件的大小bytes。然后就是把journalFileTmp更名为journalFile，然后将journalWriter跟文件绑定，通过它来向journalWrite写入数据，最后设置一些属性。
 我们可以砍到，rebuild操作是以lruEntries为准，把DIRTY和CLEAN的操作都写回到journal中。但发现没有，其实没有改动真正的value，只不过重写了一些事务的记录。事实上，lruEntries和journal文件共同确定了cache数据的有效性。lruEntries是索引，journal是归档。至此序列化部分就已经结束了

### (四)、关于Cache类调用的几个方法

上回书说道Cache调用DiskCache的几个方法，如下：

> 

- 1.DiskLruCache.get(String)获取DiskLruCache.Snapshot
- 2.DiskLruCache.remove(String)移除请求
- 3.DiskLruCache.edit(String)；获得一个DiskLruCache.Editor对象，
- 4.DiskLruCache.Editor.newSink(int)；获得一个sink流  (具体看Editor类)
- 5.DiskLruCache.Snapshot.getSource(int)；获取一个Source对象。  (具体看Editor类)
- 6.DiskLruCache.Snapshot.edit()；获得一个DiskLruCache.Editor对象，

**1、DiskLruCache.Snapshot  get(String)方法**

```java
  public synchronized Snapshot get(String key) throws IOException {
    //初始化
    initialize();
    //检查缓存是否已经关闭
    checkNotClosed();
    //检验key
    validateKey(key);
    //如果以上都通过，先获取内存中的数据，即根据key在linkedList查找
    Entry entry = lruEntries.get(key);
    //如果没有值，或者有值，但是值不可读
    if (entry == null || !entry.readable) return null;
    //获取entry里面的snapshot的值
    Snapshot snapshot = entry.snapshot();
    //如果有snapshot为null，则直接返回null
    if (snapshot == null) return null;
    //如果snapshot不为null
    //计数器自加1
    redundantOpCount++;
    //把这个内容写入文档中
    journalWriter.writeUtf8(READ).writeByte(' ').writeUtf8(key).writeByte('\n');
    //如果超过上限
    if (journalRebuildRequired()) {
      //开始清理
      executor.execute(cleanupRunnable);
    }
    //返回数据
    return snapshot;
  }


  /**
   * We only rebuild the journal when it will halve the size of the journal and eliminate at least
   * 2000 ops.
   */
  boolean journalRebuildRequired() {
    //最大计数单位
    final int redundantOpCompactThreshold = 2000;
    //清理的条件
    return redundantOpCount >= redundantOpCompactThreshold
        && redundantOpCount >= lruEntries.size();
  }
```

主要就是先去拿snapshot，然后会用journalWriter向journal写入一条read记录，最后判断是否需要清理。
 清理的条件是当前redundantOpCount大于2000，并且redundantOpCount的值大于linkedList里面的size。咱们接着看下清理任务



```java
private final Runnable cleanupRunnable = new Runnable() {
    public void run() {
      synchronized (DiskLruCache.this) {
        //如果没有初始化或者已经关闭了，则不需要清理
        if (!initialized | closed) {
          return; // Nothing to do
        }

        try {
          trimToSize();
        } catch (IOException ignored) {
         //如果抛异常了，设置最近的一次清理失败
          mostRecentTrimFailed = true;
        }

        try {
          //如果需要清理了
          if (journalRebuildRequired()) {
            //重新创建journal文件
            rebuildJournal();
            //计数器归于0
            redundantOpCount = 0;
          }
        } catch (IOException e) {
          //如果抛异常了，设置最近的一次构建失败
          mostRecentRebuildFailed = true;
          journalWriter = Okio.buffer(Okio.blackhole());
        }
      }
    }
  };


  void trimToSize() throws IOException {
    //如果超过上限
    while (size > maxSize) {
      //取出一个Entry
      Entry toEvict = lruEntries.values().iterator().next();
      //删除这个Entry
      removeEntry(toEvict);
    }
    mostRecentTrimFailed = false;
  }

  boolean removeEntry(Entry entry) throws IOException {
    if (entry.currentEditor != null) {
     //让这个editor正常的结束
      entry.currentEditor.detach(); // Prevent the edit from completing normally.
    }
   
    for (int i = 0; i < valueCount; i++) {
      //删除entry对应的clean文件
      fileSystem.delete(entry.cleanFiles[i]);
      //缓存大小减去entry的小小
      size -= entry.lengths[i];
      //设置entry的缓存为0
      entry.lengths[i] = 0;
    }
    //计数器自加1
    redundantOpCount++;
    //在journalWriter添加一条删除记录
    journalWriter.writeUtf8(REMOVE).writeByte(' ').writeUtf8(entry.key).writeByte('\n');
    //linkedList删除这个entry
    lruEntries.remove(entry.key);
    //如果需要重新构建
    if (journalRebuildRequired()) {
      //开启清理任务
      executor.execute(cleanupRunnable);
    }
    return true;
  }
```

看下cleanupRunnable对象，看他的run方法得知，主要是调用了trimToSize()和rebuildJournal()两个方法对缓存数据进行维护。rebuildJournal()前面已经说过了，这里主要关注下trimToSize()方法，trimToSize()方法主要是遍历lruEntries(注意：这个遍历科室通过accessOrder来的，也就是隐含了LRU这个算法)，来一个一个移除entry直到size小于maxSize，而removeEntry操作就是讲editor里的diryFile以及cleanFiles进行删除就是，并且要向journal文件里写入REMOVE操作，以及删除lruEntrie里面的对象。
 cleanup主要是用来调整整个cache的大小，以防止它过大，同时也能用来rebuildJournal，如果trim或者rebuild不成功，那之前edit里面也是没有办法获取Editor来进行数据修改操作的。

下面来看下boolean remove(String key)方法



```java
  /**
   * Drops the entry for {@code key} if it exists and can be removed. If the entry for {@code key}
   * is currently being edited, that edit will complete normally but its value will not be stored.
   *根据key来删除对应的entry，如果entry存在则将会被删除，如果这个entry正在被编辑，编辑将被正常结束，但是编辑的内容不会保存
   * @return true if an entry was removed.
   */
  public synchronized boolean remove(String key) throws IOException {
    //初始化
    initialize();
    //检查是否被关闭
    checkNotClosed();
    //key是否符合要求
    validateKey(key);
    //根据key来获取Entry
    Entry entry = lruEntries.get(key);
    //如果entry，返回false表示删除失败
    if (entry == null) return false;
     //然后删除这个entry
    boolean removed = removeEntry(entry);
    //如果删除成功且缓存大小小于最大值，则设置最近清理标志位
    if (removed && size <= maxSize) mostRecentTrimFailed = false;
    return removed;
  }
```

这这部分很简单，就是先做判断，然后通过key获取Entry，然后删除entry
 那我们继续，来看下DiskLruCache.edit(String)；方法



```java
  /**
   * Returns an editor for the entry named {@code key}, or null if another edit is in progress.
   * 返回一entry的编辑器，如果其他正在编辑，则返回null
   * 我的理解是根据key找entry，然后根据entry找他的编辑器
   */
  public Editor edit(String key) throws IOException {
    return edit(key, ANY_SEQUENCE_NUMBER);
  }

  synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
    //初始化
    initialize();
    //流关闭检测
    checkNotClosed();
     //检测key
    validateKey(key);
    //根据key找到Entry
    Entry entry = lruEntries.get(key);
    //如果快照是旧的
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
        || entry.sequenceNumber != expectedSequenceNumber)) {
      return null; // Snapshot is stale.
    }
   //如果 entry.currentEditor != null 表明正在编辑，是DIRTY
    if (entry != null && entry.currentEditor != null) {
      return null; // Another edit is in progress.
    }
    //如果最近清理失败，或者最近重新构建失败，我们需要开始清理任务
   //我大概翻译下注释：操作系统已经成为我们的敌人，如果清理任务失败，它意味着我们存储了过多的数据，因此我们允许超过这个限制，所以不建议编辑。如果构建日志失败，writer这个写入流就会无效，所以文件无法及时更新，导致我们无法继续编辑，会引起文件泄露。如果满足以上两种情况，我们必须进行清理，摆脱这种不好的状态。
    if (mostRecentTrimFailed || mostRecentRebuildFailed) {
      // The OS has become our enemy! If the trim job failed, it means we are storing more data than
      // requested by the user. Do not allow edits so we do not go over that limit any further. If
      // the journal rebuild failed, the journal writer will not be active, meaning we will not be
      // able to record the edit, causing file leaks. In both cases, we want to retry the clean up
      // so we can get out of this state!
      //开启清理任务
      executor.execute(cleanupRunnable);
      return null;
    }

    // Flush the journal before creating files to prevent file leaks.
    //写入DIRTY
    journalWriter.writeUtf8(DIRTY).writeByte(' ').writeUtf8(key).writeByte('\n');
    journalWriter.flush();
   //如果journal有错误，表示不能编辑，返回null
    if (hasJournalErrors) {
      return null; // Don't edit; the journal can't be written.
    }
   //如果entry==null，则new一个，并放入lruEntries
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);
    }
   //根据entry 构造一个Editor
    Editor editor = new Editor(entry);
    entry.currentEditor = editor;
    return editor;
  }
```

上面代码注释说的很清楚，这里就提几个注意事项
 注意事项：
 (1)如果已经有个别的editor在操作这个entry了，那就返回null
 (2)无时无刻不在进行cleanup判断进行cleanup操作
 (3)会把当前的key在journal文件标记为dirty状态，表示这条记录正在被编辑
 (4)如果没有entry，会new一个出来

这个方法已经结束了，那我们来看下 在Editor内部类commit()方法里面调用的completeEdit(Editor,success)方法



```java
synchronized void completeEdit(Editor editor, boolean success) throws IOException {
    Entry entry = editor.entry;
    //如果entry的编辑器不是editor则抛异常
    if (entry.currentEditor != editor) {
      throw new IllegalStateException();
    }

    // If this edit is creating the entry for the first time, every index must have a value.
    //如果successs是true,且entry不可读表明 是第一次写回，必须保证每个index里面要有数据，这是为了保证完整性
    if (success && !entry.readable) {
      for (int i = 0; i < valueCount; i++) {
        if (!editor.written[i]) {
          editor.abort();
          throw new IllegalStateException("Newly created entry didn't create value for index " + i);
        }
        if (!fileSystem.exists(entry.dirtyFiles[i])) {
          editor.abort();
          return;
        }
      }
    }
   //遍历entry下的所有文件
    for (int i = 0; i < valueCount; i++) {
      File dirty = entry.dirtyFiles[i];
      if (success) {
        //把dirtyFile重命名为cleanFile，完成数据迁移;
        if (fileSystem.exists(dirty)) {
          File clean = entry.cleanFiles[i];
          fileSystem.rename(dirty, clean);
          long oldLength = entry.lengths[i];
          long newLength = fileSystem.size(clean);
          entry.lengths[i] = newLength;
          size = size - oldLength + newLength;
        }
      } else {
       //删除dirty数据
        fileSystem.delete(dirty);
      }
    }
    //计数器加1
    redundantOpCount++;
    //编辑器指向null
    entry.currentEditor = null;

    if (entry.readable | success) {
      //开始写入数据
      entry.readable = true;
      journalWriter.writeUtf8(CLEAN).writeByte(' ');
      journalWriter.writeUtf8(entry.key);
      entry.writeLengths(journalWriter);
      journalWriter.writeByte('\n');
      if (success) {
        entry.sequenceNumber = nextSequenceNumber++;
      }
    } else {
     //删除key，并且记录
      lruEntries.remove(entry.key);
      journalWriter.writeUtf8(REMOVE).writeByte(' ');
      journalWriter.writeUtf8(entry.key);
      journalWriter.writeByte('\n');
    }
    journalWriter.flush();
    //检查是否需要清理
    if (size > maxSize || journalRebuildRequired()) {
      executor.execute(cleanupRunnable);
    }
  }
```

这样下来，数据都写入cleanFile了，currentEditor也重新设为null，表明commit彻底结束了。

总结起来DiskLruCache主要的特点：

> 

- 1、通过LinkedHashMap实现LRU替换
- 2、通过本地维护Cache操作日志保证Cache原子性与可用性，同时为防止日志过分膨胀定时执行日志精简。
- 3、 每一个Cache项对应两个状态副本：DIRTY，CLEAN。CLEAN表示当前可用的Cache。外部访问到cache快照均为CLEAN状态；DIRTY为编辑状态的cache。由于更新和创新都只操作DIRTY状态的副本，实现了读和写的分离。
- 4、每一个url请求cache有四个文件，两个状态(DIRY，CLEAN)，每个状态对应两个文件：一个0文件对应存储meta数据，一个文件存储body数据。

***至此所有的关于缓存的相关类都介绍完毕，为了帮助大家更好的理解缓存，咱们在重新看下CacheInterceptor里面执行的流程***

## 三.OKHTTP的缓存的实现---CacheInterceptor的具体执行流程

### (一)原理和注意事项：

1、原理
 (1)、okhttp的网络缓存是基于http协议，不清楚请仔细看上一篇文章
 (2)、使用DiskLruCache的缓存策略，具体请看本片文章的第一章节
 2、注意事项：
 1、目前只支持GET，其他请求方式需要自己实现。
 2、需要服务器配合，通过head设置相关头来控制缓存
 3、创建OkHttpClient时候需要配置Cache

### (二)流程：

> 

1、如果配置了缓存，则从缓存中取出(可能为null)
 2、获取缓存的策略.
 3、监测缓存
 4、如果禁止使用网络(比如飞行模式),且缓存无效，直接返回
 5、如果缓存有效，使用网络，不使用网络
 6、如果缓存无效，执行下一个拦截器
 7、本地有缓存、根据条件判断是使用缓存还是使用网络的response
 8、把response缓存到本地

### (三)源码对比：



```kotlin
@Override public Response intercept(Chain chain) throws IOException {
    //1、如果配置了缓存，则从缓存中取出(可能为null)
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //2、获取缓存的策略.
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    //3、监测缓存
    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
      //4、如果禁止使用网络(比如飞行模式),且缓存无效，直接返回
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
    //5、如果缓存有效，使用网络，不使用网络
    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
     //6、如果缓存无效，执行下一个拦截器
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
    //7、本地有缓存、根据条件判断是使用缓存还是使用网络的response
    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }
    //这个response是用来返回的
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    //8、把response缓存到本地
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

### (四)倒序具体分析：

1、什么是“倒序具体分析”？
 这里的倒序具体分析是指先分析缓存，在分析使用缓存，因为第一次使用的时候，肯定没有缓存，所以肯定先发起请求request，然后收到响应response的时候，缓存起来，等下次调用的时候，才具体获取缓存策略。

**PS:由于涉及到的类全部讲过了一遍了，下面涉及的代码就不全部粘贴了，只赞贴核心代码了。**

2、先分析获取响应response的流程,保存的流程是如下
 在CacheInterceptor的代码是

```kotlin
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }
    }
```

核心代码是CacheRequest cacheRequest = cache.put(response);
 cache就是咱们设置的Cache对象，put(reponse)方法就是调用Cache类的put方法



```kotlin
Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      entry.writeTo(editor);
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
```

*先是* 用resonse作为参数来构造Cache.Entry对象，这里强烈提示下，是Cache.Entry对象，不是DiskLruCache.Entry对象。 *然后* 调用的是DiskLruCache类的edit(String key)方法，而DiskLruCache类的edit(String key)方法调用的是DiskLruCache类的edit(String key, long expectedSequenceNumber)方法,在DiskLruCache类的edit(String key, long expectedSequenceNumber)方法里面其实是通过lruEntries的 lruEntries.get(key)方法获取的DiskLruCache.Entry对象，然后通过这个DiskLruCache.Entry获取对应的编辑器,获取到编辑器后， *再次*这个编辑器(editor)通过okio把Cache.Entry写入这个编辑器(editor)对应的文件上。注意，这里是写入的是http中的header的内容 ，*最后* 返回一个CacheRequestImpl对象
 紧接着又调用了 CacheInterceptor.cacheWritingResponse(CacheRequest, Response)方法

主要就是通过配置好的cache写入缓存，都是通过Cache和DiskLruCache来具体实现

总结：缓存实际上是一个比较复杂的逻辑，单独的功能块，实际上不属于OKhttp上的功能，实际上是通过是http协议和DiskLruCache做了处理。

LinkedHashMap可以实现LRU算法，并且在这个case里，它被用作对DiskCache的内存索引
 告诉你们一个秘密，Universal-Imager-Loader里面的DiskLruCache的实现跟这里的一模一样，除了io使用inputstream/outputstream
 使用LinkedHashMap和journal文件同时记录做过的操作，其实也就是有索引了，这样就相当于有两个备份，可以互相恢复状态
 通过dirtyFiles和cleanFiles，可以实现更新和读取同时操作，在commit的时候将cleanFiles的内容进行更新就好了

# OKHttp源码解析(八)--中阶之连接与请求前奏

## 一、为什么要做app网络优化

**1、keepalive**

在http请求中，对于请求速度提升和降低延迟，keepalive在网络连接发挥着重大作用。

所有做过http请求的同学都知道http请求的三次握手和4次挥手,所以我们一般的http链接都是先tcp握手，然后传输数据，最后释放资源。流程如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-0680d97f1ad2695d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

链接.png

的确很简单，但是在复杂的网路内容中就不够用了，因为需要先进行socket的3次握手，和四次挥手，重复的连接和释放，就像“便秘”一样，十分难受！而且每次链接大概是TTL的一次的时间。特别现在IOS那边已经HTTPS了，安卓这边HTTPS也是趋势，在TLS环境下消耗的时间更多了。很明显在复杂网络时，延时(而不是带宽)将成为一个app非常重要的核心竞争因素，特别是在移动网络的使用场景下。

当然，其实这个问题老早就已经fix了，在http里面有一个叫做keepalive connections的机制，它可以在传输数据后仍然保持连接，当客户端需要再次获取数据的时候，直接使用刚刚空闲下来的链接而不需要再次握手。

![img](https:////upload-images.jianshu.io/upload_images/5713484-1e68dd04fbc424fe.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

在PC的浏览器里面，一般会同时开启6-8个keepalive connections的socket链接，并保持一定的链路生命，当不需要时再关闭，而在服务器中，一般由软件根据负载的情况，决定是否关闭。

当然上帝给你打开一扇窗子的时候，也会关闭另外一扇窗户，凡事皆有利弊。keepalive也有缺点，在减轻客户端的延迟的同时，也妨碍了其它客户端的链路速度。比如如果存在大量空闲的keepalive connections(僵尸链接)，其它客户端正常的链接速度就会受到影响，因为毕竟的的管道大小是确定。而且有一种恶意的攻击就是产生大量的僵尸链接，耗尽服务器的资源

## 二、ConnectionSpec与ConnectionSpecSelector简介

(一)、[ConnectionSpec](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/ConnectionSpec.java)
 在OkHttp中，ConnectionSpec用于描述传输HTTP流量的socket连接的配置。对于https请求，这些配置主要包括协商安全连接时要使用的TLS版本号和密码套间，是否支持TLS扩展等；对于http请求则几乎包含什么信息。
 OkHttp有预定义几组ConnectionSpec ,如下：



```php
  //ConnectionSpec.java
  /** A modern TLS connection with extensions like SNI and ALPN available. */
  public static final ConnectionSpec MODERN_TLS = new Builder(true)
      .cipherSuites(APPROVED_CIPHER_SUITES)
      .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2, TlsVersion.TLS_1_1, TlsVersion.TLS_1_0)
      .supportsTlsExtensions(true)
      .build();

  /** A backwards-compatible fallback connection for interop with obsolete servers. */
  public static final ConnectionSpec COMPATIBLE_TLS = new Builder(MODERN_TLS)
      .tlsVersions(TlsVersion.TLS_1_0)
      .supportsTlsExtensions(true)
      .build();
```

预定义的这些ConnectionSpec被组织定义为OkHttpClint默认的ConncetionSpec集合



```php
//OkHttpClient.java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  static final List<Protocol> DEFAULT_PROTOCOLS = Util.immutableList(
      Protocol.HTTP_2, Protocol.HTTP_1_1);

  static final List<ConnectionSpec> DEFAULT_CONNECTION_SPECS = Util.immutableList(
      ConnectionSpec.MODERN_TLS, ConnectionSpec.CLEARTEXT);
}
```

OkHttp中由OkHttpClient管理ConnectionSpec集合。OkHttp的用户可以构造OkHttpClient的过程中提供自己的ConnectionSpec集合。默认情况下OkHttpClient会使用前面的默认ConnectionSpec集合。

OKHttp还提供了ConnectionSpecSelector，用以从ConnectionSpec几个中选择与SSLSocket匹配的ConnectionSpec，并对SSLSocket做配置操作。

在RetryAndFollowUpInterceptor中创建Address时，ConnectionSpec集合被从OkHttpClient获取，并由Anddress引用。

在StreamAllocation的findConnection()中，ConnectionSpec集合被从Address中取出来，用于连接建立过程。

下面介绍一个后面会调用的方法，isCompatible()方法。在ConnectionSpec集合中选择一个与SSLSocket兼容的一个，如果有兼容的返回true，不兼容返回false。



```kotlin
  /**
   * Returns {@code true} if the socket, as currently configured, supports this connection spec. In
   * order for a socket to be compatible the enabled cipher suites and protocols must intersect.
   *
   * <p>For cipher suites, at least one of the {@link #cipherSuites() required cipher suites} must
   * match the socket's enabled cipher suites. If there are no required cipher suites the socket
   * must have at least one cipher suite enabled.
   *
   * <p>For protocols, at least one of the {@link #tlsVersions() required protocols} must match the
   * socket's enabled protocols.
   */
  public boolean isCompatible(SSLSocket socket) {
    if (!tls) {
      return false;
    }

    if (tlsVersions != null && !nonEmptyIntersection(
        Util.NATURAL_ORDER, tlsVersions, socket.getEnabledProtocols())) {
      return false;
    }

    if (cipherSuites != null && !nonEmptyIntersection(
        CipherSuite.ORDER_BY_NAME, cipherSuites, socket.getEnabledCipherSuites())) {
      return false;
    }

    return true;
  }
```

isCompatible(SSLSocket) 里面调用Util.nonEmptyIntersection(Comparator, String[] , String[] )主要是对比两个String数组。必须每一个字符串都一样，才可以。即ConnectionSpec启动的TLS版本和密码套间与SSLSocket启动的有交集，如果有交集返回true，反之返回false。

再来看下，将选择的ConnectionSpec应用到SSLSocket上。



```java
  /** Applies this spec to {@code sslSocket}. */
  void apply(SSLSocket sslSocket, boolean isFallback) {
    ConnectionSpec specToApply = supportedSpec(sslSocket, isFallback);

    if (specToApply.tlsVersions != null) {
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
        : sslSocket.getEnabledCipherSuites();
    String[] tlsVersionsIntersection = tlsVersions != null
        ? intersect(Util.NATURAL_ORDER, sslSocket.getEnabledProtocols(), tlsVersions)
        : sslSocket.getEnabledProtocols();

    // In accordance with https://tools.ietf.org/html/draft-ietf-tls-downgrade-scsv-00
    // the SCSV cipher is added to signal that a protocol fallback has taken place.
    String[] supportedCipherSuites = sslSocket.getSupportedCipherSuites();
    int indexOfFallbackScsv = indexOf(
        CipherSuite.ORDER_BY_NAME, supportedCipherSuites, "TLS_FALLBACK_SCSV");
    if (isFallback && indexOfFallbackScsv != -1) {
      cipherSuitesIntersection = concat(
          cipherSuitesIntersection, supportedCipherSuites[indexOfFallbackScsv]);
    }

    return new Builder(this)
        .cipherSuites(cipherSuitesIntersection)
        .tlsVersions(tlsVersionsIntersection)
        .build();
  }
```

上面流程大体可以理解为：

> 

- 1、求得ConnectionSepc启动的TLS版本和密码套间与SSLSocket启动的TLS版本以及密码套间之间的交集，构造新的ConnectionSpec。
- 2、重新为SSLSocket设置启用的TLS版本及密码套间为上一步求得的交集。

现在我们再来看下和ConnectionSpec相关的一个类
 (二)、[ConnectionSpecSelector](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/connection/ConnectionSpecSelector.java)
 通过类的注释 我们可知该类是ConnectionSpec的选择类，当握手或者协议出现问题的时候，需要尝试不同的协议进行连接。
 处理连接规范回退策略：当安全套接字连接由于握手/协议问题而失败时，可能会使用不同的协议重试连接。当创建单个连接速的时候会被创建该了的实例。

这里先说一下后面会用到的一个方法，configureSecureSocket()方法



```java
 /**
   * Configures the supplied {@link SSLSocket} to connect to the specified host using an appropriate
   * {@link ConnectionSpec}. Returns the chosen {@link ConnectionSpec}, never {@code null}.
   *
   * @throws IOException if the socket does not support any of the TLS modes available
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
```

主要分为两个部分
 1 从OkHttp配置的ConnectionSpec集合中选择一个SSLSocket兼容的一个。
 2 将选择的ConnectionSpec应用到SSLSocket上。

## 三、HttpCodec类及他的子类

在okHttp中，HttpCodec是网络读写的管理类，也可以理解为解码器(注释上就是这样写的)，它有对应的两个子类，Http1Codec和Http2Codec，分别对应HTTP/1.1以及HTTP/2.0协议
 [HttpCodec](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http/HttpCodec.java)
 [Http1Codec](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http1/Http1Codec.java)
 [Http2Codec](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http2/Http2Codec.java)

### (一)、咱们先来看下HttpCodec接口。

```csharp
/** Encodes HTTP requests and decodes HTTP responses. */
public interface HttpCodec {
  /**
   * The timeout to use while discarding a stream of input data. Since this is used for connection
   * reuse, this timeout should be significantly less than the time it takes to establish a new
   * connection.
   */
  int DISCARD_STREAM_TIMEOUT_MILLIS = 100;

  /** Returns an output stream where the request body can be streamed. */
  Sink createRequestBody(Request request, long contentLength);

  /** This should update the HTTP engine's sentRequestMillis field. */
  void writeRequestHeaders(Request request) throws IOException;

  /** Flush the request to the underlying socket. */
  void flushRequest() throws IOException;

  /** Flush the request to the underlying socket and signal no more bytes will be transmitted. */
  void finishRequest() throws IOException;

  /**
   * Parses bytes of a response header from an HTTP transport.
   *
   * @param expectContinue true to return null if this is an intermediate response with a "100"
   *     response code. Otherwise this method never returns null.
   */
  Response.Builder readResponseHeaders(boolean expectContinue) throws IOException;

  /** Returns a stream that reads the response body. */
  ResponseBody openResponseBody(Response response) throws IOException;

  /**
   * Cancel this stream. Resources held by this stream will be cleaned up, though not synchronously.
   * That may happen later by the connection pool thread.
   */
  void cancel();
}
```

里面定义了7个抽象方法分别是：

> 

- writeRequestHeaders(Request request) ：写入请求头
- createRequestBody(Request request, long contentLength) ：写入请求体
- flushRequest()  相当于flush,把请求刷入底层socket
- finishRequest() throws IOException : 相当于flush，把请求输入底层socket并不在发出请求
- readResponseHeaders(boolean expectContinue)  //读取响应头
- openResponseBody(Response response) //读取响应体
- void cancel() ：取消请求

典型的面向接口变成，抽象了几个网络请求的几个行为，一般的网络请求包括特殊情况基本上就是下面的四个步骤

> 

- 第一步，写入请求头
- 第二步，写入请求头
- 第三步，读取响应头
- 第四步，读取响应体

因为OkHttp是同时支持HTTP/2与HTTP/1.x的，为了让上层更方便的调用。所以在CallServerInterceptor类里面使用都是HttpCodec。由于HTTP/2与HTTP/1.x的原理不一样，那就由自己的子类去实现即可。所以有了Http1Codec与Http2Codec，

### (二)、那我们来看下Http1Codec与Http2Codec

**1、先来看下Http1Codec**

在Http1Codec中主要两个重要属性即source和sink，它们分别封装了socket的输入和输出。而Http2Codec中主要属性是Http2Stream和Http2Connection。
 Http1Codec提供I/O操作完成网络通信。
 这里先简单的介绍下Http1Codec这里先看下Http1Codec的各种状态的定义：



```java
  private static final int STATE_IDLE = 0; // Idle connections are ready to write request headers.
  private static final int STATE_OPEN_REQUEST_BODY = 1;
  private static final int STATE_WRITING_REQUEST_BODY = 2;
  private static final int STATE_READ_RESPONSE_HEADERS = 3;
  private static final int STATE_OPEN_RESPONSE_BODY = 4;
  private static final int STATE_READING_RESPONSE_BODY = 5;
  private static final int STATE_CLOSED = 6;
```

上面定义了Http1Codec中使用的的状态模式，其实就是对象维护它所处的状态，在不同状态下执行对应的逻辑，并更新状态，在执行逻辑之前通过检查对象状态避免网络请求的若干执行步骤发生错乱。
 再来看下:Http1Codec的属性



```php
  /** The client that configures this stream. May be null for HTTPS proxy tunnels. */
  final OkHttpClient client;
  /** The stream allocation that owns this stream. May be null for HTTPS proxy tunnels. */
  final StreamAllocation streamAllocation;

  final BufferedSource source;
  final BufferedSink sink;
  int state = STATE_IDLE;
```

这些属性很容易理解，首先持有client，可以使用它所提供的功能，通常是获取一些用户所设置的属性，其次是streamAllocation，它是连接流的桥梁，所以很容易理解需要它获取关于连接的功能。然后就是该流对象封装的输出流和输入流，两个流内部封装的自然就是socket的了，最后就是对象所处的状态。介绍完属性之后，我们就可以再来分析下Http1Codec提供的能功能了，在这些步骤中，逻辑很很明确，首先检查状态，然后执行逻辑，最后更新状态。当然执行逻辑和更新状态时可以交换的，不会造成影响，这步骤分析中我们不再考虑状态的问题，重点是关系逻辑的执行。
 来看下写请求头的方法



```dart
  /**
   * Prepares the HTTP headers and sends them to the server.
   *
   * <p>For streaming requests with a body, headers must be prepared <strong>before</strong> the
   * output stream has been written to. Otherwise the body would need to be buffered!
   *
   * <p>For non-streaming requests with a body, headers must be prepared <strong>after</strong> the
   * output stream has been written to and closed. This ensures that the {@code Content-Length}
   * header field receives the proper value.
   */
  @Override public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());
    writeRequest(request.headers(), requestLine);
  }

  /** Returns bytes of a request header for sending on an HTTP transport. */
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
    state = STATE_OPEN_REQUEST_BODY;
  }
```

本方法主要是写请求头
 执行逻辑很清晰，可以分成两部分，对应Http协议，即写入请求行和请求头，至于RequestLine(请求行)里面的代码比较简单就是用StringBuilder进行拼接。

然后看下写入请求体



```java
  @Override public Sink createRequestBody(Request request, long contentLength) {
    if ("chunked".equalsIgnoreCase(request.header("Transfer-Encoding"))) {
      // Stream a request body of unknown length.
      return newChunkedSink();
    }

    if (contentLength != -1) {
      // Stream a request body of a known length.
      return newFixedLengthSink(contentLength);
    }

    throw new IllegalStateException(
        "Cannot stream a request body without chunked encoding or a known content length!");
  }
```

熟悉HTTP协议的同学都知道其实请求和响应体可以分为固定长度和非固定长度的两种，其中非固定长度由头部信息中的Transfer-Encoding=chunked来表示，固定长度则由对应的头部信息表示实体信息的对应长度。



```java
  public Sink newChunkedSink() {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new ChunkedSink();
  }

/**
   * An HTTP body with alternating chunk sizes and chunk bodies. It is the caller's responsibility
   * to buffer chunks; typically by using a buffered sink with this sink.
   */
  private final class ChunkedSink implements Sink {
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
```

这里使用一个内部类ChunkedSink来封装sink，这里我们只看到其中的三个重要方法，即write()、flush()、close()方法，逻辑很清晰，非固定长度的请求，都是在第一行写入一段数据的长度，然后在之后写入该段数据。从write()方法中可以看出将buffer中的数据写入到sink对象中，如果熟悉okio的执行逻辑，对此应该很容易理解。然后刷新和关闭逻辑很简单，其中关闭时注意更新状态。

对于固定长度的请求体，其封装的sink逻辑是类似的，其中需要传入一个bytesRemaining，保证写数据结束时保证数据长度是正确的。



```java
  public Sink newFixedLengthSink(long contentLength) {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new FixedLengthSink(contentLength);
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
```

PS:在读取响应体的时候也会用到这个类。
 这里可以看到有一个成员变量bytesRemaining标示剩余的字节数，保证读取到的字节长度与头部信息中的长度保证一致。read()中的代码可以看到就是将该source对象的数据读取到封装的source中，用于构建resposne。
 PS:sink,source对象以及封装的sink和source对象，代表http1Codec中的sink和source对象。即封装了socket的输出和输入流，而封装的sink和source对象则是构建的固定长度和非固定长度的输出输入流，其实它们只是对http1Codec成员变量的·中的sink和source的一种封装，其实就是装饰者模式，封装以后的sink和source对象可以用在外部请求和构建ReponseBody。

当写完请求头和请求体之后，需要完成，这时候会调用finishReqeust()方法



```java
  @Override public void finishRequest() throws IOException {
    sink.flush();
  }
```

其实这一步其实很简单，只有一行代码，就是执行流的刷新。
 注意这一步是不需要检查状态的，因为此时的状态有可能是STATE_OPEN_REQUEST_BODY(没有请求体的情况)或者STATE_READ_RESPONSE_HEADERS(已经完成请求体写入的情况)。这一步只是刷新新流，所以什么情况都不会造成影响，所以没有必要检查状态，也没有更新状态，保持之前的状态即可。

再看下读取请求头的readResponseHeaders()方法



```java
  @Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
      throw new IllegalStateException("state: " + state);
    }

    try {
      StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());

      Response.Builder responseBuilder = new Response.Builder()
          .protocol(statusLine.protocol)
          .code(statusLine.code)
          .message(statusLine.message)
          .headers(readHeaders());

      if (expectContinue && statusLine.code == HTTP_CONTINUE) {
        return null;
      }

      state = STATE_OPEN_RESPONSE_BODY;
      return responseBuilder;
    } catch (EOFException e) {
      // Provide more context if the server ends the stream before sending a response.
      IOException exception = new IOException("unexpected end of stream on " + streamAllocation);
      exception.initCause(e);
      throw exception;
    }
  }
```

此时所处的状态有可能为STATE_OPEN_REQUEST_BODY和STATE_READ_RESPONSE_HEADERS两种，然后读取请求航和请求头部信息，并返回响应的Builder。

然后再看下读取请求头体的方法再看下读取响应体的方法



```java
  @Override public ResponseBody openResponseBody(Response response) throws IOException {
    Source source = getTransferStream(response);
    return new RealResponseBody(response.headers(), Okio.buffer(source));
  }
```

在之前介绍中，我们知道响应Response对象是封装一个source对象，用于读取响应数据。所以ResponseBody的构建就是需要响应头和响应体的两部分即可，响应头在上一部分中已经添加到response对象中了，headers()获取响应头即可。下面分析，如何封装source对象，获取一个对应的source对象，可能有些拗口，让我们来看下getTransferStream()方法的代码：



```kotlin
  private Source getTransferStream(Response response) throws IOException {
    if (!HttpHeaders.hasBody(response)) {
      return newFixedLengthSource(0);
    }

    if ("chunked".equalsIgnoreCase(response.header("Transfer-Encoding"))) {
      return newChunkedSource(response.request().url());
    }

    long contentLength = HttpHeaders.contentLength(response);
    if (contentLength != -1) {
      return newFixedLengthSource(contentLength);
    }

    // Wrap the input stream from the connection (rather than just returning
    // "socketIn" directly here), so that we can control its use after the
    // reference escapes.
    return newUnknownLengthSource();
  }
```

这里和写入请求体的地方十分类似，响应体也是分为固定长度和非固定长度两种，除此以外，为了代码的健壮性okhttp还定义了UnknownLengthSource(位置长度Source，代表意外情况)。FixedLengthSource已经分析过了，这里就不详细描述了。

这里需要提及一下endOfInput()方法：



```java
    /**
     * Closes the cache entry and makes the socket available for reuse. This should be invoked when
     * the end of the body has been reached.
     */
    protected final void endOfInput(boolean reuseConnection) throws IOException {
      if (state == STATE_CLOSED) return;
      if (state != STATE_READING_RESPONSE_BODY) throw new IllegalStateException("state: " + state);

      detachTimeout(timeout);

      state = STATE_CLOSED;
      if (streamAllocation != null) {
        streamAllocation.streamFinished(!reuseConnection, Http1Codec.this);
      }
    }
  }
```

这里的执行逻辑也简单，除去检查状态和更新状态之外，就是接触超时机制，最后需要注意就是调用streamAllocation的streamFinish()方法
 这里的执行逻辑也不多，除去检查状态和更新状态之外，就是接触超时机制，最后需要注意就是调用streamAllocation的

这里我们简单介绍下Http2Codec。先看下他的属性



```php
  //这里是header
  private static final ByteString CONNECTION = ByteString.encodeUtf8("connection");
  private static final ByteString HOST = ByteString.encodeUtf8("host");
  private static final ByteString KEEP_ALIVE = ByteString.encodeUtf8("keep-alive");
  private static final ByteString PROXY_CONNECTION = ByteString.encodeUtf8("proxy-connection");
  private static final ByteString TRANSFER_ENCODING = ByteString.encodeUtf8("transfer-encoding");
  private static final ByteString TE = ByteString.encodeUtf8("te");
  private static final ByteString ENCODING = ByteString.encodeUtf8("encoding");
  private static final ByteString UPGRADE = ByteString.encodeUtf8("upgrade");

  /** See http://tools.ietf.org/html/draft-ietf-httpbis-http2-09#section-8.1.3. */
   //request里面的header组合
  private static final List<ByteString> HTTP_2_SKIPPED_REQUEST_HEADERS = Util.immutableList(
      CONNECTION,
      HOST,
      KEEP_ALIVE,
      PROXY_CONNECTION,
      TE,
      TRANSFER_ENCODING,
      ENCODING,
      UPGRADE,
      TARGET_METHOD,
      TARGET_PATH,
      TARGET_SCHEME,
      TARGET_AUTHORITY);
   //response 里面的的header组合
  private static final List<ByteString> HTTP_2_SKIPPED_RESPONSE_HEADERS = Util.immutableList(
      CONNECTION,
      HOST,
      KEEP_ALIVE,
      PROXY_CONNECTION,
      TE,
      TRANSFER_ENCODING,
      ENCODING,
      UPGRADE);

  private final OkHttpClient client;
  //负责连接、请求、流的分配
  final StreamAllocation streamAllocation;
  //HTTP/2.0的连接
  private final Http2Connection connection;
  //HTTP/2.0的流
  private Http2Stream stream;
```

这里说下关于HttpCodec 面向接口编程，在外层不关系具体的实现类，外层会调用对应的抽象方法来实现它的逻辑，而内在的实现类再根据情况去实现。
 其中Http2Connection代表着HTTP/2.0的连接，Http2Stream代表着HTTP/2.0的流。
 这里说下，由于HTTP/2 里面支持一个"连接"可以发送多个请求，所以和HTTP/1.x有着本质的区别，所以Http1Codec里面有source和sink，而Http2Codec没有，因为在HTTP/1.x里面一个连接对应一个请求。而HTTP2则不是，一个TCP连接上可以跑多个请求。所以OkHttp里面用一个Http2Connection代表一个连接。然后用Http2Stream代表一个请求的流。

## 四、[AndroidPlatform](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/platform/AndroidPlatform.java)

大家看代码的时候经常会遇到Platform.get()方法。这个方法返回的不是Plarform对象，返回的是AndroidPlatform对象
 那么这里简单介绍下其对应的方法



```java
  @Override public boolean isCleartextTrafficPermitted(String hostname) {
    try {
      Class<?> networkPolicyClass = Class.forName("android.security.NetworkSecurityPolicy");
      Method getInstanceMethod = networkPolicyClass.getMethod("getInstance");
      Object networkSecurityPolicy = getInstanceMethod.invoke(null);
      Method isCleartextTrafficPermittedMethod = networkPolicyClass
          .getMethod("isCleartextTrafficPermitted", String.class);
      return (boolean) isCleartextTrafficPermittedMethod.invoke(networkSecurityPolicy, hostname);
    } catch (ClassNotFoundException | NoSuchMethodException e) {
      return super.isCleartextTrafficPermitted(hostname);
    } catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
      throw new AssertionError();
    }
  }
```

在RealConnection.connect()方法里面被调用
 平台的安全策略：安卓平台本身的安全策略允许向相应的主机送法明文请求。对于Android平台而言，这种安全策略主要由系统组件android.security.NetworkSecurityPolicy执行，平台这种安全策略并不是每个Android版本都有，6.0之后才有有这种权限控制
 平台本身的安全策略允许向相应的主机发送明文请求。对于Android平台而言，这种安全策略主要由系统的组件。

再来看下另外一个方法的调用



```dart
  @Override public void configureTlsExtensions(
      SSLSocket sslSocket, String hostname, List<Protocol> protocols) {
    // Enable SNI and session tickets.
    if (hostname != null) {
      setUseSessionTickets.invokeOptionalWithoutCheckedException(sslSocket, true);
      setHostname.invokeOptionalWithoutCheckedException(sslSocket, hostname);
    }

    // Enable ALPN.
    if (setAlpnProtocols != null && setAlpnProtocols.isSupported(sslSocket)) {
      Object[] parameters = {concatLengthPrefixed(protocols)};
      setAlpnProtocols.invokeWithoutCheckedException(sslSocket, parameters);
    }
  }
```

在RealConnection里面调用connectTls()方法里面调用了
 Platform.get().configureTlsExtensions()方法
 这个方法主要利用了TLS的ALPN扩展来完成的。这里再来详细的看一下配置TLS扩展的过程。
 TLS扩展相关的方法不是SSLSocket接口的标准方法，不同是SSL/
 TLS实现对这些接口的支持程度不一样，因而这里荣国反射机制调用TLS扩展相关方法。
 这里主要配置了3个TLS扩展，分别是session tickes、SNI和ALPN、session tickets用于会话恢复，SNI用于支持单个主机配置多个域名的情况。ALPN则用于HTTP/2的协议协商。可以到位SNI设置的hostname最终来源于Url,也就意味着使用HttpDns时，如果没有直接将IP地址替换成Url中的域名来发起HTTPS请求的话，SNI将是IP地址，这里可能使服务器下发不恰当的证书。

TLS扩展相关方法的OptionaMethod创建过程也在AndroidPlatform中：



```kotlin
 public AndroidPlatform(Class<?> sslParametersClass, OptionalMethod<Socket> setUseSessionTickets,
      OptionalMethod<Socket> setHostname, OptionalMethod<Socket> getAlpnSelectedProtocol,
      OptionalMethod<Socket> setAlpnProtocols) {
    this.sslParametersClass = sslParametersClass;
    this.setUseSessionTickets = setUseSessionTickets;
    this.setHostname = setHostname;
    this.getAlpnSelectedProtocol = getAlpnSelectedProtocol;
    this.setAlpnProtocols = setAlpnProtocols;
  }

public static Platform buildIfSupported() {
    // Attempt to find Android 2.3+ APIs.
    try {
      Class<?> sslParametersClass;
      try {
        sslParametersClass = Class.forName("com.android.org.conscrypt.SSLParametersImpl");
      } catch (ClassNotFoundException e) {
        // Older platform before being unbundled.
        sslParametersClass = Class.forName(
            "org.apache.harmony.xnet.provider.jsse.SSLParametersImpl");
      }

      OptionalMethod<Socket> setUseSessionTickets = new OptionalMethod<>(
          null, "setUseSessionTickets", boolean.class);
      OptionalMethod<Socket> setHostname = new OptionalMethod<>(
          null, "setHostname", String.class);
      OptionalMethod<Socket> getAlpnSelectedProtocol = null;
      OptionalMethod<Socket> setAlpnProtocols = null;

      // Attempt to find Android 5.0+ APIs.
      try {
        Class.forName("android.net.Network"); // Arbitrary class added in Android 5.0.
        getAlpnSelectedProtocol = new OptionalMethod<>(byte[].class, "getAlpnSelectedProtocol");
        setAlpnProtocols = new OptionalMethod<>(null, "setAlpnProtocols", byte[].class);
      } catch (ClassNotFoundException ignored) {
      }

      return new AndroidPlatform(sslParametersClass, setUseSessionTickets, setHostname,
          getAlpnSelectedProtocol, setAlpnProtocols);
    } catch (ClassNotFoundException ignored) {
      // This isn't an Android runtime.
    }

    return null;
  }
```

建立TLS连接的步骤中，获取协议的过程与配置TLS的过程类似，同样利用反射调用SSLSocket方法。
 RealConnection.connectTls()调用Platform.get().getSelectedProtocol(sslSocket)



```java
  @Override public String getSelectedProtocol(SSLSocket socket) {
    if (getAlpnSelectedProtocol == null) return null;
    if (!getAlpnSelectedProtocol.isSupported(socket)) return null;

    byte[] alpnResult = (byte[]) getAlpnSelectedProtocol.invokeWithoutCheckedException(socket);
    return alpnResult != null ? new String(alpnResult, Util.UTF_8) : null;
  }
```

## 五、[Connection](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/Connection.java)类

Connection是个接口，里面四个抽象方法



```cpp
 Route route(); //返回一个路由
 Socket socket();  //返回一个socket
 Handshake handshake();  //如果是一个https,则返回一个TLS握手协议
Protocol protocol(); //返回一个协议类型 比如 http1.1 等或者自定义类型 
```

注意：
 大家看下这个类的注释，上面说了，这个类和HttpURLConnection不是一个东西，Connection这个类不是一个单一的请求/响应交换的连接，可用于多个HTTP请求/响应交换
 其中有一句比较重要



```dart
Each connection can carry a varying number streams, depending on the underlying protocol being used. HTTP/1.x connections can carry either zero or one streams. HTTP/2 connections can carry any number of streams, dynamically configured with {@code SETTINGS_MAX_CONCURRENT_STREAMS}. A connection currently carrying zero streams is an idle stream. We keep it alive because reusing an existing connection is typically faster than establishing a new one.
```

简单翻译一下就是：
 每个连接可以携带不同数量的流，这取决于所使用的底层协议。 HTTP / 1.x连接可以携带零个或一个流。 HTTP / 2连接可以携带任意数量的流，使用{@code SETTINGS_MAX_CONCURRENT_STREAMS}动态配置。 目前携带零流的连接是空闲流。 我们保持活着，因为重用现有的连接通常比建立新的连接更快。



```kotlin
When a single logical call requires multiple streams due to redirects or authorization challenges, we prefer to use the same physical connection for all streams in the sequence. There are potential performance and behavior consequences to this preference. To support this feature, this class separates <i>allocations</i> from <i>streams</i>. An allocation is created by a call, used for one or more streams, and then released. An allocated connection won't be stolen by other calls while a redirect or authorization challenge is being handled.
```

简单翻译一下就是：
 当一个请求被重定向或者证书验证的时候，需要多个流。为了拥有更好的性能，我们更愿意为序列中的所有流使用相同的物理连接。为了支持此功能，此类将”流“和"分配"分开。 分配由呼叫创建，用于一个或多个流，然后释放。 在处理重定向或授权挑战时，分配的连接不会被其他呼叫所窃取。

有人问，为什么要看这段注释，因为这段注释其实就是okhttp的复用连接池的精神，为后面复用连接池的时候做预热。

因为Connection是接口，他的具体实现类是RealConnection,其实大家可以发现OKHttp的代码风格是先写一个InterfaceA，然后具体的实现类是RealInterfaceA.



# OKHttp源码解析(九):OKHTTP连接中三个"核心"RealConnection、ConnectionPool、StreamAllocation

## 一、[RealConnection](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/connection/RealConnection.java)

RealConnection是Connection的实现类，代表着链接socket的链路，如果拥有了一个RealConnection就代表了我们已经跟服务器有了一条通信链路，而且通过
 RealConnection代表是连接socket链路，RealConnection对象意味着我们已经跟服务端有了一条通信链路了。很多朋友这时候会想到，有通信链路了，是不是与意味着在这个类实现的三次握手，你们猜对了，的确是在这个类里面实现的三次握手。在讲握手的之前，看下它的属性和构造函数，对他有个大概的了解。



```php
  private final ConnectionPool connectionPool;
  private final Route route;

  // The fields below are initialized by connect() and never reassigned.
  //下面这些字段，通过connect()方法开始初始化，并且绝对不会再次赋值
  /** The low-level TCP socket. */
  private Socket rawSocket; //底层socket
  /**
   * The application layer socket. Either an {@link SSLSocket} layered over {@link #rawSocket}, or
   * {@link #rawSocket} itself if this connection does not use SSL.
   */
  private Socket socket;  //应用层socket
  //握手
  private Handshake handshake;
   //协议
  private Protocol protocol;
   // http2的链接
  private Http2Connection http2Connection;
  //通过source和sink，大家可以猜到是与服务器交互的输入输出流
  private BufferedSource source;
  private BufferedSink sink;

  // The fields below track connection state and are guarded by connectionPool.
  //下面这个字段是 属于表示链接状态的字段，并且有connectPool统一管理
  /** If true, no new streams can be created on this connection. Once true this is always true. */
  //如果noNewStreams被设为true，则noNewStreams一直为true，不会被改变，并且表示这个链接不会再创新的stream流
  public boolean noNewStreams;
  
  //成功的次数
  public int successCount;

  /**
   * The maximum number of concurrent streams that can be carried by this connection. If {@code
   * allocations.size() < allocationLimit} then new streams can be created on this connection.
   */
  //此链接可以承载最大并发流的限制，如果不超过限制，可以随意增加
  public int allocationLimit = 1;
```

通过上面代码，我们可以得出以下结论：

> 

1、里面除了route 字段，部分的字段都是在connect()方法里面赋值的，并且不会再次赋值
 2、这里含有source和sink，所以可以以流的形式对服务器进行交互
 3、noNewStream可以简单理解为它表示该连接不可用。这个值一旦被设为true,则这个conncetion则不会再创建stream。
 4、allocationLimit是分配流的数量上限，一个connection最大只能支持一个1并发
 5、allocations是关联StreamAllocation,它用来统计在一个连接上建立了哪些流，通过StreamAllocation的acquire方法和release方法可以将一个allcation对方添加到链表或者移除链表，

其实大家估计已经猜到了connect()里面进行了三次握手，大家也猜对了，那咱们就简单的介绍下connect()方法：



```java
public void connect( int connectTimeout, int readTimeout, int writeTimeout, boolean connectionRetryEnabled) {
    if (protocol != null) throw new IllegalStateException("already connected");
     // 线路的选择
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
    }
    // 连接开始
    while (true) {
      try {
        // 如果要求隧道模式，建立通道连接，通常不是这种
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout);
        } else {
           // 一般都走这条逻辑了，实际上很简单就是socket的连接
          connectSocket(connectTimeout, readTimeout);
        }
        // https的建立
        establishProtocol(connectionSpecSelector);
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

    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }
```

这里的执行过程大体如下；

> 

- 1、检查连接是否已经建立，若已经建立，则抛出异常，否则继续，连接的是否简历由protocol标示，它表示在整个连接建立，及可能的协商过程中选择所有要用到的协议。
- 2、用 集合connnectionspecs构造ConnectionSpecSelector。
- 3、如果请求是不安全的请求，会对请求执行一些额外的限制:
   3.1、ConnectionSpec集合必须包含ConnectionSpec.CLEARTEXT。也就是说OkHttp用户可以通过OkHttpClient设置不包含ConnectionSpec.CLEARTEXT的ConnectionSpec集合来禁用所有的明文要求。
   3.2、平台本身的安全策略允向相应的主机发送明文请求。对于Android平台而言，这种安全策略主要由系统的组件android.security.NetworkSecurityPolicy执行。平台的这种安全策略不是每个Android版本都有的。Android6.0之后存在这种控制。
   (okhttp/okhttp/src/main/java/okhttp3/internal/platform/AndroidPlatform.java 里面的isCleartextTrafficPermitted()方法)
- 4、根据请求判断是否需要建立隧道连接，如果建立隧道连接则调用
   connectTunnel(connectTimeout, readTimeout, writeTimeout);
- 5、如果不是隧道连接则调用connectSocket(connectTimeout, readTimeout);建立普通连接。
- 6、通过调用establishProtocol建立协议
- 7、如果是HTTP/2，则设置相关属性。

整个流程已经梳理完，咱们就抠一下具体的细节，首先来看下建立普通连接，因为隧道连接也会用到普通连接的代码：
 看下connectSocket()方法



```csharp
/** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();
     // 根据代理类型来选择socket类型，是代理还是直连
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    rawSocket.setSoTimeout(readTimeout);
    try {
      // 连接socket，之所以这样写是因为支持不同的平台
      //里面实际上是  socket.connect(address, connectTimeout);
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }
     // 得到输入／输出流
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));
  }
```

有3种情况需要建立普通连接：

> 

- 无代理
- 明文的HTTP代理
- SOCKS代理

普通连接的建立过程为建立TCP连接，建立TCP连接的过程为：

> 

- 1、创建Socket，非SOCKS代理的情况下，通过SocketFactory创建；在SOCKS代理则传入proxy手动new一个出来。
- 2、为Socket设置超时
- 3、完成特定于平台的连接建立
- 4、创建用于I/O的source和sink

下面我来看下connectSocket()的具体实现，connectSocket()具体实现是AndroidPlatform.java里面的connectSocket()。
 关于AndroidPlatform.java请看上一篇文章。

设置了SOCKS代理的情况下，仅有的特别之处在于，是通过传入proxy手动创建Socket。route的socketAddress包含目标HTTP服务器的域名。由此可见SOCKS协议的处理，主要是在Java标准库的java.net.Socket中处理，对于外界而言，就好像是HTTP服务器直接建立连接一样，因此连接时传入的地址都是HTTP服务器的域名。

而对于明文的HTTP代理的情况下，这里灭有任何特殊处理。route的socketAddress包含着代理服务器的IP地址。HTTP代理自身会根据请求及相应的实际内容，建立与HTTP服务器的TCP连接，并转发数据。

这时候我们再来看下建立隧道逻辑：



```csharp
  /**
   * Does all the work to build an HTTPS connection over a proxy tunnel. The catch here is that a
   * proxy server can issue an auth challenge and then close the connection.
   */
  private void connectTunnel(int connectTimeout, int readTimeout, int writeTimeout)
      throws IOException {
    Request tunnelRequest = createTunnelRequest();
    HttpUrl url = tunnelRequest.url();
    int attemptedConnections = 0;
    int maxAttempts = 21;
    while (true) {
      if (++attemptedConnections > maxAttempts) {
        throw new ProtocolException("Too many tunnel connections attempted: " + maxAttempts);
      }

      connectSocket(connectTimeout, readTimeout);
      tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url);

      if (tunnelRequest == null) break; // Tunnel successfully created.

      // The proxy decided to close the connection after an auth challenge. We need to create a new
      // connection, but this time with the auth credentials.
      closeQuietly(rawSocket);
      rawSocket = null;
      sink = null;
      source = null;
    }
  }
```

建立隧道连接的过程又分为几个步骤：

> 

- 创建隧道请求
- 建立Socket连接
- 发送请求建立隧道

隧道请求是一个常规的HTTP请求，只是请求的内容有点特殊。最初创建的隧道请求如：



```cpp
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
```

一个隧道请求的例子如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-bfeda5ac4bb02aca.png?imageMogr2/auto-orient/strip|imageView2/2/w/633/format/webp)

隧道.png



请求的"Host" header中包含了目标HTTP服务器的域名。建立socket连接的过程这里就不细说了

创建隧道的过程是这样子的：



```csharp
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
```

在前面创建的TCP连接值上，完成代理服务器的HTTP请求/响应交互。请求的内容类似下面这样：



```bash
"CONNECT m.taobao.com:443 HTTP/1.1"
```

这里可能会根据HTTP代理是否需要认证而有多次HTTP请求/响应交互。
 总结一下OkHttp3中代理相关的处理；

> 

- 1、没有设置代理的情况下，直接与HTTP服务器建立TCP连接，然后进行HTTP请求/响应的交互。
- 2、设置了SOCKS代理的情况下，创建Socket时，为其传入proxy，连接时还是以HTTP服务器为目标。在标准库的Socket中完成SOCKS协议相关的处理。此时基本上感知不到代理的存在。
- 3、设置了HTTP代理时的HTTP请求，与HTTP代理服务器建立TCP连接。HTTP代理服务器解析HTTP请求/响应的内容，并根据其中的信息来完成数据的转发。也就是说，如果HTTP请求中不包含"Host"header，则有可能在设置了HTTP代理的情况下无法与HTTP服务器建立连接。
- 4、HTTP代理时的HTTPS/HTTP2请求，与HTTP服务器建立通过HTTP代理的隧道连接。HTTP代理不再解析传输的数据，仅仅完成数据转发的功能。此时HTTP代理的功能退化为如同SOCKS代理类似。
- 5、设置了代理类时，HTTP的服务器的域名解析会交给代理服务器执行。其中设置了HTTP代理时，会对HTTP代理的域名做域名解析。

上述流程弄明白后，来看下建立协议
 不管是建立隧道连接，还是建立普通连接，都少不了建立协议这一步骤，这一步是建立好TCP连接之后，而在该TCP能被拿来手法数据之前执行的。它主要为了数据的加密传输做一些初始化，比如TCL握手，HTTP/2的协商。



```java
  private void establishProtocol(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    //如果不是ssl
    if (route.address().sslSocketFactory() == null) {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
      return;
    }
    //如果是sll
    connectTls(connectionSpecSelector);
   //如果是HTTP2
    if (protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
      http2Connection = new Http2Connection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .listener(this)
          .build();
      http2Connection.start();
    }
  }
```

上面的代码大体上可以归纳为两点

> 

- 1、对于加密的数据传输，创建TLS连接。对于明文传输，则设置protocol和socket。socket直接指向应用层，如HTTP或HTTP/2,交互的Socket。
   1.1对于明文传输没有设置HTTP代理的HTTP请求，它是与HTTP服务器之间的TCP socket。
   1.2对于加密传输没有设置HTTP代理服务器的HTTP或HTTP2请求，它是与HTTP服务器之间的SSLSocket。
   1.3对于加密传输设置了HTTP代理服务器的HTTP或HTTP2请求，它是与HTTP服务器之间经过代理服务器的SSLSocket，一个隧道连接；
   1.4对于加密传输设置了SOCKS代理的HTTP或HTTP2请求，它是一条经过了代理服务器的SSLSocket连接。
- 2、对于HTTP/2，通过new 一个Http2Connection.Builder会建立HTTP/2连接 Http2Connection，然后执行http2Connection.start()和服务器建立协议。我们先来看下建立TLS连接的connectTls()方法



```csharp
 private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    Address address = route.address();
    SSLSocketFactory sslSocketFactory = address.sslSocketFactory();
    boolean success = false;
    SSLSocket sslSocket = null;
    try {
      // Create the wrapper over the connected socket.
      //在原来的Socket加一层ssl
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
      Handshake unverifiedHandshake = Handshake.get(sslSocket.getSession());

      // Verify that the socket's certificates are acceptable for the target host.
      if (!address.hostnameVerifier().verify(address.url().host(), sslSocket.getSession())) {
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
```

TLS连接是对原始TCP连接的一个封装，以及听过TLS握手，及数据手法过程中的加解密等功能。在Java中，用SSLSocket来描述。上面建立的TLS连接的过程大体为：

> 

- 1、用SSLSocketFactory基于原始的TCP Socket，创建一个SSLSocket。
- 2、并配置SSLSocket。
- 3、在前面选择的ConnectionSpec支持TLS扩展参数时，配置TLS扩展参数。
- 4、启动TLS握手
- 5、TLS握手完成之后，获取证书信息。
- 6、对TLS握手过程中传回来的证书进行验证。
- 7、在前面选择的ConnectionSpec支持TLS扩展参数时，获取TLS握手过程中顺便完成的协议协商过程所选择的协议。这个过程主要用于HTTP/2的ALPN扩展。
- 8、OkHttp主要使用Okio来做IO操作，这里会基于前面获取到SSLSocket创建于执行的IO的BufferedSource和BufferedSink等，并保存握手信息以及所选择的协议。

至此连接已经建立连接已经结束了。

这里说一下isHealthy(boolean doExtensiveChecks)方法，入参是一个布尔类，表示是否需要额外的检查。这里主要是检查，判断这个连接是否是健康的连接，即是否可以重用。那我们来看下



```kotlin
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
```

看上述代码可知，同时满足如下条件才是健康的连接，否则返回false

> 

- 1、socket已经关闭
- 2、输入流关闭
- 3、输出流关闭
- 4、如果是HTTP/2连接，则HTTP/2连接也要关闭。

让我们再来看下isEligible(Address, Route)方法，这个方法主要是判断面对给出的addres和route，这个realConnetion是否可以重用。



```kotlin
/**
   * Returns true if this connection can carry a stream allocation to {@code address}. If non-null
   * {@code route} is the resolved route for a connection.
   */
  public boolean isEligible(Address address, Route route) {
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

判断逻辑如下：

> 

- 如果连接达到共享上限，则不能重用
- 非host域必须完全一样，如果不一样不能重用
- 如果此时host域也相同，则符合条件，可以被复用
- 如果host不相同，在HTTP/2的域名切片场景下一样可以复用
   关于HTTP/2的可以参考下面的文章
   [https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding](https://link.jianshu.com?t=https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding)

最后再来看下newCodec(OkHttpClient, StreamAllocation)方法



```csharp
    if (http2Connection != null) {
      return new Http2Codec(client, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(client.readTimeoutMillis());
      source.timeout().timeout(client.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(client.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
```

里面主要是判断是否是HTTP/2,如果是HTTP/2则new一个Http2Codec。如果不是HTTP/2则new一个Http1Codec。

上面提到了connection的跟踪状态由ConncetionPool来管理。

## 二、[ConnectionPool](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/ConnectionPool.java)

大家先来看下一个类的注释



```dart
/**
 * Manages reuse of HTTP and HTTP/2 connections for reduced network latency. HTTP requests that
 * share the same {@link Address} may share a {@link Connection}. This class implements the policy
 * of which connections to keep open for future use.
 */
```

简单的翻译下，如下：
 管理http和http/2的链接，以便减少网络请求延迟。同一个address将共享同一个connection。该类实现了复用连接的目标。

然后看下这个类的字段:



```java
/**
   * Background threads are used to cleanup expired connections. There will be at most a single
   * thread running per connection pool. The thread pool executor permits the pool itself to be
   * garbage collected.
   */
  //这是一个用于清楚过期链接的线程池，每个线程池最多只能运行一个线程，并且这个线程池允许被垃圾回收
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

  /** The maximum number of idle connections for each address. */
  //每个address的最大空闲连接数。
  private final int maxIdleConnections;
  private final long keepAliveDurationNs;
  //清理任务
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
  //链接的双向队列
  private final Deque<RealConnection> connections = new ArrayDeque<>();
  //路由的数据库
  final RouteDatabase routeDatabase = new RouteDatabase();
   //清理任务正在执行的标志
  boolean cleanupRunning;
```

来看下它的属性，

> 

- 1、主要就是connections，可见ConnectionPool内部以队列方式存储连接；
- 2、routDatabase是一个黑名单，用来记录不可用的route，但是看代码貌似ConnectionPool并没有使用它。所以此处不做分析。
- 3、剩下的就是和清理有关了，所以executor是清理任务的线程池，cleanupRunning是清理任务的标志，cleanupRunnable是清理任务。

再来看下他的构造函数



```cpp
  /**
   * Create a new connection pool with tuning parameters appropriate for a single-user application.
   * The tuning parameters in this pool are subject to change in future OkHttp releases. Currently
   * this pool holds up to 5 idle connections which will be evicted after 5 minutes of inactivity.
   */
 //创建一个适用于单个应用程序的新连接池。
 //该连接池的参数将在未来的okhttp中发生改变
 //目前最多可容乃5个空闲的连接，存活期是5分钟
  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    //保持活着的时间，否则清理将旋转循环
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }
```

通过这个构造器我们知道了这个连接池最多维持5个连接，且每个链接最多活5分钟。并且包含一个线程池包含一个清理任务。
 所以maxIdleConnections和keepAliveDurationNs则是清理中淘汰连接的的指标，这里需要说明的是maxIdleConnections是值每个地址上最大的空闲连接数。所以OkHttp只是限制与同一个远程服务器的空闲连接数量，对整体的空闲连接并没有限制。

PS:

> 

这时候说下ConnectionPool的实例化的过程，一个OkHttpClient只包含一个ConnectionPool，其实例化也是在OkHttpClient的过程。这里说一下ConnectionPool各个方法的调用并没有直接对外暴露，而是通过OkHttpClient的Internal接口统一对外暴露。

然后我们来看下他的get和put方法



```kotlin
  /**
   * Returns a recycled connection to {@code address}, or null if no such connection exists. The
   * route is null if the address has not yet been routed.
   */
  RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    //断言，判断线程是不是被自己锁住了
    assert (Thread.holdsLock(this));
    // 遍历已有连接集合
    for (RealConnection connection : connections) { 
       //如果connection和需求中的"地址"和"路由"匹配
      if (connection.isEligible(address, route)) {
       //复用这个连接
        streamAllocation.acquire(connection);
        //返回这个连接
        return connection;
      }
    }
    return null;
  }
```

get() 方法遍历 connections 中的所有 RealConnection 寻找同时满足条件的RealConnection。



```csharp
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```

put方法更为简单，就是异步触发清理任务，然后将连接添加到队列中。那么下面开始重点分析他的清理任务。



```java
  private final long keepAliveDurationNs;
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
```

这个逻辑也很简单，就是调用cleanup方法执行清理，并等待一段时间，持续清理，其中cleanup方法返回的值来来决定而等待的时间长度。那我们继续来看下cleanup函数：



```csharp
  /**
   * Performs maintenance on this pool, evicting the connection that has been idle the longest if
   * either it has exceeded the keep alive limit or the idle connections limit.
   *
   * <p>Returns the duration in nanos to sleep until the next scheduled call to this method. Returns
   * -1 if no further cleanups are required.
   */
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }
        //统计空闲连接数量
        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          //找出空闲时间最长的连接以及对应的空闲时间
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
       //在符合清理条件下，清理空闲时间最长的连接
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        //不符合清理条件，则返回下次需要执行清理的等待时间，也就是此连接即将到期的时间
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
       //没有空闲的连接，则隔keepAliveDuration(分钟)之后再次执行
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
       //清理结束
        cleanupRunning = false;
        return -1;
      }
    }
    //关闭socket资源
    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
     //这里是在清理一个空闲时间最长的连接以后会执行到这里，需要立即再次执行清理
    return 0;
  }

  
```

这里的首先统计空闲连接数量，然后通过for循环查找最长空闲时间的连接以及对应空闲时长，然后判断是否超出最大空闲连接数(maxIdleConnections)或者或者超过最大空闲时间(keepAliveDurationNs)，满足其一则清除最长空闲时长的连接。如果不满足清理条件，则返回一个对应等待时间。
 这个对应等待的时间又分二种情况：

> 

- 1 有连接则等待下次需要清理的时间去清理:keepAliveDurationNs-longestIdleDurationNs;
- 2 没有空闲的连接，则等下一个周期去清理：keepAliveDurationNs

如果清理完毕返回-1。
 综上所述，我们来梳理一下清理任务，清理任务就是异步执行的，遵循两个指标，最大空闲连接数量和最大空闲时长，满足其一则清理空闲时长最大的那个连接，然后循环执行，要么等待一段时间，要么继续清理下一个连接，知道清理所有连接，清理任务才结束，下一次put的时候，如果已经停止的清理任务则会被再次触发



```dart
/**
   * Prunes any leaked allocations and then returns the number of remaining live allocations on
   * {@code connection}. Allocations are leaked if the connection is tracking them but the
   * application code has abandoned them. Leak detection is imprecise and relies on garbage
   * collection.
   */
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
     //遍历弱引用列表
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);
       //若StreamAllocation被使用则接着循环
      if (reference.get() != null) {
        i++;
        continue;
      }

      // We've discovered a leaked allocation. This is an application bug.
      StreamAllocation.StreamAllocationReference streamAllocRef =
          (StreamAllocation.StreamAllocationReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, streamAllocRef.callStackTrace);
       //若StreamAllocation未被使用则移除引用，这边注释为泄露
      references.remove(i);
      connection.noNewStreams = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      //如果列表为空则说明此连接没有被引用了，则返回0，表示此连接是空闲连接
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }
    return references.size();
  }
```

pruneAndGetAllocationCount主要是用来标记泄露连接的。内部通过遍历传入进来的RealConnection的StreamAllocation列表，如果StreamAllocation被使用则接着遍历下一个StreamAllocation。如果StreamAllocation未被使用则从列表中移除，如果列表中为空则说明此连接连接没有引用了，返回0，表示此连接是空闲连接，否则就返回非0表示此连接是活跃连接。
 接下来让我看下ConnectionPool的connectionBecameIdle()方法，就是当有连接空闲时，唤起cleanup线程清洗连接池



```java
  /**
   * Notify this pool that {@code connection} has become idle. Returns true if the connection has
   * been removed from the pool and should be closed.
   */
  boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
     //该连接已经不可用
    if (connection.noNewStreams || maxIdleConnections == 0) {
      connections.remove(connection);
      return true;
    } else {
      //欢迎clean 线程
      notifyAll(); // Awake the cleanup thread: we may have exceeded the idle connection limit.
      return false;
    }
  }
```

connectionBecameIdle标示一个连接处于空闲状态，即没有流任务，那么久需要调用该方法，由ConnectionPool来决定是否需要清理该连接。
 再来看下deduplicate()方法



```kotlin
  /**
   * Replaces the connection held by {@code streamAllocation} with a shared connection if possible.
   * This recovers when multiple multiplexed connections are created concurrently.
   */
  Socket deduplicate(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, null)
          && connection.isMultiplexed()
          && connection != streamAllocation.connection()) {
        return streamAllocation.releaseAndAcquire(connection);
      }
    }
    return null;
  }
```

该方法主要是针对HTTP/2场景下多个多路复用连接清除的场景。如果是当前连接是HTTP/2，那么所有指向该站点的请求都应该基于同一个TCP连接。这个方法比较简单就不详细说了，再说下另外一个方法



```csharp
  /** Close and remove all idle connections in the pool. */
  public void evictAll() {
    List<RealConnection> evictedConnections = new ArrayList<>();
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();
        if (connection.allocations.isEmpty()) {
          connection.noNewStreams = true;
          evictedConnections.add(connection);
          i.remove();
        }
      }
    }
    for (RealConnection connection : evictedConnections) {
      closeQuietly(connection.socket());
    }
  }
```

该方法是删除所有空闲的连接，比较简单，不说了

## 三、 [StreamAllocation](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/connection/StreamAllocation.java)

这个类很重要，我们先来看下类的注释



```dart
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
```

在讲解这个类的时候不得不说下背景：
 HTTP的版本：
 HTTP的版本从最初的1.0版本，到后续的1.1版本，再到后续的google推出的SPDY,后来再推出2.0版本，http协议越来越完善。(ps:okhttp也是根据2.0和1.1/1.0作为区分，实现了两种连接机制)这里要说下http2.0和http1.0,1.1的主要区别，2.0解决了老版本(1.1和1.0)最重要两个问题：连接无法复用和head of line blocking (HOL)问题.2.0使用多路复用的技术，多个stream可以共用一个socket连接，每个tcp连接都是通过一个socket来完成的，socket对应一个host和port，如果有多个stream(也就是多个request)都是连接在一个host和port上，那么它们就可以共同使用同一个socket,这样做的好处就是可以减少TCP的一个三次握手的时间。在OKHttp里面，记录一次连接的是RealConnection，这个负责连接，在这个类里面用socket来连接，用HandShake来处理握手。

在讲解这个类的之前我们先熟悉3个概念：请求、连接、流。我们要明白HTTP通信执行网络"请求"需要在"连接"上建立一个新的"流",我们将StreamAllocation称之流的桥梁，它负责为一次"请求"寻找"连接"并建立"流"，从而完成远程通信。所以说StreamAllocation与"请求"、"连接"、"流"都有关。

从注释我们看到。Connection是建立在Socket之上的物流通信信道，而Stream则是代表逻辑的流，至于Call是对一次请求过程的封装。之前也说过一个Call可能会涉及多个流(比如重定向或者auth认证等情况)。所以我们想一下，如果StreamAllocation要想解决上述问题，需要两个步骤，一是寻找连接，二是获取流。所以StreamAllocation里面应该包含一个Stream(上文已经说到了，OKHttp里面的流是HttpCodec)；还应该包含连接Connection。如果想找到合适的刘姐，还需要一个连接池ConnectionPool属性。所以应该有一个获取流的方法在StreamAllocation里面是newStream()；找到合适的流的方法findConnection()；还应该有完成请求任务的之后finish()的方法来关闭流对象，还有终止和取消等方法，以及释放资源的方法。

1、那咱们先就看下他的属性



```java
  public final Address address;//地址
  private Route route; //路由
  private final ConnectionPool connectionPool;  //连接池
  private final Object callStackTrace; //日志

  // State guarded by connectionPool.
  private final RouteSelector routeSelector; //路由选择器
  private int refusedStreamCount;  //拒绝的次数
  private RealConnection connection;  //连接
  private boolean released;  //是否已经被释放
  private boolean canceled  //是否被取消了
```

看完属性，我们来看下构造函数



```kotlin
  public StreamAllocation(ConnectionPool connectionPool, Address address, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.routeSelector = new RouteSelector(address, routeDatabase());
    this.callStackTrace = callStackTrace;
  }
 
```

这时候我们再来看下他的一个比较重要的方法



```java
  public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    int connectTimeout = client.connectTimeoutMillis();
    int readTimeout = client.readTimeoutMillis();
    int writeTimeout = client.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      //获取一个连接
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
   //实例化HttpCodec,如果是HTTP/2则是Http2Codec否则是Http1Codec
      HttpCodec resultCodec = resultConnection.newCodec(client, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

这里面两个重要方法
 1是通过findHealthyConnection获取一个连接、2是通过resultConnection.newCodec获取流。
 我们接着来看findHealthyConnection()方法



```java
  /**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

我们看到里面调用findConnection来获取一个RealConnection，然后通过RealConnection自己的方法isHealthy，去判断是否是健康的连接，如果是健康的连接，则重用，否则就继续查找。那我们继续看下findConnection()方法



```java
 /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    Route selectedRoute;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");
      //获取存在的连接
      // Attempt to use an already-allocated connection.
      RealConnection allocatedConnection = this.connection;
      if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
          // 如果已经存在的连接满足要求，则使用已存在的连接
        return allocatedConnection;
      }
      //从缓存中去取
      // Attempt to get a connection from the pool.
      Internal.instance.get(connectionPool, address, this, null);
      if (connection != null) {
        return connection;
      }

      selectedRoute = route;
    }
       // 线路的选择，多ip的支持
    // If we need a route, make one. This is a blocking operation.
    if (selectedRoute == null) {
      //里面是个递归
      selectedRoute = routeSelector.next();
    }

    RealConnection result;
    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      // Now that we have an IP address, make another attempt at getting a connection from the pool.
      // This could match due to connection coalescing.
      //更换路由再次尝试
      Internal.instance.get(connectionPool, address, this, selectedRoute);
      if (connection != null) return connection;

      // Create a connection and assign it to this allocation immediately. This makes it possible
      // for an asynchronous cancel() to interrupt the handshake we're about to do.
      route = selectedRoute;
      refusedStreamCount = 0;
     // 以上都不符合，创建一个连接
      result = new RealConnection(connectionPool, selectedRoute);
      acquire(result);
    }
     //连接并握手
    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    //更新本地数据库
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      // Pool the connection.
      //把连接放到连接池中
      Internal.instance.put(connectionPool, result);
      //如果这个连接是多路复用
      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        //调用connectionPool的deduplicate方法去重。
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    //如果是重复的socket则关闭socket，不是则socket为nul，什么也不做
    closeQuietly(socket);
    //返回整个连接
    return result;
  }
```

上面代码大概的逻辑是：

> 

- 1、先找是否有已经存在的连接，如果有已经存在的连接，并且可以使用(!noNewStreams)则直接返回。
- 2、根据已知的address在connectionPool里面找，如果有连接，则返回
- 3、更换路由，更换线路，在connectionPool里面再次查找，如果有则返回。
- 4、如果以上条件都不满足则直接new一个RealConnection出来
- 5、new出来的RealConnection通过acquire关联到connection.allocations上
- 6、做去重判断，如果有重复的socket则关闭

里面涉及到的RealConnection的connect()方法，我们已经在RealConnection里面讲过，这里就不讲了。不过这里说下acquire()方法



```java
  /**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```

这里相当于给connection的引用计数器加1
 这里说下StreamAllocationReference，StreamAllocationReference其实是弱引用的子类。具体代码如下：



```dart
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
```

下面来看下他的他的其他方法streamFinished(boolean, HttpCodec)、release(RealConnection)和deallocate(boolean, boolean, boolean)方法。



```java
  public void streamFinished(boolean noNewStreams, HttpCodec codec) {
    Socket socket;
    synchronized (connectionPool) {
      if (codec == null || codec != this.codec) {
        throw new IllegalStateException("expected " + this.codec + " but was " + codec);
      }
      if (!noNewStreams) {
        connection.successCount++;
      }
      socket = deallocate(noNewStreams, false, true);
    }
    closeQuietly(socket);
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
```

其中deallocate(boolean, boolean, boolean)和release(RealConnection)方法都是private，而且均在streamFinished里面调用。
 release(RealConnection)方法比较简单，主要是把RealConnection对应的allocations清除掉，把计数器归零。
 deallocate(boolean, boolean, boolean)方法也简单，根据传入的三个布尔类型的值进行操作，如果streamFinished为true则代表关闭流，所以要通过连接池connectionPool把这个connection设置空闲连接，如果可以设为空闲连接则返回这个socket。不能则返回null。
 streamFinished()主要做了一些异常判断，然后调用deallocate()方法
 综上所述：streamFinished(boolean, HttpCodec)主要是关闭流，release(RealConnection)主要是释放connection的引用,deallocate(boolean, boolean, boolean)主要是根据参数做一些设置。
 上面说到了release(RealConnection)，为了防止大家混淆概念，这里说一下另外一个方法release()这个是无参的方法。



```java
  public void release() {
    Socket socket;
    synchronized (connectionPool) {
      socket = deallocate(false, true, false);
    }
    closeQuietly(socket);
  }
```

注意这个和上面的带有RealConnection的参数release()的区别。
 然后说一下noNewStreams()方法，主要是设置防止别人在这个连接上开新的流。



```cpp
  /** Forbid new streams from being created on the connection that hosts this allocation. */
  public void noNewStreams() {
    Socket socket;
    synchronized (connectionPool) {
      socket = deallocate(true, false, false);
    }
    closeQuietly(socket);
  }
```

还有一个方法，平时也是经常有遇到的就是cancel()方法



```java
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
```

其实也比较简单的就是调用RealConnection的Cancel方法。
 如果在连接中过程出现异常，会调用streamFailed(IOException)方法



```java
public void streamFailed(IOException e) {
    Socket socket;
    boolean noNewStreams = false;

    synchronized (connectionPool) {
      if (e instanceof StreamResetException) {
        StreamResetException streamResetException = (StreamResetException) e;
        if (streamResetException.errorCode == ErrorCode.REFUSED_STREAM) {
          refusedStreamCount++;
        }
        // On HTTP/2 stream errors, retry REFUSED_STREAM errors once on the same connection. All
        // other errors must be retried on a new connection.
        if (streamResetException.errorCode != ErrorCode.REFUSED_STREAM || refusedStreamCount > 1) {
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
      socket = deallocate(noNewStreams, false, true);
    }

    closeQuietly(socket);
  }
```

根据异常类型来采取不同的应对措施。注释已经比较清楚了，就不细说了。
 其他的方法比较简单，我这里就不细说了。



# OkHttp源码解析(十) OKHTTP中连接与请求及总结

终于到了讲解OkHttp中的连接与请求了，这部分内容主要是在ConnectInterceptor与CallServerInterceptor中，所以本片文章主要分2部分

> 

- 1、ConnectInterceptor
- 2、CallServerInterceptor
- 3、总结

## 一、[ConnectInterceptor](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/connection/ConnectInterceptor.java)

顾名思义连接拦截器，这才是真行的开始向服务器发起器连接。
 看下这个类的代码



```java
/** Opens a connection to the target server and proceeds to the next interceptor. */
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```

主要看下ConnectInterceptor()方法，里面代码已经很简单了，受限了通过streamAllocation的newStream方法获取一个流(HttpCodec 是个接口，根据协议的不同，由具体的子类的去实现)，第二步就是获取对应的RealConnection，由于在上一篇文章已经详细解释了RealConnection和streamAllocation类了，这里就不详细说了是大概聊一下

StreamAllocation的newStream()内部其实是通过findHealthyConnection()方法获取一个RealConnection，而在findHealthyConnection()里面通过一个while(true)死循环不断去调用findConnection()方法去找RealConnection.而在findConnection()里面其实是真正的寻找RealConnection，而上面提到的findHealthyConnection()里面主要就是调用findConnection()然后去验证是否是"健康"的。在findConnection()里面主要是通过3重判断：1如果有已知连接且可用，则直接返回，2如果在连接池有对应address的连接，则返回，3切换路由再在连接池里面找下，如果有则返回，如果上述三个条件都没有满足，则直接new一个RealConnection。然后开始握手，握手结束后，把连接加入连接池，如果在连接池有重复连接，和合并连接。
 至此findHealthyConnection()就分析完毕，给大家看下大缩减后的代码，如果大家想详细了解，请看上一篇文章。



```java
  //StreamAllocation.java
  public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    // 省略代码 
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, this);
    // 省略代码 
  }

  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }
      return candidate;
    }
  }

  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
      //省略部分代码
      //条件1如果有已知连接且可用，则直接返回
      RealConnection allocatedConnection = this.connection;
      if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
        return allocatedConnection;
      }

      //条件2 如果在连接池有对应address的连接，则返回
      Internal.instance.get(connectionPool, address, this, null);
      if (connection != null) {
        return connection;
      }

      selectedRoute = route;
    }

    // 条件3切换路由再在连接池里面找下，如果有则返回
    if (selectedRoute == null) {
      selectedRoute = routeSelector.next();
    }

    RealConnection result;
    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      Internal.instance.get(connectionPool, address, this, selectedRoute);
      if (connection != null) return connection;

      
      route = selectedRoute;
      refusedStreamCount = 0;
      //以上条件都不满足则new一个
      result = new RealConnection(connectionPool, selectedRoute);
      acquire(result);
    }

    // 开始握手
    result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    //计入数据库
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      //加入连接池
      Internal.instance.put(connectionPool, result);

      // 如果是多路复用，则合并
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);
    return result;
  }
```

这里再简单的说下RealConnection的connect()因为这个方法也很重要。不过大家要注意RealConnection的connect()是StreamAllocation调用的。在RealConnection的connect()的方法里面也是一个while(true)的循环，里面判断是隧道连接还是普通连接，如果是隧道连接就走connectTunnel()，如果是普通连接则走connectSocket()，最后建立协议。隧道连接这里就不介绍了，如果大家有兴趣就去上一篇文章去看。connectSocket()方噶里面就是通过okio获取source与sink。establishProtocol()方法建立连接咱们说下，里面判断是是HTTP/1.1还是HTTP/2.0。如果是HTTP/2.0则通过Builder来创建一个Http2Connection对象，并且调用Http2Connection对象的start()方法。所以判断一个RealConnection是否是HTTP/2.0其实很简单，判断RealConnection对象的http2Connection属性是否为null即可，因为只有HTTP/2的时候http2Connection才会被赋值。
 代码如下：



```java
public void connect(
      int connectTimeout, int readTimeout, int writeTimeout, boolean connectionRetryEnabled) {
   //省略部分代码
    while (true) {
      try {
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout);
        } else {
          connectSocket(connectTimeout, readTimeout);
        }
        establishProtocol(connectionSpecSelector);
        break;
      } catch (IOException e) {
          //省略部分代码
      }
    }

    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }

  private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
    //省略部分代码    
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));
  }

 private void establishProtocol(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    if (route.address().sslSocketFactory() == null) {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
      return;
    }

    connectTls(connectionSpecSelector);

    if (protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
      http2Connection = new Http2Connection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .listener(this)
          .build();
      http2Connection.start();
    }
  }
```

这时候我们在回来看下findConnection()方法里面的一行代码



```undefined
acquire(result)
```

调用的是acquire()方法



```csharp
  public void acquire(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```

代码简单，这里解释一下，每一个RealConnection对象都有一个字段即allocations



```php
public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
```

connections中维护了一张在一个连接上的流的链表。该链表保存的是StreamAllocation的引用。如果connections字段为空，则说明该连接可以被回收，如果不为空，说明被引用，不能被回收。所以OkHttp使用了类似计数法与标记擦出法的混合使用。当连接空闲或者释放的时候，StreamAllcocation的数量就会渐渐变成0。从而被线程池检测并回收。

至此StreamAllocation的findHealthyConnection()就分析完毕了。那我们来看下



```kotlin
//StreamAllocation.java
HttpCodec resultCodec = resultConnection.newCodec(client, this);
```

其实是调用RealConnection的newCodec()方法



```java
  public HttpCodec newCodec(
      OkHttpClient client, StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(client.readTimeoutMillis());
      source.timeout().timeout(client.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(client.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```

上面主要分了HTTP/2和HTTP/1.x，如果是HTTP/2(http2Connection不为null)则构建Http2Codec。如果是HTTP/1.x。则构建Http1Codec，大家注意一下在构建Http2Codec的时候并没有传入source和sink。这是为什么那？大家好好想一下，如果大家不知道为什么可以去看一下我前面的一篇介绍HTTP/2的文章，如果看了还不懂，请在下面留言，我给大家解释下。

至此关于ConnectInterceptor已经介绍完毕了。下面我们来介绍下CallServerInterceptor。最后一个Interceptor

## 二、[CallServerInterceptor](https://link.jianshu.com?t=https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http/CallServerInterceptor.java)

上面我们已经成功连接到服务器了，那接下来要做什么那？相信你已经猜到了， 那就说发送数据了。

在OkHttp里面读取数据主要是通过以下四个步骤来实现的

> 

- 1 写入请求头
- 2 写入请求体
- 3 读取响应头
- 4 读取响应体

OkHttp的流程是完全独立的。同样读写数据月是交给相关的类来处理，就是HttpCodec(解码器)来处理。



```java
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();
    
    long sentRequestMillis = System.currentTimeMillis();
    //写入请求头
    httpCodec.writeRequestHeaders(request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return what
      // we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        responseBuilder = httpCodec.readResponseHeaders(true);
      }
     //写入请求体
      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection from
        // being reused. Otherwise we're still obligated to transmit the request body to leave the
        // connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();
    //读取响应头
    if (responseBuilder == null) {
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
    //读取响应体
    int code = response.code();
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }
    return response;
  }
```

自此整个流程已经结束了。

那我们再来看下OkHttp网络请求的整体接口图(特别声明：这个图不是我画的)

![img](https:////upload-images.jianshu.io/upload_images/5713484-a89f541f0f7797a6.png?imageMogr2/auto-orient/strip|imageView2/2/w/721/format/webp)

okhttp整体架构.png

关于OkHttp就的解析马上就要结束了，最后我们再来温习一下整体的流程图



![img](https:////upload-images.jianshu.io/upload_images/5713484-35a9989e893171f1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1000/format/webp)

