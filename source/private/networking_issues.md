注：OkHttp 代码基于 OkHttp 3.4 分析。Android 代码基于 Android 6.0.0_r26 分析。

# UnknownHostException

**错误说明：** 域名解析失败。

**错误消息：** Unable to resolve host "m9.music.126.net": No address associated with hostname

**可能原因：**

* 网络断开；
* DNS 服务器意外挂掉；
* DNS 服务器故障。

DNS 服务器挂掉或者故障这种问题比较少见，然而之前确实发生过大范围的 DNS 服务器问题。

针对这些原因，我们可以做一些模拟测试。

## 网络断开验证

对于网络连接断开的情况，我们采用如下的方法来模拟测试：

* 关闭手机的 WiFi 和移动网络，也就是使手机处于完全断网的情况；
* 执行一个 HTTP 请求。

以 OkHttp 为例，在请求开始之后，立即就报出了如下的异常：
```
java.net.UnknownHostException: Unable to resolve host "www.wolfcstech.com": No address associated with hostname
    at java.net.InetAddress.lookupHostByName(InetAddress.java:470)
    at java.net.InetAddress.getAllByNameImpl(InetAddress.java:252)
    at java.net.InetAddress.getAllByName(InetAddress.java:215)
    at okhttp3.Dns$1.lookup(Dns.java:39)
    at com.netease.netlib.OkHttp3Utils$MyDns.lookup(OkHttp3Utils.java:45)
    at okhttp3.internal.connection.RouteSelector.resetNextInetSocketAddress(RouteSelector.java:170)
    at okhttp3.internal.connection.RouteSelector.nextProxy(RouteSelector.java:136)
    at okhttp3.internal.connection.RouteSelector.next(RouteSelector.java:81)
    at okhttp3.internal.connection.StreamAllocation.findConnection(StreamAllocation.java:171)
    at okhttp3.internal.connection.StreamAllocation.findHealthyConnection(StreamAllocation.java:121)
    at okhttp3.internal.connection.StreamAllocation.newStream(StreamAllocation.java:100)
    at okhttp3.internal.connection.ConnectInterceptor.intercept(ConnectInterceptor.java:42)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.cache.CacheInterceptor.intercept(CacheInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.http.BridgeInterceptor.intercept(BridgeInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RetryAndFollowUpInterceptor.intercept(RetryAndFollowUpInterceptor.java:120)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at com.netease.netlib.OkHttp3Utils$MyInterceptor.intercept(OkHttp3Utils.java:29)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:179)
    at okhttp3.RealCall$AsyncCall.execute(RealCall.java:129)
    at okhttp3.internal.NamedRunnable.run(NamedRunnable.java:32)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:833)
Caused by: android.system.GaiException: android_getaddrinfo failed: EAI_NODATA (No address associated with hostname)
    at libcore.io.Posix.android_getaddrinfo(Native Method)
    at libcore.io.ForwardingOs.android_getaddrinfo(ForwardingOs.java:55)
    at java.net.InetAddress.lookupHostByName(InetAddress.java:451)
	... 30 more
```

## DNS 服务器意外挂掉验证

对于DNS 服务器意外挂掉的情况，我们采用如下的方法模拟测试：

* 连接 WiFi，设置手机的 IP 地址为静态 IP 地址，不修改之前 DHCP 分配的 IP 地址，但设置 DNS 服务器地址为无效的地址；
* 执行一个 HTTP 请求。

以 OkHttp 为例，在请求开始之后，将报出如下的异常：
```
java.net.UnknownHostException: Unable to resolve host "www.wolfcstech.com": No address associated with hostname
    at java.net.InetAddress.lookupHostByName(InetAddress.java:470)
    at java.net.InetAddress.getAllByNameImpl(InetAddress.java:252)
    at java.net.InetAddress.getAllByName(InetAddress.java:215)
    at okhttp3.Dns$1.lookup(Dns.java:39)
    at com.netease.netlib.OkHttp3Utils$MyDns.lookup(OkHttp3Utils.java:45)
    at okhttp3.internal.connection.RouteSelector.resetNextInetSocketAddress(RouteSelector.java:170)
    at okhttp3.internal.connection.RouteSelector.nextProxy(RouteSelector.java:136)
    at okhttp3.internal.connection.RouteSelector.next(RouteSelector.java:81)
    at okhttp3.internal.connection.StreamAllocation.findConnection(StreamAllocation.java:171)
    at okhttp3.internal.connection.StreamAllocation.findHealthyConnection(StreamAllocation.java:121)
    at okhttp3.internal.connection.StreamAllocation.newStream(StreamAllocation.java:100)
    at okhttp3.internal.connection.ConnectInterceptor.intercept(ConnectInterceptor.java:42)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.cache.CacheInterceptor.intercept(CacheInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.http.BridgeInterceptor.intercept(BridgeInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RetryAndFollowUpInterceptor.intercept(RetryAndFollowUpInterceptor.java:120)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at com.netease.netlib.OkHttp3Utils$MyInterceptor.intercept(OkHttp3Utils.java:29)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:179)
    at okhttp3.RealCall$AsyncCall.execute(RealCall.java:129)
    at okhttp3.internal.NamedRunnable.run(NamedRunnable.java:32)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:833)
Caused by: android.system.GaiException: android_getaddrinfo failed: EAI_NODATA (No address associated with hostname)
    at libcore.io.Posix.android_getaddrinfo(Native Method)
    at libcore.io.ForwardingOs.android_getaddrinfo(ForwardingOs.java:55)
    at java.net.InetAddress.lookupHostByName(InetAddress.java:451)
	... 30 more
```
这个异常与网络断开情况下报出的异常一模一样，然而这一次从请求开始执行到异常报出，经历的时间则要长得多，如我们上面看到的，这段时间长达 40 s。

## DNS 服务器故障验证

DNS 服务器故障主要是指 DNS 服务器确实查不到所请求的域名，可能是 DNS 服务器出了问题，也可能是为域名做的 DNS 配置还没有生效等。

对于这种情况，我们采用如下方法模拟测试：

* 手机正常连接网络；
* 以一个不存在的域名执行一个 HTTP 请求。

请求执行之后，将报出如下异常：
```
java.net.UnknownHostException: Unable to resolve host "www.wolfcsteach.com": No address associated with hostname
    at java.net.InetAddress.lookupHostByName(InetAddress.java:470)
    at java.net.InetAddress.getAllByNameImpl(InetAddress.java:252)
    at java.net.InetAddress.getAllByName(InetAddress.java:215)
    at okhttp3.Dns$1.lookup(Dns.java:39)
    at com.netease.netlib.OkHttp3Utils$MyDns.lookup(OkHttp3Utils.java:45)
    at okhttp3.internal.connection.RouteSelector.resetNextInetSocketAddress(RouteSelector.java:170)
    at okhttp3.internal.connection.RouteSelector.nextProxy(RouteSelector.java:136)
    at okhttp3.internal.connection.RouteSelector.next(RouteSelector.java:81)
    at okhttp3.internal.connection.StreamAllocation.findConnection(StreamAllocation.java:171)
    at okhttp3.internal.connection.StreamAllocation.findHealthyConnection(StreamAllocation.java:121)
    at okhttp3.internal.connection.StreamAllocation.newStream(StreamAllocation.java:100)
    at okhttp3.internal.connection.ConnectInterceptor.intercept(ConnectInterceptor.java:42)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.cache.CacheInterceptor.intercept(CacheInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.http.BridgeInterceptor.intercept(BridgeInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RetryAndFollowUpInterceptor.intercept(RetryAndFollowUpInterceptor.java:120)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at com.netease.netlib.OkHttp3Utils$MyInterceptor.intercept(OkHttp3Utils.java:29)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:179)
    at okhttp3.RealCall$AsyncCall.execute(RealCall.java:129)
    at okhttp3.internal.NamedRunnable.run(NamedRunnable.java:32)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:833)
Caused by: android.system.GaiException: android_getaddrinfo failed: EAI_NODATA (No address associated with hostname)
    at libcore.io.Posix.android_getaddrinfo(Native Method)
    at libcore.io.ForwardingOs.android_getaddrinfo(ForwardingOs.java:55)
    at java.net.InetAddress.lookupHostByName(InetAddress.java:451)
	... 30 more
```

可以看到，在这种情况下，异常报出与请求开始执行之间的时间差也非常小。如上面的例子，这个时间差只有 25 ms，甚至比网络断开情况下的 112 ms 还要短。

## 所需诊断信息

依据上面的测试分析，诊断域名解析问题，需要如下诊断信息：

* 请求开始时间；
* 异常报出时间；
* 网络是否连接；
* DNS 服务器 IP 地址列表；
* DNS 服务器是否可用（ping DNS 服务器 IP地址的结果）；
* DNS 的结果（nslookup 结果）。

# ConnectTimeoutException

**等价异常类型：** 
* org.apache.http.conn.ConnectTimeoutException
* com.netease.mam.org.apache.http.conn.ConnectTimeoutException

**错误说明：** 连接超时

**错误消息：** Connect to /183.214.133.45:443 timed out

**异常分析：**

异常在 `external/apache-http/src/org/apache/http/conn/scheme/PlainSocketFactory.java` 的 `connectSocket()` 中，apache httpclient 抛出，由 `Socket.connect()` 中抛出的 `SocketTimeoutException` 引起。

**可能原因：**

* 设备接入的网络本身带宽比较低；
* 设备接入的网络本身延迟比较高；
* 设备与服务器的网络路径中存在比较拥堵、负载比较重的节点；
* 网络中路由节点的临时性异常。

**所需诊断信息：**

* Traceroute 获取的网络路径信息。
* 网络路径上不同节点的繁忙程度，可通过丢包率来近似地反映。
* 网络连接条件（移动网络/WiFI），网络延时及带宽。

# SocketTimeoutException

**异常：** java.net.SocketTimeoutException

**错误说明：** socket 超时

**异常分析：**

在 OkHttp 处理 HTTP/2 的逻辑 (`okhttp3.internal.http2.Http2Stream`) 中，会由于读超时、写超时而抛出该异常。
在 Android 中，`java.net.PlainSocketImpl` 的 `accept(SocketImpl newImpl)` 执行失败，如果 `errno` 为 `EAGAIN` 将抛出该异常，如果使用 nio 的 `ServerSocketChannelImpl`，异常将不会被实际抛出；此外，在阻塞 Socket 上读取长度为 0 的数据时抛出此异常。
`libcore.io.IoBridge` 中，TCP 连接建立超时，将抛出该异常。`libcore.io.IoBridge` 中，接收数据失败，会由于 `errno` 为 `EAGAIN` 抛出该异常。
在 `external/conscrypt/src/main/native/org_conscrypt_NativeCrypto.cpp` 中执行 SSL/TLS 握手动作、数据读操作或数据写操作超时，会抛出该异常。

## 子错误 - 读超时

**错误消息：** Read timed out

**异常分析：**

在 `external/conscrypt/src/main/native/org_conscrypt_NativeCrypto.cpp` 中执行 SSL/TLS 数据读操作超时，抛出该异常。

**可能原因：**

* 设备接入的网络本身带宽比较低；
* 设备接入的网络本身延迟比较高；
* 设备与服务器的网络路径中存在比较拥堵、负载比较重的节点；
* 网络中路由节点的临时性异常。

**所需诊断信息：**

* Traceroute 获取的网络路径信息。
* 网络路径上不同节点的繁忙程度，可通过丢包率来近似地反映。
* 网络连接条件（移动网络/WiFI），网络延时及带宽。

## 子错误 - SSL 握手超时

**错误消息：** SSL handshake timed out

**异常分析：**

在 `external/conscrypt/src/main/native/org_conscrypt_NativeCrypto.cpp` 中执行 SSL/TLS 握手动作超时，抛出该异常。

**可能原因：**

* 设备接入的网络本身带宽比较低；
* 设备接入的网络本身延迟比较高；
* 设备与服务器的网络路径中存在比较拥堵、负载比较重的节点；
* 网络中路由节点的临时性异常。

**所需诊断信息：**

* Traceroute 获取的网络路径信息。
* 网络路径上不同节点的繁忙程度，可通过丢包率来近似地反映。
* 网络连接条件（移动网络/WiFI），网络延时及带宽。

## 子错误 - 未知原因

**错误消息：** null

**异常分析：**

根据 `SocketTimeoutException` 抛出的所有情况综合来看，这种异常主要由 HTTP 请求读操作执行超时引起。

**可能原因：**

* 设备接入的网络本身带宽比较低；
* 设备接入的网络本身延迟比较高；
* 设备与服务器的网络路径中存在比较拥堵、负载比较重的节点；
* 网络中路由节点的临时性异常。

**所需诊断信息：**

* 更加完整的异常堆栈
* Traceroute 获取的网络路径信息。
* 网络路径上不同节点的繁忙程度，可通过丢包率来近似地反映。
* 网络连接条件（移动网络/WiFI），网络延时及带宽。

# HttpHostConnectException

**等价异常类型：**
* org.apache.http.conn.HttpHostConnectException
* com.netease.mam.org.apache.http.conn.HttpHostConnectException

**错误说明：** 客户端的数据包可以到达目标主机，但由于各种原因，连接建立失败的问题。

**错误消息：** Connection to http://m7.music.126.net refused

**可能原因：** 

* 连接的目标主机没有开对应的端口，可能服务器发生故障
* 客户端设置了代理，而代理进程并没有跑起来。

针对这些原因，我们也可以做一些模拟和测试。

## 服务器故障验证

对于这种情况，我们采用如下的方法来模拟测试，

* 指定我们执行 HTTP 请求时访问的端口为一个无效的端口，如使用 [URL](https://www.wolfcstech.com:8080/2017/04/13/%E5%88%9D%E5%A7%8BDNS%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%9C%B0%E5%9D%80%E6%98%AF%E5%93%AA%E9%87%8C%E6%9D%A5%E7%9A%84/) ；
* 以 **HttpClient** 作为我们的 HttpStack 来执行 HTTP 请求。

请求执行之后，将报出如下异常：
```
org.apache.http.conn.HttpHostConnectException: Connection to https://www.wolfcstech.com:8080 refused
    at org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:193)
    at org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:169)
    at org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:124)
    at org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:366)
    at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:596)
    at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:517)
    at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:495)
    at com.netease.volleydemo.HttpClientUtils.httpGet(HttpClientUtils.java:21)
    at com.netease.volleydemo.MainActivity$HttpClientTask.doInBackground(MainActivity.java:152)
    at com.netease.volleydemo.MainActivity$HttpClientTask.doInBackground(MainActivity.java:142)
    at android.os.AsyncTask$2.call(AsyncTask.java:307)
    at java.util.concurrent.FutureTask.run(FutureTask.java:237)
    at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:246)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:833)
Caused by: java.net.ConnectException: failed to connect to /139.196.224.72 (port 8080): connect failed: ECONNREFUSED (Connection refused)
    at libcore.io.IoBridge.connect(IoBridge.java:124)
    at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:183)
    at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:452)
    at java.net.Socket.connect(Socket.java:938)
    at org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:124)
    at org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:149)
	... 15 more
Caused by: android.system.ErrnoException: connect failed: ECONNREFUSED (Connection refused)
04-27 14:45:44.822    at libcore.io.Posix.connect(Native Method)
    at libcore.io.BlockGuardOs.connect(BlockGuardOs.java:111)
    at libcore.io.IoBridge.connectErrno(IoBridge.java:137)
    at libcore.io.IoBridge.connect(IoBridge.java:122)
	... 20 more
```
`org.apache.http.conn.HttpHostConnectException` 的 errorMessage 向我们指出是服务器拒绝连接。

对于同样的问题，如果以 OkHttp 作为 HttpStack 来执行请求的话，则将报出稍有不同的异常：
```
java.net.ConnectException: Failed to connect to www.wolfcstech.com/139.196.224.72:8080
    at okhttp3.internal.connection.RealConnection.connectSocket(RealConnection.java:222)
    at okhttp3.internal.connection.RealConnection.connect(RealConnection.java:146)
    at okhttp3.internal.connection.StreamAllocation.findConnection(StreamAllocation.java:186)
    at okhttp3.internal.connection.StreamAllocation.findHealthyConnection(StreamAllocation.java:121)
    at okhttp3.internal.connection.StreamAllocation.newStream(StreamAllocation.java:100)
    at okhttp3.internal.connection.ConnectInterceptor.intercept(ConnectInterceptor.java:42)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.cache.CacheInterceptor.intercept(CacheInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.http.BridgeInterceptor.intercept(BridgeInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RetryAndFollowUpInterceptor.intercept(RetryAndFollowUpInterceptor.java:120)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at com.netease.netlib.OkHttp3Utils$MyInterceptor.intercept(OkHttp3Utils.java:29)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:179)
    at okhttp3.RealCall$AsyncCall.execute(RealCall.java:129)
    at okhttp3.internal.NamedRunnable.run(NamedRunnable.java:32)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:833)
Caused by: java.net.ConnectException: failed to connect to www.wolfcstech.com/139.196.224.72 (port 8080) after 10000ms: isConnected failed: ECONNREFUSED (Connection refused)
    at libcore.io.IoBridge.isConnected(IoBridge.java:234)
    at libcore.io.IoBridge.connectErrno(IoBridge.java:171)
    at libcore.io.IoBridge.connect(IoBridge.java:122)
    at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:183)
    at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:452)
    at java.net.Socket.connect(Socket.java:938)
    at okhttp3.internal.platform.AndroidPlatform.connectSocket(AndroidPlatform.java:63)
    at okhttp3.internal.connection.RealConnection.connectSocket(RealConnection.java:220)
	... 24 more
Caused by: android.system.ErrnoException: isConnected failed: ECONNREFUSED (Connection refused)
    at libcore.io.IoBridge.isConnected(IoBridge.java:223)
	... 31 more
```

OkHttp 抛出 `java.net.ConnectException`，仅仅简单地指出连接服务器失败。

## 代理服务器故障验证
这主要是指代理服务器进程没有启动，或代理设置存在问题。

对于这种情况，我们通过如下的方法来模拟测试：

* 使手机连接 WiFi；
* 为手机所连接的 WiFi 设备设置代理，其中代理服务器的地址为无效的 IP 地址，或端口为无效端口；
* 以 **HttpClient** 作为我们的 HttpStack 来执行 HTTP 请求。

请求执行之后，将报出如下异常：
```
org.apache.http.conn.HttpHostConnectException: Connection to http://10.240.252.44:8888 refused
     at org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:193)
     at org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:169)
     at org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:124)
     at org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:366)
     at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:596)
     at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:517)
     at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:495)
     at com.netease.volleydemo.HttpClientUtils.httpGet(HttpClientUtils.java:21)
     at com.netease.volleydemo.MainActivity$HttpClientTask.doInBackground(MainActivity.java:152)
     at com.netease.volleydemo.MainActivity$HttpClientTask.doInBackground(MainActivity.java:142)
     at android.os.AsyncTask$2.call(AsyncTask.java:307)
     at java.util.concurrent.FutureTask.run(FutureTask.java:237)
     at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:246)
     at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
     at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
     at java.lang.Thread.run(Thread.java:833)
 Caused by: java.net.ConnectException: failed to connect to /10.240.252.44 (port 8888): connect failed: ECONNREFUSED (Connection refused)
     at libcore.io.IoBridge.connect(IoBridge.java:124)
     at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:183)
     at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:452)
     at java.net.Socket.connect(Socket.java:938)
     at org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:124)
     at org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:149)
 	... 15 more
 Caused by: android.system.ErrnoException: connect failed: ECONNREFUSED (Connection refused)
     at libcore.io.Posix.connect(Native Method)
     at libcore.io.BlockGuardOs.connect(BlockGuardOs.java:111)
     at libcore.io.IoBridge.connectErrno(IoBridge.java:137)
     at libcore.io.IoBridge.connect(IoBridge.java:122)
 	... 20 more
```

对于同样的问题，OkHttp 报出了不同的异常：
```
java.net.ConnectException: Failed to connect to /10.240.252.44:8888
    at okhttp3.internal.connection.RealConnection.connectSocket(RealConnection.java:222)
    at okhttp3.internal.connection.RealConnection.connectTunnel(RealConnection.java:195)
    at okhttp3.internal.connection.RealConnection.connect(RealConnection.java:144)
    at okhttp3.internal.connection.StreamAllocation.findConnection(StreamAllocation.java:186)
    at okhttp3.internal.connection.StreamAllocation.findHealthyConnection(StreamAllocation.java:121)
    at okhttp3.internal.connection.StreamAllocation.newStream(StreamAllocation.java:100)
    at okhttp3.internal.connection.ConnectInterceptor.intercept(ConnectInterceptor.java:42)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.cache.CacheInterceptor.intercept(CacheInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.internal.http.BridgeInterceptor.intercept(BridgeInterceptor.java:93)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RetryAndFollowUpInterceptor.intercept(RetryAndFollowUpInterceptor.java:120)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at com.netease.netlib.OkHttp3Utils$MyInterceptor.intercept(OkHttp3Utils.java:29)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:92)
    at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:67)
    at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:179)
    at okhttp3.RealCall$AsyncCall.execute(RealCall.java:129)
    at okhttp3.internal.NamedRunnable.run(NamedRunnable.java:32)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:833)
Caused by: java.net.ConnectException: failed to connect to /10.240.252.44 (port 8888) after 10000ms: isConnected failed: ECONNREFUSED (Connection refused)
    at libcore.io.IoBridge.isConnected(IoBridge.java:234)
    at libcore.io.IoBridge.connectErrno(IoBridge.java:171)
```

## 所需诊断信息

为了确认所报出的异常产生的根源，需要下的诊断信息：

* 连接的目标主机的 IP 地址；
* 连接建立的目标端口；
* 用户设置的代理服务器地址和端口；
* TCP 连接目标 IP:端口 的结果。

## 参考资料

针对这个异常，网络上有其它的一些资料和讨论，相关链接如下：

* [Stack Overflow 讨论](http://stackoverflow.com/questions/15546589/what-causes-httphostconnectexception)
* [Android 5.0 抛出异常的相关源码](http://androidxref.com/5.0.0_r2/xref/external/apache-http/src/org/apache/http/impl/conn/DefaultClientConnectionOperator.java)
* [Android 官方文档对该异常的说明](https://developer.android.com/reference/org/apache/http/conn/HttpHostConnectException.html)

# NoRouteToHostException

**异常：** java.net.NoRouteToHostException

**错误说明：** 无法连接远程地址与端口。

**错误消息：** No route to host

**可能原因：** 

* 防火墙的规则设置导致数据包无法被发送出去；
* 中间路由节点挂掉。

**所需诊断信息：**

* 用户防火墙配置；
* 目标主机是否可以访问（ping）。

**参考资料：**

* [Stack Overflow 的讨论](http://stackoverflow.com/questions/15686445/noroutetohostexception-on-client-or-server)
* [Android 官方对异常的说明](https://developer.android.com/reference/java/net/NoRouteToHostException.html)

# SSLException

**异常：** javax.net.ssl.SSLException

**错误说明：** SSL 失败

## 子错误 - SSL 读期间连接重置

**错误消息：** Read error: ssl=0xf49f7200: I/O error during system call, Connection reset by peer

**可能原因：** 

* 在网络数据传输期间，TCP 连接被服务器异常结束。
* 遭受网络 TCP Reset 攻击。

**异常分析：**

在网络数据传输期间，TCP 连接意外结束，由于服务器进程意外终止等原因，没有经过正常的 TCP 连接断开过程。后续客户端向服务器发送 TCP 包，如数据包、ACK 包等，服务器返回 RST 包所致。

**所需诊断信息：**

 * 与目标服务器目标端口的 TCP 连接测试结果，检查目标服务器进程是否存活。

**参考资料：**

* [从TCP协议的原理来谈谈rst复位攻击](http://www.2cto.com/article/201202/118625.html)

## 子错误 - SSL 握手期间连接重置

**错误消息：** SSL handshake aborted: ssl=0x79c484c0: I/O error during system call, Connection reset by peer

**可能原因：**

* 在网络数据传输期间，TCP 连接被服务器异常结束。
* 遭受网络 TCP Reset 攻击。

**异常分析：**

以 Android 6.0.0_r1 为例，仅有的出现 "SSL handshake aborted" 的位置为 http://androidxref.com/6.0.0_r1/xref/external/conscrypt/src/main/native/org_conscrypt_NativeCrypto.cpp --- NativeCrypto_SSL_do_handshake()。

异常的 error message 构造过程为 NativeCrypto_SSL_do_handshake() -> throwSSLExceptionWithSslErrors() -> asprintf() （error code 为 SSL_ERROR_SYSCALL ）。

errorCode 由 SSL_do_handshake(SSL *s) 返回，但在 SSL_do_handshake(SSL *s) ，实际由 `SSL *s` 的回调函数 `s->handshake_func` 执行。handshake_func 在两个地方赋值，对于服务器而言，在 SSL_set_accept_state() 中，该回调等于 ssl->method->ssl_accept；对于客户端而言，该回调在 SSL_set_connect_state() 中赋值，等于 ssl->method->ssl_connect。 `SSL *s` 结构的 `SSL_PROTOCOL_METHOD *method`，在 SSL 结构创建时赋值，SSL 结构的创建主要在 `NativeCrypto_SSL_new()` -> `SSL_new()` 完成。`SSL *s` 结构的 `SSL_PROTOCOL_METHOD *method` 来源于 `SSL_CTX` 的 `method`。`SSL_CTX` 结构在 NativeCrypto_SSL_CTX_new() 中创建，`SSL_CTX` 结构的 `method` 来自于 `SSLv23_method()`，实际为 `TLS_method(void)` 函数中的静态结构，`protocol_method` 为 `TLS_protocol_method`。

最终可以发现 `SSL *s` 的回调函数 `s->handshake_func` 为 `ssl3_connect(SSL *s)`。在 `ssl3_connect(SSL *s)` 中，以一种状态机模式的方式，逐步完成 SSL/TLS 握手，在遇到错误时退出。

因而错误的发生，是由于在 `ssl3_connect(SSL *s)` 执行期间，TCP 连接意外结束，可能由于服务器进程意外终止等原因，没有经过正常的 TCP 连接断开过程。后续客户端向服务器发送 TCP 包，如数据包、ACK 包等，服务器返回 RST 包所致。但具体在 SSL/TLS 握手的哪个阶段异常发生，则难以判断。

**所需诊断信息：**

 * 与目标服务器目标端口的 TCP 连接测试结果，检查目标服务器进程是否存活。

# SSLHandshakeException

**异常：** javax.net.ssl.SSLHandshakeException

**错误说明：** SSL/TLS 握手失败

## 子错误 - 握手失败

**错误消息：** Handshake failed

**可能原因：**

* 握手失败，证书验证失败
 - 未知证书颁发者；
 - 不完整的证书链；
 - 自签名证书；
 - 服务器主机名不匹配；
 - 严格安全重新协商失败；
 - 遭遇了中间人攻击。
* 握手失败，协议协商失败/握手格式不兼容
* 协议异常，服务器名称标识不匹配
* 某些版本 SSL 库的 bug

**所需诊断信息：**

* 异常的详细错误消息；
* 客户端支持的 SSL/TLS 版本；
* 客户端设置的 SNI 信息；
* 服务器下发的证书链；
* 客户端可用的加密套件。

对于特定的操作系统版本，其 SSL 库的版本和根证书库是固定的，对于这些重要的信息不再需要移动端上传。此外，***证书链消耗流量可能会比较大，大概在几 KBytes***。

## 子错误 - 连接超时

**错误消息：** SSL handshake aborted: ssl=0xeec6d600: I/O error during system call, Connection timed out

**可能原因：**

* 设备接入的网络本身带宽比较低；
* 设备接入的网络本身延迟比较高；
* 设备与服务器的网络路径中存在比较拥堵、负载比较重的节点；
* 网络中路由节点的临时性异常。

**所需诊断信息：**

* Traceroute 获取的网络路径信息。
* 网络路径上不同节点的繁忙程度，可通过丢包率来近似地反映。
* 网络连接条件（移动网络/WiFI），网络延时及带宽。

## 子错误 - 连接重置

**错误消息：** SSL handshake aborted: ssl=0xea0e4f00: I/O error during system call, Connection reset by peer

**可能原因：** 

* 在网络数据传输期间，TCP 连接被服务器异常结束。
* 遭受网络 TCP Reset 攻击。

**错误原因分析：**

在网络数据传输期间，TCP 连接意外结束，可能由于服务器进程意外终止等原因，没有经过正常的 TCP 连接断开过程。后续客户端向服务器发送 TCP 包，如数据包、ACK 包等，服务器返回 RST 包所致。

**所需诊断信息：**

 * 与目标服务器目标端口的 TCP 连接测试结果，检查目标服务器进程是否存活。

## 子错误 - 连接被关闭

**错误消息：** Connection closed by peer

**可能原因：** 

* 在网络数据传输期间，TCP 连接被服务器异常结束。

**错误原因分析：**

在网络数据传输期间，由于服务器回收 TCP 连接，或服务器进程意外结束，连接被提前结束，正常的 TCP 连接断开过程完成所致。

**所需诊断信息：**

 * 与目标服务器目标端口的 TCP 连接测试结果，检查目标服务器进程是否存活。

# SSLProtocolException

**异常：** javax.net.ssl.SSLProtocolException

**错误说明：** 协议异常

## 子错误 - 记录MAC无效的读失败

**错误消息：** Read error: ssl=0xf475ee00: Failure in SSL library, usually a protocol error error:140943FC:SSL routines:SSL3_READ_BYTES:sslv3 alert bad record mac (external/openssl/ssl/s3_pkt.c:1308 0xe27a8ee0:0x00000003)

**可能原因：** 

* 数据在传输过程中被篡改。
* MAC 方式不支持。
* 某些版本SSL 库的 bug

**所需诊断信息：**

* 客户端支持的 SSL/TLS 版本；
* 客户端可用的加密套件。

## 子错误 - 协议版本导致的读失败

**错误消息：** Read error: ssl=0xdbab7300: Failure in SSL library, usually a protocol error error:100c542e:SSL routines:ssl3_read_bytes:TLSV1_ALERT_PROTOCOL_VERSION (external/boringssl/src/ssl/s3_pkt.c:972 0xd3fb2d20:0x00000001)

**可能原因：** 

* 遭遇降级攻击
* 某些版本 SSL 库的 bug

**所需诊断信息：** 

* 客户端支持的 SSL/TLS 版本；
* 服务器下发的证书链；
* 客户端可用的加密套件。

对于特定操作系统版本，其 SSL 库版本和根证书库是固定的，对于这些重要信息不再需要移动端上传。此外，***证书链消耗流量可能会比较大，大概在几 KBytes***。

# NoHttpResponseException

**等价异常类型：**
* org.apache.http.NoHttpResponseException
* com.netease.mam.org.apache.http.NoHttpResponseException

**错误说明：** 服务器响应异常

**错误消息：** The target server failed to respond

**可能原因：**

* 服务器故障

# InterruptedIOException

**异常：** java.io.InterruptedIOException

**错误说明：** IO 中断

## 子错误 - 连接已关闭

**错误消息：** Connection has been shut down. 

**异常分析：**

异常抛出的位置为 `external/apache-http/src/org/apache/http/impl/conn/AbstractClientConnAdapter.java` 的 `assertNotAborted()`。

**可能原因：**

* 客户端在发送请求或接收响应时，发现连接已经被终止，请求被终止/取消。

# IOException

**异常：** java.io.IOException

**错误说明：** IO 失败

## 子错误 - 连接关闭

**错误消息：** Connection already shutdown

**异常分析：**

异常抛出的位置为 `external/apache-http/src/org/apache/http/impl/conn/DefaultClientConnection.java` 的 `opening()` 方法。 

**可能原因：**

* 连接打开动作执行过程中，被客户端关闭，请求被终止/取消。

## 子错误 - 请求被终止

**错误消息：** Request already aborted

**异常分析：**

异常抛出的位置为 `external/apache-http/src/org/apache/http/client/methods/HttpRequestBase.java` 的 `setConnectionRequest()` 和 `setReleaseTrigger()` 中，设置连接请求和 `ConnectionReleaseTrigger` 时执行检查，发现请求被客户端终止。

**可能原因：**

* 请求被终止/取消。

# SocketException

**异常：** java.net.SocketException

**错误说明：** Socket 异常

## 子错误 - 读消息连接重置

**错误消息：** recvfrom failed: ECONNRESET (Connection reset by peer)

**可能原因：** 

* 在网络数据传输期间，TCP 连接被服务器异常结束。
* 遭受网络 TCP Reset 攻击。

**错误原因分析：**

在网络数据传输期间，TCP 连接意外结束，由于服务器进程意外终止等原因，没有经过正常的 TCP 连接断开过程。后续客户端向服务器发送 TCP 包，如数据包、ACK 包等，服务器返回 RST 包所致。

**所需诊断信息**

 * 与目标服务器目标端口的 TCP 连接测试结果，检查目标服务器进程是否存活。

## 子错误 - 连接重置

**错误消息：** Connection reset

**可能原因：** 

* 在网络数据传输期间，TCP 连接被服务器异常结束。
* 遭受网络 TCP Reset 攻击。

**错误原因分析：**

在网络数据传输期间，TCP 连接意外结束，可能由于服务器进程意外终止等原因，没有经过正常的 TCP 连接断开过程。后续客户端向服务器发送 TCP 包，如数据包、ACK 包等，服务器返回 RST 包所致。

**所需诊断信息**

 * 与目标服务器目标端口的 TCP 连接测试结果，检查目标服务器进程是否存活。

## 子错误 - Socket 关闭

**错误消息：** Socket closed

**异常分析：**

异常抛出的位置为 `libcore/luni/src/main/java/libcore/io/IoBridge.java` 的 `isConnected()`。

**可能原因：** 

* 连接被客户端关闭，请求被终止/取消。

# ConnectException

**异常：** java.net.ConnectException

**错误说明：** 连接失败

**异常分析：**

`ConnectException` 异常在 `libcore.io.IoBridge` 的 `connect(FileDescriptor fd, InetAddress inetAddress, int port, int timeoutMs)` 和 `isConnected()` 中抛出。在执行 socket 连接建立操作时，底层操作失败。错误信息中都会包含连接的目标主机的 IP，尝试连接的时间， `errno` 值及 `errno` 值对应的错误消息。通常可根据 errno 判断失败原因。

## 子错误 - 连接被拒绝

**错误消息：** failed to connect to /119.84.111.126 (port 80) after 30000ms: isConnected failed: ECONNREFUSED (Connection refused)

参考前面 `HttpHostConnectException` 相关说明。

## 子错误 - 主机不可达

**错误消息：** failed to connect to /153.101.65.23 (port 80) after 30000ms: isConnected failed: EHOSTUNREACH (No route to host)

**可能原因：**

* 防火墙的规则设置导致数据包无法被发送出去；
* 中间路由节点挂掉。

**异常分析：**

引发了ICMP目的不可达错误，这认为是软错误。client核心保存消息并且荏苒发送SYN。。如果超出一个确定的时间。保存的ICMP错误返回给进程 EHOSTUNREACH或者ENETUNREACH。ENETUNREACH被认为是过时的。所以返回的应该是EHOSTUNREACH。

**所需诊断信息：**

* 用户防火墙配置；
* 目标主机是否可以访问（ping）。

## 子错误 - 网络不可达

**错误消息：** failed to connect to /59.111.160.195 (port 80) after 30000ms: connect failed: ENETUNREACH (Network is unreachable)

**可能原因：**

* 防火墙的规则设置导致数据包无法被发送出去；
* 中间路由节点挂掉。

**所需诊断信息：**

* 用户防火墙配置；
* 目标主机是否可以访问（ping）。

# ClientProtocolException

**等价异常类型：** 
* com.netease.mam.org.apache.http.client.ClientProtocolException

**错误说明：** 协议错误

**异常分析：** 

异常抛出的位置为 `external/apache-http/src/org/apache/http/impl/client/AbstractHttpClient.java` 的 `execute()` 中。在执行 `RequestDirector.execute()` 时，捕获到 `HttpException` 异常时抛出该异常。 `HttpException` 异常抛出的位置主要有以下几个：
* `external/apache-http/src/org/apache/http/impl/client/DefaultRequestDirector.java` 的 `createTunnelToTarget()` 中通过 HTTP 的 CONNECT 方法建立隧道链接时，对端返回了小于 200 的响应码，隧道连接建立失败。
* `external/apache-http/src/org/apache/http/impl/conn/ProxySelectorRoutePlanner.java` 的 `determineProxy()` 中 URI 格式错误。URI 通过目标主机的 `scheme` ，主机名，端口等构造。
* `external/apache-http/src/org/apache/http/impl/conn/ProxySelectorRoutePlanner.java` 的 `determineProxy()` 中代理地址格式错误。

## 子错误 - 未知原因

**错误消息：** null

**可能原因：**

* 这个错误主要与客户端的 HTTP 代理服务器设置有关。

**所需诊断信息：**

* 如果是 HTTPS 请求，通过 HTTP 的 CONNECT 方法建立隧道链接时，CONNECT 请求的完整响应内容。
* 代理服务器配置。

Done。