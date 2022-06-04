---
title: Chromium net design
date: 2016-10-11 17:17:49
categories: 网络协议
tags:
- 网络协议
- chromium
- 翻译
---

# 总览

网络栈主要地是一个单线程跨平台的库，主要负责资源获取。它的主要接口是`URLRequest`和`URLRequestContext`。`URLRequest`，
正如它的名字所表明的那样，表示一个[URL](http://en.wikipedia.org/wiki/URL)的请求。`URLRequestContext`包含实现URL请求所需的所有相关上下文，比如[cookies](http://en.wikipedia.org/wiki/HTTP_cookie)，主机解析器，代理解析器，[cache](http://dev.chromium.org/developers/design-documents/network-stack/http-cache)，等等。多个`URLRequest`对象可以共享相同的`URLRequestContext`。大多数的net对象不是线程安全的，尽管磁盘缓存可以使用一个专门的线程，而一些组件（主机解析，证书验证等等）可以使用unjoined worker线程。由于它主要运行于一个单独的网络线程上，因而在网络线程上的操作都不允许阻塞。所以我们通过异步的回调(典型的是CompletionCallback)来使用非阻塞操作。网络栈的代码也会把大多数操作记录到NetLog，它允许消费者把表示操作的说明记录到内存中，并以用户友好的格式用于调试中。

<!--more-->

Chromium开发者编写网络栈的目的是：

 - abstractions允许编写跨平台的抽象。
 - 相对于高层的系统网络库(WinHTTP)可提供的，提供更好的控制。
   * 避免系统库中可能出现的bugs。
   * 方便更好地做性能优化。

# 代码结构
net/base - net实用例程的百宝袋，比如主机解析，cookies，网络改变探测，[SSL](http://en.wikipedia.org/wiki/Transport_Layer_Security)。
net/disk_cache - [Web资源缓存](http://dev.chromium.org/developers/design-documents/network-stack/disk-cache)
net/ftp - FTP实现。这些代码主要是基于老的HTTP实现的
net/http - HTTP实现。
net/ocsp - 当不使用系统库或系统没有提供一个OCSP实现时的[OCSP](http://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol)实现。当前只包含一个基于NSS的实现。
net/proxy - 代理 ([SOCKS](http://en.wikipedia.org/wiki/SOCKS)和HTTP) 配置，解析，脚本获取，等等。
net/quic - QUIC实现。
net/socket - TCP sockets，"SSL sockets"，和socket池的跨平台实现。
net/socket_stream - WebSockets的socket streams。
net/spdy - SPDY实现。
net/url_request - URLRequest，URLRequestContext，和URLRequestJob的实现。
net/websockets - WebSockets实现。

# 网络请求剖析（集中于HTTP）

![Chromium HTTP Network Request Diagram](../images/1315506-a1ebf2e63ed9668b.png)

## URLRequest
```
class URLRequest {
 public:
  // Construct a URLRequest for |url|, notifying events to |delegate|.
  URLRequest(const GURL& url, Delegate* delegate);
  
  // Specify the shared state
  void set_context(URLRequestContext* context);

  // Start the request.  Notifications will be sent to |delegate|.
  void Start();

  // Read data from the request.
  bool Read(IOBuffer* buf, int max_bytes, int* bytes_read);
};

class URLRequest::Delegate {
 public:
  // Called after the response has started coming in or an error occurred.
  virtual void OnResponseStarted(...) = 0;

  // Called when Read() calls complete.
  virtual void OnReadCompleted(...) = 0;
};
```

当启动一个`URLRequest`时，它要做的第一件事就是决定要创建何种类型的`URLRequestJob`。主要的job类型是`URLRequestHttpJob`，它用于实现http://请求。还有许多其它的jobs，比如URLRequestFileJob
 (file://)，URLRequestFtpJob
 (ftp://)，URLRequestDataJob
 (data://)，等等。网络栈将确定适当的job来实现请求。但它给客户端提供了两种方式来定制job的创建：URLRequest::Interceptor和URLRequest::ProtocolFactory。这些相当地多余，除了URLRequest::Interceptor的接口更具可扩展性。在job进行的过程中，它将通知URLRequest，而后者则在需要时通知URLRequest::Delegate。

## URLRequestHttpJob
`URLRequestHttpJob`将首先确定要给HTTP请求设置的cookies，这需要查询请求上下文中的`CookieMonster`。这可以是异步的，因为`CookieMonster`可能是由一个[sqlite](http://en.wikipedia.org/wiki/SQLite)数据库支持的。做完了这些之后，它将请求请求上下文的`HttpTransactionFactory`来创建一个`HttpTransaction`。典型地， [HttpCache](http://dev.chromium.org/developers/design-documents/network-stack/http-cache)将被指定为HttpTransactionFactory。HttpCache将创建一个HttpCache::Transaction来处理HTTP请求。HttpCache::Transaction将首先检查HttpCache (它会检查[磁盘缓存](http://dev.chromium.org/developers/design-documents/network-stack/disk-cache))来查看缓存项是否已经存在。如果存在，则意味着响应已经被缓存了，或者这个缓存项的一个网络事物已经存在，则只是从那个项中读取。如果缓存项不存在，则我们创建它并让HttpCache的HttpNetworkLayer给请求的服务创建一个HttpNetworkTransaction。给HttpNetworkTransaction一个包含执行HTTP请求的上下文状态的HttpNetworkSession。这些状态中的一些来自于`URLRequestContext`。

## HttpNetworkTransaction
```
class HttpNetworkSession {
 ...

 private:
  // Shim so we can mock out ClientSockets.
  ClientSocketFactory* const socket_factory_;
  // Pointer to URLRequestContext's HostResolver.
  HostResolver* const host_resolver_;
  // Reference to URLRequestContext's ProxyService
  scoped_refptr<ProxyService> proxy_service_;
  // Contains all the socket pools.
  ClientSocketPoolManager socket_pool_manager_;
  // Contains the active SpdySessions.
  scoped_ptr<SpdySessionPool> spdy_session_pool_;
  // Handles HttpStream creation.
  HttpStreamFactory http_stream_factory_;
};
```
`HttpNetworkTransaction`请求`HttpStreamFactory`创建一个`HttpStream`。

`HttpStreamFactory`返回一个`HttpStreamRequest`，它被期望处理包括如何建立连接，及一旦连接建立则把它包装为一个与网络直接对话的居间`HttpStream`的子类等所有的逻辑。
```
class HttpStream {
 public:
  virtual int SendRequest(...) = 0;
  virtual int ReadResponseHeaders(...) = 0;
  virtual int ReadResponseBody(...) = 0;
  ...
};
```
当前，只有两种主要的`HttpStream`子类：`HttpBasicStream`和`SpdyHttpStream`，尽管我们已经在计划为[HTTP pipelining](http://en.wikipedia.org/wiki/HTTP_pipelining)创建子类。`HttpBasicStream`假设它在直接地读/写一个socket。`SpdyHttpStream`读和写一个`SpdyStream`。网络事务将会调用流上的方法，并且在完成时，将会调用回调回到`HttpCache::Transaction`，而它将会根据需要通知URLRequestHttpJob和URLRequest。对于HTTP路径，http请求和响应的产生及解析将由`HttpStreamParser`处理。对于SPDY路径，请求和响应的解析由`SpdyStream`和`SpdySession`来处理。基于HTTP响应，`HttpNetworkTransaction`可能需要执行 [HTTP 认证](http://dev.chromium.org/developers/design-documents/http-authentication)。这可能包括重启网络事务。
## HttpStreamFactory
`HttpStreamFactory`首先做代理解析来决定是否需要一个代理。端点被设置为URL主机或代理服务器。然后`HttpStreamFactory`检查`SpdySessionPool`来查看我们是否有这个端点的可用`SpdySession`。如果没有，则stream factory从适当的池中请求一个 "socket"(TCP/proxy/SSL/etc) 。如果socket是一个SSL socket，则它检查[NPN](https://tools.ietf.org/id/draft-agl-tls-nextprotoneg-01.txt)是否指示了一个协议（可能是SPDY），如果是，则使用特定的协议。对于SPDY，我们将检查一个SpdySession是否已经存在，如果是则使用它，否则我们将由这个SSL socket创建一个新的`SpdySession`，并由`SpdySession`创建一个`SpdyStream`，其中包装一个`SpdyHttpStream`。对于HTTP，我们将简单地获取socket，并把它包装进一个`HttpBasicStream`。
### 代理解析
`HttpStreamFactory`查询`ProxyService`来给GURL返回`ProxyInfo`。代理服务首先需要检查它是否有一个最近的代理配置。如果没有，它使用`ProxyConfigService`来为当前的代理设置查询系统。如果代理设置被设置为没有代理或一个特定的代理，则代理解析是很简单的（我们返回没有代理或特定的代理）。否则，我们需要运行[PAC script](http://en.wikipedia.org/wiki/Proxy_auto-config)来确定合适的代理（或lack thereof）。如果我们还没有PAC script，则代理设置将指示我们应该使用[WPAD auto-detection](http://en.wikipedia.org/wiki/Web_Proxy_Autodiscovery_Protocol)，或将指定一个定制PAC url，然后我们将通过`ProxyScriptFetcher`获取PAC script。一旦我们有了PAC script，我们将通过`ProxyResolver`执行它。注意我们使用一个shim `MultiThreadedProxyResolver`对象来把PAC script执行派发给线程，它们将运行一个ProxyResolverV8实例。这是由于PAC script执行可能阻塞主机解析。然而，为了防止一个失速的PAC script执行阻塞了其它的代理解析，我们允许并发地执行多个PAC script（警告： [V8](http://en.wikipedia.org/wiki/V8_(JavaScript_engine))不是线程安全的，因此我们要为javascript bindings获取锁，以使得一个V8 script实例阻塞在主机解析，它释放锁使另一个V8实例可以执行PAC script来为不同的URL解析代理）。
### 连接管理
`HttpStreamRequest`确定了适当的端点（URL端点或代理端点）之后，它需要建立一个连接。它通过确认适当的"socket"池并从中请求一个socket来做到这一点。注意，这里的"socket"基本上表示我们可以读和写以在网络上发送数据的东西。一个SSL socket是在一个传输([TCP](http://en.wikipedia.org/wiki/Transmission_Control_Protocol)) socket之上构建的，并为用户加密/解密原始的TCP数据。不同的socket类型还处理不同的连接设置，对于HTTP/SOCKS代理，SSL握手，等等。Sockets池被设计为层次结构，因而不同的连接设置可能被放在其它sockets的上层。`HttpStream`的实际的底层socket类型可能是不可知的，由于它仅仅需要对socket做读和写。Socket池执行不同的功能。它们实现我们的单个代理，单个主机和单个进程的连接限制。当前单个代理的限制被设置为32个sockets，单个目标主机被设置为6 sockets，单个进程被设置为256个sockets（没有被完全正确的实现，但足够好）。Socket池也从实现抽象了sockets请求，从而给我们提供sockets的"late binding"。一个socket请求可以通过一个全新的连接的socket或一个idle socket（[从一个之前http事务复用](http://en.wikipedia.org/wiki/HTTP_persistent_connection)）来满足。
### 主机解析
注意传输sockets的连接设置不仅仅需要传输(TCP)握手，也可能需要主机解析。HostResolverImpl使用getaddrinfo()来执行主机解析，这是一个阻塞调用，因而解析器在unjoined worker线程上调用这些功能。典型地，主机解析通常包括[DNS](http://en.wikipedia.org/wiki/Domain_Name_System)解析，但可能包括非DNS命名空间，比如[NetBIOS](http://en.wikipedia.org/wiki/NetBIOS)/[WINS](http://en.wikipedia.org/wiki/Windows_Internet_Name_Service)。注意，自写入的事件起，我们将并发的主机解析的数量限制为8，但我们也在寻求优化这个值。HostResolverImpl也包含了一个HostCache，其中缓存最多1000个主机名。
### SSL
SSL sockets需要执行SSL连接设置和证书验证。当前，在所有的平台上，我们使用[NSS](http://www.mozilla.org/projects/security/pki/nss/)的libssl来处理SSL连接逻辑。然而，我们使用平台特有的APIs来做证书验证。我们也将使用一个证书验证缓存，这将把多个对相同证书的证书验证请求联合为一个单独的证书验证任务，也将把结果缓存一段时间。

`SSLClientSocketNSS`严格地依照这个事件顺序(忽略高级的功能，如基于证书验证的[Snap Start](http://tools.ietf.org/html/draft-agl-tls-snapstart-00)或[DNSSEC](http://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions) based certificate verification)：

 - 调用Connect()。我们将基于`SSLConfig`描述的配置和预处理器宏设置NSS的SSL选项。然后我们启动握手。
 - 握手完成。假设我们没有遇到任何错误，我们将使用`CertVerifier`验证服务器的证书。证书验证可能会消耗一些时间，因而`CertVerifier`使用了`WorkerPool`来实际地调用`X509Certificate::Verify()`，这使用平台特有的APIs来实现。

注意chromium具有它自己的NSS程序，它支持一些高级的特性，一些在系统的NSS模块中不需要的特性，比如支持[NPN](http://tools.ietf.org/html/draft-agl-tls-nextprotoneg-00)，[False Start](http://tools.ietf.org/search/draft-bmoeller-tls-falsestart-00)，Snap Start，[OCSP stapling](http://en.wikipedia.org/wiki/OCSP_Stapling)，etc。

TODO: talk about network change notifications

[原文链接](http://dev.chromium.org/developers/design-documents/network-stack)
