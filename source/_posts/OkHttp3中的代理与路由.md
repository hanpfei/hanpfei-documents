---
title: OkHttp3中的代理与路由
date: 2016-10-14 11:43:49
categories: 网络协议
tags:
- 源码分析
- Android开发
- 网络协议
---

HTTP请求的整体处理过程大体可以理解为，
1. 建立TCP连接。
2. 如果是HTTPS的话，完成SSL/TLS的协商。
3. 发送请求。
4. 获取响应。
5. 结束请求，关闭连接。

<!--more-->

然而，当为系统设置了代理的时候，整个数据流都会经过代理服务器。那么代理设置究竟是如何工作的呢？它是如何影响我们上面看到的HTTP请求的处理过程的呢？是在操作系统内核的TCP实现中的策略呢，还是HTTP stack中的机制？这里我们通过OkHttp3中的实现来一探究竟。

代理分为两种类型，一种是SOCKS代理，另一种是HTTP代理。对于SOCKS代理，在HTTP的场景下，代理服务器完成TCP数据包的转发工作。而HTTP代理服务器，在转发数据之外，还会解析HTTP的请求及响应，并根据请求及响应的内容做一些处理。这里看一下OkHttp中对代理的处理。

# 代理服务器的描述
在Java中，通过 ***java.net.Proxy*** 类描述一个代理服务器：
```
public class Proxy {

    /**
     * Represents the proxy type.
     *
     * @since 1.5
     */
    public enum Type {
        /**
         * Represents a direct connection, or the absence of a proxy.
         */
        DIRECT,
        /**
         * Represents proxy for high level protocols such as HTTP or FTP.
         */
        HTTP,
        /**
         * Represents a SOCKS (V4 or V5) proxy.
         */
        SOCKS
    };

    private Type type;
    private SocketAddress sa;

    /**
     * A proxy setting that represents a {@code DIRECT} connection,
     * basically telling the protocol handler not to use any proxying.
     * Used, for instance, to create sockets bypassing any other global
     * proxy settings (like SOCKS):
     * <P>
     * {@code Socket s = new Socket(Proxy.NO_PROXY);}
     *
     */
    public final static Proxy NO_PROXY = new Proxy();

    // Creates the proxy that represents a {@code DIRECT} connection.
    private Proxy() {
        type = Type.DIRECT;
        sa = null;
    }

    /**
     * Creates an entry representing a PROXY connection.
     * Certain combinations are illegal. For instance, for types Http, and
     * Socks, a SocketAddress <b>must</b> be provided.
     * <P>
     * Use the {@code Proxy.NO_PROXY} constant
     * for representing a direct connection.
     *
     * @param type the {@code Type} of the proxy
     * @param sa the {@code SocketAddress} for that proxy
     * @throws IllegalArgumentException when the type and the address are
     * incompatible
     */
    public Proxy(Type type, SocketAddress sa) {
        if ((type == Type.DIRECT) || !(sa instanceof InetSocketAddress))
            throw new IllegalArgumentException("type " + type + " is not compatible with address " + sa);
        this.type = type;
        this.sa = sa;
    }

    /**
     * Returns the proxy type.
     *
     * @return a Type representing the proxy type
     */
    public Type type() {
        return type;
    }

    /**
     * Returns the socket address of the proxy, or
     * {@code null} if its a direct connection.
     *
     * @return a {@code SocketAddress} representing the socket end
     *         point of the proxy
     */
    public SocketAddress address() {
        return sa;
    }
......
}
```
只用代理的类型及代理服务器地址即可描述代理服务器的全部。对于HTTP代理，代理服务器地址可以通过域名和IP地址等方式来描述。

# 代理选择器ProxySelector
在Java中通过ProxySelector为一个特定的URI选择代理：
```
public abstract class ProxySelector {
......
    /**
     * Selects all the applicable proxies based on the protocol to
     * access the resource with and a destination address to access
     * the resource at.
     * The format of the URI is defined as follow:
     * <UL>
     * <LI>http URI for http connections</LI>
     * <LI>https URI for https connections
     * <LI>{@code socket://host:port}<br>
     *     for tcp client sockets connections</LI>
     * </UL>
     *
     * @param   uri
     *          The URI that a connection is required to
     *
     * @return  a List of Proxies. Each element in the
     *          the List is of type
     *          {@link java.net.Proxy Proxy};
     *          when no proxy is available, the list will
     *          contain one element of type
     *          {@link java.net.Proxy Proxy}
     *          that represents a direct connection.
     * @throws IllegalArgumentException if the argument is null
     */
    public abstract List<Proxy> select(URI uri);

    /**
     * Called to indicate that a connection could not be established
     * to a proxy/socks server. An implementation of this method can
     * temporarily remove the proxies or reorder the sequence of
     * proxies returned by {@link #select(URI)}, using the address
     * and the IOException caught when trying to connect.
     *
     * @param   uri
     *          The URI that the proxy at sa failed to serve.
     * @param   sa
     *          The socket address of the proxy/SOCKS server
     *
     * @param   ioe
     *          The I/O exception thrown when the connect failed.
     * @throws IllegalArgumentException if either argument is null
     */
    public abstract void connectFailed(URI uri, SocketAddress sa, IOException ioe);
}
```
这个组件会读区系统中配置的所有代理，并根据调用者传入的URI，返回特定的代理服务器集合。由于不同系统中，配置代理服务器的方法，及相关配置的保存机制不同，该接口在不同的系统中有着不同的实现。

# OkHttp3的路由
OkHttp3中抽象出Route来描述网络数据包的传输路径，最主要还是要描述直接与其建立TCP连接的目标端点。
```
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
......
}
```
主要通过 **代理服务器的信息proxy** ，及 **连接的目标地址** 描述路由。 **连接的目标地址inetSocketAddress** 根据代理类型的不同而有着不同的含义，这主要是由不同代理协议的差异而造成的。对于无需代理的情况， **连接的目标地址inetSocketAddress** 中包含HTTP服务器经过了DNS域名解析的IP地址及协议端口号；对于SOCKS代理，其中包含HTTP服务器的域名及协议端口号；对于HTTP代理，其中则包含代理服务器经过域名解析的IP地址及端口号。

# 路由选择器RouteSelector
HTTP请求处理过程中所需的TCP连接建立过程，主要是找到一个Route，然后依据代理协议的规则与特定目标建立TCP连接。对于无代理的情况，是与HTTP服务器建立TCP连接；对于SOCKS代理及HTTP代理，是与代理服务器建立TCP连接，虽然都是与代理服务器建立TCP连接，而SOCKS代理协议与HTTP代理协议做这个动作的方式又会有一定的区别。

借助于域名解析做负载均衡已经是网络中非常常见的手法了，因而，常常会有相同域名对应不同IP地址的情况。同时相同系统也可以设置多个代理，这使Route的选择变得复杂起来。

在OkHttp中，对Route连接失败有一定的错误处理机制。OkHttp会逐个尝试找到的Route建立TCP连接，直到找到可用的那一个。这同样要求，对Route信息有良好的管理。

OkHttp3借助于 **`RouteSelector`** 类管理所有的路由信息，并帮助选择路由。 **`RouteSelector`** 主要完成3件事：
1. 收集所有可用的路由。
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
......

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
```
收集路由分为两个步骤：第一步收集所有的代理；第二步则是收集特定代理服务器选择情况下的所有 **连接的目标地址** 。
收集代理的过程如上面的这段代码所示，有两种方式，一是外部通过address传入了代理，此时代理集合将包含这唯一的代理。address的代理最终来源于OkHttpClient，我们可以在构造OkHttpClient时设置代理，来指定由该client执行的所有请求经过特定的代理。
另一种方式是，借助于ProxySelector获取多个代理。ProxySelector最终也来源于OkHttpClient，OkHttp的用户当然也可以对此进行配置。但通常情况下，使用系统默认的ProxySelector，来获取系统中配置的代理。
收集到的所有代理保存在列表 **`proxies`** 中。
为OkHttpClient配置Proxy或ProxySelector的场景大概是，需要让连接使用代理，但不使用系统的代理配置的情况。
收集特定代理服务器选择情况下的所有路由，因代理类型的不同而有着不同的过程：
```
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
```
收集一个特定代理服务器选择下的 **连接的目标地址** 因代理类型的不同而不同，这主要分为3种情况。 对于没有配置代理的情况，会对HTTP服务器的域名进行DNS域名解析，并为每个解析到的IP地址创建 **连接的目标地址**；对于SOCKS代理，直接以HTTP服务器的域名及协议端口号创建  **连接的目标地址**；而对于HTTP代理，则会对HTTP代理服务器的域名进行DNS域名解析，并为每个解析到的IP地址创建 **连接的目标地址**。
这里是OkHttp中发生DNS域名解析唯一的场合。对于使用代理的场景，没有对HTTP服务器的域名做DNS域名解析，也就意味着HTTP服务器的域名解析要由代理服务器完成。
代理服务器的收集是在创建 **`RouteSelector`** 完成的；而一个特定代理服务器选择下的 **连接的目标地址** 收集则是在选择Route时根据需要完成的。
2.  **`RouteSelector`** 做的第二件事情是选择可用的路由。
```
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

  /** Returns true if there's another proxy to try. */
  private boolean hasNextProxy() {
    return nextProxyIndex < proxies.size();
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
```
 **`RouteSelector`** 实现了两级迭代器来提供选择路由的服务。
3. 维护接失败的路由的信息，以避免浪费时间去连接一些不可用的路由。 **`RouteSelector`** 借助于**`RouteDatabase`** 维护失败的路由的信息。
```
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
```
`RouteDatabase`是一个简单的容器：
```
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
# 代理选择器ProxySelector的实现
在OkHttp3中，`ProxySelector`对象由OkHttpClient维护。

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

在创建OkHttpClient时，可以通过为OkHttpClient.Builder设置`ProxySelector`来定制`ProxySelector`。若没有指定，则使用系统默认的`ProxySelector`。OpenJDK 1.8版默认的`ProxySelector`为`sun.net.spi.DefaultProxySelector`：
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
Android平台下，默认的`ProxySelector` ProxySelectorImpl，其[实现 (不同Android版本实现不同，这里以android-6.0.1_r61为例)](https://android.googlesource.com/platform/libcore/+/android-6.0.1_r61/luni/src/main/java/java/net/ProxySelectorImpl.java) 如下：
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
在Android平台上，主要是从系统属性System properties中获取代理服务器的配置信息，这里会过滤掉不能进行代理的主机的访问。

前面我们看到 **`RouteSelector`** 通过 **`Address`** 提供的Proxy和ProxySelector来收集Proxy信息及连接的目标地址信息。OkHttp3中用 **`Address`** 描述建立连接所需的配置信息，包括HTTP服务器的地址，DNS，SocketFactory，Proxy，ProxySelector及TLS所需的一些设施等等：
```
public final class Address {
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

  /**
   * Returns a URL with the hostname and port of the origin server. The path, query, and fragment of
   * this URL are always empty, since they are not significant for planning a route.
   */
  public HttpUrl url() {
    return url;
  }

  /** Returns the service that will be used to resolve IP addresses for hostnames. */
  public Dns dns() {
    return dns;
  }

  /** Returns the socket factory for new connections. */
  public SocketFactory socketFactory() {
    return socketFactory;
  }

  /** Returns the client's proxy authenticator. */
  public Authenticator proxyAuthenticator() {
    return proxyAuthenticator;
  }

  /**
   * Returns the protocols the client supports. This method always returns a non-null list that
   * contains minimally {@link Protocol#HTTP_1_1}.
   */
  public List<Protocol> protocols() {
    return protocols;
  }

  public List<ConnectionSpec> connectionSpecs() {
    return connectionSpecs;
  }

  /**
   * Returns this address's proxy selector. Only used if the proxy is null. If none of this
   * selector's proxies are reachable, a direct connection will be attempted.
   */
  public ProxySelector proxySelector() {
    return proxySelector;
  }

  /**
   * Returns this address's explicitly-specified HTTP proxy, or null to delegate to the {@linkplain
   * #proxySelector proxy selector}.
   */
  public Proxy proxy() {
    return proxy;
  }

  /** Returns the SSL socket factory, or null if this is not an HTTPS address. */
  public SSLSocketFactory sslSocketFactory() {
    return sslSocketFactory;
  }

  /** Returns the hostname verifier, or null if this is not an HTTPS address. */
  public HostnameVerifier hostnameVerifier() {
    return hostnameVerifier;
  }

  /** Returns this address's certificate pinner, or null if this is not an HTTPS address. */
  public CertificatePinner certificatePinner() {
    return certificatePinner;
  }

......
}
```

OkHttp3中通过职责链执行HTTP请求。在其中的RetryAndFollowUpInterceptor里创建Address对象时，从OkHttpClient对象获取ProxySelector。Address对象会被用于创建StreamAllocation对象。StreamAllocation在建立连接时，从Address对象中获取ProxySelector以选择路由。
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

在StreamAllocation中，Address对象会被用于创建 **`RouteSelector`** 对象：
```
public final class StreamAllocation {
......

  public StreamAllocation(ConnectionPool connectionPool, Address address) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.routeSelector = new RouteSelector(address, routeDatabase());
  }
```

# 代理协议
如我们在 [OkHttp3 HTTP请求执行流程分析](http://www.jianshu.com/p/230e2e2988e0) 中看到的，OkHttp3对HTTP请求是通过Interceptor链来处理的。
`RetryAndFollowUpInterceptor`创建`StreamAllocation`对象，处理http的重定向及出错重试。对后续Interceptor的执行的影响为修改Request并创建StreamAllocation对象。
`BridgeInterceptor`补全缺失的一些http header。对后续Interceptor的执行的影响主要为修改了Request。
`CacheInterceptor`处理http缓存。对后续Interceptor的执行的影响为，若缓存中有所需请求的响应，则后续Interceptor不再执行。
`ConnectInterceptor`借助于前面分配的`StreamAllocation`对象建立与服务器之间的连接，并选定交互所用的协议是HTTP 1.1还是HTTP 2。对后续Interceptor的执行的影响为，创建了HttpStream和connection。
`CallServerInterceptor`作为Interceptor链中的最后一个Interceptor，用于处理IO，与服务器进行数据交换。

在OkHttp3中，收集的路由信息，是在`ConnectInterceptor`中建立连接时用到的。`ConnectInterceptor` 借助于 **`StreamAllocation`** 完成整个连接的建立，包括TCP连接建立，代理协议所要求的协商，以及SSL/TLS协议的协商，如ALPN等。我们暂时略过整个连接建立的完整过程，主要关注TCP连接建立及代理协议的协商过程的部分。

 **`StreamAllocation`** 的findConnection()用来为某次特定的网络请求寻找一个可用的连接。
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
    acquire(newConnection);

    synchronized (connectionPool) {
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
OkHttp3中有一套连接池的机制，这里先尝试从连接池中寻找可用的连接，找不到时才会新建连接。新建连接的过程是：
1. 选择一个Route；
2. 创建 **`RealConnection`** 连接对象。
3. 将连接对象保存进连接池中。
4. 建立连接。

**`RealConnection`** 中建立连接的过程是这样的：
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
在这个方法中，SSLSocketFactory为空，也就是要求请求/响应明文传输时，先做安全性检查，以确认系统允许明文传输，允许以请求的域名做明文传输。

然后根据路由的具体情况，执行不同的连接建立过程。对于需要创建隧道连接的路由，执行buildTunneledConnection()，对于其它情况，则执行buildConnection()。

判断是否要建立隧道连接的依据是代理的类型，以及连接的类型：
```
  /**
   * Returns true if this route tunnels HTTPS through an HTTP proxy. See <a
   * href="http://www.ietf.org/rfc/rfc2817.txt">RFC 2817, Section 5.2</a>.
   */
  public boolean requiresTunnel() {
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
  }
```
如果是HTTP代理，且请求建立SSL/TLS加密通道 (http/1.1的https和http2) ，则需要建立隧道连接。其它情形不需要建立隧道连接。

## 非隧道连接的建立
非隧道连接的建立过程为：
```
  /** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  private void buildConnection(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    connectSocket(connectTimeout, readTimeout);
    establishProtocol(readTimeout, writeTimeout, connectionSpecSelector);
  }

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
有 3 种情况需要建立非隧道连接：
1. 无代理。
2. 明文的HTTP代理。
3. SOCKS代理。

非隧道连接的建立过程为建立TCP连接，然后在需要时完成SSL/TLS的握手及HTTP/2的握手建立Protocol。建立TCP连接的过程为：
1. 创建Socket。非SOCKS代理的情况下，通过SocketFactory创建；在SOCKS代理则传入proxy手动new一个出来。
2. 为Socket设置读超时。
3. 完成特定于平台的连接建立。
4. 创建用语IO的source和sink。

**`AndroidPlatform`** 的 **`connectSocket()`** 是这样的：
```
  @Override public void connectSocket(Socket socket, InetSocketAddress address,
      int connectTimeout) throws IOException {
    try {
      socket.connect(address, connectTimeout);
    } catch (AssertionError e) {
      if (Util.isAndroidGetsocknameError(e)) throw new IOException(e);
      throw e;
    } catch (SecurityException e) {
      // Before android 4.3, socket.connect could throw a SecurityException
      // if opening a socket resulted in an EACCES error.
      IOException ioException = new IOException("Exception in connect");
      ioException.initCause(e);
      throw ioException;
    }
  }
```

设置了SOCKS代理的情况下，仅有的特别之处在于，是通过传入proxy手动创建的Socket。route的socketAddress包含着目标HTTP服务器的域名。由此可见SOCKS协议的处理，主要是在Java标准库的 **`java.net.Socket`** 中处理的。对于外界而言，就好像是于HTTP服务器直接建立连接一样，因为连接时传入的地址都是HTTP服务器的域名。

而对于明文HTTP代理的情况下，这里没有任何特殊的处理。route的socketAddress包含着代理服务器的IP地址。HTTP代理自身会根据请求及响应的实际内容，建立与HTTP服务器的TCP连接，并转发数据。猜测HTTP代理服务器是根据HTTP请求中的"Host"等header内容来确认HTTP服务器地址的。

暂时先略过对建立协议过程的分析。

## HTTP代理的隧道连接
buildTunneledConnection()用于建立隧道连接：
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
这里主要是两个过程：
1. 建立隧道连接。
2. 建立Protocol。

建立隧道连接的过程又分为几个步骤：

 - 创建隧道请求
 - 建立Socket连接
 - 发送请求建立隧道

隧道请求是一个常规的HTTP请求，只是请求的内容有点特殊。最初创建的隧道请求如：
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
一个隧道请求的例子如下：

![Tunnel Request](https://www.wolfcstech.com/images/1315506-f2b7ee35c56c732b.jpg)

请求的"Host" header中包含了目标HTTP服务器的域名。建立socket连接的过程这里不再赘述。

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
在前面创建的TCP连接之上，完成与代理服务器的HTTP请求/响应交互。请求的内容类似下面这样：
```
"CONNECT m.taobao.com:443 HTTP/1.1"
```
这里可能会根据HTTP代理是否需要认证而有多次HTTP请求/响应交互。

总结一下OkHttp3中代理相关的处理：
1. 没有设置代理的情况下，直接与HTTP服务器建立TCP连接，然后进行HTTP请求/响应的交互。
2. 设置了SOCKS代理的情况下，创建Socket时，为其传入proxy，连接时还是以HTTP服务器为目标地址。在标准库的Socket中完成SOCKS协议相关的处理。此时基本上感知不到代理的存在。
3. 设置了HTTP代理时的HTTP请求，与HTTP代理服务器建立TCP连接。HTTP代理服务器解析HTTP请求/响应的内容，并根据其中的信息来完成数据的转发。也就是说，如果HTTP请求中不包含"Host" header，则有可能在设置了HTTP代理的情况下无法与HTTP服务器建立连接。
4. 设置了HTTP代理时的HTTPS/HTTP2请求，与HTTP服务器建立通过HTTP代理的隧道连接。HTTP代理不再解析传输的数据，仅仅完成数据转发的功能。此时HTTP代理的功能退化为如同SOCKS代理类似。
5. 设置了代理时，HTTP服务器的域名解析会被交给代理服务器执行。其中设置了HTTP代理时，会对HTTP代理的域名做域名解析。

关于HTTP代理的更多内容，可以参考[HTTP 代理原理及实现（一）](https://imququ.com/post/web-proxy.html)。

OkHttp3中代理相关的处理大体如此。
