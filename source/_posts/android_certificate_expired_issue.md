---
title: Android 的 HTTPS 证书过期异常
date: 2017-06-08 15:17:49
categories: 安全
tags:
- Android开发
- 安全
- HTTPS
---

对于  HTTPS 服务器证书过期的问题，由于 Android 安全库的不断更新，尽管在证书验证的时候抛出的异常大同小异，但还是有一定的区别的。某些系统之间抛出的异常完全相同，另外的一些系统之间，抛出的异常则不太一样。

<!--more-->

## Android 4.1.1 、4.2.2、4.3
在 Android 4.1.1 、4.2.2、4.3 平台上，使用 OkHttp 3.4，访问一个 HTTPS 证书过期的网站时，OkHttp 将报出如下的异常：

```
javax.net.ssl.SSLHandshakeException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: current time: Wed Jun 07 23:08:11 EDT 2017, expiration time: Sun Jan 22 20:12:00 EST 2017
    at org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:401)
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
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1080)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:573)
    at java.lang.Thread.run(Thread.java:841)
Caused by: java.security.cert.CertificateException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: current time: Wed Jun 07 23:08:11 EDT 2017, expiration time: Sun Jan 22 20:12:00 EST 2017
    at org.apache.harmony.xnet.provider.jsse.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:308)
    at org.apache.harmony.xnet.provider.jsse.TrustManagerImpl.checkServerTrusted(TrustManagerImpl.java:202)
    at org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:595)
    at org.apache.harmony.xnet.provider.jsse.NativeCrypto.SSL_do_handshake(Native Method)
    at org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:398)
	... 26 more
Caused by: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: current time: Wed Jun 07 23:08:11 EDT 2017, expiration time: Sun Jan 22 20:12:00 EST 2017
    at com.android.org.bouncycastle.jce.provider.RFC3280CertPathUtilities.processCertA(RFC3280CertPathUtilities.java:1488)
    at com.android.org.bouncycastle.jce.provider.PKIXCertPathValidatorSpi.engineValidate(PKIXCertPathValidatorSpi.java:305)
    at java.security.cert.CertPathValidator.validate(CertPathValidator.java:190)
    at org.apache.harmony.xnet.provider.jsse.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:295)
	... 30 more
Caused by: java.security.cert.CertificateExpiredException: current time: Wed Jun 07 23:08:11 EDT 2017, expiration time: Sun Jan 22 20:12:00 EST 2017
    at org.apache.harmony.security.provider.cert.X509CertImpl.checkValidity(X509CertImpl.java:148)
    at org.apache.harmony.security.provider.cert.X509CertImpl.checkValidity(X509CertImpl.java:138)
    at com.android.org.bouncycastle.jce.provider.RFC3280CertPathUtilities.processCertA(RFC3280CertPathUtilities.java:1483)
	... 33 more
```
在这几个 Android 版本上，遇到过期 HTTPS 证书时，OkHttp 抛出异常 **`javax.net.ssl.SSLHandshakeException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: current time: Wed Jun 07 23:08:11 EDT 2017, expiration time: Sun Jan 22 20:12:00 EST 2017`**，在错误信息中会明确地指出是证书验证失败，并包含当前时间和证书的过期时间。

而这个过程，所有的网络包交互如下：

![2017-06-08 11-06-16屏幕截图.png](https://www.wolfcstech.com/images/1315506-6714dde7114225f7.png)

整个过程，包括 TCP 连接建立，SSL/TLS 握手，TCP ACK 以及 TCP 连接断开，总共只有 16 个包。

## Android 4.4.4

在 Android 4.4.4 上，与前面执行相同的操作，将报出如下的异常：

```
javax.net.ssl.SSLHandshakeException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: null
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:409)
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
Caused by: java.security.cert.CertificateException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: null
    at com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:308)
    at com.android.org.conscrypt.TrustManagerImpl.checkServerTrusted(TrustManagerImpl.java:202)
    at com.android.org.conscrypt.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:611)
    at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:405)
	... 26 more
Caused by: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: null
    at com.android.org.bouncycastle.jce.provider.RFC3280CertPathUtilities.processCertA(RFC3280CertPathUtilities.java:1488)
    at com.android.org.bouncycastle.jce.provider.PKIXCertPathValidatorSpi.engineValidate(PKIXCertPathValidatorSpi.java:305)
    at java.security.cert.CertPathValidator.validate(CertPathValidator.java:190)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:295)
	... 30 more
Caused by: java.security.cert.CertificateExpiredException
    at com.android.org.conscrypt.OpenSSLX509Certificate.checkValidity(OpenSSLX509Certificate.java:220)
    at com.android.org.bouncycastle.jce.provider.RFC3280CertPathUtilities.processCertA(RFC3280CertPathUtilities.java:1483)
	... 33 more
```

在 Android 4.4.4 上，遇到过期 HTTPS 证书时，OkHttp 抛出异常 **`
javax.net.ssl.SSLHandshakeException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: null`**，与 Android 4.1.1 、4.2.2、4.3 的异常大体相同，同样指出了异常是由于证书验证失败引起的，但其中包含的错误信息反而更少了，没有了当前时间和证书的过期时间。

由于支持的 SSL/TLS 版本相同，网络包的交互过程与 Android 4.1.1 、4.2.2、4.3 相同。

## Android 5.0.0、Android 5.1.0、6.0.0、
如果是在 Android 5.0.0、Android 5.1.0 、6.0.0 上执行相同的操作，则抛出的异常与 Android 4.1.1 、 4.2.2 和 4.3 上的更类似：
```
javax.net.ssl.SSLHandshakeException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: Certificate expired at Sun Jan 22 20:12:00 EST 2017 (compared to Wed Jun 07 23:13:49 EDT 2017)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:306)
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
    at java.lang.Thread.run(Thread.java:818)
Caused by: java.security.cert.CertificateException: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: Certificate expired at Sun Jan 22 20:12:00 EST 2017 (compared to Wed Jun 07 23:13:49 EDT 2017)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:344)
    at com.android.org.conscrypt.TrustManagerImpl.checkServerTrusted(TrustManagerImpl.java:219)
    at com.android.org.conscrypt.Platform.checkServerTrusted(Platform.java:113)
    at com.android.org.conscrypt.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:525)
    at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:302)
	... 26 more
Caused by: com.android.org.bouncycastle.jce.exception.ExtCertPathValidatorException: Could not validate certificate: Certificate expired at Sun Jan 22 20:12:00 EST 2017 (compared to Wed Jun 07 23:13:49 EDT 2017)
    at com.android.org.bouncycastle.jce.provider.RFC3280CertPathUtilities.processCertA(RFC3280CertPathUtilities.java:1488)
    at com.android.org.bouncycastle.jce.provider.PKIXCertPathValidatorSpi.engineValidate(PKIXCertPathValidatorSpi.java:305)
    at java.security.cert.CertPathValidator.validate(CertPathValidator.java:191)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:331)
	... 31 more
Caused by: java.security.cert.CertificateExpiredException: Certificate expired at Sun Jan 22 20:12:00 EST 2017 (compared to Wed Jun 07 23:13:49 EDT 2017)
    at com.android.org.conscrypt.OpenSSLX509Certificate.checkValidity(OpenSSLX509Certificate.java:234)
    at com.android.org.bouncycastle.jce.provider.RFC3280CertPathUtilities.processCertA(RFC3280CertPathUtilities.java:1483)
	... 34 more
```
在异常的错误消息中指出了异常是由证书验证失败引起的，并大体指明了证书的过期时间和当前时间。但用 Wireshark 抓到的包来看，区别要大一点：

![](https://www.wolfcstech.com/images/1315506-a2aca0be9efc528a.png)

从 Android 5.0.0 开始，提供了对 TLSv1.2 的支持，整个过程都会用 TLSv1.2 来完成。服务器发送的几个 SSL/TLS 消息 **Server Hello**，**Certificate**，**Server Key Exchange**，**Server Hello Don** 被放在了一个 TCP 包里，第 6 号包，因而，整个握手过程用了更少的包，

## Android 7.0.0 、Android 7.1.0

执行相同的操作，在 Android 7.0.0  和 Android 7.1.0 上，则抛出了如下的异常：
```
javax.net.ssl.SSLHandshakeException: Chain validation failed
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:361)
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
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1133)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:607)
    at java.lang.Thread.run(Thread.java:761)
Caused by: java.security.cert.CertificateException: Chain validation failed
    at com.android.org.conscrypt.TrustManagerImpl.verifyChain(TrustManagerImpl.java:610)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrustedRecursive(TrustManagerImpl.java:444)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrustedRecursive(TrustManagerImpl.java:464)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrustedRecursive(TrustManagerImpl.java:508)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:401)
    at com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:375)
    at com.android.org.conscrypt.TrustManagerImpl.getTrustedChainForServer(TrustManagerImpl.java:304)
    at android.security.net.config.NetworkSecurityTrustManager.checkServerTrusted(NetworkSecurityTrustManager.java:94)
    at android.security.net.config.RootTrustManager.checkServerTrusted(RootTrustManager.java:88)
    at com.android.org.conscrypt.Platform.checkServerTrusted(Platform.java:178)
    at com.android.org.conscrypt.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:596)
    at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:357)
	... 26 more
Caused by: java.security.cert.CertPathValidatorException: timestamp check failed
    at sun.security.provider.certpath.PKIXMasterCertPathValidator.validate(PKIXMasterCertPathValidator.java:127)
    at sun.security.provider.certpath.PKIXCertPathValidator.validate(PKIXCertPathValidator.java:215)
    at sun.security.provider.certpath.PKIXCertPathValidator.validate(PKIXCertPathValidator.java:143)
    at sun.security.provider.certpath.PKIXCertPathValidator.engineValidate(PKIXCertPathValidator.java:79)
    at java.security.cert.CertPathValidator.validate(CertPathValidator.java:301)
    at com.android.org.conscrypt.TrustManagerImpl.verifyChain(TrustManagerImpl.java:606)
	... 38 more
Caused by: java.security.cert.CertificateExpiredException: Certificate expired at Sun Jan 22 20:12:00 EST 2017 (compared to Thu Jun 08 02:55:14 EDT 2017)
    at com.android.org.conscrypt.OpenSSLX509Certificate.checkValidity(OpenSSLX509Certificate.java:234)
    at sun.security.provider.certpath.BasicChecker.verifyTimestamp(BasicChecker.java:194)
    at sun.security.provider.certpath.BasicChecker.check(BasicChecker.java:144)
    at sun.security.provider.certpath.PKIXMasterCertPathValidator.validate(PKIXMasterCertPathValidator.java:119)
	... 43 more
```
抛出的异常的类型依然为 **`javax.net.ssl.SSLHandshakeException`**，但异常的消息里仅指明了异常是由证书链验证失败引起的。从直接异常的多层源异常中才能看出来是由于证书过期引起的。从异常的堆栈中可以看出来，自 Android 7.0.0 开始，证书验证的部分还是有比较大的改动的。

这个握手失败的过程，同样包含 16 个网络包：

![](https://www.wolfcstech.com/images/1315506-239f0586548824c1.png)

[Done]