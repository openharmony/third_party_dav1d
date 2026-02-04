# dav1d in OpenHarmony
## Overview
The repository includes the third-party open-source software dav1d, which is a video codec implementation for the AV1 standard. In OpenHarmony, dav1d serves as a fundamental component of the media subsystem, providing AV1 stream decoding capabilities for AVCodec.

Original Repository：https://code.videolan.org/videolan/dav1d
## Directory Structure
```
//third_party_dav1d
|-- BUILD.gn       # GN build configuration file, defines build rules and dependencies.
|-- bundle.json    # Component package configuration file, describes component information and dependencies.
|-- doc            # Project documentation directory
|-- examples       # Example code directory
|-- include        # Header files directory
|-- package        # Package-related files directory
|-- src            # Source code directory
|-- tests          # Test code directory
|-- tools          # Utility tools directory
```
## Compliance Requirements
According to the OpenHarmony PCS ([Product Compatibility Specification](https://www.openharmony.cn/systematic)):

(1) If AV1 decoding is supported, the recommended AV1 decoding capability is at least 1080p (Main Profile (8bit, 10bit), High Profile(8bit, 10bit), Level 4.0).

## Using dav1d in OpenHarmony
The AVCodec component provides OpenHarmony with a unified video decoding capability. Refer to the [AVCodec Architecture Diagram](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/README_zh.md) for an overview.
Specifically：

(1) The framework layer exposes APIs to application developers: [obtain supported codecs](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/media/avcodec/obtain-supported-codecs.md)， [checking the codec profile and level supported](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/media/avcodec/obtain-supported-codecs.md#checking-the-codec-profile-and-level-supported)。

(2) In the service layer, the software decoder module wraps the dav1d.

### Build Configuration
(1) The `AVCodec` component integrates and utilizes the AV1 decoding capabilities of the `dav1d` component by referencing it in its BUILD.gn file:
```gn
// BUILD.gn
external_deps += ["dav1d:dav1d_ohos"]
```
(2) The `AVCodec` component declares the feature `av_codec_support_av1_decoder` in its [bundle.json](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/bundle.json). This feature is initialized in [Config.gni](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/config.gni) as follows:
```gn
declare_args() {
    av_codec_support_av1_decoder = true    // Setting it to true enables AV1 soft decoding by default
}
```

### Decoder Creation
```
            +------------------+      +-----------------------------+
            |   CodecServer    |----->|   CodecFactory              |
            +------------------+      +-----------------------------+
            | - codec:CodecBase|      | + createByName():CodecBase  |
            +------------------+      +-----------------------------+
                                            | calls static method createByName()
                                   +--------v----------+
                                   | AV1DecoderLoader  |
                                   +-------------------+
                                   | + createByName()  |
                                   |    :CodecBase     |
                                   +--------|----------+
                                            | inherit
                                  +---------▼-----------+
                                  |  VideoCodecLoader   |
                                  +---------------------+
                                            | use
                                    +-------v---------+
                                    |   CodecBase     |
                                    +-------▲---------+ 
                                            | implement
                                    +-----------------+
                                    |   AV1Decoder    |
                                    +-----------------+
                    Figure 1: AVCodec AV1Decoder creation process
```

(1) In the AVCodec service layer, CodecServer uses [CodecFactory](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/services/services/codec/server/video/codec_factory.cpp) to invoke the static method CreateByName() of AV1DecoderLoader, which creates an AV1 decoder(`AV1Decoder`).

(2) `AV1Decoder` is a wrapper around the AV1 decoder from **dav1d**. It also provides a capability query function, GetCodecCapability(std::vector<CapabilityData>& capaArray), which reports the supported capabilities such as Profile and Level.

(3) The Profiles and Levels are fully defined in [avcodec_info.h](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/inner_api/native/avcodec_info.h):
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
Additionally, Profile and Level are exposed in native_avcodec_base.h in both [SDK](https://gitcode.com/openharmony/interface_sdk_c/blob/master/multimedia/av_codec/native_avcodec_base.h) and [AVCodec](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/kits/c/native_avcodec_base.h). These definitions serve as the public API for application developers.
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
Full documentation is available:[OH_AV1Profile](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1profile), [OH_AV1level](https://gitcode.com/openharmony/docs/blob/master/en/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1level).

## Vendor Customization
(1) AV1 software decoding is an optional capability in OpenHarmony. You can enable or disable AV1 software decoding via product-specific configuration. The configuration file can be locate at:

    //vendor/${product_company}/${product_name}/config.json

The configuration can be set as shown below: set `av_codec_support_av1_decoder` to true to enable AV1 software decoding, or false to disable it:
```json
"multimedia:av_codec": {
    "features": {
        "av_codec_support_av1_decoder": false, // AV1 software decoding is disabled
    }
}
```
(2) Vendors may adjust the supported AV1 Profile and Level according to the device’s CPU performance and system capabilities by modifying the values assigned to the output parameter capaArray in the GetCodecCapability function in the source code below:

    //foundation/multimedia/av_codec/services/engine/codec/video/av1decoder/av1decoder.cpp

```cpp
void Av1Decoder::GetCodecCapability(std::vector<CapabilityData> &capaArray)
{
    ...
    CapabilityData capsData;
    ...
    // Assign supported profiles, e.g., AV1 Main Profile, High Profile
    capsData.profile = {
        static_cast<int32_t>(AV1_PROFILE_MAIN),
        static_cast<int32_t>(AV1_PROFILE_HIGH)
    };
    std::vector<int32_t> levels;
    // Emplace supported levels for Main Profile (e.g., up to Level 4.0)
    for (int32_t j = 0; j <= static_cast<int32_t>(AV1Level::AV1_LEVEL_40); j++) {
        levels.emplace_back(j);
    }
    capsData.profileLevelsMap.insert({static_cast<int32_t>(AV1_PROFILE_MAIN), levels});
    ...
    capaArray.emplace_back(capsData);
}
```
If support for the Main Profile(10-bit), High Profile(10-bit) and Professional profile(10-bit, 12-bit) are required, the graphics system (specifically SurfaceBuffer) must support 10-bit and 12-bit color depth.

(3) Additionally, for hardware decoding:
In the service layer, CodecFactory invokes the static method CreateByName() of HCodecLoader to create HCodec/HCodecList, which then interact with the HAL/hardware decoder through the HDI. Vendors must implement and adapt their hardware decoders to comply with the requirements of the HDI interface.

## License
For license details, see the LICENSE file in the root directory for details.