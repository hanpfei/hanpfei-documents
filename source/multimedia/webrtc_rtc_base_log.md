---
title: WebRTC 的 log 系统实现分析
date: 2020-08-29 21:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

WebRTC 有多套 log 输出系统，一套位于 `webrtc/src/base` 下，包括 `webrtc/src/base/logging.h` 和 `webrtc/src/base/logging.cc` 等文件，主要的 class 位于 namespace `logging` 中。另一套位于 `webrtc/src/rtc_base` 下，包括 `webrtc/src/rtc_base/logging.h` 和 `webrtc/src/rtc_base/logging.cc` 等文件，主要的 class 位于 namespace `rtc` 中。本文主要分析 namespace `rtc` 中的这套 log 输出系统。它可以非常方便地输出类似于下面这种格式的 log：
```
(bitrate_prober.cc:64): Bandwidth probing enabled, set to inactive
(rtp_transport_controller_send.cc:45): Using TaskQueue based SSCC
(paced_sender.cc:362): ProcessThreadAttached 0x0x7fb2e813c0d0
```
<!--more-->
log 中包含了输出 log 的具体文件位置，和特定的要输出的 log 内容信息。

整体来看，这个 log 系统可以分为四层：对于 log 系统的用户，看到的是用于输出不同重要性的 log 的一组宏，如 `RTC_LOG(sev)`，这组宏提供了类似于输出流 `std::ostream` 的接口来输出 log；log 系统的核心是 `LogMessage`，它用于对要输出的 log 做格式化，产生最终要输出的字符串，并把产生的字符串送给 log 后端完成 log 输出，同时还可以通过它对 log 系统的行为施加控制；最下面的是 log 后端，log 后端用 `LogSink` 接口描述；在 `RTC_LOG(sev)` 等宏和 `LogMessage` 之间的是用于实现 log 系统的用户看到的类似于输出流的接口的模板类  `LogStreamer` 和 `LogCall` 等。这个 log 系统整体如下图所示：

![WebRTC rtc log](images/1315506-f93b439075726485.png)

## Log 后端

Log 后端用 `LogSink` 接口描述，这个接口声明（位于 `webrtc/src/rtc_base/logging.h` 文件中）如下：
```
// Virtual sink interface that can receive log messages.
class LogSink {
 public:
  LogSink() {}
  virtual ~LogSink() {}
  virtual void OnLogMessage(const std::string& msg,
                            LoggingSeverity severity,
                            const char* tag);
  virtual void OnLogMessage(const std::string& message,
                            LoggingSeverity severity);
  virtual void OnLogMessage(const std::string& message) = 0;
};
```

`LogSink` 用于从 `LogMessage` 接收不同 severity 的 log 消息。`LogSink` 的两个非纯虚函数实现如下（位于 `webrtc/src/rtc_base/logging.cc`）：
```
// Inefficient default implementation, override is recommended.
void LogSink::OnLogMessage(const std::string& msg,
                           LoggingSeverity severity,
                           const char* tag) {
  OnLogMessage(tag + (": " + msg), severity);
}

void LogSink::OnLogMessage(const std::string& msg,
                           LoggingSeverity /* severity */) {
  OnLogMessage(msg);
}
```

## LogMessage
### log 系统的全局控制
`LogMessage` 可以用于对 log 系统施加一些全局性的控制，这主要是指如下这些全局性的状态：

 - log 输出的最低 severity，通过 `g_min_sev` 控制；
 - debug 输出的最低 severity，通过 `g_dbg_sev` 控制；
 - 是否要向标准错误输出输出 log，通过 `log_to_stderr_` 开关控制；
 - 最终输出的日志信息中是否要包含线程信息和时间戳，通过 `thread_` 和 `timestamp_` 开关控制；
 - 要输出的 log 后端 `streams_`。

这里先看一下这套 log 系统支持的 log severity：
```
// Note that the non-standard LoggingSeverity aliases exist because they are
// still in broad use.  The meanings of the levels are:
//  LS_VERBOSE: This level is for data which we do not want to appear in the
//   normal debug log, but should appear in diagnostic logs.
//  LS_INFO: Chatty level used in debugging for all sorts of things, the default
//   in debug builds.
//  LS_WARNING: Something that may warrant investigation.
//  LS_ERROR: Something that should not have occurred.
//  LS_NONE: Don't log.
enum LoggingSeverity {
  LS_VERBOSE,
  LS_INFO,
  LS_WARNING,
  LS_ERROR,
  LS_NONE,
  INFO = LS_INFO,
  WARNING = LS_WARNING,
  LERROR = LS_ERROR
};
```

这套 log 系统包括 `LS_NONE` 支持 5 级 severity，各级 severity 的使用说明，如上面代码中的注释。

`LogMessage` 用一个静态的 `StreamList` 保存 log 后端 `LogSink`：
```
 private:
. . . . . .
  typedef std::pair<LogSink*, LoggingSeverity> StreamAndSeverity;
  typedef std::list<StreamAndSeverity> StreamList;
. . . . . .
  static StreamList streams_;
```

`LogMessage` 用一个列表记录 `LogSink` 对象的指针及其期望接收的 log 的最低的 severity。

`LogMessage` 提供了一些接口来访问这些全局状态。

`LogToDebug()`/`GetLogToDebug()` 可以用于存取 `g_dbg_sev`：
```
LoggingSeverity LogMessage::GetLogToDebug() {
  return g_dbg_sev;
}
. . . . . .
void LogMessage::LogToDebug(LoggingSeverity min_sev) {
  g_dbg_sev = min_sev;
  CritScope cs(&g_log_crit);
  UpdateMinLogSeverity();
}
```

`SetLogToStderr()` 可以用于设置 log_to_stderr_ 格式化开关：
```
void LogMessage::SetLogToStderr(bool log_to_stderr) {
  log_to_stderr_ = log_to_stderr;
}
```

`GetLogToStream()`/`AddLogToStream()`/`RemoveLogToStream()` 可以用于访问 streams_：
```
int LogMessage::GetLogToStream(LogSink* stream) {
  CritScope cs(&g_log_crit);
  LoggingSeverity sev = LS_NONE;
  for (auto& kv : streams_) {
    if (!stream || stream == kv.first) {
      sev = std::min(sev, kv.second);
    }
  }
  return sev;
}

void LogMessage::AddLogToStream(LogSink* stream, LoggingSeverity min_sev) {
  CritScope cs(&g_log_crit);
  streams_.push_back(std::make_pair(stream, min_sev));
  UpdateMinLogSeverity();
}

void LogMessage::RemoveLogToStream(LogSink* stream) {
  CritScope cs(&g_log_crit);
  for (StreamList::iterator it = streams_.begin(); it != streams_.end(); ++it) {
    if (stream == it->first) {
      streams_.erase(it);
      break;
    }
  }
  UpdateMinLogSeverity();
}
```

`g_min_sev` 的状态由各个 `LogSink` 接收 log 的最低 severity 和 `g_dbg_sev` 的状态共同决定，具体来说，它是 `g_dbg_sev` 和各个 `LogSink` 接收 log 的最低 severity 中的最低 severity：
```
void LogMessage::UpdateMinLogSeverity()
    RTC_EXCLUSIVE_LOCKS_REQUIRED(g_log_crit) {
  LoggingSeverity min_sev = g_dbg_sev;
  for (const auto& kv : streams_) {
    const LoggingSeverity sev = kv.second;
    min_sev = std::min(min_sev, sev);
  }
  g_min_sev = min_sev;
}
```

`LogMessage` 提供了几个函数用于访问 `g_min_sev`：
```
bool LogMessage::Loggable(LoggingSeverity sev) {
  return sev >= g_min_sev;
}

int LogMessage::GetMinLogSeverity() {
  return g_min_sev;
}
```

`LogThreads()`/`LogTimestamps()` 可以用于设置 `thread_` 和 `timestamp_` 格式化开关：
```
void LogMessage::LogThreads(bool on) {
  thread_ = on;
}

void LogMessage::LogTimestamps(bool on) {
  timestamp_ = on;
}
```

除了上面这些专用的控制接口之外，`LogMessage` 还提供了另一个函数 `ConfigureLogging()` 对 log 系统进行控制：
```
void LogMessage::ConfigureLogging(const char* params) {
  LoggingSeverity current_level = LS_VERBOSE;
  LoggingSeverity debug_level = GetLogToDebug();

  std::vector<std::string> tokens;
  tokenize(params, ' ', &tokens);

  for (const std::string& token : tokens) {
    if (token.empty())
      continue;

    // Logging features
    if (token == "tstamp") {
      LogTimestamps();
    } else if (token == "thread") {
      LogThreads();

      // Logging levels
    } else if (token == "verbose") {
      current_level = LS_VERBOSE;
    } else if (token == "info") {
      current_level = LS_INFO;
    } else if (token == "warning") {
      current_level = LS_WARNING;
    } else if (token == "error") {
      current_level = LS_ERROR;
    } else if (token == "none") {
      current_level = LS_NONE;

      // Logging targets
    } else if (token == "debug") {
      debug_level = current_level;
    }
  }

#if defined(WEBRTC_WIN) && !defined(WINUWP)
  if ((LS_NONE != debug_level) && !::IsDebuggerPresent()) {
    // First, attempt to attach to our parent's console... so if you invoke
    // from the command line, we'll see the output there.  Otherwise, create
    // our own console window.
    // Note: These methods fail if a console already exists, which is fine.
    if (!AttachConsole(ATTACH_PARENT_PROCESS))
      ::AllocConsole();
  }
#endif  // defined(WEBRTC_WIN) && !defined(WINUWP)

  LogToDebug(debug_level);
}
```

可以给这个函数传入一个字符串，同时修改 `g_dbg_sev`、`thread_` 和 `timestamp_` 等多个状态。

`g_dbg_sev`、`log_to_stderr_` 和 `g_min_sev` 之间的关系：`g_min_sev` 定义了 log 系统输出的 log 的最低 severity，当要输出的 log 的 severity 低于 `g_min_sev` 时，`LogMessage` 会认为它是不需要输出的。debug log 输出可以看作是一个特殊的封装了向标准错误输出或本地系统特定的 log 输出系统打印 log 的 `LogSink`，`g_dbg_sev` 用于控制向这个 `LogSink` 中输出的 log 的最低 severity，而 `log_to_stderr_` 则用于控制是否最终把 log 输出到标准错误输出。

对于上述操作全局状态的函数做一下总结，如下表所示：

| 全局状态 | 访问接口 | 说明 |
|------|-------|---------|
| g_dbg_sev | `LogToDebug()`/`GetLogToDebug()` | debug 输出的最低 severity |
| log_to_stderr_ | `SetLogToStderr()` | 是否要向标准错误输出输出 log |
| streams_ | `GetLogToStream()`/`AddLogToStream()`/`RemoveLogToStream()` | 要输出的 log 后端 |
| g_min_sev | `GetMinLogSeverity()`/`IsNoop()`/`Loggable()` | 状态由各个 `LogSink` 接收 log 的最低 severity 和 `g_dbg_sev` 的状态共同决定 |
| thread_ | `LogThreads()` | 最终输出的日志信息中是否要包含线程信息 |
| timestamp_ | `LogTimestamps()` | 最终输出的日志信息中是否要包含时间戳 |

### 输出 log

这个 log 系统输出的完整 log 信息如下面这样：
```
[002:138] [6134] (bitrate_prober.cc:64): Bandwidth probing enabled, set to inactive
[002:138] [6134] (rtp_transport_controller_send.cc:45): Using TaskQueue based SSCC
[002:138] [6134] (paced_sender.cc:362): ProcessThreadAttached 0x0x7ffbd42ece10
[002:138] [6134] (aimd_rate_control.cc:108): Using aimd rate control with back off factor 0.85
[002:138] [6134] (remote_bitrate_estimator_single_stream.cc:55): RemoteBitrateEstimatorSingleStream: Instantiating.
[002:138] [6134] (call.cc:1182): UpdateAggregateNetworkState: aggregate_state=down
[002:138] [6134] (send_side_congestion_controller.cc:581): SignalNetworkState Down
```

其中包含了时间戳和线程号。

`LogMessage` 对象持有一个 `rtc::StringBuilder` 类型的对象 `print_stream_`，要最终输出的内容，都会先被送进 `print_stream_`。`LogMessage` 对象构造时，根据上面介绍的全局性的控制状态，向 `print_stream_` 输出 log header，主要包括 log 输出的文件名和行号，可能包括时间戳和线程号。之后 `LogMessage` 的使用者可以获得它的 `rtc::StringBuilder`，并向其中输出任何数量和格式的 log 信息。在 `LogMessage` 对象析构时，会从 `print_stream_` 获得字符串，并把字符串输出到 debug log，或者外部注册的 `LogSink` 中。

`LogMessage` 对象的构造过程如下：
```
LogMessage::LogMessage(const char* file, int line, LoggingSeverity sev)
    : LogMessage(file, line, sev, ERRCTX_NONE, 0) {}

LogMessage::LogMessage(const char* file,
                       int line,
                       LoggingSeverity sev,
                       LogErrorContext err_ctx,
                       int err)
    : severity_(sev) {
  if (timestamp_) {
    // Use SystemTimeMillis so that even if tests use fake clocks, the timestamp
    // in log messages represents the real system time.
    int64_t time = TimeDiff(SystemTimeMillis(), LogStartTime());
    // Also ensure WallClockStartTime is initialized, so that it matches
    // LogStartTime.
    WallClockStartTime();
    print_stream_ << "[" << rtc::LeftPad('0', 3, rtc::ToString(time / 1000))
                  << ":" << rtc::LeftPad('0', 3, rtc::ToString(time % 1000))
                  << "] ";
  }

  if (thread_) {
    PlatformThreadId id = CurrentThreadId();
    print_stream_ << "[" << id << "] ";
  }

  if (file != nullptr) {
#if defined(WEBRTC_ANDROID)
    tag_ = FilenameFromPath(file);
    print_stream_ << "(line " << line << "): ";
#else
    print_stream_ << "(" << FilenameFromPath(file) << ":" << line << "): ";
#endif
  }

  if (err_ctx != ERRCTX_NONE) {
    char tmp_buf[1024];
    SimpleStringBuilder tmp(tmp_buf);
    tmp.AppendFormat("[0x%08X]", err);
    switch (err_ctx) {
      case ERRCTX_ERRNO:
        tmp << " " << strerror(err);
        break;
#ifdef WEBRTC_WIN
      case ERRCTX_HRESULT: {
        char msgbuf[256];
        DWORD flags =
            FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS;
        if (DWORD len = FormatMessageA(
                flags, nullptr, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
                msgbuf, sizeof(msgbuf) / sizeof(msgbuf[0]), nullptr)) {
          while ((len > 0) &&
                 isspace(static_cast<unsigned char>(msgbuf[len - 1]))) {
            msgbuf[--len] = 0;
          }
          tmp << " " << msgbuf;
        }
        break;
      }
#endif  // WEBRTC_WIN
#if defined(WEBRTC_MAC) && !defined(WEBRTC_IOS)
      case ERRCTX_OSSTATUS: {
        std::string desc(DescriptionFromOSStatus(err));
        tmp << " " << (desc.empty() ? "Unknown error" : desc.c_str());
        break;
      }
#endif  // WEBRTC_MAC && !defined(WEBRTC_IOS)
      default:
        break;
    }
    extra_ = tmp.str();
  }
}

#if defined(WEBRTC_ANDROID)
LogMessage::LogMessage(const char* file,
                       int line,
                       LoggingSeverity sev,
                       const char* tag)
    : LogMessage(file, line, sev, ERRCTX_NONE, 0 /* err */) {
  tag_ = tag;
  print_stream_ << tag << ": ";
}
#endif

// DEPRECATED. Currently only used by downstream projects that use
// implementation details of logging.h. Work is ongoing to remove those
// dependencies.
LogMessage::LogMessage(const char* file,
                       int line,
                       LoggingSeverity sev,
                       const std::string& tag)
    : LogMessage(file, line, sev) {
  print_stream_ << tag << ": ";
}
```

`LogStartTime()` 用于记录或获得首次日志时间，`WallClockStartTime()` 则似乎没有看到其应用场合：
```
int64_t LogMessage::LogStartTime() {
  static const int64_t g_start = SystemTimeMillis();
  return g_start;
}

uint32_t LogMessage::WallClockStartTime() {
  static const uint32_t g_start_wallclock = time(nullptr);
  return g_start_wallclock;
}
```

WebRTC 的 log 消息中的时间戳是相对时间，而不是绝对的墙上时间。

`FilenameFromPath()` 函数用于从文件的绝对路径中提取文件名：
```
// Return the filename portion of the string (that following the last slash).
const char* FilenameFromPath(const char* file) {
  const char* end1 = ::strrchr(file, '/');
  const char* end2 = ::strrchr(file, '\\');
  if (!end1 && !end2)
    return file;
  else
    return (end1 > end2) ? end1 + 1 : end2 + 1;
}
```

`LogMessage` 的使用者可以通过其 `stream()` 函数获得其 `rtc::StringBuilder`：
```
rtc::StringBuilder& LogMessage::stream() {
  return print_stream_;
}
```

`LogMessage` 对象的析构过程如下：
```
LogMessage::~LogMessage() {
  FinishPrintStream();

  const std::string str = print_stream_.Release();

  if (severity_ >= g_dbg_sev) {
#if defined(WEBRTC_ANDROID)
    OutputToDebug(str, severity_, tag_);
#else
    OutputToDebug(str, severity_);
#endif
  }

  CritScope cs(&g_log_crit);
  for (auto& kv : streams_) {
    if (severity_ >= kv.second) {
#if defined(WEBRTC_ANDROID)
      kv.first->OnLogMessage(str, severity_, tag_);
#else
      kv.first->OnLogMessage(str, severity_);
#endif
    }
  }
}
```

在要输出的 log 的 severity 高于 `g_dbg_sev` 时，`LogMessage` 对象的析构函数会将 log 输出到 debug；`LogMessage` 对象的析构函数还会把 log 输出到所有其请求的最低 severity 低于当前这条 log 消息的 severity 的 `LogSink`。

所谓的输出到 debug 的过程 `OutputToDebug()` 如下：
```
#if defined(WEBRTC_ANDROID)
void LogMessage::OutputToDebug(const std::string& str,
                               LoggingSeverity severity,
                               const char* tag) {
#else
void LogMessage::OutputToDebug(const std::string& str,
                               LoggingSeverity severity) {
#endif
  bool log_to_stderr = log_to_stderr_;
#if defined(WEBRTC_MAC) && !defined(WEBRTC_IOS) && defined(NDEBUG)
  // On the Mac, all stderr output goes to the Console log and causes clutter.
  // So in opt builds, don't log to stderr unless the user specifically sets
  // a preference to do so.
  CFStringRef key = CFStringCreateWithCString(
      kCFAllocatorDefault, "logToStdErr", kCFStringEncodingUTF8);
  CFStringRef domain = CFBundleGetIdentifier(CFBundleGetMainBundle());
  if (key != nullptr && domain != nullptr) {
    Boolean exists_and_is_valid;
    Boolean should_log =
        CFPreferencesGetAppBooleanValue(key, domain, &exists_and_is_valid);
    // If the key doesn't exist or is invalid or is false, we will not log to
    // stderr.
    log_to_stderr = exists_and_is_valid && should_log;
  }
  if (key != nullptr) {
    CFRelease(key);
  }
#endif  // defined(WEBRTC_MAC) && !defined(WEBRTC_IOS) && defined(NDEBUG)

#if defined(WEBRTC_WIN)
  // Always log to the debugger.
  // Perhaps stderr should be controlled by a preference, as on Mac?
  OutputDebugStringA(str.c_str());
  if (log_to_stderr) {
    // This handles dynamically allocated consoles, too.
    if (HANDLE error_handle = ::GetStdHandle(STD_ERROR_HANDLE)) {
      log_to_stderr = false;
      DWORD written = 0;
      ::WriteFile(error_handle, str.data(), static_cast<DWORD>(str.size()),
                  &written, 0);
    }
  }
#endif  // WEBRTC_WIN

#if defined(WEBRTC_ANDROID)
  // Android's logging facility uses severity to log messages but we
  // need to map libjingle's severity levels to Android ones first.
  // Also write to stderr which maybe available to executable started
  // from the shell.
  int prio;
  switch (severity) {
    case LS_VERBOSE:
      prio = ANDROID_LOG_VERBOSE;
      break;
    case LS_INFO:
      prio = ANDROID_LOG_INFO;
      break;
    case LS_WARNING:
      prio = ANDROID_LOG_WARN;
      break;
    case LS_ERROR:
      prio = ANDROID_LOG_ERROR;
      break;
    default:
      prio = ANDROID_LOG_UNKNOWN;
  }

  int size = str.size();
  int line = 0;
  int idx = 0;
  const int max_lines = size / kMaxLogLineSize + 1;
  if (max_lines == 1) {
    __android_log_print(prio, tag, "%.*s", size, str.c_str());
  } else {
    while (size > 0) {
      const int len = std::min(size, kMaxLogLineSize);
      // Use the size of the string in the format (str may have \0 in the
      // middle).
      __android_log_print(prio, tag, "[%d/%d] %.*s", line + 1, max_lines, len,
                          str.c_str() + idx);
      idx += len;
      size -= len;
      ++line;
    }
  }
#endif  // WEBRTC_ANDROID
  if (log_to_stderr) {
    fprintf(stderr, "%s", str.c_str());
    fflush(stderr);
  }
}
```

`OutputToDebug()` 将日志输出到本地系统特定的 log 系统或标准错误输出，如 Android 的 logcat。前面介绍的 `log_to_stderr_` 开关在这个函数中起作用。

## Log 输出用户接口及其实现

这套 log 系统的用户一般不会直接使用 `LogMessage` 打 log，而是会使用基于 `LogMessage` 封装的宏，如 `RTC_LOG()` 和 `RTC_DLOG()` 等，像下面这样：
```
  RTC_LOG(INFO) << "AudioDeviceBuffer::~dtor";
  RTC_LOG(LS_ERROR) << "Failed to set audio transport since media was active";
  RTC_LOG(INFO) << "Terminate";
  RTC_DLOG(LS_WARNING) << "Stereo mode is enabled";
```

log 系统的用户使用的宏主要有 `RTC_DLOG*` 系列和 `RTC_LOG*` 系列。`RTC_DLOG` 宏与他们对应的 `RTC_LOG` 宏是等价的，除了它们只在 debug builds 中才生成代码外，`RTC_LOG` 系列宏的定义如下：
```
// The RTC_DLOG macros are equivalent to their RTC_LOG counterparts except that
// they only generate code in debug builds.
#if RTC_DLOG_IS_ON
#define RTC_DLOG(sev) RTC_LOG(sev)
#define RTC_DLOG_V(sev) RTC_LOG_V(sev)
#define RTC_DLOG_F(sev) RTC_LOG_F(sev)
#else
#define RTC_DLOG_EAT_STREAM_PARAMS() \
  while (false)                      \
  rtc::webrtc_logging_impl::LogStreamer<>()
#define RTC_DLOG(sev) RTC_DLOG_EAT_STREAM_PARAMS()
#define RTC_DLOG_V(sev) RTC_DLOG_EAT_STREAM_PARAMS()
#define RTC_DLOG_F(sev) RTC_DLOG_EAT_STREAM_PARAMS()
#endif
```

Android 平台有一个平台专用的可以带上 TAG 的宏：
```
#ifdef WEBRTC_ANDROID

namespace webrtc_logging_impl {
// TODO(kwiberg): Replace these with absl::string_view.
inline const char* AdaptString(const char* str) {
  return str;
}
inline const char* AdaptString(const std::string& str) {
  return str.c_str();
}
}  // namespace webrtc_logging_impl

#define RTC_LOG_TAG(sev, tag)                                                \
    rtc::webrtc_logging_impl::LogCall() &                                    \
        rtc::webrtc_logging_impl::LogStreamer<>()                            \
            << rtc::webrtc_logging_impl::LogMetadataTag {                    \
      sev, rtc::webrtc_logging_impl::AdaptString(tag)                        \
    }

#else

// DEPRECATED. This macro is only intended for Android.
#define RTC_LOG_TAG(sev, tag) RTC_LOG_V(sev)

#endif
```

Windows 平台也有一些平台专用的宏：
```
#if defined(WEBRTC_WIN)
#define RTC_LOG_GLE_EX(sev, err) RTC_LOG_E(sev, HRESULT, err)
#define RTC_LOG_GLE(sev) RTC_LOG_GLE_EX(sev, static_cast<int>(GetLastError()))
#define RTC_LOG_ERR_EX(sev, err) RTC_LOG_GLE_EX(sev, err)
#define RTC_LOG_ERR(sev) RTC_LOG_GLE(sev)
#elif defined(__native_client__) && __native_client__
#define RTC_LOG_ERR_EX(sev, err) RTC_LOG(sev)
#define RTC_LOG_ERR(sev) RTC_LOG(sev)
#elif defined(WEBRTC_POSIX)
#define RTC_LOG_ERR_EX(sev, err) RTC_LOG_ERRNO_EX(sev, err)
#define RTC_LOG_ERR(sev) RTC_LOG_ERRNO(sev)
#endif  // WEBRTC_WIN
```

`RTC_LOG` 系列宏的实现如下：
```
//////////////////////////////////////////////////////////////////////
// Logging Helpers
//////////////////////////////////////////////////////////////////////

#define RTC_LOG_FILE_LINE(sev, file, line)      \
  rtc::webrtc_logging_impl::LogCall() &         \
      rtc::webrtc_logging_impl::LogStreamer<>() \
          << rtc::webrtc_logging_impl::LogMetadata(file, line, sev)

#define RTC_LOG(sev) RTC_LOG_FILE_LINE(rtc::sev, __FILE__, __LINE__)

// The _V version is for when a variable is passed in.
#define RTC_LOG_V(sev) RTC_LOG_FILE_LINE(sev, __FILE__, __LINE__)

// The _F version prefixes the message with the current function name.
#if (defined(__GNUC__) && !defined(NDEBUG)) || defined(WANT_PRETTY_LOG_F)
#define RTC_LOG_F(sev) RTC_LOG(sev) << __PRETTY_FUNCTION__ << ": "
#define RTC_LOG_T_F(sev) \
  RTC_LOG(sev) << this << ": " << __PRETTY_FUNCTION__ << ": "
#else
#define RTC_LOG_F(sev) RTC_LOG(sev) << __FUNCTION__ << ": "
#define RTC_LOG_T_F(sev) RTC_LOG(sev) << this << ": " << __FUNCTION__ << ": "
#endif

#define RTC_LOG_CHECK_LEVEL(sev) rtc::LogCheckLevel(rtc::sev)
#define RTC_LOG_CHECK_LEVEL_V(sev) rtc::LogCheckLevel(sev)

inline bool LogCheckLevel(LoggingSeverity sev) {
  return (LogMessage::GetMinLogSeverity() <= sev);
}

#define RTC_LOG_E(sev, ctx, err)                                    \
    rtc::webrtc_logging_impl::LogCall() &                           \
        rtc::webrtc_logging_impl::LogStreamer<>()                   \
            << rtc::webrtc_logging_impl::LogMetadataErr {           \
      {__FILE__, __LINE__, rtc::sev}, rtc::ERRCTX_##ctx, (err)      \
    }

#define RTC_LOG_T(sev) RTC_LOG(sev) << this << ": "

#define RTC_LOG_ERRNO_EX(sev, err) RTC_LOG_E(sev, ERRNO, err)
#define RTC_LOG_ERRNO(sev) RTC_LOG_ERRNO_EX(sev, errno)
```

`RTC_LOG_T` 宏会把当前类的 `this` 指针的值在 log 消息中显示出来。`RTC_LOG_T` 宏基于 `RTC_LOG` 宏实现。
`RTC_LOG_F` 宏会把函数的函数名在 log 消息中显示出来。
`RTC_LOG_T_F` 宏则会把当前类的 `this` 指针和函数的函数名在 log 消息中都显示出来。
在平台支持 `__PRETTY_FUNCTION__` 时，`RTC_LOG_F` 和 `RTC_LOG_T_F` 宏还会以更漂亮的格式显示函数名，即会连同函数的签名一起来显示函数名。`RTC_LOG_F` 和 `RTC_LOG_T_F` 宏都基于 `RTC_LOG` 宏实现。

`RTC_LOG_V` 宏与 `RTC_LOG` 宏基本相同，除了它的 severity 参数可以是一个变量外。`RTC_LOG_V` 宏与 `RTC_LOG` 宏都基于 `RTC_LOG_FILE_LINE` 宏实现。***感觉这里的三个宏可以省掉一个，让 `RTC_LOG` 宏基于 `RTC_LOG_V` 宏实现，如下面这样：***
```
#define RTC_LOG_V(sev)      \
  rtc::webrtc_logging_impl::LogCall() &         \
      rtc::webrtc_logging_impl::LogStreamer<>() \
          << rtc::webrtc_logging_impl::LogMetadata(__FILE__, __LINE__, sev)

#define RTC_LOG(sev) RTC_LOG_V(rtc::sev)
```

`RTC_LOG_FILE_LINE` 宏基于 `LogCall`、`LogStreamer` 和 `LogMetadata` 等类实现。

`RTC_LOG_ERRNO` 和 `RTC_LOG_ERRNO_EX` 宏还会在 log 消息中带上一些错误相关的信息。这两个宏基于 `RTC_LOG_E` 宏实现。与 `RTC_LOG_FILE_LINE` 宏类似，`RTC_LOG_E` 也是基于 `LogCall` 和 `LogStreamer` 等类实现。

来看 `RTC_LOG_FILE_LINE` 宏的实现：
```
#define RTC_LOG_FILE_LINE(sev, file, line)      \
  rtc::webrtc_logging_impl::LogCall() &         \
      rtc::webrtc_logging_impl::LogStreamer<>() \
          << rtc::webrtc_logging_impl::LogMetadata(file, line, sev)
```

首先看一下 `LogCall` 类的定义：
```
class LogCall final {
 public:
  // This can be any binary operator with precedence lower than <<.
  template <typename... Ts>
  RTC_FORCE_INLINE void operator&(const LogStreamer<Ts...>& streamer) {
    streamer.Call();
  }
};
```

`LogCall` 类只定义了一个成员函数，即 `operator&()`，这个成员函数接收一个 `LogStreamer` 参数。如 `operator&()` 成员函数的注释的说明，这里重载的操作符不要求一定是 `&`，只要是一个比 `<<` 操作符优先级低的操作符即可。

站在 `LogCall` 类的角度看，`RTC_LOG_FILE_LINE` 宏完成的工作是：1. 创建一个 `LogCall` 类的对象；2. 以动态的方式创建一个 `LogStreamer` 对象；3. 传入创建的 `LogStreamer` 对象，在创建的 `LogCall` 类对象上调用其成员函数。

***`RTC_LOG_FILE_LINE` 宏的意图是，以动态的方式创建一个 `LogStreamer` 对象，并以该对象为参数，调用某个函数，被调用的这个函数，把 `LogStreamer` 对象的 log 消息输出出去。***

把 `RTC_LOG_FILE_LINE` 宏换一种实现方式，来更清晰地看一下这个宏的实现。首先给 `LogCall` 类增加一个函数：
```
  template <typename... Ts>
  RTC_FORCE_INLINE void func(const LogStreamer<Ts...>& streamer) {
    streamer.Call();
  }
```

然后修改 `RTC_LOG_FILE_LINE` 宏的实现方式：
```
#define RTC_LOG_FILE_LINE_HAN(sev, file, line)      \
  rtc::webrtc_logging_impl::LogCall().func(         \
      rtc::webrtc_logging_impl::LogStreamer<>() \
          << rtc::webrtc_logging_impl::LogMetadata(file, line, sev)

#define RTC_LOG_HAN(sev) RTC_LOG_FILE_LINE_HAN(rtc::sev, __FILE__, __LINE__)

// The _V version is for when a variable is passed in.
#define RTC_LOG_V_HAN(sev) RTC_LOG_FILE_LINE_HAN(sev, __FILE__, __LINE__)
```

注意，对 `LogCall` 类的 `operator&()` 的调用被替换为了对成员函数 `func()` 的调用。

经过了这样的修改，使用 `RTC_LOG` 的代码也要做一些修改，如下：
```
  RTC_LOG(INFO) << __FUNCTION__;
  RTC_LOG_HAN(INFO) << __FUNCTION__ << " HAN" );
```

注意，相对于原来的 `RTC_LOG`，修改之后的 `RTC_LOG` 宏在使用时，需要在语句的最后添加一个右括号。使用时最后的右括号显得非常奇怪。从中我们也可以体会一下 WebRTC 用 `operator&()` 实现 `LogCall` 类的原因。

`LogStreamer` 模板类的定义如下：
```
// Ephemeral type that represents the result of the logging << operator.
template <typename... Ts>
class LogStreamer;

// Base case: Before the first << argument.
template <>
class LogStreamer<> final {
 public:
  template <typename U,
            typename std::enable_if<std::is_arithmetic<U>::value ||
                                    std::is_enum<U>::value>::type* = nullptr>
  RTC_FORCE_INLINE LogStreamer<decltype(MakeVal(std::declval<U>()))> operator<<(
      U arg) const {
    return LogStreamer<decltype(MakeVal(std::declval<U>()))>(MakeVal(arg),
                                                             this);
  }

  template <typename U,
            typename std::enable_if<!std::is_arithmetic<U>::value &&
                                    !std::is_enum<U>::value>::type* = nullptr>
  RTC_FORCE_INLINE LogStreamer<decltype(MakeVal(std::declval<U>()))> operator<<(
      const U& arg) const {
    return LogStreamer<decltype(MakeVal(std::declval<U>()))>(MakeVal(arg),
                                                             this);
  }

  template <typename... Us>
  RTC_FORCE_INLINE static void Call(const Us&... args) {
    static constexpr LogArgType t[] = {Us::Type()..., LogArgType::kEnd};
    Log(t, args.GetVal()...);
  }
};

// Inductive case: We've already seen at least one << argument. The most recent
// one had type `T`, and the earlier ones had types `Ts`.
template <typename T, typename... Ts>
class LogStreamer<T, Ts...> final {
 public:
  RTC_FORCE_INLINE LogStreamer(T arg, const LogStreamer<Ts...>* prior)
      : arg_(arg), prior_(prior) {}

  template <typename U,
            typename std::enable_if<std::is_arithmetic<U>::value ||
                                    std::is_enum<U>::value>::type* = nullptr>
  RTC_FORCE_INLINE LogStreamer<decltype(MakeVal(std::declval<U>())), T, Ts...>
  operator<<(U arg) const {
    return LogStreamer<decltype(MakeVal(std::declval<U>())), T, Ts...>(
        MakeVal(arg), this);
  }

  template <typename U,
            typename std::enable_if<!std::is_arithmetic<U>::value &&
                                    !std::is_enum<U>::value>::type* = nullptr>
  RTC_FORCE_INLINE LogStreamer<decltype(MakeVal(std::declval<U>())), T, Ts...>
  operator<<(const U& arg) const {
    return LogStreamer<decltype(MakeVal(std::declval<U>())), T, Ts...>(
        MakeVal(arg), this);
  }

  template <typename... Us>
  RTC_FORCE_INLINE void Call(const Us&... args) const {
    prior_->Call(arg_, args...);
  }

 private:
  // The most recent argument.
  T arg_;

  // Earlier arguments.
  const LogStreamer<Ts...>* prior_;
};
```

对 `LogStreamer` 模板类的 `operator<<()` 成员的连续调用，创建一个 `LogStreamer` 模板类对象的单链表，创建的每个新的链表节点会被插入链表的头部。也就是说，使用 `RTC_LOG` 宏时，先传入的参数所对应的 `LogStreamer` 对象会位于链表的后面。链表的节点的值的类型为 `Val`：
```
template <LogArgType N, typename T>
struct Val {
  static constexpr LogArgType Type() { return N; }
  T GetVal() const { return val; }
  T val;
};
```

在 `LogCall` 类的 `operator&()` 函数中调用 `LogStreamer` 对象的 `Call()` 函数，将调用最后创建的 `LogStreamer` 对象的 `Call()`，将触发一系列的对 `LogStreamer` 对象的 `Call()` 函数的调用，最终将调用 `LogStreamer<>` 的 `Call()` 函数。由模板类 `LogStreamer` 的 `Call()` 函数的实现，可以知道，`LogStreamer<>` 的 `Call()` 函数将以使用 `RTC_LOG` 宏时参数传入的先后顺序接收各个参数。

`LogStreamer<>` 的 `Call()` 函数提取各个参数的类型，构造类型数组，并将类型数组和各个参数的实际值传给 `Log()` 来输出 log。

`Log()` 函数的定义如下：
```
namespace webrtc_logging_impl {

void Log(const LogArgType* fmt, ...) {
  va_list args;
  va_start(args, fmt);

  LogMetadataErr meta;
  const char* tag = nullptr;
  switch (*fmt) {
    case LogArgType::kLogMetadata: {
      meta = {va_arg(args, LogMetadata), ERRCTX_NONE, 0};
      break;
    }
    case LogArgType::kLogMetadataErr: {
      meta = va_arg(args, LogMetadataErr);
      break;
    }
#ifdef WEBRTC_ANDROID
    case LogArgType::kLogMetadataTag: {
      const LogMetadataTag tag_meta = va_arg(args, LogMetadataTag);
      meta = {{nullptr, 0, tag_meta.severity}, ERRCTX_NONE, 0};
      tag = tag_meta.tag;
      break;
    }
#endif
    default: {
      RTC_NOTREACHED();
      va_end(args);
      return;
    }
  }
  LogMessage log_message(meta.meta.File(), meta.meta.Line(),
                         meta.meta.Severity(), meta.err_ctx, meta.err);
  if (tag) {
    log_message.AddTag(tag);
  }

  for (++fmt; *fmt != LogArgType::kEnd; ++fmt) {
    switch (*fmt) {
      case LogArgType::kInt:
        log_message.stream() << va_arg(args, int);
        break;
      case LogArgType::kLong:
        log_message.stream() << va_arg(args, long);
        break;
      case LogArgType::kLongLong:
        log_message.stream() << va_arg(args, long long);
        break;
      case LogArgType::kUInt:
        log_message.stream() << va_arg(args, unsigned);
        break;
      case LogArgType::kULong:
        log_message.stream() << va_arg(args, unsigned long);
        break;
      case LogArgType::kULongLong:
        log_message.stream() << va_arg(args, unsigned long long);
        break;
      case LogArgType::kDouble:
        log_message.stream() << va_arg(args, double);
        break;
      case LogArgType::kLongDouble:
        log_message.stream() << va_arg(args, long double);
        break;
      case LogArgType::kCharP:
        log_message.stream() << va_arg(args, const char*);
        break;
      case LogArgType::kStdString:
        log_message.stream() << *va_arg(args, const std::string*);
        break;
      case LogArgType::kVoidP:
        log_message.stream() << va_arg(args, const void*);
        break;
      default:
        RTC_NOTREACHED();
        va_end(args);
        return;
    }
  }

  va_end(args);
}

}  // namespace webrtc_logging_impl
```

`Log()` 函数构造 `LogMessage` 对象，并将各个要输出的 log 值送进 `LogMessage` 的 `rtc::StringBuilder`。

## WebRTC 实现的 `LogSink`

默认情况下，`LogMessage` 的 `streams_` 中没有注册任何 `LogSink`，然而 WebRTC 实际上还是提供了几个 `LogSink` 的实现的。具体而言，是在 `webrtc/src/rtc_base/logsinks.h` 中，有两个 `LogSink` 的实现，它们分别是类 `FileRotatingLogSink` 和 `CallSessionFileRotatingLogSink`。

`FileRotatingLogSink` 使用一个 `FileRotatingStream` 把日志写入一个支持 rotate 的文件。这个类的声明如下：
```
// Log sink that uses a FileRotatingStream to write to disk.
// Init() must be called before adding this sink.
class FileRotatingLogSink : public LogSink {
 public:
  // |num_log_files| must be greater than 1 and |max_log_size| must be greater
  // than 0.
  FileRotatingLogSink(const std::string& log_dir_path,
                      const std::string& log_prefix,
                      size_t max_log_size,
                      size_t num_log_files);
  ~FileRotatingLogSink() override;

  // Writes the message to the current file. It will spill over to the next
  // file if needed.
  void OnLogMessage(const std::string& message) override;
  void OnLogMessage(const std::string& message,
                    LoggingSeverity sev,
                    const char* tag) override;

  // Deletes any existing files in the directory and creates a new log file.
  virtual bool Init();

  // Disables buffering on the underlying stream.
  bool DisableBuffering();

 protected:
  explicit FileRotatingLogSink(FileRotatingStream* stream);

 private:
  std::unique_ptr<FileRotatingStream> stream_;

  RTC_DISALLOW_COPY_AND_ASSIGN(FileRotatingLogSink);
};
```

这个类是对 `FileRotatingStream` 的一个不是很复杂的封装，它的实现如下：
```
FileRotatingLogSink::FileRotatingLogSink(const std::string& log_dir_path,
                                         const std::string& log_prefix,
                                         size_t max_log_size,
                                         size_t num_log_files)
    : FileRotatingLogSink(new FileRotatingStream(log_dir_path,
                                                 log_prefix,
                                                 max_log_size,
                                                 num_log_files)) {}

FileRotatingLogSink::FileRotatingLogSink(FileRotatingStream* stream)
    : stream_(stream) {
  RTC_DCHECK(stream);
}

FileRotatingLogSink::~FileRotatingLogSink() {}

void FileRotatingLogSink::OnLogMessage(const std::string& message) {
  if (stream_->GetState() != SS_OPEN) {
    std::fprintf(stderr, "Init() must be called before adding this sink.\n");
    return;
  }
  stream_->WriteAll(message.c_str(), message.size(), nullptr, nullptr);
}

void FileRotatingLogSink::OnLogMessage(const std::string& message,
                                       LoggingSeverity sev,
                                       const char* tag) {
  if (stream_->GetState() != SS_OPEN) {
    std::fprintf(stderr, "Init() must be called before adding this sink.\n");
    return;
  }
  stream_->WriteAll(tag, strlen(tag), nullptr, nullptr);
  stream_->WriteAll(": ", 2, nullptr, nullptr);
  stream_->WriteAll(message.c_str(), message.size(), nullptr, nullptr);
}

bool FileRotatingLogSink::Init() {
  return stream_->Open();
}

bool FileRotatingLogSink::DisableBuffering() {
  return stream_->DisableBuffering();
}
```

`FileRotatingLogSink` 在收到日志消息时，简单地把日志消息写入 `FileRotatingStream`。

`CallSessionFileRotatingLogSink` 与 `FileRotatingLogSink` 类似，仅有的区别是，它使用的是 `CallSessionFileRotatingStream`。这个类的声明如下：
```
// Log sink that uses a CallSessionFileRotatingStream to write to disk.
// Init() must be called before adding this sink.
class CallSessionFileRotatingLogSink : public FileRotatingLogSink {
 public:
  CallSessionFileRotatingLogSink(const std::string& log_dir_path,
                                 size_t max_total_log_size);
  ~CallSessionFileRotatingLogSink() override;

 private:
  RTC_DISALLOW_COPY_AND_ASSIGN(CallSessionFileRotatingLogSink);
};
```

`CallSessionFileRotatingLogSink` 的实现如下：
```
CallSessionFileRotatingLogSink::CallSessionFileRotatingLogSink(
    const std::string& log_dir_path,
    size_t max_total_log_size)
    : FileRotatingLogSink(
          new CallSessionFileRotatingStream(log_dir_path, max_total_log_size)) {
}

CallSessionFileRotatingLogSink::~CallSessionFileRotatingLogSink() {}
```

## . . .

WebRTC 的 rtc_base 的 log 系统实现，还有几点看得比较晕的地方：
 - `LogStreamer` 模板类的实现。`LogStreamer` 模板类的实现用了一些模板元编程的技巧，`std::enable_if`、`std::decay` 等 STL 的组件的使用让人觉得有点不太容易懂。
 - 函数可变参数列表的实现原理。可变参数列表的使用不独于此，其原理是需要明白的比较重要的一个 C/C++ 开发的技术点。

关于 WebRTC 的 rtc_base log 系统实现：
 - log 系统除了要功能强大，性能优异，还需要提供一个非常好用的用户接口。易用的用户接口的重要性，从这个 log 系统复杂的 `LogStreamer` 模板类实现就可见一斑。
 - WebRTC 的 rtc_base log 系统实现在向 `LogSink` 输出 log 时，使用了锁做同步，log 输出非常频繁时，锁争抢可能会成为一个问题。
 - 输出 log 一般都需要执行 IO 操作，不过 WebRTC 的 rtc_base log 系统实现没有把 log 输出异步化，输出 log 都是同步操作，即使专门实现的 `LogSink` `FileRotatingLogSink` 和 `CallSessionFileRotatingLogSink` 也是。

Done.
