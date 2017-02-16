---
title: OkHttp3 HTTP请求执行流程分析
date: 2016-11-15 17:46:49
categories: 网络协议
tags:
- 源码分析
- Android
- 网络协议
---

# OkHttp3的基本用法

使用OkHttp3发送Http请求并获得响应的过程大体为：
1. 创建OkHttpClient对象。OkHttpClient为网络请求执行的一个中心，它会管理连接池，缓存，SocketFactory，代理，各种超时时间，DNS，请求执行结果的分发等许多内容。
2. 创建Request对象。Request用于描述一个HTTP请求，比如请求的方法是"GET"还是"POST"，请求的URL，请求的header，请求的body，请求的缓存策略等。
3. 利用前面创建的OkHttpClient对象和Request对象创建Call对象。Call是一次HTTP请求的Task，它会执行网络请求以获得响应。OkHttp中的网络请求执行Call既可以同步进行，也可以异步进行。调用call.execute()将直接执行网络请求，阻塞直到获得响应。而调用call.enqueue()传入回调，则会将Call放入一个异步执行队列，由ExecutorService在后台执行。
4. 执行网络请求并获取响应。

<!--more-->

通过一段示例代码来看一下具体要如何操作：
```
    private void startRequestWithOkHttp3(String url) {
        //创建okHttpClient对象
        okhttp3.OkHttpClient mOkHttpClient = new okhttp3.OkHttpClient();
        //创建一个Request
        final okhttp3.Request request = new okhttp3.Request.Builder()
                .url(url)
                .build();
        //new call
        okhttp3.Call call = mOkHttpClient.newCall(request);
        //请求加入调度
        call.enqueue(new okhttp3.Callback() {
            @Override
            public void onFailure(okhttp3.Call call, final IOException e) {
                mTextScreen.post(new Runnable() {
                    @Override
                    public void run() {
                        mTextScreen.setText(e.toString());
                    }
                });
            }

            @Override
            public void onResponse(okhttp3.Call call, okhttp3.Response response) throws IOException {
                final String str = response.body().string();
                mTextScreen.post(new Runnable() {
                    @Override
                    public void run() {
                        mTextScreen.setText(str);
                    }
                });
            }
        });
    }
```
# Call的执行
(后面所有的代码分析都基于正式的发布版本3.4.0进行，'com.squareup.okhttp3:okhttp:3.4.0'，由于OkHttp当前依然处于比较活跃的开发状态，因而不同版本的内部实现相对于我们当前分析的这一版有可能会有比较大的变化。)

Call是一个接口，其定义如下：
```
package okhttp3;

import java.io.IOException;

/**
 * A call is a request that has been prepared for execution. A call can be canceled. As this object
 * represents a single request/response pair (stream), it cannot be executed twice.
 */
public interface Call {
  /** Returns the original request that initiated this call. */
  Request request();

  /**
   * Invokes the request immediately, and blocks until the response can be processed or is in
   * error.
   *
   * <p>The caller may read the response body with the response's {@link Response#body} method. To
   * avoid leaking resources callers must {@linkplain ResponseBody close the response body}.
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

  interface Factory {
    Call newCall(Request request);
  }
}
```
Call中还定义了一个Factory接口。

那在OkHttp中，我们调用的Call方法的实际执行过程是怎样的呢？这就需要扒出来在OkHttp中实际使用的Call实现了。OkHttpClient实现了Call.Factory接口，通过接口方法OkHttpClient.newCall()可以看到具体使用的Call实现是哪个类：
```
  /**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return new RealCall(this, request);
  }
```
在OkHttp中使用了RealCall来执行整个Http请求。
```
package okhttp3;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import okhttp3.internal.NamedRunnable;
import okhttp3.internal.cache.CacheInterceptor;
import okhttp3.internal.connection.ConnectInterceptor;
import okhttp3.internal.connection.StreamAllocation;
import okhttp3.internal.http.BridgeInterceptor;
import okhttp3.internal.http.CallServerInterceptor;
import okhttp3.internal.http.RealInterceptorChain;
import okhttp3.internal.http.RetryAndFollowUpInterceptor;
import okhttp3.internal.platform.Platform;

import static okhttp3.internal.platform.Platform.INFO;

final class RealCall implements Call {
  private final OkHttpClient client;
  private final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;

  // Guarded by this.
  private boolean executed;

  /** The application's original request unadulterated by redirects or auth headers. */
  Request originalRequest;

  protected RealCall(OkHttpClient client, Request originalRequest) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client);
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
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }

  synchronized void setForWebSocket() {
    if (executed) throw new IllegalStateException("Already Executed");
    this.retryAndFollowUpInterceptor.setForWebSocket(true);
  }

  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }

  @Override public void cancel() {
    retryAndFollowUpInterceptor.cancel();
  }

  @Override public synchronized boolean isExecuted() {
    return executed;
  }

  @Override public boolean isCanceled() {
    return retryAndFollowUpInterceptor.isCanceled();
  }

  StreamAllocation streamAllocation() {
    return retryAndFollowUpInterceptor.streamAllocation();
  }

  final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    private AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl().toString());
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

  /**
   * Returns a string that describes this call. Doesn't include a full URL as that might contain
   * sensitive information.
   */
  private String toLoggableString() {
    String string = retryAndFollowUpInterceptor.isCanceled() ? "canceled call" : "call";
    return string + " to " + redactedUrl();
  }

  HttpUrl redactedUrl() {
    return originalRequest.url().resolve("/...");
  }

  private Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!retryAndFollowUpInterceptor.isForWebSocket()) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(
        retryAndFollowUpInterceptor.isForWebSocket()));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
}
```
通过调用RealCall.execute()同步执行Http请求的过程大体为：
1. 调用client.dispatcher().executed(this)向client的dispatcher注册当前Call。
2. 调用getResponseWithInterceptorChain()执行网络请求并获得响应。
3. 调用client.dispatcher().finished(this)向client的dispatcher注销当前Call。

通过调用RealCall.enqueue()异步执行Http请求的过程则为，创建AsyncCall并将之丢给client的dispatcher。而在RealCall.AsyncCall的execute()中执行Http请求的过程与RealCall.execute()中的过程有些类似：
1. 调用getResponseWithInterceptorChain()执行网络请求并获得响应。
2. 调用Callback回调通知用户执行的结果。可以看到这里对回调接口是同步调用，也就是回调方法将在后台线程中被调用。
3. 调用client.dispatcher().finished(this)向client的dispatcher注销当前Call。

这里再通过Dispatcher的定义来看一下在OkHttp中，请求的执行管理及异步执行是怎么做的：
```
package okhttp3;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Deque;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import okhttp3.RealCall.AsyncCall;
import okhttp3.internal.Util;

/**
 * Policy on when async requests are executed.
 *
 * <p>Each dispatcher uses an {@link ExecutorService} to run calls internally. If you supply your
 * own executor, it should be able to run {@linkplain #getMaxRequests the configured maximum} number
 * of calls concurrently.
 */
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

  /**
   * Set the maximum number of requests to execute concurrently. Above this requests queue in
   * memory, waiting for the running calls to complete.
   *
   * <p>If more than {@code maxRequests} requests are in flight when this is invoked, those requests
   * will remain in flight.
   */
  public synchronized void setMaxRequests(int maxRequests) {
    if (maxRequests < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequests);
    }
    this.maxRequests = maxRequests;
    promoteCalls();
  }

  public synchronized int getMaxRequests() {
    return maxRequests;
  }

  /**
   * Set the maximum number of requests for each host to execute concurrently. This limits requests
   * by the URL's host name. Note that concurrent requests to a single IP address may still exceed
   * this limit: multiple hostnames may share an IP address or be routed through the same HTTP
   * proxy.
   *
   * <p>If more than {@code maxRequestsPerHost} requests are in flight when this is invoked, those
   * requests will remain in flight.
   */
  public synchronized void setMaxRequestsPerHost(int maxRequestsPerHost) {
    if (maxRequestsPerHost < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequestsPerHost);
    }
    this.maxRequestsPerHost = maxRequestsPerHost;
    promoteCalls();
  }

  public synchronized int getMaxRequestsPerHost() {
    return maxRequestsPerHost;
  }

  /**
   * Set a callback to be invoked each time the dispatcher becomes idle (when the number of running
   * calls returns to zero).
   *
   * <p>Note: The time at which a {@linkplain Call call} is considered idle is different depending
   * on whether it was run {@linkplain Call#enqueue(Callback) asynchronously} or
   * {@linkplain Call#execute() synchronously}. Asynchronous calls become idle after the
   * {@link Callback#onResponse onResponse} or {@link Callback#onFailure onFailure} callback has
   * returned. Synchronous calls become idle once {@link Call#execute() execute()} returns. This
   * means that if you are doing synchronous calls the network layer will not truly be idle until
   * every returned {@link Response} has been closed.
   */
  public synchronized void setIdleCallback(Runnable idleCallback) {
    this.idleCallback = idleCallback;
  }

  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }

  /**
   * Cancel all calls currently enqueued or executing. Includes calls executed both {@linkplain
   * Call#execute() synchronously} and {@linkplain Call#enqueue asynchronously}.
   */
  public synchronized void cancelAll() {
    for (AsyncCall call : readyAsyncCalls) {
      call.get().cancel();
    }

    for (AsyncCall call : runningAsyncCalls) {
      call.get().cancel();
    }

    for (RealCall call : runningSyncCalls) {
      call.cancel();
    }
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

  /** Returns the number of running calls that share a host with {@code call}. */
  private int runningCallsForHost(AsyncCall call) {
    int result = 0;
    for (AsyncCall c : runningAsyncCalls) {
      if (c.host().equals(call.host())) result++;
    }
    return result;
  }

  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }

  /** Returns a snapshot of the calls currently awaiting execution. */
  public synchronized List<Call> queuedCalls() {
    List<Call> result = new ArrayList<>();
    for (AsyncCall asyncCall : readyAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  /** Returns a snapshot of the calls currently being executed. */
  public synchronized List<Call> runningCalls() {
    List<Call> result = new ArrayList<>();
    result.addAll(runningSyncCalls);
    for (AsyncCall asyncCall : runningAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  public synchronized int queuedCallsCount() {
    return readyAsyncCalls.size();
  }

  public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
}
```
在Call的同步执行过程中，调用client.dispatcher().executed(this)向client的dispatcher注册当前Call，Dispatcher仅仅是将Call放进了runningSyncCalls，其它便什么也没做，目测同步执行Call时向Dispatcher注册的主要目的是方便全局性的cancel所有的Call。

Dispatcher中异步的AsyncCall是被放在一个ExecutorService中执行的。默认情况下，这是一个不限容量的线程池。但Dispatcher会限制每个host同时执行的最大请求数量，默认为5，同时也会限制同时执行的总的最大请求数量。runningAsyncCalls中保存所有正在被ExecutorService执行的AsyncCall，而readyAsyncCalls则用于存放由于对单个host同时执行的最大请求数量的限制，或总的同时执行最大请求数量的限制，而暂时得不到执行的AsyncCall。

finished()中，除了会将执行结束的AsyncCall从runningAsyncCalls移除之外，还会检查是否存在由于 单host同时进行的最大请求数量限制 或 总的同时执行最大请求数量限制，而暂时得不到执行的AsyncCall，若存在则满足限制条件的请求会被执行。

所有的Call，不管是异步的AsyncCall还是同步的Call在执行结束后都会检查是否没有正在进行的Http请求了。若没有了，则存在idle 回调时，该回调会被调用。

用户可以通过Dispatcher的构造函数来定制ExecutorService，这需要通过OkHttpClient.Builder在OkHttpClient的构建过程中间接的做到。

回到RealCall，继续来追Call的网络请求及响应处理。来看一下RealCall.getResponseWithInterceptorChain()：
```
  private Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!retryAndFollowUpInterceptor.isForWebSocket()) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(
        retryAndFollowUpInterceptor.isForWebSocket()));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```
这里主要是创建了一个Interceptor的列表，继而创建了一个Interceptor.Chain对象来处理请求并获得响应。我们继续追踪一下RealInterceptorChain：
```
package okhttp3.internal.http;

import java.io.IOException;
import java.util.List;
import okhttp3.Connection;
import okhttp3.HttpUrl;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.internal.connection.StreamAllocation;

/**
 * A concrete interceptor chain that carries the entire interceptor chain: all application
 * interceptors, the OkHttp core, all network interceptors, and finally the network caller.
 */
public final class RealInterceptorChain implements Interceptor.Chain {
  private final List<Interceptor> interceptors;
  private final StreamAllocation streamAllocation;
  private final HttpStream httpStream;
  private final Connection connection;
  private final int index;
  private final Request request;
  private int calls;

  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpStream httpStream, Connection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpStream = httpStream;
    this.index = index;
    this.request = request;
  }

  @Override public Connection connection() {
    return connection;
  }

  public StreamAllocation streamAllocation() {
    return streamAllocation;
  }

  public HttpStream httpStream() {
    return httpStream;
  }

  @Override public Request request() {
    return request;
  }

  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpStream, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpStream httpStream,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpStream != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpStream != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpStream, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpStream != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }

  private boolean sameConnection(HttpUrl url) {
    return url.host().equals(connection.route().address().url().host())
        && url.port() == connection.route().address().url().port();
  }
}
```
在RealInterceptorChain.proceed()中，除了对状态及获取的reponse做检查之外，最主要的事情即是构造新的RealInterceptorChain对象，获取对应Interceptor，并调用Interceptor的intercept(next)了。在这里，index充当迭代器或指示器的角色，用于指出当前正在处理的Interceptor。

RealInterceptorChain ＋ Interceptor实现了装饰器模式，实现了请求/响应的串式或流式处理。只不过内层装饰器不是外层装饰器的成员变量，而是接口方法中创建的临时变量。

但Interceptor链中具体都有哪些Interceptor呢？我们就在RealCall.getResponseWithInterceptorChain()中打个断点来看一下：
```
this = {RealCall@830033700784} 
interceptors = {ArrayList@830033704824}  size = 5
 0 = {RetryAndFollowUpInterceptor@830033700816} 
 1 = {BridgeInterceptor@830033705520} 
 2 = {CacheInterceptor@830033705536} 
 3 = {ConnectInterceptor@830033705696} 
 4 = {CallServerInterceptor@830033706024} 
originalRequest = {Request@830033700704} "Request{method=GET, url=http://ip.taobao.com//service/getIpInfo.php?ip=123.58.191.68, tag=null}"
retryAndFollowUpInterceptor = {RetryAndFollowUpInterceptor@830033700816} 
```
由此可见OkHttp中，Http请求的实际处理流程将大致如下图这样：
![okhttp3.jpg](https://www.wolfcstech.com/images/1315506-10e49a8770cec26b.jpg)

## RetryAndFollowUpInterceptor
具体这些Interceptor中每一个都会做些什么事情呢？我们后面再来详细地做分析。

首先来看RetryAndFollowUpInterceptor：
```
/**
 * This interceptor recovers from failures and follows redirects as necessary. It may throw an
 * {@link IOException} if the call was canceled.
 */
public final class RetryAndFollowUpInterceptor implements Interceptor {
  /**
   * How many redirects and auth challenges should we attempt? Chrome follows 21 redirects; Firefox,
   * curl, and wget follow 20; Safari follows 16; and HTTP/1.0 recommends 5.
   */
  private static final int MAX_FOLLOW_UPS = 20;

  private final OkHttpClient client;
  private StreamAllocation streamAllocation;
  private boolean forWebSocket;
  private volatile boolean canceled;

  public RetryAndFollowUpInterceptor(OkHttpClient client) {
    this.client = client;
  }

......

  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()));

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
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), true, request)) throw e.getLastConnectException();
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        if (!recover(e, false, request)) throw e;
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

      Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(
            client.connectionPool(), createAddress(followUp.url()));
      } else if (streamAllocation.stream() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }

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
`RetryAndFollowUpInterceptor`在`intercept()`中首先从client取得connection pool，用所请求的URL创建Address对象，并以此创建StreamAllocation对象。

Address描述某一个特定的服务器地址。StreamAllocation对象则用于分配一个到特定的服务器地址的流HttpStream，这个HttpStream可能是从connection pool中取得的之前没有释放的连接，也可能是重新分配的。RetryAndFollowUpInterceptor这里算是为后面的操作准备执行条件StreamAllocation。

随后`RetryAndFollowUpInterceptor.intercept()`利用Interceptor链中后面的Interceptor来获取网络响应。并检查是否为重定向响应。若不是就将响应返回，若是则做进一步处理。

对于重定向的响应，`RetryAndFollowUpInterceptor.intercept()`会利用响应的信息创建一个新的请求。并检查新请求的服务器地址与老地址是否相同，若不相同则会根据新的地址创建Address对象及StreamAllocation对象。

 `RetryAndFollowUpInterceptor`对重定向的响应也不会无休止的处理下去，它处理的最多的重定向级数为20次，超过20次时，它会抛异常出来。

`RetryAndFollowUpInterceptor`通过followUpRequest()从响应的信息中提取出重定向的信息，并构造新的网络请求：
```
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

        // Redirects don't include a request body.
        Request.Builder requestBuilder = userResponse.request().newBuilder();
        if (HttpMethod.permitsRequestBody(method)) {
          if (HttpMethod.redirectsToGet(method)) {
            requestBuilder.method("GET", null);
          } else {
            requestBuilder.method(method, null);
          }
          requestBuilder.removeHeader("Transfer-Encoding");
          requestBuilder.removeHeader("Content-Length");
          requestBuilder.removeHeader("Content-Type");
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
我们知道OkHttp提供了非常好用的容错功能，它可以从某些类型的网络错误中恢复，即出错重试机制。这种出错重试机制主要由recover()来实现：
```
  /**
   * Report and attempt to recover from a failure to communicate with a server. Returns true if
   * {@code e} is recoverable, or false if the failure is permanent. Requests with a body can only
   * be recovered if the body is buffered.
   */
  private boolean recover(IOException e, boolean routeException, Request userRequest) {
    streamAllocation.streamFailed(e);

    // The application layer has forbidden retries.
    if (!client.retryOnConnectionFailure()) return false;

    // We can't send the request body again.
    if (!routeException && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // This exception is fatal.
    if (!isRecoverable(e, routeException)) return false;

    // No more routes to attempt.
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }

  private boolean isRecoverable(IOException e, boolean routeException) {
    // If there was a protocol problem, don't recover.
    if (e instanceof ProtocolException) {
      return false;
    }

    // If there was an interruption don't recover, but if there was a timeout connecting to a route
    // we should try the next route (if there is one).
    if (e instanceof InterruptedIOException) {
      return e instanceof SocketTimeoutException && routeException;
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
主要是对某些类型IOException的恢复，恢复的次数会由StreamAllocation控制。

总结一下`RetryAndFollowUpInterceptor`做的事情：
1. 创建StreamAllocation对象，为后面流程的执行准备条件。
2. 处理重定向的HTTP响应。
3. 错误恢复。

## BridgeInterceptor
如我们在`RealCall.getResponseWithInterceptorChain()`中所见，紧接在`RetryAndFollowUpInterceptor`之后的`Interceptor`是`BridgeInterceptor`：
```
package okhttp3.internal.http;

import java.io.IOException;
import java.util.List;
import okhttp3.Cookie;
import okhttp3.CookieJar;
import okhttp3.Headers;
import okhttp3.Interceptor;
import okhttp3.MediaType;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.internal.Version;
import okio.GzipSource;
import okio.Okio;

import static okhttp3.internal.Util.hostHeader;

/**
 * Bridges from application code to network code. First it builds a network request from a user
 * request. Then it proceeds to call the network. Finally it builds a user response from the network
 * response.
 */
public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
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
    if (userRequest.header("Accept-Encoding") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

  /** Returns a 'Cookie' HTTP request header with all cookies, like {@code a=b; c=d}. */
  private String cookieHeader(List<Cookie> cookies) {
    StringBuilder cookieHeader = new StringBuilder();
    for (int i = 0, size = cookies.size(); i < size; i++) {
      if (i > 0) {
        cookieHeader.append("; ");
      }
      Cookie cookie = cookies.get(i);
      cookieHeader.append(cookie.name()).append('=').append(cookie.value());
    }
    return cookieHeader.toString();
  }
}
```
这个Interceptor做的事情比较简单。可以分为发送请求和收到响应两个阶段来看。在发送请求阶段，BridgeInterceptor补全一些http header，这主要包括`Content-Type`、`Content-Length`、`Transfer-Encoding`、`Host`、`Connection`、`Accept-Encoding`、`User-Agent`，还加载`Cookie`，随后创建新的Request，并交给后续的Interceptor处理，以获取响应。

而在从后续的Interceptor获取响应之后，会首先保存`Cookie`。如果服务器返回的响应的content是以gzip压缩过的，则会先进行解压缩，移除响应中的header `Content-Encoding`和`Content-Length`，构造新的响应并返回；否则直接返回响应。

`CookieJar`来自于`OkHttpClient`，它是OkHttp的`Cookie`管理器，负责`Cookie`的存取：
```
package okhttp3;

import java.util.Collections;
import java.util.List;

/**
 * Provides <strong>policy</strong> and <strong>persistence</strong> for HTTP cookies.
 *
 * <p>As policy, implementations of this interface are responsible for selecting which cookies to
 * accept and which to reject. A reasonable policy is to reject all cookies, though that may be
 * interfere with session-based authentication schemes that require cookies.
 *
 * <p>As persistence, implementations of this interface must also provide storage of cookies. Simple
 * implementations may store cookies in memory; sophisticated ones may use the file system or
 * database to hold accepted cookies. The <a
 * href="https://tools.ietf.org/html/rfc6265#section-5.3">cookie storage model</a> specifies
 * policies for updating and expiring cookies.
 */
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
由`OkHttpClient`默认的构造过程可以看到，OkHttp中默认是没有提供Cookie管理功能的。由这里的代码，我们大概也能知道要支持Cookie的话，需要做些什么事情。

## CacheInterceptor
CacheInterceptor紧接于BridgeInterceptor之后，它主要用来处理缓存：
```
public final class CacheInterceptor implements Interceptor {
  private static final ResponseBody EMPTY_BODY = new ResponseBody() {
    @Override public MediaType contentType() {
      return null;
    }

    @Override public long contentLength() {
      return 0;
    }

    @Override public BufferedSource source() {
      return new Buffer();
    }
  };

  final InternalCache cache;

  public CacheInterceptor(InternalCache cache) {
    this.cache = cache;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_BODY)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (validate(cacheResponse, networkResponse)) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
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

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (HttpHeaders.hasBody(response)) {
      CacheRequest cacheRequest = maybeCache(response, networkResponse.request(), cache);
      response = cacheWritingResponse(cacheRequest, response);
    }

    return response;
  }
```
对于`CacheInterceptor.intercept(Chain chain)`的分析同样可以分为两个阶段，即请求发送阶段和响应获取之后的阶段。这两个阶段由`chain.proceed(networkRequest)`来分割。

在请求发送阶段，主要是尝试从cache中获取响应，获取成功的话，且响应可用未过期，则响应会被直接返回；否则通过后续的Interceptor来从网络获取，获取到响应之后，若需要缓存的，则缓存起来。

关于HTTP具体的缓存策略这里暂时不再详述。

由`RealCall.getResponseWithInterceptorChain()`可见CacheInterceptor的cache同样来自于OkHttpClient。OkHttp已经有实现Cache的整套策略，在Cache类，但默认情况下不会被用起来，需要自己在创建OkHttpClient时，手动创建并传给OkHttpClient.Builder。
## ConnectInterceptor
CacheInterceptor接下来是ConnectInterceptor：
```
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
    HttpStream httpStream = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpStream, connection);
  }
}
```
这个类的定义看上去倒是蛮简洁的。ConnectInterceptor的主要职责是建立与服务器之间的连接，但这个事情它主要是委托给StreamAllocation来完成的。如我们前面看到的，StreamAllocation对象是在RetryAndFollowUpInterceptor中分配的。

ConnectInterceptor通过StreamAllocation创建了HttpStream对象和RealConnection对象，随后便调用了realChain.proceed()，向连接中写入HTTP请求，并从服务器读回响应。

连接建立过程的更多细节我们这里先不详述。

## CallServerInterceptor
ConnectInterceptor之后是CallServerInterceptor，这也是这个链中的最后一个Interceptor，它的主要职责是处理IO：
```
package okhttp3.internal.http;

import java.io.IOException;
import java.net.ProtocolException;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.internal.connection.StreamAllocation;
import okio.BufferedSink;
import okio.Okio;
import okio.Sink;

/** This is the last interceptor in the chain. It makes a network call to the server. */
public final class CallServerInterceptor implements Interceptor {
  private final boolean forWebSocket;

  public CallServerInterceptor(boolean forWebSocket) {
    this.forWebSocket = forWebSocket;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    HttpStream httpStream = ((RealInterceptorChain) chain).httpStream();
    StreamAllocation streamAllocation = ((RealInterceptorChain) chain).streamAllocation();
    Request request = chain.request();

    long sentRequestMillis = System.currentTimeMillis();
    httpStream.writeRequestHeaders(request);

    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      Sink requestBodyOut = httpStream.createRequestBody(request, request.body().contentLength());
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
    }

    httpStream.finishRequest();

    Response response = httpStream.readResponseHeaders()
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    if (!forWebSocket || response.code() != 101) {
      response = response.newBuilder()
          .body(httpStream.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    int code = response.code();
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
}
```
`CallServerInterceptor`首先将http请求头部发给服务器，如果http请求有body的话，会再将body发送给服务器，继而通过`httpStream.finishRequest()`结束http请求的发送。

随后便是从连接中读取服务器返回的http响应，并构造Response。

如果请求的header或服务器响应的header中，`Connection`值为`close`，`CallServerInterceptor`还会关闭连接。

最后便是返回Response。

总结一下这几个Interceptor的职责：
RetryAndFollowUpInterceptor --->创建StreamAllocation对象，处理http的redirect，出错重试。对后续Interceptor的执行的影响：修改request及StreamAllocation。
BridgeInterceptor-------------->补全缺失的一些http header。对后续Interceptor的执行的影响：修改request。
CacheInterceptor-------------->处理http缓存。对后续Interceptor的执行的影响：若缓存中有所需请求的响应，则后续Interceptor不再执行。
ConnectInterceptor------------>借助于前面分配的StreamAllocation对象建立与服务器之间的连接，并选定交互所用的协议是HTTP 1.1还是HTTP 2。对后续Interceptor的执行的影响：创建了httpStream和connection。
CallServerInterceptor----------->处理IO，与服务器进行数据交换。对后续Interceptor的执行的影响：为Interceptor链中的最后一个Interceptor，没有后续Interceptor。

End。
