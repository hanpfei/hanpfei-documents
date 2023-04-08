我们在 `AudioPolicyManager::onNewAudioModulesAvailableInt(DeviceVector *newDevices)` 函数中看到它创建了 `SwAudioOutputDescriptor` 对象，后者的构造函数的定义 (位于 `frameworks/av/services/audiopolicy/common/managerdefinitions/src/AudioOutputDescriptor.cpp`) 如下：
```
AudioOutputDescriptor::AudioOutputDescriptor(const sp<PolicyAudioPort>& policyAudioPort,
                                             AudioPolicyClientInterface *clientInterface)
    : mPolicyAudioPort(policyAudioPort), mClientInterface(clientInterface)
{
    if (mPolicyAudioPort.get() != nullptr) {
        mPolicyAudioPort->pickAudioProfile(mSamplingRate, mChannelMask, mFormat);
        if (mPolicyAudioPort->asAudioPort()->getGains().size() > 0) {
            mPolicyAudioPort->asAudioPort()->getGains()[0]->getDefaultConfig(&mGain);
        }
    }
}
 . . . . . .
SwAudioOutputDescriptor::SwAudioOutputDescriptor(const sp<IOProfile>& profile,
                                                 AudioPolicyClientInterface *clientInterface)
    : AudioOutputDescriptor(profile, clientInterface),
    mProfile(profile), mIoHandle(AUDIO_IO_HANDLE_NONE), mLatency(0),
    mFlags((audio_output_flags_t)0),
    mOutput1(0), mOutput2(0), mDirectOpenCount(0),
    mDirectClientSession(AUDIO_SESSION_NONE)
{
    if (profile != NULL) {
        mFlags = (audio_output_flags_t)profile->getFlags();
    }
}
```

Android 中打开音频输出流，所需的主要参数采样率、通道掩码（通道数）、采样格式、增益和标记来自于传入的 `IOProfile`。如 [Android 音频设备信息加载](https://www.jianshu.com/p/19a7fff9195d) 一文中的说明，`IOProfile` 的信息主要来自于解析 `audio_policy_configuration.xml` 音频策略配置文件，更具地说，来自于音频策略配置文件中的 ***mixPort*** 元素，如下面 (位于 `device/generic/car/emulator/audio/audio_policy_configuration.xml`) 这样：
```
            <defaultOutputDevice>bus0_media_out</defaultOutputDevice>
            <mixPorts>
                <mixPort name="mixport_bus0_media_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
 . . . . . .
                <mixPort name="mixport_bus200_audio_zone_2" role="source">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="primary input" role="sink">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </mixPort>
```

`IOProfile` 通过 `AudioRoute` 与 `DeviceDescriptor` 建立连接。在音频策略配置文件中，`AudioRoute` 和 `DeviceDescriptor` 对应的 XML 元素定义如下面这样：
```
            <devicePorts>
                <devicePort tagName="bus0_media_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus0_media_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
 . . . . . .
            <routes>
                <route type="mix" sink="bus0_media_out" sources="mixport_bus0_media_out"/>
```

`SwAudioOutputDescriptor` 的标记和增益直接来自于音频策略配置文件的 ***mixPort*** 元素。音频策略配置文件支持为 ***mixPort*** 定义增益，也支持为 ***devicePort*** 定义增益，上面的这个配置文件没有为 ***mixPort*** 定义增益。采样率、通道掩码（通道数）和采样格式则会从多个 `AudioProfile` 中选择最佳的一个，具体的策略如 `PolicyAudioPort::pickAudioProfile()` 函数的定义(位于 `frameworks/av/services/audiopolicy/common/managerdefinitions/src/PolicyAudioPort.cpp`)：
```
void PolicyAudioPort::pickSamplingRate(uint32_t &pickedRate,
                                       const SampleRateSet &samplingRates) const
{
    pickedRate = 0;
    // For direct outputs, pick minimum sampling rate: this helps ensuring that the
    // channel count / sampling rate combination chosen will be supported by the connected
    // sink
    if (isDirectOutput()) {
        uint32_t samplingRate = UINT_MAX;
        for (const auto rate : samplingRates) {
            if ((rate < samplingRate) && (rate > 0)) {
                samplingRate = rate;
            }
        }
        pickedRate = (samplingRate == UINT_MAX) ? 0 : samplingRate;
    } else {
        uint32_t maxRate = SAMPLE_RATE_HZ_MAX;

        // For mixed output and inputs, use max mixer sampling rates. Do not
        // limit sampling rate otherwise
        // For inputs, also see checkCompatibleSamplingRate().
        if (asAudioPort()->getType() == AUDIO_PORT_TYPE_MIX) {
            maxRate = UINT_MAX;
        }
        // TODO: should mSamplingRates[] be ordered in terms of our preference
        // and we return the first (and hence most preferred) match?  This is of concern if
        // we want to choose 96kHz over 192kHz for USB driver stability or resource constraints.
        for (const auto rate : samplingRates) {
            if ((rate > pickedRate) && (rate <= maxRate)) {
                pickedRate = rate;
            }
        }
    }
}

void PolicyAudioPort::pickChannelMask(audio_channel_mask_t &pickedChannelMask,
                                      const ChannelMaskSet &channelMasks) const
{
    pickedChannelMask = AUDIO_CHANNEL_NONE;
    // For direct outputs, pick minimum channel count: this helps ensuring that the
    // channel count / sampling rate combination chosen will be supported by the connected
    // sink
    if (isDirectOutput()) {
        uint32_t channelCount = UINT_MAX;
        for (const auto channelMask : channelMasks) {
            uint32_t cnlCount;
            if (asAudioPort()->useInputChannelMask()) {
                cnlCount = audio_channel_count_from_in_mask(channelMask);
            } else {
                cnlCount = audio_channel_count_from_out_mask(channelMask);
            }
            if ((cnlCount < channelCount) && (cnlCount > 0)) {
                pickedChannelMask = channelMask;
                channelCount = cnlCount;
            }
        }
    } else {
        uint32_t channelCount = 0;
        uint32_t maxCount = MAX_MIXER_CHANNEL_COUNT;

        // For mixed output and inputs, use max mixer channel count. Do not
        // limit channel count otherwise
        if (asAudioPort()->getType() != AUDIO_PORT_TYPE_MIX) {
            maxCount = UINT_MAX;
        }
        for (const auto channelMask : channelMasks) {
            uint32_t cnlCount;
            if (asAudioPort()->useInputChannelMask()) {
                cnlCount = audio_channel_count_from_in_mask(channelMask);
            } else {
                cnlCount = audio_channel_count_from_out_mask(channelMask);
            }
            if ((cnlCount > channelCount) && (cnlCount <= maxCount)) {
                pickedChannelMask = channelMask;
                channelCount = cnlCount;
            }
        }
    }
}

/* format in order of increasing preference */
const audio_format_t PolicyAudioPort::sPcmFormatCompareTable[] = {
        AUDIO_FORMAT_DEFAULT,
        AUDIO_FORMAT_PCM_16_BIT,
        AUDIO_FORMAT_PCM_8_24_BIT,
        AUDIO_FORMAT_PCM_24_BIT_PACKED,
        AUDIO_FORMAT_PCM_32_BIT,
        AUDIO_FORMAT_PCM_FLOAT,
};

int PolicyAudioPort::compareFormats(audio_format_t format1, audio_format_t format2)
{
    // NOTE: AUDIO_FORMAT_INVALID is also considered not PCM and will be compared equal to any
    // compressed format and better than any PCM format. This is by design of pickFormat()
    if (!audio_is_linear_pcm(format1)) {
        if (!audio_is_linear_pcm(format2)) {
            return 0;
        }
        return 1;
    }
    if (!audio_is_linear_pcm(format2)) {
        return -1;
    }

    int index1 = -1, index2 = -1;
    for (size_t i = 0;
            (i < ARRAY_SIZE(sPcmFormatCompareTable)) && ((index1 == -1) || (index2 == -1));
            i ++) {
        if (sPcmFormatCompareTable[i] == format1) {
            index1 = i;
        }
        if (sPcmFormatCompareTable[i] == format2) {
            index2 = i;
        }
    }
    // format1 not found => index1 < 0 => format2 > format1
    // format2 not found => index2 < 0 => format2 < format1
    return index1 - index2;
}

uint32_t PolicyAudioPort::formatDistance(audio_format_t format1, audio_format_t format2)
{
    if (format1 == format2) {
        return 0;
    }
    if (format1 == AUDIO_FORMAT_INVALID || format2 == AUDIO_FORMAT_INVALID) {
        return kFormatDistanceMax;
    }
    int diffBytes = (int)audio_bytes_per_sample(format1) -
            audio_bytes_per_sample(format2);

    return abs(diffBytes);
}

bool PolicyAudioPort::isBetterFormatMatch(audio_format_t newFormat,
                                          audio_format_t currentFormat,
                                          audio_format_t targetFormat)
{
    return formatDistance(newFormat, targetFormat) < formatDistance(currentFormat, targetFormat);
}

void PolicyAudioPort::pickAudioProfile(uint32_t &samplingRate,
                                       audio_channel_mask_t &channelMask,
                                       audio_format_t &format) const
{
    format = AUDIO_FORMAT_DEFAULT;
    samplingRate = 0;
    channelMask = AUDIO_CHANNEL_NONE;

    // special case for uninitialized dynamic profile
    if (!asAudioPort()->hasValidAudioProfile()) {
        return;
    }
    audio_format_t bestFormat = sPcmFormatCompareTable[ARRAY_SIZE(sPcmFormatCompareTable) - 1];
    // For mixed output and inputs, use best mixer output format.
    // Do not limit format otherwise
    if ((asAudioPort()->getType() != AUDIO_PORT_TYPE_MIX) || isDirectOutput()) {
        bestFormat = AUDIO_FORMAT_INVALID;
    }

    const AudioProfileVector& audioProfiles = asAudioPort()->getAudioProfiles();
    for (size_t i = 0; i < audioProfiles.size(); i ++) {
        if (!audioProfiles[i]->isValid()) {
            continue;
        }
        audio_format_t formatToCompare = audioProfiles[i]->getFormat();
        if ((compareFormats(formatToCompare, format) > 0) &&
                (compareFormats(formatToCompare, bestFormat) <= 0)) {
            uint32_t pickedSamplingRate = 0;
            audio_channel_mask_t pickedChannelMask = AUDIO_CHANNEL_NONE;
            pickChannelMask(pickedChannelMask, audioProfiles[i]->getChannels());
            pickSamplingRate(pickedSamplingRate, audioProfiles[i]->getSampleRates());

            if (formatToCompare != AUDIO_FORMAT_DEFAULT && pickedChannelMask != AUDIO_CHANNEL_NONE
                    && pickedSamplingRate != 0) {
                format = formatToCompare;
                channelMask = pickedChannelMask;
                samplingRate = pickedSamplingRate;
                // TODO: shall we return on the first one or still trying to pick a better Profile?
            }
        }
    }
    ALOGV("%s Port[nm:%s] profile rate=%d, format=%d, channels=%d", __FUNCTION__,
            asAudioPort()->getName().c_str(), samplingRate, channelMask, format);
}
```

对于采样格式，精度越高优先级越高；对于通道掩码（通道数），如果是直接输出 (Direct Output)，则通道数越小优先级越高，否则，通道数越大优先级越高；对于采样率，如果是直接输出 (Direct Output)，则采样率越小优先级越高，否则，采样率越大优先级越高。






















