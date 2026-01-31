# dav1d
Original Repository：https://code.videolan.org/videolan/dav1d

The repository includes the third-party open-source software dav1d, which is a video codec implementation for the AV1 standars. In OpenHarmony, dav1d serves as a fundamental component of the media subsystem, providing AV1 stream decoding capabilities for AVCodec.
## Directory Structure
```
//third_party_dav1d
|-- BUILD.gn       # GN build configuration file, defines build rules and dependencies. Added by OpenHarmony
|-- bundle.json    # Component package configuration file, describes component information and dependencies. Added by OpenHarmony
|-- doc            # Project documentation directory
|-- examples       # Example code directory
|-- include        # Header files directory
|-- package        # Package-related files directory
|-- src            # Source code directory
|-- tests          # Test code directory
|-- tools          # Tools and utilities directory
```
## How to Use dav1d in OpenHarmony
System components in OpenHarmony need to reference the dav1d component in BUILD.gn to use it.
```
// BUILD.gn
external_deps + = [dav1d:dav1d_ohos]
```

## Feature Support
**Since：** 6.1
(1) When the application attempt to create a decoder according to the [platform rules](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/media/avcodec/avcodec-support-formats.md), the system preferentially creates a hardware decoder instance. If the system does not support hardware decoding or the hardware decoder resources are insufficient, the system creates a software decoder instance.

(2) AV1 software decoding is an optional capability in OpenHarmony. you can enable/disable the feature via your own product configuration. The product configuration path can be as follows: 
```
//vendor/${pruduct_company}/${product_name}/config.json
```
The configuration can be set as shown below:
```json
"multimedia:av_codec": {
    "features": {
        "av_codec_support_av1_decoder": false,
    }
}
```
(3) According to the OpenHamony PCS ([Product Compatibility Specification](https://www.openharmony.cn/systematic)), If AV1 decoding is supported, the recommended AV1 decoding capability is at least 1080p(Main Profile(8bit、10bit)、High Profile(8bit、10bit),level 4.0). You can modify the supported Profile and Level in the AVCodec, the path is as follows：
```
//foundation/multimedia/avcodec/services/engine/codec/video/av1decoder/av1decoder.cpp
```
and the function:
```C++
void Av1Decoder::GetCodecCapability(std::vector<CapabilityData> &capaArray)
{
    ...
    CapabilityData capsData;
    ...
    // Assign supported profiles，e.g., AV1 Main Profile，High Profile
    capsData.profile = {
        static_cast<int32_t>(AV1_PROFILE_MAIN),
        static_cast<int32_t>(AV1_PROFILE_HIGH)
    };
    std::vector<int32_t> levels；
    // Assign supported levels for the profile，e.g., level 4.0
    for (int32_t j = 0; j <= static_cast<int32_t>(AV1Level::AV1_LEVEL_4_0); j++) {
        levels.emplace_back(j);
    }
    capsData.profileLevelsMap.insert({static_cast<int32_t>(AV1_PROFILE_MAIN), levels});
    ...
    capaArray.emplace_back(capsData);
}
```
According to the AV1 standard, the above Profiles and Levels are fully defined in [avcodec_info.h](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/inner_api/native/avcodec_info.h):
```C
enum AV1Profile : int32_t {
    AV1_PROFILE_MAIN = 0,
    AV1_PROFILE_HIGH = 1,
    AV1_PROFILE_PROFESSIONAL = 2,
};

enum AV1Level : int32_t {
    AV1_LEVEL_20 = 0,
    AV1_LEVEL_21 = 1,
    AV1_LEVEL_22 = 2,
    ...
    AV1_LEVEL_72 = 22,
    AV1_LEVEL_73 = 23,
};
```
If application developers need to be aware of them, the definitions must also be included in both the [SDK](https://gitcode.com/openharmony/interface_sdk_c/blob/master/multimedia/av_codec/native_avcodec_base.h) and [AVCodec](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/kits/c/native_avcodec_base.h):
```C
typedef enum OH_AV1Profile {
    AV1_PROFILE_MAIN = 0,
    AV1_PROFILE_HIGH = 1,
    AV1_PROFILE_PROFESSIONAL = 2,
} OH_AV1Profile;

typedef enum OH_AV1Level {
    AV1_LEVEL_20 = 0,
    AV1_LEVEL_21 = 1,
    AV1_LEVEL_22 = 2,
    ...
    AV1_LEVEL_72 = 22,
    AV1_LEVEL_73 = 23,
} OH_AV1Level;
```
Full documentation is available:[OH_AV1Profile](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1profile), [OH_AV1level](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1level).

In addition, if support for the Professional Profile (12-bit) is required, the display system (specifically SurfaceBuffer) must support 12-bit color depth.

(4) Application developers can query the above configurations and capabilities through the AVCodec interface. see [Obtaining Supported Codecs](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/media/avcodec/obtain-supported-codecs.md#obtaining-supported-codecs), [Checking the Codec Profile and Level Supported](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/media/avcodec/obtain-supported-codecs.md#checking-the-codec-profile-and-level-supported).

## License
See the LICENSE file in the root directory for details.