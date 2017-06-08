---
title: 通过 HTTPS 和 SSL 确保安全
date: 2017-05-26 13:17:49
categories: 安全
tags:
- Android开发
- 安全
- HTTPS
- 翻译
---


安全套接字层（SSL）—— 现在在技术上称为 [传输层安全 (TLS)](http://en.wikipedia.org/wiki/Transport_Layer_Security) —— 是客户端和服务器间加密通信的通用构建块。应用可能以错误的方式使用 SSL， 从而导致恶意实体能够拦截网络上的应用数据。为了帮助您确保您的应用中不会发生这种情况，本文重点介绍了使用安全网络协议的常见陷阱，并解决了关于使用 [公钥基础设施(PKI)](http://en.wikipedia.org/wiki/Public-key_infrastructure) 中更大的担忧。
<!--more-->
# 概念

在典型的 SSL 使用场景中，会给服务器配置一个包含公钥的证书及对应的私钥。作为 SSL 客户端和服务器握手的一部分，服务器通过 [公钥加密](http://en.wikipedia.org/wiki/Public-key_cryptography) 签名它的证书来证明它具有私钥。

然而，任何人都可以生成他们自己的证书和私钥，因此简单的握手，除了证明服务器知道与证书的公钥匹配的私钥外，无法证明任何东西。一种解决这一问题的方法是，让客户端具有包含一个或多个它信任的证书的集合。如果证书不在集合中，则服务器是不可信的。

这个简单的方法有一些缺点。随着时间流逝，服务器应该能够升级为更强的密钥 ("key rotation")，用一个新的替换证书中的公钥。不幸地是，现在客户端应用不得不由于服务器配置的必要改变而更新。如果服务器不在应用开发者的控制之下则尤其有问题，比如如果是一个第三方 web 服务的话。如果应用不得不与任意服务器对话的话，比如 Web 浏览器或 Email 应用，这种方法也有问题。

为了解决这些问题，则给服务器配置来自于广为人知的称为 [证书机构(CA)](http://en.wikipedia.org/wiki/Certificate_authority) 的签发者的证书。就 Android 4.2 (Jelly Bean) 而言，Android 当前包含超过 100 CAs，且在每个发布版本中会得到更新。与服务器类似，一个 CA 具有一张证书和私钥。当给服务器签发证书时，CA 用它的私钥给服务器的证书 [签名](http://en.wikipedia.org/wiki/Digital_signature)。然后客户端可以验证服务器具有一张平台已知的 CA 签发的证书。

然而，解决了一些问题的同时，使用 CA 引入了另一些问题。由于 CA 为许多服务器签发证书，你依然需要某种方式来确保你在与你的目标服务器对话。为了解决这个问题，CA 签发的证书通过一个特定的名字，比如 *gmail.com*，或主机集合通配符，比如 **.google.com*，来标识服务器。

下面的例子将使这些概念更具体一点。在下面的命令行片段中，[openssl](http://www.openssl.org/docs/apps/openssl.html) 工具的 `s_client` 命令查看 Wikipedia 的服务器证书信息。它指定端口 443 是由于 HTTPS 的默认端口是这个。该命令将 `openssl s_client` 的输出发送给 `openssl x509`，后者根据 [X.509 标准](http://en.wikipedia.org/wiki/X.509) 格式化关于证书的信息。特别地，该命令请求主题，其中包含服务器的名字信息，签发者，这些标识 CA。

```
$ openssl s_client -connect wikipedia.org:443 | openssl x509 -noout -subject -issuer
subject= /serialNumber=sOrr2rKpMVP70Z6E9BT5reY008SJEdYv/C=US/O=*.wikipedia.org/OU=GT03314600/OU=See www.rapidssl.com/resources/cps (c)11/OU=Domain Control Validated - RapidSSL(R)/CN=*.wikipedia.org
issuer= /C=US/O=GeoTrust, Inc./CN=RapidSSL CA
```

你可以看到，证书是由 RapidSSL CA 为匹配 **.wikipedia.org* 的服务器签发的。

# 一个HTTPS 的例子

假设你有一个 Web 服务器，其具有一个众所周知的 CA 签发的证书，你可以通过这么简单的代码创建一个安全请求：

```
URL url = new URL("https://wikipedia.org");
URLConnection urlConnection = url.openConnection();
InputStream in = urlConnection.getInputStream();
copyInputStreamToOutputStream(in, System.out);
```

是的，它真的可以那么简单。如果你想要裁剪 HTTP 请求，你可以强制转换为一个 [HttpURLConnection](https://developer.android.com/reference/java/net/HttpURLConnection.html)。[HttpURLConnection](https://developer.android.com/reference/java/net/HttpURLConnection.html) 的 Android 文档有更多关于如何处理请求和响应头部，posting 内容，管理 cookies，使用代理，缓存响应，等等的例子。但就验证证书和主机名的细节而言，Android 框架通过这些 APIs 替你做了考虑。如果可能的话，这就是你想要的地方。也就是说，下面是一些其他考虑。

# 服务器证书验证的常见问题

假设没能从 [getInputStream()](https://developer.android.com/reference/java/net/URLConnection.html#getInputStream()) 接收内容，而是遇到了异常：

```
javax.net.ssl.SSLHandshakeException: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.
        at org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:374)
        at libcore.net.http.HttpConnection.setupSecureSocket(HttpConnection.java:209)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.makeSslConnection(HttpsURLConnectionImpl.java:478)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.connect(HttpsURLConnectionImpl.java:433)
        at libcore.net.http.HttpEngine.sendSocketRequest(HttpEngine.java:290)
        at libcore.net.http.HttpEngine.sendRequest(HttpEngine.java:240)
        at libcore.net.http.HttpURLConnectionImpl.getResponse(HttpURLConnectionImpl.java:282)
        at libcore.net.http.HttpURLConnectionImpl.getInputStream(HttpURLConnectionImpl.java:177)
        at libcore.net.http.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:271)
```

这可能由于几个原因而发生，包括：
1. [签发服务器证书的 CA 未知](https://developer.android.com/training/articles/security-ssl.html#UnknownCa)

2. [服务器证书不是由 CA 签发的，而是自签名的](https://developer.android.com/training/articles/security-ssl.html#SelfSigned)

3. [服务器配置缺失了一个中间 CA](https://developer.android.com/training/articles/security-ssl.html#MissingCa)

下面的几节讨论，在保持你与服务器间的连接安全的前提下，如何定位这些问题。

## 未知证书机构

在这种情况下，[SSLHandshakeException](https://developer.android.com/reference/javax/net/ssl/SSLHandshakeException.html) 由于你的 CA 不被系统信任而发生。它可能由于你的证书来自于一个新 CA，而该新 CA 还没有被 Android 信任，或者你的应用运行于没有 CA 的老旧版本上。CA 未知更常见的原因是，它不是一个公共 CA，而是由某个组织，比如政府、公司或者教育机构为自用而签发的私有的证书。

幸运的是，你可以让 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 信任一个特定的 CA 集。过程可能有点绕，因而下面的例子从一个 [InputStream](https://developer.android.com/reference/java/io/InputStream.html) 获取特定的 CA，并用它创建一个 [KeyStore](https://developer.android.com/reference/java/security/KeyStore.html)，然后后者被用于创建并初始化一个 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html)。[TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html) 是系统用于验证来自于服务器的证书的工具，可以通过创建包含一个或多个 CAs 的 [KeyStore](https://developer.android.com/reference/java/security/KeyStore.html)的创建，而创建的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html) 将仅信任这些 CAs。

给出新的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html)，示例初始化一个新的 [SSLContext](https://developer.android.com/reference/javax/net/ssl/SSLContext.html)，它提供一个  [SSLSocketFactory](https://developer.android.com/reference/javax/net/ssl/SSLSocketFactory.html)，你可以用来覆盖 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 默认的 [SSLSocketFactory](https://developer.android.com/reference/javax/net/ssl/SSLSocketFactory.html)。这种方式下，连接将在证书验证时使用你的 CAs。

下面是使用华盛顿大学的组织 CA 的完整例子：

```
// Load CAs from an InputStream
// (could be from a resource or ByteArrayInputStream or ...)
CertificateFactory cf = CertificateFactory.getInstance("X.509");
// From https://www.washington.edu/itconnect/security/ca/load-der.crt
InputStream caInput = new BufferedInputStream(new FileInputStream("load-der.crt"));
Certificate ca;
try {
    ca = cf.generateCertificate(caInput);
    System.out.println("ca=" + ((X509Certificate) ca).getSubjectDN());
} finally {
    caInput.close();
}

// Create a KeyStore containing our trusted CAs
String keyStoreType = KeyStore.getDefaultType();
KeyStore keyStore = KeyStore.getInstance(keyStoreType);
keyStore.load(null, null);
keyStore.setCertificateEntry("ca", ca);

// Create a TrustManager that trusts the CAs in our KeyStore
String tmfAlgorithm = TrustManagerFactory.getDefaultAlgorithm();
TrustManagerFactory tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
tmf.init(keyStore);

// Create an SSLContext that uses our TrustManager
SSLContext context = SSLContext.getInstance("TLS");
context.init(null, tmf.getTrustManagers(), null);

// Tell the URLConnection to use a SocketFactory from our SSLContext
URL url = new URL("https://certs.cac.washington.edu/CAtest/");
HttpsURLConnection urlConnection =
    (HttpsURLConnection)url.openConnection();
urlConnection.setSSLSocketFactory(context.getSocketFactory());
InputStream in = urlConnection.getInputStream();
copyInputStreamToOutputStream(in, System.out);
```

通过一个知道你的 CAs 的定制 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html)，系统能够验证你的服务器证书来自可信的签发者。

**注意：** 许多网站描述了一个糟糕的解决方案，即安装一个什么也不做的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html)。如果你这样做了，你可能无法加密你的通信，因为在公共 Wi-Fi 热点中，任何人都可以通过 DNS 欺骗，通过他们伪装为你的服务器的自己的代理发送你的用户的流量，来攻击你的用户。然后攻击者可以记录密码和其它个人数据。此方法之所以有效是由于攻击者可以生成一张证书 —— 没有 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html) 实际验证证书来自可信源 —— 你的应用可能正与任何人对话。因此不要这样做，甚至是临时的也不要。你可以总是使你的应用信任服务器证书的签发者，仅此而已。

## 自签名服务器证书

[SSLHandshakeException](https://developer.android.com/reference/javax/net/ssl/SSLHandshakeException.html) 的第二种情况，是由于自签名证书，这意味着服务器就是它自己的 CA。这与未知证书机构类似，因而你可以使用与前一节相同的方法。

你可以创建你自己的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html)，这次直接信任服务器证书。这具有前面讨论的将你的应用与证书直接绑定的所有缺点，但可以安全地完成。然而，你应该小心确保你的自签名证书具有合理强度的密钥。截至 2012 年，一年内过期的超过 65537 指数的 2048 位 RSA 签名是可接受的。当转换密钥时，你应该检查来自于认证机构 (比如 [NIST](http://www.nist.gov/)) 的 [建议](http://csrc.nist.gov/groups/ST/key_mgmt/index.html)，什么是可接受的。

## 丢失中间证书机构

[SSLHandshakeException](https://developer.android.com/reference/javax/net/ssl/SSLHandshakeException.html) 的第三种情况，是由于丢失中间 CA。大多数公共 CAs 不直接签发服务器证书。相反，他们使用自己的主 CA 证书，称为根 CA，签名中间 CAs。他们这样做使根 CA 可以离线存储以降低泄漏的风险。然而，典型的操作系统，比如 Android，只直接信任根 CAs，这在服务器证书—— 由中间 CA 签名 —— 和只知道根 CA 的证书验证器之间留下了短暂的信任鸿沟。为了解决这个问题，服务器不只在 SSL 握手期间给客户端发送它自己的证书，而是从服务器 CA 到可信的根 CA 之间经过的任何需要的中间证书的证书链。

来看实际中这是什么样的，这是由 ` openssl s_client` 命令看到的*mail.google.com* 证书链：

```
$ openssl s_client -connect mail.google.com:443
---
Certificate chain
 0 s:/C=US/ST=California/L=Mountain View/O=Google Inc/CN=mail.google.com
   i:/C=ZA/O=Thawte Consulting (Pty) Ltd./CN=Thawte SGC CA
 1 s:/C=ZA/O=Thawte Consulting (Pty) Ltd./CN=Thawte SGC CA
   i:/C=US/O=VeriSign, Inc./OU=Class 3 Public Primary Certification Authority
---
```

这显示服务器为 *mail.google.com* 发送了由 *Thawte SGC* CA 签发的证书，这是一个中间 CA，*Thawte SGC* CA 的第二个证书由 *Verisign* CA 签发，这是 Android 信任的主 CA。

然而，服务器的配置中不包含必要的中间证书比较少见。比如，这是一个服务器，它能导致 Android 浏览器报错，在 Android 应用中发生异常：
```
$ openssl s_client -connect egov.uscis.gov:443
---
Certificate chain
 0 s:/C=US/ST=District Of Columbia/L=Washington/O=U.S. Department of Homeland Security/OU=United States Citizenship and Immigration Services/OU=Terms of use at www.verisign.com/rpa (c)05/CN=egov.uscis.gov
   i:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=Terms of use at https://www.verisign.com/rpa (c)10/CN=VeriSign Class 3 International Server CA - G3
---
```

这里需要注意的有趣的是，在大多数桌面浏览器中访问这个服务器，不会像完全地未知 CA 或自签名服务器证书那样导致错误。这是由于随着时间流逝，大多数桌面浏览器缓存了可信中间 CAs。一旦浏览器从某网站访问并学习过了一个中间 CA，则在下次它将无需在证书链中包含该中间证书。

有些网站有意在 Web 服务器第二次提供资源服务时这样做。比如，它们可能让它们服务器提供的主 HTML 页面服务包含完整的证书链，但让服务器为资源比如图片，CSS，或 JavaScript 服务时不包含该 CA，大概是为了节省带宽。不幸地是，有时这些服务器可能提供了一个你需要在你的 Android 应用中访问的 Web 服务，而这种做法不可行。

有两种方法解决这个问题：
 * 配置服务器在服务器链中包含中间 CA。大多数 CAs 提供了关于如何为所有通用 Web 服务器做这些的文档。如果你需要网站能够通过至少 Android 4.2 工作于默认的 Android 浏览器上，这是仅有的方法。
 * 或者，将中间 CA 像任何其它未知 CA 一样对待，创建一个 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html) 直接信任它，如同前两节完成的那样。

# 域名验证常见问题

如本文开头提到的，验证 SSL 连接有两个关键部分。第一个是验证证书来自于可信源，前一节聚焦于这一点。这一节的焦点是第二部分：确保正在与之对话的服务器有正确的证书。当它没有时，典型地，你将看到如下错误：

```
java.io.IOException: Hostname 'example.com' was not verified
        at libcore.net.http.HttpConnection.verifySecureSocketHostname(HttpConnection.java:223)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.connect(HttpsURLConnectionImpl.java:446)
        at libcore.net.http.HttpEngine.sendSocketRequest(HttpEngine.java:290)
        at libcore.net.http.HttpEngine.sendRequest(HttpEngine.java:240)
        at libcore.net.http.HttpURLConnectionImpl.getResponse(HttpURLConnectionImpl.java:282)
        at libcore.net.http.HttpURLConnectionImpl.getInputStream(HttpURLConnectionImpl.java:177)
        at libcore.net.http.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:271)
```

这个错误发生的一个原因是服务器配置错误。服务器配置了一个证书，但它没有与正在试图访问的网站匹配的主题或主题替代名称字段。这可能是由于一个证书被用于多个不同的服务器中。比如，通过 ` openssl
 s_client -connect google.com:443 | openssl x509 -text
 ` 查看 *google.com* 证书，你能看到一个主题支持 * \*.google.com*，以及 * \*.youtube.com*，*\*.android.com* 和其它的主题替代名称 。该错误只在你正连接的服务器域名不包含在证书可接受的域名列表中时发生。

不幸地是，这也可能由于另一个原因而发生：[虚拟主机](http://en.wikipedia.org/wiki/Virtual_hosting)。当以 HTTP 为多个域名共享服务器时，Web 服务器可以从 HTTP/1.1 请求分辨客户端寻找的是哪个目标域名。不幸的是这对于 HTTPS 来说比较复杂，由于服务器不得不在看到 HTTP 请求之前知道返回哪个证书。为了解决这个问题，更新的 SSL 版本，特别是 TLSv.1.0 及之后，支持 [Server Name Indication (SNI)](http://en.wikipedia.org/wiki/Server_Name_Indication)，以允许 SSL 客户端为服务器指定目标主机名，使其可以返回适当的证书。

幸运地是，自 Android 2.3 开始 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 就支持 SNI。如果你需要支持 Android 2.2 (及更老版本)，一个解决方案是为每个可选虚拟主机设置一个唯一的端口，以使返回服务器证书不会有歧义。

更激进的方法是用一个不使用你的虚拟主机的域名，而是用服务器返回的默认的域名做验证的 [HostnameVerifier](https://developer.android.com/reference/javax/net/ssl/HostnameVerifier.html) ，替换默认的。

**注意：** 如果其它虚拟主机不在你的控制下的话，替换 [HostnameVerifier](https://developer.android.com/reference/javax/net/ssl/HostnameVerifier.html) 可能非常危险，因为中间人攻击可能在你不知道的情况下引导流量到另一个服务器。

如果你依然确认你想要覆盖域名验证，这里的有一个例子，它将一个单独的[URLConnection](https://developer.android.com/reference/java/net/URLConnection.html) 的验证器替换为了一个依然验证域名至少在应用期望的列表中的验证器：

```
// Create an HostnameVerifier that hardwires the expected hostname.
// Note that is different than the URL's hostname:
// example.com versus example.org
HostnameVerifier hostnameVerifier = new HostnameVerifier() {
    @Override
    public boolean verify(String hostname, SSLSession session) {
        HostnameVerifier hv =
            HttpsURLConnection.getDefaultHostnameVerifier();
        return hv.verify("example.com", session);
    }
};

// Tell the URLConnection to use our HostnameVerifier
URL url = new URL("https://example.org/");
HttpsURLConnection urlConnection =
    (HttpsURLConnection)url.openConnection();
urlConnection.setHostnameVerifier(hostnameVerifier);
InputStream in = urlConnection.getInputStream();
copyInputStreamToOutputStream(in, System.out);
```

但是请记得，如果你发现你自己替换了域名验证，特别是由于虚拟主机，如果其它虚拟主机不在的控制下的话，它依然 **非常危险** ，且你应该寻找一个方法做适当安排来避免这个问题。

# 关于直接使用 SSLSocket 的警告

目前为止，例子集中于使用 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 的 HTTPS。有时应用需要离开 HTTP 使用 SSL。比如，一个 Email 应用可能使用 SMTP，POP3， 或 IMAP 的 SSL 变体。在那些情况下，应用将想要直接使用 [SSLSocket](https://developer.android.com/reference/javax/net/ssl/SSLSocket.html)，与 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 内部做的非常相似。

目前介绍的处理证书验证问题的技术也适用于 [SSLSocket](https://developer.android.com/reference/javax/net/ssl/SSLSocket.html)。实际上，当使用定制的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html) 时，传给 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 的是一个 [SSLSocketFactory](https://developer.android.com/reference/javax/net/ssl/SSLSocketFactory.html)。因此如果你需要在 [SSLSocket](https://developer.android.com/reference/javax/net/ssl/SSLSocket.html) 中使用一个定制的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html)，则按照相同的步骤并使用 [SSLSocketFactory](https://developer.android.com/reference/javax/net/ssl/SSLSocketFactory.html) 创建你的 [SSLSocket](https://developer.android.com/reference/javax/net/ssl/SSLSocket.html) 。

**注意：** [SSLSocket](https://developer.android.com/reference/javax/net/ssl/SSLSocket.html) **不** 执行域名验证。你的应用负责完成它自己的域名验证，更好地是以期望的域名为参数调用 [getDefaultHostnameVerifier()](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html#getDefaultHostnameVerifier())。还要注意 [HostnameVerifier.verify()](https://developer.android.com/reference/javax/net/ssl/HostnameVerifier.html#verify(java.lang.String, javax.net.ssl.SSLSession)) 在发生错误时不抛出异常，而是返回一个你必须显式地检查的 boolean 结果。

这里有一个例子，展示了你可以如何做。它显示当连接 *gmail.com* 端口 443，而没有 SNI 支持时，你将收到一张 *mail.google.com* 的证书。在这个例子中这正是期待的，因此检查以确保证书确实是 *mail.google.com*：

```
// Open SSLSocket directly to gmail.com
SocketFactory sf = SSLSocketFactory.getDefault();
SSLSocket socket = (SSLSocket) sf.createSocket("gmail.com", 443);
HostnameVerifier hv = HttpsURLConnection.getDefaultHostnameVerifier();
SSLSession s = socket.getSession();

// Verify that the certicate hostname is for mail.google.com
// This is due to lack of SNI support in the current SSLSocket.
if (!hv.verify("mail.google.com", s)) {
    throw new SSLHandshakeException("Expected mail.google.com, "
                                    "found " + s.getPeerPrincipal());
}

// At this point SSLSocket performed certificate verificaiton and
// we have performed hostname verification, so it is safe to proceed.

// ... use socket ...
socket.close();
```

# 黑名单

SSL 非常依赖 CAs 只给服务器和域名经过适当验证的所有者签发证书。在罕见的情况下，CAs或者犯了错，或者在 [Comodo](http://en.wikipedia.org/wiki/Comodo_Group#Breach_of_security) 或 [DigiNotar](http://en.wikipedia.org/wiki/DigiNotar) 的例子中，被攻破，导致域名的证书被签发给了服务器和域名的所有者之外的其它人。

为了缓解这种风险，Android 具有将某一证书或甚至整个 CA 加入黑名单的能力。尽管这个列表历史上是内建在操作系统中的，但自 Android 4.2 开始，这个列表可以远程更新以处理未来的问题。

# 钉扎

应用可以通过一种称为钉扎的技术保护自己免遭错误签发的证书的危害。这基本上是使用上面在未知 CA 的情况下提供的例子，限制应用的可信 CAs 为一个已知应用服务器会使用的小集合。这样可以防止因泄露系统中其他 100 多个 CA 中的某个 CA 而破坏应用安全通道。

# 客户端证书

本文聚焦于与服务器进行安全通信的 SSL 用户。SSL 还支持用于服务器验证客户端身份的客户端证书的概念。尽管超出了本文的范围，该技术与指定一个定制的 [TrustManager](https://developer.android.com/reference/javax/net/ssl/TrustManager.html) 相似。请参考 [HttpsURLConnection](https://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html) 的文档中关于创建一个定制 [KeyManager](https://developer.android.com/reference/javax/net/ssl/KeyManager.html) 的部分。

# Nogotofail：一个网络流量安全性测试工具

Nogotofail 是一个工具，它给了你一种简单的方式，确保你的应用对于已知的 TLS/SSL 漏洞和错误配置是安全的。它是一个在任何网络流量可能通过的设备上测试网络安全问题的自动化的，强大的，且可扩展的工具。

对与三个主要的使用场景 Nogotofail 很有用：
 * 查找 bugs 和漏洞
 * 验证修复和观察回归
 * 理解什么应用和设备在产生流量

Nogotofail 适用于 Android， iOS， Linux， Windows， Chrome OS， OSX，实际上适用于任何你连接互联网的设备。在 Android 和 Linux 上有一个使用简单的客户端可以配置设定及获得通知，而且攻击引擎本身可以作为路由器，VPN 服务器，或代理部署。

你可以在 [Nogotofail 开源项目](https://github.com/google/nogotofail) 访问这个工具。

[原文](https://developer.android.com/training/articles/security-ssl.html#nogotofail)

Done。