---
title: Cronet android 设计与实现分析——备选服务机制
date: 2016-10-29 14:07:49
categories: 网络协议
tags:
- 源码分析
- Android开发
- 网络协议
- chromium
---

前面我们分析到，在`URLRequestHttpJob::StartTransactionInternal()`中，会通过`URLRequestContext`的`HttpTransactionFactory`创建`HttpTransaction`，在`URLRequestContextBuilder::Build()`中创建`HttpTransactionFactory`的过程如下：

<!--more-->

```
  storage->set_http_network_session(
      base::WrapUnique(new HttpNetworkSession(network_session_params)));

  std::unique_ptr<HttpTransactionFactory> http_transaction_factory;
  if (http_cache_enabled_) {
    std::unique_ptr<HttpCache::BackendFactory> http_cache_backend;
    if (http_cache_params_.type != HttpCacheParams::IN_MEMORY) {
      BackendType backend_type =
          http_cache_params_.type == HttpCacheParams::DISK
              ? CACHE_BACKEND_DEFAULT
              : CACHE_BACKEND_SIMPLE;
      http_cache_backend.reset(new HttpCache::DefaultBackend(
          DISK_CACHE, backend_type, http_cache_params_.path,
          http_cache_params_.max_size, context->GetFileTaskRunner()));
    } else {
      http_cache_backend =
          HttpCache::DefaultBackend::InMemory(http_cache_params_.max_size);
    }

    LOG(INFO) << "Cache is enabled, to create HttpCache";
    http_transaction_factory.reset(new HttpCache(
        storage->http_network_session(), std::move(http_cache_backend), true));
  } else {
      LOG(INFO) << "Cache is disabled, to create HttpNetworkLayer";
    http_transaction_factory.reset(
        new HttpNetworkLayer(storage->http_network_session()));
  }
  storage->set_http_transaction_factory(std::move(http_transaction_factory));
```
Chromium net中有两个`HttpTransactionFactory`的实现，分别是`HttpCache`和`HttpNetworkLayer`，它们分别在cache被打开和被关闭时用到。这里还会创建`HttpNetworkSession`。而在cache打开时，在创建`HttpCache`的同时，还会为它创建http_cache_backend。

HttpCache的创建过程(chromium_android/src/net/http/http_cache.cc)如下：
```
//-----------------------------------------------------------------------------
HttpCache::HttpCache(HttpNetworkSession* session,
                     std::unique_ptr<BackendFactory> backend_factory,
                     bool set_up_quic_server_info)
    : HttpCache(base::WrapUnique(new HttpNetworkLayer(session)),
                std::move(backend_factory),
                set_up_quic_server_info) {}

HttpCache::HttpCache(std::unique_ptr<HttpTransactionFactory> network_layer,
                     std::unique_ptr<BackendFactory> backend_factory,
                     bool set_up_quic_server_info)
    : net_log_(nullptr),
      backend_factory_(std::move(backend_factory)),
      building_backend_(false),
      bypass_lock_for_test_(false),
      fail_conditionalization_for_test_(false),
      mode_(NORMAL),
      network_layer_(std::move(network_layer)),
      clock_(new base::DefaultClock()),
      weak_factory_(this) {
  HttpNetworkSession* session = network_layer_->GetSession();
  // Session may be NULL in unittests.
  // TODO(mmenke): Seems like tests could be changed to provide a session,
  // rather than having logic only used in unit tests here.
  if (session) {
    net_log_ = session->net_log();
    if (set_up_quic_server_info &&
        !session->quic_stream_factory()->has_quic_server_info_factory()) {
      // QuicStreamFactory takes ownership of QuicServerInfoFactoryAdaptor.
      session->quic_stream_factory()->set_quic_server_info_factory(
          new QuicServerInfoFactoryAdaptor(this));
    }
  }
}
```
这里还是会创建`HttpNetworkLayer`。HttpTransactionFactory相关的几个类之间的关系如下：

![HttpTransactionFactory](https://www.wolfcstech.com/images/1315506-9747918544dc1c4e.png)

**HttpNetworkTransaction**表示一个直接的网络事务，可以理解为一个网络连接。**HttpNetworkSession**用于管理网络连接。**HttpNetworkLayer**主要用于创建**HttpNetworkTransaction**。**HttpCache**和**HttpCache::Transaction**用于处理缓存。**HttpCache::Transaction**表示一个启用了缓存的网络事务，它会借助于**HttpCache**保存的**HttpNetworkLayer**引用创建**HttpNetworkTransaction**，借助于**HttpNetworkTransaction**访问网络，并根据需要将结果缓存起来。**HttpCache**则对缓存进行管理。**HttpNetworkLayer**和**HttpCache**都是**HttpTransactionFactory**，而**HttpNetworkTransaction**和**HttpCache::Transaction**都是**HttpTransaction**。

我们先不关心启用cache时，HTTP请求的处理流程，来看**HttpNetworkLayer**。则在`URLRequestHttpJob::StartTransactionInternal()`中将通过**HttpNetworkLayer**创建类型为**HttpNetworkTransaction**的**HttpTransaction**：
```
HttpNetworkLayer::HttpNetworkLayer(HttpNetworkSession* session)
    : session_(session),
      suspended_(false) {
  DCHECK(session_);
#if defined(OS_WIN)
  base::PowerMonitor* power_monitor = base::PowerMonitor::Get();
  if (power_monitor)
    power_monitor->AddObserver(this);
#endif
}

HttpNetworkLayer::~HttpNetworkLayer() {
#if defined(OS_WIN)
  base::PowerMonitor* power_monitor = base::PowerMonitor::Get();
  if (power_monitor)
    power_monitor->RemoveObserver(this);
#endif
}

int HttpNetworkLayer::CreateTransaction(
    RequestPriority priority,
    std::unique_ptr<HttpTransaction>* trans) {
  if (suspended_)
    return ERR_NETWORK_IO_SUSPENDED;

  trans->reset(new HttpNetworkTransaction(priority, GetSession()));
  return OK;
}

HttpCache* HttpNetworkLayer::GetCache() {
  return NULL;
}

HttpNetworkSession* HttpNetworkLayer::GetSession() {
  return session_;
}

void HttpNetworkLayer::OnSuspend() {
  suspended_ = true;
  session_->CloseIdleConnections();
}

void HttpNetworkLayer::OnResume() {
  suspended_ = false;
}
```
`URLRequestHttpJob::StartTransactionInternal()`创建了**HttpTransaction**之后，它会执行**HttpTransaction**的Start()来启动对HTTP事务的处理，**HttpNetworkTransaction**的Start()定义如下：
```
int HttpNetworkTransaction::Start(const HttpRequestInfo* request_info,
                                  const CompletionCallback& callback,
                                  const BoundNetLog& net_log) {
  net_log_ = net_log;
  request_ = request_info;

  // Now that we have an HttpRequestInfo object, update server_ssl_config_.
  session_->GetSSLConfig(*request_, &server_ssl_config_, &proxy_ssl_config_);

  if (request_->load_flags & LOAD_DISABLE_CERT_REVOCATION_CHECKING) {
    server_ssl_config_.rev_checking_enabled = false;
    proxy_ssl_config_.rev_checking_enabled = false;
  }

  if (request_->load_flags & LOAD_PREFETCH)
    response_.unused_since_prefetch = true;

  next_state_ = STATE_NOTIFY_BEFORE_CREATE_STREAM;
  int rv = DoLoop(OK);
  if (rv == ERR_IO_PENDING)
    callback_ = callback;
  return rv;
}
```
Chromium net将所有HTTP请求的处理抽象为几个步骤，并通过一个循环DoLoop()来一步一步地执行。DoLoop()的定义 (chromium_android/src/net/http/http_network_transaction.cc) 如下：
```
int HttpNetworkTransaction::DoLoop(int result) {
  DCHECK(next_state_ != STATE_NONE);

  int rv = result;
  do {
    State state = next_state_;
    next_state_ = STATE_NONE;
    switch (state) {
      case STATE_NOTIFY_BEFORE_CREATE_STREAM:
        DCHECK_EQ(OK, rv);
        rv = DoNotifyBeforeCreateStream();
        break;
      case STATE_CREATE_STREAM:
        DCHECK_EQ(OK, rv);
        rv = DoCreateStream();
        break;
      case STATE_CREATE_STREAM_COMPLETE:
        rv = DoCreateStreamComplete(rv);
        break;
      case STATE_INIT_STREAM:
        DCHECK_EQ(OK, rv);
        rv = DoInitStream();
        break;
      case STATE_INIT_STREAM_COMPLETE:
        rv = DoInitStreamComplete(rv);
        break;
      case STATE_GENERATE_PROXY_AUTH_TOKEN:
        DCHECK_EQ(OK, rv);
        rv = DoGenerateProxyAuthToken();
        break;
      case STATE_GENERATE_PROXY_AUTH_TOKEN_COMPLETE:
        rv = DoGenerateProxyAuthTokenComplete(rv);
        break;
      case STATE_GENERATE_SERVER_AUTH_TOKEN:
        DCHECK_EQ(OK, rv);
        rv = DoGenerateServerAuthToken();
        break;
      case STATE_GENERATE_SERVER_AUTH_TOKEN_COMPLETE:
        rv = DoGenerateServerAuthTokenComplete(rv);
        break;
      case STATE_GET_PROVIDED_TOKEN_BINDING_KEY:
        DCHECK_EQ(OK, rv);
        rv = DoGetProvidedTokenBindingKey();
        break;
      case STATE_GET_PROVIDED_TOKEN_BINDING_KEY_COMPLETE:
        rv = DoGetProvidedTokenBindingKeyComplete(rv);
        break;
      case STATE_GET_REFERRED_TOKEN_BINDING_KEY:
        DCHECK_EQ(OK, rv);
        rv = DoGetReferredTokenBindingKey();
        break;
      case STATE_GET_REFERRED_TOKEN_BINDING_KEY_COMPLETE:
        rv = DoGetReferredTokenBindingKeyComplete(rv);
        break;
      case STATE_INIT_REQUEST_BODY:
        DCHECK_EQ(OK, rv);
        rv = DoInitRequestBody();
        break;
      case STATE_INIT_REQUEST_BODY_COMPLETE:
        rv = DoInitRequestBodyComplete(rv);
        break;
      case STATE_BUILD_REQUEST:
        DCHECK_EQ(OK, rv);
        net_log_.BeginEvent(NetLog::TYPE_HTTP_TRANSACTION_SEND_REQUEST);
        rv = DoBuildRequest();
        break;
      case STATE_BUILD_REQUEST_COMPLETE:
        rv = DoBuildRequestComplete(rv);
        break;
      case STATE_SEND_REQUEST:
        DCHECK_EQ(OK, rv);
        rv = DoSendRequest();
        break;
      case STATE_SEND_REQUEST_COMPLETE:
        rv = DoSendRequestComplete(rv);
        net_log_.EndEventWithNetErrorCode(
            NetLog::TYPE_HTTP_TRANSACTION_SEND_REQUEST, rv);
        break;
      case STATE_READ_HEADERS:
        DCHECK_EQ(OK, rv);
        net_log_.BeginEvent(NetLog::TYPE_HTTP_TRANSACTION_READ_HEADERS);
        rv = DoReadHeaders();
        break;
      case STATE_READ_HEADERS_COMPLETE:
        rv = DoReadHeadersComplete(rv);
        net_log_.EndEventWithNetErrorCode(
            NetLog::TYPE_HTTP_TRANSACTION_READ_HEADERS, rv);
        break;
      case STATE_READ_BODY:
        DCHECK_EQ(OK, rv);
        net_log_.BeginEvent(NetLog::TYPE_HTTP_TRANSACTION_READ_BODY);
        rv = DoReadBody();
        break;
      case STATE_READ_BODY_COMPLETE:
        rv = DoReadBodyComplete(rv);
        net_log_.EndEventWithNetErrorCode(
            NetLog::TYPE_HTTP_TRANSACTION_READ_BODY, rv);
        break;
      case STATE_DRAIN_BODY_FOR_AUTH_RESTART:
        DCHECK_EQ(OK, rv);
        net_log_.BeginEvent(
            NetLog::TYPE_HTTP_TRANSACTION_DRAIN_BODY_FOR_AUTH_RESTART);
        rv = DoDrainBodyForAuthRestart();
        break;
      case STATE_DRAIN_BODY_FOR_AUTH_RESTART_COMPLETE:
        rv = DoDrainBodyForAuthRestartComplete(rv);
        net_log_.EndEventWithNetErrorCode(
            NetLog::TYPE_HTTP_TRANSACTION_DRAIN_BODY_FOR_AUTH_RESTART, rv);
        break;
      default:
        NOTREACHED() << "bad state";
        rv = ERR_FAILED;
        break;
    }
  } while (rv != ERR_IO_PENDING && next_state_ != STATE_NONE);

  return rv;
}
```
接下来我们逐个地分析这些步骤。

# Stream的创建
DoNotifyBeforeCreateStream()执行before_network_start_callback：
```
int HttpNetworkTransaction::DoNotifyBeforeCreateStream() {
  next_state_ = STATE_CREATE_STREAM;
  bool defer = false;
  if (!before_network_start_callback_.is_null())
    before_network_start_callback_.Run(&defer);
  if (!defer)
    return OK;
  return ERR_IO_PENDING;
}
```
在DoCreateStream()中创建Stream：
```
int HttpNetworkTransaction::DoCreateStream() {
  // TODO(mmenke): Remove ScopedTracker below once crbug.com/424359 is fixed.
  tracked_objects::ScopedTracker tracking_profile(
      FROM_HERE_WITH_EXPLICIT_FUNCTION(
          "424359 HttpNetworkTransaction::DoCreateStream"));

  response_.network_accessed = true;

  next_state_ = STATE_CREATE_STREAM_COMPLETE;
  if (ForWebSocketHandshake()) {
    stream_request_.reset(
        session_->http_stream_factory_for_websocket()
            ->RequestWebSocketHandshakeStream(
                  *request_,
                  priority_,
                  server_ssl_config_,
                  proxy_ssl_config_,
                  this,
                  websocket_handshake_stream_base_create_helper_,
                  net_log_));
  } else {
    stream_request_.reset(
        session_->http_stream_factory()->RequestStream(
            *request_,
            priority_,
            server_ssl_config_,
            proxy_ssl_config_,
            this,
            net_log_));
  }
  DCHECK(stream_request_.get());
  return ERR_IO_PENDING;
}

......

bool HttpNetworkTransaction::ForWebSocketHandshake() const {
  return websocket_handshake_stream_base_create_helper_ &&
         request_->url.SchemeIsWSOrWSS();
}
```
当请求是一个Websocket请求时，通过HttpNetworkSession的http_stream_factory_for_websocket创建Stream，而其他情况下，则会通过HttpNetworkSession的http_stream_factory创建Stream。

由**HttpNetworkSession**的创建过程 (chromium_android/src/net/http/http_network_session.cc) 可以看到，http_stream_factory_for_websocket和http_stream_factory都是**HttpStreamFactoryImpl**：
```
      http_stream_factory_(new HttpStreamFactoryImpl(this, false)),
      http_stream_factory_for_websocket_(new HttpStreamFactoryImpl(this, true)),
```

为Websocket之外的其它请求创建Stream的过程 (chromium_android/src/net/http/http_stream_factory_impl.cc) 为：
```
HttpStreamFactoryImpl::HttpStreamFactoryImpl(HttpNetworkSession* session,
                                             bool for_websockets)
    : session_(session),
      job_factory_(new DefaultJobFactory()),
      for_websockets_(for_websockets) {}

HttpStreamRequest* HttpStreamFactoryImpl::RequestStream(
    const HttpRequestInfo& request_info,
    RequestPriority priority,
    const SSLConfig& server_ssl_config,
    const SSLConfig& proxy_ssl_config,
    HttpStreamRequest::Delegate* delegate,
    const BoundNetLog& net_log) {
  DCHECK(!for_websockets_);
  return RequestStreamInternal(request_info, priority, server_ssl_config,
                               proxy_ssl_config, delegate, nullptr,
                               HttpStreamRequest::HTTP_STREAM, net_log);
}

......

HttpStreamRequest* HttpStreamFactoryImpl::RequestStreamInternal(
    const HttpRequestInfo& request_info,
    RequestPriority priority,
    const SSLConfig& server_ssl_config,
    const SSLConfig& proxy_ssl_config,
    HttpStreamRequest::Delegate* delegate,
    WebSocketHandshakeStreamBase::CreateHelper*
        websocket_handshake_stream_create_helper,
    HttpStreamRequest::StreamType stream_type,
    const BoundNetLog& net_log) {
  JobController* job_controller =
      new JobController(this, delegate, session_, job_factory_.get());
  job_controller_set_.insert(base::WrapUnique(job_controller));

  Request* request = job_controller->Start(
      request_info, delegate, websocket_handshake_stream_create_helper, net_log,
      stream_type, priority, server_ssl_config, proxy_ssl_config);

  return request;
}
```
在HttpStreamFactoryImpl::RequestStreamInternal()中，主要是创建了一个JobController，然后用job_controller->Start()创建了Request，也就是HttpStreamRequest。

由HttpStreamFactoryImpl的构造函数可以看到，**job_factory_**是DefaultJobFactory，这个类的实现也相当简单(chromium_android/src/net/http/http_stream_factory_impl.cc) ：
```
namespace {
// Default JobFactory for creating HttpStreamFactoryImpl::Jobs.
class DefaultJobFactory : public HttpStreamFactoryImpl::JobFactory {
 public:
  DefaultJobFactory() {}

  ~DefaultJobFactory() override {}

  HttpStreamFactoryImpl::Job* CreateJob(
      HttpStreamFactoryImpl::Job::Delegate* delegate,
      HttpStreamFactoryImpl::JobType job_type,
      HttpNetworkSession* session,
      const HttpRequestInfo& request_info,
      RequestPriority priority,
      const SSLConfig& server_ssl_config,
      const SSLConfig& proxy_ssl_config,
      HostPortPair destination,
      GURL origin_url,
      NetLog* net_log) override {
    return new HttpStreamFactoryImpl::Job(
        delegate, job_type, session, request_info, priority, server_ssl_config,
        proxy_ssl_config, destination, origin_url, net_log);
  }

  HttpStreamFactoryImpl::Job* CreateJob(
      HttpStreamFactoryImpl::Job::Delegate* delegate,
      HttpStreamFactoryImpl::JobType job_type,
      HttpNetworkSession* session,
      const HttpRequestInfo& request_info,
      RequestPriority priority,
      const SSLConfig& server_ssl_config,
      const SSLConfig& proxy_ssl_config,
      HostPortPair destination,
      GURL origin_url,
      AlternativeService alternative_service,
      NetLog* net_log) override {
    return new HttpStreamFactoryImpl::Job(
        delegate, job_type, session, request_info, priority, server_ssl_config,
        proxy_ssl_config, destination, origin_url, alternative_service,
        net_log);
  }
};
}  // anonymous namespace
```
JobController的Start()定义 (chromium_android/src/net/http/http_stream_factory_impl_job_controller.cc) 如下：
```
HttpStreamFactoryImpl::Request* HttpStreamFactoryImpl::JobController::Start(
    const HttpRequestInfo& request_info,
    HttpStreamRequest::Delegate* delegate,
    WebSocketHandshakeStreamBase::CreateHelper*
        websocket_handshake_stream_create_helper,
    const BoundNetLog& net_log,
    HttpStreamRequest::StreamType stream_type,
    RequestPriority priority,
    const SSLConfig& server_ssl_config,
    const SSLConfig& proxy_ssl_config) {
  DCHECK(factory_);
  DCHECK(!request_);

  request_ = new Request(request_info.url, this, delegate,
                         websocket_handshake_stream_create_helper, net_log,
                         stream_type);

  CreateJobs(request_info, priority, server_ssl_config, proxy_ssl_config,
             delegate, stream_type, net_log);

  return request_;
}
```
在这里主要是创建了一个Request，然后调用CreateJobs()创建了一些Jobs：
```
void HttpStreamFactoryImpl::JobController::CreateJobs(
    const HttpRequestInfo& request_info,
    RequestPriority priority,
    const SSLConfig& server_ssl_config,
    const SSLConfig& proxy_ssl_config,
    HttpStreamRequest::Delegate* delegate,
    HttpStreamRequest::StreamType stream_type,
    const BoundNetLog& net_log) {
  DCHECK(!main_job_);
  DCHECK(!alternative_job_);
  HostPortPair destination(HostPortPair::FromURL(request_info.url));
  GURL origin_url = ApplyHostMappingRules(request_info.url, &destination);

  main_job_.reset(job_factory_->CreateJob(
      this, MAIN, session_, request_info, priority, server_ssl_config,
      proxy_ssl_config, destination, origin_url, net_log.net_log()));
  AttachJob(main_job_.get());

  // Create an alternative job if alternative service is set up for this domain.
  const AlternativeService alternative_service =
      GetAlternativeServiceFor(request_info, delegate, stream_type);

  if (alternative_service.protocol != UNINITIALIZED_ALTERNATE_PROTOCOL) {
    // Never share connection with other jobs for FTP requests.
    DVLOG(1) << "Selected alternative service (host: "
             << alternative_service.host_port_pair().host()
             << " port: " << alternative_service.host_port_pair().port() << ")";

    DCHECK(!request_info.url.SchemeIs("ftp"));
    HostPortPair alternative_destination(alternative_service.host_port_pair());
    ignore_result(
        ApplyHostMappingRules(request_info.url, &alternative_destination));

    alternative_job_.reset(job_factory_->CreateJob(
        this, ALTERNATIVE, session_, request_info, priority, server_ssl_config,
        proxy_ssl_config, alternative_destination, origin_url,
        alternative_service, net_log.net_log()));
    AttachJob(alternative_job_.get());

    main_job_is_blocked_ = true;
    alternative_job_->Start(request_->stream_type());
  }
  // Even if |alternative_job| has already finished, it will not have notified
  // the request yet, since we defer that to the next iteration of the
  // MessageLoop, so starting |main_job_| is always safe.
  main_job_->Start(request_->stream_type());
}

 ......

GURL HttpStreamFactoryImpl::JobController::ApplyHostMappingRules(
    const GURL& url,
    HostPortPair* endpoint) {
  const HostMappingRules* mapping_rules = session_->params().host_mapping_rules;
  if (mapping_rules && mapping_rules->RewriteHost(endpoint)) {
    url::Replacements<char> replacements;
    const std::string port_str = base::UintToString(endpoint->port());
    replacements.SetPort(port_str.c_str(), url::Component(0, port_str.size()));
    replacements.SetHost(endpoint->host().c_str(),
                         url::Component(0, endpoint->host().size()));
    return url.ReplaceComponents(replacements);
  }
  return url;
}
```
这里的过程如下：
1. 应用主机映射规则，对url做修饰。
2. 通过job_factory创建main_job。
3. 查找备选服务。
4. 找到了备选服务，则创建alternative_job，并Start它。
5. Start main_job。

我们前面提到的**一些Jobs**主要是指main_job，和可能会创建的alternative_job。

接着我们来看Job的Start()方法(chromium_android/src/net/http/http_stream_factory_impl_job.cc) ：
```
void HttpStreamFactoryImpl::Job::Start(
    HttpStreamRequest::StreamType stream_type) {
  stream_type_ = stream_type;
  StartInternal();
}

.......

int HttpStreamFactoryImpl::Job::RunLoop(int result) {
  TRACE_EVENT0(TRACE_DISABLED_BY_DEFAULT("net"),
               "HttpStreamFactoryImpl::Job::RunLoop");
  LOG(INFO) << "HttpStreamFactoryImpl Job DoLoop start " << "job type " << job_type_;
  result = DoLoop(result);
  LOG(INFO) << "HttpStreamFactoryImpl Job DoLoop end " << "result " << result;

  if (result == ERR_IO_PENDING)
    return result;

  if (job_type_ == PRECONNECT) {
    base::ThreadTaskRunnerHandle::Get()->PostTask(
        FROM_HERE,
        base::Bind(&HttpStreamFactoryImpl::Job::OnPreconnectsComplete,
                   ptr_factory_.GetWeakPtr()));
    return ERR_IO_PENDING;
  }

  if (IsCertificateError(result)) {
    // Retrieve SSL information from the socket.
    GetSSLInfo();

    next_state_ = STATE_WAITING_USER_ACTION;
    base::ThreadTaskRunnerHandle::Get()->PostTask(
        FROM_HERE,
        base::Bind(&HttpStreamFactoryImpl::Job::OnCertificateErrorCallback,
                   ptr_factory_.GetWeakPtr(), result, ssl_info_));
    return ERR_IO_PENDING;
  }

  switch (result) {
    case ERR_PROXY_AUTH_REQUESTED: {
      UMA_HISTOGRAM_BOOLEAN("Net.ProxyAuthRequested.HasConnection",
                            connection_.get() != NULL);
      if (!connection_.get())
        return ERR_PROXY_AUTH_REQUESTED_WITH_NO_CONNECTION;
      CHECK(connection_->socket());
      CHECK(establishing_tunnel_);

      next_state_ = STATE_WAITING_USER_ACTION;
      ProxyClientSocket* proxy_socket =
          static_cast<ProxyClientSocket*>(connection_->socket());
      base::ThreadTaskRunnerHandle::Get()->PostTask(
          FROM_HERE,
          base::Bind(&Job::OnNeedsProxyAuthCallback, ptr_factory_.GetWeakPtr(),
                     *proxy_socket->GetConnectResponseInfo(),
                     base::RetainedRef(proxy_socket->GetAuthController())));
      return ERR_IO_PENDING;
    }

    case ERR_SSL_CLIENT_AUTH_CERT_NEEDED:
      base::ThreadTaskRunnerHandle::Get()->PostTask(
          FROM_HERE,
          base::Bind(
              &Job::OnNeedsClientAuthCallback, ptr_factory_.GetWeakPtr(),
              base::RetainedRef(
                  connection_->ssl_error_response_info().cert_request_info)));
      return ERR_IO_PENDING;

    case ERR_HTTPS_PROXY_TUNNEL_RESPONSE: {
      DCHECK(connection_.get());
      DCHECK(connection_->socket());
      DCHECK(establishing_tunnel_);

      ProxyClientSocket* proxy_socket =
          static_cast<ProxyClientSocket*>(connection_->socket());
      base::ThreadTaskRunnerHandle::Get()->PostTask(
          FROM_HERE, base::Bind(&Job::OnHttpsProxyTunnelResponseCallback,
                                ptr_factory_.GetWeakPtr(),
                                *proxy_socket->GetConnectResponseInfo(),
                                proxy_socket->CreateConnectResponseStream()));
      return ERR_IO_PENDING;
    }

    case OK:
      job_status_ = STATUS_SUCCEEDED;
      MaybeMarkAlternativeServiceBroken();
      next_state_ = STATE_DONE;
      if (new_spdy_session_.get()) {
        base::ThreadTaskRunnerHandle::Get()->PostTask(
            FROM_HERE, base::Bind(&Job::OnNewSpdySessionReadyCallback,
                                  ptr_factory_.GetWeakPtr()));
      } else if (delegate_->for_websockets()) {
        DCHECK(websocket_stream_);
        base::ThreadTaskRunnerHandle::Get()->PostTask(
            FROM_HERE, base::Bind(&Job::OnWebSocketHandshakeStreamReadyCallback,
                                  ptr_factory_.GetWeakPtr()));
      } else if (stream_type_ == HttpStreamRequest::BIDIRECTIONAL_STREAM) {
        if (!bidirectional_stream_impl_) {
          base::ThreadTaskRunnerHandle::Get()->PostTask(
              FROM_HERE, base::Bind(&Job::OnStreamFailedCallback,
                                    ptr_factory_.GetWeakPtr(), ERR_FAILED));
        } else {
          base::ThreadTaskRunnerHandle::Get()->PostTask(
              FROM_HERE,
              base::Bind(&Job::OnBidirectionalStreamImplReadyCallback,
                         ptr_factory_.GetWeakPtr()));
        }
      } else {
        DCHECK(stream_.get());
        job_stream_ready_start_time_ = base::TimeTicks::Now();
        base::ThreadTaskRunnerHandle::Get()->PostTask(
            FROM_HERE,
            base::Bind(&Job::OnStreamReadyCallback, ptr_factory_.GetWeakPtr()));
      }
      return ERR_IO_PENDING;

    default:
      if (job_status_ != STATUS_BROKEN) {
        DCHECK_EQ(STATUS_RUNNING, job_status_);
        job_status_ = STATUS_FAILED;
        MaybeMarkAlternativeServiceBroken();
      }
      base::ThreadTaskRunnerHandle::Get()->PostTask(
          FROM_HERE, base::Bind(&Job::OnStreamFailedCallback,
                                ptr_factory_.GetWeakPtr(), result));
      return ERR_IO_PENDING;
  }
}

......

int HttpStreamFactoryImpl::Job::StartInternal() {
  CHECK_EQ(STATE_NONE, next_state_);
  next_state_ = STATE_START;
  int rv = RunLoop(OK);
  DCHECK_EQ(ERR_IO_PENDING, rv);
  return rv;
}
```
执行调用流程大体如下：

![HttpStreamFactoryImpl_Job](https://www.wolfcstech.com/images/1315506-7a5e27d49cadf278.png)

在**HttpStreamFactoryImpl::Job::RunLoop()**中，主要是调用了**HttpStreamFactoryImpl::Job::DoLoop()**，并针对其执行结果，调用响应的回调函数，如：
```
void HttpStreamFactoryImpl::Job::OnStreamFailedCallback(int result) {
  DCHECK_NE(job_type_, PRECONNECT);

  MaybeCopyConnectionAttemptsFromSocketOrHandle();

  delegate_->OnStreamFailed(this, result, server_ssl_config_);
  // |this| may be deleted after this call.
}

void HttpStreamFactoryImpl::Job::OnCertificateErrorCallback(
    int result, const SSLInfo& ssl_info) {
  DCHECK_NE(job_type_, PRECONNECT);

  MaybeCopyConnectionAttemptsFromSocketOrHandle();

  delegate_->OnCertificateError(this, result, server_ssl_config_, ssl_info);
  // |this| may be deleted after this call.
}
```
从前面的**HttpStreamFactoryImpl::JobController::CreateJobs()**中可以看到，**delegate_**正是**HttpStreamFactoryImpl::JobController**。

而在**HttpStreamFactoryImpl::Job::DoLoop()**中，则是处理Stream建立的事情。与**HttpNetworkTransaction**的 **Start()** 执行的**DoLoop()**类似，**HttpStreamFactoryImpl::Job::DoLoop()**也是将Stream创建的过程抽象为一系列的步骤，通过一个循环，以一种类似于状态机模式的方式逐步骤执行：
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
      case STATE_CREATE_STREAM:
        DCHECK_EQ(OK, rv);
        rv = DoCreateStream();
        break;
      case STATE_CREATE_STREAM_COMPLETE:
        rv = DoCreateStreamComplete(rv);
        break;
      default:
        NOTREACHED() << "bad state";
        rv = ERR_FAILED;
        break;
    }
  } while (rv != ERR_IO_PENDING && next_state_ != STATE_NONE);
  return rv;
}
```

以执行一个QUIC请求为例，创建Stream的整个执行流程大体如下：
![CreateStream](https://www.wolfcstech.com/images/1315506-dd573ba5d84ca413.png)

# 备选服务机制

在**HttpStreamFactoryImpl::JobController**的**CreateJobs()**中我们看到，在通过job_factory创建main_job之后，会查找备选服务，在找到了备选服务时，还会为它创建job，并Start。那备选服务又是一套什么样的机制呢？

我们从两个方面来探究这套机制究竟是什么样的，及它又被用来做什么，一是备选服务的信息是从哪里及如何获取的，二是备选服务对**HttpStreamFactoryImpl::Job::Job**的操作的影响。

## 获取备选服务信息

我们先来看备选服务信息的获取。在**HttpStreamFactoryImpl::JobController**的**CreateJobs()**中，通过GetAlternativeServiceFor()来获取备选服务的信息：
```
AlternativeService
HttpStreamFactoryImpl::JobController::GetAlternativeServiceFor(
    const HttpRequestInfo& request_info,
    HttpStreamRequest::Delegate* delegate,
    HttpStreamRequest::StreamType stream_type) {
  AlternativeService alternative_service =
      GetAlternativeServiceForInternal(request_info, delegate, stream_type);
  AlternativeServiceType type;
  if (alternative_service.protocol == UNINITIALIZED_ALTERNATE_PROTOCOL) {
    type = NO_ALTERNATIVE_SERVICE;
  } else if (alternative_service.protocol == QUIC) {
    if (request_info.url.host() == alternative_service.host) {
      type = QUIC_SAME_DESTINATION;
    } else {
      type = QUIC_DIFFERENT_DESTINATION;
    }
  } else {
    if (request_info.url.host() == alternative_service.host) {
      type = NOT_QUIC_SAME_DESTINATION;
    } else {
      type = NOT_QUIC_DIFFERENT_DESTINATION;
    }
  }
  UMA_HISTOGRAM_ENUMERATION("Net.AlternativeServiceTypeForRequest", type,
                            MAX_ALTERNATIVE_SERVICE_TYPE);
  return alternative_service;
}
```
这里主要通过GetAlternativeServiceForInternal()获取备选服务的信息：
```
AlternativeService
HttpStreamFactoryImpl::JobController::GetAlternativeServiceForInternal(
    const HttpRequestInfo& request_info,
    HttpStreamRequest::Delegate* delegate,
    HttpStreamRequest::StreamType stream_type) {
  GURL original_url = request_info.url;

  if (!original_url.SchemeIs("https"))
    return AlternativeService();

  url::SchemeHostPort origin(original_url);
  HttpServerProperties& http_server_properties =
      *session_->http_server_properties();
  const AlternativeServiceVector alternative_service_vector =
      http_server_properties.GetAlternativeServices(origin);
  if (alternative_service_vector.empty())
    return AlternativeService();

  bool quic_advertised = false;
  bool quic_all_broken = true;

  // First Alt-Svc that is not marked as broken.
  AlternativeService first_alternative_service;

  for (const AlternativeService& alternative_service :
       alternative_service_vector) {
    DCHECK(IsAlternateProtocolValid(alternative_service.protocol));
    if (!quic_advertised && alternative_service.protocol == QUIC)
      quic_advertised = true;
    if (http_server_properties.IsAlternativeServiceBroken(
            alternative_service)) {
      HistogramAlternateProtocolUsage(ALTERNATE_PROTOCOL_USAGE_BROKEN);
      continue;
    }

    // Some shared unix systems may have user home directories (like
    // http://foo.com/~mike) which allow users to emit headers.  This is a bad
    // idea already, but with Alternate-Protocol, it provides the ability for a
    // single user on a multi-user system to hijack the alternate protocol.
    // These systems also enforce ports <1024 as restricted ports.  So don't
    // allow protocol upgrades to user-controllable ports.
    const int kUnrestrictedPort = 1024;
    if (!session_->params().enable_user_alternate_protocol_ports &&
        (alternative_service.port >= kUnrestrictedPort &&
         origin.port() < kUnrestrictedPort))
      continue;

    if (alternative_service.protocol >= NPN_SPDY_MINIMUM_VERSION &&
        alternative_service.protocol <= NPN_SPDY_MAXIMUM_VERSION) {
      if (origin.host() != alternative_service.host &&
          !session_->params()
               .enable_http2_alternative_service_with_different_host) {
        continue;
      }

      // Cache this entry if we don't have a non-broken Alt-Svc yet.
      if (first_alternative_service.protocol ==
          UNINITIALIZED_ALTERNATE_PROTOCOL)
        first_alternative_service = alternative_service;
      continue;
    }

    DCHECK_EQ(QUIC, alternative_service.protocol);
    if (origin.host() != alternative_service.host &&
        !session_->params()
             .enable_quic_alternative_service_with_different_host) {
      continue;
    }

    quic_all_broken = false;
    if (!session_->params().enable_quic)
      continue;

    if (!IsQuicWhitelistedForHost(origin.host()))
      continue;

    if (stream_type == HttpStreamRequest::BIDIRECTIONAL_STREAM &&
        session_->params().quic_disable_bidirectional_streams) {
      continue;
    }

    if (session_->quic_stream_factory()->IsQuicDisabled(
            alternative_service.port))
      continue;

    if (!original_url.SchemeIs("https"))
      continue;

    // Check whether there is an existing QUIC session to use for this origin.
    HostPortPair mapped_origin(origin.host(), origin.port());
    ignore_result(ApplyHostMappingRules(original_url, &mapped_origin));
    QuicServerId server_id(mapped_origin, request_info.privacy_mode);

    HostPortPair destination(alternative_service.host_port_pair());
    ignore_result(ApplyHostMappingRules(original_url, &destination));

    if (session_->quic_stream_factory()->CanUseExistingSession(server_id,
                                                               destination)) {
      return alternative_service;
    }

    // Cache this entry if we don't have a non-broken Alt-Svc yet.
    if (first_alternative_service.protocol == UNINITIALIZED_ALTERNATE_PROTOCOL)
      first_alternative_service = alternative_service;
  }

  // Ask delegate to mark QUIC as broken for the origin.
  if (quic_advertised && quic_all_broken && delegate != nullptr)
    delegate->OnQuicBroken();

  return first_alternative_service;
}
```
这个函数的执行流程如下：
1. 检查Url的scheme是否为https，若不是，直接返回空的AlternativeService。即备选服务只用于https。
2. 从session_获取HttpServerProperties http_server_properties。
3. 从http_server_properties获取所有的备选服务信息。
4. 遍历上一步找到的备选服务，找到一个可用的。

在Chromium net中，以https为scheme的请求有多种，一是常规的HTTP/1.1 + TLS的请求，二是SPDY/HTTP2请求，三是QUIC协议的请求。备选服务主要用于后两种协议的请求。这里会根据同源策略、端口、协议是否打开及主机是否在白名单等判断一个备选服务是否可用。

我们可以看到，备选服务的所有信息，都来源于http_server_properties。http_server_properties来源于HttpNetworkSession。HttpNetworkSession的http_server_properties在URLRequestContextBuilder::Build()中创建：
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

std::unique_ptr<URLRequestContext> URLRequestContextBuilder::Build() {
......
  if (http_server_properties_) {
    storage->set_http_server_properties(std::move(http_server_properties_));
  } else {
    storage->set_http_server_properties(
        std::unique_ptr<HttpServerProperties>(new HttpServerPropertiesImpl()));
  }
......
  storage->set_http_network_session(
      base::WrapUnique(new HttpNetworkSession(network_session_params)));
```

Cronet库的初始化过程中，会执行CronetURLRequestContextAdapter::InitializeOnNetworkThread()，在这个方法中，通过URLRequestContextBuilder构建了URLRequestContext之后，会向其中添加备选服务的信息：
```
void CronetURLRequestContextAdapter::InitializeOnNetworkThread(
    std::unique_ptr<URLRequestContextConfig> config,
    const base::android::ScopedJavaGlobalRef<jobject>&
        jcronet_url_request_context) {
......
  if (config->enable_quic) {
    for (auto hint = config->quic_hints.begin();
         hint != config->quic_hints.end(); ++hint) {
      const URLRequestContextConfig::QuicHint& quic_hint = **hint;
      if (quic_hint.host.empty()) {
        LOG(ERROR) << "Empty QUIC hint host: " << quic_hint.host;
        continue;
      }

      url::CanonHostInfo host_info;
      std::string canon_host(net::CanonicalizeHost(quic_hint.host, &host_info));
      if (!host_info.IsIPAddress() &&
          !net::IsCanonicalizedHostCompliant(canon_host)) {
        LOG(ERROR) << "Invalid QUIC hint host: " << quic_hint.host;
        continue;
      }

      if (quic_hint.port <= std::numeric_limits<uint16_t>::min() ||
          quic_hint.port > std::numeric_limits<uint16_t>::max()) {
        LOG(ERROR) << "Invalid QUIC hint port: "
                   << quic_hint.port;
        continue;
      }

      if (quic_hint.alternate_port <= std::numeric_limits<uint16_t>::min() ||
          quic_hint.alternate_port > std::numeric_limits<uint16_t>::max()) {
        LOG(ERROR) << "Invalid QUIC hint alternate port: "
                   << quic_hint.alternate_port;
        continue;
      }

      url::SchemeHostPort quic_server("https", canon_host, quic_hint.port);
      net::AlternativeService alternative_service(
          net::AlternateProtocol::QUIC, "",
          static_cast<uint16_t>(quic_hint.alternate_port));
      context_->http_server_properties()->SetAlternativeService(
          quic_server, alternative_service, base::Time::Max());
    }
  }
```
这里更是限定了只允许给QUIC协议添加备选服务。而这里添加的备选服务的信息都来自于URLRequestContextConfig。

继续来看给HttpServerProperties添加备选服务信息的过程 (chromium_android/src/net/http/http_server_properties_impl.cc)：
```
bool HttpServerPropertiesImpl::SetAlternativeService(
    const url::SchemeHostPort& origin,
    const AlternativeService& alternative_service,
    base::Time expiration) {
  return SetAlternativeServices(
      origin,
      AlternativeServiceInfoVector(
          /*size=*/1, AlternativeServiceInfo(alternative_service, expiration)));
}

bool HttpServerPropertiesImpl::SetAlternativeServices(
    const url::SchemeHostPort& origin,
    const AlternativeServiceInfoVector& alternative_service_info_vector) {
  AlternativeServiceMap::iterator it = alternative_service_map_.Peek(origin);

  if (alternative_service_info_vector.empty()) {
    RemoveCanonicalHost(origin);
    if (it == alternative_service_map_.end())
      return false;

    alternative_service_map_.Erase(it);
    return true;
  }

  bool changed = true;
  if (it != alternative_service_map_.end()) {
    DCHECK(!it->second.empty());
    if (it->second.size() == alternative_service_info_vector.size()) {
      const base::Time now = base::Time::Now();
      changed = false;
      auto new_it = alternative_service_info_vector.begin();
      for (const auto& old : it->second) {
        // Persist to disk immediately if new entry has different scheme, host,
        // or port.
        if (old.alternative_service != new_it->alternative_service) {
          changed = true;
          break;
        }
        // Also persist to disk if new expiration it more that twice as far or
        // less than half as far in the future.
        base::Time old_time = old.expiration;
        base::Time new_time = new_it->expiration;
        if (new_time - now > 2 * (old_time - now) ||
            2 * (new_time - now) < (old_time - now)) {
          changed = true;
          break;
        }
        ++new_it;
      }
    }
  }

  const bool previously_no_alternative_services =
      (GetAlternateProtocolIterator(origin) == alternative_service_map_.end());

  alternative_service_map_.Put(origin, alternative_service_info_vector);

  if (previously_no_alternative_services &&
      !GetAlternativeServices(origin).empty()) {
    // TODO(rch): Consider the case where multiple requests are started
    // before the first completes. In this case, only one of the jobs
    // would reach this code, whereas all of them should should have.
    HistogramAlternateProtocolUsage(ALTERNATE_PROTOCOL_USAGE_MAPPING_MISSING);
  }

  // If this host ends with a canonical suffix, then set it as the
  // canonical host.
  const char* kCanonicalScheme = "https";
  if (origin.scheme() == kCanonicalScheme) {
    const std::string* canonical_suffix = GetCanonicalSuffix(origin.host());
    if (canonical_suffix != nullptr) {
      url::SchemeHostPort canonical_server(kCanonicalScheme, *canonical_suffix,
                                           origin.port());
      canonical_host_to_origin_map_[canonical_server] = origin;
    }
  }
  return changed;
}
```
HttpServerPropertiesImpl用一个Map来管理备选服务的信息，key为原始服务的scheme+host+port，用url::SchemeHostPort来表示，而value则为AlternativeServiceInfoVector，即备选服务信息的列表。

### 用户添加备选服务信息

在CronetEngine.Builder中 (chromium_android/src/components/cronet/android/api/src/org/chromium/net/CronetEngine.java)，提供了接口，来添加QUIC服务器的一些信息：
```
public abstract class CronetEngine {
    /**
     * A builder for {@link CronetEngine}s, which allows runtime configuration of
     * {@code CronetEngine}. Configuration options are set on the builder and
     * then {@link #build} is called to create the {@code CronetEngine}.
     */
    public static class Builder {

......

        /**
         * A hint that a host supports QUIC.
         * @hide only used by internal implementation.
         */
        public static class QuicHint {
            // The host.
            public final String mHost;
            // Port of the server that supports QUIC.
            public final int mPort;
            // Alternate protocol port.
            public final int mAlternatePort;

            QuicHint(String host, int port, int alternatePort) {
                mHost = host;
                mPort = port;
                mAlternatePort = alternatePort;
            }
        }

......

        /**
         * Adds hint that {@code host} supports QUIC.
         * Note that {@link #enableHttpCache enableHttpCache}
         * ({@link #HTTP_CACHE_DISK}) is needed to take advantage of 0-RTT
         * connection establishment between sessions.
         *
         * @param host hostname of the server that supports QUIC.
         * @param port host of the server that supports QUIC.
         * @param alternatePort alternate port to use for QUIC.
         * @return the builder to facilitate chaining.
         */
        public Builder addQuicHint(String host, int port, int alternatePort) {
            if (host.contains("/")) {
                throw new IllegalArgumentException("Illegal QUIC Hint Host: " + host);
            }
            mQuicHints.add(new QuicHint(host, port, alternatePort));
            return this;
        }

        /**
         * @hide only used by internal implementation.
         */
        public List<QuicHint> quicHints() {
            return mQuicHints;
        }
```
在CronetUrlRequestContext创建中，创建native UrlRequestContextConfig时会将所有的QUIC hint信息传递给native层。
```
    @VisibleForTesting
    public static long createNativeUrlRequestContextConfig(
            final Context context, CronetEngine.Builder builder) {
        final long urlRequestContextConfig = nativeCreateRequestContextConfig(
                builder.getUserAgent(), builder.storagePath(), builder.quicEnabled(),
                builder.getDefaultQuicUserAgentId(context), builder.http2Enabled(),
                builder.sdchEnabled(), builder.dataReductionProxyKey(),
                builder.dataReductionProxyPrimaryProxy(), builder.dataReductionProxyFallbackProxy(),
                builder.dataReductionProxySecureProxyCheckUrl(), builder.cacheDisabled(),
                builder.httpCacheMode(), builder.httpCacheMaxSize(), builder.experimentalOptions(),
                builder.mockCertVerifier(), builder.networkQualityEstimatorEnabled(),
                builder.publicKeyPinningBypassForLocalTrustAnchorsEnabled(),
                builder.certVerifierData());
        for (Builder.QuicHint quicHint : builder.quicHints()) {
            nativeAddQuicHint(urlRequestContextConfig, quicHint.mHost, quicHint.mPort,
                    quicHint.mAlternatePort);
        }
        for (Builder.Pkp pkp : builder.publicKeyPins()) {
            nativeAddPkp(urlRequestContextConfig, pkp.mHost, pkp.mHashes, pkp.mIncludeSubdomains,
                    pkp.mExpirationDate.getTime());
        }
        return urlRequestContextConfig;
    }
```
nativeAddQuicHint()在chromium_android/src/components/cronet/android/cronet_url_request_context_adapter.cc中定义：
```
// Add a QUIC hint to a URLRequestContextConfig.
static void AddQuicHint(JNIEnv* env,
                        const JavaParamRef<jclass>& jcaller,
                        jlong jurl_request_context_config,
                        const JavaParamRef<jstring>& jhost,
                        jint jport,
                        jint jalternate_port) {
  URLRequestContextConfig* config =
      reinterpret_cast<URLRequestContextConfig*>(jurl_request_context_config);
  config->quic_hints.push_back(
      base::WrapUnique(new URLRequestContextConfig::QuicHint(
          base::android::ConvertJavaStringToUTF8(env, jhost), jport,
          jalternate_port)));
}
```
## 备选服务对**HttpStreamFactoryImpl::Job::Job**的操作的影响

为备选服务和为常规服务会以略有不同的方式创建Job：
```
HttpStreamFactoryImpl::Job::Job(Delegate* delegate,
                                JobType job_type,
                                HttpNetworkSession* session,
                                const HttpRequestInfo& request_info,
                                RequestPriority priority,
                                const SSLConfig& server_ssl_config,
                                const SSLConfig& proxy_ssl_config,
                                HostPortPair destination,
                                GURL origin_url,
                                NetLog* net_log)
    : Job(delegate,
          job_type,
          session,
          request_info,
          priority,
          server_ssl_config,
          proxy_ssl_config,
          destination,
          origin_url,
          AlternativeService(),
          net_log) {}

HttpStreamFactoryImpl::Job::Job(Delegate* delegate,
                                JobType job_type,
                                HttpNetworkSession* session,
                                const HttpRequestInfo& request_info,
                                RequestPriority priority,
                                const SSLConfig& server_ssl_config,
                                const SSLConfig& proxy_ssl_config,
                                HostPortPair destination,
                                GURL origin_url,
                                AlternativeService alternative_service,
                                NetLog* net_log)
    : request_info_(request_info),
      priority_(priority),
      server_ssl_config_(server_ssl_config),
      proxy_ssl_config_(proxy_ssl_config),
      net_log_(BoundNetLog::Make(net_log, NetLog::SOURCE_HTTP_STREAM_JOB)),
      io_callback_(base::Bind(&Job::OnIOComplete, base::Unretained(this))),
      connection_(new ClientSocketHandle),
      session_(session),
      next_state_(STATE_NONE),
      pac_request_(NULL),
      destination_(destination),
      origin_url_(origin_url),
      alternative_service_(alternative_service),
      delegate_(delegate),
      job_type_(job_type),
      using_ssl_(false),
      using_spdy_(false),
      using_quic_(false),
      quic_request_(session_->quic_stream_factory()),
      using_existing_quic_session_(false),
      spdy_certificate_error_(OK),
      establishing_tunnel_(false),
      was_npn_negotiated_(false),
      protocol_negotiated_(kProtoUnknown),
      num_streams_(0),
      spdy_session_direct_(false),
      job_status_(STATUS_RUNNING),
      other_job_status_(STATUS_RUNNING),
      stream_type_(HttpStreamRequest::BIDIRECTIONAL_STREAM),
      ptr_factory_(this) {
  DCHECK(session);

  if (IsSpdyAlternative()) {
    DCHECK(origin_url_.SchemeIs("https"));
  }
  if (IsQuicAlternative()) {
    DCHECK(session_->params().enable_quic);
    using_quic_ = true;
  }
}

......

bool HttpStreamFactoryImpl::Job::IsQuicAlternative() const {
  return alternative_service_.protocol == QUIC;
}
```
为QUIC备选服务创建的Job，在创建时期，就会将using_quic_置为true。这个标记的设置将对后续创建Stream的过程产生决定性的影响。

总结一下，备选服务机制像是一种过渡方案，用于在协议开发早期，还没有确定协议协商机制的情况下。在chromium net中，scheme为https的请求所用的协议有可能是HTTP/1.1+TLS、HTTP2和QUIC这三种的任一种，其中前两种都是基于TCP的，而QUIC是基于UDP的。当前前面的两种协议已经有了NPN和ALPN这样的协议协商的机制，而传给chromium net一个scheme为https的QUIC请求的URL，它也是不知道要用QUIC协议来做请求的。而备选服务机制，则允许chromium net的用户指定，对某些主机的访问采用特定的协议进行。此外，在**HttpStreamFactoryImpl::JobController**的CreateJobs()中，在为备选服务创建Job之外，还是会创建main_job，即是说，传给chromium net一个以https为scheme的Url，它一定会尝试用TCP的方式建立连接的，只是对于请求QUIC协议的服务，这个连接将会失败，而真正取回数据的将是alternative_job。
