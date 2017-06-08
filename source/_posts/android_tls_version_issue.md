---
title: Android TLS 版本过低造成的问题
date: 2017-06-08 09:17:49
categories: 安全
tags:
- Android开发
- 安全
- HTTPS
---

由于集成的 SSL 库版本不同，不同 Android 版本的默认 SSL/TLS 版本配置如下表：

Protocol | Supported (API Levels)  |  Enabled by default (API Levels)
-------|----------|----------
SSLv3 | 1–TBD | 1–22
TLSv1 | 1+ |  1+
TLSv1.1 | 20+ | 20+
TLSv1.2 | 20+ |  20+

<!--more-->

Android 版本与 API Level 之间的对应关系则如下表：

Platform Version | API Level  |  VERSION_CODE | Notes
-------|----------|----------|----------
[Android 7.1.1 Android 7.1](https://developer.android.com/about/versions/nougat/android-7.1.html) |  [25](https://developer.android.com/sdk/api_diff/25/changes.html)  | [N_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#N_MR1) | [Platform Highlights](https://developer.android.com/about/versions/nougat/index.html)
[Android 7.0](https://developer.android.com/about/versions/nougat/android-7.0.html) | [24](https://developer.android.com/sdk/api_diff/24/changes.html) | [N](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#N) | [Platform Highlights](https://developer.android.com/about/versions/nougat/index.html)
[Android 6.0](https://developer.android.com/about/versions/marshmallow/android-6.0.html) | [23](https://developer.android.com/sdk/api_diff/23/changes.html) |  [M](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#M) | [Platform Highlights](https://developer.android.com/about/versions/marshmallow/index.html)
[Android 5.1](https://developer.android.com/about/versions/android-5.1.html) | [22](https://developer.android.com/sdk/api_diff/22/changes.html) | [LOLLIPOP_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#LOLLIPOP_MR1) | [Platform Highlights](https://developer.android.com/about/versions/lollipop.html)
[Android 5.0](https://developer.android.com/about/versions/android-5.0.html) | [21](https://developer.android.com/sdk/api_diff/21/changes.html) | [LOLLIPOP](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#LOLLIPOP) | [Platform Highlights](https://developer.android.com/about/versions/lollipop.html)
Android 4.4W |  [20](https://developer.android.com/sdk/api_diff/20/changes.html) | [KITKAT_WATCH](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#KITKAT_WATCH) | KitKat for Wearables Only
[Android 4.4](https://developer.android.com/about/versions/android-4.4.html) | [19](https://developer.android.com/sdk/api_diff/19/changes.html) | [KITKAT](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#KITKAT) | [Platform Highlights](https://developer.android.com/about/versions/kitkat.html)
[Android 4.3](https://developer.android.com/about/versions/android-4.3.html) | [18](https://developer.android.com/sdk/api_diff/18/changes.html) | [JELLY_BEAN_MR2](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#JELLY_BEAN_MR2) | [Platform Highlights](https://developer.android.com/about/versions/jelly-bean.html)
[Android 4.2, 4.2.2](https://developer.android.com/about/versions/android-4.2.html) | [17](https://developer.android.com/sdk/api_diff/17/changes.html) | [JELLY_BEAN_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#JELLY_BEAN_MR1) | [Platform Highlights](https://developer.android.com/about/versions/jelly-bean.html#android-42)
[Android 4.1, 4.1.1](https://developer.android.com/about/versions/android-4.1.html) | [16](https://developer.android.com/sdk/api_diff/16/changes.html) | [JELLY_BEAN](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#JELLY_BEAN) | [Platform Highlights](https://developer.android.com/about/versions/jelly-bean.html#android-41)
[Android 4.0.3, 4.0.4](https://developer.android.com/about/versions/android-4.0.3.html) | [15](https://developer.android.com/sdk/api_diff/15/changes.html) | [ICE_CREAM_SANDWICH_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#ICE_CREAM_SANDWICH_MR1) | [Platform Highlights](https://developer.android.com/about/versions/android-4.0-highlights.html)
[Android 4.0, 4.0.1, 4.0.2](https://developer.android.com/about/versions/android-4.0.html) | [14](https://developer.android.com/sdk/api_diff/14/changes.html) | [ICE_CREAM_SANDWICH](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#ICE_CREAM_SANDWICH) | [Platform Highlights](https://developer.android.com/about/versions/android-4.0-highlights.html)
[Android 3.2](https://developer.android.com/about/versions/android-3.2.html) | [13](https://developer.android.com/sdk/api_diff/13/changes.html) | [HONEYCOMB_MR2](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#HONEYCOMB_MR2) | 
[Android 3.1.x](https://developer.android.com/about/versions/android-3.1.html) | [12](https://developer.android.com/sdk/api_diff/12/changes.html) | [HONEYCOMB_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#HONEYCOMB_MR1) | [Platform Highlights](https://developer.android.com/about/versions/android-3.1-highlights.html)
[Android 3.0.x](https://developer.android.com/about/versions/android-3.0.html) | [11](https://developer.android.com/sdk/api_diff/11/changes.html) |  [HONEYCOMB](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#HONEYCOMB) | [Platform Highlights](https://developer.android.com/about/versions/android-3.0-highlights.html)
[Android 2.3.4 Android 2.3.3](https://developer.android.com/about/versions/android-2.3.3.html) |  [10](https://developer.android.com/sdk/api_diff/10/changes.html) | [GINGERBREAD_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#GINGERBREAD_MR1) | [Platform Highlights](https://developer.android.com/about/versions/android-2.3-highlights.html)
[Android 2.3.2 Android 2.3.1 Android 2.3](https://developer.android.com/about/versions/android-2.3.html) | [9](https://developer.android.com/sdk/api_diff/9/changes.html) | [GINGERBREAD](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#GINGERBREAD) | [Platform Highlights](https://developer.android.com/about/versions/android-2.3-highlights.html)
[Android 2.2.x](https://developer.android.com/about/versions/android-2.2.html) | [8](https://developer.android.com/sdk/api_diff/8/changes.html) | [FROYO](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#FROYO) | [Platform Highlights](https://developer.android.com/about/versions/android-2.2-highlights.html) 
[Android 2.1.x](https://developer.android.com/about/versions/android-2.1.html) | [7](https://developer.android.com/sdk/api_diff/7/changes.html) | [ECLAIR_MR1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#ECLAIR_MR1) | [Platform Highlights](https://developer.android.com/about/versions/android-2.0-highlights.html)
[Android 2.0.1](https://developer.android.com/about/versions/android-2.0.1.html) | [6](https://developer.android.com/sdk/api_diff/6/changes.html) | [ECLAIR_0_1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#ECLAIR_0_1) |  [Platform Highlights](https://developer.android.com/about/versions/android-2.0-highlights.html)
[Android 2.0](https://developer.android.com/about/versions/android-2.0.html) | [5](https://developer.android.com/sdk/api_diff/5/changes.html) | [ECLAIR](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#ECLAIR) |  [Platform Highlights](https://developer.android.com/about/versions/android-2.0-highlights.html)
[Android 1.6](https://developer.android.com/about/versions/android-1.6.html) | [4](https://developer.android.com/sdk/api_diff/4/changes.html) | [DONUT](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#DONUT) | [Platform Highlights](https://developer.android.com/about/versions/android-1.6-highlights.html)
[Android 1.5](https://developer.android.com/about/versions/android-1.5.html) | [3](https://developer.android.com/sdk/api_diff/3/changes.html) | [CUPCAKE](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#CUPCAKE) | [Platform Highlights](https://developer.android.com/about/versions/android-1.5-highlights.html)
[Android 1.1](https://developer.android.com/about/versions/android-1.1.html) | 2 | [BASE_1_1](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#BASE_1_1) | 
Android 1.0 | 1 | [BASE](https://developer.android.com/reference/android/os/Build.VERSION_CODES.html#BASE) |

可穿戴设备的 KitKat Android 4.4W 占用了 API Leve 20。也就是说，对于 Android 手机/平板 等设备，在 Android 5.0 之前，最高支持的 SSL/TLS 版本为 TLSv1，而这个版本实际上是一个安全性并不是太好的版本，当前已经有许多网站配置为不再支持这种老版本的 SSL/TLS 。在用 Android 5.0 之前的设备，访问一些不再支持 TLSv1 及之前 SSL/TLS 版本的网站的时候，就会出现一些问题。

Android 4.4.4 的设备与支持 TLSv1 的 HTTP 服务器建立连接时，其  ***Client Hello***  消息如下：

![](https://www.wolfcstech.com/images/1315506-a6f3bf43f81f004a.png)
 
SSL/TLS 握手可以正常完成，连接能够建立成功，并顺利完成数据传输。

Android 4.4.4 的设备与最低支持 TLSv1.1 的 HTTP 服务器建立连接时，其网络包的交互则像下面这样：

![](https://www.wolfcstech.com/images/1315506-4c6f2e534be8ae30.png)

第 4 号包是 TCP 连接建立完成后， Android 客户端发向服务器的 ***Client Hello*** 消息，但服务器立即就回了一个 ***Alert*** 消息，并关闭了 TCP 连接。此时，用于执行 HTTP 请求的 OkHttp 库也将抛出异常并退出，异常为：
```
javax.net.ssl.SSLHandshakeException: javax.net.ssl.SSLProtocolException: SSL handshake aborted: ssl=0xb7f24d78: Failure in SSL library, usually a protocol error
error:1407742E:SSL routines:SSL23_GET_SERVER_HELLO:tlsv1 alert protocol version (external/openssl/ssl/s23_clnt.c:741 0x96ff8926:0x00000000)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:448)
    at okhttp3.internal.connection.RealConnection.connectTls(RealConnection.java:267)
    at okhttp3.internal.connection.RealConnection.establishProtocol(RealConnection.java:237)
    at okhttp3.internal.connection.RealConnection.connect(RealConnection.java:148)
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
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1112)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:587)
    at java.lang.Thread.run(Thread.java:841)
	Suppressed: javax.net.ssl.SSLHandshakeException: javax.net.ssl.SSLProtocolException: SSL handshake aborted: ssl=0xb7f24d78: Failure in SSL library, usually a protocol error
error:1407742E:SSL routines:SSL23_GET_SERVER_HELLO:tlsv1 alert protocol version (external/openssl/ssl/s23_clnt.c:741 0x96ff8926:0x00000000)
		... 27 more
	Caused by: javax.net.ssl.SSLProtocolException: SSL handshake aborted: ssl=0xb7f24d78: Failure in SSL library, usually a protocol error
error:1407742E:SSL routines:SSL23_GET_SERVER_HELLO:tlsv1 alert protocol version (external/openssl/ssl/s23_clnt.c:741 0x96ff8926:0x00000000)
    at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:405)
		... 26 more
Caused by: javax.net.ssl.SSLProtocolException: SSL handshake aborted: ssl=0xb7f24d78: Failure in SSL library, usually a protocol error
error:1407742E:SSL routines:SSL23_GET_SERVER_HELLO:tlsv1 alert protocol version (external/openssl/ssl/s23_clnt.c:741 0x96ff8926:0x00000000)
    at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:405)
	... 26 more
```

在 Android 4.3、Android 4.2.2 和 Android 4.1.1 上，用相同版本的 OkHttp 访问前面配置为最低支持 TLSv1.1 的 HTTP 服务器时，抛出了完全相同的异常。

客户端版本低于服务器支持的最低 SSL/TLS 版本时，HTTPS 连接会在握手阶段失败。

参考文档
[<uses-sdk>](https://developer.android.com/guide/topics/manifest/uses-sdk-element.html)
[SSLEngine](https://developer.android.com/reference/javax/net/ssl/SSLEngine.html)