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

Chromium 网络栈几乎是个单线程的跨平台库，它主要用于资源获取。它的主要接口是 `URLRequest` 和 `URLRequestContext`。`URLRequest`，就像它的名字所表明的那样，表示对 [URL](http://en.wikipedia.org/wiki/URL) 的请求。`URLRequestContext` 包含实现 URL 请求所需的所有相关上下文，比如 [cookies](http://en.wikipedia.org/wiki/HTTP_cookie)、域名解析器、代理解析器、[cache](http://dev.chromium.org/developers/design-documents/network-stack/http-cache) 等等。多个 `URLRequest` 对象可以共享同一个 `URLRequestContext`。大多数的 `net` 对象不是线程安全的，尽管磁盘缓存可以使用专门的线程，及一些组件（域名解析，证书验证等）可以使用 unjoined worker 线程。由于它主要运行在单个网络线程上，因而网络线程上的操作都不允许阻塞。我们通过异步回调（通常是 `CompletionCallback`）使用非阻塞操作。网络栈的代码也会把大多数操作记录到 `NetLog`，它允许消费者把操作记录到内存中，并以用户友好的格式用于调试中。

<!--more-->

Chromium 开发者编写网络栈的目的是：

 - 允许编写跨平台的抽象。

 - 相对于更高层的系统网络库（如 WinHTTP 或 WinINET）可以提供的，提供更好的控制。

   * 避免系统库中可能出现的 bugs。

   * 方便更好地做性能优化。

# 代码结构

 * net/base - `net` 实用例程百宝袋，比如域名解析、cookies、网络改变探测、[SSL](http://en.wikipedia.org/wiki/Transport_Layer_Security)。
 
 * net/disk_cache - [Web 资源缓存](http://dev.chromium.org/developers/design-documents/network-stack/disk-cache)
 
 * net/ftp - [FTP](http://en.wikipedia.org/wiki/File_Transfer_Protocol) 实现。代码主要基于老的 HTTP 实现。

 * net/http - [HTTP](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) 实现。

 * net/ocsp - 当不使用系统库或系统没有提供 OCSP 实现时的 [OCSP](http://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol) 实现。当前只包含一个基于 NSS 的实现。

 * net/proxy - 代理 ([SOCKS](http://en.wikipedia.org/wiki/SOCKS) 和 HTTP) 配置、解析、脚本获取等。

 * net/quic - [QUIC](https://www.chromium.org/quic) 实现。

 * net/socket - [TCP](http://en.wikipedia.org/wiki/Transmission_Control_Protocol) sockets、"SSL sockets" 和 socket 池的跨平台实现。

 * net/socket_stream - WebSockets 的 socket 流。

 * net/spdy - HTTP2（及其前身）[SPDY](https://www.chromium.org/spdy) 的实现。

 * net/url_request - `URLRequest`、`URLRequestContext` 和 `URLRequestJob` 的实现。

 * net/websockets - [WebSockets](http://en.wikipedia.org/wiki/WebSockets) 实现。

# 网络请求剖析（聚焦于 HTTP）

![Chromium HTTP Network Request Diagram](../images/1315506-a1ebf2e63ed9668b.png)

## URLRequest

```
class URLRequest {
 public:
  // Construct a URLRequest for |url|, notifying events to |delegate|.
  URLRequest(const GURL& url, Delegate* delegate);

  // Specify the shared state
  void set_context(URLRequestContext* context);

  // Start the request. Notifications will be sent to |delegate|.
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

当启动一个 `URLRequest` 时，它要做的第一件事是决定创建什么类型的 `URLRequestJob`。主要的 job 类型是 `URLRequestHttpJob`，它用于实现 **http://** 请求。还有许多其它的 jobs，如 `URLRequestFileJob` (**file://**)、`URLRequestFtpJob` (**ftp://**)，`URLRequestDataJob` (**data://**) 等。网络栈将选择适当的 job 来实现请求，但它给客户端提供了两种方式来定制 job 的创建：`URLRequest::Interceptor` 和 `URLRequest::ProtocolFactory`。这些都是相当多余的，除了 `URLRequest::Interceptor` 的接口更具可扩展性。在 job 进行过程中，它将通知 `URLRequest`，而后者则在有需要时通知 `URLRequest::Delegate`。

## URLRequestHttpJob

`URLRequestHttpJob`将首先确定要给 HTTP 请求设置的 cookies，这需要查询请求上下文中的 `CookieMonster`。这可以是异步的，因为 `CookieMonster` 可能是由一个 [sqlite](http://en.wikipedia.org/wiki/SQLite) 数据库在支持。做完了这些之后，它将让请求上下文的 `HttpTransactionFactory` 创建一个 `HttpTransaction`。通常会把 [HttpCache](http://dev.chromium.org/developers/design-documents/network-stack/http-cache) 指定为`HttpTransactionFactory`。`HttpCache` 将创建一个 `HttpCache::Transaction` 来处理 HTTP 请求。`HttpCache::Transaction` 将首先检查 `HttpCache` (它会检查 [磁盘缓存](http://dev.chromium.org/developers/design-documents/network-stack/disk-cache)) 来查看缓存项是否已经存在。如果存在，则意味着响应已经被缓存了，或者这个缓存项的网络事务已经存在，则只从该缓存项读取。如果缓存项不存在，则我们创建它，并让 `HttpCache` 的 `HttpNetworkLayer` 创建一个 `HttpNetworkTransaction`，以服务发起的请求。给 `HttpNetworkTransaction` 赋予一个 `HttpNetworkSession`，`HttpNetworkSession` 包含执行 HTTP 请求的上下文状态。这种状态中的一些来自于 `URLRequestContext`。

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

`HttpNetworkTransaction` 请求 `HttpStreamFactory` 创建 `HttpStream`。

`HttpStreamFactory` 返回 `HttpStreamRequest`，`HttpStreamRequest` 应该处理包括如何建立连接，及一旦连接建立，则把它包装为一个居间与网络直接对话的 `HttpStream`子类等所有的逻辑。

```
class HttpStream {
 public:
  virtual int SendRequest(...) = 0;
  virtual int ReadResponseHeaders(...) = 0;
  virtual int ReadResponseBody(...) = 0;
  ...
};
```

当前，只有两种主要的 `HttpStream` 子类：`HttpBasicStream` 和 `SpdyHttpStream`，尽管我们已经在计划为 [HTTP pipelining](http://en.wikipedia.org/wiki/HTTP_pipelining) 创建子类。`HttpBasicStream` 假设它直接读/写 socket。`SpdyHttpStream` 读写 `SpdyStream`。网络事务将调用流的方法，并且在完成时，将调用回调回到 `HttpCache::Transaction`，而它将根据需要通知 `URLRequestHttpJob` 和 `URLRequest`。对于 HTTP 路径，http 请求和响应的产生及解析将由 `HttpStreamParser` 处理。对于 SPDY 路径，请求和响应的解析由 `SpdyStream` 和 `SpdySession` 处理。基于 HTTP 响应，`HttpNetworkTransaction` 可能需要执行 [HTTP 认证](http://dev.chromium.org/developers/design-documents/http-authentication)。这可能包括重启网络事务。

## HttpStreamFactory

`HttpStreamFactory` 首先执行代理解析，以决定是否需要代理。端点被设置为 URL 主机或代理服务器。`HttpStreamFactory` 然后检查 `SpdySessionPool`，来查看我们是否有这个端点的可用 `SpdySession`。如果没有，则 stream factory 从适当的池中请求一个 "socket" (TCP/proxy/SSL/etc) 。如果 socket 是一个 SSL socket，则它检查 [NPN](https://tools.ietf.org/id/draft-agl-tls-nextprotoneg-01.txt) 是否指示了协议（可能是 SPDY），如果是，则使用指定的协议。对于 SPDY，我们将检查是否有一个 `SpdySession` 已经存在，如果是则使用它，否则我们将从这个 SSL socket 创建一个新的 `SpdySession`，并从 `SpdySession` 创建一个 `SpdyStream`，其中包装了一个 `SpdyHttpStream`。对于 HTTP，我们将简单地获取 socket，并把它包装进 `HttpBasicStream`。

### 代理解析

`HttpStreamFactory` 为 GURL 查询 `ProxyService`来返回 `ProxyInfo`。代理服务首先需要检查它是否有一个最新的代理配置。如果没有，它使用 `ProxyConfigService` 来查询系统，以获得当前的代理设置。如果代理设置被设置为没有代理或一个特定的代理，则代理解析是很简单的（我们返回没有代理或特定的代理）。否则，我们需要运行 [PAC 脚本](http://en.wikipedia.org/wiki/Proxy_auto-config) 来确定合适的代理（或没有代理）。如果我们还没有 PAC 脚本，则代理设置将指示我们应该使用 [WPAD auto-detection](http://en.wikipedia.org/wiki/Web_Proxy_Autodiscovery_Protocol)，或者将指定一个定制的 PAC url，然后我们将通过 `ProxyScriptFetcher` 获取 PAC 脚本。一旦我们获得了 PAC 脚本，我们将通过 `ProxyResolver` 执行它。注意我们使用一个闪烁的 `MultiThreadedProxyResolver` 对象来把 PAC 脚本的执行派发给线程，它运行一个 `ProxyResolverV8` 实例。这是由于 PAC 脚本的执行可能阻塞主机解析。然而，为了防止一个停滞的 PAC 脚本执行阻塞其它的代理解析，我们允许并发地执行多个 PAC 脚本（警告： [V8](http://en.wikipedia.org/wiki/V8_(JavaScript_engine)) 不是线程安全的，因此我们要为 javascript 绑定获取锁，这样一个 V8 实例阻塞在主机解析时，它释放锁，然后另一个 V8 实例可以执行 PAC 脚本来为不同的 URL 解析代理）。

### 连接管理

`HttpStreamRequest` 确定了适当的端点（URL 端点或代理端点）之后，它需要建立一个连接。它通过确认适当的 "socket" 池并从中请求一个 socket 来执行这一点。注意，这里的 "socket" 基本上表示我们可以读和写，以通过网络发送数据的东西。SSL socket 基于传输 ([TCP](http://en.wikipedia.org/wiki/Transmission_Control_Protocol)) socket 构建，并为用户加密/解密原始的 TCP 数据。不同的 socket 类型还处理不同的连接设置，如 HTTP/SOCKS 代理，SSL 握手等。Sockets 池被设计为分层的，因而各种各样的连接设置可能被放在其它 sockets 的上层。`HttpStream` 可能不知道其实际的底层 socket 类型，因为它仅仅需要读和写 socket。Socket 池执行大量的函数。它们实现我们的各个代理，各个主机和单个进程的连接限制。当前单个代理的限制被设置为 32 个sockets，单个目标主机被设置为 6 个 sockets，单个进程被设置为 256 个sockets（没有完全正确地实现，但已经足够好）。Socket 池也从实现抽象了 sockets 请求，从而给我们提供了 sockets 的 "late binding"。一个 socket 请求可以通过一个新连接的 socket 或一个 idle socket（[从之前的 http 事务复用](http://en.wikipedia.org/wiki/HTTP_persistent_connection)）来满足。

### 主机解析

注意传输 sockets 的连接设置不仅需要传输 (TCP) 握手，也可能需要主机解析。`HostResolverImpl` 使用包括 `getaddrinfo()` 在内的各种机制来执行主机解析，这是一个阻塞调用，因而解析器在 unjoined worker 线程上调用这些函数。通常主机解析常常包括 [DNS](http://en.wikipedia.org/wiki/Domain_Name_System) 解析，但可能包括非 DNS 命名空间，比如 [NetBIOS](http://en.wikipedia.org/wiki/NetBIOS) / [WINS](http://en.wikipedia.org/wiki/Windows_Internet_Name_Service)。注意，自相关代码编写起，我们将并发主机解析数量限制为 8，但我们也在寻求优化这个值。`HostResolverImpl` 也包含了一个 `HostCache`，其中可以缓存最多 1000 个主机名。

### SSL

Chromium 网络栈使用 [BoringSSL](https://boringssl.googlesource.com/boringssl/) 来处理 TLS 链接逻辑。`StreamSocket` 类和 BoringSSL 的桥接在 `SSLClientSocketImpl`。

SSL sockets 需要执行 SSL 连接设置和证书验证。当前，在所有的平台上，我们使用 [NSS](http://www.mozilla.org/projects/security/pki/nss/) 的 libssl 来处理 SSL 连接逻辑。然而，我们使用平台特有的 APIs 来做证书验证。我们也将使用一个证书验证缓存，这将把多个对相同证书的证书验证请求合并为单个证书验证任务，也将把结果缓存一段时间。

`SSLClientSocketNSS` 严格地遵循如下的事件顺序 (忽略高级功能，如基于证书验证的 [Snap Start](http://tools.ietf.org/html/draft-agl-tls-snapstart-00) 或 [DNSSEC](http://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions))：

 - 调用 `Connect()`。我们将基于 `SSLConfig` 描述的配置和预处理器宏设置 NSS 的 SSL 选项。然后我们启动握手。
 
 - 握手完成。假设我们没有遇到任何错误，我们将使用 `CertVerifier` 验证服务器的证书。证书验证可能会消耗一些时间，因而 `CertVerifier` 使用了 `WorkerPool` 来实际地调用 `X509Certificate::Verify()`，这使用平台特有的 APIs 实现。

注意 chromium 具有它自己的 NSS 程序，它支持一些高级的特性，一些在系统的 NSS 模块中不需要的特性，比如支持 [NPN](http://tools.ietf.org/html/draft-agl-tls-nextprotoneg-00)，[False Start](http://tools.ietf.org/search/draft-bmoeller-tls-falsestart-00)，Snap Start，[OCSP stapling](http://en.wikipedia.org/wiki/OCSP_Stapling)，等等。

TODO: talk about network change notifications

[原文链接](http://dev.chromium.org/developers/design-documents/network-stack)
