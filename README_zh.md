# third_party_dav1d
## 概述
本仓库集成第三方开源软件dav1d(AV1标准的视频解码器的开源实现)。在OpenHarmony中，dav1d主要作为媒体子系统的基础组件，为AVCodec提供AV1码流的解码能力。

dav1d来源：https://code.videolan.org/videolan/dav1d
## 目录结构
```
//third_party_dav1d
|-- BUILD.gn       # GN构建配置文件，定义编译规则和依赖关系。
|-- bundle.json    # 组件包配置文件，描述组件信息和依赖。
|-- doc            # 项目文档目录
|-- examples       # 示例代码目录
|-- include        # 头文件目录
|-- package        # 打包相关文件目录
|-- src            # 源代码目录
|-- tests          # 测试代码目录
|-- tools          # 工具程序目录
```
## OpenHarmony对解码器的要求
根据[OpenHarmony产品兼容性规范](https://www.openharmony.cn/systematic)：

(1) AV1软件解码为可选能力。

(2) 若支持AV1软件解码时，AV1解码建议规格至少为1080p (Main Profile(8bit、10bit)、High Profile(8bit、10bit)，Level 4.0)。

## OpenHarmony如何使用dav1d
部件AVCodec为OpenHarmony提供了统一的视频解码能力，参考[AVCodec 框架图](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/README_zh.md)，其中：

(1) 框架层的编解码框架对应用开发者提供接口，例如：[获取支持的编解码能力](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/obtain-supported-codecs.md)，[查询编解码档次和级别支持情况](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/obtain-supported-codecs.md#%E6%9F%A5%E8%AF%A2%E7%BC%96%E8%A7%A3%E7%A0%81%E6%A1%A3%E6%AC%A1%E5%92%8C%E7%BA%A7%E5%88%AB%E6%94%AF%E6%8C%81%E6%83%85%E5%86%B5)。

(2) 服务层中的软件解码器模块将对dav1d进行加载和封装。

### 集成与配置

部件AVCodec通过feature配置的方式来使能AV1软解码。

(1) 声明：AVCodec仓的[bundle.json](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/bundle.json)里声明的features：`av_codec_support_av1_decoder`用于使能AV1软解码。
```json
   "component": {
      "name": "av_codec",
      "subsystem": "multimedia",
      "features": [
        "av_codec_support_av1_decoder"
      ]
   }
```

(2) 定义与赋值：多条件判断是否使能AV1解码器(av_codec_support_av1_decoder)。

* 在AVCodec仓[Config.gni](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/config.gni)中通过declare_args()定义并默认使能AV1软解码。
* 产品配置feature的值会覆盖上述默认值，参考[产品如何配置部件的feature](https://gitcode.com/openharmony/build/blob/master/docs/%E9%83%A8%E4%BB%B6%E5%8C%96%E7%BC%96%E8%AF%91%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5.md#11-%E4%BA%A7%E5%93%81%E5%A6%82%E4%BD%95%E9%85%8D%E7%BD%AE%E9%83%A8%E4%BB%B6%E7%9A%84feature)。
* AVCodec仓的[Config.gni](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/config.gni)会校验部件dav1d是否装配。如果部件[未装配](https://gitcode.com/openharmony/build/blob/master/docs/%E9%83%A8%E4%BB%B6%E5%8C%96%E7%BC%96%E8%AF%91%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5.md)，则不使能AV1软解码。
* 另，使能AV1解码器时，则定义宏`SUPPORT_CODEC_AV1`，通过该宏在实际代码中控制创建AV1解码器等逻辑。
```gn
// Config.gni
declare_args() {
    // 初始化true，表示默认开启AV1软解码
    av_codec_support_av1_decoder = true         
}

// 如果产品裁剪了部件dav1d，将不开启AV1软解码
if (!defined(global_parts_info) || !defined(global_parts_info.multimedia_dav1d)) {
    av_codec_support_av1_decoder = false
}

// 开启AV1解码器时，定义宏SUPPORT_CODEC_AV1
if (av_codec_support_av1_decoder) {
    av_codec_defines += ["SUPPORT_CODEC_AV1"]
}
```

(3) 使用：AVCodec仓的[BUILD.gn](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/services/engine/codec/video/BUILD.gn)通过判断是否使能AV1解码器来决定是否引用部件dav1d，以集成并使用其AV1的解码能力。
```gn
// BUILD.gn
if (av_codec_support_av1_decoder) {
    external_deps += ["dav1d:dav1d_ohos"]
}
```

### 实现及调用
```
            +------------------+      +-----------------------------+
            |   CodecServer    |----->|   CodecFactory              |
            +------------------+      +-----------------------------+
            | - codec:CodecBase|      | + createByName():CodecBase  |
            +------------------+      +-----------------------------+
                                            | 调用静态方法
                                   +--------v----------+
                                   | AV1DecoderLoader  |
                                   +-------------------+
                                   | + createByName()  |
                                   |    :CodecBase     |
                                   +--------|----------+
                                            | 继承
                                  +---------▼-----------+
                                  |  VideoCodecLoader   |
                                  +---------------------+
                                            | 使用
                                    +-------v---------+
                                    |   CodecBase     |
                                    +-------▲---------+ 
                                            | 实现
                                    +-----------------+
                                    |   AV1Decoder    |
                                    +-----------------+
                             图1 AVCodec服务层AV1软件解码器创建示意图
```
(1) AVCodec服务层中CodecServer通过[CodecFactory](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/services/services/codec/server/video/codec_factory.cpp)创建AV1解码器。
* 若宏开关`SUPPORT_CODEC_AV1`已定义则通过`AV1DecoderLoader::CreateByName()`创建AV1解码器(`AV1Decoder`)；如果宏开关未定义，则创建解码器失败

(2) `AV1Decoder`即是对**dav1d**中AV1解码器的封装，实现能力查询函数GetCodecCapability(std::vector<CapabilityData> &capaArray)，即实现对包括Profile与Level在内的能力信息查询。

(3) AVCodec仓的[avcodec_info.h](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/inner_api/native/avcodec_info.h)中对Profile和Level进行了完整的定义：
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
同时在[SDK仓](https://gitcode.com/openharmony/interface_sdk_c/blob/master/multimedia/av_codec/native_avcodec_base.h)和[AVCodec仓](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/kits/c/native_avcodec_base.h)的native_avcodec_base.h中定义了Profile和Level，作为应用开发者的接口，详细描述见[OH_AV1Profile](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1profile)，[OH_AV1level](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1level):
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

## 能力定制说明
(1) OpenHarmony将AV1软解码作为可选能力，厂商可通过在config.json中装配部件dav1d并使能部件AVCodec特性来开启软解码能力：
* 装配部件dav1d
  * 参考[装配部件](https://gitcode.com/openharmony/docs/blob/master/zh-cn/design/OpenHarmony%E9%83%A8%E4%BB%B6%E8%AE%BE%E8%AE%A1%E5%92%8C%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97.md#%E8%A3%85%E9%85%8D%E9%83%A8%E4%BB%B6)，配置文件路径可以是`//vendor/${product_company}/${product_name}/config.json`    
  * 装配时内容可以如下：
```json
{
  ...
  "subsystems": [{
      "subsystem": "thirdparty",
      "components": [{
          "component": "dav1d",  // 装配部件dav1d
          "features": []
        }]
    }]
}
```
* 配置部件AVCodec的特性，参考[配置特性](https://gitcode.com/openharmony/docs/blob/master/zh-cn/design/OpenHarmony%E9%83%A8%E4%BB%B6%E8%AE%BE%E8%AE%A1%E5%92%8C%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97.md#%E9%85%8D%E7%BD%AE%E7%89%B9%E6%80%A7)，在上述config.json装配部件AVCodec的同时配置其特性av_codec_support_av1_decoder, 值为true表示开启AV1软解码，false表示关闭AV1软解码：
```json
{
  {
    "subsystem": "multimedia",
    "components": [{
        "component": "av_codec",   // 配置部件AVCodec的特性
        "features": [ 
            "av_codec_support_av1_decoder = true", // 配置特性为true，表示开启AV1软解码
        ] 
      }]
  }
}
```

(2) 厂商可根据自身的CPU性能及系统能力修改AV1支持的Profile与Level，即在如下源码中，修改函数GetCodecCapability出参capaArray的赋值，完成后需重新编译相关组件，可通过[OH_AVCapability_GetSupportedProfiles](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcapability-h.md#oh_avcapability_getsupportedprofiles)，[OH_AVCapability_GetSupportedLevelsForProfile](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcapability-h.md#oh_avcapability_getsupportedlevelsforprofile)接口获取实际支持的Profile与Level情况来验证：
```
    //foundation/multimedia/av_codec/services/engine/codec/video/av1decoder/av1decoder.cpp
```
```cpp
void Av1Decoder::GetCodecCapability(std::vector<CapabilityData> &capaArray)
{
    ...
    CapabilityData capsData;
    ...
    // 配置支持的Profile，例如支持AV1 Main Profile，High Profile
    capsData.profile = {
        static_cast<int32_t>(AV1_PROFILE_MAIN),
        static_cast<int32_t>(AV1_PROFILE_HIGH)
    };
    std::vector<int32_t> levels;
    // 为Profile设置支持的Level范围，例如为Main Profile最高支持到Level 4.0
    for (int32_t j = 0; j <= static_cast<int32_t>(AV1Level::AV1_LEVEL_40); j++) {
        levels.emplace_back(j);
    }
    capsData.profileLevelsMap.insert({static_cast<int32_t>(AV1_PROFILE_MAIN), levels}); // 能力信息写入输出参数
    ...
    capaArray.emplace_back(capsData);
}
```
注：配置Main Profile(10-bit), High Profile(10-bit)和Professional Profile(10-bit, 12-bit)时，依赖OH图形系统（的SurfaceBuffer）支持10-bit和12-bit，否则会出现内存申请失败导致解码器初始化失败。

(3) 另外，dav1d仅适用于软件解码。对`硬件解码`场景：服务层CodecFactory调用`HCodecLoader`的静态方法CreateByName()创建HCodec/HCodecList，它们通过HDI接口调用HAL或硬件解码器，厂商需适配并满足`HDI`接口的要求。

## 应用如何使用dav1d

(1) 应用开发者通过AVCodec Kit使用AV1软件解码器，详情见开发指南[AVCodec Kit](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/Readme-CN.md)。

(2) 此外，AVCodec声明的`Syscap`（系统能力）如下，其中`SystemCapability.Multimedia.Media.VideoDecoder`为使能AV1解码器的应支持的系统能力，应用开发者可以通过Native API：`canIUse()`（参考[系统能力使用指南](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/syscap.md)）查询设备是否具备该系统能力：

```json
   "component": {
      "name": "av_codec",
      "subsystem": "multimedia",
      "syscap": [
        "SystemCapability.Multimedia.Media.Muxer",
        "SystemCapability.Multimedia.Media.Spliter",
        "SystemCapability.Multimedia.Media.AudioCodec",
        "SystemCapability.Multimedia.Media.AudioDecoder",
        "SystemCapability.Multimedia.Media.AudioEncoder",
        "SystemCapability.Multimedia.Media.VideoDecoder",
        "SystemCapability.Multimedia.Media.VideoEncoder",
        "SystemCapability.Multimedia.Media.CodecBase"
      ],
      "features": [
        "av_codec_support_av1_decoder"
      ]
   }
```

## License
具体许可条款请参见仓库根目录下的 LICENSE 文件。