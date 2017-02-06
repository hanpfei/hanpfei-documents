---
title: Android端打开HttpDns的正确姿势
date: 2017-02-06 16:05:49
tags:
- Android
- 网络
---

# 什么是HttpDns？

DNS服务用于在网络请求时，将域名转为IP地址。传统的基于UDP协议的公共DNS服务极易发生DNS劫持，从而造成安全问题。HttpDns服务则是基于HTTP协议自建DNS服务，或者选择更加可靠的DNS服务提供商来完成DNS服务，以降低发生安全问题的风险。HttpDns还可以为精准调度提供支持。因而在当前网络环境中得到了越来越多的应用。
<!--more-->
HttpDns的协议则因具体实现而异。通常是客户端将当前设备的一些信息，比如区域、运营商、网络的连接方式（WiFi还是移动网络）以及要解析的域名等传给HttpDns服务器，服务器为客户端返回对应的IP地址列表及这些IP地址的有效期等。

新浪的微博团队有开源自己的HttpDns方案出来，OSC的码云上[项目地址](https://www.oschina.net/p/httpdnslib)，[iOS版](https://www.oschina.net/p/httpdnslib-for-ios)，项目的[GitHub地址](https://github.com/CNSRE/HTTPDNSLib/blob/master/src/DNSCache/src/com/sina/util/dnscache/DNSCache.java)。腾讯有开放自己的[HttpDns服务](https://github.com/tencentyun/httpdns-android-sdk)。[阿里云](https://help.aliyun.com/product/30100.html?spm=5176.doc30140.3.1.hkjg6L) 和 [DNSPod](https://www.dnspod.cn/httpdns/guide) 还推出了商业化的产品。其他公司在开发自有HttpDns服务时，大多也会参考前人的接口设计，及接入方法，包括 [普通HTTP请求接入](https://help.aliyun.com/document_detail/30140.html?spm=5176.product30100.6.579.RcmTFq)，[WebView接入](https://help.aliyun.com/document_detail/48972.html?spm=5176.doc30140.2.6.y50oG7)，以及 [HTTPS (SNI 与非SNI)接入](https://help.aliyun.com/document_detail/30143.html?spm=5176.doc30140.2.4.Ek1lPN)等。我们的 HttpDns 服务的设计也参考了一点阿里的思路，然而按照阿里的接入方法接入时却遇到了一些问题。

# HttpDns的基本接入手法及其问题
在移动端，我们通常不会关心Http请求的详细执行过程，一般是将URL传给网络库，比如OkHttp、Volley、HttpClient或HttpUrlConnection等，简单的设置一些必要的request header，发起请求，并在请求执行结束之后获取响应。我们通过HttpDns获得的只是一些IP地址列表，那要如何将这些IP地址应用到网络请求中呢？

将由HttpDns获得的IP地址应用到我们的网络请求中最简单的办法，就是在原有URL的基础上，将域名替换为IP，然后用新的URL发起HTTP请求。然而，标准的HTTP协议中服务端会将HTTP请求头中HOST字段的值作为请求的域名，在我们没有主动设置HOST字段的值时，网络库也会自动地从URL中提取域名，并为请求做设置。但使用HttpDns后，URL中的域名信息丢失，会导致默认情况下请求的HOST 头部字段无法被正确设置，进而导致服务端的异常。为了解决这个问题，需要主动地为请求设置HOST字段值，如：
```
        String originalUrl = "http://www.wolfcstech.com/";
        URL url = new URL(originalURL);
        String originalHost = url.getHost();
        // 同步接口获取IP
        String ip = httpdns.getIpByHost(originalHost);
        HttpURLConnection conn;
        if (ip != null) {
            // 通过HTTPDNS获取IP成功，进行URL替换和HOST头设置
            url = new URL(originalUrl.replaceFirst(originalHost, ip));
            conn = (HttpURLConnection) url.openConnection();
            // 设置请求HOST字段
            conn.setRequestProperty("Host", originHost);
        } else {
            conn = (HttpURLConnection) url.openConnection();
        }
```
这样是可以解决，服务器获取请求的域名的需要。然而，URL中的域名不只是服务器会用到。在客户端的网络库中，至少还有如下几个地方同样需要用到（具体可以参考 [OkHttp3连接建立过程分析](https://www.wolfcstech.com/2016/10/27/OkHttp3%E8%BF%9E%E6%8E%A5%E5%BB%BA%E7%AB%8B%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/) 和 [OkHttp3中的代理与路由](https://www.wolfcstech.com/2016/10/14/OkHttp3%E4%B8%AD%E7%9A%84%E4%BB%A3%E7%90%86%E4%B8%8E%E8%B7%AF%E7%94%B1/) ）：
 * COOKIE的存取。支持COOKIE存取的网络库，在存取COOKIE时，从URL中提取的域名通常是key的重要部分。
 * 连接管理。连接的 Keep-Alive参数，可以让执行HTTP请求的TCP连接在请求结束后不会被立即关闭，而是先保持一段时间。为新发起的请求查找可用连接时，主要的依据也是URL中的域名。针对相同域名同时执行的HTTP请求的最大个数 6 个的限制，也需要借助于URL中的域名来完成。
 * HTTPS的SNI及证书验证。SSL/TLS的SNI扩展用于支持虚拟主机托管。在SSL/TLS握手期间，客户端通过该扩展将要请求的域名发送给服务器，以便可以取到适当的证书。SNI信息也来源于URL中的域名。

[阿里云建议](https://help.aliyun.com/document_detail/30140.html?spm=5176.product30100.6.579.RcmTFq) 在使用HttpDns时关闭COOKIE。直接替换原URL中的域名发起请求，会使得对单域名的最大并发连接数限制退化为了对服务器IP地址的最大并发连接数限制；在发起HTTPS请求时，无法正确设置SNI信息只能拿到默认的证书，在域名验证时，会将IP地址作为验证的域名而导致验证失败。

## HTTPS 域名证书验证问题 (不含SNI) 的解法
许多服务并不是多服务（域名）共用一个物理IP的，因而丢失SNI信息并不是特别的要紧。对于这种场景，解决掉域名证书的验证问题即可。针对 HttpsURLConnection 接口，方法如下：
```
        try {
            String url = "https://140.225.164.59/?sprefer=sypc00";
            final String originHostname = "www.wolfcstech.com";
            HttpsURLConnection connection = (HttpsURLConnection) new URL(url).openConnection();
            connection.setRequestProperty("Host", originHostname);
            connection.setHostnameVerifier(new HostnameVerifier() {
                /*
                 * 关于这个接口的说明，官方有文档描述：
                 * This is an extended verification option that implementers can provide.
                 * It is to be used during a handshake if the URL's hostname does not match the
                 * peer's identification hostname.
                 *
                 * 使用HTTPDNS后URL里设置的hostname不是远程的主机名(如:m.taobao.com)，与证书颁发的域不匹配，
                 * Android HttpsURLConnection提供了回调接口让用户来处理这种定制化场景。
                 * 在确认HTTPDNS返回的源站IP与Session携带的IP信息一致后，您可以在回调方法中将待验证域名替换为原来的真实域名进行验证。
                 *
                 */
                @Override
                public boolean verify(String hostname, SSLSession session) {
                    return HttpsURLConnection.getDefaultHostnameVerifier().verify(originHostname, session);
                }
            });
            connection.connect();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
        }
```

## SNI问题解法一
对于多个域名部署在相同IP地址的主机上的场景，除了要处理域名证书验证外，SNI的设置也是必须的。阿里云给出的解决方案是，自定义SSLSocketFactory，控制SSLSocket的创建过程。在SSLSocket被创建成功之后，立即设置SNI信息进去。

定制的SSLSocketFactory实现如下：
```
public class TlsSniSocketFactory extends SSLSocketFactory {
    private final String TAG = TlsSniSocketFactory.class.getSimpleName();
    HostnameVerifier hostnameVerifier = HttpsURLConnection.getDefaultHostnameVerifier();
    private HttpsURLConnection conn;
    public TlsSniSocketFactory(HttpsURLConnection conn) {
        this.conn = conn;
    }
    @Override
    public Socket createSocket() throws IOException {
        return null;
    }
    @Override
    public Socket createSocket(String host, int port) throws IOException, UnknownHostException {
        return null;
    }

    @Override
    public Socket createSocket(String host, int port, InetAddress localHost, int localPort) throws IOException, UnknownHostException {
        return null;
    }

    @Override
    public Socket createSocket(InetAddress host, int port) throws IOException {
        return null;
    }
    @Override
    public Socket createSocket(InetAddress address, int port, InetAddress localAddress, int localPort) throws IOException {
        return null;
    }
    // TLS layer
    @Override
    public String[] getDefaultCipherSuites() {
        return new String[0];
    }
    @Override
    public String[] getSupportedCipherSuites() {
        return new String[0];
    }
    @Override
    public Socket createSocket(Socket plainSocket, String host, int port, boolean autoClose) throws IOException {
        String peerHost = this.conn.getRequestProperty("Host");
        if (peerHost == null)
            peerHost = host;
        Log.i(TAG, "customized createSocket. host: " + peerHost);
        InetAddress address = plainSocket.getInetAddress();
        if (autoClose) {
            // we don't need the plainSocket
            plainSocket.close();
        }
        // create and connect SSL socket, but don't do hostname/certificate verification yet
        SSLCertificateSocketFactory sslSocketFactory = (SSLCertificateSocketFactory) SSLCertificateSocketFactory.getDefault(0);
        SSLSocket ssl = (SSLSocket) sslSocketFactory.createSocket(address, port);
        // enable TLSv1.1/1.2 if available
        ssl.setEnabledProtocols(ssl.getSupportedProtocols());
        // set up SNI before the handshake
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            Log.i(TAG, "Setting SNI hostname");
            sslSocketFactory.setHostname(ssl, peerHost);
        } else {
            Log.d(TAG, "No documented SNI support on Android <4.2, trying with reflection");
            try {
                java.lang.reflect.Method setHostnameMethod = ssl.getClass().getMethod("setHostname", String.class);
                setHostnameMethod.invoke(ssl, peerHost);
            } catch (Exception e) {
                Log.w(TAG, "SNI not useable", e);
            }
        }
        // verify hostname and certificate
        SSLSession session = ssl.getSession();
        if (!hostnameVerifier.verify(peerHost, session))
            throw new SSLPeerUnverifiedException("Cannot verify hostname: " + peerHost);
        Log.i(TAG, "Established " + session.getProtocol() + " connection with " + session.getPeerHost() +
                " using " + session.getCipherSuite());
        return ssl;
    }
}
```
HTTPS请求发起过程如下：
```
    public void recursiveRequest(String path, String reffer) {
        URL url = null;
        try {
            url = new URL(path);
            conn = (HttpsURLConnection) url.openConnection();
            // 同步接口获取IP
            String ip = httpdns.getIpByHostAsync(url.getHost());
            if (ip != null) {
                // 通过HTTPDNS获取IP成功，进行URL替换和HOST头设置
                Log.d(TAG, "Get IP: " + ip + " for host: " + url.getHost() + " from HTTPDNS successfully!");
                String newUrl = path.replaceFirst(url.getHost(), ip);
                conn = (HttpsURLConnection) new URL(newUrl).openConnection();
                // 设置HTTP请求头Host域
                conn.setRequestProperty("Host", url.getHost());
            }
            conn.setConnectTimeout(30000);
            conn.setReadTimeout(30000);
            conn.setInstanceFollowRedirects(false);
            TlsSniSocketFactory sslSocketFactory = new TlsSniSocketFactory(conn);
            conn.setSSLSocketFactory(sslSocketFactory);
            conn.setHostnameVerifier(new HostnameVerifier() {
                /*
                 * 关于这个接口的说明，官方有文档描述：
                 * This is an extended verification option that implementers can provide.
                 * It is to be used during a handshake if the URL's hostname does not match the
                 * peer's identification hostname.
                 *
                 * 使用HTTPDNS后URL里设置的hostname不是远程的主机名(如:m.taobao.com)，与证书颁发的域不匹配，
                 * Android HttpsURLConnection提供了回调接口让用户来处理这种定制化场景。
                 * 在确认HTTPDNS返回的源站IP与Session携带的IP信息一致后，您可以在回调方法中将待验证域名替换为原来的真实域名进行验证。
                 *
                 */
                @Override
                public boolean verify(String hostname, SSLSession session) {
                    String host = conn.getRequestProperty("Host");
                    if (null == host) {
                        host = conn.getURL().getHost();
                    }
                    return HttpsURLConnection.getDefaultHostnameVerifier().verify(host, session);
                }
            });
            int code = conn.getResponseCode();// Network block
            if (needRedirect(code)) {
                //临时重定向和永久重定向location的大小写有区分
                String location = conn.getHeaderField("Location");
                if (location == null) {
                    location = conn.getHeaderField("location");
                }
                if (!(location.startsWith("http://") || location
                        .startsWith("https://"))) {
                    //某些时候会省略host，只返回后面的path，所以需要补全url
                    URL originalUrl = new URL(path);
                    location = originalUrl.getProtocol() + "://"
                            + originalUrl.getHost() + location;
                }
                recursiveRequest(location, path);
            } else {
                // redirect finish.
                DataInputStream dis = new DataInputStream(conn.getInputStream());
                int len;
                byte[] buff = new byte[4096];
                StringBuilder response = new StringBuilder();
                while ((len = dis.read(buff)) != -1) {
                    response.append(new String(buff, 0, len));
                }
                Log.d(TAG, "Response: " + response.toString());
            }
        } catch (MalformedURLException e) {
            Log.w(TAG, "recursiveRequest MalformedURLException");
        } catch (IOException e) {
            Log.w(TAG, "recursiveRequest IOException");
        } catch (Exception e) {
            Log.w(TAG, "unknow exception");
        } finally {
            if (conn != null) {
                conn.disconnect();
            }
        }
    }
    private boolean needRedirect(int code) {
        return code >= 300 && code < 400;
    }
```
但这种解法是否真的可行呢？OkHttp被集成进AOSP并作为Android Java层的HTTP stack已经有一段时间了，我们就通过OkHttp的代码来看一下这种方法是否真的可行。

在OkHttp中，TLS的处理主要在RealConnection.connectTls()中：
```
  private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
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
可以看到，在创建了SSLSocket之后，总是会再通过平台相关的接口设置SNI信息。具体对于Android而言，是AndroidPlatform.configureTlsExtensions()：
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
可见，前面的解法并不可行。在SSLSocket创建期间设置的SNI信息，总是会由于SNI的再次设置而被冲掉，而后一次SNI信息来源则是URL。

## HTTPS (含SNI) 解法二
只定制 SSLSocketFactory 的方法，看起来是比较难以达成目的了，有人就想通过更深层的定制，即同时自定义SSLSocket来实现，如GitHub中的 [某项目](https://github.com/guardianproject/NetCipher/blob/master/libnetcipher/src/info/guardianproject/netcipher/client/TlsOnlySocketFactory.java)。

但这种方法的问题更严重。支持SSL扩展的许多接口，都不是标准的SSLSocket接口，比如用于支持SNI的setHostname()接口，用于支持ALPN的setAlpnProtocols()  和 getAlpnSelectedProtocol() 接口等。这样的接口还会随着SSL/TLS协议的发展而不断增加。许多网路库，如OkHttp，在调用这些接口时主要通过反射完成。而在自己定义SSLSocket实现的时候，很容易遗漏掉这些接口的实现，进而折损掉某些系统本身支持的SSL扩展。

# 接入HttpDns的更好方法
前面遇到的那些问题，主要都是由于替换URL中的域名为IP地址发起请求时，URL中域名信息丢失，而URL中的域名在网络库的多个地方被用到而引起。接入HttpDns的更好方法是，不要替换请求的URL中的域名部分，只在需要Dns的时候，才让HttpDns登场。

具体而言，是使用那些可以定制Dns逻辑的网络库，比如OkHttp，或者 我们在Chromium的网络库基础上做的[库](https://github.com/hanpfei/chromium-net-independent)，实现域名解析的接口，并在该接口的实现中通过HttpDns模块来执行域名解析。这样就不会对网络库造成那么多未知的冲击。

如：
```
    private static class MyDns implements Dns {

        @Override
        public List<InetAddress> lookup(String hostname) throws UnknownHostException {
            List<String> strIps = HttpDns.getInstance().getIpByHost(hostname);
            List<InetAddress> ipList;
            if (strIps != null && strIps.size() > 0) {
                ipList = new ArrayList<>();
                for (String ip : strIps) {
                    ipList.add(InetAddress.getByName(ip));
                }
            } else {
                ipList = Dns.SYSTEM.lookup(hostname);
            }
            return ipList;
        }
    }

    private OkHttp3Utils() {
        okhttp3.OkHttpClient.Builder builder = new okhttp3.OkHttpClient.Builder();
        builder.dns(new MyDns());
        mOkHttpClient = builder.build();
    }
```

这种方法既简单又副作用小。
