PipeWire 的音频服务器架构有 3 个进程，分别是 pipewire 守护进程、pipewire-pulse 守护进程和 pipewire 媒体会话管理器。pipewire 守护进程负责创建节点并搭建媒体数据处理图。pipewire-pulse 守护进程作为 PulseAudio 兼容层，负责建立 PulseAudio 应用程序和 pipewire 守护进程之间的数据通道。pipewire 媒体会话管理器负责监听设备状态变化，并为 pipewire 守护进程创建节点和媒体数据处理图提供指导。此外，pipewire 体系中还有 pipewire 原生应用程序和 PulseAudio 应用程序。PipeWire 体系中任何两个进程之间的通信都需要 IPC。PipeWire 包含多个 IPC 协议，以支持不同进程之间的通信，更具体的，主要有 PulseAudio 兼容协议和原生 PipeWire IPC 协议，PulseAudio 兼容协议用于 PulseAudio 应用程序和 pipewire-pulse 守护进程之间的通信，原生 PipeWire IPC 协议则用于其它任意应用程序和 pipewire 守护进程之间的通信。

PipeWire 用 `struct pw_protocol` 类型描述 IPC 协议，这个类型定义 (位于 *pipewire/src/pipewire/private.h*) 如下：
```
struct pw_protocol {
	struct spa_list link;                   /**< link in context protocol_list */
	struct pw_context *context;                   /**< context for this protocol */

	char *name;                             /**< type name of the protocol */

	struct spa_list marshal_list;           /**< list of marshallers for supported interfaces */
	struct spa_list client_list;            /**< list of current clients */
	struct spa_list server_list;            /**< list of current servers */
	struct spa_hook_list listener_list;	/**< event listeners */

	const struct pw_protocol_implementation *implementation; /**< implementation of the protocol */

	const void *extension;  /**< extension API */

	void *user_data;        /**< user data for the implementation */
};
```

`struct pw_protocol` 对象维护 IPC 协议上当前进程提供的服务器、当前进程向 IPC 协议服务器发起的连接的客户端，客户端和服务器通信时使用的所支持接口的 marshallers，及协议实现和扩展 API 等。当前进程向 IPC 协议服务器发起的连接的客户端、当前进程提供的服务器、marshaller 和协议实现分别用 `struct pw_protocol_client`、`struct pw_protocol_server`、`struct pw_protocol_marshal` 和 `struct pw_protocol_implementation` 类型描述，这些类型定义 (位于 *pipewire/src/pipewire/protocol.h*) 如下：
```
#define PW_TYPE_INFO_Protocol		"PipeWire:Protocol"
#define PW_TYPE_INFO_PROTOCOL_BASE	PW_TYPE_INFO_Protocol ":"

struct pw_protocol_client {
	struct spa_list link;		/**< link in protocol client_list */
	struct pw_protocol *protocol;	/**< the owner protocol */

	struct pw_core *core;

	int (*connect) (struct pw_protocol_client *client,
			const struct spa_dict *props,
			void (*done_callback) (void *data, int result),
			void *data);
	int (*connect_fd) (struct pw_protocol_client *client, int fd, bool close);
	int (*steal_fd) (struct pw_protocol_client *client);
	void (*disconnect) (struct pw_protocol_client *client);
	void (*destroy) (struct pw_protocol_client *client);
	int (*set_paused) (struct pw_protocol_client *client, bool paused);
};

#define pw_protocol_client_connect(c,p,cb,d)	((c)->connect(c,p,cb,d))
#define pw_protocol_client_connect_fd(c,fd,cl)	((c)->connect_fd(c,fd,cl))
#define pw_protocol_client_steal_fd(c)		((c)->steal_fd(c))
#define pw_protocol_client_disconnect(c)	((c)->disconnect(c))
#define pw_protocol_client_destroy(c)		((c)->destroy(c))
#define pw_protocol_client_set_paused(c,p)	((c)->set_paused(c,p))

struct pw_protocol_server {
	struct spa_list link;		/**< link in protocol server_list */
	struct pw_protocol *protocol;	/**< the owner protocol */

	struct pw_impl_core *core;

	struct spa_list client_list;	/**< list of clients of this protocol */

	void (*destroy) (struct pw_protocol_server *listen);
};

#define pw_protocol_server_destroy(l)	((l)->destroy(l))

struct pw_protocol_marshal {
	const char *type;		/**< interface type */
	uint32_t version;		/**< version */
#define PW_PROTOCOL_MARSHAL_FLAG_IMPL	(1 << 0)	/**< marshal for implementations */
	uint32_t flags;			/**< version */
	uint32_t n_client_methods;	/**< number of client methods */
	uint32_t n_server_methods;	/**< number of server methods */
	const void *client_marshal;
	const void *server_demarshal;
	const void *server_marshal;
	const void *client_demarshal;
};

struct pw_protocol_implementation {
#define PW_VERSION_PROTOCOL_IMPLEMENTATION	0
	uint32_t version;

	struct pw_protocol_client * (*new_client) (struct pw_protocol *protocol,
						   struct pw_core *core,
						   const struct spa_dict *props);
	struct pw_protocol_server * (*add_server) (struct pw_protocol *protocol,
						   struct pw_impl_core *core,
						   const struct spa_dict *props);
};
```

`struct pw_protocol_client` 对象提供客户端与 IPC 协议服务器之间的连接有关的操作，包括建立连接，断开连接等。`struct pw_protocol_server` 对象维护 IPC 协议服务器中连接上来的客户端，服务器端客户端的表示为 `struct pw_impl_client`/`struct client_data`。`struct pw_protocol_marshal` 对象提供对 IPC 协议服务器和客户端相互通信时所用的特定接口的支持。`struct pw_protocol_implementation` 类型提供新建 IPC 协议服务器和发起对 IPC 协议服务器的连接的客户端的操作。

在 `struct pw_context` 对象中，PipeWire 用一个列表维护已注册的 IPC 协议 (位于 *pipewire/src/pipewire/private.h*)，各 IPC 协议用名称标识，如：
```，
struct pw_context {
	struct pw_impl_core *core;		/**< core object */

	struct pw_properties *conf;		/**< configuration of the context */
	struct pw_properties *properties;	/**< properties of the context */
 . . . . . .
	struct spa_list protocol_list;		/**< list of protocols */
 . . . . . .
};
```

实现 IPC 协议的模块通过 `pw_protocol_new()` 函数创建 `pw_protocol` 对象时，新建的 `pw_protocol` 对象会被添加进 `pw_context` 对象的 IPC 协议列表，`pw_protocol_new()` 函数定义 (位于 *pipewire/src/pipewire/protocol.c*) 如下：
```
struct impl {
	struct pw_protocol this;
};
 . . . . . .
SPA_EXPORT
struct pw_protocol *pw_protocol_new(struct pw_context *context,
				    const char *name,
				    size_t user_data_size)
{
	struct pw_protocol *protocol;

	protocol = calloc(1, sizeof(struct impl) + user_data_size);
	if (protocol == NULL)
		return NULL;

	protocol->context = context;
	protocol->name = strdup(name);

	spa_list_init(&protocol->marshal_list);
	spa_list_init(&protocol->server_list);
	spa_list_init(&protocol->client_list);
	spa_hook_list_init(&protocol->listener_list);

	if (user_data_size > 0)
		protocol->user_data = SPA_PTROFF(protocol, sizeof(struct impl), void);

	spa_list_append(&context->protocol_list, &protocol->link);

	pw_log_debug("%p: Created protocol %s", protocol, name);

	return protocol;
}
```

PipeWire 还提供接口操作 `pw_protocol` 对象：`pw_context_find_protocol()` 函数用于在 `pw_context` 对象的 IPC 协议列表中，根据协议名称查找特定 IPC 协议；`pw_protocol_add_marshal()` 和 `pw_protocol_get_marshal()` 函数分别用于向 IPC 协议添加和从  IPC 协议获取特定接口的 marshaller，`pw_protocol` 对象中用一个列表维护这些 marshallers，marshaller 在内部用 `struct marshal` 对象表示，这两个函数操作 `pw_protocol` 的 marshaller 列表；`pw_protocol_get_context()`、`pw_protocol_get_user_data()`、`pw_protocol_get_implementation()` 和 `pw_protocol_get_extension()` 函数分别用于获取 `pw_protocol` 的 `pw_context`、协议实现的私有数据、协议实现和扩展 API；`pw_protocol_add_listener()` 函数用于为 `pw_protocol` 添加监听器；`pw_protocol_destroy()` 函数用于销毁 `pw_protocol`，它通过注册的监听器通知销毁事件，并销毁 IPC 协议服务器、marshaller 等对象，这些函数定义 (位于 *pipewire/src/pipewire/protocol.c*) 如下：
```
struct marshal {
	struct spa_list link;
	const struct pw_protocol_marshal *marshal;
};
 . . . . . .
SPA_EXPORT
struct pw_context *pw_protocol_get_context(struct pw_protocol *protocol)
{
	return protocol->context;
}

SPA_EXPORT
void *pw_protocol_get_user_data(struct pw_protocol *protocol)
{
	return protocol->user_data;
}

SPA_EXPORT
const struct pw_protocol_implementation *
pw_protocol_get_implementation(struct pw_protocol *protocol)
{
	return protocol->implementation;
}

SPA_EXPORT
const void *
pw_protocol_get_extension(struct pw_protocol *protocol)
{
	return protocol->extension;
}

SPA_EXPORT
void pw_protocol_destroy(struct pw_protocol *protocol)
{
	struct impl *impl = SPA_CONTAINER_OF(protocol, struct impl, this);
	struct marshal *marshal, *t1;
	struct pw_protocol_server *server;
	struct pw_protocol_client *client;

	pw_log_debug("%p: destroy", protocol);
	pw_protocol_emit_destroy(protocol);

	spa_hook_list_clean(&protocol->listener_list);

	spa_list_remove(&protocol->link);

	spa_list_consume(server, &protocol->server_list, link)
		pw_protocol_server_destroy(server);

	spa_list_consume(client, &protocol->client_list, link)
		pw_protocol_client_destroy(client);

	spa_list_for_each_safe(marshal, t1, &protocol->marshal_list, link)
		free(marshal);

	free(protocol->name);

	free(impl);
}

SPA_EXPORT
void pw_protocol_add_listener(struct pw_protocol *protocol,
                              struct spa_hook *listener,
                              const struct pw_protocol_events *events,
                              void *data)
{
	spa_hook_list_append(&protocol->listener_list, listener, events, data);
}

SPA_EXPORT
int
pw_protocol_add_marshal(struct pw_protocol *protocol,
			const struct pw_protocol_marshal *marshal)
{
	struct marshal *impl;

	impl = calloc(1, sizeof(struct marshal));
	if (impl == NULL)
		return -errno;

	impl->marshal = marshal;

	spa_list_append(&protocol->marshal_list, &impl->link);

	pw_log_debug("%p: Add marshal %s/%d to protocol %s", protocol,
			marshal->type, marshal->version, protocol->name);

	return 0;
}

SPA_EXPORT
const struct pw_protocol_marshal *
pw_protocol_get_marshal(struct pw_protocol *protocol, const char *type, uint32_t version, uint32_t flags)
{
	struct marshal *impl;

	spa_list_for_each(impl, &protocol->marshal_list, link) {
		if (spa_streq(impl->marshal->type, type) &&
		    (impl->marshal->flags & flags) == flags)
                        return impl->marshal;
        }
	pw_log_debug("%p: No marshal %s/%d for protocol %s", protocol,
			type, version, protocol->name);
	return NULL;
}

SPA_EXPORT
struct pw_protocol *pw_context_find_protocol(struct pw_context *context, const char *name)
{
	struct pw_protocol *protocol;

	spa_list_for_each(protocol, &context->protocol_list, link) {
		if (spa_streq(protocol->name, name))
			return protocol;
	}
	return NULL;
}

```

PipeWire IPC 协议实现提供 `struct pw_protocol_client`、`struct pw_protocol_server`、`struct pw_protocol_marshal`、`struct pw_protocol_implementation` 和 扩展 API 操作的具体实现。

## 原生 PipeWire IPC 协议初始化
原生 PipeWire IPC 协议用于各应用程序和 pipewire 守护进程之间的通信，该 IPC 协议主要由 **module-protocol-native** 模块实现，该模块按照各 pipewire 进程的配置文件的指示加载，如 */usr/share/pipewire/pipewire.conf* 和 */usr/share/pipewire/pipewire-pulse.conf* 等，模块的加载初始化函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
static const struct spa_dict_item module_props[] = {
	{ PW_KEY_MODULE_AUTHOR, "Wim Taymans <wim.taymans@gmail.com>" },
	{ PW_KEY_MODULE_DESCRIPTION, "Native protocol using unix sockets" },
	{ PW_KEY_MODULE_VERSION, PACKAGE_VERSION },
};

/* Required for s390x */
#ifndef SO_PEERSEC
#define SO_PEERSEC 31
#endif

static bool debug_messages = 0;

#define LOCK_SUFFIX     ".lock"
#define LOCK_SUFFIXLEN  5

void pw_protocol_native_init(struct pw_protocol *protocol);
void pw_protocol_native0_init(struct pw_protocol *protocol);

struct protocol_data {
	struct pw_impl_module *module;
	struct spa_hook module_listener;
	struct pw_protocol *protocol;

	struct server *local;
};
 . . . . . .
static const struct pw_protocol_implementation protocol_impl = {
	PW_VERSION_PROTOCOL_IMPLEMENTATION,
	.new_client = impl_new_client,
	.add_server = impl_add_server,
};
 . . . . . .
static const struct pw_protocol_native_ext protocol_ext_impl = {
	PW_VERSION_PROTOCOL_NATIVE_EXT,
	.begin_proxy = impl_ext_begin_proxy,
	.add_proxy_fd = impl_ext_add_proxy_fd,
	.get_proxy_fd = impl_ext_get_proxy_fd,
	.end_proxy = impl_ext_end_proxy,
	.begin_resource = impl_ext_begin_resource,
	.add_resource_fd = impl_ext_add_resource_fd,
	.get_resource_fd = impl_ext_get_resource_fd,
	.end_resource = impl_ext_end_resource,
};

static void module_destroy(void *data)
{
	struct protocol_data *d = data;

	spa_hook_remove(&d->module_listener);

	pw_protocol_destroy(d->protocol);
}

static const struct pw_impl_module_events module_events = {
	PW_VERSION_IMPL_MODULE_EVENTS,
	.destroy = module_destroy,
};

static int need_server(struct pw_context *context, const struct spa_dict *props)
{
	const char *val = NULL;

	if (props)
		val = spa_dict_lookup(props, PW_KEY_CORE_DAEMON);
	if (val == NULL)
		val = getenv("PIPEWIRE_DAEMON");
	if (val && pw_properties_parse_bool(val))
		return 1;
	return 0;
}

SPA_EXPORT
int pipewire__module_init(struct pw_impl_module *module, const char *args)
{
	struct pw_context *context = pw_impl_module_get_context(module);
	struct pw_protocol *this;
	struct protocol_data *d;
	const struct pw_properties *props;
	int res;

	PW_LOG_TOPIC_INIT(mod_topic);
	PW_LOG_TOPIC_INIT(mod_topic_connection);

	if (pw_context_find_protocol(context, PW_TYPE_INFO_PROTOCOL_Native) != NULL)
		return 0;

	this = pw_protocol_new(context, PW_TYPE_INFO_PROTOCOL_Native, sizeof(struct protocol_data));
	if (this == NULL)
		return -errno;

	debug_messages = mod_topic_connection->level >= SPA_LOG_LEVEL_DEBUG;

	this->implementation = &protocol_impl;
	this->extension = &protocol_ext_impl;

	pw_protocol_native_init(this);
	pw_protocol_native0_init(this);

	pw_log_debug("%p: new debug:%d", this, debug_messages);

	d = pw_protocol_get_user_data(this);
	d->protocol = this;
	d->module = module;

	props = pw_context_get_properties(context);
	d->local = create_server(this, context->core, &props->dict);

	if (need_server(context, &props->dict)) {
		if (impl_add_server(this, context->core, &props->dict) == NULL) {
			res = -errno;
			goto error_cleanup;
		}
	}

	pw_impl_module_add_listener(module, &d->module_listener, &module_events, d);

	pw_impl_module_update_properties(module, &SPA_DICT_INIT_ARRAY(module_props));

	return 0;

error_cleanup:
	pw_protocol_destroy(this);
	return res;
}
```

**module-protocol-native** 模块加载时，执行如下操作：

1. 调用 `pw_context_find_protocol()` 函数在 `pw_context` 中查找名称为 `PW_TYPE_INFO_PROTOCOL_Native` 的 IPC 协议，如果找到，说明 IPC 协议已经注册过，则返回，否则继续执行。 
2. 调用 `pw_protocol_new()` 函数创建 `struct pw_protocol` 对象，并将其放进 `pw_context` 的 IPC 协议列表中，实际创建的是结尾部分为 `protocol_data` 对象的 `struct pw_protocol`；
3. 为 `pw_protocol` 对象初始化协议实现和扩展 API 字段；
4. 调用 `pw_protocol_native_init()` 和 `pw_protocol_native0_init()` 分别向 `pw_protocol` 注册版本 3 和版本 0 支持的接口的 marshallers；
5. 调用 `create_server()` 函数创建用于进程内原生 IPC 协议通信的服务器 `server`/`pw_protocol_server` 对象；
6. 根据 pipewire 各进程的配置，确定是否需要起跨进程原生 IPC 协议通信的 socket 服务器，需要时，调用 `impl_add_server()` 函数起 socket 服务器；
7. 向模块注册监听器，以在模块卸载并销毁时得到通知，释放资源，主要是销毁 `pw_protocol` 对象；
8. 用模块的信息更新模块的相关属性。

通过 `pw_protocol_native_init()` 函数，可以看到原生 PipeWire IPC 协议支持的主要 IPC 接口，该函数定义 (位于 *pipewire/src/modules/module-protocol-native/protocol-native.c*) 如下：
```
static const struct pw_core_methods pw_protocol_native_core_method_marshal = {
	PW_VERSION_CORE_METHODS,
	.add_listener = &core_method_marshal_add_listener,
	.hello = &core_method_marshal_hello,
	.sync = &core_method_marshal_sync,
	.pong = &core_method_marshal_pong,
	.error = &core_method_marshal_error,
	.get_registry = &core_method_marshal_get_registry,
	.create_object = &core_method_marshal_create_object,
	.destroy = &core_method_marshal_destroy,
};

static const struct pw_protocol_native_demarshal pw_protocol_native_core_method_demarshal[PW_CORE_METHOD_NUM] = {
	[PW_CORE_METHOD_ADD_LISTENER] = { NULL, 0, },
	[PW_CORE_METHOD_HELLO] = { &core_method_demarshal_hello, 0, },
	[PW_CORE_METHOD_SYNC] = { &core_method_demarshal_sync, 0, },
	[PW_CORE_METHOD_PONG] = { &core_method_demarshal_pong, 0, },
	[PW_CORE_METHOD_ERROR] = { &core_method_demarshal_error, 0, },
	[PW_CORE_METHOD_GET_REGISTRY] = { &core_method_demarshal_get_registry, 0, },
	[PW_CORE_METHOD_CREATE_OBJECT] = { &core_method_demarshal_create_object, 0, },
	[PW_CORE_METHOD_DESTROY] = { &core_method_demarshal_destroy, 0, }
};

static const struct pw_core_events pw_protocol_native_core_event_marshal = {
	PW_VERSION_CORE_EVENTS,
	.info = &core_event_marshal_info,
	.done = &core_event_marshal_done,
	.ping = &core_event_marshal_ping,
	.error = &core_event_marshal_error,
	.remove_id = &core_event_marshal_remove_id,
	.bound_id = &core_event_marshal_bound_id,
	.add_mem = &core_event_marshal_add_mem,
	.remove_mem = &core_event_marshal_remove_mem,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_core_event_demarshal[PW_CORE_EVENT_NUM] =
{
	[PW_CORE_EVENT_INFO] = { &core_event_demarshal_info, 0, },
	[PW_CORE_EVENT_DONE] = { &core_event_demarshal_done, 0, },
	[PW_CORE_EVENT_PING] = { &core_event_demarshal_ping, 0, },
	[PW_CORE_EVENT_ERROR] = { &core_event_demarshal_error, 0, },
	[PW_CORE_EVENT_REMOVE_ID] = { &core_event_demarshal_remove_id, 0, },
	[PW_CORE_EVENT_BOUND_ID] = { &core_event_demarshal_bound_id, 0, },
	[PW_CORE_EVENT_ADD_MEM] = { &core_event_demarshal_add_mem, 0, },
	[PW_CORE_EVENT_REMOVE_MEM] = { &core_event_demarshal_remove_mem, 0, },
};

static const struct pw_protocol_marshal pw_protocol_native_core_marshal = {
	PW_TYPE_INTERFACE_Core,
	PW_VERSION_CORE,
	0,
	PW_CORE_METHOD_NUM,
	PW_CORE_EVENT_NUM,
	.client_marshal = &pw_protocol_native_core_method_marshal,
	.server_demarshal = pw_protocol_native_core_method_demarshal,
	.server_marshal = &pw_protocol_native_core_event_marshal,
	.client_demarshal = pw_protocol_native_core_event_demarshal,
};

static const struct pw_registry_methods pw_protocol_native_registry_method_marshal = {
	PW_VERSION_REGISTRY_METHODS,
	.add_listener = &registry_method_marshal_add_listener,
	.bind = &registry_marshal_bind,
	.destroy = &registry_marshal_destroy,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_registry_method_demarshal[PW_REGISTRY_METHOD_NUM] =
{
	[PW_REGISTRY_METHOD_ADD_LISTENER] = { NULL, 0, },
	[PW_REGISTRY_METHOD_BIND] = { &registry_demarshal_bind, 0, },
	[PW_REGISTRY_METHOD_DESTROY] = { &registry_demarshal_destroy, 0, },
};

static const struct pw_registry_events pw_protocol_native_registry_event_marshal = {
	PW_VERSION_REGISTRY_EVENTS,
	.global = &registry_marshal_global,
	.global_remove = &registry_marshal_global_remove,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_registry_event_demarshal[PW_REGISTRY_EVENT_NUM] =
{
	[PW_REGISTRY_EVENT_GLOBAL] = { &registry_demarshal_global, 0, },
	[PW_REGISTRY_EVENT_GLOBAL_REMOVE] = { &registry_demarshal_global_remove, 0, }
};

static const struct pw_protocol_marshal pw_protocol_native_registry_marshal = {
	PW_TYPE_INTERFACE_Registry,
	PW_VERSION_REGISTRY,
	0,
	PW_REGISTRY_METHOD_NUM,
	PW_REGISTRY_EVENT_NUM,
	.client_marshal = &pw_protocol_native_registry_method_marshal,
	.server_demarshal = pw_protocol_native_registry_method_demarshal,
	.server_marshal = &pw_protocol_native_registry_event_marshal,
	.client_demarshal = pw_protocol_native_registry_event_demarshal,
};

static const struct pw_module_events pw_protocol_native_module_event_marshal = {
	PW_VERSION_MODULE_EVENTS,
	.info = &module_marshal_info,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_module_event_demarshal[PW_MODULE_EVENT_NUM] =
{
	[PW_MODULE_EVENT_INFO] = { &module_demarshal_info, 0, },
};


static const struct pw_module_methods pw_protocol_native_module_method_marshal = {
	PW_VERSION_MODULE_METHODS,
	.add_listener = &module_method_marshal_add_listener,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_module_method_demarshal[PW_MODULE_METHOD_NUM] =
{
	[PW_MODULE_METHOD_ADD_LISTENER] = { NULL, 0, },
};

static const struct pw_protocol_marshal pw_protocol_native_module_marshal = {
	PW_TYPE_INTERFACE_Module,
	PW_VERSION_MODULE,
	0,
	PW_MODULE_METHOD_NUM,
	PW_MODULE_EVENT_NUM,
	.client_marshal = &pw_protocol_native_module_method_marshal,
	.server_demarshal = pw_protocol_native_module_method_demarshal,
	.server_marshal = &pw_protocol_native_module_event_marshal,
	.client_demarshal = pw_protocol_native_module_event_demarshal,
};

static const struct pw_factory_events pw_protocol_native_factory_event_marshal = {
	PW_VERSION_FACTORY_EVENTS,
	.info = &factory_marshal_info,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_factory_event_demarshal[PW_FACTORY_EVENT_NUM] =
{
	[PW_FACTORY_EVENT_INFO] = { &factory_demarshal_info, 0, },
};

static const struct pw_factory_methods pw_protocol_native_factory_method_marshal = {
	PW_VERSION_FACTORY_METHODS,
	.add_listener = &factory_method_marshal_add_listener,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_factory_method_demarshal[PW_FACTORY_METHOD_NUM] =
{
	[PW_FACTORY_METHOD_ADD_LISTENER] = { NULL, 0, },
};

static const struct pw_protocol_marshal pw_protocol_native_factory_marshal = {
	PW_TYPE_INTERFACE_Factory,
	PW_VERSION_FACTORY,
	0,
	PW_FACTORY_METHOD_NUM,
	PW_FACTORY_EVENT_NUM,
	.client_marshal = &pw_protocol_native_factory_method_marshal,
	.server_demarshal = pw_protocol_native_factory_method_demarshal,
	.server_marshal = &pw_protocol_native_factory_event_marshal,
	.client_demarshal = pw_protocol_native_factory_event_demarshal,
};

static const struct pw_device_methods pw_protocol_native_device_method_marshal = {
	PW_VERSION_DEVICE_METHODS,
	.add_listener = &device_method_marshal_add_listener,
	.subscribe_params = &device_marshal_subscribe_params,
	.enum_params = &device_marshal_enum_params,
	.set_param = &device_marshal_set_param,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_device_method_demarshal[PW_DEVICE_METHOD_NUM] = {
	[PW_DEVICE_METHOD_ADD_LISTENER] = { NULL, 0, },
	[PW_DEVICE_METHOD_SUBSCRIBE_PARAMS] = { &device_demarshal_subscribe_params, 0, },
	[PW_DEVICE_METHOD_ENUM_PARAMS] = { &device_demarshal_enum_params, 0, },
	[PW_DEVICE_METHOD_SET_PARAM] = { &device_demarshal_set_param, PW_PERM_W, },
};

static const struct pw_device_events pw_protocol_native_device_event_marshal = {
	PW_VERSION_DEVICE_EVENTS,
	.info = &device_marshal_info,
	.param = &device_marshal_param,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_device_event_demarshal[PW_DEVICE_EVENT_NUM] = {
	[PW_DEVICE_EVENT_INFO] = { &device_demarshal_info, 0, },
	[PW_DEVICE_EVENT_PARAM] = { &device_demarshal_param, 0, }
};

static const struct pw_protocol_marshal pw_protocol_native_device_marshal = {
	PW_TYPE_INTERFACE_Device,
	PW_VERSION_DEVICE,
	0,
	PW_DEVICE_METHOD_NUM,
	PW_DEVICE_EVENT_NUM,
	.client_marshal = &pw_protocol_native_device_method_marshal,
	.server_demarshal = pw_protocol_native_device_method_demarshal,
	.server_marshal = &pw_protocol_native_device_event_marshal,
	.client_demarshal = pw_protocol_native_device_event_demarshal,
};

static const struct pw_node_methods pw_protocol_native_node_method_marshal = {
	PW_VERSION_NODE_METHODS,
	.add_listener = &node_method_marshal_add_listener,
	.subscribe_params = &node_marshal_subscribe_params,
	.enum_params = &node_marshal_enum_params,
	.set_param = &node_marshal_set_param,
	.send_command = &node_marshal_send_command,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_node_method_demarshal[PW_NODE_METHOD_NUM] =
{
	[PW_NODE_METHOD_ADD_LISTENER] = { NULL, 0, },
	[PW_NODE_METHOD_SUBSCRIBE_PARAMS] = { &node_demarshal_subscribe_params, 0, },
	[PW_NODE_METHOD_ENUM_PARAMS] = { &node_demarshal_enum_params, 0, },
	[PW_NODE_METHOD_SET_PARAM] = { &node_demarshal_set_param, PW_PERM_W, },
	[PW_NODE_METHOD_SEND_COMMAND] = { &node_demarshal_send_command, PW_PERM_W, },
};

static const struct pw_node_events pw_protocol_native_node_event_marshal = {
	PW_VERSION_NODE_EVENTS,
	.info = &node_marshal_info,
	.param = &node_marshal_param,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_node_event_demarshal[PW_NODE_EVENT_NUM] = {
	[PW_NODE_EVENT_INFO] = { &node_demarshal_info, 0, },
	[PW_NODE_EVENT_PARAM] = { &node_demarshal_param, 0, }
};

static const struct pw_protocol_marshal pw_protocol_native_node_marshal = {
	PW_TYPE_INTERFACE_Node,
	PW_VERSION_NODE,
	0,
	PW_NODE_METHOD_NUM,
	PW_NODE_EVENT_NUM,
	.client_marshal = &pw_protocol_native_node_method_marshal,
	.server_demarshal = pw_protocol_native_node_method_demarshal,
	.server_marshal = &pw_protocol_native_node_event_marshal,
	.client_demarshal = pw_protocol_native_node_event_demarshal,
};


static const struct pw_port_methods pw_protocol_native_port_method_marshal = {
	PW_VERSION_PORT_METHODS,
	.add_listener = &port_method_marshal_add_listener,
	.subscribe_params = &port_marshal_subscribe_params,
	.enum_params = &port_marshal_enum_params,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_port_method_demarshal[PW_PORT_METHOD_NUM] =
{
	[PW_PORT_METHOD_ADD_LISTENER] = { NULL, 0, },
	[PW_PORT_METHOD_SUBSCRIBE_PARAMS] = { &port_demarshal_subscribe_params, 0, },
	[PW_PORT_METHOD_ENUM_PARAMS] = { &port_demarshal_enum_params, 0, },
};

static const struct pw_port_events pw_protocol_native_port_event_marshal = {
	PW_VERSION_PORT_EVENTS,
	.info = &port_marshal_info,
	.param = &port_marshal_param,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_port_event_demarshal[PW_PORT_EVENT_NUM] =
{
	[PW_PORT_EVENT_INFO] = { &port_demarshal_info, 0, },
	[PW_PORT_EVENT_PARAM] = { &port_demarshal_param, 0, }
};

static const struct pw_protocol_marshal pw_protocol_native_port_marshal = {
	PW_TYPE_INTERFACE_Port,
	PW_VERSION_PORT,
	0,
	PW_PORT_METHOD_NUM,
	PW_PORT_EVENT_NUM,
	.client_marshal = &pw_protocol_native_port_method_marshal,
	.server_demarshal = pw_protocol_native_port_method_demarshal,
	.server_marshal = &pw_protocol_native_port_event_marshal,
	.client_demarshal = pw_protocol_native_port_event_demarshal,
};

static const struct pw_client_methods pw_protocol_native_client_method_marshal = {
	PW_VERSION_CLIENT_METHODS,
	.add_listener = &client_method_marshal_add_listener,
	.error = &client_marshal_error,
	.update_properties = &client_marshal_update_properties,
	.get_permissions = &client_marshal_get_permissions,
	.update_permissions = &client_marshal_update_permissions,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_client_method_demarshal[PW_CLIENT_METHOD_NUM] =
{
	[PW_CLIENT_METHOD_ADD_LISTENER] = { NULL, 0, },
	[PW_CLIENT_METHOD_ERROR] = { &client_demarshal_error, PW_PERM_W, },
	[PW_CLIENT_METHOD_UPDATE_PROPERTIES] = { &client_demarshal_update_properties, PW_PERM_W, },
	[PW_CLIENT_METHOD_GET_PERMISSIONS] = { &client_demarshal_get_permissions, 0, },
	[PW_CLIENT_METHOD_UPDATE_PERMISSIONS] = { &client_demarshal_update_permissions, PW_PERM_W, },
};

static const struct pw_client_events pw_protocol_native_client_event_marshal = {
	PW_VERSION_CLIENT_EVENTS,
	.info = &client_marshal_info,
	.permissions = &client_marshal_permissions,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_client_event_demarshal[PW_CLIENT_EVENT_NUM] =
{
	[PW_CLIENT_EVENT_INFO] = { &client_demarshal_info, 0, },
	[PW_CLIENT_EVENT_PERMISSIONS] = { &client_demarshal_permissions, 0, }
};

static const struct pw_protocol_marshal pw_protocol_native_client_marshal = {
	PW_TYPE_INTERFACE_Client,
	PW_VERSION_CLIENT,
	0,
	PW_CLIENT_METHOD_NUM,
	PW_CLIENT_EVENT_NUM,
	.client_marshal = &pw_protocol_native_client_method_marshal,
	.server_demarshal = pw_protocol_native_client_method_demarshal,
	.server_marshal = &pw_protocol_native_client_event_marshal,
	.client_demarshal = pw_protocol_native_client_event_demarshal,
};


static const struct pw_link_methods pw_protocol_native_link_method_marshal = {
	PW_VERSION_LINK_METHODS,
	.add_listener = &link_method_marshal_add_listener,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_link_method_demarshal[PW_LINK_METHOD_NUM] =
{
	[PW_LINK_METHOD_ADD_LISTENER] = { NULL, 0, },
};

static const struct pw_link_events pw_protocol_native_link_event_marshal = {
	PW_VERSION_LINK_EVENTS,
	.info = &link_marshal_info,
};

static const struct pw_protocol_native_demarshal
pw_protocol_native_link_event_demarshal[PW_LINK_EVENT_NUM] =
{
	[PW_LINK_EVENT_INFO] = { &link_demarshal_info, 0, }
};

static const struct pw_protocol_marshal pw_protocol_native_link_marshal = {
	PW_TYPE_INTERFACE_Link,
	PW_VERSION_LINK,
	0,
	PW_LINK_METHOD_NUM,
	PW_LINK_EVENT_NUM,
	.client_marshal = &pw_protocol_native_link_method_marshal,
	.server_demarshal = pw_protocol_native_link_method_demarshal,
	.server_marshal = &pw_protocol_native_link_event_marshal,
	.client_demarshal = pw_protocol_native_link_event_demarshal,
};

void pw_protocol_native_init(struct pw_protocol *protocol)
{
	pw_protocol_add_marshal(protocol, &pw_protocol_native_core_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_registry_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_module_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_device_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_node_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_port_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_factory_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_client_marshal);
	pw_protocol_add_marshal(protocol, &pw_protocol_native_link_marshal);
}
```

`pw_protocol_native_init()` 函数向 `pw_protocol` 注册支持的接口的 marshallers。Marshaller 用接口类型名称、版本号和标记标识。Marshaller 描述 PipeWire IPC 协议通信服务器和客户端双方，对于特定接口，两个方向的 4 个操作集合，分别为客户端向服务器发送请求，服务器对客户端发送的请求的处理，服务器向客户端发送请求，客户端对服务器发送的请求的处理，在 `pw_protocol_marshal` 对象中，这 4 个操作集合分别用 `client_marshal`、`server_demarshal`、`server_marshal` 和 `client_demarshal` 描述。对于这 4 个操作集合，`client_marshal` 为接口方法集合的代理实现，如  `pw_core_methods`，`server_marshal` 为事件方法集合的代理实现，如 `pw_core_events`，接口方法集合的每个方法和事件方法集合的每个方法都会被分配一个索引，`server_demarshal` 和 `client_demarshal` 都用 `struct pw_protocol_native_demarshal` 对象数组描述，其中对应索引位置的 `struct pw_protocol_native_demarshal` 对象描述相应的接口方法集合的方法或事件方法集合的方法发起的请求的处理函数。`struct pw_protocol_native_demarshal` 类型定义 (位于 *pipewire/src/pipewire/extensions/protocol-native.h*) 如下：
```
struct pw_protocol_native_demarshal {
	int (*func) (void *object, const struct pw_protocol_native_message *msg);
	uint32_t permissions;
	uint32_t flags;
};
```

从 `pw_protocol_native_init()` 函数可以看到，PipeWire 通过 **module-protocol-native** 模块支持的原生 IPC 协议接口主要有如下这些：

 * **PipeWire:Interface:Core**
 * **PipeWire:Interface:Registry**
 * **PipeWire:Interface:Module**
 * **PipeWire:Interface:Device**
 * **PipeWire:Interface:Node**
 * **PipeWire:Interface:Port**
 * **PipeWire:Interface:Factory**
 * **PipeWire:Interface:Client**
 * **PipeWire:Interface:Link**

这些接口将在不同类型对象的原生 IPC 协议通信中被用到，它们不是原生 PipeWire IPC 协议支持的全部接口。

`pw_protocol_native0_init()` 函数初始化 v0 版本的各个接口的 marshaller，过程与 `pw_protocol_native_init()` 类似。

在 **module-protocol-native** 模块初始化过程中，`create_server()` 函数创建一个用于进程内原生 IPC 协议通信的服务器， `impl_add_server()` 函数起 socket 服务器用于跨进程原生 IPC 协议通信，它们用来创建 IPC 通信的消息通道。这两个函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
struct server {
	struct pw_protocol_server this;

	int fd_lock;
	struct sockaddr_un addr;
	char lock_addr[UNIX_PATH_MAX + LOCK_SUFFIXLEN];

	struct pw_loop *loop;
	struct spa_source *source;
	struct spa_source *resume;
	unsigned int activated:1;
};
 . . . . . .
static const char *
get_runtime_dir(void)
{
	const char *runtime_dir;

	runtime_dir = getenv("PIPEWIRE_RUNTIME_DIR");
	if (runtime_dir == NULL)
		runtime_dir = getenv("XDG_RUNTIME_DIR");
	if (runtime_dir == NULL)
		runtime_dir = getenv("USERPROFILE");
	return runtime_dir;
}


static int init_socket_name(struct server *s, const char *name)
{
	int name_size;
	const char *runtime_dir;
	bool path_is_absolute;

	path_is_absolute = name[0] == '/';

	runtime_dir = get_runtime_dir();

	pw_log_debug("name:%s runtime_dir:%s", name, runtime_dir);

	if (runtime_dir == NULL && !path_is_absolute) {
		pw_log_error("server %p: name %s is not an absolute path and no runtime dir found. "
				"Set one of PIPEWIRE_RUNTIME_DIR, XDG_RUNTIME_DIR or "
				"USERPROFILE in the environment", s, name);
		return -ENOENT;
	}

	s->addr.sun_family = AF_LOCAL;
	if (path_is_absolute)
		name_size = snprintf(s->addr.sun_path, sizeof(s->addr.sun_path),
			     "%s", name) + 1;
	else
		name_size = snprintf(s->addr.sun_path, sizeof(s->addr.sun_path),
			     "%s/%s", runtime_dir, name) + 1;

	if (name_size > (int) sizeof(s->addr.sun_path)) {
		if (path_is_absolute)
			pw_log_error("server %p: socket path \"%s\" plus null terminator exceeds %i bytes",
				s, name, (int) sizeof(s->addr.sun_path));
		else
			pw_log_error("server %p: socket path \"%s/%s\" plus null terminator exceeds %i bytes",
				s, runtime_dir, name, (int) sizeof(s->addr.sun_path));
		*s->addr.sun_path = 0;
		return -ENAMETOOLONG;
	}
	return 0;
}

static int lock_socket(struct server *s)
{
	int res;

	snprintf(s->lock_addr, sizeof(s->lock_addr), "%s%s", s->addr.sun_path, LOCK_SUFFIX);

	s->fd_lock = open(s->lock_addr, O_CREAT | O_CLOEXEC,
			  (S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP));

	if (s->fd_lock < 0) {
		res = -errno;
		pw_log_error("server %p: unable to open lockfile '%s': %m", s, s->lock_addr);
		goto err;
	}

	if (flock(s->fd_lock, LOCK_EX | LOCK_NB) < 0) {
		res = -errno;
		pw_log_error("server %p: unable to lock lockfile '%s': %m"
				" (maybe another daemon is running)",
				s, s->lock_addr);
		goto err_fd;
	}
	return 0;

err_fd:
	close(s->fd_lock);
	s->fd_lock = -1;
err:
	*s->lock_addr = 0;
	*s->addr.sun_path = 0;
	return res;
}
 . . . . . .
static int add_socket(struct pw_protocol *protocol, struct server *s)
{
	socklen_t size;
	int fd = -1, res;
	bool activated = false;

#ifdef HAVE_SYSTEMD
	{
		int i, n = sd_listen_fds(0);
		for (i = 0; i < n; ++i) {
			if (sd_is_socket_unix(SD_LISTEN_FDS_START + i, SOCK_STREAM,
						1, s->addr.sun_path, 0) > 0) {
				fd = SD_LISTEN_FDS_START + i;
				activated = true;
				pw_log_info("server %p: Found socket activation socket for '%s'",
						s, s->addr.sun_path);
				break;
			}
		}
	}
#endif

	if (fd < 0) {
		struct stat socket_stat;

		if ((fd = socket(PF_LOCAL, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0)) < 0) {
			res = -errno;
			goto error;
		}
		if (stat(s->addr.sun_path, &socket_stat) < 0) {
			if (errno != ENOENT) {
				res = -errno;
				pw_log_error("server %p: stat %s failed with error: %m",
						s, s->addr.sun_path);
				goto error_close;
			}
		} else if (socket_stat.st_mode & S_IWUSR || socket_stat.st_mode & S_IWGRP) {
			unlink(s->addr.sun_path);
		}

		size = offsetof(struct sockaddr_un, sun_path) + strlen(s->addr.sun_path);
		if (bind(fd, (struct sockaddr *) &s->addr, size) < 0) {
			res = -errno;
			pw_log_error("server %p: bind() failed with error: %m", s);
			goto error_close;
		}

		if (listen(fd, 128) < 0) {
			res = -errno;
			pw_log_error("server %p: listen() failed with error: %m", s);
			goto error_close;
		}
	}

	s->activated = activated;
	s->loop = pw_context_get_main_loop(protocol->context);
	if (s->loop == NULL) {
		res = -errno;
		goto error_close;
	}
	s->source = pw_loop_add_io(s->loop, fd, SPA_IO_IN, true, socket_data, s);
	if (s->source == NULL) {
		res = -errno;
		goto error_close;
	}
	return 0;

error_close:
	close(fd);
error:
	return res;

}
 . . . . . .
static const char *
get_server_name(const struct spa_dict *props)
{
	const char *name = NULL;

	if (props)
		name = spa_dict_lookup(props, PW_KEY_CORE_NAME);
	if (name == NULL)
		name = getenv("PIPEWIRE_CORE");
	if (name == NULL)
		name = PW_DEFAULT_REMOTE;
	return name;
}

static struct server *
create_server(struct pw_protocol *protocol,
		struct pw_impl_core *core,
                const struct spa_dict *props)
{
	struct pw_protocol_server *this;
	struct server *s;

	if ((s = calloc(1, sizeof(struct server))) == NULL)
		return NULL;

	s->fd_lock = -1;

	this = &s->this;
	this->protocol = protocol;
	this->core = core;
	spa_list_init(&this->client_list);
	this->destroy = destroy_server;

	spa_list_append(&protocol->server_list, &this->link);

	pw_log_warn("%p: created server %p", protocol, this);

	return s;
}

static struct pw_protocol_server *
impl_add_server(struct pw_protocol *protocol,
		struct pw_impl_core *core,
                const struct spa_dict *props)
{
	struct pw_protocol_server *this;
	struct server *s;
	const char *name;
	int res;

	if ((s = create_server(protocol, core, props)) == NULL)
		return NULL;

	this = &s->this;

	name = get_server_name(props);

	if ((res = init_socket_name(s, name)) < 0)
		goto error;

	if ((res = lock_socket(s)) < 0)
		goto error;

	if ((res = add_socket(protocol, s)) < 0)
		goto error;

	if ((s->resume = pw_loop_add_event(s->loop, do_resume, s)) == NULL)
		goto error;

	pw_log_info("%p: Listening on '%s'", protocol, name);

	return this;

error:
	destroy_server(this);
	errno = -res;
	return NULL;
}
```

`create_server()` 函数创建 `struct server` 对象，并把它挂在 `pw_protocol` 的服务器列表中。`struct server` 类型可以视作 `struct pw_protocol_server` 类型的子类。这里还看不到太多与 IPC 通信的消息通道有关的内容。

`impl_add_server()` 函数则会创建 unix domain socket 服务器，具体过程如下：

1. 调用 `create_server()` 函数创建 `struct server` 对象。
2. 获得服务器名，服务器名默认为 `pipewire-0`，但也可以从 `pw_context` 的 **core.name** 配置，或环境变量 **PIPEWIRE_CORE** 获取。
3. 调用 `init_socket_name()` 函数获得完整的 unix domain socket 路径，完整路径由运行时目录路径和服务器名组成，如 */run/user/1000/pipewire-0*，运行时目录路径由 **PIPEWIRE_RUNTIME_DIR**、**XDG_RUNTIME_DIR** 或 **USERPROFILE** 环境变量获取。
4. 锁定 socket，通过锁定锁文件实现，如 */run/user/1000/pipewire-0.lock*。
5. 调用 `add_socket()` 为服务器添加 socket，socket 用文件描述符表示。如果 pipewire 进程是通过 systemd 启动的，systemd 可以给 pipewire 进程传递打开的 socket 的文件描述符，`add_socket()` 通过 `sd_listen_fds()` 获得 systemd 传递的文件描述符的个数，并通过 `sd_is_socket_unix()` 检查文件描述符是否是要打开的 unix domain socket 的路径，如果找到了这样的文件描述符，则把它作为服务器的 socket，否则打开新的 unix domain socket，绑定到获得的路径，并监听它。有了 socket 的文件描述符之后，在主事件循环中为它添加 IO 事件源，socket 上的输入 IO 事件处理函数为 `socket_data()`。
6. 在主事件循环中为服务器添加 resume 事件事件源。

在 pipewire 守护进程中，**module-protocol-native** 模块加载完成后，原生 PipeWire IPC 协议服务器端初始化完成。其它 pipewire 进程在加载 **module-protocol-native** 模块时，不会调用 `impl_add_server()` 函数。

不同的 pipewire 进程，对于原生 PipeWire IPC 的需求不同，因而注册的 marshaller 也不同，pipewire 守护进程支持的接口，还有如下这些：
 * **PipeWire:Interface:Profiler** (*pipewire/src/modules/module-profiler/protocol-native.c*)
 * **PipeWire:Interface:Metadata** (*pipewire/src/modules/module-metadata/protocol-native.c*)
 * **PipeWire:Interface:ClientNode** 的 v0 和 v4 (*pipewire/src/modules/module-client-node/protocol-native.c*)
 * **Spa:Pointer:Interface:Device** (*pipewire/src/modules/module-client-device/protocol-native.c*)
 * **PipeWire:Interface:ClientEndpoint**、**PipeWire:Interface:ClientSession**、**PipeWire:Interface:EndpointLink**、**PipeWire:Interface:EndpointStream**、**PipeWire:Interface:Endpoint**、**PipeWire:Interface:Session**、**PipeWire:Interface:EndpointLink** 和 **PipeWire:Interface:EndpointStream** (*pipewire/src/modules/module-session-manager/protocol-native.c*)

媒体会话管理器支持的接口，还有如下这些：
 * **PipeWire:Interface:ClientNode** 的 v0 和 v4 (*pipewire/src/modules/module-client-node/protocol-native.c*)
 * **Spa:Pointer:Interface:Device** (*pipewire/src/modules/module-client-device/protocol-native.c*)
 * **PipeWire:Interface:Metadata** (*pipewire/src/modules/module-metadata/protocol-native.c*)
 * **PipeWire:Interface:ClientEndpoint**、**PipeWire:Interface:ClientSession**、**PipeWire:Interface:EndpointLink**、**PipeWire:Interface:EndpointStream**、**PipeWire:Interface:Endpoint**、**PipeWire:Interface:Session**、**PipeWire:Interface:EndpointLink** 和 **PipeWire:Interface:EndpointStream** (*pipewire/src/modules/module-session-manager/protocol-native.c*)

pipewire-pulse 守护进程支持的接口，还有如下这些：
 * **PipeWire:Interface:ClientNode** 的 v0 和 v4 (*pipewire/src/modules/module-client-node/protocol-native.c*)
 * **PipeWire:Interface:Metadata** (*pipewire/src/modules/module-metadata/protocol-native.c*)

通过各个模块来支持的不同接口的 marshaller 注册完成，原生 PipeWire IPC 协议的初始化即告完成。

## 原生 PipeWire IPC 协议客户端发起连接
Core 接口是 PipeWire 客户端的核心接口，提供了与 PipeWire 守护进程通信、资源管理和事件处理的基础功能。Core 接口的核心对象是 `struct pw_core`，代表与 PipeWire 守护进程的连接，`struct pw_core` 类型定义 (位于 *pipewire/src/pipewire/private.h*) 如下：
```
struct pw_proxy {
	struct spa_interface impl;	/**< object implementation */

	struct pw_core *core;		/**< the owner core of this proxy */

	uint32_t id;			/**< client side id */
	const char *type;		/**< type of the interface */
	uint32_t version;		/**< client side version */
	uint32_t bound_id;		/**< global id we are bound to */
	int refcount;
	unsigned int zombie:1;		/**< proxy is removed locally and waiting to
					  *  be removed from server */
	unsigned int removed:1;		/**< proxy was removed from server */
	unsigned int destroyed:1;	/**< proxy was destroyed by client */
	unsigned int in_map:1;		/**< proxy is in core object map */

	struct spa_hook_list listener_list;
	struct spa_hook_list object_listener_list;

	const struct pw_protocol_marshal *marshal;	/**< protocol specific marshal functions */

	void *user_data;		/**< extra user data */
};

struct pw_core {
	struct pw_proxy proxy;

	struct pw_context *context;		/**< context */
	struct spa_list link;			/**< link in context core_list */
	struct pw_properties *properties;	/**< extra properties */

	struct pw_mempool *pool;		/**< memory pool */
	struct pw_core *core;			/**< proxy for the core object */
	struct spa_hook core_listener;
	struct spa_hook proxy_core_listener;

	struct pw_map objects;			/**< map of client side proxy objects
						 *   indexed with the client id */
	struct pw_client *client;		/**< proxy for the client object */

	struct spa_list stream_list;		/**< list of \ref pw_stream objects */
	struct spa_list filter_list;		/**< list of \ref pw_stream objects */

	struct pw_protocol_client *conn;	/**< the protocol client connection */
	int recv_seq;				/**< last received sequence number */
	int send_seq;				/**< last protocol result code */
	uint64_t recv_generation;		/**< last received registry generation */

	unsigned int removed:1;
	unsigned int destroyed:1;

	void *user_data;			/**< extra user data */
};
```

`struct pw_core` 对象为 pipewire 守护进程中 `pw_impl_core` 对象的代理，所代理对象的接口方法集合由 `proxy.impl` 字段描述，接口的 marshaller 由 `proxy.marshal` 字段描述。所有的 `struct pw_core` 对象通过其 `link` 字段，挂在 `pw_context` 的 core 列表中。底层的 IPC 协议连接通道由 `struct pw_protocol_client` 类型的 `conn` 字段描述。

PipeWire 应用程序（包括媒体会话管理器、pipewire-pulse 守护进程和原生 PipeWire 客户端应用程序）与 pipewire 守护进程之间的原生 IPC 协议连接的建立，从 `pw_context_connect()` 函数被调用开始，它创建 `struct pw_core` 对象。`pw_context_connect()` 函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
static struct pw_core *core_new(struct pw_context *context,
		struct pw_properties *properties, size_t user_data_size)
{
	struct pw_core *p;
	struct pw_protocol *protocol;
	const char *protocol_name;
	int res;

	p = calloc(1, sizeof(struct pw_core) + user_data_size);
	if (p == NULL) {
		res = -errno;
		goto exit_cleanup;
	}
	pw_log_debug("%p: new", p);

	if (properties == NULL)
		properties = pw_properties_new(NULL, NULL);
	if (properties == NULL)
		goto error_properties;

	pw_properties_add(properties, &context->properties->dict);

	p->proxy.core = p;
	p->context = context;
	p->properties = properties;
	p->pool = pw_mempool_new(NULL);
	p->core = p;
	if (user_data_size > 0)
		p->user_data = SPA_PTROFF(p, sizeof(struct pw_core), void);
	p->proxy.user_data = p->user_data;

	pw_map_init(&p->objects, 64, 32);
	spa_list_init(&p->stream_list);
	spa_list_init(&p->filter_list);

	if ((protocol_name = pw_properties_get(properties, PW_KEY_PROTOCOL)) == NULL &&
	    (protocol_name = pw_properties_get(context->properties, PW_KEY_PROTOCOL)) == NULL)
		protocol_name = PW_TYPE_INFO_PROTOCOL_Native;

	protocol = pw_context_find_protocol(context, protocol_name);
	if (protocol == NULL) {
		res = -ENOTSUP;
		goto error_protocol;
	}

	p->conn = pw_protocol_new_client(protocol, p, &properties->dict);
	if (p->conn == NULL)
		goto error_connection;

	if ((res = pw_proxy_init(&p->proxy, PW_TYPE_INTERFACE_Core, PW_VERSION_CORE)) < 0)
		goto error_proxy;

	p->client = (struct pw_client*)pw_proxy_new(&p->proxy,
			PW_TYPE_INTERFACE_Client, PW_VERSION_CLIENT, 0);
	if (p->client == NULL) {
		res = -errno;
		goto error_proxy;
	}

	pw_core_add_listener(p, &p->core_listener, &core_events, p);
	pw_proxy_add_listener(&p->proxy, &p->proxy_core_listener, &proxy_core_events, p);

	pw_core_hello(p, PW_VERSION_CORE);
	pw_client_update_properties(p->client, &p->properties->dict);

	spa_list_append(&context->core_list, &p->link);

	return p;

error_properties:
	res = -errno;
	pw_log_error("%p: can't create properties: %m", p);
	goto exit_free;
error_protocol:
	pw_log_error("%p: can't find protocol '%s': %s", p, protocol_name, spa_strerror(res));
	goto exit_free;
error_connection:
	res = -errno;
	pw_log_error("%p: can't create new native protocol connection: %m", p);
	goto exit_free;
error_proxy:
	pw_log_error("%p: can't initialize proxy: %s", p, spa_strerror(res));
	goto exit_free;

exit_free:
	free(p);
exit_cleanup:
	pw_properties_free(properties);
	errno = -res;
	return NULL;
}

SPA_EXPORT
struct pw_core *
pw_context_connect(struct pw_context *context, struct pw_properties *properties,
	      size_t user_data_size)
{
	struct pw_core *core;
	int res;

	core = core_new(context, properties, user_data_size);
	if (core == NULL)
		return NULL;

	pw_log_debug("%p: connect", core);

	if ((res = pw_protocol_client_connect(core->conn,
					&core->properties->dict,
					NULL, NULL)) < 0)
		goto error_free;

	return core;

error_free:
	pw_core_disconnect(core);
	errno = -res;
	return NULL;
}
```

`pw_context_connect()` 函数调用 `core_new()` 新建 `pw_core` 对象 (其中包含所选择的 IPC 协议的 `pw_protocol_client`)，并执行其 `pw_protocol_client` 的 `connect()` 方法。

`core_new()` 函数的执行过程如下：
1. 为 `pw_core` 对象分配内存。
2. 为 `pw_core` 对象构造属性。`core_new()` 函数的 `properties` 参数来自 `pw_context_connect()` 函数的 `properties` 参数。当 `core_new()` 函数接收的 `properties` 参数为空时，会为 `pw_core` 对象新建 `pw_properties`，否则，使用传入的 `properties` 参数。之后 `pw_context` 的属性被添加到 `pw_core` 对象的属性集合中。`pw_context` 的属性从 pipewire 进程的配置文件中获得，`pw_context_connect()` 函数的 `properties` 参数用来传一些调用方期望配置的属性，可以为空。`pw_core` 对象的属性为调用方传入的定制化属性和配置文件中的属性的并集，但定制化属性的优先级更高。
3. 初始化 `pw_core` 对象的一些字段。
4. 从 `pw_core` 对象的属性中获得协议名，如果失败，默认使用 `PW_TYPE_INFO_PROTOCOL_Native`，即 **PipeWire:Protocol:Native**，随后根据协议名从 `pw_context` 获得协议，即 `pw_protocol` 对象。
5. 通过协议 `pw_protocol` 新建协议客户端，获得 `pw_protocol_client` 对象。`pw_protocol_new_client()` 是个宏，它封装了对 `pw_protocol` 的协议实现字段 `pw_protocol_implementation` 的 `new_client()` 回调的调用。
6. 调用 `pw_proxy_init()` 函数初始化 `pw_core` 的代理对象，即用于 Core 的代理。
7. 调用 `pw_proxy_new()` 函数新建 **PipeWire:Interface:Client** 接口的代理，用 `struct pw_client` 对象表示。
8. 为 `pw_core` 对象添加 core 监听者和 proxy 的监听者。
9. 调用 `pw_core_hello()` 向 pipewire 守护进程发送握手消息。
10. 更新 **PipeWire:Interface:Client** 接口的代理的属性。
11. 将 `pw_core` 对象挂在 `pw_context` 的 core 对象列表中。

对于 **module-protocol-native** 模块实现的原生 PipeWire IPC 协议，协议实现字段 `pw_protocol_implementation` 的 `new_client()` 回调为 `impl_new_client()`，这个函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
struct client {
	struct pw_protocol_client this;
	struct pw_context *context;

	struct spa_source *source;

	struct pw_protocol_native_connection *connection;
	struct spa_hook conn_listener;

	int ref;

	struct footer_core_global_state footer_state;

	unsigned int connected:1;
	unsigned int disconnecting:1;
	unsigned int need_flush:1;
	unsigned int paused:1;
};
 . . . . . .
static struct pw_protocol_client *
impl_new_client(struct pw_protocol *protocol,
		struct pw_core *core,
		const struct spa_dict *props)
{
	struct client *impl;
	struct pw_protocol_client *this;
	const char *str = NULL;
	int res;

	if ((impl = calloc(1, sizeof(struct client))) == NULL)
		return NULL;

	pw_log_debug("%p: new client %p", protocol, impl);

	this = &impl->this;
	this->protocol = protocol;
	this->core = core;

	impl->ref = 1;
	impl->context = protocol->context;
	impl->connection = pw_protocol_native_connection_new(protocol->context, -1);
	if (impl->connection == NULL) {
		res = -errno;
		goto error_free;
	}

	if (props) {
		str = spa_dict_lookup(props, PW_KEY_REMOTE_INTENTION);
		if (str == NULL &&
		   (str = spa_dict_lookup(props, PW_KEY_REMOTE_NAME)) != NULL &&
		    spa_streq(str, "internal"))
			str = "internal";
	}
	if (str == NULL)
		str = "generic";

	pw_log_debug("%p: connect %s", protocol, str);

	if (spa_streq(str, "screencast"))
		this->connect = pw_protocol_native_connect_portal_screencast;
	else if (spa_streq(str, "internal"))
		this->connect = pw_protocol_native_connect_internal;
	else
		this->connect = pw_protocol_native_connect_local_socket;

	this->steal_fd = impl_steal_fd;
	this->connect_fd = impl_connect_fd;
	this->disconnect = impl_disconnect;
	this->destroy = impl_destroy;
	this->set_paused = impl_set_paused;

	spa_list_append(&protocol->client_list, &this->link);

	return this;

error_free:
	free(impl);
	errno = -res;
	return NULL;
}
```

**module-protocol-native** 模块用 `struct client` 类型描述 IPC 协议客户端侧的客户端，它是 `pw_protocol_client` 类型的子类。在 `impl_new_client()` 函数中，做的主要的事情是创建对象，即创建 `struct client`/`struct pw_protocol_client` 对象，包括为对象分配内存空间，初始化对象的各个字段，特别是 `connect`、`connect_fd`、`disconnect` 和 `destroy` 这些对象上的关键操作，对于 `connect` 操作，根据传入的 **remote.intention** 或 **remote.name** 参数选择不同的函数，并将对象挂在 `pw_protocol` 的客户端列表中。这里还没有执行建立连接这样的操作。

`core_new()` 函数中调用的 `pw_proxy_init()` 和 `pw_proxy_new()`，执行代理的初始化或创建，这两个函数定义 (位于 *pipewire/src/pipewire/proxy.c*) 如下：
```
struct proxy {
	struct pw_proxy this;
};
 . . . . . .
int pw_proxy_init(struct pw_proxy *proxy, const char *type, uint32_t version)
{
	int res;

	proxy->refcount = 1;
	proxy->type = type;
	proxy->version = version;
	proxy->bound_id = SPA_ID_INVALID;

	proxy->id = pw_map_insert_new(&proxy->core->objects, proxy);
	if (proxy->id == SPA_ID_INVALID) {
		res = -errno;
		pw_log_error("%p: can't allocate new id: %m", proxy);
		goto error;
	}

	spa_hook_list_init(&proxy->listener_list);
	spa_hook_list_init(&proxy->object_listener_list);

	if ((res = pw_proxy_install_marshal(proxy, false)) < 0) {
		pw_log_error("%p: no marshal for type %s/%d: %s", proxy,
				type, version, spa_strerror(res));
		goto error_clean;
	}
	proxy->in_map = true;
	return 0;

error_clean:
	pw_map_remove(&proxy->core->objects, proxy->id);
error:
	return res;
}

/** Create a proxy object with a given id and type
 *
 * \param factory another proxy object that serves as a factory
 * \param type Type of the proxy object
 * \param version Interface version
 * \param user_data_size size of user_data
 * \return A newly allocated proxy object or NULL on failure
 *
 * This function creates a new proxy object with the supplied id and type. The
 * proxy object will have an id assigned from the client id space.
 *
 * \sa pw_core
 */
SPA_EXPORT
struct pw_proxy *pw_proxy_new(struct pw_proxy *factory,
			      const char *type, uint32_t version,
			      size_t user_data_size)
{
	struct proxy *impl;
	struct pw_proxy *this;
	int res;

	impl = calloc(1, sizeof(struct proxy) + user_data_size);
	if (impl == NULL)
		return NULL;

	this = &impl->this;
	this->core = factory->core;

	if ((res = pw_proxy_init(this, type, version)) < 0)
		goto error_init;

	if (user_data_size > 0)
		this->user_data = SPA_PTROFF(impl, sizeof(struct proxy), void);

	pw_log_debug("%p: new %u type %s/%d core-proxy:%p, marshal:%p",
			this, this->id, type, version, this->core, this->marshal);
	return this;

error_init:
	free(impl);
	errno = -res;
	return NULL;
}

SPA_EXPORT
int pw_proxy_install_marshal(struct pw_proxy *this, bool implementor)
{
	struct pw_core *core = this->core;
	const struct pw_protocol_marshal *marshal;

	if (core == NULL)
		return -EIO;

	marshal = pw_protocol_get_marshal(core->conn->protocol,
			this->type, this->version,
			implementor ? PW_PROTOCOL_MARSHAL_FLAG_IMPL : 0);
	if (marshal == NULL)
		return -EPROTO;

	this->marshal = marshal;
	this->type = marshal->type;

	this->impl = SPA_INTERFACE_INIT(
			this->type,
			this->marshal->version,
			this->marshal->client_marshal, this);
	return 0;
}
```

`core_new()` 函数调用 `pw_proxy_init()` 初始化 Core 的代理，具体过程如下：
1. 初始化 Core 的代理 `pw_proxy` 对象的引用计数，类型等字段。
2. 将 `pw_proxy` 对象插入 `pw_core` 的客户端侧以 proxy 对象 ID 索引的 proxy 对象映射中，获得代理对象 ID。
3. 初始化 `pw_proxy` 对象的监听器字段。
4. 调用 `pw_proxy_install_marshal()` 函数为 `pw_proxy` 对象安装所代理的接口的 marshaller。具体而言，根据接口类型名从 IPC 协议 `pw_protocol` 获得 marshaller `pw_protocol_marshal` 对象，`pw_protocol_marshal` 有用于不同方向 IPC 消息传递，客户端和服务器端共 4 个的方法集合，使用 `pw_protocol_marshal` 的 `client_marshal` 方法集合初始化代理的接口实现字段。这个操作赋予代理对象 `pw_proxy` 以灵魂。
5. 将 `pw_proxy` 对象的 `in_map` 字段置为 `true` 以表示 `pw_proxy` 对象已经插入 `pw_core` 的 proxy 对象映射中，并返回。

`pw_proxy_new()` 函数，相对于 `pw_proxy_init()`，除了会为代理对象分配内存外，并没有多做太多事情。

`core_new()` 函数调用 `pw_proxy_new()` 为 **PipeWire:Interface:Client** 接口新建代理 `pw_proxy` 并初始化。

`core_new()` 函数的执行过程中，会调用 `pw_core_hello()` 执行 core 的握手动作。`pw_core_hello()` 是个宏，是对所代理的 Core 接口的封装，也就是 **module-protocol-native** 模块中，**PipeWire:Interface:Core** 接口的 marshaller 的 `client_marshal` 方法集合中的 `hello` 方法 `core_method_marshal_hello()`，这个函数定义 (位于 *pipewire/src/modules/module-protocol-native/protocol-native.c*) 如下：
```
static int core_method_marshal_hello(void *object, uint32_t version)
{
	struct pw_proxy *proxy = object;
	struct spa_pod_builder *b;

	b = pw_protocol_native_begin_proxy(proxy, PW_CORE_METHOD_HELLO, NULL);

	spa_pod_builder_add_struct(b,
			SPA_POD_Int(version));

	return pw_protocol_native_end_proxy(proxy, b);
}
```

marshaller 的这个方法，调用 `pw_protocol_native_begin_proxy()` 构造一个可以构建 POD 对象的上下文，获得 `spa_pod_builder`，之后在 POD builder 中加入各个字段，并调用 `pw_protocol_native_end_proxy()` 结束 POD 对象的构建。`pw_protocol_native_begin_proxy()` 和 `pw_protocol_native_end_proxy()` 是宏，是
对 `pw_protocol` 的 `extension` API 方法的封装， 这两个宏最终调用的函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
static struct spa_pod_builder *
impl_ext_begin_proxy(struct pw_proxy *proxy, uint8_t opcode, struct pw_protocol_native_message **msg)
{
	struct client *impl = SPA_CONTAINER_OF(proxy->core->conn, struct client, this);
	return pw_protocol_native_connection_begin(impl->connection, proxy->id, opcode, msg);
}
 . . . . . .
static int impl_ext_end_proxy(struct pw_proxy *proxy,
			       struct spa_pod_builder *builder)
{
	struct pw_core *core = proxy->core;
	struct client *impl = SPA_CONTAINER_OF(core->conn, struct client, this);
	assert_single_pod(builder);
	marshal_core_footers(&impl->footer_state, core, builder);
	return core->send_seq = pw_protocol_native_connection_end(impl->connection, builder);
}
```

`pw_protocol` 的 `extension` API 方法最终又通过 `pw_protocol_native_connection` 的方法实现。

`pw_context_connect()` 函数里，`core_new()` 执行的整个过程中，pipewire 进程和 pipewire 守护进程之间的原生 IPC 协议连接都还没有真正建立，因而 `core_new()` 调用 `pw_core_hello()` 也不会真正将消息发送出去。原生 IPC 协议连接真正建立，是在 `pw_protocol_client_connect()` 被调用的时候。在 pipewire 原生应用程序 API 实现这一层，`pw_protocol_client` 用来表示客户端与 pipewire 守护进程之间的连接，在更底层则有不同的表示。

`pw_protocol_client_connect()` 是个宏，是对 `pw_protocol_client` 的 `connect` 方法的包装。`pw_protocol_client` 的 `connect` 方法是在前面 `core_new()` 中调用 `pw_protocol_new_client()` 新建 `pw_protocol_client` 时初始化的。针对不同的场景，`connect` 方法的可能实现有多个：

 * `pw_protocol_native_connect_portal_screencast()`：似乎用于屏幕录制的特殊原生 IPC 协议连接通道，但没有实现。
 * `pw_protocol_native_connect_internal()`：用于进程内原生 IPC 协议通信连接通道的建立。
 * `pw_protocol_native_connect_local_socket()`：用于其它进程和 pipewire 守护进程之间原生 IPC 协议通信连接通道的建立。

`pw_protocol_native_connect_internal()` 函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
struct client_data {
	struct pw_impl_client *client;
	struct spa_hook client_listener;

	struct spa_list protocol_link;
	struct server *server;

	struct spa_source *source;
	struct pw_protocol_native_connection *connection;
	struct spa_hook conn_listener;

	struct footer_client_global_state footer_state;

	unsigned int busy:1;
	unsigned int need_flush:1;

	struct protocol_compat_v2 compat_v2;
};
 . . . . . .
static void on_server_connection_destroy(void *data)
{
	struct client_data *this = data;
	spa_hook_remove(&this->conn_listener);
}

static void on_start(void *data, uint32_t version)
{
	struct client_data *this = data;
	struct pw_impl_client *client = this->client;

	pw_log_debug("version %d", version);

	if (client->core_resource != NULL)
		pw_resource_remove(client->core_resource);

	if (pw_global_bind(pw_impl_core_get_global(client->core), client,
			PW_PERM_ALL, version, 0) < 0)
		return;

	if (version == 0)
		client->compat_v2 = &this->compat_v2;

	return;
}

static void on_server_need_flush(void *data)
{
	struct client_data *this = data;
	struct pw_impl_client *client = this->client;

	pw_log_trace("need flush");
	this->need_flush = true;

	if (this->source && !(this->source->mask & SPA_IO_OUT)) {
		pw_loop_update_io(client->context->main_loop,
				this->source, this->source->mask | SPA_IO_OUT);
	}
}

static const struct pw_protocol_native_connection_events server_conn_events = {
	PW_VERSION_PROTOCOL_NATIVE_CONNECTION_EVENTS,
	.destroy = on_server_connection_destroy,
	.start = on_start,
	.need_flush = on_server_need_flush,
};

static bool check_print(const uint8_t *buffer, int len)
{
	int i;
	while (len > 1 && buffer[len-1] == 0)
		len--;
	for (i = 0; i < len; i++)
		if (!isprint(buffer[i]))
			return false;
	return true;
}

static struct client_data *client_new(struct server *s, int fd)
{
	struct client_data *this;
	struct pw_impl_client *client;
	struct pw_protocol *protocol = s->this.protocol;
	socklen_t len;
#if defined(__FreeBSD__)
	struct xucred xucred;
#else
	struct ucred ucred;
#endif
	struct pw_context *context = protocol->context;
	struct pw_properties *props;
	uint8_t buffer[1024];
	struct protocol_data *d = pw_protocol_get_user_data(protocol);
	int i, res;

	props = pw_properties_new(PW_KEY_PROTOCOL, "protocol-native", NULL);
	if (props == NULL)
		goto exit;

#if defined(__linux__)
	len = sizeof(ucred);
	if (getsockopt(fd, SOL_SOCKET, SO_PEERCRED, &ucred, &len) < 0) {
		pw_log_warn("server %p: no peercred: %m", s);
	} else {
		pw_properties_setf(props, PW_KEY_SEC_PID, "%d", ucred.pid);
		pw_properties_setf(props, PW_KEY_SEC_UID, "%d", ucred.uid);
		pw_properties_setf(props, PW_KEY_SEC_GID, "%d", ucred.gid);
	}

	len = sizeof(buffer);
	if (getsockopt(fd, SOL_SOCKET, SO_PEERSEC, buffer, &len) < 0) {
		if (errno == ENOPROTOOPT)
			pw_log_info("server %p: security label not available", s);
		else
			pw_log_warn("server %p: security label error: %m", s);
	} else {
		if (!check_print(buffer, len)) {
			char *hex, *p;
			static const char *ch = "0123456789abcdef";

			p = hex = alloca(len * 2 + 10);
			p += snprintf(p, 5, "hex:");
			for(i = 0; i < (int)len; i++)
				p += snprintf(p, 3, "%c%c",
						ch[buffer[i] >> 4], ch[buffer[i] & 0xf]);
			pw_properties_set(props, PW_KEY_SEC_LABEL, hex);

		} else {
			/* buffer is not null terminated, must use length explicitly */
			pw_properties_setf(props, PW_KEY_SEC_LABEL, "%.*s",
					(int)len, buffer);
		}
	}
#elif defined(__FreeBSD__)
	len = sizeof(xucred);
	if (getsockopt(fd, 0, LOCAL_PEERCRED, &xucred, &len) < 0) {
		pw_log_warn("server %p: no peercred: %m", s);
	} else {
#if __FreeBSD__ >= 13
		pw_properties_setf(props, PW_KEY_SEC_PID, "%d", xucred.cr_pid);
#endif
		pw_properties_setf(props, PW_KEY_SEC_UID, "%d", xucred.cr_uid);
		pw_properties_setf(props, PW_KEY_SEC_GID, "%d", xucred.cr_gid);
		// this is what Linuxulator does at the moment, see sys/compat/linux/linux_socket.c
		pw_properties_set(props, PW_KEY_SEC_LABEL, "unconfined");
	}
#endif

	pw_properties_setf(props, PW_KEY_MODULE_ID, "%d", d->module->global->id);

	client = pw_context_create_client(s->this.core,
			protocol, props, sizeof(struct client_data));
	if (client == NULL)
		goto exit;


	this = pw_impl_client_get_user_data(client);
	spa_list_append(&s->this.client_list, &this->protocol_link);

	this->server = s;
	this->client = client;
	this->source = pw_loop_add_io(pw_context_get_main_loop(context),
				      fd, SPA_IO_ERR | SPA_IO_HUP, true,
				      connection_data, this);
	if (this->source == NULL) {
		res = -errno;
		goto cleanup_client;
	}

	this->connection = pw_protocol_native_connection_new(protocol->context, fd);
	if (this->connection == NULL) {
		res = -errno;
		goto cleanup_client;
	}

	pw_map_init(&this->compat_v2.types, 0, 32);

	pw_protocol_native_connection_add_listener(this->connection,
						   &this->conn_listener,
						   &server_conn_events,
						   this);

	pw_impl_client_add_listener(client, &this->client_listener, &client_events, this);

	if ((res = pw_impl_client_register(client, NULL)) < 0)
		goto cleanup_client;

	if (!client->busy)
		pw_loop_update_io(pw_context_get_main_loop(context),
				this->source, this->source->mask | SPA_IO_IN);

	return this;

cleanup_client:
	pw_impl_client_destroy(client);
	errno = -res;
exit:
	return NULL;
}
 . . . . . .
static int impl_connect_fd(struct pw_protocol_client *client, int fd, bool do_close)
{
	struct client *impl = SPA_CONTAINER_OF(client, struct client, this);
	int res;

	impl->connected = false;
	impl->disconnecting = false;

	pw_protocol_native_connection_set_fd(impl->connection, fd);
	impl->source = pw_loop_add_io(impl->context->main_loop,
					fd,
					SPA_IO_IN | SPA_IO_OUT | SPA_IO_HUP | SPA_IO_ERR,
					do_close, on_remote_data, impl);
	if (impl->source == NULL) {
		res = -errno;
		goto error_cleanup;
	}

	pw_protocol_native_connection_add_listener(impl->connection,
						   &impl->conn_listener,
						   &client_conn_events,
						   impl);
	return 0;

error_cleanup:
	if (impl->connection) {
		pw_protocol_native_connection_destroy(impl->connection);
		impl->connection = NULL;
	}
	return res;
}
 . . . . . .
static int pw_protocol_native_connect_internal(struct pw_protocol_client *client,
					    const struct spa_dict *props,
					    void (*done_callback) (void *data, int res),
					    void *data)
{
	int res, sv[2];
	struct pw_protocol *protocol = client->protocol;
	struct protocol_data *d = pw_protocol_get_user_data(protocol);
	struct server *s = d->local;
	struct pw_permission permissions[1];
	struct client_data *c;

	pw_log_debug("server %p: internal connect", s);

	if (socketpair(PF_LOCAL, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0, sv) < 0) {
		res = -errno;
		pw_log_error("server %p: socketpair() failed with error: %m", s);
		goto error;
	}

	c = client_new(s, sv[0]);
	if (c == NULL) {
		res = -errno;
		pw_log_error("server %p: failed to create client: %m", s);
		goto error_close;
	}
	permissions[0] = PW_PERMISSION_INIT(PW_ID_ANY, PW_PERM_ALL);
	pw_impl_client_update_permissions(c->client, 1, permissions);

	res = pw_protocol_client_connect_fd(client, sv[1], true);
done:
	if (done_callback)
		done_callback(data, res);
	return res;

error_close:
	close(sv[0]);
	close(sv[1]);
error:
	goto done;
}
```

进程内原生 IPC 协议通信的基本思路是，用 `socketpair()` 创建一个 socket 对，一个作为服务器端 socket，用于服务器端的数据收发，另一个作为客户端 socket，用于客户端的数据收发。`pw_protocol_native_connect_internal()` 函数的执行过程如下：
1. 调用 `socketpair()` 创建一个 socket 对。
2. 将 socket 对的 socket 0 用作服务器端 socket，调用 `client_new()` 创建服务器端的原生 IPC 协议通信的客户端表示 `struct client_data`/`struct pw_impl_client`。
3. 更新 `struct client_data`/`struct pw_impl_client` 的权限。
4. 将 socket 对的 socket 1 用作客户端 socket，调用 `pw_protocol_client_connect_fd()` 将 socket 对的 socket 1 绑定到 `pw_protocol_client`。

`pw_protocol_client_connect_fd()` 是个宏，是对 `pw_protocol_client` 的 `connect_fd()` 方法的包装，这个方法的具体实现在 `impl_new_client()` 中新建 `pw_protocol_client` 对象时确定，具体为 `impl_connect_fd()`。`impl_connect_fd()` 函数的执行过程如下：
1. 设置 `struct client` 对象的连接状态。
2. 根据传入的文件描述符，为 `pw_protocol_client` 的 `pw_protocol_native_connection` 设置文件描述符。
3. 在主事件循环中，为客户端文件描述符添加 IO 事件源，IO 事件处理函数为 `on_remote_data()`。
4. 向 `pw_protocol_native_connection` 注册监听器。

`client_new()` 创建服务器端的原生 IPC 协议通信的客户端表示，具体过程如下：
1. 为原生 IPC 协议通信的客户端表示新建属性。
2. 通过 socket 的 fd 获得对端的 `ucred`，并将这些信息放进属性中。
3. 通过 socket 的 fd 获得对端的安全标签，并将这些信息放进属性中。
4. 在属性中设置实现协议的模块 ID。
5. 调用 `pw_context_create_client()` 创建 `struct client_data`/`struct pw_impl_client` 对象。
6. 将 `struct client_data`/`struct pw_impl_client` 对象挂进 `pw_protocol_server` 的客户端列表中。
7. 在主事件循环中，为服务器端文件描述符添加 IO 事件源，IO 事件处理函数为 `connection_data()`。
8. 为客户端创建 `pw_protocol_native_connection`。
9. 向 `pw_protocol_native_connection` 注册监听器。
10. 向 `pw_impl_client` 注册监听器。
11. 注册 `pw_impl_client`，这会把 `pw_impl_client` 挂进 `pw_context` 的客户端列表中。
12. 根据需要，更新服务器端文件描述符 IO 事件源。

`pw_protocol_native_connect_local_socket()` 函数定义 (位于 *pipewire/src/modules/module-protocol-native/local-socket.c*) 如下：
```
#define DEFAULT_SYSTEM_RUNTIME_DIR "/run/pipewire"

PW_LOG_TOPIC_EXTERN(mod_topic);
#define PW_LOG_TOPIC_DEFAULT mod_topic

static const char *
get_remote(const struct spa_dict *props)
{
	const char *name = NULL;

	if (props)
		name = spa_dict_lookup(props, PW_KEY_REMOTE_NAME);
	if (name == NULL || name[0] == '\0')
		name = getenv("PIPEWIRE_REMOTE");
	if (name == NULL || name[0] == '\0')
		name = PW_DEFAULT_REMOTE;
	return name;
}

static const char *
get_runtime_dir(void)
{
	const char *runtime_dir;

	runtime_dir = getenv("PIPEWIRE_RUNTIME_DIR");
	if (runtime_dir == NULL)
		runtime_dir = getenv("XDG_RUNTIME_DIR");
	if (runtime_dir == NULL)
		runtime_dir = getenv("USERPROFILE");
	return runtime_dir;
}

static const char *
get_system_dir(void)
{
	return DEFAULT_SYSTEM_RUNTIME_DIR;
}

static int try_connect(struct pw_protocol_client *client,
		const char *runtime_dir, const char *name,
		void (*done_callback) (void *data, int res),
		void *data)
{
	struct sockaddr_un addr;
	socklen_t size;
	int res, name_size, fd;

	pw_log_info("connecting to '%s' runtime_dir:%s", name, runtime_dir);

	if ((fd = socket(PF_LOCAL, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0)) < 0) {
		res = -errno;
		goto error;
	}

	memset(&addr, 0, sizeof(addr));
	addr.sun_family = AF_LOCAL;
	if (runtime_dir == NULL)
		name_size = snprintf(addr.sun_path, sizeof(addr.sun_path), "%s", name) + 1;
	else
		name_size = snprintf(addr.sun_path, sizeof(addr.sun_path), "%s/%s", runtime_dir, name) + 1;

	if (name_size > (int) sizeof addr.sun_path) {
		if (runtime_dir == NULL)
			pw_log_error("client %p: socket path \"%s\" plus null terminator exceeds %i bytes",
				client, name, (int) sizeof(addr.sun_path));
		else
			pw_log_error("client %p: socket path \"%s/%s\" plus null terminator exceeds %i bytes",
				client, runtime_dir, name, (int) sizeof(addr.sun_path));
		res = -ENAMETOOLONG;
		goto error_close;
	};

	size = offsetof(struct sockaddr_un, sun_path) + name_size;

	if (connect(fd, (struct sockaddr *) &addr, size) < 0) {
		pw_log_debug("connect to '%s' failed: %m", name);
		if (errno == ENOENT)
			errno = EHOSTDOWN;
		if (errno == EAGAIN) {
			pw_log_info("client %p: connect pending, fd %d", client, fd);
		} else {
			res = -errno;
			goto error_close;
		}
	}

	res = pw_protocol_client_connect_fd(client, fd, true);

	if (done_callback)
		done_callback(data, res);

	return res;

error_close:
	close(fd);
error:
	return res;
}

int pw_protocol_native_connect_local_socket(struct pw_protocol_client *client,
					    const struct spa_dict *props,
					    void (*done_callback) (void *data, int res),
					    void *data)
{
	const char *runtime_dir, *name;
	int res;

	name = get_remote(props);
	if (name == NULL)
		return -EINVAL;

	if (name[0] == '/') {
		res = try_connect(client, NULL, name, done_callback, data);
	} else {
		runtime_dir = get_runtime_dir();
		if (runtime_dir != NULL) {
			res = try_connect(client, runtime_dir, name, done_callback, data);
			if (res >= 0)
				goto exit;
		}
		runtime_dir = get_system_dir();
		if (runtime_dir != NULL)
			res = try_connect(client, runtime_dir, name, done_callback, data);
	}
exit:
	return res;
}
```

`pw_protocol_native_connect_local_socket()` 函数的整体执行过程，与 `pw_protocol_native_connect_internal()` 函数的类似，都是先获得连接到 IPC 协议服务器的 socket 文件描述符，然后调用 `pw_protocol_client_connect_fd()` 将 socket 文件描述符绑定到 `pw_protocol_client`，但具体过程略有区别。`pw_protocol_native_connect_local_socket()` 函数的详细执行过程如下：
1. 获得 unix domain socket 名称。先尝试从传进来的 `remote.name` 配置项或 **PIPEWIRE_REMOTE** 环境变量获取，失败时默认使用 `pipewire-0`。
2. 获得运行时目录路径，这分为 3 种情况：
(1). unix domain socket 名称即为 unix domain socket 的完整路径，运行时目录路径取空值。
(2). 运行时目录路径从环境变量 **PIPEWIRE_RUNTIME_DIR**、**XDG_RUNTIME_DIR** 或 **USERPROFILE** 获取。
(3). 从环境变量获得的运行时目录路径不可用，用系统目录路径，为 `DEFAULT_SYSTEM_RUNTIME_DIR`，即 */run/pipewire*。
3. 调用 `try_connect()` 函数建立与原生 IPC 协议服务器端的连接。具体过程如下：
(1). 新建 unix domain socket。
(2). 获得 unix domain socket 的完整路径。
(3). 连接 unix domain socket 的完整路径。
(4). 调用 `pw_protocol_client_connect_fd()` 将 socket 文件描述符绑定到 `pw_protocol_client`。
(5). 传入的 `done_callback` 非空时调用它。

至此，原生 PipeWire IPC 协议客户端侧的准备工作结束。

## 原生 PipeWire IPC 协议服务器接受连接

原生 PipeWire IPC 协议服务器端的协议初始化过程中，调用 `impl_add_server()` 函数起 socket 服务器，并为服务器端 socket 在主事件循环中添加 IO 事件源，socket 上的输入 IO 事件处理函数为 `socket_data()`，这个函数定义 (位于 *pipewire/src/modules/module-protocol-native.c*) 如下：
```
static void
socket_data(void *data, int fd, uint32_t mask)
{
	struct server *s = data;
	struct client_data *client;
	struct sockaddr_un name;
	socklen_t length;
	int client_fd;

	length = sizeof(name);
	client_fd = accept4(fd, (struct sockaddr *) &name, &length, SOCK_CLOEXEC);
	if (client_fd < 0) {
		pw_log_error("server %p: failed to accept: %m", s);
		return;
	}

	client = client_new(s, client_fd);
	if (client == NULL) {
		pw_log_error("server %p: failed to create client", s);
		close(client_fd);
		return;
	}
}
```

这个函数的执行过程包括，调用 `accept4()` 获得客户端连接的 socket 文件描述符，调用 `client_new()` 创建服务器端的原生 IPC 协议通信的客户端表示 `struct client_data`/`struct pw_impl_client`。`client_new()` 函数的具体执行过程，前面已有说明。

至此，原生 PipeWire IPC 协议客户端和服务器端进行通信的准备工作结束，双方可以进行消息通信了。






















