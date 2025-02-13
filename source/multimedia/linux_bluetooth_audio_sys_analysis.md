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












































