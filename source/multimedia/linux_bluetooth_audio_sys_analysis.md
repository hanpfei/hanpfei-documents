@[TOC](Linux 系统蓝牙音频服务实现分析)

Linux 系统中，蓝牙音频服务实现为系统音频服务 PulseAudio 的可加载模块，它用来以 PulseAudio 标准的方式描述蓝牙音频设备，将其嵌入 PulseAudio 的音频处理流水线，并呈现给用户，支持用户切换音频设备，如蓝牙耳机。

## 蓝牙音频设备变化监听
Linux 系统蓝牙音频服务的入口点是 **module-bluetooth-discover** 和 **module-bluetooth-policy** 模块，Linux 系统音频服务 PulseAudio 启动时加载 */etc/pulse/default.pa* 配置文件，以确定要加载的模块，与蓝牙相关的模块如下：
```
### Automatically load driver modules for Bluetooth hardware
.ifexists module-bluetooth-policy.so
load-module module-bluetooth-policy
.endif

.ifexists module-bluetooth-discover.so
load-module module-bluetooth-discover
.endif
```

**module-bluetooth-discover** 模块用来处理蓝牙音频设备。bluez 是 Linux 系统的官方蓝牙协议栈实现，**module-bluetooth-discover** 模块是 **module-bluez5-discover** 模块壳。**module-bluetooth-discover** 模块的全部实现 (位于 *pulseaudio/src/modules/bluetooth/module-bluetooth-discover.c*) 如下：
```
PA_MODULE_USAGE(
    "headset=ofono|native|auto"
    "autodetect_mtu=<boolean>"
);

struct userdata {
    uint32_t bluez5_module_idx;
};

int pa__init(pa_module* m) {
    struct userdata *u;
    pa_module *mm;

    pa_assert(m);

    m->userdata = u = pa_xnew0(struct userdata, 1);
    u->bluez5_module_idx = PA_INVALID_INDEX;

    if (pa_module_exists("module-bluez5-discover")) {
        pa_module_load(&mm, m->core, "module-bluez5-discover", m->argument);
        if (mm)
            u->bluez5_module_idx = mm->index;
    }

    if (u->bluez5_module_idx == PA_INVALID_INDEX) {
        pa_xfree(u);
        return -1;
    }

    return 0;
}

void pa__done(pa_module* m) {
    struct userdata *u;

    pa_assert(m);

    if (!(u = m->userdata))
        return;

    if (u->bluez5_module_idx != PA_INVALID_INDEX)
        pa_module_unload_by_index(m->core, u->bluez5_module_idx, true);

    pa_xfree(u);
}
```

**module-bluetooth-discover** 模块仅被用来加载 **module-bluez5-discover** 模块。

Linux 系统蓝牙音频服务的中心是一个 `pa_bluetooth_discovery` 对象，在系统音频服务 PulseAudio 中，它用来管理蓝牙音频相关的各种对象，包括 dbus 连接、蓝牙适配器/控制器、蓝牙设备、传输和蓝牙后端等。`pa_bluetooth_discovery` 结构定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
struct pa_bluetooth_discovery {
    PA_REFCNT_DECLARE;

    pa_core *core;
    pa_dbus_connection *connection;
    bool filter_added;
    bool matches_added;
    bool objects_listed;
    pa_hook hooks[PA_BLUETOOTH_HOOK_MAX];
    pa_hashmap *adapters;
    pa_hashmap *devices;
    pa_hashmap *transports;
    pa_bluetooth_profile_status_t profiles_status[PA_BLUETOOTH_PROFILE_COUNT];

    int headset_backend;
    pa_bluetooth_backend *ofono_backend, *native_backend;
    PA_LLIST_HEAD(pa_dbus_pending, pending);
    bool enable_native_hsp_hs;
    bool enable_native_hfp_hf;
    bool enable_msbc;
};
```

**module-bluez5-discover** 模块本身所做的主要工作有两个，一是解析传入的模块参数，并创建 `pa_bluetooth_discovery` 对象；二是向 `pa_bluetooth_discovery` 对象注册蓝牙设备连接变化回调函数，在回调函数里，**module-bluez5-discover** 模块为蓝牙音频设备加载 **module-bluez5-device** 模块。**module-bluez5-device** 模块为蓝牙音频设备创建 sink 和 source 对象，之后蓝牙音频设备对用户可用。

**module-bluez5-discover** 模块的加载和卸载函数定义 (位于 *pulseaudio/src/modules/bluetooth/module-bluez5-discover.c*) 如下：
```
PA_MODULE_LOAD_ONCE(true);
PA_MODULE_USAGE(
    "headset=ofono|native|auto"
    "autodetect_mtu=<boolean>"
    "enable_msbc=<boolean, enable mSBC support in native and oFono backends, default is true>"
    "output_rate_refresh_interval_ms=<interval between attempts to improve output rate in milliseconds>"
    "enable_native_hsp_hs=<boolean, enable HSP support in native backend>"
    "enable_native_hfp_hf=<boolean, enable HFP support in native backend>"
    "avrcp_absolute_volume=<synchronize volume with peer, true by default>"
);

static const char* const valid_modargs[] = {
    "headset",
    "autodetect_mtu",
    "enable_msbc",
    "output_rate_refresh_interval_ms",
    "enable_native_hsp_hs",
    "enable_native_hfp_hf",
    "avrcp_absolute_volume",
    NULL
};

struct userdata {
    pa_module *module;
    pa_core *core;
    pa_hashmap *loaded_device_paths;
    pa_hook_slot *device_connection_changed_slot;
    pa_bluetooth_discovery *discovery;
    bool autodetect_mtu;
    bool avrcp_absolute_volume;
    uint32_t output_rate_refresh_interval_ms;
};
 . . . . . .
#ifdef HAVE_BLUEZ_5_NATIVE_HEADSET
const char *default_headset_backend = "native";
#else
const char *default_headset_backend = "ofono";
#endif

int pa__init(pa_module *m) {
    struct userdata *u;
    pa_modargs *ma;
    const char *headset_str;
    int headset_backend;
    bool autodetect_mtu;
    bool enable_msbc;
    bool avrcp_absolute_volume;
    uint32_t output_rate_refresh_interval_ms;
    bool enable_native_hsp_hs;
    bool enable_native_hfp_hf;

    pa_assert(m);

    if (!(ma = pa_modargs_new(m->argument, valid_modargs))) {
        pa_log("failed to parse module arguments.");
        goto fail;
    }

    pa_assert_se(headset_str = pa_modargs_get_value(ma, "headset", default_headset_backend));
    if (pa_streq(headset_str, "ofono"))
        headset_backend = HEADSET_BACKEND_OFONO;
    else if (pa_streq(headset_str, "native"))
        headset_backend = HEADSET_BACKEND_NATIVE;
    else if (pa_streq(headset_str, "auto"))
        headset_backend = HEADSET_BACKEND_AUTO;
    else {
        pa_log("headset parameter must be either ofono, native or auto (found %s)", headset_str);
        goto fail;
    }

    /* default value if no module parameter */
    enable_native_hfp_hf = (headset_backend == HEADSET_BACKEND_NATIVE);

    autodetect_mtu = false;
    if (pa_modargs_get_value_boolean(ma, "autodetect_mtu", &autodetect_mtu) < 0) {
        pa_log("Invalid boolean value for autodetect_mtu parameter");
    }
    enable_msbc = true;
    if (pa_modargs_get_value_boolean(ma, "enable_msbc", &enable_msbc) < 0) {
        pa_log("Invalid boolean value for enable_msbc parameter");
    }
    enable_native_hfp_hf = true;
    if (pa_modargs_get_value_boolean(ma, "enable_native_hfp_hf", &enable_native_hfp_hf) < 0) {
        pa_log("enable_native_hfp_hf must be true or false");
        goto fail;
    }
    enable_native_hsp_hs = !enable_native_hfp_hf;
    if (pa_modargs_get_value_boolean(ma, "enable_native_hsp_hs", &enable_native_hsp_hs) < 0) {
        pa_log("enable_native_hsp_hs must be true or false");
        goto fail;
    }

    avrcp_absolute_volume = true;
    if (pa_modargs_get_value_boolean(ma, "avrcp_absolute_volume", &avrcp_absolute_volume) < 0) {
        pa_log("avrcp_absolute_volume must be true or false");
        goto fail;
    }

    output_rate_refresh_interval_ms = DEFAULT_OUTPUT_RATE_REFRESH_INTERVAL_MS;
    if (pa_modargs_get_value_u32(ma, "output_rate_refresh_interval_ms", &output_rate_refresh_interval_ms) < 0) {
        pa_log("Invalid value for output_rate_refresh_interval parameter.");
        goto fail;
    }

    m->userdata = u = pa_xnew0(struct userdata, 1);
    u->module = m;
    u->core = m->core;
    u->autodetect_mtu = autodetect_mtu;
    u->avrcp_absolute_volume = avrcp_absolute_volume;
    u->output_rate_refresh_interval_ms = output_rate_refresh_interval_ms;
    u->loaded_device_paths = pa_hashmap_new(pa_idxset_string_hash_func, pa_idxset_string_compare_func);

    if (!(u->discovery = pa_bluetooth_discovery_get(u->core, headset_backend, enable_native_hsp_hs, enable_native_hfp_hf, enable_msbc)))
        goto fail;

    u->device_connection_changed_slot =
        pa_hook_connect(pa_bluetooth_discovery_hook(u->discovery, PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED),
                        PA_HOOK_NORMAL, (pa_hook_cb_t) device_connection_changed_cb, u);

    pa_modargs_free(ma);
    return 0;

fail:
    if (ma)
        pa_modargs_free(ma);
    pa__done(m);
    return -1;
}

void pa__done(pa_module *m) {
    struct userdata *u;

    pa_assert(m);

    if (!(u = m->userdata))
        return;

    if (u->device_connection_changed_slot)
        pa_hook_slot_free(u->device_connection_changed_slot);

    if (u->loaded_device_paths)
        pa_hashmap_free(u->loaded_device_paths);

    if (u->discovery)
        pa_bluetooth_discovery_unref(u->discovery);

    pa_xfree(u);
}
```

**module-bluez5-discover** 模块调用 `pa_bluetooth_discovery_get()` 函数创建 `pa_bluetooth_discovery` 对象。`pa_bluetooth_discovery_get()` 函数定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
static pa_dbus_pending* send_and_add_to_pending(pa_bluetooth_discovery *y, DBusMessage *m,
                                                                  DBusPendingCallNotifyFunction func, void *call_data) {
    pa_dbus_pending *p;
    DBusPendingCall *call;

    pa_assert(y);
    pa_assert(m);

    pa_assert_se(dbus_connection_send_with_reply(pa_dbus_connection_get(y->connection), m, &call, -1));

    p = pa_dbus_pending_new(pa_dbus_connection_get(y->connection), m, call, y, call_data);
    PA_LLIST_PREPEND(pa_dbus_pending, y->pending, p);
    dbus_pending_call_set_notify(call, func, p, NULL);

    return p;
}
 . . . . . .
static void get_managed_objects(pa_bluetooth_discovery *y) {
    DBusMessage *m;

    pa_assert(y);

    pa_assert_se(m = dbus_message_new_method_call(BLUEZ_SERVICE, "/", DBUS_INTERFACE_OBJECT_MANAGER,
                                                  "GetManagedObjects"));
    send_and_add_to_pending(y, m, get_managed_objects_reply, NULL);
}
 . . . . . .
static void endpoint_init(pa_bluetooth_discovery *y, const char *endpoint) {
    static const DBusObjectPathVTable vtable_endpoint = {
        .message_function = endpoint_handler,
    };

    pa_assert(y);
    pa_assert(endpoint);

    pa_assert_se(dbus_connection_register_object_path(pa_dbus_connection_get(y->connection), endpoint,
                                                      &vtable_endpoint, y));
}
 . . . . . .
static void object_manager_init(pa_bluetooth_discovery *y) {
    static const DBusObjectPathVTable vtable = {
        .message_function = object_manager_handler,
    };

    pa_assert(y);
    pa_assert_se(dbus_connection_register_object_path(pa_dbus_connection_get(y->connection),
                A2DP_OBJECT_MANAGER_PATH, &vtable, y));
}
 . . . . . .
pa_bluetooth_discovery* pa_bluetooth_discovery_get(pa_core *c, int headset_backend, bool enable_native_hsp_hs, bool enable_native_hfp_hf, bool enable_msbc) {
    pa_bluetooth_discovery *y;
    DBusError err;
    DBusConnection *conn;
    unsigned i, count;
    const pa_a2dp_endpoint_conf *endpoint_conf;
    char *endpoint;

    pa_bluetooth_a2dp_codec_gst_init();
    y = pa_xnew0(pa_bluetooth_discovery, 1);
    PA_REFCNT_INIT(y);
    y->core = c;
    y->headset_backend = headset_backend;
    y->enable_native_hsp_hs = enable_native_hsp_hs;
    y->enable_native_hfp_hf = enable_native_hfp_hf;
    y->enable_msbc = enable_msbc;
    y->adapters = pa_hashmap_new_full(pa_idxset_string_hash_func, pa_idxset_string_compare_func, NULL,
                                      (pa_free_cb_t) adapter_free);
    y->devices = pa_hashmap_new_full(pa_idxset_string_hash_func, pa_idxset_string_compare_func, NULL,
                                     (pa_free_cb_t) device_free);
    y->transports = pa_hashmap_new(pa_idxset_string_hash_func, pa_idxset_string_compare_func);
    PA_LLIST_HEAD_INIT(pa_dbus_pending, y->pending);

    for (i = 0; i < PA_BLUETOOTH_HOOK_MAX; i++)
        pa_hook_init(&y->hooks[i], y);

    pa_shared_set(c, "bluetooth-discovery", y);

    dbus_error_init(&err);

    if (!(y->connection = pa_dbus_bus_get(y->core, DBUS_BUS_SYSTEM, &err))) {
        pa_log_error("Failed to get D-Bus connection: %s", err.message);
        goto fail;
    }

    conn = pa_dbus_connection_get(y->connection);

    /* dynamic detection of bluetooth audio devices */
    if (!dbus_connection_add_filter(conn, filter_cb, y, NULL)) {
        pa_log_error("Failed to add filter function");
        goto fail;
    }
    y->filter_added = true;

    if (pa_dbus_add_matches(conn, &err,
            "type='signal',sender='" DBUS_SERVICE_DBUS "',interface='" DBUS_INTERFACE_DBUS "',member='NameOwnerChanged'"
            ",arg0='" BLUEZ_SERVICE "'",
            "type='signal',sender='" BLUEZ_SERVICE "',interface='" DBUS_INTERFACE_OBJECT_MANAGER "',member='InterfacesAdded'",
            "type='signal',sender='" BLUEZ_SERVICE "',interface='" DBUS_INTERFACE_OBJECT_MANAGER "',"
            "member='InterfacesRemoved'",
            "type='signal',sender='" BLUEZ_SERVICE "',interface='" DBUS_INTERFACE_PROPERTIES "',member='PropertiesChanged'"
            ",arg0='" BLUEZ_ADAPTER_INTERFACE "'",
            "type='signal',sender='" BLUEZ_SERVICE "',interface='" DBUS_INTERFACE_PROPERTIES "',member='PropertiesChanged'"
            ",arg0='" BLUEZ_DEVICE_INTERFACE "'",
            "type='signal',sender='" BLUEZ_SERVICE "',interface='" DBUS_INTERFACE_PROPERTIES "',member='PropertiesChanged'"
            ",arg0='" BLUEZ_MEDIA_ENDPOINT_INTERFACE "'",
            "type='signal',sender='" BLUEZ_SERVICE "',interface='" DBUS_INTERFACE_PROPERTIES "',member='PropertiesChanged'"
            ",arg0='" BLUEZ_MEDIA_TRANSPORT_INTERFACE "'",
            NULL) < 0) {
        pa_log_error("Failed to add D-Bus matches: %s", err.message);
        goto fail;
    }
    y->matches_added = true;

    object_manager_init(y);

    count = pa_bluetooth_a2dp_endpoint_conf_count();
    for (i = 0; i < count; i++) {
        endpoint_conf = pa_bluetooth_a2dp_endpoint_conf_iter(i);
        if (endpoint_conf->can_be_supported(false)) {
            endpoint = pa_sprintf_malloc("%s/%s", A2DP_SINK_ENDPOINT, endpoint_conf->bt_codec.name);
            endpoint_init(y, endpoint);
            pa_xfree(endpoint);
        }

        if (endpoint_conf->can_be_supported(true)) {
            endpoint = pa_sprintf_malloc("%s/%s", A2DP_SOURCE_ENDPOINT, endpoint_conf->bt_codec.name);
            endpoint_init(y, endpoint);
            pa_xfree(endpoint);
        }
    }

    get_managed_objects(y);

    return y;

fail:
    pa_bluetooth_discovery_unref(y);
    dbus_error_free(&err);

    return NULL;
}
```

`pa_bluetooth_discovery_get()` 函数的执行过程如下：

1. 初始化 GStreamer 的蓝牙 A2DP codec。
2. 新建 `pa_bluetooth_discovery` 对象，初始化其各个字段，并将其以 **bluetooth-discovery** 键放进 PulseAudio 核心 `pa_core`。
3. 获得 dbus 连接。PulseAudio 中需要访问 dbus 连接的地方有多个，dbus 连接在 PulseAudio 核心 `pa_core` 中管理，需要 dbus 连接的地方传入所需连接的类型获得 dbus 连接，这里获得 **DBUS_BUS_SYSTEM** 类型的 dbus 连接。
4. 为 dbus 连接添加 filter 回调 `filter_cb()`，为 dbus 连接添加 matches，即添加要监听的消息或通知类型。
5. 初始化对象管理器。具体是指向 DBUS 注册 **/MediaEndpoint** 路径的处理函数 `object_manager_handler()`。注册的处理函数在 bluez 蓝牙服务中访问。
6. 初始化蓝牙 A2DP endpoint 配置。具体是指向 DBUS 注册 **/MediaEndpoint/A2DPSink/[codec_name]** 和 **/MediaEndpoint/A2DPSource/[codec_name]** 路径，如 **/MediaEndpoint/A2DPSink/sbc** 和 **/MediaEndpoint/A2DPSource/sbc** 的处理函数 `endpoint_handler()`。
7. 向蓝牙服务发起方法调用请求，获取蓝牙音频相关对象，响应消息由 `get_managed_objects_reply()` 函数处理。

`pa_bluetooth_discovery` 对象的创建过程，主要初始化了 dbus 通信相关的逻辑，通信是双向的，通信的方式有多种，有远程过程调用和消息信号。通过为 dbus 连接添加 matches 注册要处理的消息信号类型，收到的 dbus 消息由 `filter_cb()` 回调处理，这些消息信号由 bluez 蓝牙服务发出。这些消息信号主要有如下这些：
![pulseauido_bluetooth_dbus_signal](./images/pulseauido_bluetooth_dbus_signal.png)
这些消息信号来自 2 个发送者，共有 7 种。注册了 **/MediaEndpoint** 等多个路径的远程过程调用服务，它们由 bluez 蓝牙服务调用。最后发起了一个对 bluez 蓝牙服务的远程过程调用，获取蓝牙音频相关对象。

**module-bluez5-discover** 模块提供蓝牙设备连接变化回调函数 `device_connection_changed_cb()`，这个函数定义 (位于 *pulseaudio/src/modules/bluetooth/module-bluez5-discover.c*) 如下：
```
static pa_hook_result_t device_connection_changed_cb(pa_bluetooth_discovery *y, const pa_bluetooth_device *d, struct userdata *u) {
    bool module_loaded;

    pa_assert(d);
    pa_assert(u);

    module_loaded = pa_hashmap_get(u->loaded_device_paths, d->path) ? true : false;

    /* When changing A2DP codec there is no transport connected, ensure that no module is unloaded */
    if (module_loaded && !pa_bluetooth_device_any_transport_connected(d) &&
            !d->codec_switching_in_progress) {
        /* disconnection, the module unloads itself */
        pa_log_debug("Unregistering module for %s", d->path);
        pa_hashmap_remove(u->loaded_device_paths, d->path);
        return PA_HOOK_OK;
    }

    if (!module_loaded && pa_bluetooth_device_any_transport_connected(d)) {
        /* a new device has been connected */
        pa_module *m;
        char *args = pa_sprintf_malloc("path=%s autodetect_mtu=%i output_rate_refresh_interval_ms=%u"
                                       " avrcp_absolute_volume=%i",
                                       d->path,
                                       (int)u->autodetect_mtu,
                                       u->output_rate_refresh_interval_ms,
                                       (int)u->avrcp_absolute_volume);

        pa_log_debug("Loading module-bluez5-device %s", args);
        pa_module_load(&m, u->module->core, "module-bluez5-device", args);
        pa_xfree(args);

        if (m)
            /* No need to duplicate the path here since the device object will
             * exist for the whole hashmap entry lifespan */
            pa_hashmap_put(u->loaded_device_paths, d->path, d->path);
        else
            pa_log_warn("Failed to load module for device %s", d->path);

        return PA_HOOK_OK;
    }

    return PA_HOOK_OK;
}
```

回调函数 `device_connection_changed_cb()` 处理两种情况：

 * 蓝牙设备路径在已加载蓝牙设备路径列表中，且蓝牙设备上没有任何已连接的 transport，将蓝牙设备路径从已加载蓝牙设备路径列表移出。
 * 蓝牙设备路径不在已加载蓝牙设备路径列表中，且蓝牙设备上存在已连接的 transport，则为蓝牙音频设备加载 **module-bluez5-device** 模块，模块加载成功时，将蓝牙设备路径添加进已加载蓝牙设备路径列表。

蓝牙音频设备变化监听，将通过 DBUS 连接，接收包括蓝牙控制器/适配器、蓝牙设备和 profile 等的变化通知。**module-bluez5-discover** 模块在蓝牙音频服务中扮演的角色，和 **module-udev-detect** 及 **module-alsa-card** 模块在 ALSA 中扮演的角色一样。
## 蓝牙设备管理
通过 `pa_bluetooth_discovery_get()` 函数创建 `pa_bluetooth_discovery` 对象的最后，会向蓝牙系统服务发起方法调用请求，获取蓝牙音频设备有关信息，响应消息由 `get_managed_objects_reply()` 函数处理。这是系统蓝牙音频服务获取蓝牙音频设备有关信息的开始。

蓝牙音频设备有关信息有多种类型，分别为蓝牙适配器/控制器 (Adapter)、蓝牙设备 (Device) 和媒体端点 (MediaEndpoint)，它们分别由 `pa_bluetooth_adapter`、`pa_bluetooth_device` 和 `pa_a2dp_codec_id`/`pa_a2dp_codec_capabilities` 对象描述。`pa_bluetooth_adapter` 和 `pa_bluetooth_device` 类型定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.h*) 如下：
```
struct pa_bluetooth_device {
    pa_bluetooth_discovery *discovery;
    pa_bluetooth_adapter *adapter;

    bool enable_hfp_hf;
    bool properties_received;
    bool tried_to_link_with_adapter;
    bool valid;
    bool autodetect_mtu;
    bool codec_switching_in_progress;
    bool avrcp_absolute_volume;
    uint32_t output_rate_refresh_interval_ms;

    /* Device information */
    char *path;
    char *adapter_path;
    char *alias;
    char *address;
    uint32_t class_of_device;
    pa_hashmap *uuids; /* char* -> char* (hashmap-as-a-set) */
    /* pa_a2dp_codec_id* -> pa_hashmap ( char* (remote endpoint) -> struct a2dp_codec_capabilities* ) */
    pa_hashmap *a2dp_sink_endpoints;
    pa_hashmap *a2dp_source_endpoints;

    pa_bluetooth_transport *transports[PA_BLUETOOTH_PROFILE_COUNT];

    pa_time_event *wait_for_profiles_timer;

    bool has_battery_level;
    uint8_t battery_level;
    const char *battery_source;
};

struct pa_bluetooth_adapter {
    pa_bluetooth_discovery *discovery;
    char *path;
    char *address;
    pa_hashmap *uuids; /* char* -> char* (hashmap-as-a-set) */

    bool valid;
    bool application_registered;
    bool battery_provider_registered;
};
```

`pa_a2dp_codec_id`/`pa_a2dp_codec_capabilities` 类型定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
#define MAX_A2DP_CAPS_SIZE 254
#define DEFAULT_OUTPUT_RATE_REFRESH_INTERVAL_MS 500

typedef struct pa_a2dp_codec_capabilities {
    uint8_t size;
    uint8_t buffer[]; /* max size is 254 bytes */
} pa_a2dp_codec_capabilities;

typedef struct pa_a2dp_codec_id {
    uint8_t codec_id;
    uint32_t vendor_id;
    uint16_t vendor_codec_id;
} pa_a2dp_codec_id;
```

`pa_bluetooth_adapter` 和 `pa_bluetooth_device` 对象在 `pa_bluetooth_discovery` 对象中维护，从属于蓝牙设备 (Device) 的媒体端点 (MediaEndpoint) 信息在 `pa_bluetooth_device` 对象中维护。

解析系统蓝牙服务 bluez 发出的蓝牙音频设备有关信息的 `get_managed_objects_reply()` 函数定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
static pa_bluetooth_device* device_create(pa_bluetooth_discovery *y, const char *path) {
    pa_bluetooth_device *d;

    pa_assert(y);
    pa_assert(path);

    d = pa_xnew0(pa_bluetooth_device, 1);
    d->discovery = y;
    d->enable_hfp_hf = pa_bluetooth_discovery_get_enable_native_hfp_hf(y);
    d->path = pa_xstrdup(path);
    d->uuids = pa_hashmap_new_full(pa_idxset_string_hash_func, pa_idxset_string_compare_func, NULL, pa_xfree);
    d->a2dp_sink_endpoints = pa_hashmap_new_full(pa_a2dp_codec_id_hash_func, pa_a2dp_codec_id_compare_func, pa_xfree, (pa_free_cb_t)pa_hashmap_free);
    d->a2dp_source_endpoints = pa_hashmap_new_full(pa_a2dp_codec_id_hash_func, pa_a2dp_codec_id_compare_func, pa_xfree, (pa_free_cb_t)pa_hashmap_free);

    pa_hashmap_put(y->devices, d->path, d);

    return d;
}
 . . . . . .
static pa_bluetooth_adapter* adapter_create(pa_bluetooth_discovery *y, const char *path) {
    pa_bluetooth_adapter *a;

    pa_assert(y);
    pa_assert(path);

    a = pa_xnew0(pa_bluetooth_adapter, 1);
    a->discovery = y;
    a->path = pa_xstrdup(path);
    a->uuids = pa_hashmap_new_full(pa_idxset_string_hash_func, pa_idxset_string_compare_func, NULL, pa_xfree);

    pa_hashmap_put(y->adapters, a->path, a);

    return a;
}
 . . . . . .
static void parse_device_property(pa_bluetooth_device *d, DBusMessageIter *i) {
    const char *key;
    DBusMessageIter variant_i;

    pa_assert(d);

    key = check_variant_property(i);
    if (key == NULL) {
        pa_log_error("Received invalid property for device %s", d->path);
        return;
    }

    dbus_message_iter_recurse(i, &variant_i);

    switch (dbus_message_iter_get_arg_type(&variant_i)) {

        case DBUS_TYPE_STRING: {
            const char *value;
            dbus_message_iter_get_basic(&variant_i, &value);

            if (pa_streq(key, "Alias")) {
                pa_xfree(d->alias);
                d->alias = pa_xstrdup(value);
                pa_log_debug("%s: %s", key, value);
            } else if (pa_streq(key, "Address")) {
                if (d->properties_received) {
                    pa_log_warn("Device property 'Address' expected to be constant but changed for %s, ignoring", d->path);
                    return;
                }

                if (d->address) {
                    pa_log_warn("Device %s: Received a duplicate 'Address' property, ignoring", d->path);
                    return;
                }

                d->address = pa_xstrdup(value);
                pa_log_debug("%s: %s", key, value);
            }

            break;
        }

        case DBUS_TYPE_OBJECT_PATH: {
            const char *value;
            dbus_message_iter_get_basic(&variant_i, &value);

            if (pa_streq(key, "Adapter")) {

                if (d->properties_received) {
                    pa_log_warn("Device property 'Adapter' expected to be constant but changed for %s, ignoring", d->path);
                    return;
                }

                if (d->adapter_path) {
                    pa_log_warn("Device %s: Received a duplicate 'Adapter' property, ignoring", d->path);
                    return;
                }

                d->adapter_path = pa_xstrdup(value);
                pa_log_debug("%s: %s", key, value);
            }

            break;
        }

        case DBUS_TYPE_UINT32: {
            uint32_t value;
            dbus_message_iter_get_basic(&variant_i, &value);

            if (pa_streq(key, "Class")) {
                d->class_of_device = value;
                pa_log_debug("%s: %d", key, value);
            }

            break;
        }

        case DBUS_TYPE_ARRAY: {
            DBusMessageIter ai;
            dbus_message_iter_recurse(&variant_i, &ai);

            if (dbus_message_iter_get_arg_type(&ai) == DBUS_TYPE_STRING && pa_streq(key, "UUIDs")) {
                /* bluetoothd never removes UUIDs from a device object so we
                 * don't need to check for disappeared UUIDs here. */
                while (dbus_message_iter_get_arg_type(&ai) != DBUS_TYPE_INVALID) {
                    const char *value;
                    char *uuid;

                    dbus_message_iter_get_basic(&ai, &value);

                    if (pa_hashmap_get(d->uuids, value)) {
                        dbus_message_iter_next(&ai);
                        continue;
                    }

                    uuid = pa_xstrdup(value);
                    pa_hashmap_put(d->uuids, uuid, uuid);

                    pa_log_debug("%s: %s", key, value);
                    dbus_message_iter_next(&ai);
                }
            }

            break;
        }
    }
}

static void parse_device_properties(pa_bluetooth_device *d, DBusMessageIter *i) {
    DBusMessageIter element_i;

    dbus_message_iter_recurse(i, &element_i);

    while (dbus_message_iter_get_arg_type(&element_i) == DBUS_TYPE_DICT_ENTRY) {
        DBusMessageIter dict_i;

        dbus_message_iter_recurse(&element_i, &dict_i);
        parse_device_property(d, &dict_i);
        dbus_message_iter_next(&element_i);
    }

    if (!d->properties_received) {
        d->properties_received = true;
        device_update_valid(d);

        if (!d->address || !d->adapter_path || !d->alias)
            pa_log_error("Non-optional information missing for device %s", d->path);
    }
}

static void parse_adapter_properties(pa_bluetooth_adapter *a, DBusMessageIter *i, bool is_property_change) {
    DBusMessageIter element_i;

    pa_assert(a);

    dbus_message_iter_recurse(i, &element_i);

    while (dbus_message_iter_get_arg_type(&element_i) == DBUS_TYPE_DICT_ENTRY) {
        DBusMessageIter dict_i, variant_i;
        const char *key;

        dbus_message_iter_recurse(&element_i, &dict_i);

        key = check_variant_property(&dict_i);
        if (key == NULL) {
            pa_log_error("Received invalid property for adapter %s", a->path);
            return;
        }

        dbus_message_iter_recurse(&dict_i, &variant_i);

        if (dbus_message_iter_get_arg_type(&variant_i) == DBUS_TYPE_STRING && pa_streq(key, "Address")) {
            const char *value;

            if (is_property_change) {
                pa_log_warn("Adapter property 'Address' expected to be constant but changed for %s, ignoring", a->path);
                return;
            }

            if (a->address) {
                pa_log_warn("Adapter %s received a duplicate 'Address' property, ignoring", a->path);
                return;
            }

            dbus_message_iter_get_basic(&variant_i, &value);
            a->address = pa_xstrdup(value);
            a->valid = true;
        } else if (dbus_message_iter_get_arg_type(&variant_i) == DBUS_TYPE_ARRAY) {
            DBusMessageIter ai;
            dbus_message_iter_recurse(&variant_i, &ai);

            if (dbus_message_iter_get_arg_type(&ai) == DBUS_TYPE_STRING && pa_streq(key, "UUIDs")) {
                pa_hashmap_remove_all(a->uuids);
                while (dbus_message_iter_get_arg_type(&ai) != DBUS_TYPE_INVALID) {
                    const char *value;
                    char *uuid;

                    dbus_message_iter_get_basic(&ai, &value);

                    if (pa_hashmap_get(a->uuids, value)) {
                        dbus_message_iter_next(&ai);
                        continue;
                    }

                    uuid = pa_xstrdup(value);
                    pa_hashmap_put(a->uuids, uuid, uuid);

                    pa_log_debug("%s: %s", key, value);
                    dbus_message_iter_next(&ai);
                }
                pa_hook_fire(pa_bluetooth_discovery_hook(a->discovery, PA_BLUETOOTH_HOOK_ADAPTER_UUIDS_CHANGED), a);
            }
        }

        dbus_message_iter_next(&element_i);
    }
}

static void register_legacy_sbc_endpoint_reply(DBusPendingCall *pending, void *userdata) {
    DBusMessage *r;
    pa_dbus_pending *p;
    pa_bluetooth_discovery *y;
    char *endpoint;

    pa_assert(pending);
    pa_assert_se(p = userdata);
    pa_assert_se(y = p->context_data);
    pa_assert_se(endpoint = p->call_data);
    pa_assert_se(r = dbus_pending_call_steal_reply(pending));

    if (dbus_message_is_error(r, BLUEZ_ERROR_NOT_SUPPORTED)) {
        pa_log_info("Couldn't register endpoint %s because it is disabled in BlueZ", endpoint);
        goto finish;
    }

    if (dbus_message_get_type(r) == DBUS_MESSAGE_TYPE_ERROR) {
        pa_log_error(BLUEZ_MEDIA_INTERFACE ".RegisterEndpoint() failed: %s: %s", dbus_message_get_error_name(r),
                     pa_dbus_get_error_message(r));
        goto finish;
    }

finish:
    dbus_message_unref(r);

    PA_LLIST_REMOVE(pa_dbus_pending, y->pending, p);
    pa_dbus_pending_free(p);

    pa_xfree(endpoint);
}

static void register_legacy_sbc_endpoint(pa_bluetooth_discovery *y, const pa_a2dp_endpoint_conf *endpoint_conf, const char *path, const char *endpoint, const char *uuid) {
    DBusMessage *m;
    DBusMessageIter i, d;
    uint8_t capabilities[MAX_A2DP_CAPS_SIZE];
    size_t capabilities_size;
    uint8_t codec_id;

    pa_log_debug("Registering %s on adapter %s", endpoint, path);

    codec_id = endpoint_conf->id.codec_id;
    capabilities_size = endpoint_conf->fill_capabilities(capabilities);
    pa_assert(capabilities_size != 0);

    pa_assert_se(m = dbus_message_new_method_call(BLUEZ_SERVICE, path, BLUEZ_MEDIA_INTERFACE, "RegisterEndpoint"));

    dbus_message_iter_init_append(m, &i);
    pa_assert_se(dbus_message_iter_append_basic(&i, DBUS_TYPE_OBJECT_PATH, &endpoint));
    dbus_message_iter_open_container(&i, DBUS_TYPE_ARRAY,
                                     DBUS_DICT_ENTRY_BEGIN_CHAR_AS_STRING
                                     DBUS_TYPE_STRING_AS_STRING
                                     DBUS_TYPE_VARIANT_AS_STRING
                                     DBUS_DICT_ENTRY_END_CHAR_AS_STRING,
                                     &d);
    pa_dbus_append_basic_variant_dict_entry(&d, "UUID", DBUS_TYPE_STRING, &uuid);
    pa_dbus_append_basic_variant_dict_entry(&d, "Codec", DBUS_TYPE_BYTE, &codec_id);
    pa_dbus_append_basic_array_variant_dict_entry(&d, "Capabilities", DBUS_TYPE_BYTE, &capabilities, capabilities_size);

    dbus_message_iter_close_container(&i, &d);

    send_and_add_to_pending(y, m, register_legacy_sbc_endpoint_reply, pa_xstrdup(endpoint));
}

static void register_application_reply(DBusPendingCall *pending, void *userdata) {
    DBusMessage *r;
    pa_dbus_pending *p;
    pa_bluetooth_adapter *a;
    pa_bluetooth_discovery *y;
    char *path;
    bool fallback = true;

    pa_assert(pending);
    pa_assert_se(p = userdata);
    pa_assert_se(y = p->context_data);
    pa_assert_se(path = p->call_data);
    pa_assert_se(r = dbus_pending_call_steal_reply(pending));

    if (dbus_message_is_error(r, BLUEZ_ERROR_NOT_SUPPORTED)) {
        pa_log_info("Couldn't register media application for adapter %s because it is disabled in BlueZ", path);
        goto finish;
    }

    if (dbus_message_get_type(r) == DBUS_MESSAGE_TYPE_ERROR) {
        pa_log_warn(BLUEZ_MEDIA_INTERFACE ".RegisterApplication() failed: %s: %s",
                dbus_message_get_error_name(r), pa_dbus_get_error_message(r));
        pa_log_warn("Couldn't register media application for adapter %s", path);
        goto finish;
    }

    a = pa_hashmap_get(y->adapters, path);
    if (!a) {
        pa_log_error("Couldn't register media application for adapter %s because it does not exist anymore", path);
        goto finish;
    }

    fallback = false;
    a->application_registered = true;
    pa_log_debug("Media application for adapter %s was successfully registered", path);

finish:
    dbus_message_unref(r);

    PA_LLIST_REMOVE(pa_dbus_pending, y->pending, p);
    pa_dbus_pending_free(p);

    if (fallback) {
        /* If bluez does not support RegisterApplication, fallback to old legacy API with just one SBC codec */
        const pa_a2dp_endpoint_conf *endpoint_conf;
        endpoint_conf = pa_bluetooth_get_a2dp_endpoint_conf("sbc");
        pa_assert(endpoint_conf);
        register_legacy_sbc_endpoint(y, endpoint_conf, path, A2DP_SINK_ENDPOINT "/sbc",
                PA_BLUETOOTH_UUID_A2DP_SINK);
        register_legacy_sbc_endpoint(y, endpoint_conf, path, A2DP_SOURCE_ENDPOINT "/sbc",
                PA_BLUETOOTH_UUID_A2DP_SOURCE);
        pa_log_warn("Only SBC codec is available for A2DP profiles");
    }

    pa_xfree(path);
}

static void register_application(pa_bluetooth_adapter *a) {
    DBusMessage *m;
    DBusMessageIter i, d;
    const char *object_manager_path = A2DP_OBJECT_MANAGER_PATH;

    if (a->application_registered) {
        pa_log_info("Media application is already registered for adapter %s", a->path);
        return;
    }

    pa_log_debug("Registering media application for adapter %s", a->path);

    pa_assert_se(m = dbus_message_new_method_call(BLUEZ_SERVICE, a->path,
                BLUEZ_MEDIA_INTERFACE, "RegisterApplication"));

    dbus_message_iter_init_append(m, &i);
    pa_assert_se(dbus_message_iter_append_basic(&i, DBUS_TYPE_OBJECT_PATH, &object_manager_path));
    dbus_message_iter_open_container(&i, DBUS_TYPE_ARRAY,
                                     DBUS_DICT_ENTRY_BEGIN_CHAR_AS_STRING
                                     DBUS_TYPE_STRING_AS_STRING
                                     DBUS_TYPE_VARIANT_AS_STRING
                                     DBUS_DICT_ENTRY_END_CHAR_AS_STRING,
                                     &d);
    dbus_message_iter_close_container(&i, &d);

    send_and_add_to_pending(a->discovery, m, register_application_reply, pa_xstrdup(a->path));
}

static void parse_remote_endpoint_properties(pa_bluetooth_discovery *y, const char *endpoint, DBusMessageIter *i) {
    DBusMessageIter element_i;
    pa_bluetooth_device *device;
    pa_hashmap *codec_endpoints;
    pa_hashmap *endpoints;
    pa_a2dp_codec_id *a2dp_codec_id;
    pa_a2dp_codec_capabilities *a2dp_codec_capabilities;
    const char *uuid = NULL;
    const char *device_path = NULL;
    uint8_t codec_id = 0;
    bool have_codec_id = false;
    const uint8_t *capabilities = NULL;
    int capabilities_size = 0;

    pa_log_debug("Parsing remote endpoint %s", endpoint);

    dbus_message_iter_recurse(i, &element_i);

    while (dbus_message_iter_get_arg_type(&element_i) == DBUS_TYPE_DICT_ENTRY) {
        DBusMessageIter dict_i, variant_i;
        const char *key;

        dbus_message_iter_recurse(&element_i, &dict_i);

        key = check_variant_property(&dict_i);
        if (key == NULL) {
            pa_log_error("Received invalid property for remote endpoint %s", endpoint);
            return;
        }

        dbus_message_iter_recurse(&dict_i, &variant_i);

        if (pa_streq(key, "UUID")) {
            if (dbus_message_iter_get_arg_type(&variant_i) != DBUS_TYPE_STRING) {
                pa_log_warn("Remote endpoint %s property 'UUID' is not string, ignoring", endpoint);
                return;
            }

            dbus_message_iter_get_basic(&variant_i, &uuid);
        } else if (pa_streq(key, "Codec")) {
            if (dbus_message_iter_get_arg_type(&variant_i) != DBUS_TYPE_BYTE) {
                pa_log_warn("Remote endpoint %s property 'Codec' is not byte, ignoring", endpoint);
                return;
            }

            dbus_message_iter_get_basic(&variant_i, &codec_id);
            have_codec_id = true;
        } else if (pa_streq(key, "Capabilities")) {
            DBusMessageIter array;

            if (dbus_message_iter_get_arg_type(&variant_i) != DBUS_TYPE_ARRAY) {
                pa_log_warn("Remote endpoint %s property 'Capabilities' is not array, ignoring", endpoint);
                return;
            }

            dbus_message_iter_recurse(&variant_i, &array);
            if (dbus_message_iter_get_arg_type(&array) != DBUS_TYPE_BYTE) {
                pa_log_warn("Remote endpoint %s property 'Capabilities' is not array of bytes, ignoring", endpoint);
                return;
            }

            dbus_message_iter_get_fixed_array(&array, &capabilities, &capabilities_size);
        } else if (pa_streq(key, "Device")) {
            if (dbus_message_iter_get_arg_type(&variant_i) != DBUS_TYPE_OBJECT_PATH) {
                pa_log_warn("Remote endpoint %s property 'Device' is not path, ignoring", endpoint);
                return;
            }

            dbus_message_iter_get_basic(&variant_i, &device_path);
        }

        dbus_message_iter_next(&element_i);
    }

    if (!uuid) {
        pa_log_warn("Remote endpoint %s does not have property 'UUID', ignoring", endpoint);
        return;
    }

    if (!have_codec_id) {
        pa_log_warn("Remote endpoint %s does not have property 'Codec', ignoring", endpoint);
        return;
    }

    if (!capabilities || !capabilities_size) {
        pa_log_warn("Remote endpoint %s does not have property 'Capabilities', ignoring", endpoint);
        return;
    }

    if (!device_path) {
        pa_log_warn("Remote endpoint %s does not have property 'Device', ignoring", endpoint);
        return;
    }

    device = pa_hashmap_get(y->devices, device_path);
    if (!device) {
        pa_log_warn("Device for remote endpoint %s was not found", endpoint);
        return;
    }

    if (pa_streq(uuid, PA_BLUETOOTH_UUID_A2DP_SINK)) {
        codec_endpoints = device->a2dp_sink_endpoints;
    } else if (pa_streq(uuid, PA_BLUETOOTH_UUID_A2DP_SOURCE)) {
        codec_endpoints = device->a2dp_source_endpoints;
    } else {
        pa_log_warn("Remote endpoint %s does not have valid property 'UUID', ignoring", endpoint);
        return;
    }

    if (capabilities_size < 0 || capabilities_size > MAX_A2DP_CAPS_SIZE) {
        pa_log_warn("Remote endpoint %s does not have valid property 'Capabilities', ignoring", endpoint);
        return;
    }

    a2dp_codec_id = pa_xmalloc0(sizeof(*a2dp_codec_id));
    a2dp_codec_id->codec_id = codec_id;
    if (codec_id == A2DP_CODEC_VENDOR) {
        if ((size_t)capabilities_size < sizeof(a2dp_vendor_codec_t)) {
            pa_log_warn("Remote endpoint %s does not have valid property 'Capabilities', ignoring", endpoint);
            pa_xfree(a2dp_codec_id);
            return;
        }
        a2dp_codec_id->vendor_id = A2DP_GET_VENDOR_ID(*(a2dp_vendor_codec_t *)capabilities);
        a2dp_codec_id->vendor_codec_id = A2DP_GET_CODEC_ID(*(a2dp_vendor_codec_t *)capabilities);
    } else {
        a2dp_codec_id->vendor_id = 0;
        a2dp_codec_id->vendor_codec_id = 0;
    }

    if (!pa_bluetooth_a2dp_codec_is_available(a2dp_codec_id, pa_streq(uuid, PA_BLUETOOTH_UUID_A2DP_SINK))) {
        pa_xfree(a2dp_codec_id);
        return;
    }

    a2dp_codec_capabilities = pa_xmalloc0(sizeof(*a2dp_codec_capabilities) + capabilities_size);
    a2dp_codec_capabilities->size = capabilities_size;
    memcpy(a2dp_codec_capabilities->buffer, capabilities, capabilities_size);

    endpoints = pa_hashmap_get(codec_endpoints, a2dp_codec_id);
    if (!endpoints) {
        endpoints = pa_hashmap_new_full(pa_idxset_string_hash_func, pa_idxset_string_compare_func, pa_xfree, pa_xfree);
        pa_hashmap_put(codec_endpoints, a2dp_codec_id, endpoints);
    }

    if (pa_hashmap_remove_and_free(endpoints, endpoint) >= 0)
        pa_log_debug("Replacing existing remote endpoint %s", endpoint);
    pa_hashmap_put(endpoints, pa_xstrdup(endpoint), a2dp_codec_capabilities);
}

static void parse_interfaces_and_properties(pa_bluetooth_discovery *y, DBusMessageIter *dict_i) {
    DBusMessageIter element_i;
    const char *path;
    void *state;
    pa_bluetooth_device *d;

    pa_assert(dbus_message_iter_get_arg_type(dict_i) == DBUS_TYPE_OBJECT_PATH);
    dbus_message_iter_get_basic(dict_i, &path);

    pa_assert_se(dbus_message_iter_next(dict_i));
    pa_assert(dbus_message_iter_get_arg_type(dict_i) == DBUS_TYPE_ARRAY);

    dbus_message_iter_recurse(dict_i, &element_i);

    while (dbus_message_iter_get_arg_type(&element_i) == DBUS_TYPE_DICT_ENTRY) {
        DBusMessageIter iface_i;
        const char *interface;

        dbus_message_iter_recurse(&element_i, &iface_i);

        pa_assert(dbus_message_iter_get_arg_type(&iface_i) == DBUS_TYPE_STRING);
        dbus_message_iter_get_basic(&iface_i, &interface);

        pa_assert_se(dbus_message_iter_next(&iface_i));
        pa_assert(dbus_message_iter_get_arg_type(&iface_i) == DBUS_TYPE_ARRAY);

        if (pa_streq(interface, BLUEZ_ADAPTER_INTERFACE)) {
            pa_bluetooth_adapter *a;

            if ((a = pa_hashmap_get(y->adapters, path))) {
                pa_log_error("Found duplicated D-Bus path for adapter %s", path);
                return;
            } else
                a = adapter_create(y, path);

            pa_log_debug("Adapter %s found", path);

            parse_adapter_properties(a, &iface_i, false);

            if (!a->valid)
                return;

            register_application(a);
            adapter_register_battery_provider(a);
        } else if (pa_streq(interface, BLUEZ_DEVICE_INTERFACE)) {

            if ((d = pa_hashmap_get(y->devices, path))) {
                if (d->properties_received) {
                    pa_log_error("Found duplicated D-Bus path for device %s", path);
                    return;
                }
            } else
                d = device_create(y, path);

            pa_log_debug("Device %s found", d->path);

            parse_device_properties(d, &iface_i);
        } else if (pa_streq(interface, BLUEZ_MEDIA_ENDPOINT_INTERFACE)) {
            parse_remote_endpoint_properties(y, path, &iface_i);
        } else
            pa_log_debug("Unknown interface %s found, skipping", interface);

        dbus_message_iter_next(&element_i);
    }

    PA_HASHMAP_FOREACH(d, y->devices, state) {
        if (d->properties_received && !d->tried_to_link_with_adapter) {
            if (d->adapter_path) {
                device_set_adapter(d, pa_hashmap_get(d->discovery->adapters, d->adapter_path));

                if (!d->adapter)
                    pa_log("Device %s points to a nonexistent adapter %s.", d->path, d->adapter_path);
                else if (!d->adapter->valid)
                    pa_log("Device %s points to an invalid adapter %s.", d->path, d->adapter_path);
            }

            d->tried_to_link_with_adapter = true;
        }
    }

    return;
}
 . . . . . .
static void get_managed_objects_reply(DBusPendingCall *pending, void *userdata) {
    pa_dbus_pending *p;
    pa_bluetooth_discovery *y;
    DBusMessage *r;
    DBusMessageIter arg_i, element_i;

    pa_assert_se(p = userdata);
    pa_assert_se(y = p->context_data);
    pa_assert_se(r = dbus_pending_call_steal_reply(pending));

    if (dbus_message_is_error(r, DBUS_ERROR_UNKNOWN_METHOD)) {
        pa_log_warn("BlueZ D-Bus ObjectManager not available");
        goto finish;
    }

    if (dbus_message_get_type(r) == DBUS_MESSAGE_TYPE_ERROR) {
        pa_log_error("GetManagedObjects() failed: %s: %s", dbus_message_get_error_name(r), pa_dbus_get_error_message(r));
        goto finish;
    }

    if (!dbus_message_iter_init(r, &arg_i) || !pa_streq(dbus_message_get_signature(r), "a{oa{sa{sv}}}")) {
        pa_log_error("Invalid reply signature for GetManagedObjects()");
        goto finish;
    }

    dbus_message_iter_recurse(&arg_i, &element_i);
    while (dbus_message_iter_get_arg_type(&element_i) == DBUS_TYPE_DICT_ENTRY) {
        DBusMessageIter dict_i;

        dbus_message_iter_recurse(&element_i, &dict_i);

        parse_interfaces_and_properties(y, &dict_i);

        dbus_message_iter_next(&element_i);
    }

    y->objects_listed = true;

    if (!y->native_backend && y->headset_backend != HEADSET_BACKEND_OFONO)
        y->native_backend = pa_bluetooth_native_backend_new(y->core, y, (y->headset_backend == HEADSET_BACKEND_NATIVE));
    if (!y->ofono_backend && y->headset_backend != HEADSET_BACKEND_NATIVE)
        y->ofono_backend = pa_bluetooth_ofono_backend_new(y->core, y);

finish:
    dbus_message_unref(r);

    PA_LLIST_REMOVE(pa_dbus_pending, y->pending, p);
    pa_dbus_pending_free(p);
}
```

`get_managed_objects_reply()` 函数获得的 DBUS 消息为一个数组，数组元素为字典，字典中包含特定**路径**的信息，如某个蓝牙适配器/控制器 (Adapter) 的**路径**为 */org/bluez/hci0*，某个蓝牙设备 (Device) 的**路径**为 */org/bluez/hci0/dev_24_D0_DF_96_AD_6E*，特定**路径**由多个**接口**描述，各个**接口**具有一些**属性**。这里的**路径**指蓝牙设备的标识，**接口**指 DBUS 中的一个消息结构类型，如蓝牙适配器/控制器 (Adapter)、蓝牙设备 (Device) 和媒体端点 (MediaEndpoint) 等。`get_managed_objects_reply()` 函数的某次执行日志输出如下：
```
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.AgentManager1 found, skipping
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.ProfileManager1 found, skipping
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Adapter /org/bluez/hci0 found
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001200-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001132-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001133-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001800-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000112f-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001801-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001106-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000180a-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001104-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001105-0000-1000-8000-00805f9b34fb
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00005005-0000-1000-8000-0002ee000001
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Registering media application for adapter /org/bluez/hci0
(  54.711|   0.000) D: [pulseaudio] bluez5-util.c: Registering battery provider for /org/bluez/hci0 at /org/pulseaudio/bluez/hci0
(  54.712|   0.001) N: [pulseaudio] bluez5-util.c: Could not find org.bluez.BatteryProviderManager1.RegisterBatteryProvider(), is bluetoothd started with experimental features enabled (-E flag)?
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Properties found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.GattManager1 found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.Media1 found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.NetworkServer1 found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.LEAdvertisingManager1 found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Device /org/bluez/hci0/dev_24_D0_DF_96_AD_6E found
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Address: 24:D0:DF:96:AD:6E
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Alias: AirPods Pro
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Class: 2360344
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001000-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000110b-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000110d-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 0000111e-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 00001200-0000-1000-8000-00805f9b34fb
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: UUIDs: 74ec2172-0bad-4d01-8f77-997b2be0722a
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Adapter: /org/bluez/hci0
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Properties found, skipping
(  54.712|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.MediaControl1 found, skipping
(  54.712|   0.000) D: [pulseaudio] backend-native.c: Bluetooth Headset Backend API support using the native backend
(  54.712|   0.000) D: [pulseaudio] backend-native.c: Registering Profile headset_audio_gateway 00001108-0000-1000-8000-00805f9b34fb
(  54.713|   0.000) D: [pulseaudio] backend-native.c: Registering Profile handsfree_head_unit 0000111f-0000-1000-8000-00805f9b34fb
```

上面的日志中可以看到 3 个**路径**的信息，其中第 1 个**路径**为 */org/bluez*，有 3 个 PulseAudio 不处理的**接口**；第 2 个为 */org/bluez/hci0*，有一个蓝牙适配器/控制器 (Adapter) **接口**和 6 个 PulseAudio 不处理的**接口**；第 3 个为 */org/bluez/hci0/dev_24_D0_DF_96_AD_6E*，有一个蓝牙设备 (Device) **接口**和 3 个 PulseAudio 不处理的**接口**。

`get_managed_objects_reply()` 函数遵循从 DBUS 获得的 bluez 发出的消息的格式，为 DBUS 消息的各个字典调用 `parse_interfaces_and_properties()` 函数解析各个**路径**的信息，随后创建蓝牙后端对象。

特定路径的信息的字典中，包含一个路径字段及一个字典数组，字典数组的一个元素描述一个**接口**，特定接口的字典中，包含一个接口名称字段及一个字典数组，这个字典数组中包含**接口**的属性。`parse_interfaces_and_properties()` 函数解析特定路径的信息，它的执行过程如下：

1. 获得蓝牙设备的**路径**。
2. 解析**路径**下的各个**接口**的信息。对于各个**接口**，它获得接口名称，获得**接口**的属性的字典数组，对不同**接口**执行不同的处理：
     - 蓝牙适配器/控制器 (Adapter)：检查对应蓝牙适配器/控制器 (Adapter) 路径的 `pa_bluetooth_adapter` 对象是否存在，不存在时，新建 `pa_bluetooth_adapter` 对象并放进 `pa_bluetooth_discovery` 的对应列表里；调用 `parse_adapter_properties()` 函数解析蓝牙适配器/控制器 (Adapter) 的各个属性；向 bluez 蓝牙服务为蓝牙适配器/控制器 (Adapter) 注册媒体应用；向 bluez 蓝牙服务为蓝牙适配器/控制器 (Adapter) 注册battery_provider。
     - 蓝牙设备 (Device) ：检查对应蓝牙设备 (Device) 路径的 `pa_bluetooth_device` 对象是否存在，不存在时，新建 `pa_bluetooth_device` 对象并放进 `pa_bluetooth_discovery` 的对应列表里；调用 `parse_device_properties()` 函数解析蓝牙设备 (Device) 的各个属性。
     - 媒体端点 (MediaEndpoint)：调用 `parse_remote_endpoint_properties()` 函数解析媒体端点 (MediaEndpoint) 的各个属性。
     - 其它：忽略。
3. 为各个蓝牙设备 (Device) 设置蓝牙适配器/控制器 (Adapter)，即建立蓝牙设备 (Device) 与蓝牙适配器/控制器 (Adapter) 的关联。

不同蓝牙设备的属性不同，`parse_adapter_properties()` 函数解析获得的蓝牙适配器/控制器 (Adapter) 的属性如下：

 * 蓝牙适配器/控制器 (Adapter) 地址 (Address)：如 **28:D0:43:9A:CB:EF**。
 * UUID 数组：UUID 如 **0000110e-0000-1000-8000-00805f9b34fb**。蓝牙子系统中，会用 UUID 标识支持的特性。这里的 UUID 数组描述蓝牙适配器/控制器 (Adapter) 支持的特性。

`parse_device_properties()` 函数解析获得的蓝牙设备 (Device) 的属性如下：

 * 别名 (Alias)：如苹果 AirPods 蓝牙耳机的 **AirPods Pro**。
 * 蓝牙设备 (Device) 的地址 (Address)：如 **24:D0:DF:96:AD:6E**。
 * 蓝牙适配器/控制器 (Adapter)：**路径**形式的蓝牙适配器/控制器描述，如 **/org/bluez/hci0**。用于与所属的蓝牙适配器/控制器 (Adapter) 关联。
 * 类别 (Class)：如 **2360344**。
 * UUID 数组：描述蓝牙设备 (Device) 支持的特性。

`parse_remote_endpoint_properties()` 函数解析获得的媒体端点 (MediaEndpoint) 的属性如下：

 * UUID：描述媒体端点 (MediaEndpoint) 的类型特性，如 **PA_BLUETOOTH_UUID_A2DP_SINK** 或 **PA_BLUETOOTH_UUID_A2DP_SOURCE**。
 * Codec ID (Codec)
 * Capabilities
 * 蓝牙设备路径 (Device)：如 **/org/bluez/hci0/dev_24_D0_DF_96_AD_6E**，用于与所属的蓝牙设备 (Device) 关联。

在 `pa_bluetooth_discovery_get()` 函数之后，蓝牙设备状态的更新维护动作，由 `filter_cb()` 回调触发，对于不同的事件，执行不同的更新维护动作。`filter_cb()` 回调函数定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
static DBusHandlerResult filter_cb(DBusConnection *bus, DBusMessage *m, void *userdata) {
    pa_bluetooth_discovery *y;
    DBusError err;

    pa_assert(bus);
    pa_assert(m);
    pa_assert_se(y = userdata);

    dbus_error_init(&err);

    if (dbus_message_is_signal(m, DBUS_INTERFACE_DBUS, "NameOwnerChanged")) {
        const char *name, *old_owner, *new_owner;

        if (!dbus_message_get_args(m, &err,
                                   DBUS_TYPE_STRING, &name,
                                   DBUS_TYPE_STRING, &old_owner,
                                   DBUS_TYPE_STRING, &new_owner,
                                   DBUS_TYPE_INVALID)) {
            pa_log_error("Failed to parse " DBUS_INTERFACE_DBUS ".NameOwnerChanged: %s", err.message);
            goto fail;
        }

        if (pa_streq(name, BLUEZ_SERVICE)) {
            if (old_owner && *old_owner) {
                pa_log_debug("Bluetooth daemon disappeared");
                pa_hashmap_remove_all(y->devices);
                pa_hashmap_remove_all(y->adapters);
                y->objects_listed = false;
                if (y->ofono_backend) {
                    pa_bluetooth_ofono_backend_free(y->ofono_backend);
                    y->ofono_backend = NULL;
                }
                if (y->native_backend) {
                    pa_bluetooth_native_backend_free(y->native_backend);
                    y->native_backend = NULL;
                }
            }

            if (new_owner && *new_owner) {
                pa_log_debug("Bluetooth daemon appeared");
                get_managed_objects(y);
            }
        }

        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
    } else if (dbus_message_is_signal(m, DBUS_INTERFACE_OBJECT_MANAGER, "InterfacesAdded")) {
        DBusMessageIter arg_i;

        if (!y->objects_listed)
            return DBUS_HANDLER_RESULT_NOT_YET_HANDLED; /* No reply received yet from GetManagedObjects */

        if (!dbus_message_iter_init(m, &arg_i) || !pa_streq(dbus_message_get_signature(m), "oa{sa{sv}}")) {
            pa_log_error("Invalid signature found in InterfacesAdded");
            goto fail;
        }

        parse_interfaces_and_properties(y, &arg_i);

        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
    } else if (dbus_message_is_signal(m, DBUS_INTERFACE_OBJECT_MANAGER, "InterfacesRemoved")) {
        const char *p;
        DBusMessageIter arg_i;
        DBusMessageIter element_i;

        if (!y->objects_listed)
            return DBUS_HANDLER_RESULT_NOT_YET_HANDLED; /* No reply received yet from GetManagedObjects */

        if (!dbus_message_iter_init(m, &arg_i) || !pa_streq(dbus_message_get_signature(m), "oas")) {
            pa_log_error("Invalid signature found in InterfacesRemoved");
            goto fail;
        }

        dbus_message_iter_get_basic(&arg_i, &p);

        pa_assert_se(dbus_message_iter_next(&arg_i));
        pa_assert(dbus_message_iter_get_arg_type(&arg_i) == DBUS_TYPE_ARRAY);

        dbus_message_iter_recurse(&arg_i, &element_i);

        while (dbus_message_iter_get_arg_type(&element_i) == DBUS_TYPE_STRING) {
            const char *iface;

            dbus_message_iter_get_basic(&element_i, &iface);

            if (pa_streq(iface, BLUEZ_DEVICE_INTERFACE))
                device_remove(y, p);
            else if (pa_streq(iface, BLUEZ_ADAPTER_INTERFACE))
                adapter_remove(y, p);
            else if (pa_streq(iface, BLUEZ_MEDIA_ENDPOINT_INTERFACE))
                remote_endpoint_remove(y, p);

            dbus_message_iter_next(&element_i);
        }

        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

    } else if (dbus_message_is_signal(m, DBUS_INTERFACE_PROPERTIES, "PropertiesChanged")) {
        DBusMessageIter arg_i;
        const char *iface;

        if (!y->objects_listed)
            return DBUS_HANDLER_RESULT_NOT_YET_HANDLED; /* No reply received yet from GetManagedObjects */

        if (!dbus_message_iter_init(m, &arg_i) || !pa_streq(dbus_message_get_signature(m), "sa{sv}as")) {
            pa_log_error("Invalid signature found in PropertiesChanged");
            goto fail;
        }

        dbus_message_iter_get_basic(&arg_i, &iface);

        pa_assert_se(dbus_message_iter_next(&arg_i));
        pa_assert(dbus_message_iter_get_arg_type(&arg_i) == DBUS_TYPE_ARRAY);

        if (pa_streq(iface, BLUEZ_ADAPTER_INTERFACE)) {
            pa_bluetooth_adapter *a;

            pa_log_debug("Properties changed in adapter %s", dbus_message_get_path(m));

            if (!(a = pa_hashmap_get(y->adapters, dbus_message_get_path(m)))) {
                pa_log_warn("Properties changed in unknown adapter");
                return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
            }

            parse_adapter_properties(a, &arg_i, true);

        } else if (pa_streq(iface, BLUEZ_DEVICE_INTERFACE)) {
            pa_bluetooth_device *d;

            pa_log_debug("Properties changed in device %s", dbus_message_get_path(m));

            if (!(d = pa_hashmap_get(y->devices, dbus_message_get_path(m)))) {
                pa_log_warn("Properties changed in unknown device");
                return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
            }

            if (!d->properties_received)
                return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

            parse_device_properties(d, &arg_i);
        } else if (pa_streq(iface, BLUEZ_MEDIA_TRANSPORT_INTERFACE)) {
            pa_bluetooth_transport *t;

            pa_log_debug("Properties changed in transport %s", dbus_message_get_path(m));

            if (!(t = pa_hashmap_get(y->transports, dbus_message_get_path(m))))
                return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

            parse_transport_properties(t, &arg_i);
        } else if (pa_streq(iface, BLUEZ_MEDIA_ENDPOINT_INTERFACE)) {
            pa_log_info("Properties changed in remote endpoint %s", dbus_message_get_path(m));

            parse_remote_endpoint_properties(y, dbus_message_get_path(m), &arg_i);
        }

        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
    }

fail:
    dbus_error_free(&err);

    return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
}
```

这主要分为下面几种情况：

 * 系统蓝牙服务 bluez 停止或恢复：消息由 DBUS 发出。 系统蓝牙服务 bluez 停止时，`pa_bluetooth_discovery` 的所有蓝牙适配器/控制器 (Adapter) 和蓝牙设备 (Device) 被销毁，销毁蓝牙后端对象。系统蓝牙服务 bluez 恢复时，再次调用 `get_managed_objects()` 函数重新获取蓝牙音频设备有关信息，重走一遍上面 `get_managed_objects_reply()` 的流程。
 * 收到 **InterfacesAdded** 消息：消息中包含添加的特定蓝牙设备路径的**接口**和**属性**信息，再次调用 `parse_interfaces_and_properties()` 函数解析特定路径的信息。
 * 收到 **InterfacesRemoved** 消息：消息中包含移除的特定蓝牙设备路径的**接口**的信息，根据具体的**接口**信息，移除蓝牙适配器/控制器 (Adapter)、蓝牙设备 (Device) 或媒体端点 (MediaEndpoint)。
 * 收到 **PropertiesChanged** 消息：消息中包含改变的特定蓝牙设备路径的**接口**的**属性**信息，根据具体的**接口**类型，分别进行处理：
     * 蓝牙适配器/控制器 (Adapter)：检查对应路径的蓝牙适配器/控制器 (Adapter) 是否存在，存在时，再次调用 `parse_adapter_properties()` 函数解析蓝牙适配器/控制器 (Adapter) 属性。
     * 蓝牙设备 (Device)：检查对应路径的蓝牙设备 (Device) 是否存在，存在时，再次调用 `parse_device_properties()` 函数解析蓝牙设备 (Device) 属性。
     * 媒体端点 (MediaEndpoint)：再次调用 `parse_remote_endpoint_properties()` 函数解析媒体端点 (MediaEndpoint) 属性。



`parse_interfaces_and_properties()` 函数

其中的蓝牙设备相关信息。

## 蓝牙音频设备连接管理
音频蓝牙协议栈主要有 HSP/HFP 和 A2DP/AVRCP 两种，这两种协议栈在系统蓝牙服务 bluez 和音频服务 pulseaudio 中具有不同的处理流程。通常一个蓝牙耳麦对这两种协议栈都支持，并可以在这两种协议栈之间进行切换。

对于 A2DP/AVRCP 协议栈，在 `pa_bluetooth_discovery_get()` 函数的执行过程中，会初始化蓝牙 A2DP endpoint 配置，并向 D-BUS 注册 endpoint 处理程序 `endpoint_handler()`。蓝牙音频设备在物理上与蓝牙控制器/适配器建立连接之后，系统蓝牙服务 bluez 会收到通知，在回调中处理连接，并通过 D-BUS 调用 pulseaudio 注册的 endpoint 处理程序 `endpoint_handler()`。

对于 HSP/HFP 协议栈，主要通过蓝牙后端，即蓝牙 headset 后端处理。 `get_managed_objects_reply()` 函数执行的最后，创建蓝牙后端对象。pulseaudio 支持两种蓝牙后端，分别为 native 和 ofono，native 后端直接与 BlueZ 交互；ofono 后端通过 ofono（一个开源的电话栈）实现蓝牙音频支持，主要用于支持电话功能（如 HFP），pulseaudio 通常使用 native 蓝牙后端。`get_managed_objects_reply()` 调用 `pa_bluetooth_native_backend_new()` 函数创建 native 蓝牙后端对象，这个函数定义 (位于 *pulseaudio/src/modules/bluetooth/backend-native.c*) 如下：
```
struct pa_bluetooth_backend {
  pa_core *core;
  pa_dbus_connection *connection;
  pa_bluetooth_discovery *discovery;
  pa_hook_slot *adapter_uuids_changed_slot;
  bool enable_shared_profiles;
  bool enable_hsp_hs;
  bool enable_hfp_hf;

  PA_LLIST_HEAD(pa_dbus_pending, pending);
};
 . . . . . .
static pa_dbus_pending* send_and_add_to_pending(pa_bluetooth_backend *backend, DBusMessage *m,
        DBusPendingCallNotifyFunction func, void *call_data) {

    pa_dbus_pending *p;
    DBusPendingCall *call;

    pa_assert(backend);
    pa_assert(m);

    pa_assert_se(dbus_connection_send_with_reply(pa_dbus_connection_get(backend->connection), m, &call, -1));

    p = pa_dbus_pending_new(pa_dbus_connection_get(backend->connection), m, call, backend, call_data);
    PA_LLIST_PREPEND(pa_dbus_pending, backend->pending, p);
    dbus_pending_call_set_notify(call, func, p, NULL);

    return p;
}
 . . . . . .
static void register_profile_reply(DBusPendingCall *pending, void *userdata) {
    DBusMessage *r;
    pa_dbus_pending *p;
    pa_bluetooth_backend *b;
    pa_bluetooth_profile_t profile;

    pa_assert(pending);
    pa_assert_se(p = userdata);
    pa_assert_se(b = p->context_data);
    pa_assert_se(profile = (pa_bluetooth_profile_t)p->call_data);
    pa_assert_se(r = dbus_pending_call_steal_reply(pending));

    if (dbus_message_is_error(r, BLUEZ_ERROR_NOT_SUPPORTED)) {
        pa_log_info("Couldn't register profile %s because it is disabled in BlueZ", pa_bluetooth_profile_to_string(profile));
        profile_status_set(b->discovery, profile, PA_BLUETOOTH_PROFILE_STATUS_ACTIVE);
        goto finish;
    }

    if (dbus_message_get_type(r) == DBUS_MESSAGE_TYPE_ERROR) {
        pa_log_error(BLUEZ_PROFILE_MANAGER_INTERFACE ".RegisterProfile() failed: %s: %s", dbus_message_get_error_name(r),
                     pa_dbus_get_error_message(r));
        profile_status_set(b->discovery, profile, PA_BLUETOOTH_PROFILE_STATUS_ACTIVE);
        goto finish;
    }

    profile_status_set(b->discovery, profile, PA_BLUETOOTH_PROFILE_STATUS_REGISTERED);

finish:
    dbus_message_unref(r);

    PA_LLIST_REMOVE(pa_dbus_pending, b->pending, p);
    pa_dbus_pending_free(p);
}

static void register_profile(pa_bluetooth_backend *b, const char *object, const char *uuid, pa_bluetooth_profile_t profile) {
    DBusMessage *m;
    DBusMessageIter i, d;
    dbus_bool_t autoconnect;
    dbus_uint16_t version, chan;

    pa_assert(profile_status_get(b->discovery, profile) == PA_BLUETOOTH_PROFILE_STATUS_ACTIVE);

    pa_log_debug("Registering Profile %s %s", pa_bluetooth_profile_to_string(profile), uuid);

    pa_assert_se(m = dbus_message_new_method_call(BLUEZ_SERVICE, "/org/bluez", BLUEZ_PROFILE_MANAGER_INTERFACE, "RegisterProfile"));

    dbus_message_iter_init_append(m, &i);
    pa_assert_se(dbus_message_iter_append_basic(&i, DBUS_TYPE_OBJECT_PATH, &object));
    pa_assert_se(dbus_message_iter_append_basic(&i, DBUS_TYPE_STRING, &uuid));
    dbus_message_iter_open_container(&i, DBUS_TYPE_ARRAY,
                                     DBUS_DICT_ENTRY_BEGIN_CHAR_AS_STRING
                                     DBUS_TYPE_STRING_AS_STRING
                                     DBUS_TYPE_VARIANT_AS_STRING
                                     DBUS_DICT_ENTRY_END_CHAR_AS_STRING,
                                     &d);
    if (pa_bluetooth_uuid_is_hsp_hs(uuid)) {
        /* In the headset role, the connection will only be initiated from the remote side */
        autoconnect = 0;
        pa_dbus_append_basic_variant_dict_entry(&d, "AutoConnect", DBUS_TYPE_BOOLEAN, &autoconnect);
        chan = HSP_HS_DEFAULT_CHANNEL;
        pa_dbus_append_basic_variant_dict_entry(&d, "Channel", DBUS_TYPE_UINT16, &chan);
        /* HSP version 1.2 */
        version = 0x0102;
        pa_dbus_append_basic_variant_dict_entry(&d, "Version", DBUS_TYPE_UINT16, &version);
    }
    dbus_message_iter_close_container(&i, &d);

    profile_status_set(b->discovery, profile, PA_BLUETOOTH_PROFILE_STATUS_REGISTERING);
    send_and_add_to_pending(b, m, register_profile_reply, (void *)profile);
}
 . . . . . .
static void profile_init(pa_bluetooth_backend *b, pa_bluetooth_profile_t profile) {
    static const DBusObjectPathVTable vtable_profile = {
        .message_function = profile_handler,
    };
    const char *object_name;
    const char *uuid;

    pa_assert(b);

    switch (profile) {
        case PA_BLUETOOTH_PROFILE_HSP_HS:
            object_name = HSP_AG_PROFILE;
            uuid = PA_BLUETOOTH_UUID_HSP_AG;
            break;
        case PA_BLUETOOTH_PROFILE_HSP_AG:
            object_name = HSP_HS_PROFILE;
            uuid = PA_BLUETOOTH_UUID_HSP_HS;
            break;
        case PA_BLUETOOTH_PROFILE_HFP_HF:
            object_name = HFP_AG_PROFILE;
            uuid = PA_BLUETOOTH_UUID_HFP_AG;
            break;
        default:
            pa_assert_not_reached();
            break;
    }

    pa_assert_se(dbus_connection_register_object_path(pa_dbus_connection_get(b->connection), object_name, &vtable_profile, b));

    profile_status_set(b->discovery, profile, PA_BLUETOOTH_PROFILE_STATUS_ACTIVE);
    register_profile(b, object_name, uuid, profile);
}
 . . . . . .
static void native_backend_apply_profile_registration_change(pa_bluetooth_backend *native_backend, bool enable_shared_profiles) {
    if (enable_shared_profiles) {
        profile_init(native_backend, PA_BLUETOOTH_PROFILE_HSP_AG);
        if (native_backend->enable_hfp_hf)
            profile_init(native_backend, PA_BLUETOOTH_PROFILE_HFP_HF);
    } else {
        profile_done(native_backend, PA_BLUETOOTH_PROFILE_HSP_AG);
        if (native_backend->enable_hfp_hf)
            profile_done(native_backend, PA_BLUETOOTH_PROFILE_HFP_HF);
    }
}
 . . . . . .
 pa_bluetooth_backend *pa_bluetooth_native_backend_new(pa_core *c, pa_bluetooth_discovery *y, bool enable_shared_profiles) {
    pa_bluetooth_backend *backend;
    DBusError err;

    pa_log_debug("Bluetooth Headset Backend API support using the native backend");

    backend = pa_xnew0(pa_bluetooth_backend, 1);
    backend->core = c;

    dbus_error_init(&err);
    if (!(backend->connection = pa_dbus_bus_get(c, DBUS_BUS_SYSTEM, &err))) {
        pa_log("Failed to get D-Bus connection: %s", err.message);
        dbus_error_free(&err);
        pa_xfree(backend);
        return NULL;
    }

    backend->discovery = y;
    backend->enable_shared_profiles = enable_shared_profiles;
    backend->enable_hfp_hf = pa_bluetooth_discovery_get_enable_native_hfp_hf(y);
    backend->enable_hsp_hs = pa_bluetooth_discovery_get_enable_native_hsp_hs(y);

    backend->adapter_uuids_changed_slot =
        pa_hook_connect(pa_bluetooth_discovery_hook(y, PA_BLUETOOTH_HOOK_ADAPTER_UUIDS_CHANGED), PA_HOOK_NORMAL,
                        (pa_hook_cb_t) adapter_uuids_changed_cb, backend);

    if (!backend->enable_hsp_hs && !backend->enable_hfp_hf)
        pa_log_warn("Both HSP HS and HFP HF bluetooth profiles disabled in native backend. Native backend will not register for headset connections.");

    if (backend->enable_hsp_hs)
        profile_init(backend, PA_BLUETOOTH_PROFILE_HSP_HS);

    if (backend->enable_shared_profiles)
        native_backend_apply_profile_registration_change(backend, true);

    return backend;
}
```

pulseaudio 中用 `pa_bluetooth_backend` 对象描述蓝牙后端对象，`pa_bluetooth_native_backend_new()` 函数创建 `pa_bluetooth_backend` 对象，注册蓝牙控制器/适配器 UUID 改变处理程序，并根据配置初始化不同的 profile。配置具体指 `enable_hsp_hs`、`enable_hfp_hf` 和 `enable_shared_profiles`，它们主要来源于 **module-bluez5-discover** 模块的模块参数。

HSP 是蓝牙协议栈中最早的耳机配置文件，主要用于支持蓝牙耳机和手机之间的基本音频通信，它支持单声道音频传输（通常用于语音通话），和基本的远程控制功能（如接听/挂断电话），主要用于简单的蓝牙耳机，功能较为基础。HFP 是 HSP 的升级版，提供了更丰富的功能，主要用于支持免提设备（如车载蓝牙系统）与手机之间的通信，它支持单声道音频传输（语音通话），更复杂的远程控制功能（如音量控制、重拨、拒接电话等），电话状态信息（如电池电量、信号强度等），及多方通话和语音识别，主要用于车载蓝牙系统、高端蓝牙耳机等。

`enable_hfp_hf` 用于配置是否开启 HFP，默认为 `ture`，`enable_hsp_hs` 用于配置是否开启 HSP，默认为 `false`，`enable_shared_profiles` 根据 headset 蓝牙后端的配置得到，headset 蓝牙后端为 native 时为 `ture`，否则为 `false`，默认为 `ture`。默认初始化的 profile 如下：
```
(4480.011|   0.000) D: [pulseaudio] backend-native.c: Registering Profile headset_audio_gateway 00001108-0000-1000-8000-00805f9b34fb
(4480.011|   0.000) D: [pulseaudio] backend-native.c: Registering Profile handsfree_head_unit 0000111f-0000-1000-8000-00805f9b34fb
```

即初始化 profile `PA_BLUETOOTH_PROFILE_HSP_AG` 和 `PA_BLUETOOTH_PROFILE_HFP_HF`。

`dbus_connection_register_object_path()` 是 D-Bus 库的一个函数，用于向 D-Bus 连接注册一个对象路径。这个对象路径表示一个在 D-Bus 总线上可以被其它应用程序调用的方法、信号或者属性。通过注册对象路径，可以定义应用程序在 D-Bus 上的接口，使其可以被其它应用程序访问和调用。初始化 profile 由 `profile_init()` 函数处理，主要是为 profile 注册对应的对象路径，并通过 D-BUS 向蓝牙服务 BlueZ 注册 profile。不同 profile 的对象路径不同，它们的对应关系如下：
 * **PA_BLUETOOTH_PROFILE_HSP_HS** ：**HSP_AG_PROFILE** (**/Profile/HSPAGProfile**)
 * **PA_BLUETOOTH_PROFILE_HSP_AG** ：**HSP_HS_PROFILE** (**/Profile/HSPHSProfile**)
 * **PA_BLUETOOTH_PROFILE_HFP_HF** ：**HSP_AG_PROFILE** (**/Profile/HFPAGProfile**)

对象路径上的处理程序为 `profile_handler()` 函数。向蓝牙服务 BlueZ 注册 profile，将调用 BlueZ 注册的 **org.bluez.ProfileManager1** 接口的 **RegisterProfile** 方法。

对于 HFP/HSP 协议栈，蓝牙音频设备在物理上与蓝牙控制器/适配器建立连接之后，系统蓝牙服务 bluez 会收到通知，在回调中处理连接，并通过 D-BUS 调用 pulseaudio 注册的 profile 的 D-BUS 对象路径上的处理程序 `profile_handler()`。

下面是蓝牙耳机连上蓝牙服务 BlueZ 时的一段日志：
```
2月 17 14:41:45 workstation bluetoothd[17672]: src/adapter.c:connected_callback() hci0 device 24:D0:DF:96:AD:6E connected eir_len 18
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avctp.c:avctp_confirm_cb() AVCTP: incoming connect from 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avctp.c:avctp_set_state() AVCTP Connecting
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avctp.c:avctp_connect_cb() AVCTP: connected to 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avctp.c:init_uinput() AVRCP: uinput initialized for AirPods Pro
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avrcp.c:controller_init() 0xaaab0f5d2f90 version 0x0105
2月 17 14:41:46 workstation bluetoothd[17672]: src/service.c:change_state() 0xaaab0f5bf950: device 24:D0:DF:96:AD:6E profile audio-avrcp-target state changed: disconnected -> connected (0)
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avrcp.c:target_init() 0xaaab0f5e00e0 version 0x0105
2月 17 14:41:46 workstation bluetoothd[17672]: src/service.c:change_state() 0xaaab0f5cafb0: device 24:D0:DF:96:AD:6E profile avrcp-controller state changed: disconnected -> connected (0)
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avctp.c:avctp_set_state() AVCTP Connected
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avrcp.c:handle_vendordep_pdu() AVRCP PDU 0x10, company 0x001958 len 0x0001
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avrcp.c:avrcp_handle_get_capabilities() id=3
2月 17 14:41:46 workstation bluetoothd[17672]: src/profile.c:ext_confirm() incoming connect from 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: src/service.c:btd_service_ref() 0xaaab0f5db3e0: ref=3
2月 17 14:41:46 workstation bluetoothd[17672]: src/profile.c:ext_confirm() Hands-Free Voice gateway authorizing connection from 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: src/profile.c:ext_auth() 24:D0:DF:96:AD:6E authorized to connect to Hands-Free Voice gateway
2月 17 14:41:46 workstation bluetoothd[17672]: src/profile.c:ext_connect() Hands-Free Voice gateway connected to 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: src/service.c:change_state() 0xaaab0f5db3e0: device 24:D0:DF:96:AD:6E profile Hands-Free Voice gateway state changed: disconnected -> connecting (0)
2月 17 14:41:46 workstation bluetoothd[17672]: src/service.c:change_state() 0xaaab0f5db3e0: device 24:D0:DF:96:AD:6E profile Hands-Free Voice gateway state changed: connecting -> connected (0)
2月 17 14:41:46 workstation bluetoothd[17672]: src/device.c:device_profile_connected() Hands-Free Voice gateway Success (0)
2月 17 14:41:46 workstation bluetoothd[17672]: plugins/policy.c:service_cb() Added Hands-Free Voice gateway reconnect 1
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avrcp.c:handle_vendordep_pdu() AVRCP PDU 0x31, company 0x001958 len 0x0005
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:confirm_cb() AVDTP: incoming connect from 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/sink.c:sink_set_state() State changed /org/bluez/hci0/dev_24_D0_DF_96_AD_6E: SINK_STATE_DISCONNECTED -> SINK_STATE_CONNECTING
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_connect_cb() AVDTP: connected signaling channel to 24:D0:DF:96:AD:6E
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_connect_cb() AVDTP imtu=672, omtu=1023
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_register_remote_sep() seid 1 type 1 media 0 delay_reporting true
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:register_remote_sep() Found remote SEP: /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/sep1
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_register_remote_sep() seid 2 type 1 media 0 delay_reporting true
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:register_remote_sep() Found remote SEP: /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/sep2
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_register_remote_sep() seid 3 type 1 media 0 delay_reporting true
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:register_remote_sep() Found remote SEP: /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/sep3
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:load_remote_sep() LastUsed: lseid 4 rseid 1
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_ref() 0xaaab0f5e7c20: ref=1
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:set_disconnect_timer() timeout 1
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received DISCOVER_CMD
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5b0030: Get_Capability_Ind
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5ce470: Get_Capability_Ind
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:46 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5d11e0: Get_Capability_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5d1480: Get_Capability_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5c0720: Get_Capability_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5c0ac0: Get_Capability_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5c0f20: Get_Capability_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received  GET_ALL_CAPABILITIES_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_getcap_ind() Source 0xaaab0f5c1380: Get_Capability_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received SET_CONFIGURATION_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_setconf_ind() Source 0xaaab0f5c1380: Set_Configuration_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_ref() 0xaaab0f5e7c20: ref=2
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_unref() 0xaaab0f5e7c20: ref=1
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_ref() 0xaaab0f5e2220: ref=1
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_ref() 0xaaab0f5e2220: ref=2
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/media.c:media_endpoint_async_call() Calling SetConfiguration: name = :1.146 path = /MediaEndpoint/A2DPSource/sbc_xq_552
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_ref() 0xaaab0f5e7c20: ref=2
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_sep_set_state() stream state changed: IDLE -> CONFIGURED
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_unref() 0xaaab0f5e2220: ref=1
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_unref() 0xaaab0f5e2220: ref=0
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_free() 0xaaab0f5e2220
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_unref() 0xaaab0f5e7c20: ref=1
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/transport.c:media_owner_create() Owner created: sender=:1.146
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_ref() 0xaaab0f5e7c20: ref=2
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:a2dp_sep_lock() SEP 0xaaab0f5c1380 locked
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_ref() 0xaaab0f5e7c20: ref=3
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_ref() 0xaaab0f5e2220: ref=1
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/transport.c:transport_set_state() State changed /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2: TRANSPORT_STATE_IDLE -> TRANSPORT_STATE_REQUESTING
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/transport.c:media_request_create() Request created: method=Acquire id=4
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/transport.c:media_owner_add() Owner :1.146 Request Acquire
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/transport.c:media_transport_set_owner() Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 Owner :1.146
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received DELAY_REPORT_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:endpoint_delayreport_ind() Source 0xaaab0f5c1380: DelayReport_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_cmd() Received OPEN_CMD
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:open_ind() Source 0xaaab0f5c1380: Open_Ind
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_ref() 0xaaab0f5e2220: ref=2
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:confirm_cb() AVDTP: incoming connect from 24:D0:DF:96:AD:6E
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:handle_transport_connect() Flushable packets enabled
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:handle_transport_connect() sk 41, omtu 1023, send buffer size 106496
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_sep_set_state() stream state changed: CONFIGURED -> OPEN
2月 17 14:41:47 workstation bluetoothd[17672]: src/service.c:change_state() 0xaaab0f5c7b90: device 24:D0:DF:96:AD:6E profile a2dp-sink state changed: disconnected -> connected (0)
2月 17 14:41:47 workstation bluetoothd[17672]: plugins/policy.c:service_cb() Added a2dp-sink reconnect 1
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/sink.c:sink_set_state() State changed /org/bluez/hci0/dev_24_D0_DF_96_AD_6E: SINK_STATE_CONNECTING -> SINK_STATE_CONNECTED
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/transport.c:transport_update_playing() /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 State=TRANSPORT_STATE_REQUESTING Playing=0
2月 17 14:41:47 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_unref() 0xaaab0f5e2220: ref=1
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/avdtp.c:session_cb()
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_parse_resp() START request succeeded
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/a2dp.c:start_cfm() Source 0xaaab0f5c1380: Start_Cfm
2月 17 14:41:48 workstation bluetoothd[17672]: /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2: fd(41) ready
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/transport.c:media_owner_remove() Owner :1.146 Request Acquire
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/transport.c:transport_set_state() State changed /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2: TRANSPORT_STATE_REQUESTING -> TRANSPORT_STATE_ACTIVE
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_unref() 0xaaab0f5e2220: ref=0
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/a2dp.c:setup_free() 0xaaab0f5e2220
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_unref() 0xaaab0f5e7c20: ref=2
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/avdtp.c:avdtp_sep_set_state() stream state changed: OPEN -> STREAMING
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/sink.c:sink_set_state() State changed /org/bluez/hci0/dev_24_D0_DF_96_AD_6E: SINK_STATE_CONNECTED -> SINK_STATE_PLAYING
2月 17 14:41:48 workstation bluetoothd[17672]: profiles/audio/transport.c:transport_update_playing() /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 State=TRANSPORT_STATE_ACTIVE Playing=1
```

蓝牙耳机连上蓝牙适配器/控制器之后，蓝牙服务 BlueZ 首先连接 AVRCP profile，然后连接 HFP profile，之后连接 A2DP profile，随后处理 HFP 和 A2DP 的传输配置。下面是同一时刻 pulseaudio 的日志：
```
( 492.455|  19.611) D: [pulseaudio] bluez5-util.c: Properties changed in device /org/bluez/hci0/dev_24_D0_DF_96_AD_6E
( 492.786|   0.331) D: [pulseaudio] backend-native.c: dbus: path=/Profile/HFPAGProfile, interface=org.bluez.Profile1, member=NewConnection
( 492.786|   0.000) D: [pulseaudio] backend-native.c: dbus: NewConnection path=/org/bluez/hci0/dev_24_D0_DF_96_AD_6E, fd=46, profile handsfree_head_unit
( 492.786|   0.000) I: [pulseaudio] backend-native.c: doing listen
( 492.823|   0.036) D: [pulseaudio] backend-native.c: RFCOMM << AT+BRSF=667
( 492.823|   0.000) I: [pulseaudio] backend-native.c: HFP capabilities returns 0x29b
( 492.823|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> +BRSF: 1600
( 492.823|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 492.886|   0.062) D: [pulseaudio] backend-native.c: RFCOMM << AT+BAC=1,2,128,256
( 492.886|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 492.892|   0.006) D: [pulseaudio] backend-native.c: RFCOMM << AT+CIND=?
( 492.892|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> +CIND: ("service",(0-1)),("call",(0-1)),("callsetup",(0-3)),("callheld",(0-2))
( 492.892|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 492.899|   0.007) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
( 492.899|   0.000) D: [pulseaudio] bluez5-util.c: Parsing remote endpoint /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/sep1
( 492.899|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Properties found, skipping
( 492.900|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
( 492.900|   0.000) D: [pulseaudio] bluez5-util.c: Parsing remote endpoint /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/sep2
( 492.900|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Properties found, skipping
( 492.900|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
( 492.900|   0.000) D: [pulseaudio] bluez5-util.c: Parsing remote endpoint /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/sep3
( 492.900|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Properties found, skipping
( 492.926|   0.025) D: [pulseaudio] backend-native.c: RFCOMM << AT+CIND?
( 492.926|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> +CIND: 0,0,0,0
( 492.926|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 492.934|   0.008) D: [pulseaudio] backend-native.c: RFCOMM << AT+CMER=3,0,0,1
( 492.934|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 492.934|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> +BCS:2
( 492.943|   0.008) D: [pulseaudio] backend-native.c: RFCOMM << AT+VGS=15
( 492.943|   0.000) D: [pulseaudio] backend-native.c: HS/HF peer supports speaker gain control
( 492.943|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 493.303|   0.360) D: [pulseaudio] backend-native.c: RFCOMM << AT+VGM=15
( 493.303|   0.000) D: [pulseaudio] backend-native.c: HS/HF peer supports microphone gain control
( 493.303|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 493.756|   0.452) D: [pulseaudio] backend-native.c: RFCOMM << AT+BIA=0,1,1,1
( 493.756|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 493.787|   0.031) D: [pulseaudio] backend-native.c: RFCOMM << AT+NREC=0
( 493.787|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 493.817|   0.030) D: [pulseaudio] backend-native.c: RFCOMM << AT+BCS=2
( 493.817|   0.000) I: [pulseaudio] backend-native.c: HFP negotiated codec mSBC
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd46 state: disconnected -> idle
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_sink: true
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_source: false
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile headset_head_unit: false
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile headset_audio_gateway: false
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_head_unit: true
( 493.817|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_audio_gateway: false
( 493.817|   0.000) D: [pulseaudio] backend-native.c: Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd46 available for profile handsfree_head_unit
( 493.817|   0.000) D: [pulseaudio] backend-native.c: RFCOMM >> OK
( 493.885|   0.067) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Introspectable found, skipping
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.bluez.MediaTransport1 found, skipping
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Unknown interface org.freedesktop.DBus.Properties found, skipping
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: dbus: path=/MediaEndpoint/A2DPSource/sbc_xq_552, interface=org.bluez.MediaEndpoint1, member=SetConfiguration
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 state: disconnected -> idle
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_sink: true
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_source: false
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile headset_head_unit: false
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile headset_audio_gateway: false
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_head_unit: true
( 493.885|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_audio_gateway: false
( 493.885|   0.000) D: [pulseaudio] module-bluez5-discover.c: Loading module-bluez5-device path=/org/bluez/hci0/dev_24_D0_DF_96_AD_6E autodetect_mtu=0 output_rate_refresh_interval_ms=500 avrcp_absolute_volume=1
( 493.886|   0.000) D: [pulseaudio] module-bluez5-device.c: Trying to create profile a2dp_sink (0000110b-0000-1000-8000-00805f9b34fb) for device AirPods Pro (24:D0:DF:96:AD:6E)
( 493.886|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_sink: true
( 493.886|   0.000) D: [pulseaudio] module-bluez5-device.c: Trying to create profile handsfree_head_unit (0000111e-0000-1000-8000-00805f9b34fb) for device AirPods Pro (24:D0:DF:96:AD:6E)
( 493.886|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_head_unit: true
( 493.886|   0.000) I: [pulseaudio] module-card-restore.c: Restoring port latency offsets for card bluez_card.24_D0_DF_96_AD_6E.
( 493.886|   0.000) D: [pulseaudio] card.c: Looking for initial profile for card bluez_card.24_D0_DF_96_AD_6E
( 493.886|   0.000) D: [pulseaudio] card.c: a2dp_sink availability unknown
( 493.886|   0.000) D: [pulseaudio] card.c: handsfree_head_unit availability unknown
( 493.886|   0.000) D: [pulseaudio] card.c: off availability yes
( 493.886|   0.000) I: [pulseaudio] card.c: bluez_card.24_D0_DF_96_AD_6E: active_profile: a2dp_sink
( 493.886|   0.000) I: [pulseaudio] card.c: Created 3 "bluez_card.24_D0_DF_96_AD_6E"
( 493.886|   0.000) D: [pulseaudio] module-bluez5-device.c: Acquiring transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2
( 495.419|   1.533) I: [pulseaudio] module-bluez5-device.c: Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 acquired: fd 48
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 state: idle -> playing
( 495.419|   0.000) D: [pulseaudio] card.c: Setting card bluez_card.24_D0_DF_96_AD_6E profile a2dp_sink to availability status yes
( 495.419|   0.000) D: [pulseaudio] core-subscribe.c: Dropped redundant event due to change event.
( 495.419|   0.000) D: [pulseaudio] device-port.c: Setting port headphone-output to status yes
( 495.419|   0.000) D: [pulseaudio] core-subscribe.c: Dropped redundant event due to change event.
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_sink: true
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile a2dp_source: false
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile headset_head_unit: false
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile headset_audio_gateway: false
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_head_unit: true
( 495.419|   0.000) D: [pulseaudio] bluez5-util.c: Checking if device AirPods Pro (24:D0:DF:96:AD:6E) supports profile handsfree_audio_gateway: false
( 495.419|   0.000) I: [pulseaudio] a2dp-codec-sbc.c: SBC parameters: allocation=Loudness, subbands=8, blocks=16, mode=DualChannel bitpool=53 codesize=512 frame_length=224
( 495.419|   0.000) I: [pulseaudio] module-device-restore.c: Restoring volume for sink bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink: front-left: 24770 /  38%,   front-right: 24770 /  38%
( 495.419|   0.000) I: [pulseaudio] sink.c: Created sink 4 "bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink" with sample spec s16le 2ch 44100Hz and channel map front-left,front-right
( 495.419|   0.000) I: [pulseaudio] sink.c:     bluetooth.protocol = "a2dp_sink"
( 495.419|   0.000) I: [pulseaudio] sink.c:     bluetooth.codec = "sbc_xq_552"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.description = "AirPods Pro"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.string = "24:D0:DF:96:AD:6E"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.api = "bluez"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.class = "sound"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.bus = "bluetooth"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.form_factor = "headphone"
( 495.419|   0.000) I: [pulseaudio] sink.c:     bluez.path = "/org/bluez/hci0/dev_24_D0_DF_96_AD_6E"
( 495.419|   0.000) I: [pulseaudio] sink.c:     bluez.class = "0x240418"
( 495.419|   0.000) I: [pulseaudio] sink.c:     bluez.alias = "AirPods Pro"
( 495.419|   0.000) I: [pulseaudio] sink.c:     device.icon_name = "audio-headphones-bluetooth"
( 495.419|   0.000) I: [pulseaudio] source.c: Created source 6 "bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink.monitor" with sample spec s16le 2ch 44100Hz and channel map front-left,front-right
( 495.419|   0.000) I: [pulseaudio] source.c:     device.description = "Monitor of AirPods Pro"
( 495.419|   0.000) I: [pulseaudio] source.c:     device.class = "monitor"
( 495.419|   0.000) I: [pulseaudio] source.c:     device.string = "24:D0:DF:96:AD:6E"
( 495.419|   0.000) I: [pulseaudio] source.c:     device.api = "bluez"
( 495.419|   0.000) I: [pulseaudio] source.c:     device.bus = "bluetooth"
( 495.419|   0.000) I: [pulseaudio] source.c:     device.form_factor = "headphone"
( 495.419|   0.000) I: [pulseaudio] source.c:     bluez.path = "/org/bluez/hci0/dev_24_D0_DF_96_AD_6E"
( 495.419|   0.000) I: [pulseaudio] source.c:     bluez.class = "0x240418"
( 495.419|   0.000) I: [pulseaudio] source.c:     bluez.alias = "AirPods Pro"
( 495.419|   0.000) I: [pulseaudio] source.c:     device.icon_name = "audio-headphones-bluetooth"
[New Thread 0xffffeb61efe0 (LWP 20106)]
( 495.420|   0.000) D: [bluetooth] module-bluez5-device.c: IO Thread starting up
( 495.424|   0.003) D: [bluetooth] util.c: RealtimeKit worked.
( 495.424|   0.000) I: [bluetooth] util.c: Successfully enabled SCHED_RR scheduling for thread, with priority 5.
( 495.424|   0.000) I: [bluetooth] module-bluez5-device.c: Transport /org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd2 resuming
( 495.424|   0.000) I: [bluetooth] module-bluez5-device.c: Changing bluetooth buffer size: Changed from 106496 to 4096
( 495.424|   0.000) D: [bluetooth] module-bluez5-device.c: Stream properly set up, we're ready to roll!
( 495.424|   0.000) D: [pulseaudio] sink.c: bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink: state: INIT -> IDLE
( 495.424|   0.000) D: [pulseaudio] source.c: bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink.monitor: state: INIT -> IDLE
( 495.424|   0.000) D: [pulseaudio] module-device-restore.c: Could not set format on sink bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink
( 495.424|   0.000) D: [pulseaudio] module-bluetooth-policy.c: Profile a2dp_sink cannot be selected for loopback
( 495.424|   0.000) D: [pulseaudio] module-suspend-on-idle.c: Sink bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink becomes idle, timeout in 5 seconds.
( 495.424|   0.000) I: [pulseaudio] core.c: default_sink: alsa_output.platform-MSNC0001_00.HiFi__hw_mtaport5672__sink -> bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink
( 495.424|   0.000) I: [pulseaudio] sink.c: The sink input 0 "QtPulseAudio:2236" is moving to bluez_sink.24_D0_DF_96_AD_6E.a2dp_sink due to change of the default sink.
```

pulseaudio 收到蓝牙耳机设备 D-BUS 路径上的属性变化，这主要用于 A2DP profile。

注册的 `profile_handler()` 回调被调用，来处理 HFP 连接传输。`profile_handler()` 函数定义 (位于 *pulseaudio/src/modules/bluetooth/backend-native.c*) 如下：
```
struct transport_data {
    int rfcomm_fd;
    pa_io_event *rfcomm_io;
    int sco_fd;
    pa_io_event *sco_io;
    pa_mainloop_api *mainloop;
};

struct hfp_config {
    uint32_t capabilities;
    int state;
    bool support_codec_negotiation;
    bool support_msbc;
    bool supports_indicators;
    int selected_codec;
};
 . . . . . .
static int sco_listen(pa_bluetooth_transport *t) {
    struct transport_data *trd = t->userdata;
    struct sockaddr_sco addr;
    int sock, i;
    bdaddr_t src;
    const char *src_addr;

    sock = socket(PF_BLUETOOTH, SOCK_SEQPACKET | SOCK_NONBLOCK | SOCK_CLOEXEC, BTPROTO_SCO);
    if (sock < 0) {
        pa_log_error("socket(SEQPACKET, SCO) %s", pa_cstrerror(errno));
        return -1;
    }

    src_addr = t->device->adapter->address;

    /* don't use ba2str to avoid -lbluetooth */
    for (i = 5; i >= 0; i--, src_addr += 3)
        src.b[i] = strtol(src_addr, NULL, 16);

    /* Bind to local address */
    memset(&addr, 0, sizeof(addr));
    addr.sco_family = AF_BLUETOOTH;
    bacpy(&addr.sco_bdaddr, &src);

    if (bind(sock, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
        pa_log_error("bind(): %s", pa_cstrerror(errno));
        goto fail_close;
    }

    pa_log_info ("doing listen");
    if (listen(sock, 1) < 0) {
        pa_log_error("listen(): %s", pa_cstrerror(errno));
        goto fail_close;
    }

    trd->sco_fd = sock;
    trd->sco_io = trd->mainloop->io_new(trd->mainloop, sock, PA_IO_EVENT_INPUT,
        sco_io_callback, t);

    return sock;

fail_close:
    close(sock);
    return -1;
}
 . . . . . .
static void transport_put(pa_bluetooth_transport *t)
{
    pa_bluetooth_transport_put(t);

    pa_log_debug("Transport %s available for profile %s", t->path, pa_bluetooth_profile_to_string(t->profile));
}
 . . . . . .
static DBusMessage *profile_new_connection(DBusConnection *conn, DBusMessage *m, void *userdata) {
    pa_bluetooth_backend *b = userdata;
    pa_bluetooth_device *d;
    pa_bluetooth_transport *t;
    pa_bluetooth_profile_t p;
    DBusMessage *r;
    int fd;
    const char *sender, *path, PA_UNUSED *handler;
    DBusMessageIter arg_i;
    char *pathfd;
    struct transport_data *trd;

    if (!dbus_message_iter_init(m, &arg_i) || !pa_streq(dbus_message_get_signature(m), "oha{sv}")) {
        pa_log_error("Invalid signature found in NewConnection");
        goto fail;
    }

    handler = dbus_message_get_path(m);
    if (pa_streq(handler, HSP_AG_PROFILE)) {
        p = PA_BLUETOOTH_PROFILE_HSP_HS;
    } else if (pa_streq(handler, HSP_HS_PROFILE)) {
        p = PA_BLUETOOTH_PROFILE_HSP_AG;
    } else if (pa_streq(handler, HFP_AG_PROFILE)) {
        p = PA_BLUETOOTH_PROFILE_HFP_HF;
    } else {
        pa_log_error("Invalid handler");
        goto fail;
    }

    pa_assert(dbus_message_iter_get_arg_type(&arg_i) == DBUS_TYPE_OBJECT_PATH);
    dbus_message_iter_get_basic(&arg_i, &path);

    d = pa_bluetooth_discovery_get_device_by_path(b->discovery, path);
    if (d == NULL) {
        pa_log_error("Device doesn't exist for %s", path);
        goto fail;
    }

    if (d->enable_hfp_hf) {
        if (p == PA_BLUETOOTH_PROFILE_HSP_HS && pa_hashmap_get(d->uuids, PA_BLUETOOTH_UUID_HFP_HF)) {
            /* If peer connecting to HSP Audio Gateway supports HFP HF profile
             * reject this connection to force it to connect to HSP Audio Gateway instead.
             */
            pa_log_info("HFP HF enabled in native backend and is supported by peer, rejecting HSP HS peer connection");
            goto fail;
        }
    }

    pa_assert_se(dbus_message_iter_next(&arg_i));

    pa_assert(dbus_message_iter_get_arg_type(&arg_i) == DBUS_TYPE_UNIX_FD);
    dbus_message_iter_get_basic(&arg_i, &fd);

    pa_log_debug("dbus: NewConnection path=%s, fd=%d, profile %s", path, fd,
        pa_bluetooth_profile_to_string(p));

    sender = dbus_message_get_sender(m);

    pathfd = pa_sprintf_malloc ("%s/fd%d", path, fd);
    t = pa_bluetooth_transport_new(d, sender, pathfd, p, NULL,
                                   p == PA_BLUETOOTH_PROFILE_HFP_HF ?
                                   sizeof(struct hfp_config) : 0);
    pa_xfree(pathfd);

    t->acquire = sco_acquire_cb;
    t->release = sco_release_cb;
    t->destroy = transport_destroy;

    /* If PA is the HF/HS we are in control of volume attenuation and
     * can always send volume commands (notifications) to keep the peer
     * updated on actual volume value.
     *
     * If the peer is the HF/HS it is responsible for attenuation of both
     * speaker and microphone gain.
     * On HFP speaker/microphone gain support is reported by bit 4 in the
     * `AT+BRSF=` command. Since it isn't explicitly documented whether this
     * applies to speaker or microphone gain but the peer is required to send
     * an initial value with `AT+VG[MS]=` either callback is hooked
     * independently as soon as this command is received.
     * On HSP this is not specified and is assumed to be dynamic for both
     * speaker and microphone.
     */
    if (is_peer_audio_gateway(p)) {
        t->set_sink_volume = set_sink_volume;
        t->set_source_volume = set_source_volume;
    }

    pa_bluetooth_transport_reconfigure(t, pa_bluetooth_get_hf_codec("CVSD"), sco_transport_write, NULL);

    trd = pa_xnew0(struct transport_data, 1);
    trd->rfcomm_fd = fd;
    trd->mainloop = b->core->mainloop;
    trd->rfcomm_io = trd->mainloop->io_new(b->core->mainloop, fd, PA_IO_EVENT_INPUT,
        rfcomm_io_callback, t);
    t->userdata =  trd;

    sco_listen(t);

    if (p != PA_BLUETOOTH_PROFILE_HFP_HF)
        transport_put(t);

    pa_assert_se(r = dbus_message_new_method_return(m));

    return r;

fail:
    pa_assert_se(r = dbus_message_new_error(m, BLUEZ_ERROR_INVALID_ARGUMENTS, "Unable to handle new connection"));
    return r;
}

static DBusMessage *profile_request_disconnection(DBusConnection *conn, DBusMessage *m, void *userdata) {
    DBusMessage *r;

    pa_assert_se(r = dbus_message_new_method_return(m));

    return r;
}

static DBusHandlerResult profile_handler(DBusConnection *c, DBusMessage *m, void *userdata) {
    pa_bluetooth_backend *b = userdata;
    DBusMessage *r = NULL;
    const char *path, *interface, *member;

    pa_assert(b);

    path = dbus_message_get_path(m);
    interface = dbus_message_get_interface(m);
    member = dbus_message_get_member(m);

    pa_log_debug("dbus: path=%s, interface=%s, member=%s", path, interface, member);

    if (!pa_streq(path, HSP_AG_PROFILE) && !pa_streq(path, HSP_HS_PROFILE)
        && !pa_streq(path, HFP_AG_PROFILE))
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

    if (dbus_message_is_method_call(m, DBUS_INTERFACE_INTROSPECTABLE, "Introspect")) {
        const char *xml = PROFILE_INTROSPECT_XML;

        pa_assert_se(r = dbus_message_new_method_return(m));
        pa_assert_se(dbus_message_append_args(r, DBUS_TYPE_STRING, &xml, DBUS_TYPE_INVALID));

    } else if (dbus_message_is_method_call(m, BLUEZ_PROFILE_INTERFACE, "Release")) {
        pa_log_debug("Release not handled");
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
    } else if (dbus_message_is_method_call(m, BLUEZ_PROFILE_INTERFACE, "RequestDisconnection")) {
        r = profile_request_disconnection(c, m, userdata);
    } else if (dbus_message_is_method_call(m, BLUEZ_PROFILE_INTERFACE, "NewConnection"))
        r = profile_new_connection(c, m, userdata);
    else
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

    if (r) {
        pa_assert_se(dbus_connection_send(pa_dbus_connection_get(b->connection), r, NULL));
        dbus_message_unref(r);
    }

    return DBUS_HANDLER_RESULT_HANDLED;
}
```

`profile_handler()` 回调函数主要处理蓝牙设备的 profile 的 D-BUS 对象路径上的 **org.freedesktop.DBus.Introspectable.Introspect**、新建连接 (**org.bluez.Profile1.NewConnection**)、请求连接断开 (**org.bluez.Profile1.RequestDisconnection**) 和释放 (**org.bluez.Profile1.Release**) 方法。**org.freedesktop.DBus.Introspectable.IntrospectIntrospect** 方法用于返回 profile 的 D-BUS 对象路径上支持的接口和方法，释放 (**org.bluez.Profile1.Release**) 方法不处理，请求连接断开 (**org.bluez.Profile1.RequestDisconnection**) 方法简单的向调用方返回，不做额外处理。`profile_new_connection()` 函数用来处理 profile 的 D-BUS 对象路径上的新建连接 (**org.bluez.Profile1.NewConnection**) 方法的调用。

在 PulseAudio 中，对 HFP (Hands-Free Profile) 和 HSP (Headset Profile) 的处理涉及多个 I/O 通道的协同工作，包括 **RFCOMM**、listen 的 **SCO** 和 **Stream**。这些 I/O 通道分别负责不同的任务，共同实现蓝牙音频数据和控制信息的传输。以下是它们的详细分工和协同工作原理：

 * **RFCOMM**：RFCOMM I/O 通道用于处理控制信道的通信。它基于 RFCOMM 协议，负责传输 HFP/HSP 的控制命令（如接听/挂断电话、音量控制等）。蓝牙服务 BlueZ 通过 D-BUS 调用 profile 的新建连接 (**NewConnection**) 方法时，消息中包含 RFCOMM I/O 通道的文件描述符，PulseAudio 通过这个文件描述符与蓝牙设备建立控制信道连接，控制信道的通信使用 AT 命令（如 **AT+CKPD** 用于接听电话），RFCOMM I/O 通道负责接收和发送这些控制命令。它处理与蓝牙设备的控制交互，管理电话状态（如通话中、挂断等）。

 * Listen 的 **SCO**：用于监听 SCO（Synchronous Connection-Oriented）音频信道的连接请求。它负责接受来自蓝牙设备的音频信道连接请求。这个 I/O 通道在 PulseAudio 中创建。

 * **Stream**：**Stream** I/O 通道用于处理音频数据的传输。它负责在 HFP/HSP 中传输语音数据（如通话音频）。PulseAudio 在需要时创建 **Stream** I/O 通道。

PulseAudio 用 `pa_bluetooth_transport` 对象描述特定 profile 类型的传输，这个类型定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.h*) 如下：
```
typedef enum pa_bluetooth_transport_state {
    PA_BLUETOOTH_TRANSPORT_STATE_DISCONNECTED,
    PA_BLUETOOTH_TRANSPORT_STATE_IDLE,
    PA_BLUETOOTH_TRANSPORT_STATE_PLAYING
} pa_bluetooth_transport_state_t;

typedef int (*pa_bluetooth_transport_acquire_cb)(pa_bluetooth_transport *t, bool optional, size_t *imtu, size_t *omtu);
typedef void (*pa_bluetooth_transport_release_cb)(pa_bluetooth_transport *t);
typedef void (*pa_bluetooth_transport_destroy_cb)(pa_bluetooth_transport *t);
typedef pa_volume_t (*pa_bluetooth_transport_set_volume_cb)(pa_bluetooth_transport *t, pa_volume_t volume);
typedef ssize_t (*pa_bluetooth_transport_write_cb)(pa_bluetooth_transport *t, int fd, const void* buffer, size_t size, size_t write_mtu);
typedef int (*pa_bluetooth_transport_setsockopt_cb)(pa_bluetooth_transport *t, int fd);

struct pa_bluetooth_transport {
    pa_bluetooth_device *device;

    char *owner;
    char *path;
    pa_bluetooth_profile_t profile;

    void *config;
    size_t config_size;

    const pa_bt_codec *bt_codec;
    int stream_write_type;
    size_t last_read_size;

    pa_volume_t source_volume;
    pa_volume_t sink_volume;

    pa_bluetooth_transport_state_t state;

    pa_bluetooth_transport_acquire_cb acquire;
    pa_bluetooth_transport_release_cb release;
    pa_bluetooth_transport_write_cb write;
    pa_bluetooth_transport_setsockopt_cb setsockopt;
    pa_bluetooth_transport_destroy_cb destroy;
    pa_bluetooth_transport_set_volume_cb set_sink_volume;
    pa_bluetooth_transport_set_volume_cb set_source_volume;
    void *userdata;
};
```

`pa_bluetooth_transport` 对象维护特定 profile 类型的传输有关的各种资源和支持的操作，如 I/O 通道等。`profile_new_connection()` 函数为 profile 连接创建 `pa_bluetooth_transport` 对象，并初始化所需的 I/O 通道，更详细的过程如下：

1. 根据 D-BUS 消息中包含的 profile 的 D-BUS 对象路径获得具体的 profile。
2. 根据 D-BUS 消息中包含的蓝牙耳机设备的路径获得对应的蓝牙设备 `pa_bluetooth_device` 对象。
3. 如果蓝牙耳机设备支持 HFP_HF profile，但要处理的 profile 是 HSP_HS，则报错返回。蓝牙耳机设备支持 HFP_HF profile 时，对 HSP_HS 的处理显得多多余。
4. 从 D-BUS 消息中获得 BlueZ 传过来的 RFCOMM I/O 通道的文件描述符及发送者信息。
5. 创建 `pa_bluetooth_transport` 对象。调用 `pa_bluetooth_transport_new()` 函数新建 `pa_bluetooth_transport` 对象，初始化各个回调函数为 HFP/HSP 协议的对应函数，具体如下：
     * `acquire`：用于获得 **Stream** I/O 通道文件描述符，设置为 `sco_acquire_cb()`；
     * `release`：用于释放 **Stream** I/O 通道文件描述符，设置为 `sco_release_cb()`；
     * `destroy`：用于销毁 `pa_bluetooth_transport` 对象时释放 HFP/HSP 协议传输特有的资源，设置为 `transport_destroy()`；
     * `set_sink_volume`：用于设置播放音量，设置为 `set_sink_volume()`；
     * `set_source_volume`：用于设置录音音量，设置为 `set_source_volume()`。
6. 调用 `pa_bluetooth_transport_reconfigure()` 函数为 `pa_bluetooth_transport` 对象设置 bt_codec 和用于向 **Stream** I/O 通道写入音频数据的 `write` 回调。bt_codec 默认为 **CVSD**，`write` 回调默认为 `sco_transport_write()`，这些设置可能在后续的协商中被修改。
7. 配置 **RFCOMM** I/O 通道，配置通道上的 I/O 处理程序为 `rfcomm_io_callback()`。
8. 调用 `sco_listen()` 函数创建并配置 listen 的 **SCO** I/O 通道，配置通道上的 I/O 处理程序为 `sco_io_callback()`。监听的地址为蓝牙控制器/适配器的地址。
9. 如果 profile 不是 **HFP_HF**，将创建的 `pa_bluetooth_transport` 对象保存在 `pa_bluetooth_device` 对象的对应 profile 的位置上，并设置 `pa_bluetooth_transport` 的状态为 **PA_BLUETOOTH_TRANSPORT_STATE_IDLE**。**HFP_HF** profile 的 bt_codec 信息需要进一步协商。

 `pa_bluetooth_transport` 类型没有从 **Stream** I/O 通道读取音频数据的回调。读取数据通过更一般的接口实现。

`pa_bluetooth_transport` 类型相关的几个函数定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
pa_bluetooth_transport *pa_bluetooth_transport_new(pa_bluetooth_device *d, const char *owner, const char *path,
                                                   pa_bluetooth_profile_t p, const uint8_t *config, size_t size) {
    pa_bluetooth_transport *t;

    t = pa_xnew0(pa_bluetooth_transport, 1);
    t->device = d;
    t->owner = pa_xstrdup(owner);
    t->path = pa_xstrdup(path);
    t->profile = p;
    t->config_size = size;
    /* Always force initial volume to be set/propagated correctly */
    t->sink_volume = PA_VOLUME_INVALID;
    t->source_volume = PA_VOLUME_INVALID;

    if (size > 0) {
        t->config = pa_xnew(uint8_t, size);
        if (config)
            memcpy(t->config, config, size);
        else
            memset(t->config, 0, size);
    }

    return t;
}

void pa_bluetooth_transport_reconfigure(pa_bluetooth_transport *t, const pa_bt_codec *bt_codec,
                                        pa_bluetooth_transport_write_cb write_cb, pa_bluetooth_transport_setsockopt_cb setsockopt_cb) {
    pa_assert(t);

    t->bt_codec = bt_codec;

    t->write = write_cb;
    t->setsockopt = setsockopt_cb;

    /* reset stream write type hint */
    t->stream_write_type = 0;

    /* reset SCO MTU adjustment hint */
    t->last_read_size = 0;
}
 . . . . . .
void pa_bluetooth_transport_put(pa_bluetooth_transport *t) {
    pa_assert(t);

    t->device->transports[t->profile] = t;
    pa_assert_se(pa_hashmap_put(t->device->discovery->transports, t->path, t) >= 0);
    pa_bluetooth_transport_set_state(t, PA_BLUETOOTH_TRANSPORT_STATE_IDLE);
}
```

`profile_new_connection()` 函数执行的结束，对于 **HFP_HF** profile，不意味着 profile 的传输配置的结束，还需要通过 **RFCOMM** I/O 通道传输 **AT** 命令来进一步处理，如在上面的 pulseaudio 的日志中看到的那样。这里进一步的处理从收到 **AT** 命令开始。**RFCOMM** I/O 通道的 I/O 事件处理程序 `rfcomm_io_callback()` 定义 (位于 *pulseaudio/src/modules/bluetooth/backend-native.c*) 如下：
```
static void rfcomm_fmt_write(int fd, const char* fmt_line, const char *fmt_command, va_list ap)
{
    size_t len;
    char buf[512];
    char command[512];

    pa_vsnprintf(command, sizeof(command), fmt_command, ap);

    pa_log_debug("RFCOMM >> %s", command);

    len = pa_snprintf(buf, sizeof(buf), fmt_line, command);

    /* we ignore any errors, it's not critical and real errors should
     * be caught with the HANGUP and ERROR events handled above */

    if ((size_t)write(fd, buf, len) != len)
        pa_log_error("RFCOMM write error: %s", pa_cstrerror(errno));
}

/* The format of COMMAND line sent from HS to AG is COMMAND<cr> */
static void rfcomm_write_command(int fd, const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    rfcomm_fmt_write(fd, "%s\r", fmt, ap);
    va_end(ap);
}

/* The format of RESPONSE line sent from AG to HS is <cr><lf>RESPONSE<cr><lf> */
static void rfcomm_write_response(int fd, const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    rfcomm_fmt_write(fd, "\r\n%s\r\n", fmt, ap);
    va_end(ap);
}
 . . . . . .
static bool hfp_rfcomm_handle(int fd, pa_bluetooth_transport *t, const char *buf)
{
    struct hfp_config *c = t->config;
    int indicator, val;
    char str[5];
    const char *r;
    size_t len;
    const char *state;

    /* first-time initialize selected codec to CVSD */
    if (c->selected_codec == 0)
        c->selected_codec = 1;

    /* stateful negotiation */
    if (c->state == 0 && sscanf(buf, "AT+BRSF=%d", &val) == 1) {
        c->capabilities = val;
        pa_log_info("HFP capabilities returns 0x%x", val);
        rfcomm_write_response(fd, "+BRSF: %d", hfp_features);
        c->supports_indicators = !!(1 << HFP_HF_INDICATORS);
        c->state = 1;

        return true;
    } else if (sscanf(buf, "AT+BAC=%3s", str) == 1) {
        c->support_msbc = false;

        state = NULL;

        /* check if codec id 2 (mSBC) is in the list of supported codecs */
        while ((r = pa_split_in_place(str, ",", &len, &state))) {
            if (len == 1 && r[0] == '2') {
                c->support_msbc = true;
                break;
            }
        }

        c->support_codec_negotiation = true;

        if (c->state == 1) {
            /* initial list of codecs supported by HF */
        } else {
            /* HF sent updated list of codecs */
        }

        /* no state change */

        return true;
    } else if (c->state == 1 && pa_startswith(buf, "AT+CIND=?")) {
        /* we declare minimal no indicators */
        rfcomm_write_response(fd, "+CIND: "
                     /* many indicators can be supported, only call and
                      * callheld are mandatory, so that's all we reply */
                     "(\"service\",(0-1)),"
                     "(\"call\",(0-1)),"
                     "(\"callsetup\",(0-3)),"
                     "(\"callheld\",(0-2))");
        c->state = 2;

        return true;
    } else if (c->state == 2 && pa_startswith(buf, "AT+CIND?")) {
        rfcomm_write_response(fd, "+CIND: 0,0,0,0");
        c->state = 3;

        return true;
    } else if ((c->state == 2 || c->state == 3) && pa_startswith(buf, "AT+CMER=")) {
        rfcomm_write_response(fd, "OK");

        if (c->support_codec_negotiation) {
            if (c->support_msbc && pa_bluetooth_discovery_get_enable_msbc(t->device->discovery)) {
                rfcomm_write_response(fd, "+BCS:2");
                c->state = 4;
            } else {
                rfcomm_write_response(fd, "+BCS:1");
                c->state = 4;
            }
        } else {
            c->state = 5;
            pa_bluetooth_transport_reconfigure(t, pa_bluetooth_get_hf_codec("CVSD"), sco_transport_write, NULL);
            transport_put(t);
        }

        return false;
    } else if (sscanf(buf, "AT+BCS=%d", &val)) {
        if (val == 1) {
            pa_bluetooth_transport_reconfigure(t, pa_bluetooth_get_hf_codec("CVSD"), sco_transport_write, NULL);
        } else if (val == 2 && pa_bluetooth_discovery_get_enable_msbc(t->device->discovery)) {
            pa_bluetooth_transport_reconfigure(t, pa_bluetooth_get_hf_codec("mSBC"), sco_transport_write, sco_setsockopt_enable_bt_voice);
        } else {
            pa_assert_fp(val != 1 && val != 2);
            rfcomm_write_response(fd, "ERROR");
            return false;
        }

        c->selected_codec = val;

        if (c->state == 4) {
            c->state = 5;
            pa_log_info("HFP negotiated codec %s", t->bt_codec->name);
            transport_put(t);
        }

        return true;
    } else if (c->supports_indicators && pa_startswith(buf, "AT+BIND=?")) {
        // Support battery indication
        rfcomm_write_response(fd, "+BIND: (2)");
        return true;
    } else if (c->supports_indicators && pa_startswith(buf, "AT+BIND?")) {
        // Battery indication is enabled
        rfcomm_write_response(fd, "+BIND: 2,1");
        return true;
    } else if (c->supports_indicators && pa_startswith(buf, "AT+BIND=")) {
        // If this comma-separated list contains `2`, the HF is
        // able to report values for the battery indicator.
        return true;
    } else if (c->supports_indicators && sscanf(buf, "AT+BIEV=%u,%u", &indicator, &val)) {
        switch (indicator) {
            case 2:
                pa_log_notice("Battery Level: %d%%", val);
                if (val < 0 || val > 100) {
                    pa_log_error("Battery HF indicator %d out of [0, 100] range", val);
                    rfcomm_write_response(fd, "ERROR");
                    return false;
                }
                pa_bluetooth_device_report_battery_level(t->device, val, "HFP 1.7 HF indicator");
                break;
            default:
                pa_log_error("Unknown HF indicator %u", indicator);
                rfcomm_write_response(fd, "ERROR");
                return false;
        }
        return true;
    } if (c->state == 4) {
        /* the ack for the codec setting may take a while. we need
         * to reply OK to everything else until then */
        return true;
    }

    /* if we get here, negotiation should be complete */
    if (c->state != 5) {
        pa_log_error("HFP negotiation failed in state %d with inbound %s\n",
                     c->state, buf);
        rfcomm_write_response(fd, "ERROR");
        return false;
    }

    /*
     * once we're fully connected, just reply OK to everything
     * it will just be the headset sending the occasional status
     * update, but we process only the ones we care about
     */
    return true;
}

static void rfcomm_io_callback(pa_mainloop_api *io, pa_io_event *e, int fd, pa_io_event_flags_t events, void *userdata) {
    pa_bluetooth_transport *t = userdata;

    pa_assert(io);
    pa_assert(t);

    if (events & (PA_IO_EVENT_HANGUP|PA_IO_EVENT_ERROR)) {
        pa_log_info("Lost RFCOMM connection.");
        // TODO: Keep track of which profile is the current battery provider,
        // only deregister if it is us currently providing these levels.
        // (Also helpful to fill the 'Source' property)
        // We might also move this to Profile1::RequestDisconnection
        pa_bluetooth_device_deregister_battery(t->device);
        goto fail;
    }

    if (events & PA_IO_EVENT_INPUT) {
        char buf[512];
        ssize_t len;
        int gain, dummy;
        bool do_reply = false;
        int vendor, product, version, features;
        int num;

        len = pa_read(fd, buf, 511, NULL);
        if (len < 0) {
            pa_log_error("RFCOMM read error: %s", pa_cstrerror(errno));
            goto fail;
        }
        buf[len] = 0;
        pa_log_debug("RFCOMM << %s", buf);

        /* There are only four HSP AT commands:
         * AT+VGS=value: value between 0 and 15, sent by the HS to AG to set the speaker gain.
         * +VGS=value is sent by AG to HS as a response to an AT+VGS command or when the gain
         * is changed on the AG side.
         * AT+VGM=value: value between 0 and 15, sent by the HS to AG to set the microphone gain.
         * +VGM=value is sent by AG to HS as a response to an AT+VGM command or when the gain
         * is changed on the AG side.
         * AT+CKPD=200: Sent by HS when headset button is pressed.
         * RING: Sent by AG to HS to notify of an incoming call. It can safely be ignored because
         * it does not expect a reply. */
        if (sscanf(buf, "AT+VGS=%d", &gain) == 1 || sscanf(buf, "\r\n+VGM%*[=:]%d\r\n", &gain) == 1) {
            if (!t->set_sink_volume) {
                pa_log_debug("HS/HF peer supports speaker gain control");
                t->set_sink_volume = set_sink_volume;
            }

            t->sink_volume = hsp_gain_to_volume(gain);
            pa_hook_fire(pa_bluetooth_discovery_hook(t->device->discovery, PA_BLUETOOTH_HOOK_TRANSPORT_SINK_VOLUME_CHANGED), t);
            do_reply = true;

        } else if (sscanf(buf, "AT+VGM=%d", &gain) == 1 || sscanf(buf, "\r\n+VGS%*[=:]%d\r\n", &gain) == 1) {
            if (!t->set_source_volume) {
                pa_log_debug("HS/HF peer supports microphone gain control");
                t->set_source_volume = set_source_volume;
            }

            t->source_volume = hsp_gain_to_volume(gain);
            pa_hook_fire(pa_bluetooth_discovery_hook(t->device->discovery, PA_BLUETOOTH_HOOK_TRANSPORT_SOURCE_VOLUME_CHANGED), t);
            do_reply = true;
        } else if (sscanf(buf, "AT+CKPD=%d", &dummy) == 1) {
            do_reply = true;
        } else if (sscanf(buf, "AT+XAPL=%04x-%04x-%04x,%d", &vendor, &product, &version, &features) == 4) {
            if (features & 0x2)
                /* claim, that we support battery status reports */
                rfcomm_write_response(fd, "+XAPL=iPhone,6");
            do_reply = true;
        } else if (sscanf(buf, "AT+IPHONEACCEV=%d", &num) == 1) {
            char *substr = buf, *keystr;
            int key, val, i;

            do_reply = true;

            for (i = 0; i < num; ++i) {
                keystr = strchr(substr, ',');
                if (!keystr) {
                    pa_log_warn("%s misses key for argument #%d", buf, i);
                    do_reply = false;
                    break;
                }
                keystr++;
                substr = strchr(keystr, ',');
                if (!substr) {
                    pa_log_warn("%s misses value for argument #%d", buf, i);
                    do_reply = false;
                    break;
                }
                substr++;

                key = atoi(keystr);
                val = atoi(substr);

                switch (key) {
                    case 1:
                        pa_log_notice("Battery Level: %d0%%", val + 1);
                        pa_bluetooth_device_report_battery_level(t->device, (val + 1) * 10, "Apple accessory indication");
                        break;
                    case 2:
                        pa_log_notice("Dock Status: %s", val ? "docked" : "undocked");
                        break;
                    default:
                        pa_log_debug("Unexpected IPHONEACCEV key %#x", key);
                        break;
                }
            }
            if (!do_reply)
                rfcomm_write_response(fd, "ERROR");
        } else if (t->config) { /* t->config is only non-null for hfp profile */
            do_reply = hfp_rfcomm_handle(fd, t, buf);
        } else {
            rfcomm_write_response(fd, "ERROR");
            do_reply = false;
        }

        if (do_reply)
            rfcomm_write_response(fd, "OK");
    }

    return;

fail:
    pa_bluetooth_transport_unlink(t);
    pa_bluetooth_transport_free(t);
}
```

`rfcomm_io_callback()` 函数从 **RFCOMM** I/O 通道读取数据，根据数据中包含的具体的 **AT** 命令做进一步处理。系统通过 **AT+BRSF** 命令上报 HFP capabilities，通过 **AT+BAC** 命令上报支持的 bt_codec，通过 **AT+CMER** 命令控制和配置移动设备的事件报告机制，通过 **AT+VGS** 命令表示支持设置播放音量，通过 **AT+VGM** 命令表示支持设置录音音量，通过 **AT+BCS** 命令协商最终的 bt_codec。协商出最终的 bt_codec 之后，将创建的 `pa_bluetooth_transport` 对象保存在 `pa_bluetooth_device` 对象的对应 profile 的位置上，并设置 `pa_bluetooth_transport` 的状态为 **PA_BLUETOOTH_TRANSPORT_STATE_IDLE**，表示对应 profile 的连接可用。

`pa_bluetooth_transport_set_state()` 函数用于设置 `pa_bluetooth_transport` 的状态，这个函数定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
static void wait_for_profiles_cb(pa_mainloop_api *api, pa_time_event* event, const struct timeval *tv, void *userdata) {
    pa_bluetooth_device *device = userdata;
    pa_strbuf *buf;
    pa_bluetooth_profile_t profile;
    bool first = true;
    char *profiles_str;

    device_stop_waiting_for_profiles(device);

    buf = pa_strbuf_new();

    for (profile = 0; profile < PA_BLUETOOTH_PROFILE_COUNT; profile++) {
        if (device_is_profile_connected(device, profile))
            continue;

        if (!pa_bluetooth_device_supports_profile(device, profile))
            continue;

        if (first)
            first = false;
        else
            pa_strbuf_puts(buf, ", ");

        pa_strbuf_puts(buf, pa_bluetooth_profile_to_string(profile));
    }

    profiles_str = pa_strbuf_to_string_free(buf);
    pa_log_debug("Timeout expired, and device %s still has disconnected profiles: %s",
                 device->path, profiles_str);
    pa_xfree(profiles_str);
    pa_hook_fire(&device->discovery->hooks[PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED], device);
}

static void device_start_waiting_for_profiles(pa_bluetooth_device *device) {
    pa_assert(!device->wait_for_profiles_timer);
    device->wait_for_profiles_timer = pa_core_rttime_new(device->discovery->core,
                                                         pa_rtclock_now() + WAIT_FOR_PROFILES_TIMEOUT_USEC,
                                                         wait_for_profiles_cb, device);
}
 . . . . . .
void pa_bluetooth_transport_set_state(pa_bluetooth_transport *t, pa_bluetooth_transport_state_t state) {
    bool old_any_connected;
    unsigned n_disconnected_profiles;
    bool new_device_appeared;
    bool device_disconnected;

    pa_assert(t);

    if (t->state == state)
        return;

    old_any_connected = pa_bluetooth_device_any_transport_connected(t->device);

    pa_log_debug("Transport %s state: %s -> %s",
                 t->path, transport_state_to_string(t->state), transport_state_to_string(state));

    t->state = state;

    pa_hook_fire(&t->device->discovery->hooks[PA_BLUETOOTH_HOOK_TRANSPORT_STATE_CHANGED], t);

    /* If there are profiles that are expected to get connected soon (based
     * on the UUID list), we wait for a bit before announcing the new
     * device, so that all profiles have time to get connected before the
     * card object is created. If we didn't wait, the card would always
     * have only one profile marked as available in the initial state,
     * which would prevent module-card-restore from restoring the initial
     * profile properly. */

    n_disconnected_profiles = device_count_disconnected_profiles(t->device);

    new_device_appeared = !old_any_connected && pa_bluetooth_device_any_transport_connected(t->device);
    device_disconnected = old_any_connected && !pa_bluetooth_device_any_transport_connected(t->device);

    if (new_device_appeared) {
        if (n_disconnected_profiles > 0)
            device_start_waiting_for_profiles(t->device);
        else
            pa_hook_fire(&t->device->discovery->hooks[PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED], t->device);
        return;
    }

    if (device_disconnected) {
        if (t->device->wait_for_profiles_timer) {
            /* If the timer is still running when the device disconnects, we
             * never sent the notification of the device getting connected, so
             * we don't need to send a notification about the disconnection
             * either. Let's just stop the timer. */
            device_stop_waiting_for_profiles(t->device);
        } else
            pa_hook_fire(&t->device->discovery->hooks[PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED], t->device);
        return;
    }

    if (n_disconnected_profiles == 0 && t->device->wait_for_profiles_timer) {
        /* All profiles are now connected, so we can stop the wait timer and
         * send a notification of the new device. */
        device_stop_waiting_for_profiles(t->device);
        pa_hook_fire(&t->device->discovery->hooks[PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED], t->device);
    }
}
```

对于有新的 profile 连接的 `pa_bluetooth_transport` 状态切换到连接状态的蓝牙耳机设备，分为几种情况处理：

 * 新的 profile 连接的 `pa_bluetooth_transport` 是设备的第一个切换到连接状态的 `pa_bluetooth_transport`，且蓝牙耳机设备只支持这个 profile：发出 **PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED** 事件。
 * 新的 profile 连接的 `pa_bluetooth_transport` 是设备的第一个切换到连接状态的 `pa_bluetooth_transport`，且蓝牙耳机设备支持多个 profile：起一个 3s 的定时器等待蓝牙耳机设备支持的其它 profile 连接的 `pa_bluetooth_transport` 建立并切换到连接状态。
 * 新的 profile 连接的 `pa_bluetooth_transport` 不是设备的第一个切换到连接状态的 `pa_bluetooth_transport`，是最后一个切换到连接状态的 `pa_bluetooth_transport`：意味着定时器已经创建，先销毁前面创建的定时器，再发出 **PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED** 事件。
 * 新的 profile 连接的 `pa_bluetooth_transport` 不是设备的第一个切换到连接状态的 `pa_bluetooth_transport`，也不是最后一个切换到连接状态的 `pa_bluetooth_transport`：不做任何处理。
 * 起的定时器没有等到支持的所有 profile 的连接的 `pa_bluetooth_transport` 都切换到连接状态既已超时，仍然发出 **PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED** 事件。

**PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED** 事件的处理程序由 **module-bluez5-discover**，它会为蓝牙耳机设备加载 **module-bluez5-device** 模块。

对于 A2DP/AVRCP 协议栈，蓝牙耳机设备的连接处理，从 `endpoint_handler()` 回调函数开始，这个函数定义 (位于 *pulseaudio/src/modules/bluetooth/bluez5-util.c*) 如下：
```
static const pa_a2dp_endpoint_conf *a2dp_sep_to_a2dp_endpoint_conf(const char *endpoint) {
    const char *codec_name;

    if (pa_startswith(endpoint, A2DP_SINK_ENDPOINT "/"))
        codec_name = endpoint + strlen(A2DP_SINK_ENDPOINT "/");
    else if (pa_startswith(endpoint, A2DP_SOURCE_ENDPOINT "/"))
        codec_name = endpoint + strlen(A2DP_SOURCE_ENDPOINT "/");
    else
        return NULL;

    return pa_bluetooth_get_a2dp_endpoint_conf(codec_name);
}

static DBusMessage *endpoint_set_configuration(DBusConnection *conn, DBusMessage *m, void *userdata) {
    pa_bluetooth_discovery *y = userdata;
    pa_bluetooth_device *d;
    pa_bluetooth_transport *t;
    const pa_a2dp_endpoint_conf *endpoint_conf = NULL;
    const char *sender, *path, *endpoint_path, *dev_path = NULL, *uuid = NULL;
    const uint8_t *config = NULL;
    int size = 0;
    pa_bluetooth_profile_t p = PA_BLUETOOTH_PROFILE_OFF;
    DBusMessageIter args, props;
    DBusMessage *r;

    if (!dbus_message_iter_init(m, &args) || !pa_streq(dbus_message_get_signature(m), "oa{sv}")) {
        pa_log_error("Invalid signature for method SetConfiguration()");
        goto fail2;
    }

    dbus_message_iter_get_basic(&args, &path);

    if (pa_hashmap_get(y->transports, path)) {
        pa_log_error("Endpoint SetConfiguration(): Transport %s is already configured.", path);
        goto fail2;
    }

    pa_assert_se(dbus_message_iter_next(&args));

    dbus_message_iter_recurse(&args, &props);
    if (dbus_message_iter_get_arg_type(&props) != DBUS_TYPE_DICT_ENTRY)
        goto fail;

    endpoint_path = dbus_message_get_path(m);

    /* Read transport properties */
    while (dbus_message_iter_get_arg_type(&props) == DBUS_TYPE_DICT_ENTRY) {
        const char *key;
        DBusMessageIter value, entry;
        int var;

        dbus_message_iter_recurse(&props, &entry);
        dbus_message_iter_get_basic(&entry, &key);

        dbus_message_iter_next(&entry);
        dbus_message_iter_recurse(&entry, &value);

        var = dbus_message_iter_get_arg_type(&value);

        if (pa_streq(key, "UUID")) {
            if (var != DBUS_TYPE_STRING) {
                pa_log_error("Property %s of wrong type %c", key, (char)var);
                goto fail;
            }

            dbus_message_iter_get_basic(&value, &uuid);

            if (pa_startswith(endpoint_path, A2DP_SINK_ENDPOINT "/"))
                p = PA_BLUETOOTH_PROFILE_A2DP_SOURCE;
            else if (pa_startswith(endpoint_path, A2DP_SOURCE_ENDPOINT "/"))
                p = PA_BLUETOOTH_PROFILE_A2DP_SINK;

            if ((pa_streq(uuid, PA_BLUETOOTH_UUID_A2DP_SOURCE) && p != PA_BLUETOOTH_PROFILE_A2DP_SINK) ||
                (pa_streq(uuid, PA_BLUETOOTH_UUID_A2DP_SINK) && p != PA_BLUETOOTH_PROFILE_A2DP_SOURCE)) {
                pa_log_error("UUID %s of transport %s incompatible with endpoint %s", uuid, path, endpoint_path);
                goto fail;
            }
        } else if (pa_streq(key, "Device")) {
            if (var != DBUS_TYPE_OBJECT_PATH) {
                pa_log_error("Property %s of wrong type %c", key, (char)var);
                goto fail;
            }

            dbus_message_iter_get_basic(&value, &dev_path);
        } else if (pa_streq(key, "Configuration")) {
            DBusMessageIter array;

            if (var != DBUS_TYPE_ARRAY) {
                pa_log_error("Property %s of wrong type %c", key, (char)var);
                goto fail;
            }

            dbus_message_iter_recurse(&value, &array);
            var = dbus_message_iter_get_arg_type(&array);
            if (var != DBUS_TYPE_BYTE) {
                pa_log_error("%s is an array of wrong type %c", key, (char)var);
                goto fail;
            }

            dbus_message_iter_get_fixed_array(&array, &config, &size);

            endpoint_conf = a2dp_sep_to_a2dp_endpoint_conf(endpoint_path);
            pa_assert(endpoint_conf);

            if (!endpoint_conf->is_configuration_valid(config, size))
                goto fail;
        }

        dbus_message_iter_next(&props);
    }

    if (!endpoint_conf)
        goto fail2;

    if ((d = pa_hashmap_get(y->devices, dev_path))) {
        if (!d->valid) {
            pa_log_error("Information about device %s is invalid", dev_path);
            goto fail2;
        }
    } else {
        /* InterfacesAdded signal is probably on its way, device_info_valid is kept as 0. */
        pa_log_warn("SetConfiguration() received for unknown device %s", dev_path);
        d = device_create(y, dev_path);
    }

    if (d->transports[p] != NULL) {
        pa_log_error("Cannot configure transport %s because profile %s is already used", path, pa_bluetooth_profile_to_string(p));
        goto fail2;
    }

    sender = dbus_message_get_sender(m);

    pa_assert_se(r = dbus_message_new_method_return(m));
    pa_assert_se(dbus_connection_send(pa_dbus_connection_get(y->connection), r, NULL));
    dbus_message_unref(r);

    t = pa_bluetooth_transport_new(d, sender, path, p, config, size);
    t->acquire = bluez5_transport_acquire_cb;
    t->release = bluez5_transport_release_cb;
    /* A2DP Absolute Volume is optional but BlueZ unconditionally reports
     * feature category 2, meaning supporting it is mandatory.
     * PulseAudio can and should perform the attenuation anyway in
     * the source role as it is the audio rendering device.
     */
    t->set_source_volume = pa_bluetooth_transport_set_source_volume;

    pa_bluetooth_transport_reconfigure(t, &endpoint_conf->bt_codec, a2dp_transport_write, NULL);
    pa_bluetooth_transport_put(t);

    pa_log_debug("Transport %s available for profile %s", t->path, pa_bluetooth_profile_to_string(t->profile));
    pa_log_info("Selected codec: %s", endpoint_conf->bt_codec.name);

    return NULL;

fail:
    pa_log_error("Endpoint SetConfiguration(): invalid arguments");

fail2:
    pa_assert_se(r = dbus_message_new_error(m, BLUEZ_ERROR_INVALID_ARGUMENTS, "Unable to set configuration"));
    return r;
}

static DBusMessage *endpoint_select_configuration(DBusConnection *conn, DBusMessage *m, void *userdata) {
    pa_bluetooth_discovery *y = userdata;
    const char *endpoint_path;
    uint8_t *cap;
    int size;
    const pa_a2dp_endpoint_conf *endpoint_conf;
    uint8_t config[MAX_A2DP_CAPS_SIZE];
    uint8_t *config_ptr = config;
    size_t config_size;
    DBusMessage *r;
    DBusError err;

    endpoint_path = dbus_message_get_path(m);

    dbus_error_init(&err);

    if (!dbus_message_get_args(m, &err, DBUS_TYPE_ARRAY, DBUS_TYPE_BYTE, &cap, &size, DBUS_TYPE_INVALID)) {
        pa_log_error("Endpoint SelectConfiguration(): %s", err.message);
        dbus_error_free(&err);
        goto fail;
    }

    endpoint_conf = a2dp_sep_to_a2dp_endpoint_conf(endpoint_path);
    pa_assert(endpoint_conf);

    config_size = endpoint_conf->fill_preferred_configuration(&y->core->default_sample_spec, cap, size, config);
    if (config_size == 0)
        goto fail;

    pa_assert_se(r = dbus_message_new_method_return(m));
    pa_assert_se(dbus_message_append_args(r, DBUS_TYPE_ARRAY, DBUS_TYPE_BYTE, &config_ptr, config_size, DBUS_TYPE_INVALID));

    return r;

fail:
    pa_assert_se(r = dbus_message_new_error(m, BLUEZ_ERROR_INVALID_ARGUMENTS, "Unable to select configuration"));
    return r;
}

static DBusMessage *endpoint_clear_configuration(DBusConnection *conn, DBusMessage *m, void *userdata) {
    pa_bluetooth_discovery *y = userdata;
    pa_bluetooth_transport *t;
    DBusMessage *r = NULL;
    DBusError err;
    const char *path;

    dbus_error_init(&err);

    if (!dbus_message_get_args(m, &err, DBUS_TYPE_OBJECT_PATH, &path, DBUS_TYPE_INVALID)) {
        pa_log_error("Endpoint ClearConfiguration(): %s", err.message);
        dbus_error_free(&err);
        goto fail;
    }

    if ((t = pa_hashmap_get(y->transports, path))) {
        pa_log_debug("Clearing transport %s profile %s", t->path, pa_bluetooth_profile_to_string(t->profile));
        pa_bluetooth_transport_free(t);
    }

    if (!dbus_message_get_no_reply(m))
        pa_assert_se(r = dbus_message_new_method_return(m));

    return r;

fail:
    if (!dbus_message_get_no_reply(m))
        pa_assert_se(r = dbus_message_new_error(m, BLUEZ_ERROR_INVALID_ARGUMENTS, "Unable to clear configuration"));
    return r;
}

static DBusMessage *endpoint_release(DBusConnection *conn, DBusMessage *m, void *userdata) {
    DBusMessage *r = NULL;

    /* From doc/media-api.txt in bluez:
     *
     *    This method gets called when the service daemon
     *    unregisters the endpoint. An endpoint can use it to do
     *    cleanup tasks. There is no need to unregister the
     *    endpoint, because when this method gets called it has
     *    already been unregistered.
     *
     * We don't have any cleanup to do. */

    /* Reply only if requested. Generally bluetoothd doesn't request a reply
     * to the Release() call. Sending replies when not requested on the system
     * bus tends to cause errors in syslog from dbus-daemon, because it
     * doesn't let unexpected replies through, so it's important to have this
     * check here. */
    if (!dbus_message_get_no_reply(m))
        pa_assert_se(r = dbus_message_new_method_return(m));

    return r;
}

static DBusHandlerResult endpoint_handler(DBusConnection *c, DBusMessage *m, void *userdata) {
    struct pa_bluetooth_discovery *y = userdata;
    DBusMessage *r = NULL;
    const char *path, *interface, *member;

    pa_assert(y);

    path = dbus_message_get_path(m);
    interface = dbus_message_get_interface(m);
    member = dbus_message_get_member(m);

    pa_log_debug("dbus: path=%s, interface=%s, member=%s", path, interface, member);

    if (!a2dp_sep_to_a2dp_endpoint_conf(path))
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

    if (dbus_message_is_method_call(m, DBUS_INTERFACE_INTROSPECTABLE, "Introspect")) {
        const char *xml = ENDPOINT_INTROSPECT_XML;

        pa_assert_se(r = dbus_message_new_method_return(m));
        pa_assert_se(dbus_message_append_args(r, DBUS_TYPE_STRING, &xml, DBUS_TYPE_INVALID));

    } else if (dbus_message_is_method_call(m, BLUEZ_MEDIA_ENDPOINT_INTERFACE, "SetConfiguration"))
        r = endpoint_set_configuration(c, m, userdata);
    else if (dbus_message_is_method_call(m, BLUEZ_MEDIA_ENDPOINT_INTERFACE, "SelectConfiguration"))
        r = endpoint_select_configuration(c, m, userdata);
    else if (dbus_message_is_method_call(m, BLUEZ_MEDIA_ENDPOINT_INTERFACE, "ClearConfiguration"))
        r = endpoint_clear_configuration(c, m, userdata);
    else if (dbus_message_is_method_call(m, BLUEZ_MEDIA_ENDPOINT_INTERFACE, "Release"))
        r = endpoint_release(c, m, userdata);
    else
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

    if (r) {
        pa_assert_se(dbus_connection_send(pa_dbus_connection_get(y->connection), r, NULL));
        dbus_message_unref(r);
    }

    return DBUS_HANDLER_RESULT_HANDLED;
}
```

`endpoint_handler()` 回调函数主要处理 A2DP media endpoint 的 D-BUS 对象路径上的 **org.freedesktop.DBus.Introspectable.Introspect**、设置配置 (**org.bluez.MediaEndpoint1.SetConfiguration**)、选择配置 (**org.bluez.MediaEndpoint1.SelectConfiguration**)、清除配置 (**org.bluez.MediaEndpoint1.ClearConfiguration**) 和释放 (**org.bluez.MediaEndpoint1.Release**) 方法。**org.freedesktop.DBus.Introspectable.Introspect** 方法用于返回 A2DP media endpoint 的 D-BUS 对象路径上支持的接口和方法，释放 (**org.bluez.MediaEndpoint1.Release**) 方法简单响应请求，不做过多处理。蓝牙耳机设备和蓝牙控制器/适配器连上之后，蓝牙服务 BlueZ 通过设置配置 (**org.bluez.MediaEndpoint1.SetConfiguration**) 方法配置 A2DP 的 profile 传输，具体由 `endpoint_set_configuration()` 函数处理。`endpoint_set_configuration()` 函数的处理过程如下：

1. 检查蓝牙耳机设备的对应 A2DP media endpoint 路径的 `pa_bluetooth_transport` 是否存在。通过传入的传输的路径，如 */org/bluez/hci0/dev_24_D0_DF_96_AD_6E/fd7* 来查找对应的 `pa_bluetooth_transport` 对象，以检查是否存在。
2. 获得 A2DP media endpoint 的路径，如 **/MediaEndpoint/A2DPSource/sbc_xq_552**。
3. 从 D-BUS 消息中解析获得表示 profile 的 UUID，根据 A2DP media endpoint 的路径得到 profile，对两者进行一致性验证；从 D-BUS 消息中解析获得蓝牙耳机设备的路径；从 D-BUS 消息中解析获得传过来的详细配置，据 A2DP media endpoint 的路径获得 media endpoint conf，并用它对详细配置做验证检查。
4. 获得蓝牙耳机设备的路径对应的 `pa_bluetooth_device` 对象，不存在时，新创建一个。
5. 创建 `pa_bluetooth_transport` 对象。调用 `pa_bluetooth_transport_new()` 函数新建 `pa_bluetooth_transport` 对象，初始化各个回调函数为 A2DP 协议的对应函数，具体如下：
     * `acquire`：用于获得 **Stream** I/O 通道文件描述符，设置为 `bluez5_transport_acquire_cb()`；
     * `release`：用于释放 **Stream** I/O 通道文件描述符，设置为 `bluez5_transport_release_cb()`；
     * `set_source_volume`：用于设置录音音量，设置为 `pa_bluetooth_transport_set_source_volume()`。
6. 调用 `pa_bluetooth_transport_reconfigure()` 函数为 `pa_bluetooth_transport` 对象设置 bt_codec 和用于向 **Stream** I/O 通道写入音频数据的 `write` 回调。bt_codec 从 media endpoint conf 获得，`write` 回调默认为 `a2dp_transport_write()`，这些设置可能在后续的协商中被修改。
7. 将创建的 `pa_bluetooth_transport` 对象保存在 `pa_bluetooth_device` 对象的对应 profile 的位置上，并设置 `pa_bluetooth_transport` 的状态为 **PA_BLUETOOTH_TRANSPORT_STATE_IDLE**。

用于处理 A2DP 连接的 `endpoint_handler()` 和用于处理 HFP/HSP 连接的 `profile_handler()` 尽管有着不同的执行过程，但都有着相同的归宿：它们都以为对应 profile 创建 `pa_bluetooth_transport` 对象，将创建的 `pa_bluetooth_transport` 对象保存在蓝牙耳机音频设备的对应 profile 的位置上，切换 `pa_bluetooth_transport` 的状态为 **PA_BLUETOOTH_TRANSPORT_STATE_IDLE**，发出 **PA_BLUETOOTH_HOOK_DEVICE_CONNECTION_CHANGED** 事件，由 **module-bluez5-discover** 模块为蓝牙耳机设备加载 **module-bluez5-device** 模块结束。

之后，对蓝牙耳机连接的处理进入新的阶段。



















































