---
title: WebRTC Audio Encoder/Decoder Factory 的实现
date: 2020-08-31 21:55:49
categories:
- C/C++开发
tags:
- C/C++开发
---

Audio encoder factory 用于创建完成各种 audio codec 编码的 encoder 对象，audio decoder factory 则用于创建完成各种 audio codec 解码的 decoder 对象。
<!--more-->
WebRTC 的 Audio Encoder Factory 接口的定义（位于 `webrtc/src/api/audio_codecs/audio_encoder_factory.h`）如下：
```
namespace webrtc {

// A factory that creates AudioEncoders.
class AudioEncoderFactory : public rtc::RefCountInterface {
 public:
  // Returns a prioritized list of audio codecs, to use for signaling etc.
  virtual std::vector<AudioCodecSpec> GetSupportedEncoders() = 0;

  // Returns information about how this format would be encoded, provided it's
  // supported. More format and format variations may be supported than those
  // returned by GetSupportedEncoders().
  virtual absl::optional<AudioCodecInfo> QueryAudioEncoder(
      const SdpAudioFormat& format) = 0;

  // Creates an AudioEncoder for the specified format. The encoder will tags its
  // payloads with the specified payload type. The `codec_pair_id` argument is
  // used to link encoders and decoders that talk to the same remote entity: if
  // a AudioEncoderFactory::MakeAudioEncoder() and a
  // AudioDecoderFactory::MakeAudioDecoder() call receive non-null IDs that
  // compare equal, the factory implementations may assume that the encoder and
  // decoder form a pair. (The intended use case for this is to set up
  // communication between the AudioEncoder and AudioDecoder instances, which is
  // needed for some codecs with built-in bandwidth adaptation.)
  //
  // Note: Implementations need to be robust against combinations other than
  // one encoder, one decoder getting the same ID; such encoders must still
  // work.
  //
  // TODO(ossu): Try to avoid audio encoders having to know their payload type.
  virtual std::unique_ptr<AudioEncoder> MakeAudioEncoder(
      int payload_type,
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) = 0;
};

}  // namespace webrtc
```

直觉上，`AudioEncoderFactory` 应该使用类似于组合模式的方式实现：
 - 首先为每个 audio codec 实现一个 `AudioEncoderFactory` 的子类实现，比如定义一个 `AudioEncoderFactory` 的子类实现用于创建 OPUS 的 encoder，定义一个 `AudioEncoderFactory` 的子类实现用于创建 AAC 的 encoder 等；
 - 然后创建一个组合的 `AudioEncoderFactory` 子类实现，作为所有要支持的 audio codec 的 encoder factory 的容器，并借助于各个 audio codec 的 encoder factory 实现来实现其接口功能；
 - 最后创建一个 `AudioEncoderFactory` 的工厂方法，该方法中创建组合的 `AudioEncoderFactory` 子类对象，且创建每个要支持的 audio codec 的 encoder factory 对象并注册给组合的 `AudioEncoderFactory` 子类对象，然后返回组合的 `AudioEncoderFactory` 子类对象。

一个可能的实现如下。首先是 audio encoder factory 的工厂方法声明：
```
#ifndef API_AUDIO_CODECS_FAKE_AUDIO_ENCODER_FACTORY_H_
#define API_AUDIO_CODECS_FAKE_AUDIO_ENCODER_FACTORY_H_

#include "api/audio_codecs/audio_encoder_factory.h"
#include "rtc_base/scoped_ref_ptr.h"

namespace webrtc {

class CodecAudioEncoderFactory: public AudioEncoderFactory {
public:
  virtual bool IsSupported(const SdpAudioFormat &format) = 0;
};

// Creates a new factory that can create the built-in types of audio encoders.
// NOTE: This function is still under development and may change without notice.
rtc::scoped_refptr<AudioEncoderFactory> CreateBuiltinAudioEncoderFactory();

}  // namespace webrtc

#endif /* API_AUDIO_CODECS_FAKE_AUDIO_ENCODER_FACTORY_H_ */
```

除了 audio encoder factory 的工厂方法声明外，还创建了一个新的 `AudioEncoderFactory` 子类，用于描述具体的 audio codec 的 encoder factory 的接口 `CodecAudioEncoderFactory`， 并为该接口类添加了一个接口函数 `IsSupported()`，用于判断一个 encoder factory 对于特定 SdpFormat 的支持情况，以辅助组合的 `AudioEncoderFactory` 子类对象的实现。

然后是相关几个类的实现：
```
#include "fake_audio_encoder_factory.h"
#include <vector>
#include "rtc_base/refcountedobject.h"

namespace webrtc {

class OpusEncoderFactory : public CodecAudioEncoderFactory {
public:
  std::vector<AudioCodecSpec> GetSupportedEncoders() override {
    std::vector<AudioCodecSpec> specs;

    return specs;
  }

  absl::optional<AudioCodecInfo> QueryAudioEncoder(const SdpAudioFormat &format)
      override {
    return absl::nullopt;
  }

  std::unique_ptr<AudioEncoder> MakeAudioEncoder(int payload_type,
      const SdpAudioFormat &format,
      absl::optional<AudioCodecPairId> codec_pair_id) override {

    return nullptr;
  }

  bool IsSupported(const SdpAudioFormat &format) override {
    return true;
  }
};

class FakeAudioEncoderFactory: public AudioEncoderFactory {
public:
  std::vector<AudioCodecSpec> GetSupportedEncoders() override {
    std::vector<AudioCodecSpec> specs;

    for (auto &factory : audio_encoder_factories) {
      specs.insert(specs.end(), factory->GetSupportedEncoders().begin(), factory->GetSupportedEncoders().end());
    }

    return specs;
  }

  absl::optional<AudioCodecInfo> QueryAudioEncoder(const SdpAudioFormat &format)
      override {
    for (auto &factory : audio_encoder_factories) {
      if (factory->IsSupported(format)) {
        return factory->QueryAudioEncoder(format);
      }
    }

    return absl::nullopt;
  }

  std::unique_ptr<AudioEncoder> MakeAudioEncoder(int payload_type,
      const SdpAudioFormat &format,
      absl::optional<AudioCodecPairId> codec_pair_id) override {
    for (auto &factory : audio_encoder_factories) {
      if (factory->IsSupported(format)) {
        return factory->MakeAudioEncoder(payload_type, format, codec_pair_id);
      }
    }

    return nullptr;
  }

  void AddAudioEncoderFactory(rtc::scoped_refptr<CodecAudioEncoderFactory> factory) {
    audio_encoder_factories.push_back(factory);
  }

private:
  std::vector<rtc::scoped_refptr<CodecAudioEncoderFactory>> audio_encoder_factories;
};

rtc::scoped_refptr<AudioEncoderFactory> CreateBuiltinAudioEncoderFactory() {
  rtc::scoped_refptr<FakeAudioEncoderFactory> factory(new rtc::RefCountedObject<FakeAudioEncoderFactory>);

  rtc::scoped_refptr<OpusEncoderFactory> opus_factory(new rtc::RefCountedObject<OpusEncoderFactory>);
  factory->AddAudioEncoderFactory(opus_factory);

  return factory;
}
```

然而，WebRTC 的 builtin audio encoder factory 没有以类似的这种方式实现。

在 `webrtc/src/api/audio_codecs/builtin_audio_encoder_factory.h` 文件中声明了用于创建 `AudioEncoderFactory` 的工厂方法：
```
namespace webrtc {

// Creates a new factory that can create the built-in types of audio encoders.
// NOTE: This function is still under development and may change without notice.
rtc::scoped_refptr<AudioEncoderFactory> CreateBuiltinAudioEncoderFactory();

}  // namespace webrtc
```

在 `webrtc/src/api/audio_codecs/builtin_audio_encoder_factory.cc` 文件中，`CreateBuiltinAudioEncoderFactory()` 函数的实现如下：
```
namespace webrtc {

namespace {

// Modify an audio encoder to not advertise support for anything.
template <typename T>
struct NotAdvertised {
  using Config = typename T::Config;
  static absl::optional<Config> SdpToConfig(
      const SdpAudioFormat& audio_format) {
    return T::SdpToConfig(audio_format);
  }
  static void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs) {
    // Don't advertise support for anything.
  }
  static AudioCodecInfo QueryAudioEncoder(const Config& config) {
    return T::QueryAudioEncoder(config);
  }
  static std::unique_ptr<AudioEncoder> MakeAudioEncoder(
      const Config& config,
      int payload_type,
      absl::optional<AudioCodecPairId> codec_pair_id = absl::nullopt) {
    return T::MakeAudioEncoder(config, payload_type, codec_pair_id);
  }
};

}  // namespace

rtc::scoped_refptr<AudioEncoderFactory> CreateBuiltinAudioEncoderFactory() {
  return CreateAudioEncoderFactory<

#if WEBRTC_USE_BUILTIN_OPUS
      AudioEncoderOpus,
#endif

      AudioEncoderIsac, AudioEncoderG722,

#if WEBRTC_USE_BUILTIN_ILBC
      AudioEncoderIlbc,
#endif

      AudioEncoderG711, NotAdvertised<AudioEncoderL16>>();
}

}  // namespace webrtc
```

`CreateBuiltinAudioEncoderFactory()` 函数的实现中规中矩，它调用了一个模板函数 `CreateAudioEncoderFactory()` 创建 audio encoder factory，多个带有 `Encoder` 字眼的 class 作为模板函数的类型参数。

模板函数 `CreateAudioEncoderFactory()` 的定义位于 `webrtc/src/api/audio_codecs/audio_encoder_factory_template.h` 文件中：
```
namespace webrtc {

namespace audio_encoder_factory_template_impl {

template <typename... Ts>
struct Helper;

// Base case: 0 template parameters.
template <>
struct Helper<> {
  static void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs) {}
  static absl::optional<AudioCodecInfo> QueryAudioEncoder(
      const SdpAudioFormat& format) {
    return absl::nullopt;
  }
  static std::unique_ptr<AudioEncoder> MakeAudioEncoder(
      int payload_type,
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) {
    return nullptr;
  }
};

// Inductive case: Called with n + 1 template parameters; calls subroutines
// with n template parameters.
template <typename T, typename... Ts>
struct Helper<T, Ts...> {
  static void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs) {
    T::AppendSupportedEncoders(specs);
    Helper<Ts...>::AppendSupportedEncoders(specs);
  }
  static absl::optional<AudioCodecInfo> QueryAudioEncoder(
      const SdpAudioFormat& format) {
    auto opt_config = T::SdpToConfig(format);
    static_assert(std::is_same<decltype(opt_config),
                               absl::optional<typename T::Config>>::value,
                  "T::SdpToConfig() must return a value of type "
                  "absl::optional<T::Config>");
    return opt_config ? absl::optional<AudioCodecInfo>(
                            T::QueryAudioEncoder(*opt_config))
                      : Helper<Ts...>::QueryAudioEncoder(format);
  }
  static std::unique_ptr<AudioEncoder> MakeAudioEncoder(
      int payload_type,
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) {
    auto opt_config = T::SdpToConfig(format);
    if (opt_config) {
      return T::MakeAudioEncoder(*opt_config, payload_type, codec_pair_id);
    } else {
      return Helper<Ts...>::MakeAudioEncoder(payload_type, format,
                                             codec_pair_id);
    }
  }
};

template <typename... Ts>
class AudioEncoderFactoryT : public AudioEncoderFactory {
 public:
  std::vector<AudioCodecSpec> GetSupportedEncoders() override {
    std::vector<AudioCodecSpec> specs;
    Helper<Ts...>::AppendSupportedEncoders(&specs);
    return specs;
  }

  absl::optional<AudioCodecInfo> QueryAudioEncoder(
      const SdpAudioFormat& format) override {
    return Helper<Ts...>::QueryAudioEncoder(format);
  }

  std::unique_ptr<AudioEncoder> MakeAudioEncoder(
      int payload_type,
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) override {
    return Helper<Ts...>::MakeAudioEncoder(payload_type, format, codec_pair_id);
  }
};

}  // namespace audio_encoder_factory_template_impl

// Make an AudioEncoderFactory that can create instances of the given encoders.
//
// Each encoder type is given as a template argument to the function; it should
// be a struct with the following static member functions:
//
//   // Converts |audio_format| to a ConfigType instance. Returns an empty
//   // optional if |audio_format| doesn't correctly specify an encoder of our
//   // type.
//   absl::optional<ConfigType> SdpToConfig(const SdpAudioFormat& audio_format);
//
//   // Appends zero or more AudioCodecSpecs to the list that will be returned
//   // by AudioEncoderFactory::GetSupportedEncoders().
//   void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs);
//
//   // Returns information about how this format would be encoded. Used to
//   // implement AudioEncoderFactory::QueryAudioEncoder().
//   AudioCodecInfo QueryAudioEncoder(const ConfigType& config);
//
//   // Creates an AudioEncoder for the specified format. Used to implement
//   // AudioEncoderFactory::MakeAudioEncoder().
//   std::unique_ptr<AudioDecoder> MakeAudioEncoder(
//       const ConfigType& config,
//       int payload_type,
//       absl::optional<AudioCodecPairId> codec_pair_id);
//
// ConfigType should be a type that encapsulates all the settings needed to
// create an AudioEncoder. T::Config (where T is the encoder struct) should
// either be the config type, or an alias for it.
//
// Whenever it tries to do something, the new factory will try each of the
// encoders in the order they were specified in the template argument list,
// stopping at the first one that claims to be able to do the job.
//
// NOTE: This function is still under development and may change without notice.
//
// TODO(kwiberg): Point at CreateBuiltinAudioEncoderFactory() for an example of
// how it is used.
template <typename... Ts>
rtc::scoped_refptr<AudioEncoderFactory> CreateAudioEncoderFactory() {
  // There's no technical reason we couldn't allow zero template parameters,
  // but such a factory couldn't create any encoders, and callers can do this
  // by mistake by simply forgetting the <> altogether. So we forbid it in
  // order to prevent caller foot-shooting.
  static_assert(sizeof...(Ts) >= 1,
                "Caller must give at least one template parameter");

  return rtc::scoped_refptr<AudioEncoderFactory>(
      new rtc::RefCountedObject<
          audio_encoder_factory_template_impl::AudioEncoderFactoryT<Ts...>>());
}

}  // namespace webrtc
```

对照我们前面实现的 `FakeAudioEncoderFactory` 来理解 WebRTC 的实现：
 - Audio codec encoder 的 factory 的列表不是一个动态的列表，而是借助于模板机制构建的静态的列表；
 - 模板类 `Helper` 充当了 audio codec encoder 的 factory 列表的遍历者和访问者的角色；
3. audio codec encoder factory 的接口没有复用 `AudioEncoderFactory`，而是隐式地定义了另一个接口，如 `CreateAudioEncoderFactory()` 模板函数的注释中的说明，这个接口包含如下几个成员函数：
```
  absl::optional<ConfigType> SdpToConfig(const SdpAudioFormat& audio_format);
  void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs);
  AudioCodecInfo QueryAudioEncoder(const ConfigType& config);
  std::unique_ptr<AudioDecoder> MakeAudioEncoder(
      const ConfigType& config,
      int payload_type,
      absl::optional<AudioCodecPairId> codec_pair_id);
```
4. `AudioEncoderFactory` 接口的最终实现者为 `AudioEncoderFactoryT`，它的各个接口实现主要借助于模板类 `Helper` 完成。
5. 我们前面为 `CodecAudioEncoderFactory` 接口添加的 `IsSupported()` 大体等价于 WebRTC 的 `SdpToConfig()`。

可以具体看一个 ***Encoder*** 的实现，如 `AudioEncoderOpus` 的声明如下 (位于 `webrtc/src/api/audio_codecs/opus/audio_encoder_opus.h`)：
```
namespace webrtc {

// Opus encoder API for use as a template parameter to
// CreateAudioEncoderFactory<...>().
//
// NOTE: This struct is still under development and may change without notice.
struct AudioEncoderOpus {
  using Config = AudioEncoderOpusConfig;
  static absl::optional<AudioEncoderOpusConfig> SdpToConfig(
      const SdpAudioFormat& audio_format);
  static void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs);
  static AudioCodecInfo QueryAudioEncoder(const AudioEncoderOpusConfig& config);
  static std::unique_ptr<AudioEncoder> MakeAudioEncoder(
      const AudioEncoderOpusConfig& config,
      int payload_type,
      absl::optional<AudioCodecPairId> codec_pair_id = absl::nullopt);
};

}  // namespace webrtc
```

`AudioEncoderOpus` 的实现如下 (位于 `webrtc/src/api/audio_codecs/opus/audio_encoder_opus.cc`)：
```
namespace webrtc {

absl::optional<AudioEncoderOpusConfig> AudioEncoderOpus::SdpToConfig(
    const SdpAudioFormat& format) {
  return AudioEncoderOpusImpl::SdpToConfig(format);
}

void AudioEncoderOpus::AppendSupportedEncoders(
    std::vector<AudioCodecSpec>* specs) {
  AudioEncoderOpusImpl::AppendSupportedEncoders(specs);
}

AudioCodecInfo AudioEncoderOpus::QueryAudioEncoder(
    const AudioEncoderOpusConfig& config) {
  return AudioEncoderOpusImpl::QueryAudioEncoder(config);
}

std::unique_ptr<AudioEncoder> AudioEncoderOpus::MakeAudioEncoder(
    const AudioEncoderOpusConfig& config,
    int payload_type,
    absl::optional<AudioCodecPairId> /*codec_pair_id*/) {
  return AudioEncoderOpusImpl::MakeAudioEncoder(config, payload_type);
}

}  // namespace webrtc
```

尽管 `AudioEncoderOpus` 名字中只有 `encoder` 字眼，没有 `factory` 字眼，但它却是个货真价实的 encoder factory。

WebRTC 的 decoder factory 的实现与其 encoder factory 的实现非常相似。

在 `webrtc/src/api/audio_codecs/audio_decoder_factory.h` 文件中定义了 `AudioDecoderFactory` 接口：
```
namespace webrtc {

// A factory that creates AudioDecoders.
// NOTE: This class is still under development and may change without notice.
class AudioDecoderFactory : public rtc::RefCountInterface {
 public:
  virtual std::vector<AudioCodecSpec> GetSupportedDecoders() = 0;

  virtual bool IsSupportedDecoder(const SdpAudioFormat& format) = 0;

  // Create a new decoder instance. The `codec_pair_id` argument is used to
  // link encoders and decoders that talk to the same remote entity; if a
  // MakeAudioEncoder() and a MakeAudioDecoder() call receive non-null IDs that
  // compare equal, the factory implementations may assume that the encoder and
  // decoder form a pair.
  //
  // Note: Implementations need to be robust against combinations other than
  // one encoder, one decoder getting the same ID; such decoders must still
  // work.
  virtual std::unique_ptr<AudioDecoder> MakeAudioDecoder(
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) = 0;
};

}  // namespace webrtc
```

文件 `webrtc/src/api/audio_codecs/builtin_audio_decoder_factory.h` 中声明了 `CreateBuiltinAudioDecoderFactory()` 工厂方法：
```
namespace webrtc {

// Creates a new factory that can create the built-in types of audio decoders.
// NOTE: This function is still under development and may change without notice.
rtc::scoped_refptr<AudioDecoderFactory> CreateBuiltinAudioDecoderFactory();

}  // namespace webrtc
```

文件 `webrtc/src/api/audio_codecs/builtin_audio_decoder_factory.cc` 中 `CreateBuiltinAudioDecoderFactory()` 工厂方法的定义如下：
```
namespace webrtc {

namespace {

// Modify an audio decoder to not advertise support for anything.
template <typename T>
struct NotAdvertised {
  using Config = typename T::Config;
  static absl::optional<Config> SdpToConfig(
      const SdpAudioFormat& audio_format) {
    return T::SdpToConfig(audio_format);
  }
  static void AppendSupportedDecoders(std::vector<AudioCodecSpec>* specs) {
    // Don't advertise support for anything.
  }
  static std::unique_ptr<AudioDecoder> MakeAudioDecoder(
      const Config& config,
      absl::optional<AudioCodecPairId> codec_pair_id = absl::nullopt) {
    return T::MakeAudioDecoder(config, codec_pair_id);
  }
};

}  // namespace

rtc::scoped_refptr<AudioDecoderFactory> CreateBuiltinAudioDecoderFactory() {
  return CreateAudioDecoderFactory<

#if WEBRTC_USE_BUILTIN_OPUS
      AudioDecoderOpus,
#endif

      AudioDecoderIsac, AudioDecoderG722,

#if WEBRTC_USE_BUILTIN_ILBC
      AudioDecoderIlbc,
#endif

      AudioDecoderG711, NotAdvertised<AudioDecoderL16>>();
}

}  // namespace webrtc
```

其中的 `CreateAudioDecoderFactory()` 模板函数的定义位于 `webrtc/src/api/audio_codecs/audio_decoder_factory_template.h`：
```
namespace webrtc {

namespace audio_decoder_factory_template_impl {

template <typename... Ts>
struct Helper;

// Base case: 0 template parameters.
template <>
struct Helper<> {
  static void AppendSupportedDecoders(std::vector<AudioCodecSpec>* specs) {}
  static bool IsSupportedDecoder(const SdpAudioFormat& format) { return false; }
  static std::unique_ptr<AudioDecoder> MakeAudioDecoder(
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) {
    return nullptr;
  }
};

// Inductive case: Called with n + 1 template parameters; calls subroutines
// with n template parameters.
template <typename T, typename... Ts>
struct Helper<T, Ts...> {
  static void AppendSupportedDecoders(std::vector<AudioCodecSpec>* specs) {
    T::AppendSupportedDecoders(specs);
    Helper<Ts...>::AppendSupportedDecoders(specs);
  }
  static bool IsSupportedDecoder(const SdpAudioFormat& format) {
    auto opt_config = T::SdpToConfig(format);
    static_assert(std::is_same<decltype(opt_config),
                               absl::optional<typename T::Config>>::value,
                  "T::SdpToConfig() must return a value of type "
                  "absl::optional<T::Config>");
    return opt_config ? true : Helper<Ts...>::IsSupportedDecoder(format);
  }
  static std::unique_ptr<AudioDecoder> MakeAudioDecoder(
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) {
    auto opt_config = T::SdpToConfig(format);
    return opt_config ? T::MakeAudioDecoder(*opt_config, codec_pair_id)
                      : Helper<Ts...>::MakeAudioDecoder(format, codec_pair_id);
  }
};

template <typename... Ts>
class AudioDecoderFactoryT : public AudioDecoderFactory {
 public:
  std::vector<AudioCodecSpec> GetSupportedDecoders() override {
    std::vector<AudioCodecSpec> specs;
    Helper<Ts...>::AppendSupportedDecoders(&specs);
    return specs;
  }

  bool IsSupportedDecoder(const SdpAudioFormat& format) override {
    return Helper<Ts...>::IsSupportedDecoder(format);
  }

  std::unique_ptr<AudioDecoder> MakeAudioDecoder(
      const SdpAudioFormat& format,
      absl::optional<AudioCodecPairId> codec_pair_id) override {
    return Helper<Ts...>::MakeAudioDecoder(format, codec_pair_id);
  }
};

}  // namespace audio_decoder_factory_template_impl

// Make an AudioDecoderFactory that can create instances of the given decoders.
//
// Each decoder type is given as a template argument to the function; it should
// be a struct with the following static member functions:
//
//   // Converts |audio_format| to a ConfigType instance. Returns an empty
//   // optional if |audio_format| doesn't correctly specify an decoder of our
//   // type.
//   absl::optional<ConfigType> SdpToConfig(const SdpAudioFormat& audio_format);
//
//   // Appends zero or more AudioCodecSpecs to the list that will be returned
//   // by AudioDecoderFactory::GetSupportedDecoders().
//   void AppendSupportedDecoders(std::vector<AudioCodecSpec>* specs);
//
//   // Creates an AudioDecoder for the specified format. Used to implement
//   // AudioDecoderFactory::MakeAudioDecoder().
//   std::unique_ptr<AudioDecoder> MakeAudioDecoder(
//       const ConfigType& config,
//       absl::optional<AudioCodecPairId> codec_pair_id);
//
// ConfigType should be a type that encapsulates all the settings needed to
// create an AudioDecoder. T::Config (where T is the decoder struct) should
// either be the config type, or an alias for it.
//
// Whenever it tries to do something, the new factory will try each of the
// decoder types in the order they were specified in the template argument
// list, stopping at the first one that claims to be able to do the job.
//
// NOTE: This function is still under development and may change without notice.
//
// TODO(kwiberg): Point at CreateBuiltinAudioDecoderFactory() for an example of
// how it is used.
template <typename... Ts>
rtc::scoped_refptr<AudioDecoderFactory> CreateAudioDecoderFactory() {
  // There's no technical reason we couldn't allow zero template parameters,
  // but such a factory couldn't create any decoders, and callers can do this
  // by mistake by simply forgetting the <> altogether. So we forbid it in
  // order to prevent caller foot-shooting.
  static_assert(sizeof...(Ts) >= 1,
                "Caller must give at least one template parameter");

  return rtc::scoped_refptr<AudioDecoderFactory>(
      new rtc::RefCountedObject<
          audio_decoder_factory_template_impl::AudioDecoderFactoryT<Ts...>>());
}

}  // namespace webrtc
```

一个 audio codec decoder factory 的实现 `AudioDecoderOpus` 声明位于 `webrtc/src/api/audio_codecs/opus/audio_decoder_opus.h`：
```
namespace webrtc {

// Opus decoder API for use as a template parameter to
// CreateAudioDecoderFactory<...>().
//
// NOTE: This struct is still under development and may change without notice.
struct AudioDecoderOpus {
  struct Config {
    int num_channels;
  };
  static absl::optional<Config> SdpToConfig(const SdpAudioFormat& audio_format);
  static void AppendSupportedDecoders(std::vector<AudioCodecSpec>* specs);
  static std::unique_ptr<AudioDecoder> MakeAudioDecoder(
      Config config,
      absl::optional<AudioCodecPairId> codec_pair_id = absl::nullopt);
};

}  // namespace webrtc
```

`AudioDecoderOpus` 的定义位于 `webrtc/src/api/audio_codecs/opus/audio_decoder_opus.cc`：
```
namespace webrtc {

absl::optional<AudioDecoderOpus::Config> AudioDecoderOpus::SdpToConfig(
    const SdpAudioFormat& format) {
  const auto num_channels = [&]() -> absl::optional<int> {
    auto stereo = format.parameters.find("stereo");
    if (stereo != format.parameters.end()) {
      if (stereo->second == "0") {
        return 1;
      } else if (stereo->second == "1") {
        return 2;
      } else {
        return absl::nullopt;  // Bad stereo parameter.
      }
    }
    return 1;  // Default to mono.
  }();

  if (STR_CASE_CMP(format.name.c_str(), "opus") == 0 &&
      format.clockrate_hz == 16000 && format.num_channels == 1 &&
      num_channels) {
    return Config{static_cast<int>(format.num_channels)};
  } else if (STR_CASE_CMP(format.name.c_str(), "opusswb") == 0 &&
      format.clockrate_hz == 32000 && format.num_channels == 1 &&
      num_channels) {
    return Config{static_cast<int>(format.num_channels)};
  } else if (STR_CASE_CMP(format.name.c_str(), "opusfb") == 0 &&
      format.clockrate_hz == 48000 && format.num_channels == 2 &&
      num_channels) {
    return Config{static_cast<int>(format.num_channels)};
  } else if (STR_CASE_CMP(format.name.c_str(), "opusfb") == 0 &&
      format.clockrate_hz == 48000 && format.num_channels == 1 &&
      num_channels) {
    return Config{static_cast<int>(format.num_channels)};
  } else {
    return absl::nullopt;
  }
}

void AudioDecoderOpus::AppendSupportedDecoders(
    std::vector<AudioCodecSpec>* specs) {
  AudioCodecInfo opus_info{48000, 1, 64000, 6000, 510000};
  opus_info.allow_comfort_noise = false;
  opus_info.supports_network_adaption = true;
  SdpAudioFormat opus_format(
      {"opus", 48000, 2, {{"minptime", "10"}, {"useinbandfec", "1"}}});
  specs->push_back({std::move(opus_format), std::move(opus_info)});
}

std::unique_ptr<AudioDecoder> AudioDecoderOpus::MakeAudioDecoder(
    Config config,
    absl::optional<AudioCodecPairId> /*codec_pair_id*/) {
  return absl::make_unique<AudioDecoderOpusImpl>(config.num_channels);
}

}  // namespace webrtc
```

WebRTC builtin audio decoder factory 和 builtin audio encoder factory 的实现套路几乎完全一样，此处不再赘述。

Done.
