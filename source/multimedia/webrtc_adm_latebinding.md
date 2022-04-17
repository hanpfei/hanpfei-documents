---
title: WebRTC Linux ADM 实现中的符号延迟加载机制
date: 2020-08-17 21:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

ADM（AudioDeviceModule）在 WebRTC 中主要用于音频数据的录制采集和音频数据的播放，这里是 WebRTC 的实时音视频系统与系统的音频硬件衔接的地方。WebRTC 为 Linux 平台实现了 ALSA 和 Pulse 等类型的 ADM `AudioDeviceLinuxALSA` 和 `AudioDeviceLinuxPulse`，它们分别基于 Linux 系统提供的库 `libasound` 和 `libpulse` 实现。
<!--more-->
WebRTC 为做到 Linux ADM 实现的高度灵活性，实现了一套符号的延迟加载机制，即在 WebRTC 编译的时候，无需链接 `libasound` 和 `libpulse` 库，在创建需要的 ADM 对象时，才通过动态链接加载机制加载需要的库及其中的符号。

这套机制在底层基于 Linux 平台的 libdl 提供的 `dlopen()`、`dlclose()`、`dlsym()` 和 `dlerror()` 等接口实现动态链接库的动态加载，及符号的查找。更上一层，WebRTC 为每种 ADM 实现基于 libdl 提供的接口，封装一个用于符号延迟加载及访问的 wrapper 类。在 ADM 的具体实现中，借助于一个宏，以类似于普通的符号访问的方式，访问延迟加载的符号。

在 `webrtc/modules/audio_device/linux/latebindingsymboltable_linux.h` 中的 libdl 接口的 wrapper 类 `LateBindingSymbolTable` 声明如下：
```
#ifdef WEBRTC_LINUX
typedef void* DllHandle;

const DllHandle kInvalidDllHandle = NULL;
#else
#error Not implemented
#endif

// These are helpers for use only by the class below.
DllHandle InternalLoadDll(const char dll_name[]);

void InternalUnloadDll(DllHandle handle);

bool InternalLoadSymbols(DllHandle handle,
                         int num_symbols,
                         const char* const symbol_names[],
                         void* symbols[]);

template <int SYMBOL_TABLE_SIZE,
          const char kDllName[],
          const char* const kSymbolNames[]>
class LateBindingSymbolTable {
 public:
  LateBindingSymbolTable()
      : handle_(kInvalidDllHandle), undefined_symbols_(false) {
    memset(symbols_, 0, sizeof(symbols_));
  }

  ~LateBindingSymbolTable() { Unload(); }

  static int NumSymbols() { return SYMBOL_TABLE_SIZE; }

  // We do not use this, but we offer it for theoretical convenience.
  static const char* GetSymbolName(int index) {
    assert(index < NumSymbols());
    return kSymbolNames[index];
  }

  bool IsLoaded() const { return handle_ != kInvalidDllHandle; }

  // Loads the DLL and the symbol table. Returns true iff the DLL and symbol
  // table loaded successfully.
  bool Load() {
    if (IsLoaded()) {
      return true;
    }
    if (undefined_symbols_) {
      // We do not attempt to load again because repeated attempts are not
      // likely to succeed and DLL loading is costly.
      return false;
    }
    handle_ = InternalLoadDll(kDllName);
    if (!IsLoaded()) {
      return false;
    }
    if (!InternalLoadSymbols(handle_, NumSymbols(), kSymbolNames, symbols_)) {
      undefined_symbols_ = true;
      Unload();
      return false;
    }
    return true;
  }

  void Unload() {
    if (!IsLoaded()) {
      return;
    }
    InternalUnloadDll(handle_);
    handle_ = kInvalidDllHandle;
    memset(symbols_, 0, sizeof(symbols_));
  }

  // Retrieves the given symbol. NOTE: Recommended to use LATESYM_GET below
  // instead of this.
  void* GetSymbol(int index) const {
    assert(IsLoaded());
    assert(index < NumSymbols());
    return symbols_[index];
  }

 private:
  DllHandle handle_;
  bool undefined_symbols_;
  void* symbols_[SYMBOL_TABLE_SIZE];

  RTC_DISALLOW_COPY_AND_ASSIGN(LateBindingSymbolTable);
};
```

`LateBindingSymbolTable` 是一个模板类，其类定义中调用的几个用于动态链接库加载和符号查找的函数在 `webrtc/modules/audio_device/linux/latebindingsymboltable_linux.cc` 文件中的定义如下：
```
inline static const char* GetDllError() {
#ifdef WEBRTC_LINUX
  char* err = dlerror();
  if (err) {
    return err;
  } else {
    return "No error";
  }
#else
#error Not implemented
#endif
}

DllHandle InternalLoadDll(const char dll_name[]) {
#ifdef WEBRTC_LINUX
  DllHandle handle = dlopen(dll_name, RTLD_NOW);
#else
#error Not implemented
#endif
  if (handle == kInvalidDllHandle) {
    RTC_LOG(LS_WARNING) << "Can't load " << dll_name << " : " << GetDllError();
  }
  return handle;
}

void InternalUnloadDll(DllHandle handle) {
#ifdef WEBRTC_LINUX
// TODO(pbos): Remove this dlclose() exclusion when leaks and suppressions from
// here are gone (or AddressSanitizer can display them properly).
//
// Skip dlclose() on AddressSanitizer as leaks including this module in the
// stack trace gets displayed as <unknown module> instead of the actual library
// -> it can not be suppressed.
// https://code.google.com/p/address-sanitizer/issues/detail?id=89
#if !defined(ADDRESS_SANITIZER)
  if (dlclose(handle) != 0) {
    RTC_LOG(LS_ERROR) << GetDllError();
  }
#endif  // !defined(ADDRESS_SANITIZER)
#else
#error Not implemented
#endif
}

static bool LoadSymbol(DllHandle handle,
                       const char* symbol_name,
                       void** symbol) {
#ifdef WEBRTC_LINUX
  *symbol = dlsym(handle, symbol_name);
  char* err = dlerror();
  if (err) {
    RTC_LOG(LS_ERROR) << "Error loading symbol " << symbol_name << " : " << err;
    return false;
  } else if (!*symbol) {
    RTC_LOG(LS_ERROR) << "Symbol " << symbol_name << " is NULL";
    return false;
  }
  return true;
#else
#error Not implemented
#endif
}

// This routine MUST assign SOME value for every symbol, even if that value is
// NULL, or else some symbols may be left with uninitialized data that the
// caller may later interpret as a valid address.
bool InternalLoadSymbols(DllHandle handle,
                         int num_symbols,
                         const char* const symbol_names[],
                         void* symbols[]) {
#ifdef WEBRTC_LINUX
  // Clear any old errors.
  dlerror();
#endif
  for (int i = 0; i < num_symbols; ++i) {
    if (!LoadSymbol(handle, symbol_names[i], &symbols[i])) {
      return false;
    }
  }
  return true;
}
```

符号表大小，动态链接库的名称，及符号表的定义是模板类 `LateBindingSymbolTable` 的模板参数，这几个模板参数都是数值参数，而不是类型参数。`LateBindingSymbolTable` 也可以不实现为模板类，比如，把符号表大小，动态链接库的名称，及符号表的定义作为构造函数的参数传入 `LateBindingSymbolTable` 类。但将动态链接库名称、符号表大小及符号表定义做成类的模板参数更方便对这些数据进行访问，同时也有利于为符号地址数组分配空间。

WebRTC 定义了几个宏，用于方便符号表的定义（`webrtc/modules/audio_device/linux/latebindingsymboltable_linux.h`），具体的 `LateBindingSymbolTable` 类的特化，及符号的访问：
```
// This macro must be invoked in a header to declare a symbol table class.
#define LATE_BINDING_SYMBOL_TABLE_DECLARE_BEGIN(ClassName) enum {
// This macro must be invoked in the header declaration once for each symbol
// (recommended to use an X-Macro to avoid duplication).
// This macro defines an enum with names built from the symbols, which
// essentially creates a hash table in the compiler from symbol names to their
// indices in the symbol table class.
#define LATE_BINDING_SYMBOL_TABLE_DECLARE_ENTRY(ClassName, sym) \
  ClassName##_SYMBOL_TABLE_INDEX_##sym,

// This macro completes the header declaration.
#define LATE_BINDING_SYMBOL_TABLE_DECLARE_END(ClassName)       \
  ClassName##_SYMBOL_TABLE_SIZE                                \
  }                                                            \
  ;                                                            \
                                                               \
  extern const char ClassName##_kDllName[];                    \
  extern const char* const                                     \
      ClassName##_kSymbolNames[ClassName##_SYMBOL_TABLE_SIZE]; \
                                                               \
  typedef ::webrtc::adm_linux::LateBindingSymbolTable<         \
      ClassName##_SYMBOL_TABLE_SIZE, ClassName##_kDllName,     \
      ClassName##_kSymbolNames>                                \
      ClassName;

// This macro must be invoked in a .cc file to define a previously-declared
// symbol table class.
#define LATE_BINDING_SYMBOL_TABLE_DEFINE_BEGIN(ClassName, dllName) \
  const char ClassName##_kDllName[] = dllName;                     \
  const char* const ClassName##_kSymbolNames[ClassName##_SYMBOL_TABLE_SIZE] = {
// This macro must be invoked in the .cc definition once for each symbol
// (recommended to use an X-Macro to avoid duplication).
// This would have to use the mangled name if we were to ever support C++
// symbols.
#define LATE_BINDING_SYMBOL_TABLE_DEFINE_ENTRY(ClassName, sym) #sym,

#define LATE_BINDING_SYMBOL_TABLE_DEFINE_END(ClassName) \
  }                                                     \
  ;
```

`LATE_BINDING_SYMBOL_TABLE_DECLARE_BEGIN`、`LATE_BINDING_SYMBOL_TABLE_DECLARE_ENTRY` 和 `LATE_BINDING_SYMBOL_TABLE_DECLARE_END` 这几个宏，需要在头文件中调用，它们首先定义了一个 enum，用于得到每个符号在符号表中的索引。`LATE_BINDING_SYMBOL_TABLE_DECLARE_END` 宏除了给 enum 定义添加结束的 `}` 和 `;` 之外，它还为 enum 增加了最后一个 item，用于得到符号表的大小。此外，`LATE_BINDING_SYMBOL_TABLE_DECLARE_END` 宏还声明了动态链接库名称，符号表，及具体 `LateBindingSymbolTable` 类的特化。

如 Pulse ADM 的 `webrtc/modules/audio_device/linux/pulseaudiosymboltable_linux.h` 头文件：
```
// The PulseAudio symbols we need, as an X-Macro list.
// This list must contain precisely every libpulse function that is used in
// the ADM LINUX PULSE Device and Mixer classes
#define PULSE_AUDIO_SYMBOLS_LIST           \
  X(pa_bytes_per_second)                   \
  X(pa_context_connect)                    \
  X(pa_context_disconnect)                 \
  X(pa_context_errno)                      \
  X(pa_context_get_protocol_version)       \
  X(pa_context_get_server_info)            \
  X(pa_context_get_sink_info_list)         \
  X(pa_context_get_sink_info_by_index)     \
  X(pa_context_get_sink_info_by_name)      \
  X(pa_context_get_sink_input_info)        \
  X(pa_context_get_source_info_by_index)   \
  X(pa_context_get_source_info_by_name)    \
  X(pa_context_get_source_info_list)       \
  X(pa_context_get_state)                  \
  X(pa_context_new)                        \
  X(pa_context_set_sink_input_volume)      \
  X(pa_context_set_sink_input_mute)        \
  X(pa_context_set_source_volume_by_index) \
  X(pa_context_set_source_mute_by_index)   \
  X(pa_context_set_state_callback)         \
  X(pa_context_unref)                      \
  X(pa_cvolume_set)                        \
  X(pa_operation_get_state)                \
  X(pa_operation_unref)                    \
  X(pa_stream_connect_playback)            \
  X(pa_stream_connect_record)              \
  X(pa_stream_disconnect)                  \
  X(pa_stream_drop)                        \
  X(pa_stream_get_device_index)            \
  X(pa_stream_get_index)                   \
  X(pa_stream_get_latency)                 \
  X(pa_stream_get_sample_spec)             \
  X(pa_stream_get_state)                   \
  X(pa_stream_new)                         \
  X(pa_stream_peek)                        \
  X(pa_stream_readable_size)               \
  X(pa_stream_set_buffer_attr)             \
  X(pa_stream_set_overflow_callback)       \
  X(pa_stream_set_read_callback)           \
  X(pa_stream_set_state_callback)          \
  X(pa_stream_set_underflow_callback)      \
  X(pa_stream_set_write_callback)          \
  X(pa_stream_unref)                       \
  X(pa_stream_writable_size)               \
  X(pa_stream_write)                       \
  X(pa_strerror)                           \
  X(pa_threaded_mainloop_free)             \
  X(pa_threaded_mainloop_get_api)          \
  X(pa_threaded_mainloop_lock)             \
  X(pa_threaded_mainloop_new)              \
  X(pa_threaded_mainloop_signal)           \
  X(pa_threaded_mainloop_start)            \
  X(pa_threaded_mainloop_stop)             \
  X(pa_threaded_mainloop_unlock)           \
  X(pa_threaded_mainloop_wait)

LATE_BINDING_SYMBOL_TABLE_DECLARE_BEGIN(PulseAudioSymbolTable)
#define X(sym) \
  LATE_BINDING_SYMBOL_TABLE_DECLARE_ENTRY(PulseAudioSymbolTable, sym)
PULSE_AUDIO_SYMBOLS_LIST
#undef X
LATE_BINDING_SYMBOL_TABLE_DECLARE_END(PulseAudioSymbolTable)
```

这个头文件里，X 宏是对 `LATE_BINDING_SYMBOL_TABLE_DECLARE_ENTRY` 的封装。`webrtc/modules/audio_device/linux/pulseaudiosymboltable_linux.h` 头文件中调用的各个宏都展开之后，得到如下内容：
```
enum {
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_bytes_per_second,                   \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_connect,                    \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_disconnect,                 \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_errno,                      \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_protocol_version,       \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_server_info,            \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_sink_info_list,         \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_sink_info_by_index,     \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_sink_info_by_name,      \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_sink_input_info,        \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_source_info_by_index,   \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_source_info_by_name,    \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_source_info_list,       \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_get_state,                  \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_new,                        \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_set_sink_input_volume,      \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_set_sink_input_mute,        \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_set_source_volume_by_index, \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_set_source_mute_by_index,   \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_set_state_callback,         \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_context_unref,                      \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_cvolume_set,                        \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_operation_get_state,                \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_operation_unref,                    \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_connect_playback,            \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_connect_record,              \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_disconnect,                  \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_drop,                        \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_get_device_index,            \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_get_index,                   \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_get_latency,                 \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_get_sample_spec,             \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_get_state,                   \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_new,                         \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_peek,                        \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_readable_size,               \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_set_buffer_attr,             \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_set_overflow_callback,       \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_set_read_callback,           \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_set_state_callback,          \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_set_underflow_callback,      \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_set_write_callback,          \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_unref,                       \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_writable_size,               \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_stream_write,                       \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_strerror,                           \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_free,             \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_get_api,          \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_lock,             \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_new,              \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_signal,           \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_start,            \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_stop,             \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_unlock,           \
  PulseAudioSymbolTable_SYMBOL_TABLE_INDEX_pa_threaded_mainloop_wait,
  PulseAudioSymbolTable_SYMBOL_TABLE_SIZE                                \
  }                                                            \
  ;                                                            \
                                                               \
  extern const char PulseAudioSymbolTable_kDllName[];                    \
  extern const char* const                                     \
      PulseAudioSymbolTable_kSymbolNames[PulseAudioSymbolTable_SYMBOL_TABLE_SIZE]; \
                                                               \
  typedef ::webrtc::adm_linux::LateBindingSymbolTable<         \
      PulseAudioSymbolTable_SYMBOL_TABLE_SIZE, PulseAudioSymbolTable_kDllName,     \
      PulseAudioSymbolTable_kSymbolNames>                                \
      PulseAudioSymbolTable;
```

`LATE_BINDING_SYMBOL_TABLE_DEFINE_BEGIN` 、`LATE_BINDING_SYMBOL_TABLE_DEFINE_ENTRY` 和 `LATE_BINDING_SYMBOL_TABLE_DEFINE_END` 宏，需要在 C++ 源文件中调用，用于定义动态链接库的名称，及符号表。如在 `webrtc/modules/audio_device/linux/pulseaudiosymboltable_linux.cc` 文件中：
```
#include "modules/audio_device/linux/pulseaudiosymboltable_linux.h"

namespace webrtc {
namespace adm_linux_pulse {

LATE_BINDING_SYMBOL_TABLE_DEFINE_BEGIN(PulseAudioSymbolTable, "libpulse.so.0")
#define X(sym) \
  LATE_BINDING_SYMBOL_TABLE_DEFINE_ENTRY(PulseAudioSymbolTable, sym)
PULSE_AUDIO_SYMBOLS_LIST
#undef X
LATE_BINDING_SYMBOL_TABLE_DEFINE_END(PulseAudioSymbolTable)

}  // namespace adm_linux_pulse
}  // namespace webrtc
```

在这个源文件中，`X` 宏是对 `LATE_BINDING_SYMBOL_TABLE_DEFINE_ENTRY` 宏的封装。这些宏展开之后得到如下内容：
```
const char PulseAudioSymbolTable_kDllName[] = "libpulse.so.0";                     \
  const char* const PulseAudioSymbolTable_kSymbolNames[PulseAudioSymbolTable_SYMBOL_TABLE_SIZE] = {
"pa_bytes_per_second",                   \
  "pa_context_connect",                    \
  "pa_context_disconnect",                 \
  "pa_context_errno",                      \
  "pa_context_get_protocol_version",       \
  "pa_context_get_server_info",            \
  "pa_context_get_sink_info_list",         \
  "pa_context_get_sink_info_by_index",     \
  "pa_context_get_sink_info_by_name",      \
  "pa_context_get_sink_input_info",        \
  "pa_context_get_source_info_by_index",   \
  "pa_context_get_source_info_by_name",    \
  "pa_context_get_source_info_list",       \
  "pa_context_get_state",                  \
  "pa_context_new",                        \
  "pa_context_set_sink_input_volume",      \
  "pa_context_set_sink_input_mute",        \
  "pa_context_set_source_volume_by_index", \
  "pa_context_set_source_mute_by_index",   \
  "pa_context_set_state_callback",         \
  "pa_context_unref",                      \
  "pa_cvolume_set",                        \
  "pa_operation_get_state",                \
  "pa_operation_unref",                    \
  "pa_stream_connect_playback",            \
  "pa_stream_connect_record",              \
  "pa_stream_disconnect",                  \
  "pa_stream_drop",                        \
  "pa_stream_get_device_index",            \
  "pa_stream_get_index",                   \
  "pa_stream_get_latency",                 \
  "pa_stream_get_sample_spec",             \
  "pa_stream_get_state",                   \
  "pa_stream_new",                         \
  "pa_stream_peek",                        \
  "pa_stream_readable_size",               \
  "pa_stream_set_buffer_attr",             \
  "pa_stream_set_overflow_callback",       \
  "pa_stream_set_read_callback",           \
  "pa_stream_set_state_callback",          \
  "pa_stream_set_underflow_callback",      \
  "pa_stream_set_write_callback",          \
  "pa_stream_unref",                       \
  "pa_stream_writable_size",               \
  "pa_stream_write",                       \
  "pa_strerror",                           \
  "pa_threaded_mainloop_free",             \
  "pa_threaded_mainloop_get_api",          \
  "pa_threaded_mainloop_lock",             \
  "pa_threaded_mainloop_new",              \
  "pa_threaded_mainloop_signal",           \
  "pa_threaded_mainloop_start",            \
  "pa_threaded_mainloop_stop",             \
  "pa_threaded_mainloop_unlock",           \
  "pa_threaded_mainloop_wait",
}                                                     \
  ;
```

WebRTC 定义了几个宏（`webrtc/modules/audio_device/linux/latebindingsymboltable_linux.h`），用于方便对符号的调用：
```
// Index of a given symbol in the given symbol table class.
#define LATESYM_INDEXOF(ClassName, sym) (ClassName##_SYMBOL_TABLE_INDEX_##sym)

// Returns a reference to the given late-binded symbol, with the correct type.
#define LATESYM_GET(ClassName, inst, sym) \
  (*reinterpret_cast<__typeof__(&sym)>(   \
      (inst)->GetSymbol(LATESYM_INDEXOF(ClassName, sym))))
```

需要访问动态加载的库的符号的组件，需要包含符号声明的头文件，如 `AudioDeviceLinuxPulse` 类需要调用 Pulse 库的函数，则在头文件`webrtc/modules/audio_device/linux/audio_device_pulse_linux.h` 中包含了 `pulse/pulseaudio.h` 头文件。上面看到的 ***`LATESYM_GET` 宏，通过符号的声明及编译器的 `__typeof__` 扩展得到一个符号的类型，以方便对于函数的调用***。

`LATESYM_GET` 宏用于借助具体的 `LateBindingSymbolTable` 类的对象实例，访问某个函数，`LATESYM_INDEXOF` 宏用于获得某个符号在符号表中的索引。对动态加载的库的符号的调用，需要配合在源文件中定义的另一个宏来实现，如 `webrtc/modules/audio_device/linux/audio_device_pulse_linux.cc` 文件中：
```
webrtc::adm_linux_pulse::PulseAudioSymbolTable PaSymbolTable;

// Accesses Pulse functions through our late-binding symbol table instead of
// directly. This way we don't have to link to libpulse, which means our binary
// will work on systems that don't have it.
#define LATE(sym)                                                             \
  LATESYM_GET(webrtc::adm_linux_pulse::PulseAudioSymbolTable, &PaSymbolTable, \
              sym)
```

如 `AudioDeviceLinuxPulse` 类的 `RecordingDevices()` 函数调用 Pulse 库的 `pa_context_get_source_info_list()` 函数：
```
int16_t AudioDeviceLinuxPulse::RecordingDevices() {
  PaLock();

  pa_operation* paOperation = NULL;
  _numRecDevices = 1;  // Init to 1 to account for "default"

  // Get the whole list of devices and update _numRecDevices
  paOperation = LATE(pa_context_get_source_info_list)(
      _paContext, PaSourceInfoCallback, this);

  WaitForOperationCompletion(paOperation);

  PaUnLock();

  return _numRecDevices;
}
```

总结一下，WebRTC Linux ADM 实现中的符号延迟加载机制由如下这些部分组成：
 - libdl 的模板 wrapper 类
 - 辅助定义具体的 libdl wrapper 类的宏，定义方便访问符号的 enum 的宏，用于声明符号表和动态链接库名称的宏
 - 用于定义动态链接库名称和符号表的宏。
 - 用于访问特定符号的宏。

WebRTC 实现的这套东西存在一些问题：
1. 重复逻辑没有完全消除。为了访问延迟加载的符号，重复定义了用于访问符号的宏。如在 `webrtc/modules/audio_device/linux/audio_device_alsa_linux.cc`、`webrtc/modules/audio_device/linux/audio_device_pulse_linux.cc`、`webrtc/modules/audio_device/linux/audio_mixer_manager_alsa_linux.cc`、`webrtc/modules/audio_device/linux/audio_mixer_manager_pulse_linux.cc` 这几个文件中重复定义了 `LATE` 宏。

2. 创建具体的 AudioDeviceGeneric 类型对象时，没有判断系统是否真的能够支持。即在 `webrtc/modules/audio_device/audio_device_impl.cc` 文件中，仅仅根据编译时的宏开关和传入的 audio_layer 参数来决定创建的具体 AudioDeviceGeneric 类型对象，而没有判断系统是否能够支持。

对于上面的第 2 个问题，可以创建一个 `AudioManager` 类，用于判断系统是否支持，如 `AudioManager` 的声明如下：
```
#ifndef MODULES_AUDIO_DEVICE_LINUX_AUDIO_MANAGER_H_
#define MODULES_AUDIO_DEVICE_LINUX_AUDIO_MANAGER_H_

namespace webrtc {

class AudioManager {
public:
  AudioManager();
  virtual ~AudioManager();
  bool IsALSAAudioSupported();
  bool IsPulseAudioSupported();
};

}  // namespace webrtc

#endif /* MODULES_AUDIO_DEVICE_LINUX_AUDIO_MANAGER_H_ */
```

`AudioManager` 类的定义如下：
```
#include "audio_manager.h"

#if defined(LINUX_ALSA)
#include "modules/audio_device/linux/alsasymboltable_linux.h"
#endif

#if defined(LINUX_PULSE)
#include "modules/audio_device/linux/pulseaudiosymboltable_linux.h"
#endif

#if defined(LINUX_ALSA)
extern webrtc::adm_linux_alsa::AlsaSymbolTable AlsaSymbolTable;
#endif

#if defined(LINUX_PULSE)
extern webrtc::adm_linux_pulse::PulseAudioSymbolTable PaSymbolTable;
#endif

namespace webrtc {

AudioManager::AudioManager() {}

AudioManager::~AudioManager() {}

bool AudioManager::IsALSAAudioSupported() {
#if defined(LINUX_ALSA)
  return AlsaSymbolTable.Load();
#else
  return false;
#endif
}

bool AudioManager::IsPulseAudioSupported() {
#if defined(LINUX_PULSE)
  return PaSymbolTable.Load();
#else
  return false;
#endif
}

}  // namespace webrtc
```

而 `webrtc/modules/audio_device/audio_device_impl.cc` 中的相关逻辑则可以修改为如下这样：
```
#elif defined(WEBRTC_LINUX)
  std::unique_ptr<AudioManager> audio_manager(new AudioManager());
#if !defined(LINUX_PULSE)
  // Build flag 'rtc_include_pulse_audio' is set to false. In this mode:
  // - kPlatformDefaultAudio => ALSA, and
  // - kLinuxAlsaAudio => ALSA, and
  // - kLinuxPulseAudio => Invalid selection.
  RTC_LOG(WARNING) << "PulseAudio is disabled using build flag.";
  if ((audio_layer == kLinuxAlsaAudio) ||
      (audio_layer == kPlatformDefaultAudio)) {
    audio_device_.reset(new AudioDeviceLinuxALSA());
    RTC_LOG(INFO) << "Linux ALSA APIs will be utilized.";
  }
#else
  // Build flag 'rtc_include_pulse_audio' is set to true (default). In this
  // mode:
  // - kPlatformDefaultAudio => PulseAudio, and
  // - kLinuxPulseAudio => PulseAudio, and
  // - kLinuxAlsaAudio => ALSA (supported but not default).
  RTC_LOG(INFO) << "PulseAudio support is enabled.";
  if (((audio_layer == kLinuxPulseAudio) ||
      (audio_layer == kPlatformDefaultAudio)) && audio_manager->IsPulseAudioSupported()) {
    // Linux PulseAudio implementation is default.
    audio_device_.reset(new AudioDeviceLinuxPulse());
    RTC_LOG(INFO) << "Linux PulseAudio APIs will be utilized";
  } else if ((audio_layer == kLinuxAlsaAudio) ||
      (audio_layer == kPlatformDefaultAudio)) {
    audio_device_.reset(new AudioDeviceLinuxALSA());
    RTC_LOG(WARNING) << "Linux ALSA APIs will be utilized.";
  }
#endif  // #if !defined(LINUX_PULSE)
#endif  // #if defined(WEBRTC_LINUX)
```

不过这种做法也有一些问题，即在判断系统是否支持时，就提前加载了对应的系统库。
