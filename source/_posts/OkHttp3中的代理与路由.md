---
title: OkHttp3中的代理与路由
---

﻿路由是什么呢？路由即是网络数据包在网络中的传输路径，或者说数据包在传输过程中所经过的网络节点，比如路由器，代理服务器之类的。

那像OkHttp3这样的网络库对于数据包的路由需要做些什么事呢？用户可以为终端设置代理服务器，HTTP/HTTPS代理或SOCK代理。OkHttp3中的路由相关逻辑，需要从系统中获取用户设置的代理服务器的地址，将HTTP请求转换为代理协议的数据包，发给代理服务器，然后等待代理服务器帮助完成了网络请求之后，从代理服务器读取响应数据返回给用户。只有这样，用户设置的代理才能生效。如果网络库无视用户设置的代理服务器，直接进行DNS并做网络请求，则用户设置的代理服务器不生效。

这里就来看一下OkHttp3中路由相关的处理。

# 路由选择

如同Internet上的其它设备一样，每个路由节点都有自己的IP地址，加上端口号，则可以确定唯一的路由服务。以域名描述的HTTP/HTTPS代理服务器地址，可能对应于多个实际的代理服务器主机，因而一个代理服务器可能包含有多条路由。而SOCK代理服务器，则有着唯一确定的IP地址和端口号。

OkHttp3借助于RouteSelector来选择路由节点，并维护路由的信息。
```
public final class RouteSelector {
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

  /**
   * Returns true if there's another route to attempt. Every address has at least one route.
   */
  public boolean hasNext() {
    return hasNextInetSocketAddress()
        || hasNextProxy()
        || hasNextPostponed();
  }

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

  /**
   * Clients should invoke this method when they encounter a connectivity failure on a connection
   * returned by this route selector.
   */
  public void connectFailed(Route failedRoute, IOException failure) {
    if (failedRoute.proxy().type() != Proxy.Type.DIRECT && address.proxySelector() != null) {
      // Tell the proxy selector when we fail to connect on a fresh connection.
      address.proxySelector().connectFailed(
          address.url().uri(), failedRoute.proxy().address(), failure);
    }

    routeDatabase.failed(failedRoute);
  }

  /** Prepares the proxy servers to try. */
  private void resetNextProxy(HttpUrl url, Proxy proxy) {
    if (proxy != null) {
      // If the user specifies a proxy, try that and only that.
      proxies = Collections.singletonList(proxy);
    } else {
      // Try each of the ProxySelector choices until one connection succeeds. If none succeed
      // then we'll try a direct connection below.
      proxies = new ArrayList<>();
      List<Proxy> selectedProxies = address.proxySelector().select(url.uri());
      if (selectedProxies != null) proxies.addAll(selectedProxies);
      // Finally try a direct connection. We only try it once!
      proxies.removeAll(Collections.singleton(Proxy.NO_PROXY));
      proxies.add(Proxy.NO_PROXY);
    }
    nextProxyIndex = 0;
  }

  /** Returns true if there's another proxy to try. */
  private boolean hasNextProxy() {
    return nextProxyIndex < proxies.size();
  }

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

  /** Prepares the socket addresses to attempt for the current proxy or host. */
  private void resetNextInetSocketAddress(Proxy proxy) throws IOException {
    // Clear the addresses. Necessary if getAllByName() below throws!
    inetSocketAddresses = new ArrayList<>();

    String socketHost;
    int socketPort;
    if (proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.SOCKS) {
      socketHost = address.url().host();
      socketPort = address.url().port();
    } else {
      SocketAddress proxyAddress = proxy.address();
      if (!(proxyAddress instanceof InetSocketAddress)) {
        throw new IllegalArgumentException(
            "Proxy.address() is not an " + "InetSocketAddress: " + proxyAddress.getClass());
      }
      InetSocketAddress proxySocketAddress = (InetSocketAddress) proxyAddress;
      socketHost = getHostString(proxySocketAddress);
      socketPort = proxySocketAddress.getPort();
    }

    if (socketPort < 1 || socketPort > 65535) {
      throw new SocketException("No route to " + socketHost + ":" + socketPort
          + "; port is out of range");
    }

    if (proxy.type() == Proxy.Type.SOCKS) {
      inetSocketAddresses.add(InetSocketAddress.createUnresolved(socketHost, socketPort));
    } else {
      // Try each address for best behavior in mixed IPv4/IPv6 environments.
      List<InetAddress> addresses = address.dns().lookup(socketHost);
      for (int i = 0, size = addresses.size(); i < size; i++) {
        InetAddress inetAddress = addresses.get(i);
        inetSocketAddresses.add(new InetSocketAddress(inetAddress, socketPort));
      }
    }

    nextInetSocketAddressIndex = 0;
  }

  /**
   * Obtain a "host" from an {@link InetSocketAddress}. This returns a string containing either an
   * actual host name or a numeric IP address.
   */
  // Visible for testing
  static String getHostString(InetSocketAddress socketAddress) {
    InetAddress address = socketAddress.getAddress();
    if (address == null) {
      // The InetSocketAddress was specified with a string (either a numeric IP or a host name). If
      // it is a name, all IPs for that name should be tried. If it is an IP address, only that IP
      // address should be tried.
      return socketAddress.getHostName();
    }
    // The InetSocketAddress has a specific address: we should only try that address. Therefore we
    // return the address and ignore any host name that may be available.
    return address.getHostAddress();
  }

  /** Returns true if there's another socket address to try. */
  private boolean hasNextInetSocketAddress() {
    return nextInetSocketAddressIndex < inetSocketAddresses.size();
  }

  /** Returns the next socket address to try. */
  private InetSocketAddress nextInetSocketAddress() throws IOException {
    if (!hasNextInetSocketAddress()) {
      throw new SocketException("No route to " + address.url().host()
          + "; exhausted inet socket addresses: " + inetSocketAddresses);
    }
    return inetSocketAddresses.get(nextInetSocketAddressIndex++);
  }

  /** Returns true if there is another postponed route to try. */
  private boolean hasNextPostponed() {
    return !postponedRoutes.isEmpty();
  }

  /** Returns the next postponed route to try. */
  private Route nextPostponed() {
    return postponedRoutes.remove(0);
  }
}
```
`RouteSelector`主要做了这样一些事情：
1. 在`RouteSelector`对象创建时，获取并保存用户设置的所有的代理。这里主要通过`ProxySelector`，根据uri来得到系统中的所有代理，并保存在Proxy列表proxies中。
2. 给调用者提供接口，来选择可用的路由。调用者通过next()可以获取`RouteSelector`中维护的下一个可用路由。调用者在连接失败时，可以再次调用这个接口来获取下一个路由。这个接口会逐个地返回每个代理的每个代理主机服务给调用者。在所有的代理的每个代理主机都被访问过了之后，还会返回曾经连接失败的路由。
3. 维护路由节点的信息。`RouteDatabase`用于维护连接失败的路由的信息，以避免浪费时间去连接一些不可用的路由。`RouteDatabase`中的路由信息主要由`RouteSelector`来维护。

`RouteDatabase`是一个简单的容器：
```
package okhttp3.internal.connection;

import java.util.LinkedHashSet;
import java.util.Set;
import okhttp3.Route;

/**
 * A blacklist of failed routes to avoid when creating a new connection to a target address. This is
 * used so that OkHttp can learn from its mistakes: if there was a failure attempting to connect to
 * a specific IP address or proxy server, that failure is remembered and alternate routes are
 * preferred.
 */
public final class RouteDatabase {
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
}
```

OkHttp3主要用(Address, Proxy, InetSocketAddress)的三元组来描述路由信息：
```
package okhttp3;

import java.net.InetSocketAddress;
import java.net.Proxy;

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
public final class Route {
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

  public Address address() {
    return address;
  }

  /**
   * Returns the {@link Proxy} of this route.
   *
   * <strong>Warning:</strong> This may disagree with {@link Address#proxy} when it is null. When
   * the address's proxy is null, the proxy selector is used.
   */
  public Proxy proxy() {
    return proxy;
  }

  public InetSocketAddress socketAddress() {
    return inetSocketAddress;
  }

  /**
   * Returns true if this route tunnels HTTPS through an HTTP proxy. See <a
   * href="http://www.ietf.org/rfc/rfc2817.txt">RFC 2817, Section 5.2</a>.
   */
  public boolean requiresTunnel() {
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
  }

  @Override public boolean equals(Object obj) {
    if (obj instanceof Route) {
      Route other = (Route) obj;
      return address.equals(other.address)
          && proxy.equals(other.proxy)
          && inetSocketAddress.equals(other.inetSocketAddress);
    }
    return false;
  }

  @Override public int hashCode() {
    int result = 17;
    result = 31 * result + address.hashCode();
    result = 31 * result + proxy.hashCode();
    result = 31 * result + inetSocketAddress.hashCode();
    return result;
  }
}
```

在StreamAllocation中建立连接时，会通过`RouteSelector`获取可用路由。

在OkHttp3中，`ProxySelector`对象主要由OkHttpClient维护。
```
public class OkHttpClient implements Cloneable, Call.Factory {
......
  final ProxySelector proxySelector;
  
  private OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    this.protocols = builder.protocols;
    this.connectionSpecs = builder.connectionSpecs;
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    this.proxySelector = builder.proxySelector;

......

  public ProxySelector proxySelector() {
    return proxySelector;
  }

......

    public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      proxySelector = ProxySelector.getDefault();

......

    Builder(OkHttpClient okHttpClient) {
      this.dispatcher = okHttpClient.dispatcher;
      this.proxy = okHttpClient.proxy;
      this.protocols = okHttpClient.protocols;
      this.connectionSpecs = okHttpClient.connectionSpecs;
      this.interceptors.addAll(okHttpClient.interceptors);
      this.networkInterceptors.addAll(okHttpClient.networkInterceptors);
      this.proxySelector = okHttpClient.proxySelector;
```

在创建OkHttpClient时，可以通过为OkHttpClient.Builder设置`ProxySelector`来定制`ProxySelector`。若没有指定，则所有的为默认`ProxySelector`。OpenJDK 1.8版默认的`ProxySelector`为`sun.net.spi.DefaultProxySelector`：
```
public abstract class ProxySelector {
    /**
     * The system wide proxy selector that selects the proxy server to
     * use, if any, when connecting to a remote object referenced by
     * an URL.
     *
     * @see #setDefault(ProxySelector)
     */
    private static ProxySelector theProxySelector;

    static {
        try {
            Class<?> c = Class.forName("sun.net.spi.DefaultProxySelector");
            if (c != null && ProxySelector.class.isAssignableFrom(c)) {
                theProxySelector = (ProxySelector) c.newInstance();
            }
        } catch (Exception e) {
            theProxySelector = null;
        }
    }

    /**
     * Gets the system-wide proxy selector.
     *
     * @throws  SecurityException
     *          If a security manager has been installed and it denies
     * {@link NetPermission}{@code ("getProxySelector")}
     * @see #setDefault(ProxySelector)
     * @return the system-wide {@code ProxySelector}
     * @since 1.5
     */
    public static ProxySelector getDefault() {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(SecurityConstants.GET_PROXYSELECTOR_PERMISSION);
        }
        return theProxySelector;
    }
```
在Android平台上，默认`ProxySelector`所用的则是[另外的实现](http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/net/ProxySelector.java)：
```
public abstract class ProxySelector {

    private static ProxySelector defaultSelector = new ProxySelectorImpl();

    /**
     * Returns the default proxy selector, or null if none exists.
     */
    public static ProxySelector getDefault() {
        return defaultSelector;
    }

    /**
     * Sets the default proxy selector. If {@code selector} is null, the current
     * proxy selector will be removed.
     */
    public static void setDefault(ProxySelector selector) {
        defaultSelector = selector;
    }
```
Android平台下，默认的`ProxySelector` ProxySelectorImpl，其[实现(不同版本的Android，实现不同，这里是android-6.0.1_r61的实现)](https://android.googlesource.com/platform/libcore/+/android-6.0.1_r61/luni/src/main/java/java/net/ProxySelectorImpl.java)如下：
```
package java.net;
import java.io.IOException;
import java.util.Collections;
import java.util.List;
final class ProxySelectorImpl extends ProxySelector {
    @Override public void connectFailed(URI uri, SocketAddress sa, IOException ioe) {
        if (uri == null || sa == null || ioe == null) {
            throw new IllegalArgumentException();
        }
    }
    @Override public List<Proxy> select(URI uri) {
        return Collections.singletonList(selectOneProxy(uri));
    }
    private Proxy selectOneProxy(URI uri) {
        if (uri == null) {
            throw new IllegalArgumentException("uri == null");
        }
        String scheme = uri.getScheme();
        if (scheme == null) {
            throw new IllegalArgumentException("scheme == null");
        }
        int port = -1;
        Proxy proxy = null;
        String nonProxyHostsKey = null;
        boolean httpProxyOkay = true;
        if ("http".equalsIgnoreCase(scheme)) {
            port = 80;
            nonProxyHostsKey = "http.nonProxyHosts";
            proxy = lookupProxy("http.proxyHost", "http.proxyPort", Proxy.Type.HTTP, port);
        } else if ("https".equalsIgnoreCase(scheme)) {
            port = 443;
            nonProxyHostsKey = "https.nonProxyHosts"; // RI doesn't support this
            proxy = lookupProxy("https.proxyHost", "https.proxyPort", Proxy.Type.HTTP, port);
        } else if ("ftp".equalsIgnoreCase(scheme)) {
            port = 80; // not 21 as you might guess
            nonProxyHostsKey = "ftp.nonProxyHosts";
            proxy = lookupProxy("ftp.proxyHost", "ftp.proxyPort", Proxy.Type.HTTP, port);
        } else if ("socket".equalsIgnoreCase(scheme)) {
            httpProxyOkay = false;
        } else {
            return Proxy.NO_PROXY;
        }
        if (nonProxyHostsKey != null
                && isNonProxyHost(uri.getHost(), System.getProperty(nonProxyHostsKey))) {
            return Proxy.NO_PROXY;
        }
        if (proxy != null) {
            return proxy;
        }
        if (httpProxyOkay) {
            proxy = lookupProxy("proxyHost", "proxyPort", Proxy.Type.HTTP, port);
            if (proxy != null) {
                return proxy;
            }
        }
        proxy = lookupProxy("socksProxyHost", "socksProxyPort", Proxy.Type.SOCKS, 1080);
        if (proxy != null) {
            return proxy;
        }
        return Proxy.NO_PROXY;
    }
    /**
     * Returns the proxy identified by the {@code hostKey} system property, or
     * null.
     */
    private Proxy lookupProxy(String hostKey, String portKey, Proxy.Type type, int defaultPort) {
        String host = System.getProperty(hostKey);
        if (host == null || host.isEmpty()) {
            return null;
        }
        int port = getSystemPropertyInt(portKey, defaultPort);
        return new Proxy(type, InetSocketAddress.createUnresolved(host, port));
    }
    private int getSystemPropertyInt(String key, int defaultValue) {
        String string = System.getProperty(key);
        if (string != null) {
            try {
                return Integer.parseInt(string);
            } catch (NumberFormatException ignored) {
            }
        }
        return defaultValue;
    }
    /**
     * Returns true if the {@code nonProxyHosts} system property pattern exists
     * and matches {@code host}.
     */
    private boolean isNonProxyHost(String host, String nonProxyHosts) {
        if (host == null || nonProxyHosts == null) {
            return false;
        }
        // construct pattern
        StringBuilder patternBuilder = new StringBuilder();
        for (int i = 0; i < nonProxyHosts.length(); i++) {
            char c = nonProxyHosts.charAt(i);
            switch (c) {
            case '.':
                patternBuilder.append("\\.");
                break;
            case '*':
                patternBuilder.append(".*");
                break;
            default:
                patternBuilder.append(c);
            }
        }
        // check whether the host is the nonProxyHosts.
        String pattern = patternBuilder.toString();
        return host.matches(pattern);
    }
}
```
可以看到，在Android平台上，主要是从System properties中获取的代理服务器的主机及其端口号，会过滤掉不能进行代理的主机的访问。

回到OkHttp中，在RetryAndFollowUpInterceptor中，创建Address对象时，从OkHttpClient对象获取ProxySelector。Address对象会被用于创建StreamAllocation对象，StreamAllocation在建立连接时，从Address对象中获取ProxySelector以选择路由。
```
public final class RetryAndFollowUpInterceptor implements Interceptor {
......
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

# 代理协议
OkHttp3发送给HTTP代理服务器的HTTP请求，与直接发送给HTTP服务器的HTTP请求有什么样的区别呢，还是说两者其实毫无差别呢？也就是HTTP代理的协议是什么样的呢？这里我们就通过对代码进行分析来仔细地看一下。

如我们在[OkHttp3 HTTP请求执行流程分析](http://www.jianshu.com/p/230e2e2988e0)中看到的，OkHttp3对HTTP请求是通过Interceptor链来处理的。
`RetryAndFollowUpInterceptor`创建`StreamAllocation`对象，处理http的重定向及出错重试。对后续Interceptor的执行的影响为修改Request并创建StreamAllocation对象。
`BridgeInterceptor`补全缺失的一些http header。对后续Interceptor的执行的影响主要为修改了Request。
`CacheInterceptor`处理http缓存。对后续Interceptor的执行的影响为，若缓存中有所需请求的响应，则后续Interceptor不再执行。
`ConnectInterceptor`借助于前面分配的`StreamAllocation`对象建立与服务器之间的连接，并选定交互所用的协议是HTTP 1.1还是HTTP 2。对后续Interceptor的执行的影响为，创建了HttpStream和connection。
`CallServerInterceptor`作为Interceptor链中的最后一个Interceptor，用于处理IO，与服务器进行数据交换。

OkHttp3对代理的处理是在`ConnectInterceptor`和`CallServerInterceptor`中完成的。再来看`ConnectInterceptor`的定义：
```
package okhttp3.internal.connection;

import java.io.IOException;
import okhttp3.Interceptor;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.internal.http.HttpCodec;
import okhttp3.internal.http.RealInterceptorChain;

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

`ConnectInterceptor`利用前面的Interceptor创建的StreamAllocation对象，创建stream HttpCodec，以及RealConnection connection。然后把这些对象传给链中后继的Interceptor，也就是`CallServerInterceptor`处理。

为了厘清StreamAllocation的两个操作的详细执行过程，这里再回过头来看一下`StreamAllocation`对象的创建。StreamAllocation对象在`RetryAndFollowUpInterceptor`中创建：
```
  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);
```
创建`StreamAllocation`对象时，传入的ConnectionPool来自于OkHttpClient，创建的Address主要用于描述HTTP服务的目标地址相关的信息。
```
public final class StreamAllocation {
  public final Address address;
  private Route route;
  private final ConnectionPool connectionPool;
  private final Object callStackTrace;

  // State guarded by connectionPool.
  private final RouteSelector routeSelector;
  private int refusedStreamCount;
  private RealConnection connection;
  private boolean released;
  private boolean canceled;
  private HttpCodec codec;

  public StreamAllocation(ConnectionPool connectionPool, Address address, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.routeSelector = new RouteSelector(address, routeDatabase());
    this.callStackTrace = callStackTrace;
  }
```
创建`StreamAllocation`对象时，除了创建`RouteSelector`之外，并没有其它特别的地方。

然后来看`ConnectInterceptor`中用来创建HttpCodec的newStream()方法：
```
public final class StreamAllocation {

......

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
这个方法的执行流程为：
1. 建立连接。
通过调用findHealthyConnection()方法来建立连接，后面我们通过分析这个方法的实现来了解连接的具体含义。
2. 用前面创建的连接来创建HttpCodec。
对于HTTP/1.1创建Http1Codec，对于HTTP/2则创建Http2Codec。HttpCodec用于处理与HTTP具体协议相关的部分。比如HTTP/1.1是基于文本的协议，而HTTP/2则是基于二进制格式的协议，HttpCodec用于将请求编码为对应协议要求的传输格式，并在得到响应时，对数据进行解码。

然后来看`findHealthyConnection()`中创建连接的过程：
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
在这个方法中，是找到一个连接，然后判断其是否可用。如果可用则将找到的连接返回给调用者，否则寻找下一个连接。寻找连接可能是建立一个新的连接，也可能是复用连接池中的一个连接。

接着来看寻找连接的过程`findConnection()`：
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
这个过程大体为：
1. 检查上次分配的连接是否可用，若可用则，则将上次分配的连接返回给调用者。
2. 上次分配的连接不存在，或不可用，则从连接池中查找一个连接，查找的依据就是Address，也就是连接的对端地址，以及路由等信息。Internal.instance指向OkHttpClient的一个内部类的对象，Internal.instance.get()实际会通过ConnectionPool的`get(Address address, StreamAllocation streamAllocation)`方法来尝试获取RealConnection。
若能从连接池中找到所需要的连接，则将连接返回给调用者。
3. 从连接池中没有找到所需要的连接，则会首先选择路由。
4. 然后创建新的连接RealConnection对象。
5. acquire新创建的连接RealConnection对象，并将它放进连接池。不太确定这个地方的synchronized是不是太长了。貌似只有Internal.instance.put(connectionPool, newConnection)涉及到了全局对象的访问，而其它操作并没有。
6. 调用newConnection.connect()建立连接。

这里再来看一下在ConnectionPool的get()操作执行的过程：
```
  private final Deque<RealConnection> connections = new ArrayDeque<>();
  final RouteDatabase routeDatabase = new RouteDatabase();
  boolean cleanupRunning;

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
ConnectionPool连接池是连接的容器，这里用了一个Deque来保存所有的连接RealConnection。而get的过程就是，遍历保存的所有连接来匹配address。同时connection.allocations.size()要满足connection.allocationLimit的限制。
在找到了所需要的连接之后，会acquire该连接。

acquire连接的过程又是什么样的呢？
```
public final class StreamAllocation {

......

  /**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```
基本上就是给RealConnection的allocations添加一个到该StreamAllocation的引用。这样看来，同一个连接RealConnection似乎同时可以为多个HTTP请求服务。而我们知道，多个HTTP/1.1请求是不能在同一个连接上交叉处理的。那这又是怎么回事呢？

我们来看connection.allocationLimit的更新设置。RealConnection中如下的两个地方会设置这个值：
```
public final class RealConnection extends Http2Connection.Listener implements Connection {

......

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
  
  /** When settings are received, adjust the allocation limit. */
  @Override public void onSettings(Http2Connection connection) {
    allocationLimit = connection.maxConcurrentStreams();
  }
```
可以看到，若不是HTTP/2的连接，则allocationLimit的值总是1。由此可见，StreamAllocation以及RealConnection的allocations/allocationLimit这样的设计，主要是为了实现HTTP/2 multi stream的特性。否则的话，大概为RealConnection用一个inUse标记就可以了。
那

回到StreamAllocation的`findConnection()`，来看新创建的RealConnection对象建立连接的过程，即RealConnection的connect()：
```
public final class RealConnection extends Http2Connection.Listener implements Connection {
  private final Route route;

  /** The low-level TCP socket. */
  private Socket rawSocket;

  /**
   * The application layer socket. Either an {@link SSLSocket} layered over {@link #rawSocket}, or
   * {@link #rawSocket} itself if this connection does not use SSL.
   */
  public Socket socket;
  private Handshake handshake;
  private Protocol protocol;
  public volatile Http2Connection http2Connection;
  public int successCount;
  public BufferedSource source;
  public BufferedSink sink;
  public int allocationLimit;
  public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
  public boolean noNewStreams;
  public long idleAtNanos = Long.MAX_VALUE;

  public RealConnection(Route route) {
    this.route = route;
  }

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
根据路由的类型，来执行不同的创建连接的过程。对于需要创建隧道连接的路由，执行buildTunneledConnection()，而对于普通连接，则执行buildConnection()。

如何判断是否要建立隧道连接呢？来看
```
  /**
   * Returns true if this route tunnels HTTPS through an HTTP proxy. See <a
   * href="http://www.ietf.org/rfc/rfc2817.txt">RFC 2817, Section 5.2</a>.
   */
  public boolean requiresTunnel() {
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
  }
```
可以看到，通过代理服务器，来做https请求的连接(http/1.1的https和http2)需要建立隧道连接，而其它的连接则不需要建立隧道连接。

用于建立隧道连接的buildTunneledConnection()的过程：
```
  /**
   * Does all the work to build an HTTPS connection over a proxy tunnel. The catch here is that a
   * proxy server can issue an auth challenge and then close the connection.
   */
  private void buildTunneledConnection(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
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

    establishProtocol(readTimeout, writeTimeout, connectionSpecSelector);
  }
```
基本上是两个过程：
1. 建立隧道连接。
2. 建立Protocol。

建立隧道连接的过程，又分为了几个过程：

 - 创建隧道请求
 - 建立Socket连接
 - 发送请求建立隧道

隧道请求是一个常规的HTTP请求，只是请求的内容有点特殊。初始的隧道请求如：
```
  /**
   * Returns a request that creates a TLS tunnel via an HTTP proxy. Everything in the tunnel request
   * is sent unencrypted to the proxy server, so tunnels include only the minimum set of headers.
   * This avoids sending potentially sensitive data like HTTP cookies to the proxy unencrypted.
   */
  private Request createTunnelRequest() {
    return new Request.Builder()
        .url(route.address().url())
        .header("Host", Util.hostHeader(route.address().url(), true))
        .header("Proxy-Connection", "Keep-Alive")
        .header("User-Agent", Version.userAgent()) // For HTTP/1.0 proxies like Squid.
        .build();
  }
```
建立socket连接的过程如下：
```
  private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();

    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    rawSocket.setSoTimeout(readTimeout);
    try {
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      throw new ConnectException("Failed to connect to " + route.socketAddress());
    }
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));
  }
```
主要是创建一个到代理服务器或HTTP服务器的Socket连接。socketFactory最终来自于OkHttpClient，对于OpenJDK 8而言，默认为DefaultSocketFactory：
```
    /**
     * Returns a copy of the environment's default socket factory.
     *
     * @return the default <code>SocketFactory</code>
     */
    public static SocketFactory getDefault()
    {
        synchronized (SocketFactory.class) {
            if (theFactory == null) {
                //
                // Different implementations of this method SHOULD
                // work rather differently.  For example, driving
                // this from a system property, or using a different
                // implementation than JavaSoft's.
                //
                theFactory = new DefaultSocketFactory();
            }
        }

        return theFactory;
    }
```
创建隧道的过程是这样子的：
```
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
      Response response = tunnelConnection.readResponse().request(tunnelRequest).build();
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
主要HTTP 的 CONNECT 方法建立隧道。

而建立常规的连接的过程则为：
```
  /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  private void buildConnection(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    connectSocket(connectTimeout, readTimeout);
    establishProtocol(readTimeout, writeTimeout, connectionSpecSelector);
  }
```
建立socket连接，然后建立Protocol。建立Protocol的过程为：
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

HTTP/2协议的协商过程在connectTls()的过程中完成。

总结一下OkHttp3的连接RealConnection的含义，或者说是ConnectInterceptor从StreamAllocation中获取的RealConnection对象的状态：
1. 对于不使用HTTP代理的HTTP请求，为一个到HTTP服务器的Socket连接。后续直接向该Socket连接中写入常规的HTTP请求，并从中读取常规的HTTP响应。
2. 对于不使用代理的https请求，为一个到https服务器的Socket连接，但经过了TLS握手，协议协商等过程。后续直接向该Socket连接中写入常规的请求，并从中读取常规的响应。
3. 对于使用HTTP代理的HTTP请求，为一个到HTTP代理服务器的Socket连接。后续直接向该Socket连接中写入常规的HTTP请求，并从中读取常规的HTTP响应。
4. 对于使用代理的https请求，为一个到代理服务器的隧道连接，但经过了TLS握手，协议协商等过程。后续直接向该Socket连接中写入常规的请求，并从中读取常规的响应。

关于HTTP代理的更多内容，可以参考[HTTP 代理原理及实现（一）](https://imququ.com/post/web-proxy.html)。

OkHttp3中对路由的处理大体如此。
