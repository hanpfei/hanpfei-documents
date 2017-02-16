---
title: OkHttp3连接建立过程分析
date: 2016-10-27 11:43:49
categories: 网络协议
tags:
- 源码分析
- Android开发
- 网络协议
---

如我们前面在 [OkHttp3 HTTP请求执行流程分析](https://www.wolfcstech.com/2016/10/14/OkHttp3-HTTP%E8%AF%B7%E6%B1%82%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/) 中的分析，OkHttp3通过Interceptor链来执行HTTP请求，整体的执行过程大体如下：

<!--more-->

![OkHttp Flow](https://www.wolfcstech.com/images/1315506-8492fecef4238d86.png)

这些Interceptor中每一个的职责，这里不再赘述。

在OkHttp3中，`StreamAllocation`是用来建立执行HTTP请求所需网络设施的组件，如其名字所显示的那样，分配Stream。但它具体做的事情根据是否设置了代理，以及请求的类型，如HTTP、HTTPS或HTTP/2的不同而有所不同。代理相关的处理，包括TCP连接的建立，在 [OkHttp3中的代理与路由](https://www.wolfcstech.cn/2016/10/14/OkHttp3%E4%B8%AD%E7%9A%84%E4%BB%A3%E7%90%86%E4%B8%8E%E8%B7%AF%E7%94%B1/) 一文中有详细的说明。

在整个HTTP请求的执行过程中，**`StreamAllocation`** 对象分配的比较早，在RetryAndFollowUpInterceptor.intercept(Chain chain)中就完成了：
```
  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);
```

`StreamAllocation `的对象构造过程没有什么特别的：
```
  public StreamAllocation(ConnectionPool connectionPool, Address address, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.routeSelector = new RouteSelector(address, routeDatabase());
    this.callStackTrace = callStackTrace;
  }
```

在OkHttp3中，`okhttp3.internal.http.RealInterceptorChain`将Interceptor连接成执行链。`RetryAndFollowUpInterceptor`借助于`RealInterceptorChain`将创建的`StreamAllocation`对象传递给后面执行的Interceptor。而在`RealInterceptorChain`中，`StreamAllocation`对象并没有被真正用到。紧跟在`RetryAndFollowUpInterceptor`之后执行的 `okhttp3.internal.http.BridgeInterceptor` 和 `okhttp3.internal.cache.CacheInterceptor`，它们的职责分别是补足用户创建的请求中缺少的必须的请求头和处理缓存，也没有真正用到`StreamAllocation`对象。

在OkHttp3的HTTP请求执行过程中，`okhttp3.internal.connection.ConnectInterceptor`和`okhttp3.internal.http.CallServerInterceptor`是与网络交互的关键。

`CallServerInterceptor`负责将HTTP请求写入网络IO流，并从网络IO流中读取服务器返回的数据。而`ConnectInterceptor`则负责为`CallServerInterceptor`建立可用的连接。此处 **可用的** 含义主要为，可以直接写入HTTP请求的数据：

* 设置了HTTP代理的HTTP请求，与代理建立好TCP连接；
* 设置了HTTP代理的HTTPS请求，与HTTP服务器建立通过HTTP代理的隧道连接，并完成TLS握手；
* 设置了HTTP代理的HTTP/2请求，与HTTP服务器建立通过HTTP代理的隧道连接，并完成与服务器的TLS握手及协议协商；
* 设置了SOCKS代理的HTTP请求，通过代理与HTTP服务器建立好连接；
* 设置了SOCKS代理的HTTPS请求，通过代理与HTTP服务器建立好连接，并完成TLS握手；
* 设置了SOCKS代理的HTTP/2请求，通过代理与HTTP服务器建立好连接，并完成与服务器的TLS握手及协议协商；
* 无代理的HTTP请求，与服务器建立好TCP连接；
* 无代理的HTTPS请求，与服务器建立TCP连接，并完成TLS握手；
* 无代理的HTTP/2请求，与服务器建立好TCP连接，完成TLS握手及协议协商。

后面我们更详细地来看一下这个过程。

`ConnectInterceptor`的代码看上去比较简单：
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
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```
`ConnectInterceptor`从`RealInterceptorChain`获取前面的Interceptor传过来的`StreamAllocation`对象，执行 `streamAllocation.newStream()` 完成前述所有的连接建立工作，并将这个过程中创建的用于网络IO的RealConnection对象，以及对于与服务器交互最为关键的HttpCodec等对象传递给后面的Interceptor，也就是`CallServerInterceptor`。

# OkHttp3的连接池

在具体地分析 `streamAllocation.newStream()` 的执行过程之前，我们先来看一下OkHttp3的连接池的设计实现。

OkHttp3将客户端与服务器之间的连接抽象为Connection/RealConnection，为了管理这些连接的复用而设计了ConnectionPool。共享相同`Address`的请求可以复用连接，ConnectionPool实现了哪些连接保持打开状态以备后用的策略。

## ConnectionPool是什么？
借助于ConnectionPool的成员变量声明来一窥ConnectionPool究竟是什么：
```
/**
 * Manages reuse of HTTP and HTTP/2 connections for reduced network latency. HTTP requests that
 * share the same {@link Address} may share a {@link Connection}. This class implements the policy
 * of which connections to keep open for future use.
 */
public final class ConnectionPool {
  /**
   * Background threads are used to cleanup expired connections. There will be at most a single
   * thread running per connection pool. The thread pool executor permits the pool itself to be
   * garbage collected.
   */
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

  /** The maximum number of idle connections for each address. */
  private final int maxIdleConnections;
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

  private final Deque<RealConnection> connections = new ArrayDeque<>();
  final RouteDatabase routeDatabase = new RouteDatabase();
  boolean cleanupRunning;
```
`ConnectionPool`的核心是`RealConnection`的容器，且是顺序容器，而不是关联容器。`ConnectionPool`用双端队列`Deque<RealConnection>`来保存它所管理的所有`RealConnection`。

`ConnectionPool`还会对连接池中最大的空闲连接数及连接的保活时间进行控制，`maxIdleConnections`和`keepAliveDurationNs`成员分别体现对最大空闲连接数及连接保活时间的控制。这种控制通过匿名的`Runnable cleanupRunnable`在线程池`executor`中执行，并在向连接池中添加新的`RealConnection`触发。

## 连接池ConnectionPool的创建

OkHttp3的用户可以自行创建ConnectionPool，对最大空闲连接数及连接的保活时间进行配置，并在OkHttpClient创建期间，将其传给OkHttpClient.Builder，在OkHttpClient中启用它。没有定制连接池的情况下，则在OkHttpClient.Builder构造过程中以默认参数创建：

```
    public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
```
ConnectionPool的默认构造过程如下：
```
  /**
   * Create a new connection pool with tuning parameters appropriate for a single-user application.
   * The tuning parameters in this pool are subject to change in future OkHttp releases. Currently
   * this pool holds up to 5 idle connections which will be evicted after 5 minutes of inactivity.
   */
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
在默认情况下，`ConnectionPool` 最多保存 ***5个*** 处于空闲状态的连接，且连接的默认保活时间为 ***5分钟***。

## RealConnection的存/取

OkHttp内部的组件可以通过put()方法向`ConnectionPool`中添加`RealConnection`：
```
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```
在向`ConnectionPool`中添加`RealConnection`时，若发现cleanupRunnable还没有运行会触发它的运行。

cleanupRunnable的职责本就是清理无效的`RealConnection`，只要`ConnectionPool`中存在`RealConnection`，则这种清理的需求总是存在的，因而这里会去启动cleanupRunnable。

根据需要启动了cleanupRunnable之后，将`RealConnection`添加进双端队列connections。

这里先启动 `cleanupRunnable`，后向 `connections` 中添加`RealConnection`。有没有可能发生：

启动`cleanupRunnable`之后，向`connections`中添加`RealConnection`之前，执行 put() 的线程被抢占，`cleanupRunnable`的线程被执行，它发现`connections`中没有任何`RealConnection`，于是从容地退出而导致后面添加的`RealConnection`永远不会得得清理。

这样的情况呢？答案是 不会。为什么呢？`put()`执行之前总是会用`ConnectionPool`对象锁来保护，而在`ConnectionPool.cleanup()`中，遍历`connections`也总是会先对`ConnectionPool`对象加锁保护的。即使执行 put() 的线程被抢占，`cleanupRunnable`的线程也会由于拿不到`ConnectionPool`对象锁而等待 put() 执行结束。

OkHttp内部的组件可以通过 `get()` 方法从`ConnectionPool`中获取`RealConnection`：
```
  /** Returns a recycled connection to {@code address}, or null if no such connection exists. */
  RealConnection get(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.allocations.size() < connection.allocationLimit
          && address.equals(connection.route().address)
          && !connection.noNewStreams) {
        streamAllocation.acquire(connection);
        return connection;
      }
    }
    return null;
  }
```
`get()` 方法遍历 `connections` 中的所有 `RealConnection` 寻找同时满足如下三个条件的`RealConnection`：
* `RealConnection`的allocations的数量小于allocationLimit。每个allocation代表在该`RealConnection`上正在执行的一个请求。这个条件用于控制相同连接上，同一时间执行的并发请求的个数。对于HTTP/2连接而言，allocationLimit限制是在连接建立阶段由双方协商的。对于HTTP或HTTPS连接而言，这个值则总是1。从`RealConnection.establishProtocol()`可以清晰地看到这一点：
```
    if (protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // Framed connection timeouts are set per-stream.

      Http2Connection http2Connection = new Http2Connection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .listener(this)
          .build();
      http2Connection.start();

      // Only assign the framed connection once the preface has been sent successfully.
      this.allocationLimit = http2Connection.maxConcurrentStreams();
      this.http2Connection = http2Connection;
    } else {
      this.allocationLimit = 1;
    }
```
*  `RealConnection` 的 `address` 与传入的 `Address` 参数相等。`RealConnection` 的 `address` 描述建立连接所需的配置信息，包括对端的信息等，不难理解只有所有相关配置相等时 `RealConnection` 才是真正能复用的。具体看一下`Address`相等性比较的依据：
```
  @Override public boolean equals(Object other) {
    if (other instanceof Address) {
      Address that = (Address) other;
      return this.url.equals(that.url)
          && this.dns.equals(that.dns)
          && this.proxyAuthenticator.equals(that.proxyAuthenticator)
          && this.protocols.equals(that.protocols)
          && this.connectionSpecs.equals(that.connectionSpecs)
          && this.proxySelector.equals(that.proxySelector)
          && equal(this.proxy, that.proxy)
          && equal(this.sslSocketFactory, that.sslSocketFactory)
          && equal(this.hostnameVerifier, that.hostnameVerifier)
          && equal(this.certificatePinner, that.certificatePinner);
    }
    return false;
  }
```
这种相等性的条件给人感觉还是蛮苛刻的，特别是对url的对比。
这难免会让我们有些担心，对 `Address` 如此苛刻的相等性比较，又有多大的机会能复用连接呢？
我们的担心其实是多余的。只有在 `StreamAllocation.findConnection()` 中，会通过`Internal.instance` 调用 `ConnectionPool.get()` 来获取 `RealConnection` ：

```
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    Route selectedRoute;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
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
        refusedStreamCount = 0;
      }
    }
    RealConnection newConnection = new RealConnection(selectedRoute);

    synchronized (connectionPool) {
      acquire(newConnection);
      Internal.instance.put(connectionPool, newConnection);
      this.connection = newConnection;
      if (canceled) throw new IOException("Canceled");
    }

```

Internal.instance的实现在OkHttpClient 中：
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

      @Override public StreamAllocation callEngineGetStreamAllocation(Call call) {
        return ((RealCall) call).streamAllocation();
      }
```
可见 `ConnectionPool.get()` 的 `Address` 参数来自于`StreamAllocation`。`StreamAllocation`的`Address` 在构造时由外部传入。构造了`StreamAllocation`对象的`RetryAndFollowUpInterceptor`，其构造`Address`的过程是这样的：
```
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
`Address` 除了 `uriHost` 和 `uriPort` 外的所有构造参数均来自于OkHttpClient，而`Address`的`url` 字段正是根据这两个参数构造的：
```
  public Address(String uriHost, int uriPort, Dns dns, SocketFactory socketFactory,
      SSLSocketFactory sslSocketFactory, HostnameVerifier hostnameVerifier,
      CertificatePinner certificatePinner, Authenticator proxyAuthenticator, Proxy proxy,
      List<Protocol> protocols, List<ConnectionSpec> connectionSpecs, ProxySelector proxySelector) {
    this.url = new HttpUrl.Builder()
        .scheme(sslSocketFactory != null ? "https" : "http")
        .host(uriHost)
        .port(uriPort)
        .build();
```
可见 `Address` 的 `url` 字段仅包含HTTP请求url的 schema + host + port 这三部分的信息，而不包含 path 和 query 等信息。`ConnectionPool`主要是根据服务器的地址来决定复用的。
* `RealConnection`还有可分配的Stream。对于HTTP或HTTPS而言，不能同时在相同的连接上执行多个请求。即使对于HTTP/2而言，StreamID的空间也是有限的，同一个连接上的StreamID总有分配完的时候，而在StreamID被分配完了之后，该连接就不能再被使用了。

OkHttp内部对`ConnectionPool`的访问总是通过Internal.instance来进行。整个OkHttp中也只有`StreamAllocation` 存取了 `ConnectionPool`，也就是我们前面列出的`StreamAllocation.findConnection()` 方法，相关的组件之间的关系大体如下图：

![OkHttp Connection Pool](https://www.wolfcstech.com/images/1315506-6a60bf781625e166.png)

## RealConnection的清理
`ConnectionPool` 中对于 `RealConnection` 的清理在put()方法中触发，执行 `cleanupRunnable` 来完成清理动作：
```
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
`cleanupRunnable`每执行一次清理动作，都会等待一段时间再次执行，而具体等待的时长由`cleanup()`方法决定，直到`cleanup()`方法返回-1退出。`cleanup()`方法定义如下：
```
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

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }

  /**
   * Prunes any leaked allocations and then returns the number of remaining live allocations on
   * {@code connection}. Allocations are leaked if the connection is tracking them but the
   * application code has abandoned them. Leak detection is imprecise and relies on garbage
   * collection.
   */
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);

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

      references.remove(i);
      connection.noNewStreams = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }

    return references.size();
  }
```
`cleanup()`方法遍历`connections`，并从中找到处于空闲状态时间最长的一个`RealConnection`，然后根据查找结果的不同，分为以下几种情况处理：
 * 找到一个处于空闲状态的`RealConnection`，且该`RealConnection`处于空闲状态的时间超出了设置的保活时间，或者当前`ConnectionPool`中处于空闲状态的连接数超出了设置的最大空闲连接数，将该`RealConnection`从`connections`中移除，并关闭该`RealConnection`关联的底层socket，然后返回0，以此请求`cleanupRunnable`立即再次检查所有的连接。
 * 找到一个处于空闲状态的`RealConnection`，但该`RealConnection`处于空闲状态的时间尚未超出设置的保活时间，且当前`ConnectionPool`中处于空闲状态的连接数尚未超出设置的最大空闲连接数，则返回保活时间与该`RealConnection`处于空闲状态的时间之间的差值，请求`cleanupRunnable`等待这么长一段时间之后再次检查所有的连接。
 * 没有找到处于空闲状态的连接，但找到了使用中的连接，则返回保活时间，请求`cleanupRunnable`等待这么长一段时间之后再次检查所有的连接。
 * 没有找到处于空闲状态的连接，也没有找到使用中的连接，也就意味着连接池中尚没有任何连接，则将 `cleanupRunning` 置为false，并返回 -1，请求 `cleanupRunnable` 退出。

`cleanup()` 通过 `pruneAndGetAllocationCount()` 检查正在使用一个特定连接的请求个数，并以此来判断一个连接是否处于空闲状态。后者通遍历 `connection.allocations` 并检查每个元素的`StreamAllocation` 的状态，若`StreamAllocation` 为空，则认为是发现了一个leak，它会更新连接的空闲时间为当前时间减去保活时间并返回0，以此请求 `cleanup()` 立即关闭、清理掉该 leak 的连接。

## ConnectionPool的用户接口
OkHttp的用户可以自己创建 `ConnectionPool` 对象，这个类也提供了一些用户接口以方便用户获取空闲状态的连接数、总的连接数，以及手动清除空闲状态的连接：
```
  /** Returns the number of idle connections in the pool. */
  public synchronized int idleConnectionCount() {
    int total = 0;
    for (RealConnection connection : connections) {
      if (connection.allocations.isEmpty()) total++;
    }
    return total;
  }

  /**
   * Returns total number of connections in the pool. Note that prior to OkHttp 2.7 this included
   * only idle connections and HTTP/2 connections. Since OkHttp 2.7 this includes all connections,
   * both active and inactive. Use {@link #idleConnectionCount()} to count connections not currently
   * in use.
   */
  public synchronized int connectionCount() {
    return connections.size();
  }

......

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

# 新建流
回到新建流的过程，连接建立的各种细节处理都在这里。 `StreamAllocation.newStream()` 完成新建流的动作：
```
  public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    int connectTimeout = client.connectTimeoutMillis();
    int readTimeout = client.readTimeoutMillis();
    int writeTimeout = client.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

      HttpCodec resultCodec;
      if (resultConnection.http2Connection != null) {
        resultCodec = new Http2Codec(client, this, resultConnection.http2Connection);
      } else {
        resultConnection.socket().setSoTimeout(readTimeout);
        resultConnection.source.timeout().timeout(readTimeout, MILLISECONDS);
        resultConnection.sink.timeout().timeout(writeTimeout, MILLISECONDS);
        resultCodec = new Http1Codec(
            client, this, resultConnection.source, resultConnection.sink);
      }

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```
所谓的流，是封装了底层的IO，可以直接用来收发数据的组件，它会将请求的数据序列化之后发送到网络，并将接收的数据反序列化为应用程序方便操作的格式。在 OkHttp3 中，这样的组件被抽象为`HttpCodec`。`HttpCodec`的定义如下 (okhttp/okhttp/src/main/java/okhttp3/internal/http/HttpCodec.java)：
```
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
  void finishRequest() throws IOException;

  /** Read and return response headers. */
  Response.Builder readResponseHeaders() throws IOException;

  /** Returns a stream that reads the response body. */
  ResponseBody openResponseBody(Response response) throws IOException;

  /**
   * Cancel this stream. Resources held by this stream will be cleaned up, though not synchronously.
   * That may happen later by the connection pool thread.
   */
  void cancel();
}
```
`HttpCodec`提供了这样的一些操作：
* 为发送请求而提供的，写入请求头部。
* 为发送请求而提供的，创建请求体，以用于发送请求体数据。
* 为发送请求而提供的，结束请求发送。
* 为获得响应而提供的，读取响应头部。
* 为获得响应而提供的，打开请求体，以用于后续获取请求体数据。
* 取消请求执行。

`StreamAllocation.newStream()` 主要做的事情正是创建`HttpCodec`。`StreamAllocation.newStream()` 根据 `OkHttpClient`中的设置，连接超时、读超时、写超时及连接失败是否重试，调用 `findHealthyConnection()` 完成 连接，即`RealConnection` 的创建。然后根据HTTP协议的版本创建Http1Codec或Http2Codec。

`findHealthyConnection()` 根据目标服务器地址查找一个连接，如果它是可用的就直接返回，如果不可用则会重复查找直到找到一个可用的为止。在连接已被破坏而不可用时，还会释放连接：
```
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
连接是否可用的标准如下 (okhttp/okhttp/src/main/java/okhttp3/internal/connection/RealConnection.java)：
```
  /** Returns true if this connection is ready to host new streams. */
  public boolean isHealthy(boolean doExtensiveChecks) {
    if (socket.isClosed() || socket.isInputShutdown() || socket.isOutputShutdown()) {
      return false;
    }

    if (http2Connection != null) {
      return true; // TODO: check framedConnection.shutdown.
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
首先要可以进行IO，此外对于HTTP/2，只要`http2Connection`存在即可。如我们前面在`ConnectInterceptor` 中看到的，如果HTTP请求的method不是 "GET" ，`doExtensiveChecks`为true时，需要做额外的检查。

`findHealthyConnection()` 通过 `findConnection()`查找一个连接：
```
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
        refusedStreamCount = 0;
      }
    }
    RealConnection newConnection = new RealConnection(selectedRoute);

    synchronized (connectionPool) {
      acquire(newConnection);
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
`findConnection()` 返回一个用于流执行底层IO的连接。这个方法优先复用已经创建的连接；在没有可复用连接的情况下新建一个。

在同一次 `newStream()` 的执行过程中，有没有可能两次执行 `findConnection()` ，第一次`connection` 字段为空，第二次不为空？这个地方对`connection`字段的检查，看起来有点多余。执行 `findConnection()` 时，`connection` 不为空的话，意味着 `codec` 不为空，而在方法的开始处已经有对`codec`字段的状态做过检查。真的是这样的吗？

答案当然是否定的。同一次 `newStream()` 的执行过程中，没有可能两次执行`findConnection()`，第一次`connection`字段为空，第二次不为空，然而一个HTTP请求的执行过程，又不是一定只调用一次`newStream()`。

`newStream()`的直接调用者是`ConnectInterceptor`，所有的Interceptor用`RealInterceptorChain`链起来，在Interceptor链中，`ConnectInterceptor` 和`RetryAndFollowUpInterceptor` 隔着 `CacheInterceptor` 和 `BridgeInterceptor` 。然而`newStream()` 如果出错的话，则是会通过抛出`Exception`返回到`RetryAndFollowUpInterceptor` 来处理错误的。

`RetryAndFollowUpInterceptor` 中会尝试基于相同的 `StreamAllocation` 对象来恢复对HTTP请求的处理。`RetryAndFollowUpInterceptor` 通过 `hasMoreRoutes()` 等方法，来检查`StreamAllocation` 对象的状态，通过 `streamFailed(IOException e)`、`release()`、`streamFinished(boolean noNewStreams, HttpCodec codec)`等方法来reset `StreamAllocation`对象的一些状态。

回到`StreamAllocation`的 `findConnection()`方法。没有连接存在，且连接池中也没有找到所需的连接时，它会新建一个连接。通过如下的步骤新建连接：
* 为连接选择一个`Route`。
* 新建一个`RealConnection`对象。
```
  public RealConnection(Route route) {
    this.route = route;
  }
```
* 将当前`StreamAllocation`对象的引用保存进`RealConnection`的allocations。如我们前面在分析ConnectionPool时所见的那样，这主要是为了追踪`RealConnection`。
```
  /**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```
* 将`RealConnection`保存进连接池。
* 保存对`RealConnection`的引用。
* 检查请求是否被取消，若取消，则抛出异常。
* 建立连接。
* 更新RouteDatabase中`Route`的状态。

# ConnectionSpec

在OkHttp中，ConnectionSpec用于描述传输HTTP流量的socket连接的配置。对于https请求，这些配置主要包括协商安全连接时要使用的TLS版本号和密码套件，是否支持TLS扩展等；对于http请求则几乎不包含什么信息。

OkHttp有预定义几组ConnectionSpec (okhttp/okhttp/src/main/java/okhttp3/ConnectionSpec.java)：
```
  /** A modern TLS connection with extensions like SNI and ALPN available. */
  public static final ConnectionSpec MODERN_TLS = new Builder(true)
      .cipherSuites(APPROVED_CIPHER_SUITES)
      .tlsVersions(TlsVersion.TLS_1_2, TlsVersion.TLS_1_1, TlsVersion.TLS_1_0)
      .supportsTlsExtensions(true)
      .build();

  /** A backwards-compatible fallback connection for interop with obsolete servers. */
  public static final ConnectionSpec COMPATIBLE_TLS = new Builder(MODERN_TLS)
      .tlsVersions(TlsVersion.TLS_1_0)
      .supportsTlsExtensions(true)
      .build();

  /** Unencrypted, unauthenticated connections for {@code http:} URLs. */
  public static final ConnectionSpec CLEARTEXT = new Builder(false).build();
```

预定义的这些`ConnectionSpec`被组织为默认`ConnectionSpec`集合 (okhttp/okhttp/src/main/java/okhttp3/OkHttpClient.java)：
```
public class OkHttpClient implements Cloneable, Call.Factory {
  private static final List<Protocol> DEFAULT_PROTOCOLS = Util.immutableList(
      Protocol.HTTP_2, Protocol.HTTP_1_1);

  private static final List<ConnectionSpec> DEFAULT_CONNECTION_SPECS = Util.immutableList(
      ConnectionSpec.MODERN_TLS, ConnectionSpec.COMPATIBLE_TLS, ConnectionSpec.CLEARTEXT);
```

OkHttp中由OkHttpClient管理`ConnectionSpec`集合 。OkHttp的用户可以在构造`OkHttpClient`的过程中提供自己的`ConnectionSpec`集合。默认情况下`OkHttpClient`会使用前面看到的默认`ConnectionSpec`集合。

在`RetryAndFollowUpInterceptor`中创建`Address`时，`ConnectionSpec`集合被从`OkHttpClient`获取，并由`Address`引用。

OkHttp还提供了`ConnectionSpecSelector`，用以从`ConnectionSpec`集合中选择与SSLSocket匹配的`ConnectionSpec`，并对SSLSocket做配置的操作。

在`StreamAllocation`的findConnection()中，`ConnectionSpec`集合被从`Address`中取出来，以用于连接建立过程。

# 建立连接

回到连接建立的过程。`RealConnection.connect()`执行连接建立的过程(okhttp/okhttp/src/main/java/okhttp3/internal/connection/RealConnection.java)：
```
  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      List<ConnectionSpec> connectionSpecs, boolean connectionRetryEnabled) {
    if (protocol != null) throw new IllegalStateException("already connected");

    RouteException routeException = null;
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

    while (protocol == null) {
      try {
        if (route.requiresTunnel()) {
          buildTunneledConnection(connectTimeout, readTimeout, writeTimeout,
              connectionSpecSelector);
        } else {
          buildConnection(connectTimeout, readTimeout, writeTimeout, connectionSpecSelector);
        }
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
```
这里的执行过程大体如下：
* 检查连接是否已经建立，若已经建立，则抛出异常，否则继续执行。连接是否建立由`protocol` 标识，它表示在整个连接建立，及可能的协议协商过程中选择的所要使用的协议。
* 根据`ConnectionSpec`集合`connectionSpecs`构造`ConnectionSpecSelector`。
* 若请求不是安全的请求，会对请求再执行一些额外的限制。这些限制包括：
 - `ConnectionSpec`集合中必须要包含`ConnectionSpec.CLEARTEXT`。这也就是说，OkHttp的用户可以通过为`OkHttpClient`设置不包含`ConnectionSpec.CLEARTEXT`的`ConnectionSpec`集合来禁用所有的明文请求。
 - 平台本身的安全策略允许向相应的主机发送明文请求。对于Android平台而言，这种安全策略主要由系统的组件`android.security.NetworkSecurityPolicy`执行 (okhttp/okhttp/src/main/java/okhttp3/internal/platform/AndroidPlatform.java)：
```
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
平台的这种安全策略并不是每个Android版本都有的。Android 6.0之后存在这种控制。
* 根据请求是否需要建立隧道连接，而分别执行`buildTunneledConnection()` 和 `buildConnection()`。是否需要建立隧道连接的依据为 (okhttp/okhttp/src/main/java/okhttp3/Route.java)：
```
  /**
   * Returns true if this route tunnels HTTPS through an HTTP proxy. See <a
   * href="http://www.ietf.org/rfc/rfc2817.txt">RFC 2817, Section 5.2</a>.
   */
  public boolean requiresTunnel() {
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
  }
```
即对于设置了HTTP代理，且安全的连接 (SSL) 需要请求代理服务器建立一个到目标HTTP服务器的隧道连接，客户端与HTTP代理建立TCP连接，以此请求HTTP代理服务在客户端与HTTP服务器之间进行数据的盲转发。

## 建立隧道连接
建立隧道连接的过程如下：

 1. 构造一个 建立隧道连接 请求。
 2. 与HTTP代理服务器建立TCP连接。
 3. 创建隧道。这主要是将 建立隧道连接 请求发送给HTTP代理服务器，并处理它的响应。
 4. 重复上面的第2和第3步，知道建立好了隧道连接。至于为什么要重复多次，及关于代理认证的内容，可以参考代理协议相关的内容。
 5. 建立协议。

关于建立隧道连接更详细的过程可参考 [OkHttp3中的代理与路由](https://www.wolfcstech.cn/2016/10/14/OkHttp3%E4%B8%AD%E7%9A%84%E4%BB%A3%E7%90%86%E4%B8%8E%E8%B7%AF%E7%94%B1/) 的相关部分。

## 建立普通连接
建立普通连接的过程比较直接：
```
  /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  private void buildConnection(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    connectSocket(connectTimeout, readTimeout);
    establishProtocol(readTimeout, writeTimeout, connectionSpecSelector);
  }
```
 1. 建立一个TCP连接。
 2. 建立协议。

更详细的过程可参考 [OkHttp3中的代理与路由](https://www.wolfcstech.cn/2016/10/14/OkHttp3%E4%B8%AD%E7%9A%84%E4%BB%A3%E7%90%86%E4%B8%8E%E8%B7%AF%E7%94%B1/) 的相关部分。

## 建立协议

不管是建立隧道连接，还是建立普通连接，都少不了 建立协议 这一步。这一步是在建立好了TCP连接之后，而在该TCP能被拿来收发数据之前执行的。它主要为数据的加密传输做一些初始化，比如TLS握手，HTTP/2的协议协商等。
```
  private void establishProtocol(int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    if (route.address().sslSocketFactory() != null) {
      connectTls(readTimeout, writeTimeout, connectionSpecSelector);
    } else {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
    }

    if (protocol == Protocol.HTTP_2) {
      socket.setSoTimeout(0); // Framed connection timeouts are set per-stream.

      Http2Connection http2Connection = new Http2Connection.Builder(true)
          .socket(socket, route.address().url().host(), source, sink)
          .listener(this)
          .build();
      http2Connection.start();

      // Only assign the framed connection once the preface has been sent successfully.
      this.allocationLimit = http2Connection.maxConcurrentStreams();
      this.http2Connection = http2Connection;
    } else {
      this.allocationLimit = 1;
    }
  }
```
 1. 对于加密的数据传输，创建TLS连接。对于明文传输，则设置`protocol`和`socket`。
`socket`指向直接与应用层，如HTTP或HTTP/2，交互的Socket：
对于明文传输没有设置HTTP代理的HTTP请求，它是与HTTP服务器之间的TCP socket；
对于明文传输设置了HTTP代理或SOCKS代理的HTTP请求，它是与代理服务器之间的TCP socket；
对于加密传输没有设置HTTP代理服务器的HTTP或HTTP2请求，它是与HTTP服务器之间的SSLScoket；
对于加密传输设置了HTTP代理服务器的HTTP或HTTP2请求，它是与HTTP服务器之间经过了代理服务器的SSLSocket，一个隧道连接；
对于加密传输设置了SOCKS代理的HTTP或HTTP2请求，它是一条经过了代理服务器的SSLSocket连接。

 2. 对于HTTP/2，会建立HTTP/2连接，并进一步协商连接参数，如连接上可同时执行的并发请求数等。而对于非HTTP/2，则将连接上可同时执行的并发请求数设置为1。

## 建立TLS连接
进一步来看建立协议过程中，为安全请求所做的建立TLS连接的过程：
```
  private void connectTls(int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    Address address = route.address();
    SSLSocketFactory sslSocketFactory = address.sslSocketFactory();
    boolean success = false;
    SSLSocket sslSocket = null;
    try {
      // Create the wrapper over the connected socket.
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
TLS连接是对原始的TCP连接的一个封装，以提供TLS握手，及数据收发过程中的加密解密等功能。在Java中，用SSLSocket来描述。上面建立TLS连接的过程大体为：
 1. 用SSLSocketFactory基于原始的TCP Socket，创建一个SSLSocket。
 2. 配置SSLSocket。
 3. 在前面选择的ConnectionSpec支持TLS扩展参数时，配置TLS扩展参数。
 4. 启动TLS握手。
 5. TLS握手完成之后，获取握手信息。
 6. 对TLS握手过程中传回来的证书进行验证。
 7. 检查证书钉扎。
 8. 在前面选择的ConnectionSpec支持TLS扩展参数时，获取TLS握手过程中顺便完成的协议协商过程所选择的协议。这个过程主要用于HTTP/2的ALPN扩展。
 9. OkHttp主要使用Okio来做IO操作，这里会基于前面获取的SSLSocket创建用于执行IO的BufferedSource和BufferedSink等，并保存握手信息及所选择的协议。

具体来看`ConnectionSpecSelector`中配置SSLSocket的过程：
```
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
这个过程分为如下的两个步骤：
 1. 从为OkHttp配置的ConnectionSpec集合中选择一个与SSLSocket兼容的一个。SSLSocket与ConnectionSpec兼容的标准如下：
```
  public boolean isCompatible(SSLSocket socket) {
    if (!tls) {
      return false;
    }

    if (tlsVersions != null
        && !nonEmptyIntersection(tlsVersions, socket.getEnabledProtocols())) {
      return false;
    }

    if (cipherSuites != null
        && !nonEmptyIntersection(cipherSuites, socket.getEnabledCipherSuites())) {
      return false;
    }

    return true;
  }

  /**
   * An N*M intersection that terminates if any intersection is found. The sizes of both arguments
   * are assumed to be so small, and the likelihood of an intersection so great, that it is not
   * worth the CPU cost of sorting or the memory cost of hashing.
   */
  private static boolean nonEmptyIntersection(String[] a, String[] b) {
    if (a == null || b == null || a.length == 0 || b.length == 0) {
      return false;
    }
    for (String toFind : a) {
      if (indexOf(b, toFind) != -1) {
        return true;
      }
    }
    return false;
  }
```
即ConnectionSpec启用的TLS版本及密码套件，与SSLSocket启用的有交集。
 2 将选择的ConnectionSpec应用在SSLSocket上。OkHttpClient中ConnectionSpec的应用：
```
      @Override
      public void apply(ConnectionSpec tlsConfiguration, SSLSocket sslSocket, boolean isFallback) {
        tlsConfiguration.apply(sslSocket, isFallback);
      }
```
而在ConnectionSpec中：
```
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
        ? intersect(String.class, cipherSuites, sslSocket.getEnabledCipherSuites())
        : sslSocket.getEnabledCipherSuites();
    String[] tlsVersionsIntersection = tlsVersions != null
        ? intersect(String.class, tlsVersions, sslSocket.getEnabledProtocols())
        : sslSocket.getEnabledProtocols();

    // In accordance with https://tools.ietf.org/html/draft-ietf-tls-downgrade-scsv-00
    // the SCSV cipher is added to signal that a protocol fallback has taken place.
    if (isFallback && indexOf(sslSocket.getSupportedCipherSuites(), "TLS_FALLBACK_SCSV") != -1) {
      cipherSuitesIntersection = concat(cipherSuitesIntersection, "TLS_FALLBACK_SCSV");
    }

    return new Builder(this)
        .cipherSuites(cipherSuitesIntersection)
        .tlsVersions(tlsVersionsIntersection)
        .build();
  }
```
主要是：
  - 求得ConnectionSpec启用的TLS版本及密码套件与SSLSocket启用的TLS版本及密码套件之间的交集，构造新的ConnectionSpec。
  - 重新为SSLSocket设置启用的TLS版本及密码套件为上一步求得的交集。

我们知道HTTP/2的协议协商主要是利用了TLS的ALPN扩展来完成的。这里再来详细的看一下配置TLS扩展的过程。对于Android平台而言，这部分逻辑在AndroidPlatform：
```
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
TLS扩展相关的方法不是SSLSocket接口的标准方法，不同的SSL/TLS实现库对这些接口的支持程度不一样，因而这里通过反射机制调用TLS扩展相关的方法。

这里主要配置了3个TLS扩展，分别是session tickets，SNI和ALPN。session tickets用于会话回复，SNI用于支持单个主机配置了多个域名的情况，ALPN则用于HTTP/2的协议协商。可以看到为SNI设置的hostname最终来源于Url，也就意味着使用HttpDns时，如果直接将IP地址替换原来Url中的域名来发起HTTPS请求的话，SNI将是IP地址，这有可能使服务器下发不恰当的证书。

TLS扩展相关方法的OptionalMethod创建过程也在AndroidPlatform中：
```
  public AndroidPlatform(Class<?> sslParametersClass, OptionalMethod<Socket> setUseSessionTickets,
      OptionalMethod<Socket> setHostname, OptionalMethod<Socket> getAlpnSelectedProtocol,
      OptionalMethod<Socket> setAlpnProtocols) {
    this.sslParametersClass = sslParametersClass;
    this.setUseSessionTickets = setUseSessionTickets;
    this.setHostname = setHostname;
    this.getAlpnSelectedProtocol = getAlpnSelectedProtocol;
    this.setAlpnProtocols = setAlpnProtocols;
  }

......

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

建立TLS连接的第7步，获取协议的过程与配置TLS的过程类似，同样利用反射调用SSLSocket的方法，在AndroidPlatform中：
```
  @Override public String getSelectedProtocol(SSLSocket socket) {
    if (getAlpnSelectedProtocol == null) return null;
    if (!getAlpnSelectedProtocol.isSupported(socket)) return null;

    byte[] alpnResult = (byte[]) getAlpnSelectedProtocol.invokeWithoutCheckedException(socket);
    return alpnResult != null ? new String(alpnResult, Util.UTF_8) : null;
  }
```

至此我们分析了OkHttp3中，所有HTTP请求，包括设置了代理的明文HTTP请求，设置了代理的HTTPS请求，设置了代理的HTTP/2请求，无代理的明文HTTP请求，无代理的HTTPS请求，无代理的HTTP/2请求的连接建立过程，其中包括TLS的握手，HTTP/2的协议协商等。

总结一下，OkHttp中，IO相关的组件的其关系大体如下图所示：

![Connection Component](https://www.wolfcstech.com/images/1315506-338a7a0b0a39a278.png)

Done。
