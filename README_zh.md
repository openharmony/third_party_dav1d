# third_party_dav1d
## 概述
本文档适用于OpenHarmony 6.1及以上版本。

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

(1) 若支持AV1软件解码时，AV1解码建议规格至少为1080p (Main Profile(8bit、10bit)、High Profile(8bit、10bit)， Level 4.0)。

## OpenHarmony如何使用dav1d
部件AVCodec为OpenHarmony提供了统一的视频解码能力，参考[AVCodec 框架图](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/README_zh.md)，其中：

(1) 框架层的编解码框架对应用开发者提供接口，例如：[获取支持的编解码能力](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/obtain-supported-codecs.md)， [查询编解码档次和级别支持情况](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/obtain-supported-codecs.md#%E6%9F%A5%E8%AF%A2%E7%BC%96%E8%A7%A3%E7%A0%81%E6%A1%A3%E6%AC%A1%E5%92%8C%E7%BA%A7%E5%88%AB%E6%94%AF%E6%8C%81%E6%83%85%E5%86%B5)。

(2) 服务层中的软件解码器模块将对dav1d进行加载和封装。

### 集成与配置
(1) 部件AVCodec通过BUILD.gn文件引用部件dav1d，以集成并使用其AV1的解码能力。
```
// BUILD.gn
external_deps += ["dav1d:dav1d_ohos"]
```
(2) 部件AVCodec在[bundle.json](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/bundle.json)里声明的features：av_codec_support_av1_decoder用于使能AV1软解码，并在[Config.gni](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/config.gni)中做了初始化，方式如下：
```
declare_args() {
    av_codec_support_av1_decoder = true    // true表示默认开启AV1软解码
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
(1) AVCodec服务层中CodecServer通过[CodecFactory](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/services/services/codec/server/video/codec_factory.cpp)，调用AV1DecoderLoader的静态方法CreateByName()创建AV1的解码器(`AV1Decoder`)。

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
(1) AV1软解码为OpenHarmony中的可选能力。厂商可通过配置文件使能，配置文件路径可以如下：
```
    //vendor/${product_company}/${product_name}/config.json
```
配置方式可以如下，在av_codec部件节点的features中配置av_codec_support_av1_decoder, 值为true表示开启，false表示关闭：
```json
"multimedia:av_codec": {
    "features": {
        "av_codec_support_av1_decoder": false, // 设置false为关闭AV1软解码
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

(3) 另外，dav1d仅适用于软解解码。对`硬件解码`场景：服务层CodecFactory调用`HCodecLoader`的静态方法CreateByName()创建HCodec/HCodecList，它们通过HDI接口调用HAL或硬件解码器，厂商需适配并满足`HDI`接口的要求。

## License
具体许可条款请参见仓库根目录下的 LICENSE 文件。