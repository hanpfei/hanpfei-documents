---
title: Cronet android设计与实现分析--HTTP请求启动
date: 2016-10-29 14:06:49
tags: Cronet
---

在简单地分析了cronet的初始化过程之后，我们来看HTTP请求的提交和执行。

<!--more-->

# UrlRequest的创建

如我们在前面的[Cronet android 设计与实现分析——库的初始化](https://my.oschina.net/wolfcs/blog/752882)中看到的那样，Cronet的客户端，需要通过UrlRequest.Builder创建Request，然后提交给CronetEngine执行。这里我们从UrlRequest的创建开始我们的分析。

与CronetEngine类似，UrlRequest也需要通过Builder来创建。首先需要创建UrlRequest.Builder对象，url，callback，executor和CronetEngine是该创建过程所必须的：
```
        public Builder(
                String url, Callback callback, Executor executor, CronetEngine cronetEngine) {
            if (url == null) {
                throw new NullPointerException("URL is required.");
            }
            if (callback == null) {
                throw new NullPointerException("Callback is required.");
            }
            if (executor == null) {
                throw new NullPointerException("Executor is required.");
            }
            if (cronetEngine == null) {
                throw new NullPointerException("CronetEngine is required.");
            }
            mUrl = url;
            mCallback = callback;
            mExecutor = executor;
            mCronetEngine = cronetEngine;
        }
```
为我们前面创建的Builder设置了所有我们希望定制的属性之后，我们通过调用UrlRequest.Builder.build()创建UrlRequest对象：
```
        /**
         * Creates a {@link UrlRequest} using configuration within this
         * {@link Builder}. The returned {@code UrlRequest} can then be started
         * by calling {@link UrlRequest#start}.
         *
         * @return constructed {@link UrlRequest} using configuration within
         *         this {@link Builder}.
         */
        public UrlRequest build() {
            final UrlRequest request = mCronetEngine.createRequest(mUrl, mCallback, mExecutor,
                    mPriority, mRequestAnnotations, mDisableCache, mDisableConnectionMigration);
            if (mMethod != null) {
                request.setHttpMethod(mMethod);
            }
            for (Pair<String, String> header : mRequestHeaders) {
                request.addHeader(header.first, header.second);
            }
            if (mUploadDataProvider != null) {
                request.setUploadDataProvider(mUploadDataProvider, mUploadDataProviderExecutor);
            }
            return request;
        }
    }
```
这里用了一种类似于 原型模式 的方法，先由CronetEngine创建一个UrlRequest对象；然后将Builder中为UrlRequest定制的一些选项，如Http method，request的header等，设置给UrlRequest；最后将UrlRequest对象返回。

我们追一下org.chromium.net.impl.CronetUrlRequestContext的createRequest()来看看实际的UrlRequest创建过程是什么：
```
    @Override
    public UrlRequest createRequest(String url, UrlRequest.Callback callback, Executor executor,
            int priority, Collection<Object> requestAnnotations, boolean disableCache,
            boolean disableConnectionMigration) {
        synchronized (mLock) {
            checkHaveAdapter();
            boolean metricsCollectionEnabled = false;
            synchronized (mFinishedListenerLock) {
                metricsCollectionEnabled = !mFinishedListenerList.isEmpty();
            }
            return new CronetUrlRequest(this, url, priority, callback, executor, requestAnnotations,
                    metricsCollectionEnabled, disableCache, disableConnectionMigration);
        }
    }

......

    private void checkHaveAdapter() throws IllegalStateException {
        if (!haveRequestContextAdapter()) {
            throw new IllegalStateException("Engine is shut down.");
        }
    }

    private boolean haveRequestContextAdapter() {
        return mUrlRequestContextAdapter != 0;
    }
```
Chromium net提供了强大的debug的能力，可以帮我们抓到许多网络请求执行过程中的信息。但要抓取这些信息，总是要更耗费资源一些。这里会检查是否存在FinishedListener，只有当FinishedListener存在时，抓取的信息才是有意义的，因而，在FinishedListener不存在时，这里会关闭对信息的抓取。

随后，这个方法创建CronetUrlRequest对象，也就是实际的UrlRequest类型，而它的构造过程如下：
```
    CronetUrlRequest(CronetUrlRequestContext requestContext, String url, int priority,
            UrlRequest.Callback callback, Executor executor, Collection<Object> requestAnnotations,
            boolean metricsCollectionEnabled, boolean disableCache,
            boolean disableConnectionMigration) {
        if (url == null) {
            throw new NullPointerException("URL is required");
        }
        if (callback == null) {
            throw new NullPointerException("Listener is required");
        }
        if (executor == null) {
            throw new NullPointerException("Executor is required");
        }
        if (requestAnnotations == null) {
            throw new NullPointerException("requestAnnotations is required");
        }

        mRequestContext = requestContext;
        mInitialUrl = url;
        mUrlChain.add(url);
        mPriority = convertRequestPriority(priority);
        mCallback = callback;
        mExecutor = executor;
        mRequestAnnotations = requestAnnotations;
        mRequestMetricsAccumulator =
                metricsCollectionEnabled ? new UrlRequestMetricsAccumulator() : null;
        mDisableCache = disableCache;
        mDisableConnectionMigration = disableConnectionMigration;
    }
```
这里主要就是根据传入的参数，设置了一些字段的值。

# 事件通知

我们创建UrlRequest的时候，总是要传入一个Executor。在CronetUrlRequest中，只有postTaskToExecutor()一个方法访问了Executor：
```
    /**
     * Posts task to application Executor. Used for Listener callbacks
     * and other tasks that should not be executed on network thread.
     */
    private void postTaskToExecutor(Runnable task) {
        try {
            mExecutor.execute(task);
        } catch (RejectedExecutionException failException) {
            Log.e(CronetUrlRequestContext.LOG_TAG, "Exception posting task to executor",
                    failException);
            // If posting a task throws an exception, then there is no choice
            // but to destroy the request without invoking the callback.
            destroyRequestAdapter(false);
        }
    }
```
而CronetUrlRequest中有访问到postTaskToExecutor()的方法则有如下这些：
```
CronetUrlRequest
    failWithException(UrlRequestException)
    getStatus(StatusListener)
    onCanceled()
    onReadCompleted(ByteBuffer, int, int, int, long)
    onRedirectReceived(String, int, String, String[], boolean, String, String, long)
    onResponseStarted(int, String, String[], boolean, String, String)
    onStatus(StatusListener, int)
    onSucceeded(long)
```
如在onSucceeded()和onCanceled()中：
```
    /**
     * Called when request is completed successfully, no callbacks will be
     * called afterwards.
     *
     * @param receivedBytesCount number of bytes received.
     */
    @SuppressWarnings("unused")
    @CalledByNative
    private void onSucceeded(long receivedBytesCount) {
        mResponseInfo.setReceivedBytesCount(mReceivedBytesCountFromRedirects + receivedBytesCount);
        Runnable task = new Runnable() {
            @Override
            public void run() {
                synchronized (mUrlRequestAdapterLock) {
                    if (isDoneLocked()) {
                        return;
                    }
                    // Destroy adapter first, so request context could be shut
                    // down from the listener.
                    destroyRequestAdapter(false);
                }
                try {
                    mCallback.onSucceeded(CronetUrlRequest.this, mResponseInfo);
                } catch (Exception e) {
                    Log.e(CronetUrlRequestContext.LOG_TAG, "Exception in onComplete method", e);
                }
            }
        };
        postTaskToExecutor(task);
    }
    
    
    /**
     * Called when request is canceled, no callbacks will be called afterwards.
     */
    @SuppressWarnings("unused")
    @CalledByNative
    private void onCanceled() {
        Runnable task = new Runnable() {
            @Override
            public void run() {
                try {
                    mCallback.onCanceled(CronetUrlRequest.this, mResponseInfo);
                } catch (Exception e) {
                    Log.e(CronetUrlRequestContext.LOG_TAG, "Exception in onCanceled method", e);
                }
            }
        };
        postTaskToExecutor(task);
    }
```
可见，网络请求被丢给chromium之后，在发生某个事件时，native层会通过JNI机制反call java层的CronetUrlRequest的回调函数，在CronetUrlRequest的回调函数中则向Executor中抛出Task，通过客户端传入的UrlRequest.Callback，在Executor的线程中将事件抛给客户端。

可见传给UrlRequest的Executor主要是用于做事件通知的。

# Chromium net URLRequest的创建

语义上而言，客户端通过调用UrlRequest的start()方法启动HTTP请求的执行。CronetUrlRequest的start()方法的实现如下：
```
    @Override
    public void start() {
        synchronized (mUrlRequestAdapterLock) {
            checkNotStarted();

            try {
                mUrlRequestAdapter =
                        nativeCreateRequestAdapter(mRequestContext.getUrlRequestContextAdapter(),
                                mInitialUrl, mPriority, mDisableCache, mDisableConnectionMigration);
                mRequestContext.onRequestStarted();
                if (mInitialMethod != null) {
                    if (!nativeSetHttpMethod(mUrlRequestAdapter, mInitialMethod)) {
                        throw new IllegalArgumentException("Invalid http method " + mInitialMethod);
                    }
                }

                boolean hasContentType = false;
                for (Map.Entry<String, String> header : mRequestHeaders) {
                    if (header.getKey().equalsIgnoreCase("Content-Type")
                            && !header.getValue().isEmpty()) {
                        hasContentType = true;
                    }
                    if (!nativeAddRequestHeader(
                                mUrlRequestAdapter, header.getKey(), header.getValue())) {
                        throw new IllegalArgumentException(
                                "Invalid header " + header.getKey() + "=" + header.getValue());
                    }
                }
                if (mUploadDataStream != null) {
                    if (!hasContentType) {
                        throw new IllegalArgumentException(
                                "Requests with upload data must have a Content-Type.");
                    }
                    mStarted = true;
                    mUploadDataStream.postTaskToExecutor(new Runnable() {
                        @Override
                        public void run() {
                            mUploadDataStream.initializeWithRequest(CronetUrlRequest.this);
                            synchronized (mUrlRequestAdapterLock) {
                                if (isDoneLocked()) {
                                    return;
                                }
                                mUploadDataStream.attachNativeAdapterToRequest(mUrlRequestAdapter);
                                startInternalLocked();
                            }
                        }
                    });
                    return;
                }
            } catch (RuntimeException e) {
                // If there's an exception, cleanup and then throw the
                // exception to the caller.
                destroyRequestAdapter(false);
                throw e;
            }
            mStarted = true;
            startInternalLocked();
        }
    }
    
    /*
     * Starts fully configured request. Could execute on UploadDataProvider executor.
     * Caller is expected to ensure that request isn't canceled and mUrlRequestAdapter is valid.
     */
    @GuardedBy("mUrlRequestAdapterLock")
    private void startInternalLocked() {
        if (mRequestMetricsAccumulator != null) {
            mRequestMetricsAccumulator.onRequestStarted();
        }
        nativeStart(mUrlRequestAdapter);
    }

......

    private void checkNotStarted() {
        synchronized (mUrlRequestAdapterLock) {
            if (mStarted || isDoneLocked()) {
                throw new IllegalStateException("Request is already started.");
            }
        }
    }
```
在这个方法中会做如下这些事情：

1. 检查这个Request是否已经被启动了，若已经被启动了的话，会抛出异常。
2. 调用nativeCreateRequestAdapter()创建native层的UrlRequestAdapter。Cronet的大部分逻辑都在native层，因而需要通过UrlRequestAdapter来将Java层和native联系起来。UrlRequestAdapter将Java层的UrlRequest与native层的URLRequest对应起来。
3. 回调RequestContext的onRequestStarted()。
4. 调用nativeSetHttpMethod()来设置HTTP method。
5. 调用nativeAddRequestHeader()来将HTTP header一个个传到native层。
6. 设置mStarted标志，以表明Request已经被启动了。
7. 调用startInternalLocked()，来启动Request。这个方法会调用nativeStart()来启动Request。

不难看出Java层的CronetUrlRequest只是native的Url Request的一个简单的包装。UrlRequestAdapter连接Java层的CronetUrlRequest和native层中实际表示Url Request的结构，就像CronetURLRequestContextAdapter将Java层的CronetEngine和native层chromium net的URLRequestContext连接起来一样。

UrlRequestAdapter和native层中实际表示Url Request的结构，没有在CronetUrlRequest创建的时候创建，而是在用户实际要启动请求的时候才开始创建。

nativeCreateRequestAdapter()的实现在chromium_android/src/out/Default/gen/components/cronet/android/cronet_jni_headers/cronet/jni/CronetUrlRequest_jni.h中，由构建系统自动产生。它将职责完全委托给CreateRequestAdapter()方法：
```
static jlong CreateRequestAdapter(JNIEnv* env,
                                  const JavaParamRef<jobject>& jcaller,
                                  jlong jurl_request_context_adapter,
                                  const JavaParamRef<jstring>& jurl,
                                  jint jrequest_priority) {
  URLRequestContextAdapter* context_adapter =
      reinterpret_cast<URLRequestContextAdapter*>(jurl_request_context_adapter);
  DCHECK(context_adapter);

  GURL url(ConvertJavaStringToUTF8(env, jurl));

  VLOG(1) << "New chromium network request: " << url.possibly_invalid_spec();

  URLRequestAdapter* adapter = new URLRequestAdapter(
      context_adapter, new JniURLRequestAdapterDelegate(env, jcaller), url,
      ConvertRequestPriority(jrequest_priority));

  return reinterpret_cast<jlong>(adapter);
}
```
在这里主要是创建了一个URLRequestAdapter对象，并将对象的地址返回给Java层保存，像CronetURLRequestContextAdapter的创建那样。

语义上而言，nativeStart()被CronetUrlRequest用于启动请求的执行，其定义同样在chromium_android/src/out/Default/gen/components/cronet/android/cronet_jni_headers/cronet/jni/CronetUrlRequest_jni中。它将职责委托给CronetURLRequestAdapter的Start()方法：
```
void CronetURLRequestAdapter::Start(JNIEnv* env,
                                    const JavaParamRef<jobject>& jcaller) {
  DCHECK(!context_->IsOnNetworkThread());
  context_->PostTaskToNetworkThread(
      FROM_HERE, base::Bind(&CronetURLRequestAdapter::StartOnNetworkThread,
                            base::Unretained(this)));
}
```
这个函数向CronetURLRequestContextAdapter的network thread抛了一个task，也就是CronetURLRequestAdapter::StartOnNetworkThread：
```
void CronetURLRequestAdapter::StartOnNetworkThread() {
  DCHECK(context_->IsOnNetworkThread());
  VLOG(1) << "Starting chromium request: "
          << initial_url_.possibly_invalid_spec().c_str()
          << " priority: " << RequestPriorityToString(initial_priority_);
  url_request_ = context_->GetURLRequestContext()->CreateRequest(
      initial_url_, net::DEFAULT_PRIORITY, this);
  url_request_->SetLoadFlags(load_flags_);
  url_request_->set_method(initial_method_);
  url_request_->SetExtraRequestHeaders(initial_request_headers_);
  url_request_->SetPriority(initial_priority_);
  if (upload_)
    url_request_->set_upload(std::move(upload_));
  url_request_->Start();
}
```
到这里才会真正的通过URLRequestContext的CreateRequest()创建chromium net的URLRequest。

可见我们调用Java层的CornetUrlRequest的start()及其内部调用的nativeStart()，只是Url Request相关结构的创建过程的延续而已，Url Request相关结构主要是指用于连接Java层的CronetUrlRequest和chromium net的URLRequest的CronetURLRequestAdapter和chromium net的URLRequest。chromium net的URLRequest的创建及启动，都是在CronetURLRequestAdapter::Start()，也就是Java层的CornetUrlRequest的nativeStart()方法的最后，抛向另外的一个线程中的task中完成的。

继续来看URLRequestContext的CreateRequest()创建URLRequest的过程，这个过程倒是蛮直接的(chromium_android/src/net/url_request/url_request_context.cc)：
```
std::unique_ptr<URLRequest> URLRequestContext::CreateRequest(
    const GURL& url,
    RequestPriority priority,
    URLRequest::Delegate* delegate) const {
  return base::WrapUnique(
      new URLRequest(url, priority, delegate, this, network_delegate_));
}
```
这里创建chromium net的URLRequest时，传入的URLRequest::Delegate是CronetURLRequestAdapter::StartOnNetworkThread传进来的，也就是CronetURLRequestAdapter对象本身，而NetworkDelegate则是来自于URLRequestContext的network_delegate_。可见，CronetURLRequestAdapter不仅用于Java层向chromium net的URLRequest传递信息，也会被chromium net的URLRequest用于向Java层通知事件。

net::URLRequest::Delegate的定义如下：
```
  class NET_EXPORT Delegate {
   public:
    // Called upon receiving a redirect.  The delegate may call the request's
    // Cancel method to prevent the redirect from being followed.  Since there
    // may be multiple chained redirects, there may also be more than one
    // redirect call.
    //
    // When this function is called, the request will still contain the
    // original URL, the destination of the redirect is provided in
    // |redirect_info.new_url|.  If the delegate does not cancel the request
    // and |*defer_redirect| is false, then the redirect will be followed, and
    // the request's URL will be changed to the new URL.  Otherwise if the
    // delegate does not cancel the request and |*defer_redirect| is true, then
    // the redirect will be followed once FollowDeferredRedirect is called
    // on the URLRequest.
    //
    // The caller must set |*defer_redirect| to false, so that delegates do not
    // need to set it if they are happy with the default behavior of not
    // deferring redirect.
    virtual void OnReceivedRedirect(URLRequest* request,
                                    const RedirectInfo& redirect_info,
                                    bool* defer_redirect);

    // Called when we receive an authentication failure.  The delegate should
    // call request->SetAuth() with the user's credentials once it obtains them,
    // or request->CancelAuth() to cancel the login and display the error page.
    // When it does so, the request will be reissued, restarting the sequence
    // of On* callbacks.
    virtual void OnAuthRequired(URLRequest* request,
                                AuthChallengeInfo* auth_info);

    // Called when we receive an SSL CertificateRequest message for client
    // authentication.  The delegate should call
    // request->ContinueWithCertificate() with the client certificate the user
    // selected and its private key, or request->ContinueWithCertificate(NULL,
    // NULL)
    // to continue the SSL handshake without a client certificate.
    virtual void OnCertificateRequested(
        URLRequest* request,
        SSLCertRequestInfo* cert_request_info);

    // Called when using SSL and the server responds with a certificate with
    // an error, for example, whose common name does not match the common name
    // we were expecting for that host.  The delegate should either do the
    // safe thing and Cancel() the request or decide to proceed by calling
    // ContinueDespiteLastError().  cert_error is a ERR_* error code
    // indicating what's wrong with the certificate.
    // If |fatal| is true then the host in question demands a higher level
    // of security (due e.g. to HTTP Strict Transport Security, user
    // preference, or built-in policy). In this case, errors must not be
    // bypassable by the user.
    virtual void OnSSLCertificateError(URLRequest* request,
                                       const SSLInfo& ssl_info,
                                       bool fatal);

    // After calling Start(), the delegate will receive an OnResponseStarted
    // callback when the request has completed.  If an error occurred, the
    // request->status() will be set.  On success, all redirects have been
    // followed and the final response is beginning to arrive.  At this point,
    // meta data about the response is available, including for example HTTP
    // response headers if this is a request for a HTTP resource.
    virtual void OnResponseStarted(URLRequest* request) = 0;

    // Called when the a Read of the response body is completed after an
    // IO_PENDING status from a Read() call.
    // The data read is filled into the buffer which the caller passed
    // to Read() previously.
    //
    // If an error occurred, request->status() will contain the error,
    // and bytes read will be -1.
    virtual void OnReadCompleted(URLRequest* request, int bytes_read) = 0;

   protected:
    virtual ~Delegate() {}
  };
```
而CronetURLRequestAdapter对这些回调函数的实现则为：
```
void CronetURLRequestAdapter::OnReceivedRedirect(
    net::URLRequest* request,
    const net::RedirectInfo& redirect_info,
    bool* defer_redirect) {
  DCHECK(context_->IsOnNetworkThread());
  DCHECK(request->status().is_success());
  JNIEnv* env = base::android::AttachCurrentThread();
  cronet::Java_CronetUrlRequest_onRedirectReceived(
      env, owner_.obj(),
      ConvertUTF8ToJavaString(env, redirect_info.new_url.spec()).obj(),
      redirect_info.status_code,
      ConvertUTF8ToJavaString(env, request->response_headers()->GetStatusText())
          .obj(),
      GetResponseHeaders(env).obj(),
      request->response_info().was_cached ? JNI_TRUE : JNI_FALSE,
      ConvertUTF8ToJavaString(env,
                              request->response_info().npn_negotiated_protocol)
          .obj(),
      ConvertUTF8ToJavaString(env,
                              request->response_info().proxy_server.ToString())
          .obj(),
      request->GetTotalReceivedBytes());
  *defer_redirect = true;
}

void CronetURLRequestAdapter::OnCertificateRequested(
    net::URLRequest* request,
    net::SSLCertRequestInfo* cert_request_info) {
  DCHECK(context_->IsOnNetworkThread());
  // Cronet does not support client certificates.
  request->ContinueWithCertificate(nullptr, nullptr);
}

void CronetURLRequestAdapter::OnSSLCertificateError(
    net::URLRequest* request,
    const net::SSLInfo& ssl_info,
    bool fatal) {
  DCHECK(context_->IsOnNetworkThread());
  request->Cancel();
  int net_error = net::MapCertStatusToNetError(ssl_info.cert_status);
  JNIEnv* env = base::android::AttachCurrentThread();
  cronet::Java_CronetUrlRequest_onError(
      env, owner_.obj(), NetErrorToUrlRequestError(net_error), net_error,
      net::QUIC_NO_ERROR,
      ConvertUTF8ToJavaString(env, net::ErrorToString(net_error)).obj(),
      request->GetTotalReceivedBytes());
}

void CronetURLRequestAdapter::OnResponseStarted(net::URLRequest* request) {
  DCHECK(context_->IsOnNetworkThread());
  if (MaybeReportError(request))
    return;
  JNIEnv* env = base::android::AttachCurrentThread();
  cronet::Java_CronetUrlRequest_onResponseStarted(
      env, owner_.obj(), request->GetResponseCode(),
      ConvertUTF8ToJavaString(env, request->response_headers()->GetStatusText())
          .obj(),
      GetResponseHeaders(env).obj(),
      request->response_info().was_cached ? JNI_TRUE : JNI_FALSE,
      ConvertUTF8ToJavaString(env,
                              request->response_info().npn_negotiated_protocol)
          .obj(),
      ConvertUTF8ToJavaString(env,
                              request->response_info().proxy_server.ToString())
          .obj());
}

void CronetURLRequestAdapter::OnReadCompleted(net::URLRequest* request,
                                              int bytes_read) {
  DCHECK(context_->IsOnNetworkThread());
  if (MaybeReportError(request))
    return;
  if (bytes_read != 0) {
    JNIEnv* env = base::android::AttachCurrentThread();
    cronet::Java_CronetUrlRequest_onReadCompleted(
        env, owner_.obj(), read_buffer_->byte_buffer(), bytes_read,
        read_buffer_->initial_position(), read_buffer_->initial_limit(),
        request->GetTotalReceivedBytes());
    // Free the read buffer. This lets the Java ByteBuffer be freed, if the
    // embedder releases it, too.
    read_buffer_ = nullptr;
  } else {
    JNIEnv* env = base::android::AttachCurrentThread();
    cronet::Java_CronetUrlRequest_onSucceeded(
        env, owner_.obj(), url_request_->GetTotalReceivedBytes());
  }
}
```
这些函数都通过JNI机制反call到Java层的回调。

CronetURLRequestAdapter::StartOnNetworkThread中创建了chromium net的URLRequest之后会为它设置HTTP请求的各种参数。在CronetUrlRequest.start()中，通过nativeSetHttpMethod()、nativeAddRequestHeader()等方法向native层设置的Http method和Http headers等参数都是暂存在CronetURLRequestAdapter中的，而这里会将那些参数真正的设置给chromium net的URLRequest。

# HTTP请求执行

创建了Chromium net的URLRequest之后，就是请求的执行了。这通过URLRequest::Start()来完成：
```
void URLRequest::Start() {
  DCHECK(delegate_);

  // TODO(pkasting): Remove ScopedTracker below once crbug.com/456327 is fixed.
  tracked_objects::ScopedTracker tracking_profile(
      FROM_HERE_WITH_EXPLICIT_FUNCTION("456327 URLRequest::Start"));

  // Some values can be NULL, but the job factory must not be.
  DCHECK(context_->job_factory());

  // Anything that sets |blocked_by_| before start should have cleaned up after
  // itself.
  DCHECK(blocked_by_.empty());

  g_url_requests_started = true;
  response_info_.request_time = base::Time::Now();

  load_timing_info_ = LoadTimingInfo();
  load_timing_info_.request_start_time = response_info_.request_time;
  load_timing_info_.request_start = base::TimeTicks::Now();

  if (network_delegate_) {
    // TODO(mmenke): Remove ScopedTracker below once crbug.com/456327 is fixed.
    tracked_objects::ScopedTracker tracking_profile25(
        FROM_HERE_WITH_EXPLICIT_FUNCTION("456327 URLRequest::Start 2.5"));

    OnCallToDelegate();
    int error = network_delegate_->NotifyBeforeURLRequest(
        this, before_request_callback_, &delegate_redirect_url_);
    // If ERR_IO_PENDING is returned, the delegate will invoke
    // |before_request_callback_| later.
    if (error != ERR_IO_PENDING)
      BeforeRequestComplete(error);
    return;
  }

  // TODO(mmenke): Remove ScopedTracker below once crbug.com/456327 is fixed.
  tracked_objects::ScopedTracker tracking_profile2(
      FROM_HERE_WITH_EXPLICIT_FUNCTION("456327 URLRequest::Start 2"));

  StartJob(URLRequestJobManager::GetInstance()->CreateJob(
      this, network_delegate_));
}

///////////////////////////////////////////////////////////////////////////////

URLRequest::URLRequest(const GURL& url,
                       RequestPriority priority,
                       Delegate* delegate,
                       const URLRequestContext* context,
                       NetworkDelegate* network_delegate)
    : context_(context),
      network_delegate_(network_delegate ? network_delegate
                                         : context->network_delegate()),
      net_log_(
          BoundNetLog::Make(context->net_log(), NetLog::SOURCE_URL_REQUEST)),
      url_chain_(1, url),
      method_("GET"),
      referrer_policy_(CLEAR_REFERRER_ON_TRANSITION_FROM_SECURE_TO_INSECURE),
      first_party_url_policy_(NEVER_CHANGE_FIRST_PARTY_URL),
      load_flags_(LOAD_NORMAL),
      delegate_(delegate),
      is_pending_(false),
      is_redirecting_(false),
      redirect_limit_(kMaxRedirects),
      priority_(priority),
      identifier_(GenerateURLRequestIdentifier()),
      calling_delegate_(false),
      use_blocked_by_as_load_param_(false),
      before_request_callback_(base::Bind(&URLRequest::BeforeRequestComplete,
                                          base::Unretained(this))),
      has_notified_completion_(false),
      received_response_content_length_(0),
      creation_time_(base::TimeTicks::Now()) {
  // Sanity check out environment.
  DCHECK(base::MessageLoop::current())
      << "The current base::MessageLoop must exist";

  context->url_requests()->insert(this);
  net_log_.BeginEvent(NetLog::TYPE_REQUEST_ALIVE);
}

......

void URLRequest::StartJob(URLRequestJob* job) {
  // TODO(mmenke): Remove ScopedTracker below once crbug.com/456327 is fixed.
  tracked_objects::ScopedTracker tracking_profile(
      FROM_HERE_WITH_EXPLICIT_FUNCTION("456327 URLRequest::StartJob"));

  DCHECK(!is_pending_);
  DCHECK(!job_.get());

  net_log_.BeginEvent(
      NetLog::TYPE_URL_REQUEST_START_JOB,
      base::Bind(&NetLogURLRequestStartCallback,
                 &url(), &method_, load_flags_, priority_,
                 upload_data_stream_ ? upload_data_stream_->identifier() : -1));

  job_.reset(job);
  job_->SetExtraRequestHeaders(extra_request_headers_);
  job_->SetPriority(priority_);

  if (upload_data_stream_.get())
    job_->SetUpload(upload_data_stream_.get());

  is_pending_ = true;
  is_redirecting_ = false;

  response_info_.was_cached = false;

  if (GURL(referrer_) != URLRequestJob::ComputeReferrerForRedirect(
                             referrer_policy_, referrer_, url())) {
    if (!network_delegate_ ||
        !network_delegate_->CancelURLRequestWithPolicyViolatingReferrerHeader(
            *this, url(), GURL(referrer_))) {
      referrer_.clear();
    } else {
      // We need to clear the referrer anyway to avoid an infinite recursion
      // when starting the error job.
      referrer_.clear();
      std::string source("delegate");
      net_log_.AddEvent(NetLog::TYPE_CANCELLED,
                        NetLog::StringCallback("source", &source));
      RestartWithJob(new URLRequestErrorJob(
          this, network_delegate_, ERR_BLOCKED_BY_CLIENT));
      return;
    }
  }

  // Start() always completes asynchronously.
  //
  // Status is generally set by URLRequestJob itself, but Start() calls
  // directly into the URLRequestJob subclass, so URLRequestJob can't set it
  // here.
  // TODO(mmenke):  Make the URLRequest manage its own status.
  status_ = URLRequestStatus::FromError(ERR_IO_PENDING);
  job_->Start();
}
```
URLRequest::Start()首先会初始化LoadTimingInfo结构load_timing_info_，这个结构用于记录请求执行的一些时间信息；其次，如果network_delegate_存在的话，则会调用network_delegate_的回调方法NotifyBeforeURLRequest()等；然后，通过URLRequestJobManager创建创建一个URLRequestJob对象；最后，调用URLRequest::StartJob()启动URLRequestJob。

## URLRequestJob的创建

如我们在前面看到的，URLRequest::Start()通过URLRequestJobManager创建URLRequestJob对象。URLRequestJobManager被设计为单例的类，其构造函数和析构函数被声明为private，而其对象的创建、获取和销毁则是借助于模板base::Singleton来完成的，为了方便模板中访问private构造函数和析构函数，这里(chromium_android/src/net/url_request/url_request_job_manager.h)声明了friend struct：
```
 private:
  friend struct base::DefaultSingletonTraits<URLRequestJobManager>;

  URLRequestJobManager();
  ~URLRequestJobManager();
```
URLRequestJobManager的GetInstance()方法定义(chromium_android/src/net/url_request/url_request_job_manager.cc)如下：
```
// static
URLRequestJobManager* URLRequestJobManager::GetInstance() {
  return base::Singleton<URLRequestJobManager>::get();
}
```
Singleton模板类在chromium的base模块中(chromium_android/src/base/memory/singleton.h)定义：
```
// Default traits for Singleton<Type>. Calls operator new and operator delete on
// the object. Registers automatic deletion at process exit.
// Overload if you need arguments or another memory allocation function.
template<typename Type>
struct DefaultSingletonTraits {
  // Allocates the object.
  static Type* New() {
    // The parenthesis is very important here; it forces POD type
    // initialization.
    return new Type();
  }

  // Destroys the object.
  static void Delete(Type* x) {
    delete x;
  }

  // Set to true to automatically register deletion of the object on process
  // exit. See below for the required call that makes this happen.
  static const bool kRegisterAtExit = true;

#if DCHECK_IS_ON()
  // Set to false to disallow access on a non-joinable thread.  This is
  // different from kRegisterAtExit because StaticMemorySingletonTraits allows
  // access on non-joinable threads, and gracefully handles this.
  static const bool kAllowedToAccessOnNonjoinableThread = false;
#endif
};

......

template <typename Type,
          typename Traits = DefaultSingletonTraits<Type>,
          typename DifferentiatingType = Type>
class Singleton {
 private:
  // Classes using the Singleton<T> pattern should declare a GetInstance()
  // method and call Singleton::get() from within that.
  friend Type* Type::GetInstance();

  // Allow TraceLog tests to test tracing after OnExit.
  friend class internal::DeleteTraceLogForTesting;

  // This class is safe to be constructed and copy-constructed since it has no
  // member.

  // Return a pointer to the one true instance of the class.
  static Type* get() {
#if DCHECK_IS_ON()
    // Avoid making TLS lookup on release builds.
    if (!Traits::kAllowedToAccessOnNonjoinableThread)
      ThreadRestrictions::AssertSingletonAllowed();
#endif

    // The load has acquire memory ordering as the thread which reads the
    // instance_ pointer must acquire visibility over the singleton data.
    subtle::AtomicWord value = subtle::Acquire_Load(&instance_);
    if (value != 0 && value != internal::kBeingCreatedMarker) {
      return reinterpret_cast<Type*>(value);
    }

    // Object isn't created yet, maybe we will get to create it, let's try...
    if (subtle::Acquire_CompareAndSwap(&instance_, 0,
                                       internal::kBeingCreatedMarker) == 0) {
      // instance_ was NULL and is now kBeingCreatedMarker.  Only one thread
      // will ever get here.  Threads might be spinning on us, and they will
      // stop right after we do this store.
      Type* newval = Traits::New();

      // Releases the visibility over instance_ to the readers.
      subtle::Release_Store(&instance_,
                            reinterpret_cast<subtle::AtomicWord>(newval));

      if (newval != NULL && Traits::kRegisterAtExit)
        AtExitManager::RegisterCallback(OnExit, NULL);

      return newval;
    }

    // We hit a race. Wait for the other thread to complete it.
    value = internal::WaitForInstance(&instance_);

    return reinterpret_cast<Type*>(value);
  }

  // Adapter function for use with AtExit().  This should be called single
  // threaded, so don't use atomic operations.
  // Calling OnExit while singleton is in use by other threads is a mistake.
  static void OnExit(void* /*unused*/) {
    // AtExit should only ever be register after the singleton instance was
    // created.  We should only ever get here with a valid instance_ pointer.
    Traits::Delete(reinterpret_cast<Type*>(subtle::NoBarrier_Load(&instance_)));
    instance_ = 0;
  }
  static subtle::AtomicWord instance_;
};

template <typename Type, typename Traits, typename DifferentiatingType>
subtle::AtomicWord Singleton<Type, Traits, DifferentiatingType>::instance_ = 0;
```
URLRequestJobManager的CreateJob创建URLRequestJob (chromium_android/src/net/url_request/url_request_job_manager.cc)：
```
URLRequestJob* URLRequestJobManager::CreateJob(
    URLRequest* request, NetworkDelegate* network_delegate) const {
  DCHECK(IsAllowedThread());

  // If we are given an invalid URL, then don't even try to inspect the scheme.
  if (!request->url().is_valid())
    return new URLRequestErrorJob(request, network_delegate, ERR_INVALID_URL);

  // We do this here to avoid asking interceptors about unsupported schemes.
  const URLRequestJobFactory* job_factory =
      request->context()->job_factory();

  const std::string& scheme = request->url().scheme();  // already lowercase
  if (!job_factory->IsHandledProtocol(scheme)) {
    return new URLRequestErrorJob(
        request, network_delegate, ERR_UNKNOWN_URL_SCHEME);
  }

  // THREAD-SAFETY NOTICE:
  //   We do not need to acquire the lock here since we are only reading our
  //   data structures.  They should only be modified on the current thread.

  // See if the request should be intercepted.
  //
  URLRequestJob* job = job_factory->MaybeCreateJobWithProtocolHandler(
      scheme, request, network_delegate);
  if (job)
    return job;

  // See if the request should be handled by a built-in protocol factory.
  for (size_t i = 0; i < arraysize(kBuiltinFactories); ++i) {
    if (scheme == kBuiltinFactories[i].scheme) {
      URLRequestJob* new_job =
          (kBuiltinFactories[i].factory)(request, network_delegate, scheme);
      DCHECK(new_job);  // The built-in factories are not expected to fail!
      return new_job;
    }
  }

  // If we reached here, then it means that a registered protocol factory
  // wasn't interested in handling the URL.  That is fairly unexpected, and we
  // don't have a specific error to report here :-(
  LOG(WARNING) << "Failed to map: " << request->url().spec();
  return new URLRequestErrorJob(request, network_delegate, ERR_FAILED);
}
```
URLRequestJob是通过factory创建的。根据URL的scheme的不同，也就是协议的不同，而使用不同的factory创建URLRequestJob对象。在这里会首先判断URL的协议是能被处理。chromium net能处理的协议主要有两类，一类是只要在编译期enable了，就always处理的，如http、https、ws和wss；另一类协议，则不仅要在编译期enable对相关协议的支持，在运行期创建CronetEngine/URLRequestContext时，还可以动态的打开或关闭对相关协议的处理，如data、file、ftp等。

前一类协议的相关信息主要由URLRequestJobManager维护，而后一类协议的信息则由URLRequestContext的URLRequestJobFactory维护。URLRequestJobFactory的实际类型是在URLRequestContextBuilder::Build()中(chromium_android/src/net/url_request/url_request_context_builder.cc)确定的，实际类型为URLRequestJobFactoryImpl：
```
  URLRequestJobFactoryImpl* job_factory = new URLRequestJobFactoryImpl;
  // Adds caller-provided protocol handlers first so that these handlers are
  // used over data/file/ftp handlers below.
  for (auto& scheme_handler : protocol_handlers_) {
    job_factory->SetProtocolHandler(scheme_handler.first,
                                    std::move(scheme_handler.second));
  }
  protocol_handlers_.clear();

  if (data_enabled_)
    job_factory->SetProtocolHandler("data",
                                    base::WrapUnique(new DataProtocolHandler));

#if !defined(DISABLE_FILE_SUPPORT)
  if (file_enabled_) {
    job_factory->SetProtocolHandler(
        "file", base::WrapUnique(
                    new FileProtocolHandler(context->GetFileTaskRunner())));
  }
#endif  // !defined(DISABLE_FILE_SUPPORT)

#if !defined(DISABLE_FTP_SUPPORT)
  if (ftp_enabled_) {
    ftp_transaction_factory_.reset(
        new FtpNetworkLayer(context->host_resolver()));
    job_factory->SetProtocolHandler(
        "ftp", base::WrapUnique(
                   new FtpProtocolHandler(ftp_transaction_factory_.get())));
  }
#endif  // !defined(DISABLE_FTP_SUPPORT)
```
判断协议是否支持，也是通过LRequestContext的URLRequestJobFactory，也就是URLRequestJobFactoryImpl来完成的：
```
bool URLRequestJobFactoryImpl::SetProtocolHandler(
    const std::string& scheme,
    std::unique_ptr<ProtocolHandler> protocol_handler) {
  DCHECK(CalledOnValidThread());

  if (!protocol_handler) {
    ProtocolHandlerMap::iterator it = protocol_handler_map_.find(scheme);
    if (it == protocol_handler_map_.end())
      return false;

    protocol_handler_map_.erase(it);
    return true;
  }

  if (ContainsKey(protocol_handler_map_, scheme))
    return false;
  protocol_handler_map_[scheme] = std::move(protocol_handler);
  return true;
}

......

bool URLRequestJobFactoryImpl::IsHandledProtocol(
    const std::string& scheme) const {
  DCHECK(CalledOnValidThread());
  return ContainsKey(protocol_handler_map_, scheme) ||
      URLRequestJobManager::GetInstance()->SupportsScheme(scheme);
}
```
这里判断可以处理的两种类型的协议的列表中是否包含了请求的协议。

URLRequestJobManager维护的，只要编译期enable就always处理的协议的信息通过一个表kBuiltinFactories来维护(chromium_android/src/net/url_request/url_request_job_manager.cc)：
```
static const SchemeToFactory kBuiltinFactories[] = {
    {"http", URLRequestHttpJob::Factory},
    {"https", URLRequestHttpJob::Factory},

#if defined(ENABLE_WEBSOCKETS)
    {"ws", URLRequestHttpJob::Factory},
    {"wss", URLRequestHttpJob::Factory},
#endif  // defined(ENABLE_WEBSOCKETS)
};

......

// static
bool URLRequestJobManager::SupportsScheme(const std::string& scheme) {
  for (size_t i = 0; i < arraysize(kBuiltinFactories); ++i) {
    if (base::LowerCaseEqualsASCII(scheme, kBuiltinFactories[i].scheme))
      return true;
  }

  return false;
}
```
URLRequestJobManager的CreateJob()，在确定URL的协议是支持的之后，会首先尝试使用URLRequestContext的URLRequestJobFactory来创建URLRequestJob，如果失败，则会使用builtin的URLRequestJob的factory。

这里可以看到chromium对URL请求的处理的灵活性。URLRequestJobManager builtin的URLRequestJob的factory，定义了一个http和https协议处理的实现，但通过这里的这套机制，chromium net也可以插入自己定义的http协议处理的实现。

对于http协议和https协议，默认情况下，其URLRequestJob由URLRequestHttpJob::Factory()工厂方法创建。这里我们来看一下它的定义(chromium_android/src/net/url_request/url_request_http_job.cc)：
```
// static
URLRequestJob* URLRequestHttpJob::Factory(URLRequest* request,
                                          NetworkDelegate* network_delegate,
                                          const std::string& scheme) {
  DCHECK(scheme == "http" || scheme == "https" || scheme == "ws" ||
         scheme == "wss");

  if (!request->context()->http_transaction_factory()) {
    NOTREACHED() << "requires a valid context";
    return new URLRequestErrorJob(
        request, network_delegate, ERR_INVALID_ARGUMENT);
  }

  URLRequestRedirectJob* redirect =
      MaybeInternallyRedirect(request, network_delegate);
  if (redirect)
    return redirect;

  return new URLRequestHttpJob(request,
                               network_delegate,
                               request->context()->http_user_agent_settings());
}
```
`URLRequestHttpJob::Factory()`工厂方法会首先判断`URLRequestContext`的`http_transaction_factory`是否存在，不存在时创建`URLRequestErrorJob`并返回。

否则，尝试通过`MaybeInternallyRedirect()`创建`URLRequestRedirectJob`，若成功则返回给调用者。

`URLRequestRedirectJob`创建失败时，则会创建`URLRequestHttpJob`。`MaybeInternallyRedirect`中创建URLRequestRedirectJob的过程如下(chromium_android/src/net/url_request/url_request_http_job.cc)：
```
net::URLRequestRedirectJob* MaybeInternallyRedirect(
    net::URLRequest* request,
    net::NetworkDelegate* network_delegate) {
  const GURL& url = request->url();
  if (url.SchemeIsCryptographic())
    return nullptr;

  net::TransportSecurityState* hsts =
      request->context()->transport_security_state();
  if (!hsts || !hsts->ShouldUpgradeToSSL(url.host()))
    return nullptr;

  GURL::Replacements replacements;
  replacements.SetSchemeStr(url.SchemeIs(url::kHttpScheme) ? url::kHttpsScheme
                                                           : url::kWssScheme);
  return new net::URLRequestRedirectJob(
      request, network_delegate, url.ReplaceComponents(replacements),
      // Use status code 307 to preserve the method, so POST requests work.
      net::URLRequestRedirectJob::REDIRECT_307_TEMPORARY_REDIRECT, "HSTS");
}
```
`URLRequestRedirectJob`主要用于将非安全的访问方式转换为安全的访问方式。这里首先会判断URL的scheme是否已经是Security的https或wss，若是，则无需再做其它的而直接返回。

其次，会借助于`URLRequestContext`的`TransportSecurityState`判断要访问的主机是否强制将访问方式确定为security的，若不是，则用非安全的方式访问就可以，而直接返回。

最后，要访问的主机强制要求访问方式为security的，而URL的scheme不是https或wss的情况，则先修改URL的scheme为security的，如https或wss，然后创建URLRequestRedirectJob。

## URLRequestRedirectJob的执行

URLRequestRedirectJob的构造过程(chromium_android/src/net/url_request/url_request_redirect_job.cc)如下：
```
URLRequestRedirectJob::URLRequestRedirectJob(URLRequest* request,
                                             NetworkDelegate* network_delegate,
                                             const GURL& redirect_destination,
                                             ResponseCode response_code,
                                             const std::string& redirect_reason)
    : URLRequestJob(request, network_delegate),
      redirect_destination_(redirect_destination),
      response_code_(response_code),
      redirect_reason_(redirect_reason),
      weak_factory_(this) {
  DCHECK(!redirect_reason_.empty());
}
```
在URLRequest::Start()中我们看到，会通过调用URLRequestJob::Start()来启动执行。这里我们看一下URLRequestRedirectJob的Start()实现：
```
void URLRequestRedirectJob::Start() {
  request()->net_log().AddEvent(
      NetLog::TYPE_URL_REQUEST_REDIRECT_JOB,
      NetLog::StringCallback("reason", &redirect_reason_));
  base::ThreadTaskRunnerHandle::Get()->PostTask(
      FROM_HERE, base::Bind(&URLRequestRedirectJob::StartAsync,
                            weak_factory_.GetWeakPtr()));
}
```
这里主要是抛了一个task，也就是URLRequestRedirectJob::StartAsync()，给TaskRunner执行：
```
void URLRequestRedirectJob::StartAsync() {
  DCHECK(request_);
  DCHECK(request_->status().is_success());

  receive_headers_end_ = base::TimeTicks::Now();
  response_time_ = base::Time::Now();

  std::string header_string =
      base::StringPrintf("HTTP/1.1 %i Internal Redirect\n"
                             "Location: %s\n"
                             "Non-Authoritative-Reason: %s",
                         response_code_,
                         redirect_destination_.spec().c_str(),
                         redirect_reason_.c_str());

  std::string http_origin;
  const HttpRequestHeaders& request_headers = request_->extra_request_headers();
  if (request_headers.GetHeader("Origin", &http_origin)) {
    // If this redirect is used in a cross-origin request, add CORS headers to
    // make sure that the redirect gets through. Note that the destination URL
    // is still subject to the usual CORS policy, i.e. the resource will only
    // be available to web pages if the server serves the response with the
    // required CORS response headers.
    header_string += base::StringPrintf(
        "\n"
        "Access-Control-Allow-Origin: %s\n"
        "Access-Control-Allow-Credentials: true",
        http_origin.c_str());
  }

  fake_headers_ = new HttpResponseHeaders(
      HttpUtil::AssembleRawHeaders(header_string.c_str(),
                                   header_string.length()));
  DCHECK(fake_headers_->IsRedirect(NULL));

  request()->net_log().AddEvent(
      NetLog::TYPE_URL_REQUEST_FAKE_RESPONSE_HEADERS_CREATED,
      base::Bind(
          &HttpResponseHeaders::NetLogCallback,
          base::Unretained(fake_headers_.get())));

  // TODO(mmenke):  Consider calling the NetworkDelegate with the headers here.
  // There's some weirdness about how to handle the case in which the delegate
  // tries to modify the redirect location, in terms of how IsSafeRedirect
  // should behave, and whether the fragment should be copied.
  URLRequestJob::NotifyHeadersComplete();
}
```
这里主要是构造了一个`HttpResponseHeaders`，继而调用了`URLRequestJob::NotifyHeadersComplete()`。在`URLRequestJob::NotifyHeadersComplete()`中会对redirect的响应作出适当的处理：
```
void URLRequestJob::NotifyHeadersComplete() {
  if (has_handled_response_)
    return;

  // The URLRequest status should still be IO_PENDING, which it was set to
  // before the URLRequestJob was started.  On error or cancellation, this
  // method should not be called.
  DCHECK(request_->status().is_io_pending());
  SetStatus(URLRequestStatus());

  // Initialize to the current time, and let the subclass optionally override
  // the time stamps if it has that information.  The default request_time is
  // set by URLRequest before it calls our Start method.
  request_->response_info_.response_time = base::Time::Now();
  GetResponseInfo(&request_->response_info_);

  MaybeNotifyNetworkBytes();
  request_->OnHeadersComplete();

  GURL new_location;
  int http_status_code;

  if (IsRedirectResponse(&new_location, &http_status_code)) {
    // Redirect response bodies are not read. Notify the transaction
    // so it does not treat being stopped as an error.
    DoneReadingRedirectResponse();

    // When notifying the URLRequest::Delegate, it can destroy the request,
    // which will destroy |this|.  After calling to the URLRequest::Delegate,
    // pointer must be checked to see if |this| still exists, and if not, the
    // code must return immediately.
    base::WeakPtr<URLRequestJob> weak_this(weak_factory_.GetWeakPtr());

    RedirectInfo redirect_info =
        ComputeRedirectInfo(new_location, http_status_code);
    bool defer_redirect = false;
    request_->NotifyReceivedRedirect(redirect_info, &defer_redirect);

    // Ensure that the request wasn't detached, destroyed, or canceled in
    // NotifyReceivedRedirect.
    if (!weak_this || !request_->status().is_success())
      return;

    if (defer_redirect) {
      deferred_redirect_info_ = redirect_info;
    } else {
      FollowRedirect(redirect_info);
    }
    return;
  }

  if (NeedsAuth()) {
    scoped_refptr<AuthChallengeInfo> auth_info;
    GetAuthChallengeInfo(&auth_info);

    // Need to check for a NULL auth_info because the server may have failed
    // to send a challenge with the 401 response.
    if (auth_info.get()) {
      request_->NotifyAuthRequired(auth_info.get());
      // Wait for SetAuth or CancelAuth to be called.
      return;
    }
  }

  has_handled_response_ = true;
  if (request_->status().is_success())
    filter_ = SetupFilter();

  if (!filter_.get()) {
    std::string content_length;
    request_->GetResponseHeaderByName("content-length", &content_length);
    if (!content_length.empty())
      base::StringToInt64(content_length, &expected_content_size_);
  } else {
    request_->net_log().AddEvent(
        NetLog::TYPE_URL_REQUEST_FILTERS_SET,
        base::Bind(&FiltersSetCallback, base::Unretained(filter_.get())));
  }

  request_->NotifyResponseStarted();

  // |this| may be destroyed at this point.
}
```
这里先通过GetResponseInfo()函数读取URL请求的响应信息，包括headers等；然后检查响应是否是重定向的响应，若是，则从响应的headers中抓取重定向信息，然后将这一事件通知给客户端；若客户端不follow重定向，则保存重定向信息并退出；若客户端要follow重定向，则通过`FollowRedirect()`来follow重定向：
```
void URLRequestJob::FollowRedirect(const RedirectInfo& redirect_info) {
  int rv = request_->Redirect(redirect_info);
  if (rv != OK)
    NotifyDone(URLRequestStatus(URLRequestStatus::FAILED, rv));
}
```
重定向的任务被委托给了URLRequest来完成：
```
int URLRequest::Redirect(const RedirectInfo& redirect_info) {
  // Matches call in NotifyReceivedRedirect.
  OnCallToDelegateComplete();
  if (net_log_.IsCapturing()) {
    net_log_.AddEvent(
        NetLog::TYPE_URL_REQUEST_REDIRECTED,
        NetLog::StringCallback("location",
                               &redirect_info.new_url.possibly_invalid_spec()));
  }

  // TODO(davidben): Pass the full RedirectInfo to the NetworkDelegate.
  if (network_delegate_)
    network_delegate_->NotifyBeforeRedirect(this, redirect_info.new_url);

  if (redirect_limit_ <= 0) {
    DVLOG(1) << "disallowing redirect: exceeds limit";
    return ERR_TOO_MANY_REDIRECTS;
  }

  if (!redirect_info.new_url.is_valid())
    return ERR_INVALID_URL;

  if (!job_->IsSafeRedirect(redirect_info.new_url)) {
    DVLOG(1) << "disallowing redirect: unsafe protocol";
    return ERR_UNSAFE_REDIRECT;
  }

  if (!final_upload_progress_.position())
    final_upload_progress_ = job_->GetUploadProgress();
  PrepareToRestart();

  if (redirect_info.new_method != method_) {
    // TODO(davidben): This logic still needs to be replicated at the consumers.
    if (method_ == "POST") {
      // If being switched from POST, must remove Origin header.
      // TODO(jww): This is Origin header removal is probably layering violation
      // and
      // should be refactored into //content. See https://crbug.com/471397.
      extra_request_headers_.RemoveHeader(HttpRequestHeaders::kOrigin);
    }
    // The inclusion of a multipart Content-Type header can cause problems with
    // some
    // servers:
    // http://code.google.com/p/chromium/issues/detail?id=843
    extra_request_headers_.RemoveHeader(HttpRequestHeaders::kContentLength);
    extra_request_headers_.RemoveHeader(HttpRequestHeaders::kContentType);
    upload_data_stream_.reset();
    method_ = redirect_info.new_method;
  }

  // Cross-origin redirects should not result in an Origin header value that is
  // equal to the original request's Origin header. This is necessary to prevent
  // a reflection of POST requests to bypass CSRF protections. If the header was
  // not set to "null", a POST request from origin A to a malicious origin M
  // could be redirected by M back to A.
  //
  // This behavior is specified in step 1 of step 10 of the 301, 302, 303, 307,
  // 308 block of step 5 of Section 4.2 of Fetch[1] (which supercedes the
  // behavior outlined in RFC 6454[2].
  //
  // [1]: https://fetch.spec.whatwg.org/#concept-http-fetch
  // [2]: https://tools.ietf.org/html/rfc6454#section-7
  //
  // TODO(jww): This is a layering violation and should be refactored somewhere
  // up into //net's embedder. https://crbug.com/471397
  if (!url::Origin(redirect_info.new_url)
           .IsSameOriginWith(url::Origin(url())) &&
      extra_request_headers_.HasHeader(HttpRequestHeaders::kOrigin)) {
    extra_request_headers_.SetHeader(HttpRequestHeaders::kOrigin,
                                     url::Origin().Serialize());
  }

  referrer_ = redirect_info.new_referrer;
  referrer_policy_ = redirect_info.new_referrer_policy;
  first_party_for_cookies_ = redirect_info.new_first_party_for_cookies;
  token_binding_referrer_ = redirect_info.referred_token_binding_host;

  url_chain_.push_back(redirect_info.new_url);
  --redirect_limit_;

  Start();
  return OK;
}
```
在URLRequest的Redirect()中，是做了一些准备工作之后，再次调用URLRequest的Start()。

## URLRequestHttpJob的执行

在URLRequestHttpJob::Factory()中，不需要重定向时会直接创建URLRequestHttpJob用以处理URLRequest。这里来看URLRequestHttpJob的执行(chromium_android/src/net/url_request/url_request_http_job.cc)：
```
void URLRequestHttpJob::Start() {
  // TODO(mmenke): Remove ScopedTracker below once crbug.com/456327 is fixed.
  tracked_objects::ScopedTracker tracking_profile(
      FROM_HERE_WITH_EXPLICIT_FUNCTION("456327 URLRequestHttpJob::Start"));

  DCHECK(!transaction_.get());

  // URLRequest::SetReferrer ensures that we do not send username and password
  // fields in the referrer.
  GURL referrer(request_->referrer());

  request_info_.url = request_->url();
  request_info_.method = request_->method();
  request_info_.load_flags = request_->load_flags();
  // Enable privacy mode if cookie settings or flags tell us not send or
  // save cookies.
  bool enable_privacy_mode =
      (request_info_.load_flags & LOAD_DO_NOT_SEND_COOKIES) ||
      (request_info_.load_flags & LOAD_DO_NOT_SAVE_COOKIES) ||
      CanEnablePrivacyMode();
  // Privacy mode could still be disabled in SetCookieHeaderAndStart if we are
  // going to send previously saved cookies.
  request_info_.privacy_mode = enable_privacy_mode ?
      PRIVACY_MODE_ENABLED : PRIVACY_MODE_DISABLED;

  // Strip Referer from request_info_.extra_headers to prevent, e.g., plugins
  // from overriding headers that are controlled using other means. Otherwise a
  // plugin could set a referrer although sending the referrer is inhibited.
  request_info_.extra_headers.RemoveHeader(HttpRequestHeaders::kReferer);

  // Our consumer should have made sure that this is a safe referrer. See for
  // instance WebCore::FrameLoader::HideReferrer.
  if (referrer.is_valid()) {
    request_info_.extra_headers.SetHeader(HttpRequestHeaders::kReferer,
                                          referrer.spec());
  }

  request_info_.token_binding_referrer = request_->token_binding_referrer();

  request_info_.extra_headers.SetHeaderIfMissing(
      HttpRequestHeaders::kUserAgent,
      http_user_agent_settings_ ?
          http_user_agent_settings_->GetUserAgent() : std::string());

  AddExtraHeaders();
  AddCookieHeaderAndStart();
}
```
这里主要做了三件事：
1. 将URLRequest里的一些关于URL请求的信息拷贝出来放进request_info_结构里。
2. 添加一些用户没有显式地设置的HTTP header，如UserAgent等，及其它一些headers。
3. 添加Cookie header并启动执行。当然是在Cookie enable的情况下，否则就是直接启动执行。

由AddExtraHeaders()的定义可以看到，其它的headers主要是指AcceptEncoding和AcceptLanguage这两个。

添加Cookie header并启动的过程则大体为：
```
void URLRequestHttpJob::AddCookieHeaderAndStart() {
  // If the request was destroyed, then there is no more work to do.
  if (!request_)
    return;

  CookieStore* cookie_store = request_->context()->cookie_store();
  if (cookie_store && !(request_info_.load_flags & LOAD_DO_NOT_SEND_COOKIES)) {
    CookieOptions options;
    options.set_include_httponly();

    // Set SameSiteCookieMode according to the rules laid out in
    // https://tools.ietf.org/html/draft-west-first-party-cookies:
    //
    // * Include both "strict" and "lax" same-site cookies if the request's
    //   |url|, |initiator|, and |first_party_for_cookies| all have the same
    //   registrable domain.
    //
    // * Include only "lax" same-site cookies if the request's |URL| and
    //   |first_party_for_cookies| have the same registrable domain, _and_ the
    //   request's |method| is "safe" ("GET" or "HEAD").
    //
    //   Note that this will generally be the case only for cross-site requests
    //   which target a top-level browsing context.
    //
    // * Otherwise, do not include same-site cookies.
    url::Origin requested_origin(request_->url());
    url::Origin site_for_cookies(request_->first_party_for_cookies());

    if (registry_controlled_domains::SameDomainOrHost(
            requested_origin, site_for_cookies,
            registry_controlled_domains::INCLUDE_PRIVATE_REGISTRIES)) {
      if (registry_controlled_domains::SameDomainOrHost(
              requested_origin, request_->initiator(),
              registry_controlled_domains::INCLUDE_PRIVATE_REGISTRIES)) {
        options.set_same_site_cookie_mode(
            CookieOptions::SameSiteCookieMode::INCLUDE_STRICT_AND_LAX);
      } else if (IsMethodSafe(request_->method())) {
        options.set_same_site_cookie_mode(
            CookieOptions::SameSiteCookieMode::INCLUDE_LAX);
      }
    }

    cookie_store->GetCookieListWithOptionsAsync(
        request_->url(), options,
        base::Bind(&URLRequestHttpJob::SetCookieHeaderAndStart,
                   weak_factory_.GetWeakPtr()));
  } else {
    DoStartTransaction();
  }
}

void URLRequestHttpJob::SetCookieHeaderAndStart(const CookieList& cookie_list) {
  if (cookie_list.size() && CanGetCookies(cookie_list)) {
    request_info_.extra_headers.SetHeader(
        HttpRequestHeaders::kCookie, CookieStore::BuildCookieLine(cookie_list));
    // Disable privacy mode as we are sending cookies anyway.
    request_info_.privacy_mode = PRIVACY_MODE_DISABLED;
  }
  DoStartTransaction();
}
```
如果不需要设置Cookies，则会直接调用DoStartTransaction()启动HTTP请求执行；否则，通过cookie_store异步地获取CookieCookieList，并设置给URLRequestHttpJob，然后调用DoStartTransaction()。

DoStartTransaction()：
```
void URLRequestHttpJob::StartTransaction() {
  // TODO(mmenke): Remove ScopedTracker below once crbug.com/456327 is fixed.
  tracked_objects::ScopedTracker tracking_profile(
      FROM_HERE_WITH_EXPLICIT_FUNCTION(
          "456327 URLRequestHttpJob::StartTransaction"));

  if (network_delegate()) {
    OnCallToDelegate();
    int rv = network_delegate()->NotifyBeforeStartTransaction(
        request_, notify_before_headers_sent_callback_,
        &request_info_.extra_headers);
    // If an extension blocks the request, we rely on the callback to
    // MaybeStartTransactionInternal().
    if (rv == ERR_IO_PENDING)
      return;
    MaybeStartTransactionInternal(rv);
    return;
  }
  StartTransactionInternal();
}

......

void URLRequestHttpJob::MaybeStartTransactionInternal(int result) {
  // TODO(mmenke): Remove ScopedTracker below once crbug.com/456327 is fixed.
  tracked_objects::ScopedTracker tracking_profile(
      FROM_HERE_WITH_EXPLICIT_FUNCTION(
          "456327 URLRequestHttpJob::MaybeStartTransactionInternal"));

  OnCallToDelegateComplete();
  if (result == OK) {
    StartTransactionInternal();
  } else {
    std::string source("delegate");
    request_->net_log().AddEvent(NetLog::TYPE_CANCELLED,
                                 NetLog::StringCallback("source", &source));
    NotifyStartError(URLRequestStatus(URLRequestStatus::FAILED, result));
  }
}

......

void URLRequestHttpJob::DoStartTransaction() {
  // We may have been canceled while retrieving cookies.
  if (GetStatus().is_success()) {
    StartTransaction();
  } else {
    NotifyCanceled();
  }
}
```
DoStartTransaction()首先检查一下请求是否被取消，如果已经取消了，会调用NotifyCanceled()来通知取消事件。

否则会调用StartTransaction()启动Transaction。这里会根据是否设置了回调，而有着不同的执行路径。如果没有设置回调，会直接调用StartTransactionInternal()启动Transaction。

而如果设置了回调，则先调用回调，并记录一些trace信息，然后调用StartTransactionInternal()。

来看StartTransactionInternal()：
```
void URLRequestHttpJob::StartTransactionInternal() {
  // This should only be called while the request's status is IO_PENDING.
  DCHECK_EQ(URLRequestStatus::IO_PENDING, request_->status().status());

  // NOTE: This method assumes that request_info_ is already setup properly.

  // If we already have a transaction, then we should restart the transaction
  // with auth provided by auth_credentials_.

  int rv;

  // Notify NetworkQualityEstimator.
  NetworkQualityEstimator* network_quality_estimator =
      request()->context()->network_quality_estimator();
  if (network_quality_estimator)
    network_quality_estimator->NotifyStartTransaction(*request_);

  if (network_delegate()) {
    network_delegate()->NotifyStartTransaction(request_,
                                               request_info_.extra_headers);
  }

  if (transaction_.get()) {
    rv = transaction_->RestartWithAuth(auth_credentials_, start_callback_);
    auth_credentials_ = AuthCredentials();
  } else {
    DCHECK(request_->context()->http_transaction_factory());

    rv = request_->context()->http_transaction_factory()->CreateTransaction(
        priority_, &transaction_);

    if (rv == OK && request_info_.url.SchemeIsWSOrWSS()) {
      base::SupportsUserData::Data* data = request_->GetUserData(
          WebSocketHandshakeStreamBase::CreateHelper::DataKey());
      if (data) {
        transaction_->SetWebSocketHandshakeStreamCreateHelper(
            static_cast<WebSocketHandshakeStreamBase::CreateHelper*>(data));
      } else {
        rv = ERR_DISALLOWED_URL_SCHEME;
      }
    }

    if (rv == OK) {
      transaction_->SetBeforeNetworkStartCallback(
          base::Bind(&URLRequestHttpJob::NotifyBeforeNetworkStart,
                     base::Unretained(this)));
      transaction_->SetBeforeHeadersSentCallback(
          base::Bind(&URLRequestHttpJob::NotifyBeforeSendHeadersCallback,
                     base::Unretained(this)));

      if (!throttling_entry_.get() ||
          !throttling_entry_->ShouldRejectRequest(*request_)) {
        rv = transaction_->Start(
            &request_info_, start_callback_, request_->net_log());
        start_time_ = base::TimeTicks::Now();
      } else {
        // Special error code for the exponential back-off module.
        rv = ERR_TEMPORARILY_THROTTLED;
      }
    }
  }

  if (rv == ERR_IO_PENDING)
    return;

  // The transaction started synchronously, but we need to notify the
  // URLRequest delegate via the message loop.
  base::ThreadTaskRunnerHandle::Get()->PostTask(
      FROM_HERE, base::Bind(&URLRequestHttpJob::OnStartCompleted,
                            weak_factory_.GetWeakPtr(), rv));
}
```
在这个函数中，做了这样一些事情：
1. 获取URLRequestContext的NetworkQualityEstimator，以向其通知Transaction启动事件。
2. 向network_delegate通知Transaction启动事件。
3. 如果已经创建了HttpTransaction，则restart。
4. 还没有创建HttpTransaction的情况，先通过URLRequestContext的HttpTransactionFactory创建HttpTransaction。
5. 处理WS和WSS的情况。
6. 为HttpTransaction设置BeforeNetworkStartCallback和BeforeHeadersSentCallback回调。
7. 执行HttpTransaction的Start()。
8. Post一个Task URLRequestHttpJob::OnStartCompleted()。

## HttpTransaction的创建

HttpTransaction由URLRequestContext的HttpTransactionFactory创建的。回顾URLRequestContextBuilder::Build()的如下这段代码：
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

    http_transaction_factory.reset(new HttpCache(
        storage->http_network_session(), std::move(http_cache_backend), true));
  } else {
    http_transaction_factory.reset(
        new HttpNetworkLayer(storage->http_network_session()));
  }
  storage->set_http_transaction_factory(std::move(http_transaction_factory));
```
这里根据是否启用Cache，而使用不同的HttpTransactionFactory。启用了Cache时，会使用HttpCache作为HttpTransactionFactory；而没有启用Cache时，则会使用HttpNetworkLayer作为HttpTransactionFactory。

在URLRequestHttpJob::StartTransactionInternal()中会调用HttpTransactionFactory的CreateTransaction()来创建Transaction。

HttpCache的CreateTransaction()定义(chromium_android/src/net/http/http_cache.cc)如下：
```
int HttpCache::CreateTransaction(RequestPriority priority,
                                 std::unique_ptr<HttpTransaction>* trans) {
  // Do lazy initialization of disk cache if needed.
  if (!disk_cache_.get()) {
    // We don't care about the result.
    CreateBackend(NULL, CompletionCallback());
  }

   HttpCache::Transaction* transaction =
      new HttpCache::Transaction(priority, this);
   if (bypass_lock_for_test_)
    transaction->BypassLockForTest();
   if (fail_conditionalization_for_test_)
     transaction->FailConditionalizationForTest();

  trans->reset(transaction);
  return OK;
}
```
这里是创建类型为HttpCache::Transaction的对象。HttpCache::Transaction (chromium_android/src/net/http/http_cache_transaction.cc)看起来还是蛮复杂的：
```
HttpCache::Transaction::Transaction(RequestPriority priority, HttpCache* cache)
    : next_state_(STATE_NONE),
      request_(NULL),
      priority_(priority),
      cache_(cache->GetWeakPtr()),
      entry_(NULL),
      new_entry_(NULL),
      new_response_(NULL),
      mode_(NONE),
      reading_(false),
      invalid_range_(false),
      truncated_(false),
      is_sparse_(false),
      range_requested_(false),
      handling_206_(false),
      cache_pending_(false),
      done_reading_(false),
      vary_mismatch_(false),
      couldnt_conditionalize_request_(false),
      bypass_lock_for_test_(false),
      fail_conditionalization_for_test_(false),
      io_buf_len_(0),
      read_offset_(0),
      effective_load_flags_(0),
      write_len_(0),
      cache_entry_status_(CacheEntryStatus::ENTRY_UNDEFINED),
      validation_cause_(VALIDATION_CAUSE_UNDEFINED),
      total_received_bytes_(0),
      total_sent_bytes_(0),
      websocket_handshake_stream_base_create_helper_(NULL),
      weak_factory_(this) {
  static_assert(HttpCache::Transaction::kNumValidationHeaders ==
                    arraysize(kValidationHeaders),
                "invalid number of validation headers");

  io_callback_ = base::Bind(&Transaction::OnIOComplete,
                              weak_factory_.GetWeakPtr());
}
```

然后来看HttpNetworkLayer (chromium_android/src/net/http/http_network_layer.cc)的CreateTransaction()函数：
```
int HttpNetworkLayer::CreateTransaction(
    RequestPriority priority,
    std::unique_ptr<HttpTransaction>* trans) {
  if (suspended_)
    return ERR_NETWORK_IO_SUSPENDED;

  trans->reset(new HttpNetworkTransaction(priority, GetSession()));
  return OK;
}
```
这里创建的则是HttpNetworkTransaction。

至此，http请求启动的过程就基本分析完了。
