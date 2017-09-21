---
title: Android 图形系统之图形缓冲区分配
date: 2017-09-20 13:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
---

BufferQueue 是 Android 中所有图形处理操作的核心。它的作用很简单：将生成图形数据缓冲区的一方（生产者）连接到接受数据以显示或进一步处理的一方（消费者）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。
<!--more-->
Android 定义了一个类 `BufferQueue`，用于创建 BufferQueue、生产者和消费者。该类定义（位于`frameworks/native/include/gui/BufferQueue.h`）如下：
```
namespace android {

class BufferQueue {
public:
    // BufferQueue will keep track of at most this value of buffers.
    // Attempts at runtime to increase the number of buffers past this will fail.
    enum { NUM_BUFFER_SLOTS = BufferQueueDefs::NUM_BUFFER_SLOTS };
    // Used as a placeholder slot# when the value isn't pointing to an existing buffer.
    enum { INVALID_BUFFER_SLOT = BufferItem::INVALID_BUFFER_SLOT };
    // Alias to <IGraphicBufferConsumer.h> -- please scope from there in future code!
    enum {
        NO_BUFFER_AVAILABLE = IGraphicBufferConsumer::NO_BUFFER_AVAILABLE,
        PRESENT_LATER = IGraphicBufferConsumer::PRESENT_LATER,
    };

    // When in async mode we reserve two slots in order to guarantee that the
    // producer and consumer can run asynchronously.
    enum { MAX_MAX_ACQUIRED_BUFFERS = NUM_BUFFER_SLOTS - 2 };

    // for backward source compatibility
    typedef ::android::ConsumerListener ConsumerListener;

    // ProxyConsumerListener is a ConsumerListener implementation that keeps a weak
    // reference to the actual consumer object.  It forwards all calls to that
    // consumer object so long as it exists.
    //
    // This class exists to avoid having a circular reference between the
    // BufferQueue object and the consumer object.  The reason this can't be a weak
    // reference in the BufferQueue class is because we're planning to expose the
    // consumer side of a BufferQueue as a binder interface, which doesn't support
    // weak references.
    class ProxyConsumerListener : public BnConsumerListener {
    public:
        ProxyConsumerListener(const wp<ConsumerListener>& consumerListener);
        virtual ~ProxyConsumerListener();
        virtual void onFrameAvailable(const BufferItem& item) override;
        virtual void onFrameReplaced(const BufferItem& item) override;
        virtual void onBuffersReleased() override;
        virtual void onSidebandStreamChanged() override;
        virtual bool getFrameTimestamps(uint64_t frameNumber,
                FrameTimestamps* outTimestamps) const override;
    private:
        // mConsumerListener is a weak reference to the IConsumerListener.  This is
        // the raison d'etre of ProxyConsumerListener.
        wp<ConsumerListener> mConsumerListener;
    };

    // BufferQueue manages a pool of gralloc memory slots to be used by
    // producers and consumers. allocator is used to allocate all the
    // needed gralloc buffers.
    static void createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
            sp<IGraphicBufferConsumer>* outConsumer,
            const sp<IGraphicBufferAlloc>& allocator = NULL);

private:
    BufferQueue(); // Create through createBufferQueue
};

// ----------------------------------------------------------------------------
}; // namespace android
```

`BufferQueue` 定义了一个类 `ProxyConsumerListener` 用于方便 `ConsumerListener` 的 IPC，它会把所有对它的调用，都转发给实际的 consumer 对象。

`BufferQueue` 类只有一个静态成员函数 `createBufferQueue()` 用于创建 BufferQueue，该函数定义（位于 `frameworks/native/libs/gui/BufferQueueCore.cpp`）如下：
```
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
        sp<IGraphicBufferConsumer>* outConsumer,
        const sp<IGraphicBufferAlloc>& allocator) {
    LOG_ALWAYS_FATAL_IF(outProducer == NULL,
            "BufferQueue: outProducer must not be NULL");
    LOG_ALWAYS_FATAL_IF(outConsumer == NULL,
            "BufferQueue: outConsumer must not be NULL");

    sp<BufferQueueCore> core(new BufferQueueCore(allocator));
    LOG_ALWAYS_FATAL_IF(core == NULL,
            "BufferQueue: failed to create BufferQueueCore");

    sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core));
    LOG_ALWAYS_FATAL_IF(producer == NULL,
            "BufferQueue: failed to create BufferQueueProducer");

    sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
    LOG_ALWAYS_FATAL_IF(consumer == NULL,
            "BufferQueue: failed to create BufferQueueConsumer");

    *outProducer = producer;
    *outConsumer = consumer;
}
```

`createBufferQueue()`函数基于 `IGraphicBufferAlloc` 创建 `BufferQueueCore`，并基于后者创建 `BufferQueueProducer` 和 `BufferQueueConsumer` 返回给调用者。Android 图形系统的核心在 BufferQueue，BufferQueue 的核心则在 `BufferQueueCore` 类，而不是 `BufferQueue` 类。

# 图形缓冲区分配器 IGraphicBufferAlloc

`BufferQueueConsumer` 管理的图形缓冲区均由 `BufferQueueProducer` 通过 `IGraphicBufferAlloc` 分配。创建 BufferQueue 时，传入的 allocator 通常为空值，此时 `BufferQueueCore` 通过如下方式（位于
 `frameworks/native/libs/gui/BufferQueueCore.cpp`）获得 `IGraphicBufferAlloc`：
```
    if (allocator == NULL) {
        sp<ISurfaceComposer> composer(ComposerService::getComposerService());
        mAllocator = composer->createGraphicBufferAlloc();
        if (mAllocator == NULL) {
            BQ_LOGE("createGraphicBufferAlloc failed");
        }
    }
```

`BufferQueueCore` 的 `IGraphicBufferAlloc` 来自于 `ComposerService`。`ComposerService` 定义（位于`frameworks/native/include/private/gui/ComposerService.h`）如下：
```
class ComposerService : public Singleton<ComposerService>
{
    sp<ISurfaceComposer> mComposerService;
    sp<IBinder::DeathRecipient> mDeathObserver;
    Mutex mLock;

    ComposerService();
    void connectLocked();
    void composerServiceDied();
    friend class Singleton<ComposerService>;
public:

    // Get a connection to the Composer Service.  This will block until
    // a connection is established.
    static sp<ISurfaceComposer> getComposerService();
};
```

`ComposerService` 类本身仅仅持有到 composer service，如 SurfaceFlinger 的连接，即 `ISurfaceComposer`。如果远程服务挂掉了，这个类通过 Binder 的 linkToDeath 机制得到通知，并将重新建立连接。

`ComposerService` 类的成员函数定义（位于 `frameworks/native/libs/gui/SurfaceComposerClient.cpp`）如下：
```
ANDROID_SINGLETON_STATIC_INSTANCE(ComposerService);

ComposerService::ComposerService()
: Singleton<ComposerService>() {
    Mutex::Autolock _l(mLock);
    connectLocked();
}

void ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    while (getService(name, &mComposerService) != NO_ERROR) {
        usleep(250000);
    }
    assert(mComposerService != NULL);

    // Create the death listener.
    class DeathObserver : public IBinder::DeathRecipient {
        ComposerService& mComposerService;
        virtual void binderDied(const wp<IBinder>& who) {
            ALOGW("ComposerService remote (surfaceflinger) died [%p]",
                  who.unsafe_get());
            mComposerService.composerServiceDied();
        }
     public:
        DeathObserver(ComposerService& mgr) : mComposerService(mgr) { }
    };

    mDeathObserver = new DeathObserver(*const_cast<ComposerService*>(this));
    IInterface::asBinder(mComposerService)->linkToDeath(mDeathObserver);
}

/*static*/ sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    Mutex::Autolock _l(instance.mLock);
    if (instance.mComposerService == NULL) {
        ComposerService::getInstance().connectLocked();
        assert(instance.mComposerService != NULL);
        ALOGD("ComposerService reconnected");
    }
    return instance.mComposerService;
}
```

实际的 composer service 是 SurfaceFlinger。SurfaceFlinger 中是这样创建 `IGraphicBufferAlloc` 的（配置使用 HWC2 的情况，位于`frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp`）：
```
sp<IGraphicBufferAlloc> SurfaceFlinger::createGraphicBufferAlloc()
{
    sp<GraphicBufferAlloc> gba(new GraphicBufferAlloc());
    return gba;
}
```

`IGraphicBufferAlloc` 实际为 `GraphicBufferAlloc`，该类定义（位于 `frameworks/native/include/gui/GraphicBufferAlloc.h`）如下：
```
namespace android {
// ---------------------------------------------------------------------------

class GraphicBuffer;

class GraphicBufferAlloc : public BnGraphicBufferAlloc {
public:
    GraphicBufferAlloc();
    virtual ~GraphicBufferAlloc();
    virtual sp<GraphicBuffer> createGraphicBuffer(uint32_t width,
            uint32_t height, PixelFormat format, uint32_t usage,
            std::string requestorName, status_t* error) override;
};

// ---------------------------------------------------------------------------
}; // namespace android
```

`GraphicBufferAlloc` 继承自 `BnGraphicBufferAlloc`，后者定义（位于 `frameworks/native/include/gui/IGraphicBufferAlloc.h`）如下：
```
class IGraphicBufferAlloc : public IInterface
{
public:
    DECLARE_META_INTERFACE(GraphicBufferAlloc);

    /* Create a new GraphicBuffer for the client to use.
     */
    virtual sp<GraphicBuffer> createGraphicBuffer(uint32_t w, uint32_t h,
            PixelFormat format, uint32_t usage, std::string requestorName,
            status_t* error) = 0;

    sp<GraphicBuffer> createGraphicBuffer(uint32_t w, uint32_t h,
            PixelFormat format, uint32_t usage, status_t* error) {
        return createGraphicBuffer(w, h, format, usage, "<Unknown>", error);
    }
};

// ----------------------------------------------------------------------------

class BnGraphicBufferAlloc : public BnInterface<IGraphicBufferAlloc>
{
public:
    virtual status_t onTransact(uint32_t code,
                                const Parcel& data,
                                Parcel* reply,
                                uint32_t flags = 0);
};

// ----------------------------------------------------------------------------

}; // namespace android
```

`GraphicBufferAlloc` 只有一个成员函数，该函数定义（`frameworks/native/libs/gui/GraphicBufferAlloc.cpp`）如下：
```
namespace android {
// ----------------------------------------------------------------------------

GraphicBufferAlloc::GraphicBufferAlloc() {
}

GraphicBufferAlloc::~GraphicBufferAlloc() {
}

sp<GraphicBuffer> GraphicBufferAlloc::createGraphicBuffer(uint32_t width,
        uint32_t height, PixelFormat format, uint32_t usage,
        std::string requestorName, status_t* error) {
    sp<GraphicBuffer> graphicBuffer(new GraphicBuffer(
            width, height, format, usage, std::move(requestorName)));
    status_t err = graphicBuffer->initCheck();
    *error = err;
    if (err != 0 || graphicBuffer->handle == 0) {
        if (err == NO_MEMORY) {
            GraphicBuffer::dumpAllocationsToSystemLog();
        }
        ALOGE("GraphicBufferAlloc::createGraphicBuffer(w=%d, h=%d) "
             "failed (%s), handle=%p",
                width, height, strerror(-err), graphicBuffer->handle);
        return 0;
    }
    return graphicBuffer;
}

// ----------------------------------------------------------------------------
}; // namespace android
```

`BufferQueueCore` 为 `GraphicBuffer` 的容器，`IGraphicBufferAlloc` 仅仅用于创建 `GraphicBuffer` 对象。但对于实际的图形内存块的管理，还不在 `IGraphicBufferAlloc` 这一层。

# 图形缓冲区 GraphicBuffer

`GraphicBuffer` 类是更底层 操作系统/硬件 层图形内存块的封装，该类的定义如下：
```
class GraphicBuffer
    : public ANativeObjectBase< ANativeWindowBuffer, GraphicBuffer, RefBase >,
      public Flattenable<GraphicBuffer>
{
    friend class Flattenable<GraphicBuffer>;
public:
. . . . . .
    GraphicBuffer();

    // creates w * h buffer
    GraphicBuffer(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
            uint32_t inUsage, std::string requestorName = "<Unknown>");

    // create a buffer from an existing handle
    GraphicBuffer(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
            uint32_t inUsage, uint32_t inStride, native_handle_t* inHandle,
            bool keepOwnership);

    // create a buffer from an existing ANativeWindowBuffer
    GraphicBuffer(ANativeWindowBuffer* buffer, bool keepOwnership);

    // return status
    status_t initCheck() const;

    uint32_t getWidth() const           { return static_cast<uint32_t>(width); }
    uint32_t getHeight() const          { return static_cast<uint32_t>(height); }
    uint32_t getStride() const          { return static_cast<uint32_t>(stride); }
    uint32_t getUsage() const           { return static_cast<uint32_t>(usage); }
    PixelFormat getPixelFormat() const  { return format; }
    Rect getBounds() const              { return Rect(width, height); }
    uint64_t getId() const              { return mId; }

    uint32_t getGenerationNumber() const { return mGenerationNumber; }
    void setGenerationNumber(uint32_t generation) {
        mGenerationNumber = generation;
    }

    status_t reallocate(uint32_t inWidth, uint32_t inHeight,
            PixelFormat inFormat, uint32_t inUsage);

    bool needsReallocation(uint32_t inWidth, uint32_t inHeight,
            PixelFormat inFormat, uint32_t inUsage);

    status_t lock(uint32_t inUsage, void** vaddr);
    status_t lock(uint32_t inUsage, const Rect& rect, void** vaddr);
    // For HAL_PIXEL_FORMAT_YCbCr_420_888
    status_t lockYCbCr(uint32_t inUsage, android_ycbcr *ycbcr);
    status_t lockYCbCr(uint32_t inUsage, const Rect& rect,
            android_ycbcr *ycbcr);
    status_t unlock();
    status_t lockAsync(uint32_t inUsage, void** vaddr, int fenceFd);
    status_t lockAsync(uint32_t inUsage, const Rect& rect, void** vaddr,
            int fenceFd);
    status_t lockAsyncYCbCr(uint32_t inUsage, android_ycbcr *ycbcr,
            int fenceFd);
    status_t lockAsyncYCbCr(uint32_t inUsage, const Rect& rect,
            android_ycbcr *ycbcr, int fenceFd);
    status_t unlockAsync(int *fenceFd);

    ANativeWindowBuffer* getNativeBuffer() const;

    // for debugging
    static void dumpAllocationsToSystemLog();

    // Flattenable protocol
    size_t getFlattenedSize() const;
    size_t getFdCount() const;
    status_t flatten(void*& buffer, size_t& size, int*& fds, size_t& count) const;
    status_t unflatten(void const*& buffer, size_t& size, int const*& fds, size_t& count);

private:
    ~GraphicBuffer();

    enum {
        ownNone   = 0,
        ownHandle = 1,
        ownData   = 2,
    };

    inline const GraphicBufferMapper& getBufferMapper() const {
        return mBufferMapper;
    }
    inline GraphicBufferMapper& getBufferMapper() {
        return mBufferMapper;
    }
    uint8_t mOwner;

private:
    friend class Surface;
    friend class BpSurface;
    friend class BnSurface;
    friend class LightRefBase<GraphicBuffer>;
    GraphicBuffer(const GraphicBuffer& rhs);
    GraphicBuffer& operator = (const GraphicBuffer& rhs);
    const GraphicBuffer& operator = (const GraphicBuffer& rhs) const;

    status_t initSize(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
            uint32_t inUsage, std::string requestorName);

    void free_handle();

    GraphicBufferMapper& mBufferMapper;
    ssize_t mInitCheck;

    // If we're wrapping another buffer then this reference will make sure it
    // doesn't get freed.
    sp<ANativeWindowBuffer> mWrappedBuffer;

    uint64_t mId;

    // Stores the generation number of this buffer. If this number does not
    // match the BufferQueue's internal generation number (set through
    // IGBP::setGenerationNumber), attempts to attach the buffer will fail.
    uint32_t mGenerationNumber;
};
```

`ANativeObjectBase` 模板的声明（位于 `frameworks/native/include/ui/ANativeObjectBase.h`）是这样的：
```
template <typename NATIVE_TYPE, typename TYPE, typename REF>
class ANativeObjectBase : public NATIVE_TYPE, public REF
{
public:
    // Disambiguate between the incStrong in REF and NATIVE_TYPE
    void incStrong(const void* id) const {
        REF::incStrong(id);
    }
    void decStrong(const void* id) const {
        REF::decStrong(id);
    }

protected:
    typedef ANativeObjectBase<NATIVE_TYPE, TYPE, REF> BASE;
    ANativeObjectBase() : NATIVE_TYPE(), REF() {
        NATIVE_TYPE::common.incRef = incRef;
        NATIVE_TYPE::common.decRef = decRef;
    }
    static inline TYPE* getSelf(NATIVE_TYPE* self) {
        return static_cast<TYPE*>(self);
    }
    static inline TYPE const* getSelf(NATIVE_TYPE const* self) {
        return static_cast<TYPE const *>(self);
    }
    static inline TYPE* getSelf(android_native_base_t* base) {
        return getSelf(reinterpret_cast<NATIVE_TYPE*>(base));
    }
    static inline TYPE const * getSelf(android_native_base_t const* base) {
        return getSelf(reinterpret_cast<NATIVE_TYPE const*>(base));
    }
    static void incRef(android_native_base_t* base) {
        ANativeObjectBase* self = getSelf(base);
        self->incStrong(self);
    }
    static void decRef(android_native_base_t* base) {
        ANativeObjectBase* self = getSelf(base);
        self->decStrong(self);
    }
};

} // namespace android
#endif // __cplusplus
```

以此来看，`GraphicBuffer` 也将继承 `ANativeWindowBuffer`。`ANativeWindowBuffer` 定义（位于 `system/core/include/system/window.h`）如下：
```
typedef const native_handle_t* buffer_handle_t;
. . . . . .
typedef struct ANativeWindowBuffer
{
#ifdef __cplusplus
    ANativeWindowBuffer() {
        common.magic = ANDROID_NATIVE_BUFFER_MAGIC;
        common.version = sizeof(ANativeWindowBuffer);
        memset(common.reserved, 0, sizeof(common.reserved));
    }

    // Implement the methods that sp<ANativeWindowBuffer> expects so that it
    // can be used to automatically refcount ANativeWindowBuffer's.
    void incStrong(const void* /*id*/) const {
        common.incRef(const_cast<android_native_base_t*>(&common));
    }
    void decStrong(const void* /*id*/) const {
        common.decRef(const_cast<android_native_base_t*>(&common));
    }
#endif

    struct android_native_base_t common;

    int width;
    int height;
    int stride;
    int format;
    int usage;

    void* reserved[2];

    buffer_handle_t handle;

    void* reserved_proc[8];
} ANativeWindowBuffer_t;
```

这个结构体描述了更底层 操作系统/硬件 层图形内存块的信息，包括图形内存块的句柄 `handle`，图像的宽度、高度，像素格式等。图形内存块的句柄类型 `buffer_handle_t` 为 `const native_handle_t*` 的别名，`native_handle_t` 定义（位于 `system/core/include/cutils/native_handle.h`）如下：
```
typedef struct native_handle
{
    int version;        /* sizeof(native_handle_t) */
    int numFds;         /* number of file-descriptors at &data[0] */
    int numInts;        /* number of ints at &data[numFds] */
    int data[0];        /* numFds + numInts ints */
} native_handle_t;
```

可以看到 `GraphicBuffer` 类的主要职责主要有三块：
1. 主要通过继承自 `ANativeWindowBuffer` 结构体的成员，来描述图形内存块的信息。
2. 分配释放图形内存块。这主要通过 `initSize()` / `reallocate()` / `free_handle()` 等操作完成。
3. 分配的图形内存块未必已经映射到应用程序的虚拟地址空间了。应用程序要想像访问普通内存那样访问图形内存块，还需要通过 ***`lockXXX`*** 操作将图形内存块映射到应用程序进程的虚拟地址空间内。应用程序在把图形内存块还回去的时候则需要 ***`unlockXXX`*** 操作。

此外 `GraphicBuffer` 的 `mInitCheck` 用于记录图形缓冲区的状态；`mGenerationNumber` 用于记录 generation number；`mId` 用于标识图形缓冲区，它通过如下方式计算得到：
```
static uint64_t getUniqueId() {
    static volatile int32_t nextId = 0;
    uint64_t id = static_cast<uint64_t>(getpid()) << 32;
    id |= static_cast<uint32_t>(android_atomic_inc(&nextId));
    return id;
}
```

mID 通过进程的 PID 和一个不断递增的整数计算获得。

`GraphicBuffer` 依赖于 `GraphicBufferAllocator` 完成图形内存块的分配和释放，依赖于 `GraphicBufferMapper` 执行图形内存块的 lock/unlock 操作。

`GraphicBuffer` 的图形内存块分配和释放操作实现（位于 `frameworks/native/libs/ui/GraphicBuffer.cpp`）如下：
```
GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inUsage, std::string requestorName)
    : BASE(), mOwner(ownData), mBufferMapper(GraphicBufferMapper::get()),
      mInitCheck(NO_ERROR), mId(getUniqueId()), mGenerationNumber(0)
{
    width  =
    height =
    stride =
    format =
    usage  = 0;
    handle = NULL;
    mInitCheck = initSize(inWidth, inHeight, inFormat, inUsage,
            std::move(requestorName));
}

GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inUsage, uint32_t inStride,
        native_handle_t* inHandle, bool keepOwnership)
    : BASE(), mOwner(keepOwnership ? ownHandle : ownNone),
      mBufferMapper(GraphicBufferMapper::get()),
      mInitCheck(NO_ERROR), mId(getUniqueId()), mGenerationNumber(0)
{
    width  = static_cast<int>(inWidth);
    height = static_cast<int>(inHeight);
    stride = static_cast<int>(inStride);
    format = inFormat;
    usage  = static_cast<int>(inUsage);
    handle = inHandle;
}

GraphicBuffer::GraphicBuffer(ANativeWindowBuffer* buffer, bool keepOwnership)
    : BASE(), mOwner(keepOwnership ? ownHandle : ownNone),
      mBufferMapper(GraphicBufferMapper::get()),
      mInitCheck(NO_ERROR), mWrappedBuffer(buffer), mId(getUniqueId()),
      mGenerationNumber(0)
{
    width  = buffer->width;
    height = buffer->height;
    stride = buffer->stride;
    format = buffer->format;
    usage  = buffer->usage;
    handle = buffer->handle;
}

GraphicBuffer::~GraphicBuffer()
{
    if (handle) {
        free_handle();
    }
}

void GraphicBuffer::free_handle()
{
    if (mOwner == ownHandle) {
        mBufferMapper.unregisterBuffer(handle);
        native_handle_close(handle);
        native_handle_delete(const_cast<native_handle*>(handle));
    } else if (mOwner == ownData) {
        GraphicBufferAllocator& allocator(GraphicBufferAllocator::get());
        allocator.free(handle);
    }
    handle = NULL;
    mWrappedBuffer = 0;
}
. . . . . .
status_t GraphicBuffer::reallocate(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inUsage)
{
    if (mOwner != ownData)
        return INVALID_OPERATION;

    if (handle &&
            static_cast<int>(inWidth) == width &&
            static_cast<int>(inHeight) == height &&
            inFormat == format &&
            static_cast<int>(inUsage) == usage)
        return NO_ERROR;

    if (handle) {
        GraphicBufferAllocator& allocator(GraphicBufferAllocator::get());
        allocator.free(handle);
        handle = 0;
    }
    return initSize(inWidth, inHeight, inFormat, inUsage, "[Reallocation]");
}
. . . . . .
status_t GraphicBuffer::initSize(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inUsage, std::string requestorName)
{
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    uint32_t outStride = 0;
    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inUsage,
            &handle, &outStride, mId, std::move(requestorName));
    if (err == NO_ERROR) {
        width = static_cast<int>(inWidth);
        height = static_cast<int>(inHeight);
        format = inFormat;
        usage = static_cast<int>(inUsage);
        stride = static_cast<int>(outStride);
    }
    return err;
}
```

`GraphicBuffer` 的图形内存块的 lock/unlock 操作实现（位于 `frameworks/native/libs/ui/GraphicBuffer.cpp`）如下：
```
status_t GraphicBuffer::lock(uint32_t inUsage, void** vaddr)
{
    const Rect lockBounds(width, height);
    status_t res = lock(inUsage, lockBounds, vaddr);
    return res;
}

status_t GraphicBuffer::lock(uint32_t inUsage, const Rect& rect, void** vaddr)
{
    if (rect.left < 0 || rect.right  > width ||
        rect.top  < 0 || rect.bottom > height) {
        ALOGE("locking pixels (%d,%d,%d,%d) outside of buffer (w=%d, h=%d)",
                rect.left, rect.top, rect.right, rect.bottom,
                width, height);
        return BAD_VALUE;
    }
    status_t res = getBufferMapper().lock(handle, inUsage, rect, vaddr);
    return res;
}

status_t GraphicBuffer::lockYCbCr(uint32_t inUsage, android_ycbcr* ycbcr)
{
    const Rect lockBounds(width, height);
    status_t res = lockYCbCr(inUsage, lockBounds, ycbcr);
    return res;
}

status_t GraphicBuffer::lockYCbCr(uint32_t inUsage, const Rect& rect,
        android_ycbcr* ycbcr)
{
    if (rect.left < 0 || rect.right  > width ||
        rect.top  < 0 || rect.bottom > height) {
        ALOGE("locking pixels (%d,%d,%d,%d) outside of buffer (w=%d, h=%d)",
                rect.left, rect.top, rect.right, rect.bottom,
                width, height);
        return BAD_VALUE;
    }
    status_t res = getBufferMapper().lockYCbCr(handle, inUsage, rect, ycbcr);
    return res;
}

status_t GraphicBuffer::unlock()
{
    status_t res = getBufferMapper().unlock(handle);
    return res;
}

status_t GraphicBuffer::lockAsync(uint32_t inUsage, void** vaddr, int fenceFd)
{
    const Rect lockBounds(width, height);
    status_t res = lockAsync(inUsage, lockBounds, vaddr, fenceFd);
    return res;
}

status_t GraphicBuffer::lockAsync(uint32_t inUsage, const Rect& rect,
        void** vaddr, int fenceFd)
{
    if (rect.left < 0 || rect.right  > width ||
        rect.top  < 0 || rect.bottom > height) {
        ALOGE("locking pixels (%d,%d,%d,%d) outside of buffer (w=%d, h=%d)",
                rect.left, rect.top, rect.right, rect.bottom,
                width, height);
        return BAD_VALUE;
    }
    status_t res = getBufferMapper().lockAsync(handle, inUsage, rect, vaddr,
            fenceFd);
    return res;
}

status_t GraphicBuffer::lockAsyncYCbCr(uint32_t inUsage, android_ycbcr* ycbcr,
        int fenceFd)
{
    const Rect lockBounds(width, height);
    status_t res = lockAsyncYCbCr(inUsage, lockBounds, ycbcr, fenceFd);
    return res;
}

status_t GraphicBuffer::lockAsyncYCbCr(uint32_t inUsage, const Rect& rect,
        android_ycbcr* ycbcr, int fenceFd)
{
    if (rect.left < 0 || rect.right  > width ||
        rect.top  < 0 || rect.bottom > height) {
        ALOGE("locking pixels (%d,%d,%d,%d) outside of buffer (w=%d, h=%d)",
                rect.left, rect.top, rect.right, rect.bottom,
                width, height);
        return BAD_VALUE;
    }
    status_t res = getBufferMapper().lockAsyncYCbCr(handle, inUsage, rect,
            ycbcr, fenceFd);
    return res;
}

status_t GraphicBuffer::unlockAsync(int *fenceFd)
{
    status_t res = getBufferMapper().unlockAsync(handle, fenceFd);
    return res;
}
```

这些操作基本上都是比较直接的委托。

#  GraphicBufferAllocator 和 GraphicBufferMapper

`GraphicBufferAllocator` 和 `GraphicBufferMapper` 则依赖于 `Gralloc1::Loader` 和
 `Gralloc1::Device` 完成图形内存的分配释放和 `lock` / `unlock` 操作，其中 `Gralloc1::Loader` 用于加载 HAL 层的 gralloc 模块并创建 `Gralloc1::Device`，`Gralloc1::Device` 则用于执行最终的图形内存的分配释放和 `lock` / `unlock` 操作。

`GraphicBufferAllocator` 类定义（位于 `frameworks/native/include/ui/GraphicBufferAllocator.h`）如下：
```
namespace android {

class Gralloc1Loader;
class String8;

class GraphicBufferAllocator : public Singleton<GraphicBufferAllocator>
{
public:
    enum {
        USAGE_SW_READ_NEVER     = GRALLOC1_CONSUMER_USAGE_CPU_READ_NEVER,
        USAGE_SW_READ_RARELY    = GRALLOC1_CONSUMER_USAGE_CPU_READ,
        USAGE_SW_READ_OFTEN     = GRALLOC1_CONSUMER_USAGE_CPU_READ_OFTEN,
        USAGE_SW_READ_MASK      = GRALLOC1_CONSUMER_USAGE_CPU_READ_OFTEN,

        USAGE_SW_WRITE_NEVER    = GRALLOC1_PRODUCER_USAGE_CPU_WRITE_NEVER,
        USAGE_SW_WRITE_RARELY   = GRALLOC1_PRODUCER_USAGE_CPU_WRITE,
        USAGE_SW_WRITE_OFTEN    = GRALLOC1_PRODUCER_USAGE_CPU_WRITE_OFTEN,
        USAGE_SW_WRITE_MASK     = GRALLOC1_PRODUCER_USAGE_CPU_WRITE_OFTEN,

        USAGE_SOFTWARE_MASK     = USAGE_SW_READ_MASK|USAGE_SW_WRITE_MASK,

        USAGE_HW_TEXTURE        = GRALLOC1_CONSUMER_USAGE_GPU_TEXTURE,
        USAGE_HW_RENDER         = GRALLOC1_PRODUCER_USAGE_GPU_RENDER_TARGET,
        USAGE_HW_2D             = 0x00000400, // Deprecated
        USAGE_HW_MASK           = 0x00071F00, // Deprecated
    };

    static inline GraphicBufferAllocator& get() { return getInstance(); }

    status_t allocate(uint32_t w, uint32_t h, PixelFormat format,
            uint32_t usage, buffer_handle_t* handle, uint32_t* stride,
            uint64_t graphicBufferId, std::string requestorName);

    status_t free(buffer_handle_t handle);

    void dump(String8& res) const;
    static void dumpToSystemLog();

private:
    struct alloc_rec_t {
        uint32_t width;
        uint32_t height;
        uint32_t stride;
        PixelFormat format;
        uint32_t usage;
        size_t size;
        std::string requestorName;
    };

    static Mutex sLock;
    static KeyedVector<buffer_handle_t, alloc_rec_t> sAllocList;

    friend class Singleton<GraphicBufferAllocator>;
    GraphicBufferAllocator();
    ~GraphicBufferAllocator();

    std::unique_ptr<Gralloc1::Loader> mLoader;
    std::unique_ptr<Gralloc1::Device> mDevice;
};

// ---------------------------------------------------------------------------
}; // namespace android
```

`GraphicBufferAllocator` 主要定义了分配图形内存块的 `allocate()` 和释放图形内存块的 `free()` 两个函数，这两个函数的实现（位于 `frameworks/native/libs/ui/GraphicBufferAllocator.cpp`）如下：
```
ANDROID_SINGLETON_STATIC_INSTANCE( GraphicBufferAllocator )

Mutex GraphicBufferAllocator::sLock;
KeyedVector<buffer_handle_t,
    GraphicBufferAllocator::alloc_rec_t> GraphicBufferAllocator::sAllocList;

GraphicBufferAllocator::GraphicBufferAllocator()
  : mLoader(std::make_unique<Gralloc1::Loader>()),
    mDevice(mLoader->getDevice()) {}

GraphicBufferAllocator::~GraphicBufferAllocator() {}
. . . . . .
status_t GraphicBufferAllocator::allocate(uint32_t width, uint32_t height,
        PixelFormat format, uint32_t usage, buffer_handle_t* handle,
        uint32_t* stride, uint64_t graphicBufferId, std::string requestorName)
{
    ATRACE_CALL();

    // make sure to not allocate a N x 0 or 0 x N buffer, since this is
    // allowed from an API stand-point allocate a 1x1 buffer instead.
    if (!width || !height)
        width = height = 1;

    // Filter out any usage bits that should not be passed to the gralloc module
    usage &= GRALLOC_USAGE_ALLOC_MASK;

    auto descriptor = mDevice->createDescriptor();
    auto error = descriptor->setDimensions(width, height);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("Failed to set dimensions to (%u, %u): %d", width, height, error);
        return BAD_VALUE;
    }
    error = descriptor->setFormat(static_cast<android_pixel_format_t>(format));
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("Failed to set format to %d: %d", format, error);
        return BAD_VALUE;
    }
    error = descriptor->setProducerUsage(
            static_cast<gralloc1_producer_usage_t>(usage));
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("Failed to set producer usage to %u: %d", usage, error);
        return BAD_VALUE;
    }
    error = descriptor->setConsumerUsage(
            static_cast<gralloc1_consumer_usage_t>(usage));
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("Failed to set consumer usage to %u: %d", usage, error);
        return BAD_VALUE;
    }

    error = mDevice->allocate(descriptor, graphicBufferId, handle);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("Failed to allocate (%u x %u) format %d usage %u: %d",
                width, height, format, usage, error);
        return NO_MEMORY;
    }

    error = mDevice->getStride(*handle, stride);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGW("Failed to get stride from buffer: %d", error);
    }

    if (error == NO_ERROR) {
        Mutex::Autolock _l(sLock);
        KeyedVector<buffer_handle_t, alloc_rec_t>& list(sAllocList);
        uint32_t bpp = bytesPerPixel(format);
        alloc_rec_t rec;
        rec.width = width;
        rec.height = height;
        rec.stride = *stride;
        rec.format = format;
        rec.usage = usage;
        rec.size = static_cast<size_t>(height * (*stride) * bpp);
        rec.requestorName = std::move(requestorName);
        list.add(*handle, rec);
    }

    return NO_ERROR;
}

status_t GraphicBufferAllocator::free(buffer_handle_t handle)
{
    ATRACE_CALL();

    auto error = mDevice->release(handle);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("Failed to free buffer: %d", error);
    }

    Mutex::Autolock _l(sLock);
    KeyedVector<buffer_handle_t, alloc_rec_t>& list(sAllocList);
    list.removeItem(handle);

    return NO_ERROR;
}
```

`GraphicBufferAllocator` 分配图形内存块时，步骤如下：
1. 通过 `Gralloc1::Device` 创建 `Gralloc1::Descriptor`，并为其设置要分配的图形内存块的规格，包括图像的长和宽，图像的像素格式，图形内存块的使用场景，其中图形内存块的使用场景参数主要用于性能优化。
2. 以 `Gralloc1::Descriptor`、图形内存块的标识 ID，和图形内存块句柄的指针作为参数，通过 `Gralloc1::Device` 的 `allocate()` 分配图形内存块，分配的结果通过图形内存块句柄的指针返回。
3. 分配完成之后，可以通过`Gralloc1::Device` 图形内存块的步进，即单行像素数据占用的内存字节数。底层可能为了性能优化，内存对齐等，分配的内存块可能大于保存实际图像所需要的大小。
4. 对分配结果做记录。`GraphicBufferAllocator` 维护一个图形内存块句柄到图形内存块规格的映射。

`GraphicBufferAllocator` 释放图形内存块时的步骤则基本相反：
1. 通过 `Gralloc1::Device` 释放图形内存块句柄。
2. 移除分配记录。

`GraphicBufferMapper` 类提供了对图形内存块的 lock / unlock 操作。该类定义（位于 `frameworks/native/include/ui/GraphicBufferMapper.h`）如下：
```
class GraphicBufferMapper : public Singleton<GraphicBufferMapper>
{
public:
    static inline GraphicBufferMapper& get() { return getInstance(); }

    status_t registerBuffer(buffer_handle_t handle);
    status_t registerBuffer(const GraphicBuffer* buffer);

    status_t unregisterBuffer(buffer_handle_t handle);

    status_t lock(buffer_handle_t handle,
            uint32_t usage, const Rect& bounds, void** vaddr);

    status_t lockYCbCr(buffer_handle_t handle,
            uint32_t usage, const Rect& bounds, android_ycbcr *ycbcr);

    status_t unlock(buffer_handle_t handle);

    status_t lockAsync(buffer_handle_t handle,
            uint32_t usage, const Rect& bounds, void** vaddr, int fenceFd);

    status_t lockAsyncYCbCr(buffer_handle_t handle,
            uint32_t usage, const Rect& bounds, android_ycbcr *ycbcr,
            int fenceFd);

    status_t unlockAsync(buffer_handle_t handle, int *fenceFd);

private:
    friend class Singleton<GraphicBufferMapper>;

    GraphicBufferMapper();

    std::unique_ptr<Gralloc1::Loader> mLoader;
    std::unique_ptr<Gralloc1::Device> mDevice;
};
```

`GraphicBufferMapper` 通过 `Gralloc1::Device` 提供对图形内存块的 lock / unlock 操作，这些操作的定义（位于 `frameworks/native/libs/ui/GraphicBufferMapper.cpp`）如下：
```
ANDROID_SINGLETON_STATIC_INSTANCE( GraphicBufferMapper )

GraphicBufferMapper::GraphicBufferMapper()
  : mLoader(std::make_unique<Gralloc1::Loader>()),
    mDevice(mLoader->getDevice()) {}



status_t GraphicBufferMapper::registerBuffer(buffer_handle_t handle)
{
    ATRACE_CALL();

    gralloc1_error_t error = mDevice->retain(handle);
    ALOGW_IF(error != GRALLOC1_ERROR_NONE, "registerBuffer(%p) failed: %d",
            handle, error);

    return error;
}

status_t GraphicBufferMapper::registerBuffer(const GraphicBuffer* buffer)
{
    ATRACE_CALL();

    gralloc1_error_t error = mDevice->retain(buffer);
    ALOGW_IF(error != GRALLOC1_ERROR_NONE, "registerBuffer(%p) failed: %d",
            buffer->getNativeBuffer()->handle, error);

    return error;
}

status_t GraphicBufferMapper::unregisterBuffer(buffer_handle_t handle)
{
    ATRACE_CALL();

    gralloc1_error_t error = mDevice->release(handle);
    ALOGW_IF(error != GRALLOC1_ERROR_NONE, "unregisterBuffer(%p): failed %d",
            handle, error);

    return error;
}

static inline gralloc1_rect_t asGralloc1Rect(const Rect& rect) {
    gralloc1_rect_t outRect{};
    outRect.left = rect.left;
    outRect.top = rect.top;
    outRect.width = rect.width();
    outRect.height = rect.height();
    return outRect;
}

status_t GraphicBufferMapper::lock(buffer_handle_t handle, uint32_t usage,
        const Rect& bounds, void** vaddr)
{
    return lockAsync(handle, usage, bounds, vaddr, -1);
}

status_t GraphicBufferMapper::lockYCbCr(buffer_handle_t handle, uint32_t usage,
        const Rect& bounds, android_ycbcr *ycbcr)
{
    return lockAsyncYCbCr(handle, usage, bounds, ycbcr, -1);
}

status_t GraphicBufferMapper::unlock(buffer_handle_t handle)
{
    int32_t fenceFd = -1;
    status_t error = unlockAsync(handle, &fenceFd);
    if (error == NO_ERROR) {
        sync_wait(fenceFd, -1);
        close(fenceFd);
    }
    return error;
}

status_t GraphicBufferMapper::lockAsync(buffer_handle_t handle,
        uint32_t usage, const Rect& bounds, void** vaddr, int fenceFd)
{
    ATRACE_CALL();

    gralloc1_rect_t accessRegion = asGralloc1Rect(bounds);
    sp<Fence> fence = new Fence(fenceFd);
    gralloc1_error_t error = mDevice->lock(handle,
            static_cast<gralloc1_producer_usage_t>(usage),
            static_cast<gralloc1_consumer_usage_t>(usage),
            &accessRegion, vaddr, fence);
    ALOGW_IF(error != GRALLOC1_ERROR_NONE, "lock(%p, ...) failed: %d", handle,
            error);

    return error;
}

static inline bool isValidYCbCrPlane(const android_flex_plane_t& plane) {
    if (plane.bits_per_component != 8) {
        ALOGV("Invalid number of bits per component: %d",
                plane.bits_per_component);
        return false;
    }
    if (plane.bits_used != 8) {
        ALOGV("Invalid number of bits used: %d", plane.bits_used);
        return false;
    }

    bool hasValidIncrement = plane.h_increment == 1 ||
            (plane.component != FLEX_COMPONENT_Y && plane.h_increment == 2);
    hasValidIncrement = hasValidIncrement && plane.v_increment > 0;
    if (!hasValidIncrement) {
        ALOGV("Invalid increment: h %d v %d", plane.h_increment,
                plane.v_increment);
        return false;
    }

    return true;
}

status_t GraphicBufferMapper::lockAsyncYCbCr(buffer_handle_t handle,
        uint32_t usage, const Rect& bounds, android_ycbcr *ycbcr, int fenceFd)
{
    ATRACE_CALL();

    gralloc1_rect_t accessRegion = asGralloc1Rect(bounds);
    sp<Fence> fence = new Fence(fenceFd);

    if (mDevice->hasCapability(GRALLOC1_CAPABILITY_ON_ADAPTER)) {
        gralloc1_error_t error = mDevice->lockYCbCr(handle,
                static_cast<gralloc1_producer_usage_t>(usage),
                static_cast<gralloc1_consumer_usage_t>(usage),
                &accessRegion, ycbcr, fence);
        ALOGW_IF(error != GRALLOC1_ERROR_NONE, "lockYCbCr(%p, ...) failed: %d",
                handle, error);
        return error;
    }

    uint32_t numPlanes = 0;
    gralloc1_error_t error = mDevice->getNumFlexPlanes(handle, &numPlanes);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGV("Failed to retrieve number of flex planes: %d", error);
        return error;
    }
    if (numPlanes < 3) {
        ALOGV("Not enough planes for YCbCr (%u found)", numPlanes);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }

    std::vector<android_flex_plane_t> planes(numPlanes);
    android_flex_layout_t flexLayout{};
    flexLayout.num_planes = numPlanes;
    flexLayout.planes = planes.data();

    error = mDevice->lockFlex(handle,
            static_cast<gralloc1_producer_usage_t>(usage),
            static_cast<gralloc1_consumer_usage_t>(usage),
            &accessRegion, &flexLayout, fence);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGW("lockFlex(%p, ...) failed: %d", handle, error);
        return error;
    }
    if (flexLayout.format != FLEX_FORMAT_YCbCr) {
        ALOGV("Unable to convert flex-format buffer to YCbCr");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }

    // Find planes
    auto yPlane = planes.cend();
    auto cbPlane = planes.cend();
    auto crPlane = planes.cend();
    for (auto planeIter = planes.cbegin(); planeIter != planes.cend();
            ++planeIter) {
        if (planeIter->component == FLEX_COMPONENT_Y) {
            yPlane = planeIter;
        } else if (planeIter->component == FLEX_COMPONENT_Cb) {
            cbPlane = planeIter;
        } else if (planeIter->component == FLEX_COMPONENT_Cr) {
            crPlane = planeIter;
        }
    }
    if (yPlane == planes.cend()) {
        ALOGV("Unable to find Y plane");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }
    if (cbPlane == planes.cend()) {
        ALOGV("Unable to find Cb plane");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }
    if (crPlane == planes.cend()) {
        ALOGV("Unable to find Cr plane");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }

    // Validate planes
    if (!isValidYCbCrPlane(*yPlane)) {
        ALOGV("Y plane is invalid");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }
    if (!isValidYCbCrPlane(*cbPlane)) {
        ALOGV("Cb plane is invalid");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }
    if (!isValidYCbCrPlane(*crPlane)) {
        ALOGV("Cr plane is invalid");
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }
    if (cbPlane->v_increment != crPlane->v_increment) {
        ALOGV("Cb and Cr planes have different step (%d vs. %d)",
                cbPlane->v_increment, crPlane->v_increment);
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }
    if (cbPlane->h_increment != crPlane->h_increment) {
        ALOGV("Cb and Cr planes have different stride (%d vs. %d)",
                cbPlane->h_increment, crPlane->h_increment);
        unlock(handle);
        return GRALLOC1_ERROR_UNSUPPORTED;
    }

    // Pack plane data into android_ycbcr struct
    ycbcr->y = yPlane->top_left;
    ycbcr->cb = cbPlane->top_left;
    ycbcr->cr = crPlane->top_left;
    ycbcr->ystride = static_cast<size_t>(yPlane->v_increment);
    ycbcr->cstride = static_cast<size_t>(cbPlane->v_increment);
    ycbcr->chroma_step = static_cast<size_t>(cbPlane->h_increment);

    return error;
}

status_t GraphicBufferMapper::unlockAsync(buffer_handle_t handle, int *fenceFd)
{
    ATRACE_CALL();

    sp<Fence> fence = Fence::NO_FENCE;
    gralloc1_error_t error = mDevice->unlock(handle, &fence);
    if (error != GRALLOC1_ERROR_NONE) {
        ALOGE("unlock(%p) failed: %d", handle, error);
        return error;
    }

    *fenceFd = fence->dup();
    return error;
}
```

lock 时，需要以 `Rect` 的形式给 `GraphicBufferMapper` 传入 lock 的区域，这个区域会被做一个转换。`GraphicBufferMapper` 通过 `Gralloc1::Device` 执行 lock 操作。并将 lock 的结果，也就是映射到应用程序进程的虚拟地址空间的图形内存块的地址通过传入的 `vaddr` 返回给调用者。

像素数据格式有 RGB 和 YUV 之分，在 lock YUV 图形内存块时，如果设备支持 `GRALLOC1_CAPABILITY_ON_ADAPTER`，会直接通过 `Gralloc1::Device` 完成操作；否则，通过 `Gralloc1::Device` 的 `lockFlex()` 完成操作。

***这里的 fence 是什么，用来做什么的？***

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# Android OpenGL 图形系统分析系列文章
[在 Android 中使用 OpenGL](https://www.wolfcstech.com/2017/09/13/opengl_on_android_with_sv/)
[Android 图形驱动初始化](https://www.wolfcstech.com/2017/09/14/egl_init_drivers/)
[EGL Context 创建](https://www.wolfcstech.com/2017/09/15/egl_context_creation/)
[Android 图形系统之图形缓冲区分配](https://www.wolfcstech.com/2017/09/20/android_graphics_bufferalloc/)
[Android 图形系统之gralloc](https://www.wolfcstech.com/2017/09/21/android_graphics_gralloc/)