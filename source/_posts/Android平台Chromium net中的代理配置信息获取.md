---
title: Android平台Chromium net中的代理配置信息获取
date: 2017-02-09 17:17:49
categories:
- Chromium
- 网络
---

在计算机网络中，***代理服务器*** 扮演着发起请求的客户端与服务器之间的中间人的角色。客户端连接到代理服务器，请求一些服务，比如文件，网页，或其它可以从服务器获得的资源，代理服务器以简化和控制复杂度的形式获取请求的响应。代理被发明以为分布式系统添加结构和封装。
<!--more-->
在我们做移动端开发时，代理常常可以作为我们网络调试的利器。然而我们设置的代理究竟是如何对网络访问的整个过程产生影响的呢？本文将尝试回答这个问题。

# 系统静态代理服务器信息的解析
一个HTTP请求的执行过程，大体为：
0. 连接准备。
1. 建立TCP连接。
2. 如果是HTTPS的话，完成SSL/TLS的握手。
3. 如果是HTTP2的话，在SSL/TLS握手完成之后，执行HTTP2的协商。
4. 发送请求。
5. 获取响应。
6. 结束请求，关闭连接。

与代理相关的处理，主要发生在上面的连接准备与连接建立阶段，这主要包括解析系统中保存的静态代理服务器设置信息，以及以特有的方式建立与代理之间的连接。

Chromium net在 ***HttpStreamFactoryImpl::Job::DoLoop(int result)*** 中执行解析代理信息、建立连接、处理TLS握手/HTTP2握手/QUIC握手，并创建Stream的过程：
```
int HttpStreamFactoryImpl::Job::DoLoop(int result) {
  DCHECK_NE(next_state_, STATE_NONE);
  int rv = result;
  do {
    State state = next_state_;
    next_state_ = STATE_NONE;
    switch (state) {
      case STATE_START:
        DCHECK_EQ(OK, rv);
        rv = DoStart();
        break;
      case STATE_RESOLVE_PROXY:
        DCHECK_EQ(OK, rv);
        rv = DoResolveProxy();
        break;
      case STATE_RESOLVE_PROXY_COMPLETE:
        rv = DoResolveProxyComplete(rv);
        break;
      case STATE_WAIT:
        DCHECK_EQ(OK, rv);
        rv = DoWait();
        break;
      case STATE_WAIT_COMPLETE:
        rv = DoWaitComplete(rv);
        break;
      case STATE_INIT_CONNECTION:
        DCHECK_EQ(OK, rv);
        rv = DoInitConnection();
        break;
      case STATE_INIT_CONNECTION_COMPLETE:
        rv = DoInitConnectionComplete(rv);
        break;
      case STATE_WAITING_USER_ACTION:
        rv = DoWaitingUserAction(rv);
        break;
      case STATE_RESTART_TUNNEL_AUTH:
        DCHECK_EQ(OK, rv);
        rv = DoRestartTunnelAuth();
        break;
      case STATE_RESTART_TUNNEL_AUTH_COMPLETE:
        rv = DoRestartTunnelAuthComplete(rv);
        break;
```
具体到系统静态代理服务器设置信息的解析，
```
int HttpStreamFactoryImpl::Job::DoResolveProxy() {
  DCHECK(!pac_request_);
  DCHECK(session_);

  next_state_ = STATE_RESOLVE_PROXY_COMPLETE;

  if (request_info_.load_flags & LOAD_BYPASS_PROXY) {
    proxy_info_.UseDirect();
    return OK;
  }

  // TODO(rch): remove this code since Alt-Svc seems to prohibit it.
  GURL url_for_proxy = origin_url_;

  // For SPDY via Alt-Svc, set |alternative_service_url_| to
  // https://<alternative host>:<alternative port>/...
  // so the proxy resolution works with the actual destination, and so
  // that the correct socket pool is used.
  if (IsSpdyAlternative()) {
    // TODO(rch):  Figure out how to make QUIC iteract with PAC
    // scripts.  By not re-writing the URL, we will query the PAC script
    // for the proxy to use to reach the original URL via TCP.  But
    // the alternate request will be going via UDP to a different port.
    GURL::Replacements replacements;
    // new_port needs to be in scope here because GURL::Replacements references
    // the memory contained by it directly.
    const std::string new_port = base::UintToString(alternative_service_.port);
    replacements.SetSchemeStr("https");
    replacements.SetPortStr(new_port);
    url_for_proxy = url_for_proxy.ReplaceComponents(replacements);
  }

  return session_->proxy_service()->ResolveProxy(
      url_for_proxy, request_info_.method, &proxy_info_, io_callback_,
      &pac_request_, session_->params().proxy_delegate, net_log_);
}

int HttpStreamFactoryImpl::Job::DoResolveProxyComplete(int result) {
  pac_request_ = NULL;

  if (result == OK) {
    // Remove unsupported proxies from the list.
    int supported_proxies =
        ProxyServer::SCHEME_DIRECT | ProxyServer::SCHEME_HTTP |
        ProxyServer::SCHEME_HTTPS | ProxyServer::SCHEME_SOCKS4 |
        ProxyServer::SCHEME_SOCKS5;

    if (session_->params().enable_quic)
      supported_proxies |= ProxyServer::SCHEME_QUIC;

    proxy_info_.RemoveProxiesWithoutScheme(supported_proxies);

    if (proxy_info_.is_empty()) {
      // No proxies/direct to choose from. This happens when we don't support
      // any of the proxies in the returned list.
      result = ERR_NO_SUPPORTED_PROXIES;
    } else if (using_quic_ &&
               (!proxy_info_.is_quic() && !proxy_info_.is_direct())) {
      // QUIC can not be spoken to non-QUIC proxies.  This error should not be
      // user visible, because the non-alternative Job should be resumed.
      result = ERR_NO_SUPPORTED_PROXIES;
    }
  }

  if (result != OK) {
    return result;
  }

  next_state_ = STATE_WAIT;
  return OK;
}
```
Chromium net在对请求的代理的处理上比较灵活，它允许为请求设置一个标记 LOAD_BYPASS_PROXY ，以使该请求的执行总是绕过代理。在`HttpStreamFactoryImpl::Job::DoResolveProxy()` 中会首先检查请求是否设置了这个标记，若设置，则将与服务器直连而立即返回，不再执行后面解析系统代理服务器设置系统的过程。否则继续执行。

对于代理的使用，用户通常都可以设置一些规则，比如代理的类型，比如对设置对某些域名的访问不使用代理等等。因而对于适当的代理的选择，是根据设置的规则和要访问的URL进行的。Alternative-Service是一种用于支持新协议，比如HTTP2，SPDY和QUIC这种，的机制。这种机制通过服务器向客户端返回一个 "Alt-Svc" 头部字段以表明服务器期望客户端采用的新协议。如果要使用新协议，则发送请求的URL可能会有一定的改变。在`HttpStreamFactoryImpl::Job::DoResolveProxy()` 中，若要使用 "Alt-Svc" SPDY/HTTP2，会先对原始的Url做一定的修饰，并以修饰后的Url为基础去选择代理。

最后通过ProxyService解析代理信息，选择代理服务器。

解析代理之后，执行的`HttpStreamFactoryImpl::Job::DoResolveProxyComplete()` 主要是对解析的结果做检查。在这里会过滤掉不支持的代理，并返回最终的检查结果。为了保持处理逻辑的简便统一，即使没有设置任何代理服务器，解析的代理服务器列表也不会是空的，而是包含一个类型为DIRECT的代理设置。

# 默认的ProxyService
`HttpStreamFactoryImpl::Job::DoResolveProxy()` 所用到的 `ProxyService` 来自于`HttpNetworkSession`。而 `HttpNetworkSession` 的 `ProxyService` 则是通过如下过程一步一步从 `URLRequestContextBuilder` 传过来的：

```
HttpStreamFactoryImpl::Job::Job()
<- DefaultJobFactory::CreateJob() 
<- HttpStreamFactoryImpl::HttpStreamFactoryImpl()
<- HttpNetworkSession::HttpNetworkSession(const Params& params)
<- URLRequestContextBuilder::Build()
```

在 `URLRequestContextBuilder::Build()` 中可以看到如下的几行代码：
```
void URLRequestContextBuilder::SetHttpNetworkSessionComponents(
    const URLRequestContext* context,
    HttpNetworkSession::Params* params) {
  params->host_resolver = context->host_resolver();
  params->cert_verifier = context->cert_verifier();
  params->transport_security_state = context->transport_security_state();
  params->cert_transparency_verifier = context->cert_transparency_verifier();
  params->ct_policy_enforcer = context->ct_policy_enforcer();
  params->proxy_service = context->proxy_service();
  params->ssl_config_service = context->ssl_config_service();
  params->http_auth_handler_factory = context->http_auth_handler_factory();
  params->http_server_properties = context->http_server_properties();
  params->net_log = context->net_log();
  params->channel_id_service = context->channel_id_service();
}
. . . . . .
std::unique_ptr<URLRequestContext> URLRequestContextBuilder::Build() {
  std::unique_ptr<ContainerURLRequestContext> context(
      new ContainerURLRequestContext(file_task_runner_));
  URLRequestContextStorage* storage = context->storage();
. . . . . .
  if (!proxy_service_) {
    // TODO(willchan): Switch to using this code when
    // ProxyService::CreateSystemProxyConfigService()'s signature doesn't suck.
#if !defined(OS_LINUX) && !defined(OS_ANDROID)
    if (!proxy_config_service_) {
      proxy_config_service_ = ProxyService::CreateSystemProxyConfigService(
          base::ThreadTaskRunnerHandle::Get().get(),
          context->GetFileTaskRunner());
    }
#endif  // !defined(OS_LINUX) && !defined(OS_ANDROID)
    proxy_service_ = ProxyService::CreateUsingSystemProxyResolver(
        std::move(proxy_config_service_),
        0,  // This results in using the default value.
        context->net_log());
  }
  storage->set_proxy_service(std::move(proxy_service_));
```
然而，对于Android而言，使用的并不是这里创建的  `ProxyService` 。`ProxyConfigService` 和 `ProxyService` 都是在更早的时候创建的。`ProxyConfigService` 创建的位置 (components/cronet/android/cronet_url_request_context_adapter.cc) 如下：
```
void CronetURLRequestContextAdapter::InitRequestContextOnMainThread(
    JNIEnv* env,
    const JavaParamRef<jobject>& jcaller) {
  base::android::ScopedJavaGlobalRef<jobject> jcaller_ref;
  jcaller_ref.Reset(env, jcaller);
  proxy_config_service_ = net::ProxyService::CreateSystemProxyConfigService(
      GetNetworkTaskRunner(), nullptr /* Ignored on Android */);
  net::ProxyConfigServiceAndroid* android_proxy_config_service =
      static_cast<net::ProxyConfigServiceAndroid*>(proxy_config_service_.get());
  // If a PAC URL is present, ignore it and use the address and port of
  // Android system's local HTTP proxy server. See: crbug.com/432539.
  // TODO(csharrison) Architect the wrapper better so we don't need to cast for
  // android ProxyConfigServices.
  android_proxy_config_service->set_exclude_pac_url(true);
  g_net_log.Get().EnsureInitializedOnMainThread();
  GetNetworkTaskRunner()->PostTask(
      FROM_HERE,
      base::Bind(&CronetURLRequestContextAdapter::InitializeOnNetworkThread,
                 base::Unretained(this), base::Passed(&context_config_),
                 jcaller_ref));
}
```

而  `ProxyService` 的创建位置 (components/cronet/android/cronet_url_request_context_adapter.cc) 则在 `CronetURLRequestContextAdapter::InitializeOnNetworkThread()`：
```
void CronetURLRequestContextAdapter::InitializeOnNetworkThread(
    std::unique_ptr<URLRequestContextConfig> config,
    const base::android::ScopedJavaGlobalRef<jobject>&
        jcronet_url_request_context) {
  DCHECK(GetNetworkTaskRunner()->BelongsToCurrentThread());
  DCHECK(!is_context_initialized_);
  DCHECK(proxy_config_service_);
  // TODO(mmenke):  Add method to have the builder enable SPDY.
  net::URLRequestContextBuilder context_builder;

  std::unique_ptr<net::NetworkDelegate> network_delegate(
      new BasicNetworkDelegate());
#if defined(DATA_REDUCTION_PROXY_SUPPORT)
. . . . . .
#endif  // defined(DATA_REDUCTION_PROXY_SUPPORT)
  context_builder.set_network_delegate(std::move(network_delegate));
  context_builder.set_net_log(g_net_log.Get().net_log());

  // Android provides a local HTTP proxy server that handles proxying when a PAC
  // URL is present. Create a proxy service without a resolver and rely on this
  // local HTTP proxy. See: crbug.com/432539.
  context_builder.set_proxy_service(
      net::ProxyService::CreateWithoutProxyResolver(
          std::move(proxy_config_service_), g_net_log.Get().net_log()));
```

`ProxyConfigService` 和 `ProxyService` 的实际创建过程（位于net/proxy/proxy_service.cc ）如下：
```
// static
std::unique_ptr<ProxyService> ProxyService::CreateWithoutProxyResolver(
    std::unique_ptr<ProxyConfigService> proxy_config_service,
    NetLog* net_log) {
  return base::WrapUnique(new ProxyService(
      std::move(proxy_config_service),
      base::WrapUnique(new ProxyResolverFactoryForNullResolver), net_log));
}
. . . . . .
// static
std::unique_ptr<ProxyConfigService>
ProxyService::CreateSystemProxyConfigService(
    const scoped_refptr<base::SingleThreadTaskRunner>& io_task_runner,
    const scoped_refptr<base::SingleThreadTaskRunner>& file_task_runner) {
#if defined(OS_WIN)
. . . . . .
#elif defined(OS_ANDROID)
  return base::WrapUnique(new ProxyConfigServiceAndroid(
      io_task_runner, base::ThreadTaskRunnerHandle::Get()));
#else
  LOG(WARNING) << "Failed to choose a system proxy settings fetcher "
                  "for this platform.";
  return base::WrapUnique(new ProxyConfigServiceDirect());
#endif
}
```
可见在Android平台，默认的`ProxyConfigService` 为 `ProxyConfigServiceAndroid`， `ProxyService` 本身并不单单是接口，它在解析代理信息时，除了依赖静态信息外，还会依赖 `ProxyResolverFactory`  和 `ProxyResolver` 去获得代理信息。按照设计， `ProxyResolver` 将会填充用于特定URL的代理的列表。通常的 `ProxyResolver` 后端都是一个PAC脚本，但也不一定。一个 `ProxyResolver` 可以在同一时间为多个URL服务。

而在Android平台  `ProxyResolverFactory`  和 `ProxyResolver` 的实现分别为 `ProxyResolverFactoryForNullResolver` 和 `ProxyResolverNull`。可以看一下`ProxyResolverFactoryForNullResolver` 和 `ProxyResolverNull`的实现（位于net/proxy/proxy_service.cc ）：
```
// Proxy resolver that fails every time.
class ProxyResolverNull : public ProxyResolver {
 public:
  ProxyResolverNull() {}

  // ProxyResolver implementation.
  int GetProxyForURL(const GURL& url,
                     ProxyInfo* results,
                     const CompletionCallback& callback,
                     RequestHandle* request,
                     const BoundNetLog& net_log) override {
    return ERR_NOT_IMPLEMENTED;
  }

  void CancelRequest(RequestHandle request) override { NOTREACHED(); }

  LoadState GetLoadState(RequestHandle request) const override {
    NOTREACHED();
    return LOAD_STATE_IDLE;
  }

};
. . . . . .
class ProxyResolverFactoryForNullResolver : public ProxyResolverFactory {
 public:
  ProxyResolverFactoryForNullResolver() : ProxyResolverFactory(false) {}

  // ProxyResolverFactory overrides.
  int CreateProxyResolver(
      const scoped_refptr<ProxyResolverScriptData>& pac_script,
      std::unique_ptr<ProxyResolver>* resolver,
      const net::CompletionCallback& callback,
      std::unique_ptr<Request>* request) override {
    resolver->reset(new ProxyResolverNull());
    return OK;
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(ProxyResolverFactoryForNullResolver);
};
```
由此，可以认为在Android平台是没有 `ProxyResolver` 后端的，也就是代理解析，基本上只依赖系统的静态配置信息。

# ProxyService的初始化
在 `ProxyService` 创建的时候，会做一些初始化：
```
ProxyService::ProxyService(
    std::unique_ptr<ProxyConfigService> config_service,
    std::unique_ptr<ProxyResolverFactory> resolver_factory,
    NetLog* net_log)
    : resolver_factory_(std::move(resolver_factory)),
      next_config_id_(1),
      current_state_(STATE_NONE),
      net_log_(net_log),
      stall_proxy_auto_config_delay_(
          TimeDelta::FromMilliseconds(kDelayAfterNetworkChangesMs)),
      quick_check_enabled_(true),
      sanitize_url_policy_(SanitizeUrlPolicy::SAFE) {
  NetworkChangeNotifier::AddIPAddressObserver(this);
  NetworkChangeNotifier::AddDNSObserver(this);
  ResetConfigService(std::move(config_service));
}
. . . . . .
ProxyService::State ProxyService::ResetProxyConfig(bool reset_fetched_config) {
  DCHECK(CalledOnValidThread());
  State previous_state = current_state_;

  permanent_error_ = OK;
  proxy_retry_info_.clear();
  script_poller_.reset();
  init_proxy_resolver_.reset();
  SuspendAllPendingRequests();
  resolver_.reset();
  config_ = ProxyConfig();
  if (reset_fetched_config)
    fetched_config_ = ProxyConfig();
  current_state_ = STATE_NONE;

  return previous_state;
}

void ProxyService::ResetConfigService(
    std::unique_ptr<ProxyConfigService> new_proxy_config_service) {
  DCHECK(CalledOnValidThread());
  State previous_state = ResetProxyConfig(true);

  // Release the old configuration service.
  if (config_service_.get())
    config_service_->RemoveObserver(this);

  // Set the new configuration service.
  config_service_ = std::move(new_proxy_config_service);
  config_service_->AddObserver(this);

  if (previous_state != STATE_NONE)
    ApplyProxyConfigIfAvailable();
}
```
这里主要是将 `ProxyService` 对象注册为网络状态的监听者，以监听IP地址和 DNS 的改变，并注册为 `ProxyConfigService` 的监听者以监听。由于创建初始，previous_state 为 STATE_NONE，因而并不会做更多别的事情。

# ProxyConfigService的初始化
`ProxyConfigService` 是Android平台中 `ProxyService` 获取代理配置信息的关键，回头再来看 `ProxyConfigService` 的创建及初始化过程。如我们前面看到的，创建对象的位置在`CronetURLRequestContextAdapter::InitRequestContextOnMainThread()`。具体的过程（ 位于net/proxy/proxy_config_service_android.cc ）如下：
```
ProxyConfigServiceAndroid::ProxyConfigServiceAndroid(
    const scoped_refptr<base::SequencedTaskRunner>& network_task_runner,
    const scoped_refptr<base::SequencedTaskRunner>& jni_task_runner)
    : delegate_(new Delegate(
        network_task_runner, jni_task_runner, base::Bind(&GetJavaProperty))) {
  delegate_->SetupJNI();
  delegate_->FetchInitialConfig();
}
```
在这里主要是创建 `ProxyConfigServiceAndroid::Delegate`，并做初始化。初始化主要包括 `SetupJNI()` 和 `FetchInitialConfig()`，其中 `SetupJNI()` 是这样的：
```
  class JNIDelegateImpl : public ProxyConfigServiceAndroid::JNIDelegate {
   public:
    explicit JNIDelegateImpl(Delegate* delegate) : delegate_(delegate) {}
. . . . . .
class ProxyConfigServiceAndroid::Delegate
    : public base::RefCountedThreadSafe<Delegate> {
 public:
  Delegate(const scoped_refptr<base::SequencedTaskRunner>& network_task_runner,
           const scoped_refptr<base::SequencedTaskRunner>& jni_task_runner,
           const GetPropertyCallback& get_property_callback)
      : jni_delegate_(this),
        network_task_runner_(network_task_runner),
        jni_task_runner_(jni_task_runner),
        get_property_callback_(get_property_callback),
        exclude_pac_url_(false) {
  }

  void SetupJNI() {
    DCHECK(OnJNIThread());
    JNIEnv* env = AttachCurrentThread();
    if (java_proxy_change_listener_.is_null()) {
      VLOG(1) << "ProxyConfigServiceAndroid::Delegate SetupJNI, try to create java_proxy_change_listener_ object";
      java_proxy_change_listener_.Reset(
          Java_ProxyChangeListener_create(
              env, base::android::GetApplicationContext()));
      CHECK(!java_proxy_change_listener_.is_null());
    }
    Java_ProxyChangeListener_start(
        env,
        java_proxy_change_listener_.obj(),
        reinterpret_cast<intptr_t>(&jni_delegate_));
  }
```
可以看到，它主要是创建了一个类型为`org.chromium.net.ProxyChangeListener` 的Java对象，并调用了该对象的  `start(long nativePtr)` 方法（ 位于net/android/java/src/org/chromium/net/ProxyChangeListener.java ）。来看这个Java类的实现：
```
    private ProxyChangeListener(Context context) {
        mContext = context;
    }
. . . . . .
    @CalledByNative
    public static ProxyChangeListener create(Context context) {
        return new ProxyChangeListener(context);
    }
. . . . . .
    @CalledByNative
    public void start(long nativePtr) {
        assert mNativePtr == 0;
        mNativePtr = nativePtr;
        registerReceiver();
    }
. . . . . .
    private void registerReceiver() {
        if (mProxyReceiver != null) {
            return;
        }
        IntentFilter filter = new IntentFilter();
        filter.addAction(Proxy.PROXY_CHANGE_ACTION);
        mProxyReceiver = new ProxyReceiver();
        mContext.getApplicationContext().registerReceiver(mProxyReceiver, filter);
    }
```
可以看到，这里主要是注册了一个监听 Action 为 **Proxy.PROXY_CHANGE_ACTION** 的 BroadcastReceiver。再来看 `FetchInitialConfig()`：
```
// Returns whether the provided string was successfully converted to a port.
bool ConvertStringToPort(const std::string& port, int* output) {
  url::Component component(0, port.size());
  int result = url::ParsePort(port.c_str(), component);
  if (result == url::PORT_INVALID || result == url::PORT_UNSPECIFIED)
    return false;
  *output = result;
  return true;
}

ProxyServer ConstructProxyServer(ProxyServer::Scheme scheme,
                                 const std::string& proxy_host,
                                 const std::string& proxy_port) {
  DCHECK(!proxy_host.empty());
  int port_as_int = 0;
  if (proxy_port.empty())
    port_as_int = ProxyServer::GetDefaultPortForScheme(scheme);
  else if (!ConvertStringToPort(proxy_port, &port_as_int))
    return ProxyServer();
  DCHECK(port_as_int > 0);
  return ProxyServer(
      scheme, HostPortPair(proxy_host, static_cast<uint16_t>(port_as_int)));
}

ProxyServer LookupProxy(const std::string& prefix,
                        const GetPropertyCallback& get_property,
                        ProxyServer::Scheme scheme) {
  DCHECK(!prefix.empty());
  std::string proxy_host = get_property.Run(prefix + ".proxyHost");
  if (!proxy_host.empty()) {
    std::string proxy_port = get_property.Run(prefix + ".proxyPort");
    return ConstructProxyServer(scheme, proxy_host, proxy_port);
  }
  // Fall back to default proxy, if any.
  proxy_host = get_property.Run("proxyHost");
  if (!proxy_host.empty()) {
    std::string proxy_port = get_property.Run("proxyPort");
    return ConstructProxyServer(scheme, proxy_host, proxy_port);
  }
  return ProxyServer();
}

ProxyServer LookupSocksProxy(const GetPropertyCallback& get_property) {
  std::string proxy_host = get_property.Run("socksProxyHost");
  if (!proxy_host.empty()) {
    std::string proxy_port = get_property.Run("socksProxyPort");
    return ConstructProxyServer(ProxyServer::SCHEME_SOCKS5, proxy_host,
                                proxy_port);
  }
  return ProxyServer();
}

void AddBypassRules(const std::string& scheme,
                    const GetPropertyCallback& get_property,
                    ProxyBypassRules* bypass_rules) {
  // The format of a hostname pattern is a list of hostnames that are separated
  // by | and that use * as a wildcard. For example, setting the
  // http.nonProxyHosts property to *.android.com|*.kernel.org will cause
  // requests to http://developer.android.com to be made without a proxy.

  std::string non_proxy_hosts =
      get_property.Run(scheme + ".nonProxyHosts");
  if (non_proxy_hosts.empty())
    return;
  base::StringTokenizer tokenizer(non_proxy_hosts, "|");
  while (tokenizer.GetNext()) {
    std::string token = tokenizer.token();
    std::string pattern;
    base::TrimWhitespaceASCII(token, base::TRIM_ALL, &pattern);
    if (pattern.empty())
      continue;
    // '?' is not one of the specified pattern characters above.
    DCHECK_EQ(std::string::npos, pattern.find('?'));
    bypass_rules->AddRuleForHostname(scheme, pattern, -1);
  }
}

// Returns true if a valid proxy was found.
bool GetProxyRules(const GetPropertyCallback& get_property,
                   ProxyConfig::ProxyRules* rules) {
  // See libcore/luni/src/main/java/java/net/ProxySelectorImpl.java for the
  // mostly equivalent Android implementation.  There is one intentional
  // difference: by default Chromium uses the HTTP port (80) for HTTPS
  // connections via proxy.  This default is identical on other platforms.
  // On the opposite, Java spec suggests to use HTTPS port (443) by default (the
  // default value of https.proxyPort).
  rules->type = ProxyConfig::ProxyRules::TYPE_PROXY_PER_SCHEME;
  rules->proxies_for_http.SetSingleProxyServer(
      LookupProxy("http", get_property, ProxyServer::SCHEME_HTTP));
  rules->proxies_for_https.SetSingleProxyServer(
      LookupProxy("https", get_property, ProxyServer::SCHEME_HTTP));
  rules->proxies_for_ftp.SetSingleProxyServer(
      LookupProxy("ftp", get_property, ProxyServer::SCHEME_HTTP));
  rules->fallback_proxies.SetSingleProxyServer(LookupSocksProxy(get_property));
  rules->bypass_rules.Clear();
  AddBypassRules("ftp", get_property, &rules->bypass_rules);
  AddBypassRules("http", get_property, &rules->bypass_rules);
  AddBypassRules("https", get_property, &rules->bypass_rules);
  // We know a proxy was found if not all of the proxy lists are empty.
  return !(rules->proxies_for_http.IsEmpty() &&
      rules->proxies_for_https.IsEmpty() &&
      rules->proxies_for_ftp.IsEmpty() &&
      rules->fallback_proxies.IsEmpty());
};

void GetLatestProxyConfigInternal(const GetPropertyCallback& get_property,
                                  ProxyConfig* config) {
  if (!GetProxyRules(get_property, &config->proxy_rules()))
    *config = ProxyConfig::CreateDirect();
}

std::string GetJavaProperty(const std::string& property) {
  // Use Java System.getProperty to get configuration information.
  // TODO(pliard): Conversion to/from UTF8 ok here?
  JNIEnv* env = AttachCurrentThread();
  ScopedJavaLocalRef<jstring> str = ConvertUTF8ToJavaString(env, property);
  ScopedJavaLocalRef<jstring> result =
      Java_ProxyChangeListener_getProperty(env, str.obj());
  return result.is_null() ?
      std::string() : ConvertJavaStringToUTF8(env, result.obj());
}
. . . . . .
  void FetchInitialConfig() {
    DCHECK(OnJNIThread());
    ProxyConfig proxy_config;
    GetLatestProxyConfigInternal(get_property_callback_, &proxy_config);
    network_task_runner_->PostTask(
        FROM_HERE,
        base::Bind(&Delegate::SetNewConfigOnNetworkThread, this, proxy_config));
  }
. . . . . .
  // Called on the network thread.
  void SetNewConfigOnNetworkThread(const ProxyConfig& proxy_config) {
    DCHECK(OnNetworkThread());
    proxy_config_ = proxy_config;
    FOR_EACH_OBSERVER(Observer, observers_,
                      OnProxyConfigChanged(proxy_config,
                                           ProxyConfigService::CONFIG_VALID));
  }
```
可以看到，这里做了两件事，一是获取系统的代理配置信息，方法主要还是通过读取系统属性完成；二是通知监听者，这主要是`ProxyService`。

对于  Action 为 **Proxy.PROXY_CHANGE_ACTION** 的 BroadcastReceiver，是在注册完成之后几乎立即就会得到通知的。ProxyReceiver的实现如下：

```
    private class ProxyReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals(Proxy.PROXY_CHANGE_ACTION)) {
                proxySettingsChanged(extractNewProxy(intent));
            }
        }

        // Extract a ProxyConfig object from the supplied Intent's extra data
        // bundle. The android.net.ProxyProperties class is not exported from
        // the Android SDK, so we have to use reflection to get at it and invoke
        // methods on it. If we fail, return an empty proxy config (meaning
        // 'direct').
        // TODO(sgurun): once android.net.ProxyInfo is public, rewrite this.
        private ProxyConfig extractNewProxy(Intent intent) {
            try {
                final String getHostName = "getHost";
                final String getPortName = "getPort";
                final String getPacFileUrl = "getPacFileUrl";
                final String getExclusionList = "getExclusionList";
                String className;
                String proxyInfo;
                if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                    className = "android.net.ProxyProperties";
                    proxyInfo = "proxy";
                } else {
                    className = "android.net.ProxyInfo";
                    proxyInfo = "android.intent.extra.PROXY_INFO";
                }

                Object props = intent.getExtras().get(proxyInfo);
                if (props == null) {
                    return null;
                }

                Class<?> cls = Class.forName(className);
                Method getHostMethod = cls.getDeclaredMethod(getHostName);
                Method getPortMethod = cls.getDeclaredMethod(getPortName);
                Method getExclusionListMethod = cls.getDeclaredMethod(getExclusionList);

                String host = (String) getHostMethod.invoke(props);
                int port = (Integer) getPortMethod.invoke(props);

                String[] exclusionList;
                if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                    String s = (String) getExclusionListMethod.invoke(props);
                    exclusionList = s.split(",");
                } else {
                    exclusionList = (String[]) getExclusionListMethod.invoke(props);
                }
                // TODO(xunjieli): rewrite this once the API is public.
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT
                        && Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                    Method getPacFileUrlMethod = cls.getDeclaredMethod(getPacFileUrl);
                    String pacFileUrl = (String) getPacFileUrlMethod.invoke(props);
                    if (!TextUtils.isEmpty(pacFileUrl)) {
                        return new ProxyConfig(host, port, pacFileUrl, exclusionList);
                    }
                } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                    Method getPacFileUrlMethod = cls.getDeclaredMethod(getPacFileUrl);
                    Uri pacFileUrl = (Uri) getPacFileUrlMethod.invoke(props);
                    if (!Uri.EMPTY.equals(pacFileUrl)) {
                        return new ProxyConfig(host, port, pacFileUrl.toString(), exclusionList);
                    }
                }
                return new ProxyConfig(host, port, null, exclusionList);
            } catch (ClassNotFoundException ex) {
                Log.e(TAG, "Using no proxy configuration due to exception:" + ex);
                return null;
            } catch (NoSuchMethodException ex) {
                Log.e(TAG, "Using no proxy configuration due to exception:" + ex);
                return null;
            } catch (IllegalAccessException ex) {
                Log.e(TAG, "Using no proxy configuration due to exception:" + ex);
                return null;
            } catch (InvocationTargetException ex) {
                Log.e(TAG, "Using no proxy configuration due to exception:" + ex);
                return null;
            } catch (NullPointerException ex) {
                Log.e(TAG, "Using no proxy configuration due to exception:" + ex);
                return null;
            }
        }
    }

    private void proxySettingsChanged(ProxyConfig cfg) {
        if (!sEnabled) {
            return;
        }
        if (mDelegate != null) {
            mDelegate.proxySettingsChanged();
        }
        if (mNativePtr == 0) {
            return;
        }
        // Note that this code currently runs on a MESSAGE_LOOP_UI thread, but
        // the C++ code must run the callbacks on the network thread.
        if (cfg != null) {
            nativeProxySettingsChangedTo(mNativePtr, cfg.mHost, cfg.mPort, cfg.mPacUrl,
                    cfg.mExclusionList);
        } else {
            nativeProxySettingsChanged(mNativePtr);
        }
    }
```
这个Receiver在收到通知后，会将代理信息传递到C/C++层。最终调用`ProxyConfigServiceAndroid::Delegate::JNIDelegateImpl`：
```
    // ProxyConfigServiceAndroid::JNIDelegate overrides.
    void ProxySettingsChangedTo(
        JNIEnv* env,
        const JavaParamRef<jobject>& jself,
        const JavaParamRef<jstring>& jhost,
        jint jport,
        const JavaParamRef<jstring>& jpac_url,
        const JavaParamRef<jobjectArray>& jexclusion_list) override {
      std::string host = ConvertJavaStringToUTF8(env, jhost);
      std::string pac_url;
      if (jpac_url)
        ConvertJavaStringToUTF8(env, jpac_url, &pac_url);
      std::vector<std::string> exclusion_list;
      base::android::AppendJavaStringArrayToStringVector(
          env, jexclusion_list, &exclusion_list);
      delegate_->ProxySettingsChangedTo(host, jport, pac_url, exclusion_list);
    }

    void ProxySettingsChanged(JNIEnv* env,
                              const JavaParamRef<jobject>& self) override {
      delegate_->ProxySettingsChanged();
    }

   private:
    Delegate* const delegate_;
  };
```
继而调用 `ProxyConfigServiceAndroid::Delegate` 的相应方法：
```
void CreateStaticProxyConfig(const std::string& host,
                             int port,
                             const std::string& pac_url,
                             const std::vector<std::string>& exclusion_list,
                             ProxyConfig* config) {
  if (!pac_url.empty()) {
    config->set_pac_url(GURL(pac_url));
    config->set_pac_mandatory(false);
  } else if (port != 0) {
    std::string rules = base::StringPrintf("%s:%d", host.c_str(), port);
    config->proxy_rules().ParseFromString(rules);
    config->proxy_rules().bypass_rules.Clear();

    std::vector<std::string>::const_iterator it;
    for (it = exclusion_list.begin(); it != exclusion_list.end(); ++it) {
      std::string pattern;
      base::TrimWhitespaceASCII(*it, base::TRIM_ALL, &pattern);
      if (pattern.empty())
          continue;
      config->proxy_rules().bypass_rules.AddRuleForHostname("", pattern, -1);
    }
  } else {
    *config = ProxyConfig::CreateDirect();
  }
}
. . . . . .
  // Called on the JNI thread.
  void ProxySettingsChanged() {
    DCHECK(OnJNIThread());
    ProxyConfig proxy_config;
    GetLatestProxyConfigInternal(get_property_callback_, &proxy_config);
    network_task_runner_->PostTask(
        FROM_HERE,
        base::Bind(
            &Delegate::SetNewConfigOnNetworkThread, this, proxy_config));
  }

  // Called on the JNI thread.
  void ProxySettingsChangedTo(const std::string& host,
                              int port,
                              const std::string& pac_url,
                              const std::vector<std::string>& exclusion_list) {
    DCHECK(OnJNIThread());
    ProxyConfig proxy_config;
    if (exclude_pac_url_) {
      CreateStaticProxyConfig(host, port, "", exclusion_list, &proxy_config);
    } else {
      CreateStaticProxyConfig(host, port, pac_url, exclusion_list,
          &proxy_config);
    }
    network_task_runner_->PostTask(
        FROM_HERE,
        base::Bind(
            &Delegate::SetNewConfigOnNetworkThread, this, proxy_config));
  }
```
`ProxySettingsChanged()`如同 `FetchInitialConfig()` 一样，是从system property中获取代理配置信息，并通知监听者。而 `ProxySettingsChangedTo()` 则是以传入的代理配置信息构造配置，并通知监听者。

可见，`BroadcastReceiver` 通知时的这次配置信息更新会冲掉最初通过 `FetchInitialConfig()` 获取的那些。

总结一下，在Android中，chromium net获取代理配置信息的方法是：
1. 在 `ProxyConfigServiceAndroid` 创建过程中，从system property中读取代理设置信息，同时注册BroadcastReceiver以监听系统代理配置的改变。
2. 在收到系统广播消息的通知时，若广播中包含详细的代理配置信息，则以这些信息更新Chromium net的代理设置；否则，再次读取system property获取代理配置信息。

# 代理解析
`ProxyConfigService` 在代理配置发生改变时，会将新的代理配置通知给`ProxyService`：
```
void ProxyService::SetReady() {
  DCHECK(!init_proxy_resolver_.get());
  current_state_ = STATE_READY;

  // Make a copy in case |this| is deleted during the synchronous completion
  // of one of the requests. If |this| is deleted then all of the PacRequest
  // instances will be Cancel()-ed.
  PendingRequests pending_copy = pending_requests_;

  for (PendingRequests::iterator it = pending_copy.begin();
       it != pending_copy.end();
       ++it) {
    PacRequest* req = it->get();
    if (!req->is_started() && !req->was_cancelled()) {
      req->net_log()->EndEvent(NetLog::TYPE_PROXY_SERVICE_WAITING_FOR_INIT_PAC);

      // Note that we re-check for synchronous completion, in case we are
      // no longer using a ProxyResolver (can happen if we fell-back to manual).
      req->StartAndCompleteCheckingForSynchronous();
    }
  }
}
. . . . . .
void ProxyService::OnProxyConfigChanged(
    const ProxyConfig& config,
    ProxyConfigService::ConfigAvailability availability) {
  // Retrieve the current proxy configuration from the ProxyConfigService.
  // If a configuration is not available yet, we will get called back later
  // by our ProxyConfigService::Observer once it changes.
  ProxyConfig effective_config;
  switch (availability) {
    case ProxyConfigService::CONFIG_PENDING:
      // ProxyConfigService implementors should never pass CONFIG_PENDING.
      NOTREACHED() << "Proxy config change with CONFIG_PENDING availability!";
      return;
    case ProxyConfigService::CONFIG_VALID:
      effective_config = config;
      break;
    case ProxyConfigService::CONFIG_UNSET:
      effective_config = ProxyConfig::CreateDirect();
      break;
  }

  // Emit the proxy settings change to the NetLog stream.
  if (net_log_) {
    net_log_->AddGlobalEntry(NetLog::TYPE_PROXY_CONFIG_CHANGED,
                             base::Bind(&NetLogProxyConfigChangedCallback,
                                        &fetched_config_, &effective_config));
  }

  // Set the new configuration as the most recently fetched one.
  fetched_config_ = effective_config;
  fetched_config_.set_id(1);  // Needed for a later DCHECK of is_valid().

  InitializeUsingLastFetchedConfig();
}

void ProxyService::InitializeUsingLastFetchedConfig() {
  ResetProxyConfig(false);

  DCHECK(fetched_config_.is_valid());

  // Increment the ID to reflect that the config has changed.
  fetched_config_.set_id(next_config_id_++);

  if (!fetched_config_.HasAutomaticSettings()) {
    config_ = fetched_config_;
    SetReady();
    return;
  }

  // Start downloading + testing the PAC scripts for this new configuration.
  current_state_ = STATE_WAITING_FOR_INIT_PROXY_RESOLVER;

  // If we changed networks recently, we should delay running proxy auto-config.
  TimeDelta wait_delay =
      stall_proxy_autoconfig_until_ - TimeTicks::Now();

  init_proxy_resolver_.reset(new InitProxyResolver());
  init_proxy_resolver_->set_quick_check_enabled(quick_check_enabled_);
  int rv = init_proxy_resolver_->Start(
      &resolver_, resolver_factory_.get(), proxy_script_fetcher_.get(),
      dhcp_proxy_script_fetcher_.get(), net_log_, fetched_config_, wait_delay,
      base::Bind(&ProxyService::OnInitProxyResolverComplete,
                 base::Unretained(this)));

  if (rv != ERR_IO_PENDING)
    OnInitProxyResolverComplete(rv);
}
```
在这里主要是将新的代理配置信息保存在 `fetched_config_` 中，继而将配置保存在 config_ 中，并设置状态标记 current_state_ 为 ready。

`HttpStreamFactoryImpl::Job::DoResolveProxy()` 通过 `ProxyService` 的 `ResolveProxy()` 来为特定的URL找到合适的代理服务器：
```
int ProxyService::ResolveProxy(const GURL& raw_url,
                               const std::string& method,
                               ProxyInfo* result,
                               const CompletionCallback& callback,
                               PacRequest** pac_request,
                               ProxyDelegate* proxy_delegate,
                               const BoundNetLog& net_log) {
  DCHECK(!callback.is_null());
  return ResolveProxyHelper(raw_url, method, result, callback, pac_request,
                            proxy_delegate, net_log);
}

int ProxyService::ResolveProxyHelper(const GURL& raw_url,
                                     const std::string& method,
                                     ProxyInfo* result,
                                     const CompletionCallback& callback,
                                     PacRequest** pac_request,
                                     ProxyDelegate* proxy_delegate,
                                     const BoundNetLog& net_log) {
  DCHECK(CalledOnValidThread());

  net_log.BeginEvent(NetLog::TYPE_PROXY_SERVICE);

  // Notify our polling-based dependencies that a resolve is taking place.
  // This way they can schedule their polls in response to network activity.
  config_service_->OnLazyPoll();
  if (script_poller_.get())
     script_poller_->OnLazyPoll();

  if (current_state_ == STATE_NONE)
    ApplyProxyConfigIfAvailable();

  // Sanitize the URL before passing it on to the proxy resolver (i.e. PAC
  // script). The goal is to remove sensitive data (like embedded user names
  // and password), and local data (i.e. reference fragment) which does not need
  // to be disclosed to the resolver.
  GURL url = SanitizeUrl(raw_url, sanitize_url_policy_);

  // Check if the request can be completed right away. (This is the case when
  // using a direct connection for example).
  int rv = TryToCompleteSynchronously(url, proxy_delegate, result);
  if (rv != ERR_IO_PENDING) {
    rv = DidFinishResolvingProxy(
        url, method, proxy_delegate, result, rv, net_log,
        callback.is_null() ? TimeTicks() : TimeTicks::Now(), false);
    return rv;
  }

  if (callback.is_null())
    return ERR_IO_PENDING;

  scoped_refptr<PacRequest> req(new PacRequest(
      this, url, method, proxy_delegate, result, callback, net_log));

  if (current_state_ == STATE_READY) {
    // Start the resolve request.
    rv = req->Start();
    if (rv != ERR_IO_PENDING)
      return req->QueryDidComplete(rv);
  } else {
    req->net_log()->BeginEvent(NetLog::TYPE_PROXY_SERVICE_WAITING_FOR_INIT_PAC);
  }

  DCHECK_EQ(ERR_IO_PENDING, rv);
  DCHECK(!ContainsPendingRequest(req.get()));
  pending_requests_.insert(req);

  // Completion will be notified through |callback|, unless the caller cancels
  // the request using |pac_request|.
  if (pac_request)
    *pac_request = req.get();
  return rv;  // ERR_IO_PENDING
}
. . . . . .
int ProxyService::TryToCompleteSynchronously(const GURL& url,
                                             ProxyDelegate* proxy_delegate,
                                             ProxyInfo* result) {
  DCHECK_NE(STATE_NONE, current_state_);

  if (current_state_ != STATE_READY)
    return ERR_IO_PENDING;  // Still initializing.

  DCHECK_NE(config_.id(), ProxyConfig::kInvalidConfigID);

  // If it was impossible to fetch or parse the PAC script, we cannot complete
  // the request here and bail out.
  if (permanent_error_ != OK)
    return permanent_error_;

  if (config_.HasAutomaticSettings())
    return ERR_IO_PENDING;  // Must submit the request to the proxy resolver.

  // Use the manual proxy settings.
  config_.proxy_rules().Apply(url, result);
  result->config_source_ = config_.source();
  result->config_id_ = config_.id();

  return OK;
}
```
这个过程应用代理规则，选择适当的代理服务器给调用者。

Done。

# 参考资料
[Proxy server](https://en.wikipedia.org/wiki/Proxy_server)
[教你在Android手机上使用全局代理](http://www.7edown.com/edu/article/soft_5850_1.html)
