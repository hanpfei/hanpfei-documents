---
title: Android 图形系统之gralloc
date: 2017-09-21 13:05:49
categories: Android 图形系统
tags:
- Android开发
- 图形图像
---

# Gralloc1::Loader 与 gralloc 模块加载
`Gralloc1::Loader` 用于加载 HAL gralloc 模块。其类定义（位于 `frameworks/native/include/ui/Gralloc1.h`）如下：
<!--more-->
```
class Loader
{
public:
    Loader();
    ~Loader();

    std::unique_ptr<Device> getDevice();

private:
    static std::unique_ptr<Gralloc1On0Adapter> mAdapter;
    std::unique_ptr<Device> mDevice;
};
```

这个类只包含构造函数，析够函数和一个 Get 函数用于获得 `Gralloc1::Device`。该类全部方法实现（位于`frameworks/native/libs/ui/Gralloc1.cpp`）如下：
```
Loader::Loader()
  : mDevice(nullptr)
{
    hw_module_t const* module;
    int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    uint8_t majorVersion = (module->module_api_version >> 8) & 0xFF;
    uint8_t minorVersion = module->module_api_version & 0xFF;
    gralloc1_device_t* device = nullptr;
    if (majorVersion == 1) {
        gralloc1_open(module, &device);
    } else {
        if (!mAdapter) {
            mAdapter = std::make_unique<Gralloc1On0Adapter>(module);
        }
        device = mAdapter->getDevice();
    }
    mDevice = std::make_unique<Gralloc1::Device>(device);
}

Loader::~Loader() {}

std::unique_ptr<Device> Loader::getDevice()
{
    return std::move(mDevice);
}
```

`Gralloc1::Loader` 在其构造函数中通过 `hw_get_module()` 函数完成 HAL gralloc 模块的加载，随后创建一个 `Gralloc1::Device` 。当 gralloc API 版本为 1 时，通过 `frameworks/native/include/ui/Gralloc1.h` 中定义的 `gralloc1_open()` 函数创建：
```
static inline int gralloc1_open(const struct hw_module_t* module,
        gralloc1_device_t** device) {
    return module->methods->open(module, GRALLOC_HARDWARE_MODULE_ID,
            (struct hw_device_t**) device);
}
```

当 gralloc API 版本不为 1 时，则创建 `Gralloc1On0Adapter` 作为 `Gralloc1::Device`，相关函数（位于`frameworks/native/include/ui/Gralloc1On0Adapter.h`）如下：
```
class Gralloc1On0Adapter : public gralloc1_device_t
{
public:
    Gralloc1On0Adapter(const hw_module_t* module);
    ~Gralloc1On0Adapter();

    gralloc1_device_t* getDevice() {
        return static_cast<gralloc1_device_t*>(this);
    }
```

`Gralloc1::Loader` 的构造函数为 `hw_get_module()` 传入模块 ID 和 `hw_module_t` 指针 `module` 加载 gralloc 模块，其中 `module` 用于接收加载的结果。gralloc 的模块 ID 定义（位于 `hardware/libhardware/include/hardware/gralloc1.h`）如下：
```
#define GRALLOC_HARDWARE_MODULE_ID "gralloc"
```

`hw_get_module()` 函数用于加载 HAL gralloc 模块，其定义（位于 `hardware/libhardware/hardware.c`）如下：
```
static const char *variant_keys[] = {
    "ro.hardware",  /* This goes first so that it can pick up a different
                       file on the emulator. */
    "ro.product.board",
    "ro.board.platform",
    "ro.arch"
};

static const int HAL_VARIANT_KEYS_COUNT =
    (sizeof(variant_keys)/sizeof(variant_keys[0]));
. . . . . . 

int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module)
{
    int i = 0;
    char prop[PATH_MAX] = {0};
    char path[PATH_MAX] = {0};
    char name[PATH_MAX] = {0};
    char prop_name[PATH_MAX] = {0};


    if (inst)
        snprintf(name, PATH_MAX, "%s.%s", class_id, inst);
    else
        strlcpy(name, class_id, PATH_MAX);

    /*
     * Here we rely on the fact that calling dlopen multiple times on
     * the same .so will simply increment a refcount (and not load
     * a new copy of the library).
     * We also assume that dlopen() is thread-safe.
     */

    /* First try a property specific to the class and possibly instance */
    snprintf(prop_name, sizeof(prop_name), "ro.hardware.%s", name);
    if (property_get(prop_name, prop, NULL) > 0) {
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Loop through the configuration variants looking for a module */
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT; i++) {
        if (property_get(variant_keys[i], prop, NULL) == 0) {
            continue;
        }
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Nothing found, try the default */
    if (hw_module_exists(path, sizeof(path), name, "default") == 0) {
        goto found;
    }

    return -ENOENT;

found:
    /* load the module, if this fails, we're doomed, and we should not try
     * to load a different variant. */
    return load(class_id, path, module);
}

int hw_get_module(const char *id, const struct hw_module_t **module)
{
    return hw_get_module_by_class(id, NULL, module);
}
```

`hw_get_module()` 调用 `hw_get_module_by_class()` 实现其功能。`hw_get_module_by_class()` 通过两个步骤完成 gralloc 模块的加载：
1. 找到 gralloc 模块文件的路径。
2. 加载 gralloc 模块文件。

`hw_get_module_by_class()` 首先通过模块 ID 和模块实例名称得到模块名称，对于 gralloc 而言，模块 ID为 `gralloc`，模块实例名称为空，然后获得模块子名称。模块子名称来自于系统属性。`hw_get_module_by_class()` 依次尝试从如下几个系统属性中获得模块子名称：
```
ro.hardware.gralloc
ro.hardware
ro.product.board
ro.board.platform
ro.arch
```

对于 Pixel 设备而言，系统属性 `ro.hardware.gralloc` 和 `ro.arch` 未定义，其它几个系统属性值如下：
```
[ro.hardware]: [sailfish]
[ro.product.board]: [sailfish]
[ro.board.platform]: [msm8996]
```

无法由从系统属性获得的模块子名称找到所对应的模块文件时，则以 `default` 作为模块子名称继续查找。

有了模块名称和模块子名称，`hw_get_module_by_class()` 则依次在 /odm -> /vendor -> /system 等目录下查找相应的模块文件：
```
#if defined(__LP64__)
#define HAL_LIBRARY_PATH1 "/system/lib64/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib64/hw"
#define HAL_LIBRARY_PATH3 "/odm/lib64/hw"
#else
#define HAL_LIBRARY_PATH1 "/system/lib/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"
#define HAL_LIBRARY_PATH3 "/odm/lib/hw"
#endif
. . . . . .
static int hw_module_exists(char *path, size_t path_len, const char *name,
                            const char *subname)
{
    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH3, name, subname);
    if (access(path, R_OK) == 0)
        return 0;

    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH2, name, subname);
    if (access(path, R_OK) == 0)
        return 0;

    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH1, name, subname);
    if (access(path, R_OK) == 0)
        return 0;

    return -ENOENT;
}
```

对于 Pixel 设备，可以在 `/system/lib64/hw/` 目录下找到如下的 gralloc 模块文件：
```
1|sailfish:/ # ls /vendor/lib64/hw/ | grep gralloc                           
gralloc.default.so
gralloc.msm8996.so
```

最终将采用 `/vendor/lib64/hw/gralloc.msm8996.so` 作为 gralloc 模块文件。关于这一点，可以从 surfacefinger 进程的内存映射镜像中看到：
```
7105377000-710537d000 r-xp 00000000 fd:00 1575                           /system/lib64/hw/gralloc.msm8996.so
710537d000-710537e000 r--p 00005000 fd:00 1575                           /system/lib64/hw/gralloc.msm8996.so
710537e000-710537f000 rw-p 00006000 fd:00 1575                           /system/lib64/hw/gralloc.msm8996.so
```

`hw_get_module_by_class()` 通过 `load()` 函数来加载模块文件：
```
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    int status = -EINVAL;
    void *handle = NULL;
    struct hw_module_t *hmi = NULL;

    /*
     * load the symbols resolving undefined symbols before
     * dlopen returns. Since RTLD_GLOBAL is not or'd in with
     * RTLD_NOW the external symbols will not be global
     */
    handle = dlopen(path, RTLD_NOW);
    if (handle == NULL) {
        char const *err_str = dlerror();
        ALOGE("load: module=%s\n%s", path, err_str?err_str:"unknown");
        status = -EINVAL;
        goto done;
    }

    /* Get the address of the struct hal_module_info. */
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym);
    if (hmi == NULL) {
        ALOGE("load: couldn't find symbol %s", sym);
        status = -EINVAL;
        goto done;
    }

    /* Check that the id matches */
    if (strcmp(id, hmi->id) != 0) {
        ALOGE("load: id=%s != hmi->id=%s", id, hmi->id);
        status = -EINVAL;
        goto done;
    }

    hmi->dso = handle;

    /* success */
    status = 0;

    done:
    if (status != 0) {
        hmi = NULL;
        if (handle != NULL) {
            dlclose(handle);
            handle = NULL;
        }
    } else {
        ALOGV("loaded HAL id=%s path=%s hmi=%p handle=%p",
                id, path, *pHmi, handle);
    }

    *pHmi = hmi;

    return status;
}
```

加载过程主要是加载动态链接库文件，找到其中名为 `HAL_MODULE_INFO_SYM_AS_STR`，类型为 `struct hw_module_t` 的变量，并返回给调用者。

HAL 层要求 HAL 模块中都需要定义一个名为 HAL_MODULE_INFO_SYM_AS_STR 的 `struct hw_module_t` 变量。HAL_MODULE_INFO_SYM_AS_STR 定义（位于 `hardware/libhardware/include/hardware/hardware.h`）如下：
```
/**
 * Name of the hal_module_info
 */
#define HAL_MODULE_INFO_SYM         HMI

/**
 * Name of the hal_module_info as a string
 */
#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"
```

前面看到的 gralloc 实现 `gralloc.msm8996.so` 位于（ gralloc 接口实现） `hardware/qcom/display/msm8996/libgralloc`，或位于 `hardware/qcom/display/msm8996/libgralloc1`（ gralloc1 接口实现）。

`gralloc.default.so` 实现位于 `hardware/libhardware/modules/gralloc`，其 `struct hw_module_t` 定义（位于 `hardware/libhardware/modules/gralloc/gralloc.cpp`）如下：
```
struct private_module_t HAL_MODULE_INFO_SYM = {
    .base = {
        .common = {
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = GRALLOC_HARDWARE_MODULE_ID,
            .name = "Graphics Memory Allocator Module",
            .author = "The Android Open Source Project",
            .methods = &gralloc_module_methods
        },
        .registerBuffer = gralloc_register_buffer,
        .unregisterBuffer = gralloc_unregister_buffer,
        .lock = gralloc_lock,
        .unlock = gralloc_unlock,
    },
    .framebuffer = 0,
    .flags = 0,
    .numBuffers = 0,
    .bufferMask = 0,
    .lock = PTHREAD_MUTEX_INITIALIZER,
    .currentBuffer = 0,
};
```

不同类型的 HAL 模块，不同类型 HAL 模块的具体实现通常会定义自己的 `struct hw_module_t` 结构，如 `gralloc.default.so` 的 `struct private_module_t`，其定义（位于 `hardware/libhardware/modules/gralloc/gralloc_priv.h`）如下：
```
struct private_module_t {
    gralloc_module_t base;

    private_handle_t* framebuffer;
    uint32_t flags;
    uint32_t numBuffers;
    uint32_t bufferMask;
    pthread_mutex_t lock;
    buffer_handle_t currentBuffer;
    int pmem_master;
    void* pmem_master_base;

    struct fb_var_screeninfo info;
    struct fb_fix_screeninfo finfo;
    float xdpi;
    float ydpi;
    float fps;
};
```

`struct private_module_t` 的成员 `base` 类型为 `gralloc_module_t`，该类型定义如下：
```
typedef struct gralloc_module_t {
    struct hw_module_t common;
    
    int (*registerBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);

    int (*unregisterBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    
    int (*lock)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr);

    int (*unlock)(struct gralloc_module_t const* module,
            buffer_handle_t handle);

    int (*perform)(struct gralloc_module_t const* module,
            int operation, ... );

    int (*lock_ycbcr)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            struct android_ycbcr *ycbcr);

    int (*lockAsync)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr, int fenceFd);

    int (*unlockAsync)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int* fenceFd);

    int (*lockAsync_ycbcr)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            struct android_ycbcr *ycbcr, int fenceFd);

    /* reserved for future use */
    void* reserved_proc[3];
} gralloc_module_t;
```

`gralloc_module_t` 类型的第一个成员 `common` 类型为 `struct hw_module_t`。这实际是 C 语言对于继承的实现 —— 父结构体总是作为子结构体的第一个成员存在。对于 HAL gralloc 模块，其 `struct hw_module_t` 将总是 `struct gralloc_module_t` 结构的子结构，各个 gralloc 模块实现，通常又都会定义自己的私有 `struct gralloc_module_t` 结构。

按照接口约定，实现了 gralloc1 接口的 HAL gralloc 模块，其 `module->methods->open()` 函数应该通过参数 `device` 返回一个类型为 `gralloc1_device_t` 的结构，一个 `struct hw_device_t` 结构的子结构；而实现了 gralloc 接口的 HAL gralloc 模块，则应该通过相同的方法，返回 `struct hw_device_t` 结构的子结构 `struct alloc_device_t`。

`gralloc1_device_t` 定义（位于 `hardware/libhardware/include/hardware/gralloc1.h`）如下：
```
typedef struct gralloc1_device {
    /* Must be the first member of this struct, since a pointer to this struct
     * will be generated by casting from a hw_device_t* */
    struct hw_device_t common;

    void (*getCapabilities)(struct gralloc1_device* device, uint32_t* outCount,
            int32_t* /*gralloc1_capability_t*/ outCapabilities);

    gralloc1_function_pointer_t (*getFunction)(struct gralloc1_device* device,
            int32_t /*gralloc1_function_descriptor_t*/ descriptor);
} gralloc1_device_t;
```

`gralloc1_device_t` 新定义了两个函数指针成员，它们也是 gralloc1 的实现模块必须要提供的函数。其中 `getCapabilities` 用于获得设备支持的 capabilities 的列表，`getFunction` 则用于获得实现不同功能的函数的指针。

`struct alloc_device_t` 接口定义如下：
```
typedef struct alloc_device_t {
    struct hw_device_t common;

    int (*alloc)(struct alloc_device_t* dev,
            int w, int h, int format, int usage,
            buffer_handle_t* handle, int* stride);

    int (*free)(struct alloc_device_t* dev,
            buffer_handle_t handle);

    void (*dump)(struct alloc_device_t *dev, char *buff, int buff_len);

    void* reserved_proc[7];
} alloc_device_t;
```

`struct alloc_device_t` 新定义了用于分配和释放图形缓冲区的接口。

`Gralloc1::Loader` 的构造函数中，当 `majorVersion` 不为 1 时，将创建 `Gralloc1On0Adapter` 把 gralloc 接口下的 `struct alloc_device_t` 转为 gralloc1 接口下的 `gralloc1_device_t`。

前面看到的 `gralloc.default.so`，它实现了 gralloc 接口，而不是 gralloc1。

#  Gralloc1::Device
`Gralloc1::Device` 是对 `gralloc1_device_t` 的一个包装。`Gralloc1::Device` 定义（位于 `frameworks/native/include/ui/Gralloc1.h`）如下：
```
class Device {
    friend class Gralloc1::Descriptor;

public:
    Device(gralloc1_device_t* device);

    bool hasCapability(gralloc1_capability_t capability) const;

    std::string dump();

    std::shared_ptr<Descriptor> createDescriptor();

    gralloc1_error_t getStride(buffer_handle_t buffer, uint32_t* outStride);

    gralloc1_error_t allocate(
            const std::vector<std::shared_ptr<const Descriptor>>& descriptors,
            std::vector<buffer_handle_t>* outBuffers);
    gralloc1_error_t allocate(
            const std::shared_ptr<const Descriptor>& descriptor,
            gralloc1_backing_store_t id, buffer_handle_t* outBuffer);

    gralloc1_error_t retain(buffer_handle_t buffer);
    gralloc1_error_t retain(const GraphicBuffer* buffer);

    gralloc1_error_t release(buffer_handle_t buffer);

    gralloc1_error_t getNumFlexPlanes(buffer_handle_t buffer,
            uint32_t* outNumPlanes);

    gralloc1_error_t lock(buffer_handle_t buffer,
            gralloc1_producer_usage_t producerUsage,
            gralloc1_consumer_usage_t consumerUsage,
            const gralloc1_rect_t* accessRegion, void** outData,
            const sp<Fence>& acquireFence);
    gralloc1_error_t lockFlex(buffer_handle_t buffer,
            gralloc1_producer_usage_t producerUsage,
            gralloc1_consumer_usage_t consumerUsage,
            const gralloc1_rect_t* accessRegion,
            struct android_flex_layout* outData, const sp<Fence>& acquireFence);
    gralloc1_error_t lockYCbCr(buffer_handle_t buffer,
            gralloc1_producer_usage_t producerUsage,
            gralloc1_consumer_usage_t consumerUsage,
            const gralloc1_rect_t* accessRegion, struct android_ycbcr* outData,
            const sp<Fence>& acquireFence);

    gralloc1_error_t unlock(buffer_handle_t buffer, sp<Fence>* outFence);

private:
    std::unordered_set<gralloc1_capability_t> loadCapabilities();

    bool loadFunctions();

    template <typename LockType, typename OutType>
    gralloc1_error_t lockHelper(LockType pfn, buffer_handle_t buffer,
            gralloc1_producer_usage_t producerUsage,
            gralloc1_consumer_usage_t consumerUsage,
            const gralloc1_rect_t* accessRegion, OutType* outData,
            const sp<Fence>& acquireFence) {
        int32_t intError = pfn(mDevice, buffer,
                static_cast<uint64_t>(producerUsage),
                static_cast<uint64_t>(consumerUsage), accessRegion, outData,
                acquireFence->dup());
        return static_cast<gralloc1_error_t>(intError);
    }

    gralloc1_device_t* const mDevice;

    const std::unordered_set<gralloc1_capability_t> mCapabilities;

    template <typename PFN, gralloc1_function_descriptor_t descriptor>
    struct FunctionLoader {
        FunctionLoader() : pfn(nullptr) {}

        bool load(gralloc1_device_t* device, bool errorIfNull) {
            gralloc1_function_pointer_t rawPointer =
                    device->getFunction(device, descriptor);
            pfn = reinterpret_cast<PFN>(rawPointer);
            if (errorIfNull && !rawPointer) {
                ALOG(LOG_ERROR, GRALLOC1_LOG_TAG,
                        "Failed to load function pointer %d", descriptor);
            }
            return rawPointer != nullptr;
        }

        template <typename ...Args>
        typename std::result_of<PFN(Args...)>::type operator()(Args... args) {
            return pfn(args...);
        }

        PFN pfn;
    };

    // Function pointers
    struct Functions {
        FunctionLoader<GRALLOC1_PFN_DUMP, GRALLOC1_FUNCTION_DUMP> dump;
        FunctionLoader<GRALLOC1_PFN_CREATE_DESCRIPTOR,
                GRALLOC1_FUNCTION_CREATE_DESCRIPTOR> createDescriptor;
        FunctionLoader<GRALLOC1_PFN_DESTROY_DESCRIPTOR,
                GRALLOC1_FUNCTION_DESTROY_DESCRIPTOR> destroyDescriptor;
        FunctionLoader<GRALLOC1_PFN_SET_CONSUMER_USAGE,
                GRALLOC1_FUNCTION_SET_CONSUMER_USAGE> setConsumerUsage;
        FunctionLoader<GRALLOC1_PFN_SET_DIMENSIONS,
                GRALLOC1_FUNCTION_SET_DIMENSIONS> setDimensions;
        FunctionLoader<GRALLOC1_PFN_SET_FORMAT,
                GRALLOC1_FUNCTION_SET_FORMAT> setFormat;
        FunctionLoader<GRALLOC1_PFN_SET_PRODUCER_USAGE,
                GRALLOC1_FUNCTION_SET_PRODUCER_USAGE> setProducerUsage;
        FunctionLoader<GRALLOC1_PFN_GET_BACKING_STORE,
                GRALLOC1_FUNCTION_GET_BACKING_STORE> getBackingStore;
        FunctionLoader<GRALLOC1_PFN_GET_CONSUMER_USAGE,
                GRALLOC1_FUNCTION_GET_CONSUMER_USAGE> getConsumerUsage;
        FunctionLoader<GRALLOC1_PFN_GET_DIMENSIONS,
                GRALLOC1_FUNCTION_GET_DIMENSIONS> getDimensions;
        FunctionLoader<GRALLOC1_PFN_GET_FORMAT,
                GRALLOC1_FUNCTION_GET_FORMAT> getFormat;
        FunctionLoader<GRALLOC1_PFN_GET_PRODUCER_USAGE,
                GRALLOC1_FUNCTION_GET_PRODUCER_USAGE> getProducerUsage;
        FunctionLoader<GRALLOC1_PFN_GET_STRIDE,
                GRALLOC1_FUNCTION_GET_STRIDE> getStride;
        FunctionLoader<GRALLOC1_PFN_ALLOCATE,
                GRALLOC1_FUNCTION_ALLOCATE> allocate;
        FunctionLoader<GRALLOC1_PFN_RETAIN,
                GRALLOC1_FUNCTION_RETAIN> retain;
        FunctionLoader<GRALLOC1_PFN_RELEASE,
                GRALLOC1_FUNCTION_RELEASE> release;
        FunctionLoader<GRALLOC1_PFN_GET_NUM_FLEX_PLANES,
                GRALLOC1_FUNCTION_GET_NUM_FLEX_PLANES> getNumFlexPlanes;
        FunctionLoader<GRALLOC1_PFN_LOCK,
                GRALLOC1_FUNCTION_LOCK> lock;
        FunctionLoader<GRALLOC1_PFN_LOCK_FLEX,
                GRALLOC1_FUNCTION_LOCK_FLEX> lockFlex;
        FunctionLoader<GRALLOC1_PFN_LOCK_YCBCR,
                GRALLOC1_FUNCTION_LOCK_YCBCR> lockYCbCr;
        FunctionLoader<GRALLOC1_PFN_UNLOCK,
                GRALLOC1_FUNCTION_UNLOCK> unlock;

        // Adapter-only functions
        FunctionLoader<GRALLOC1_PFN_RETAIN_GRAPHIC_BUFFER,
                GRALLOC1_FUNCTION_RETAIN_GRAPHIC_BUFFER> retainGraphicBuffer;
        FunctionLoader<GRALLOC1_PFN_ALLOCATE_WITH_ID,
                GRALLOC1_FUNCTION_ALLOCATE_WITH_ID> allocateWithId;
    } mFunctions;

}; // class android::Gralloc1::Device
```

`Gralloc1::Device` 定义了接口以提供图形内存的分配/释放、lock/unlock 操作，所有的这些操作都依赖于从 `gralloc1_device_t` 获得的函数指针实现。在 
`Gralloc1::Device` 构造期间，会加载相关的函数指针，以及设备的 capabilities：
```
Device::Device(gralloc1_device_t* device)
  : mDevice(device),
    mCapabilities(loadCapabilities()),
    mFunctions()
{
    if (!loadFunctions()) {
        ALOGE("Failed to load a required function, aborting");
        abort();
    }
}
. . . . . . 
std::unordered_set<gralloc1_capability_t> Device::loadCapabilities()
{
    std::vector<int32_t> intCapabilities;
    uint32_t numCapabilities = 0;
    mDevice->getCapabilities(mDevice, &numCapabilities, nullptr);

    intCapabilities.resize(numCapabilities);
    mDevice->getCapabilities(mDevice, &numCapabilities, intCapabilities.data());

    std::unordered_set<gralloc1_capability_t> capabilities;
    for (const auto intCapability : intCapabilities) {
        capabilities.emplace(static_cast<gralloc1_capability_t>(intCapability));
    }
    return capabilities;
}

bool Device::loadFunctions()
{
    // Functions which must always be present
    if (!mFunctions.dump.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.createDescriptor.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.destroyDescriptor.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.setConsumerUsage.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.setDimensions.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.setFormat.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.setProducerUsage.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getBackingStore.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getConsumerUsage.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getDimensions.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getFormat.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getProducerUsage.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getStride.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.retain.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.release.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.getNumFlexPlanes.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.lock.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.lockFlex.load(mDevice, true)) {
        return false;
    }
    if (!mFunctions.unlock.load(mDevice, true)) {
        return false;
    }

    if (hasCapability(GRALLOC1_CAPABILITY_ON_ADAPTER)) {
        // These should always be present on the adapter
        if (!mFunctions.retainGraphicBuffer.load(mDevice, true)) {
            return false;
        }
        if (!mFunctions.lockYCbCr.load(mDevice, true)) {
            return false;
        }

        // allocateWithId may not be present if we're only able to map in this
        // process
        mFunctions.allocateWithId.load(mDevice, false);
    } else {
        // allocate may not be present if we're only able to map in this process
        mFunctions.allocate.load(mDevice, false);
    }

    return true;
}
```

`Gralloc1::Device` 中定义的 `FunctionLoader` 模板，即可以加载函数指针，同时它实现了 `operator()`，也是一个函数对象：
```
    template <typename PFN, gralloc1_function_descriptor_t descriptor>
    struct FunctionLoader {
        FunctionLoader() : pfn(nullptr) {}

        bool load(gralloc1_device_t* device, bool errorIfNull) {
            gralloc1_function_pointer_t rawPointer =
                    device->getFunction(device, descriptor);
            pfn = reinterpret_cast<PFN>(rawPointer);
            if (errorIfNull && !rawPointer) {
                ALOG(LOG_ERROR, GRALLOC1_LOG_TAG,
                        "Failed to load function pointer %d", descriptor);
            }
            return rawPointer != nullptr;
        }

        template <typename ...Args>
        typename std::result_of<PFN(Args...)>::type operator()(Args... args) {
            return pfn(args...);
        }

        PFN pfn;
    };
```

`FunctionLoader` 模板藉由 `gralloc1_device_t` 的 `getFunction()` 接口获得函数指针。`Gralloc1::Device` 的函数函数类型定义像下面这样：
```
typedef void (*GRALLOC1_PFN_DUMP)(gralloc1_device_t* device, uint32_t* outSize,
        char* outBuffer);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_CREATE_DESCRIPTOR)(
        gralloc1_device_t* device, gralloc1_buffer_descriptor_t* outDescriptor);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_DESTROY_DESCRIPTOR)(
        gralloc1_device_t* device, gralloc1_buffer_descriptor_t descriptor);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_SET_CONSUMER_USAGE)(
        gralloc1_device_t* device, gralloc1_buffer_descriptor_t descriptor,
        uint64_t /*gralloc1_consumer_usage_t*/ usage);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_SET_DIMENSIONS)(
        gralloc1_device_t* device, gralloc1_buffer_descriptor_t descriptor,
        uint32_t width, uint32_t height);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_SET_FORMAT)(
        gralloc1_device_t* device, gralloc1_buffer_descriptor_t descriptor,
        int32_t /*android_pixel_format_t*/ format);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_SET_PRODUCER_USAGE)(
        gralloc1_device_t* device, gralloc1_buffer_descriptor_t descriptor,
        uint64_t /*gralloc1_producer_usage_t*/ usage);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_GET_BACKING_STORE)(
        gralloc1_device_t* device, buffer_handle_t buffer,
        gralloc1_backing_store_t* outStore);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_GET_CONSUMER_USAGE)(
        gralloc1_device_t* device, buffer_handle_t buffer,
        uint64_t* /*gralloc1_consumer_usage_t*/ outUsage);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_GET_DIMENSIONS)(
        gralloc1_device_t* device, buffer_handle_t buffer, uint32_t* outWidth,
        uint32_t* outHeight);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_GET_FORMAT)(
        gralloc1_device_t* device, buffer_handle_t descriptor,
        int32_t* outFormat);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_GET_PRODUCER_USAGE)(
        gralloc1_device_t* device, buffer_handle_t buffer,
        uint64_t* /*gralloc1_producer_usage_t*/ outUsage);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_GET_STRIDE)(
        gralloc1_device_t* device, buffer_handle_t buffer, uint32_t* outStride);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_ALLOCATE)(
        gralloc1_device_t* device, uint32_t numDescriptors,
        const gralloc1_buffer_descriptor_t* descriptors,
        buffer_handle_t* outBuffers);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_RETAIN)(
        gralloc1_device_t* device, buffer_handle_t buffer);
typedef int32_t /*gralloc1_error_t*/ (*GRALLOC1_PFN_RELEASE)(
        gralloc1_device_t* device, buffer_handle_t buffer);
```

而所谓的函数 descriptor 则是用于标识函数的整数：
```
typedef enum {
    GRALLOC1_FUNCTION_INVALID = 0,
    GRALLOC1_FUNCTION_DUMP = 1,
    GRALLOC1_FUNCTION_CREATE_DESCRIPTOR = 2,
    GRALLOC1_FUNCTION_DESTROY_DESCRIPTOR = 3,
    GRALLOC1_FUNCTION_SET_CONSUMER_USAGE = 4,
    GRALLOC1_FUNCTION_SET_DIMENSIONS = 5,
    GRALLOC1_FUNCTION_SET_FORMAT = 6,
    GRALLOC1_FUNCTION_SET_PRODUCER_USAGE = 7,
    GRALLOC1_FUNCTION_GET_BACKING_STORE = 8,
    GRALLOC1_FUNCTION_GET_CONSUMER_USAGE = 9,
    GRALLOC1_FUNCTION_GET_DIMENSIONS = 10,
    GRALLOC1_FUNCTION_GET_FORMAT = 11,
    GRALLOC1_FUNCTION_GET_PRODUCER_USAGE = 12,
    GRALLOC1_FUNCTION_GET_STRIDE = 13,
    GRALLOC1_FUNCTION_ALLOCATE = 14,
    GRALLOC1_FUNCTION_RETAIN = 15,
    GRALLOC1_FUNCTION_RELEASE = 16,
    GRALLOC1_FUNCTION_GET_NUM_FLEX_PLANES = 17,
    GRALLOC1_FUNCTION_LOCK = 18,
    GRALLOC1_FUNCTION_LOCK_FLEX = 19,
    GRALLOC1_FUNCTION_UNLOCK = 20,
    GRALLOC1_LAST_FUNCTION = 20,
} gralloc1_function_descriptor_t;
```

`Gralloc1::Device` 的所有功能实现，都依赖于从 `gralloc1_device_t` 的函数指针：
```
std::string Device::dump()
{
    uint32_t length = 0;
    mFunctions.dump(mDevice, &length, nullptr);

    std::vector<char> output;
    output.resize(length);
    mFunctions.dump(mDevice, &length, output.data());

    return std::string(output.cbegin(), output.cend());
}

std::shared_ptr<Descriptor> Device::createDescriptor()
{
    gralloc1_buffer_descriptor_t descriptorId;
    int32_t intError = mFunctions.createDescriptor(mDevice, &descriptorId);
    auto error = static_cast<gralloc1_error_t>(intError);
    if (error != GRALLOC1_ERROR_NONE) {
        return nullptr;
    }
    auto descriptor = std::make_shared<Descriptor>(*this, descriptorId);
    return descriptor;
}

gralloc1_error_t Device::getStride(buffer_handle_t buffer, uint32_t* outStride)
{
    int32_t intError = mFunctions.getStride(mDevice, buffer, outStride);
    return static_cast<gralloc1_error_t>(intError);
}

static inline bool allocationSucceded(gralloc1_error_t error)
{
    return error == GRALLOC1_ERROR_NONE || error == GRALLOC1_ERROR_NOT_SHARED;
}

gralloc1_error_t Device::allocate(
        const std::vector<std::shared_ptr<const Descriptor>>& descriptors,
        std::vector<buffer_handle_t>* outBuffers)
{
    if (mFunctions.allocate.pfn == nullptr) {
        // Allocation is not supported on this device
        return GRALLOC1_ERROR_UNSUPPORTED;
    }

    std::vector<gralloc1_buffer_descriptor_t> deviceIds;
    for (const auto& descriptor : descriptors) {
        deviceIds.emplace_back(descriptor->getDeviceId());
    }

    std::vector<buffer_handle_t> buffers(descriptors.size());
    int32_t intError = mFunctions.allocate(mDevice,
            static_cast<uint32_t>(descriptors.size()), deviceIds.data(),
            buffers.data());
    auto error = static_cast<gralloc1_error_t>(intError);
    if (allocationSucceded(error)) {
        *outBuffers = std::move(buffers);
    }

    return error;
}

gralloc1_error_t Device::allocate(
        const std::shared_ptr<const Descriptor>& descriptor,
        gralloc1_backing_store_t id, buffer_handle_t* outBuffer)
{
    gralloc1_error_t error = GRALLOC1_ERROR_NONE;

    if (hasCapability(GRALLOC1_CAPABILITY_ON_ADAPTER)) {
        buffer_handle_t buffer = nullptr;
        int32_t intError = mFunctions.allocateWithId(mDevice,
                descriptor->getDeviceId(), id, &buffer);
        error = static_cast<gralloc1_error_t>(intError);
        if (allocationSucceded(error)) {
            *outBuffer = buffer;
        }
    } else {
        std::vector<std::shared_ptr<const Descriptor>> descriptors;
        descriptors.emplace_back(descriptor);
        std::vector<buffer_handle_t> buffers;
        error = allocate(descriptors, &buffers);
        if (allocationSucceded(error)) {
            *outBuffer = buffers[0];
        }
    }

    return error;
}

gralloc1_error_t Device::retain(buffer_handle_t buffer)
{
    int32_t intError = mFunctions.retain(mDevice, buffer);
    return static_cast<gralloc1_error_t>(intError);
}

gralloc1_error_t Device::retain(const GraphicBuffer* buffer)
{
    if (hasCapability(GRALLOC1_CAPABILITY_ON_ADAPTER)) {
        return mFunctions.retainGraphicBuffer(mDevice, buffer);
    } else {
        return retain(buffer->getNativeBuffer()->handle);
    }
}

gralloc1_error_t Device::release(buffer_handle_t buffer)
{
    int32_t intError = mFunctions.release(mDevice, buffer);
    return static_cast<gralloc1_error_t>(intError);
}

gralloc1_error_t Device::getNumFlexPlanes(buffer_handle_t buffer,
        uint32_t* outNumPlanes)
{
    uint32_t numPlanes = 0;
    int32_t intError = mFunctions.getNumFlexPlanes(mDevice, buffer, &numPlanes);
    auto error = static_cast<gralloc1_error_t>(intError);
    if (error == GRALLOC1_ERROR_NONE) {
        *outNumPlanes = numPlanes;
    }
    return error;
}

gralloc1_error_t Device::lock(buffer_handle_t buffer,
        gralloc1_producer_usage_t producerUsage,
        gralloc1_consumer_usage_t consumerUsage,
        const gralloc1_rect_t* accessRegion, void** outData,
        const sp<Fence>& acquireFence)
{
    ALOGV("Calling lock(%p)", buffer);
    return lockHelper(mFunctions.lock.pfn, buffer, producerUsage,
            consumerUsage, accessRegion, outData, acquireFence);
}

gralloc1_error_t Device::lockFlex(buffer_handle_t buffer,
        gralloc1_producer_usage_t producerUsage,
        gralloc1_consumer_usage_t consumerUsage,
        const gralloc1_rect_t* accessRegion,
        struct android_flex_layout* outData,
        const sp<Fence>& acquireFence)
{
    ALOGV("Calling lockFlex(%p)", buffer);
    return lockHelper(mFunctions.lockFlex.pfn, buffer, producerUsage,
            consumerUsage, accessRegion, outData, acquireFence);
}

gralloc1_error_t Device::lockYCbCr(buffer_handle_t buffer,
        gralloc1_producer_usage_t producerUsage,
        gralloc1_consumer_usage_t consumerUsage,
        const gralloc1_rect_t* accessRegion,
        struct android_ycbcr* outData,
        const sp<Fence>& acquireFence)
{
    ALOGV("Calling lockYCbCr(%p)", buffer);
    return lockHelper(mFunctions.lockYCbCr.pfn, buffer, producerUsage,
            consumerUsage, accessRegion, outData, acquireFence);
}

gralloc1_error_t Device::unlock(buffer_handle_t buffer, sp<Fence>* outFence)
{
    int32_t fenceFd = -1;
    int32_t intError = mFunctions.unlock(mDevice, buffer, &fenceFd);
    auto error = static_cast<gralloc1_error_t>(intError);
    if (error == GRALLOC1_ERROR_NONE) {
        *outFence = new Fence(fenceFd);
    }
    return error;
}
```

我们可以看一下 `Gralloc1On0Adapter`，来简单了解 HAL gralloc 模块，`gralloc1_device_t` 可能的实现和功能。`Gralloc1On0Adapter` 继承自 `gralloc1_device_t`：
```
class Gralloc1On0Adapter : public gralloc1_device_t
{
public:
    Gralloc1On0Adapter(const hw_module_t* module);
    ~Gralloc1On0Adapter();

    gralloc1_device_t* getDevice() {
        return static_cast<gralloc1_device_t*>(this);
    }
```

`Gralloc1On0Adapter` 需要提供 `gralloc1_device_t` 所必需的 `getCapabilities` 和 `getFunction` 这两个回调：
```
namespace android {

Gralloc1On0Adapter::Gralloc1On0Adapter(const hw_module_t* module)
  : mModule(reinterpret_cast<const gralloc_module_t*>(module)),
    mMinorVersion(mModule->common.module_api_version & 0xFF),
    mDevice(nullptr)
{
    ALOGV("Constructing");
    getCapabilities = getCapabilitiesHook;
    getFunction = getFunctionHook;
    int error = ::gralloc_open(&(mModule->common), &mDevice);
    if (error) {
        ALOGE("Failed to open gralloc0 module: %d", error);
    }
    ALOGV("Opened gralloc0 device %p", mDevice);
}
```

这两个回调的实现在定义类的头文件里：
```
private:
    static inline Gralloc1On0Adapter* getAdapter(gralloc1_device_t* device) {
        return static_cast<Gralloc1On0Adapter*>(device);
    }

    // getCapabilities

    void doGetCapabilities(uint32_t* outCount,
            int32_t* /*gralloc1_capability_t*/ outCapabilities);
    static void getCapabilitiesHook(gralloc1_device_t* device,
            uint32_t* outCount,
            int32_t* /*gralloc1_capability_t*/ outCapabilities) {
        getAdapter(device)->doGetCapabilities(outCount, outCapabilities);
    };

    // getFunction

    gralloc1_function_pointer_t doGetFunction(
            int32_t /*gralloc1_function_descriptor_t*/ descriptor);
    static gralloc1_function_pointer_t getFunctionHook(
            gralloc1_device_t* device,
            int32_t /*gralloc1_function_descriptor_t*/ descriptor) {
        return getAdapter(device)->doGetFunction(descriptor);
    }
```

两个回调最终通过 `doGetCapabilities()` 和 `doGetFunction()` 两个成员函数完成其功能，这两个函数实现如下：
```
void Gralloc1On0Adapter::doGetCapabilities(uint32_t* outCount,
        int32_t* outCapabilities)
{
    if (outCapabilities == nullptr) {
        *outCount = 1;
        return;
    }
    if (*outCount >= 1) {
        *outCapabilities = GRALLOC1_CAPABILITY_ON_ADAPTER;
        *outCount = 1;
    }
}

gralloc1_function_pointer_t Gralloc1On0Adapter::doGetFunction(
        int32_t intDescriptor)
{
    constexpr auto lastDescriptor =
            static_cast<int32_t>(GRALLOC1_LAST_ADAPTER_FUNCTION);
    if (intDescriptor < 0 || intDescriptor > lastDescriptor) {
        ALOGE("Invalid function descriptor");
        return nullptr;
    }

    auto descriptor =
            static_cast<gralloc1_function_descriptor_t>(intDescriptor);
    switch (descriptor) {
        case GRALLOC1_FUNCTION_DUMP:
            return asFP<GRALLOC1_PFN_DUMP>(dumpHook);
        case GRALLOC1_FUNCTION_CREATE_DESCRIPTOR:
            return asFP<GRALLOC1_PFN_CREATE_DESCRIPTOR>(createDescriptorHook);
        case GRALLOC1_FUNCTION_DESTROY_DESCRIPTOR:
            return asFP<GRALLOC1_PFN_DESTROY_DESCRIPTOR>(destroyDescriptorHook);
        case GRALLOC1_FUNCTION_SET_CONSUMER_USAGE:
            return asFP<GRALLOC1_PFN_SET_CONSUMER_USAGE>(setConsumerUsageHook);
        case GRALLOC1_FUNCTION_SET_DIMENSIONS:
            return asFP<GRALLOC1_PFN_SET_DIMENSIONS>(setDimensionsHook);
        case GRALLOC1_FUNCTION_SET_FORMAT:
            return asFP<GRALLOC1_PFN_SET_FORMAT>(setFormatHook);
        case GRALLOC1_FUNCTION_SET_PRODUCER_USAGE:
            return asFP<GRALLOC1_PFN_SET_PRODUCER_USAGE>(setProducerUsageHook);
        case GRALLOC1_FUNCTION_GET_BACKING_STORE:
            return asFP<GRALLOC1_PFN_GET_BACKING_STORE>(
                    bufferHook<decltype(&Buffer::getBackingStore),
                    &Buffer::getBackingStore, gralloc1_backing_store_t*>);
        case GRALLOC1_FUNCTION_GET_CONSUMER_USAGE:
            return asFP<GRALLOC1_PFN_GET_CONSUMER_USAGE>(getConsumerUsageHook);
        case GRALLOC1_FUNCTION_GET_DIMENSIONS:
            return asFP<GRALLOC1_PFN_GET_DIMENSIONS>(
                    bufferHook<decltype(&Buffer::getDimensions),
                    &Buffer::getDimensions, uint32_t*, uint32_t*>);
        case GRALLOC1_FUNCTION_GET_FORMAT:
            return asFP<GRALLOC1_PFN_GET_FORMAT>(
                    bufferHook<decltype(&Buffer::getFormat),
                    &Buffer::getFormat, int32_t*>);
        case GRALLOC1_FUNCTION_GET_PRODUCER_USAGE:
            return asFP<GRALLOC1_PFN_GET_PRODUCER_USAGE>(getProducerUsageHook);
        case GRALLOC1_FUNCTION_GET_STRIDE:
            return asFP<GRALLOC1_PFN_GET_STRIDE>(
                    bufferHook<decltype(&Buffer::getStride),
                    &Buffer::getStride, uint32_t*>);
        case GRALLOC1_FUNCTION_ALLOCATE:
            // Not provided, since we'll use ALLOCATE_WITH_ID
            return nullptr;
        case GRALLOC1_FUNCTION_ALLOCATE_WITH_ID:
            if (mDevice != nullptr) {
                return asFP<GRALLOC1_PFN_ALLOCATE_WITH_ID>(allocateWithIdHook);
            } else {
                return nullptr;
            }
        case GRALLOC1_FUNCTION_RETAIN:
            return asFP<GRALLOC1_PFN_RETAIN>(
                    managementHook<&Gralloc1On0Adapter::retain>);
        case GRALLOC1_FUNCTION_RELEASE:
            return asFP<GRALLOC1_PFN_RELEASE>(
                    managementHook<&Gralloc1On0Adapter::release>);
        case GRALLOC1_FUNCTION_RETAIN_GRAPHIC_BUFFER:
            return asFP<GRALLOC1_PFN_RETAIN_GRAPHIC_BUFFER>(
                    retainGraphicBufferHook);
        case GRALLOC1_FUNCTION_GET_NUM_FLEX_PLANES:
            return asFP<GRALLOC1_PFN_GET_NUM_FLEX_PLANES>(
                    bufferHook<decltype(&Buffer::getNumFlexPlanes),
                    &Buffer::getNumFlexPlanes, uint32_t*>);
        case GRALLOC1_FUNCTION_LOCK:
            return asFP<GRALLOC1_PFN_LOCK>(
                    lockHook<void*, &Gralloc1On0Adapter::lock>);
        case GRALLOC1_FUNCTION_LOCK_FLEX:
            return asFP<GRALLOC1_PFN_LOCK_FLEX>(
                    lockHook<struct android_flex_layout,
                    &Gralloc1On0Adapter::lockFlex>);
        case GRALLOC1_FUNCTION_LOCK_YCBCR:
            return asFP<GRALLOC1_PFN_LOCK_YCBCR>(
                    lockHook<struct android_ycbcr,
                    &Gralloc1On0Adapter::lockYCbCr>);
        case GRALLOC1_FUNCTION_UNLOCK:
            return asFP<GRALLOC1_PFN_UNLOCK>(unlockHook);
        case GRALLOC1_FUNCTION_INVALID:
            ALOGE("Invalid function descriptor");
            return nullptr;
    }

    ALOGE("Unknown function descriptor: %d", intDescriptor);
    return nullptr;
}
```

最终 `Gralloc1On0Adapter` 的这些功能函数有将依赖于 `alloc_device_t` 和 `gralloc_module_t` 提供的那些回调来完成其功能，如 `alloc()` 函数依赖于 `alloc_device_t`：
```
gralloc1_error_t Gralloc1On0Adapter::allocate(
        const std::shared_ptr<Descriptor>& descriptor,
        gralloc1_backing_store_t store,
        buffer_handle_t* outBufferHandle)
{
    ALOGV("allocate(%" PRIu64 ", %#" PRIx64 ")", descriptor->id, store);

    // If this function is being called, it's because we handed out its function
    // pointer, which only occurs when mDevice has been loaded successfully and
    // we are permitted to allocate

    int usage = static_cast<int>(descriptor->producerUsage) |
            static_cast<int>(descriptor->consumerUsage);
    buffer_handle_t handle = nullptr;
    int stride = 0;
    ALOGV("Calling alloc(%p, %u, %u, %i, %u)", mDevice, descriptor->width,
            descriptor->height, descriptor->format, usage);
    auto error = mDevice->alloc(mDevice,
            static_cast<int>(descriptor->width),
            static_cast<int>(descriptor->height), descriptor->format,
            usage, &handle, &stride);
    if (error != 0) {
        ALOGE("gralloc0 allocation failed: %d (%s)", error,
                strerror(-error));
        return GRALLOC1_ERROR_NO_RESOURCES;
    }

    *outBufferHandle = handle;
    auto buffer = std::make_shared<Buffer>(handle, store, *descriptor, stride,
            true);

    std::lock_guard<std::mutex> lock(mBufferMutex);
    mBuffers.emplace(handle, std::move(buffer));

    return GRALLOC1_ERROR_NONE;
}
```

lock/unlock 操作则都依赖于 `gralloc_module_t`，以 lock 为例：
```
gralloc1_error_t Gralloc1On0Adapter::lock(
        const std::shared_ptr<Buffer>& buffer,
        gralloc1_producer_usage_t producerUsage,
        gralloc1_consumer_usage_t consumerUsage,
        const gralloc1_rect_t& accessRegion, void** outData,
        const sp<Fence>& acquireFence)
{
    if (mMinorVersion >= 3) {
        int result = mModule->lockAsync(mModule, buffer->getHandle(),
                static_cast<int32_t>(producerUsage | consumerUsage),
                accessRegion.left, accessRegion.top, accessRegion.width,
                accessRegion.height, outData, acquireFence->dup());
        if (result != 0) {
            return GRALLOC1_ERROR_UNSUPPORTED;
        }
    } else {
        acquireFence->waitForever("Gralloc1On0Adapter::lock");
        int result = mModule->lock(mModule, buffer->getHandle(),
                static_cast<int32_t>(producerUsage | consumerUsage),
                accessRegion.left, accessRegion.top, accessRegion.width,
                accessRegion.height, outData);
        ALOGV("gralloc0 lock returned %d", result);
        if (result != 0) {
            return GRALLOC1_ERROR_UNSUPPORTED;
        }
    }
    return GRALLOC1_ERROR_NONE;
}
```

关于 HAL gralloc 模块更具体的实现，这里不再深追。

总结一下 Android 图形系统中，对于 gralloc 接口的情况，与图形缓冲区分配、映射有关的组件之间的结构，如下图所示：

![](../images/1315506-7182c5530b0cfa25.png)

如果是 gralloc1 接口的情况，则稍微有一点不同：

![](../images/1315506-54007139e9f15b0b.png)

此外，在前面 `BufferQueueCore` 中创建 `IGraphicBufferAlloc` 时，我们看到这个对象总是会通过 surfacefinger 创建，然后通过 Binder IPC 传递给创建 BufferQueue 的进程一个句柄。后续在创建 BufferQueue 的进程中的生产者要分配 `GraphicBuffer`，则这一分配过程实际也将发生在 surfaceflinger 进程中，创建的 `GraphicBuffer`经过序列化之后，传回给创建 BufferQueue 的进程。在 `GraphicBuffer` 对象真正创建的时候，也会直接为其分配图形缓冲区。也就是说，在 Android 系统中，真正会分配图形缓冲区的进程只有 surfaceflinger，尽管可能会创建 BufferQueue 的进程有多个。

`GraphicBufferAllocator` 和 `GraphicBufferMapper` 在对象创建的时候，都会通过 `Gralloc1::Loader` 加载 HAL gralloc。只有需要分配图形缓冲区的进程才需要创建 `GraphicBufferAllocator`，只有需要访问图形缓冲区的进程，才需要创建 `GraphicBufferMapper` 对象。从 Android 的日志分析，可以看到，只有 surfaceflinger 进程中，同时发生了为创建 `GraphicBufferAllocator` 和 `GraphicBufferMapper` 而加载 HAL gralloc。而为创建 `GraphicBufferMapper` 而加载 HAL gralloc 则发生在 zygote、bootanimation 等多个进程。可见能够访问图形缓冲区的进程包括 Android Java 应用等多个进程。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# Android OpenGL 图形系统分析系列文章
[在 Android 中使用 OpenGL](https://www.wolfcstech.com/2017/09/13/opengl_on_android_with_sv/)
[Android 图形驱动初始化](https://www.wolfcstech.com/2017/09/14/egl_init_drivers/)
[EGL Context 创建](https://www.wolfcstech.com/2017/09/15/egl_context_creation/)
[Android 图形系统之图形缓冲区分配](https://www.wolfcstech.com/2017/09/20/android_graphics_bufferalloc/)
[Android 图形系统之gralloc](https://www.wolfcstech.com/2017/09/21/android_graphics_gralloc/)