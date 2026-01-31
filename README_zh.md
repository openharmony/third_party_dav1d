# dav1d
原始仓来源：https://code.videolan.org/videolan/dav1d

仓库包含第三方开源软件dav1d，dav1d是AV1标准的视频编解码器。在OpenHarmony中，dav1d主要是媒体子系统的基础组件，为AVCodec提供AV1码流的解码能力。
## 目录结构
```
//third_party_dav1d
|-- BUILD.gn       # GN构建配置文件，定义编译规则和依赖关系。由OpenHarmony增加
|-- bundle.json    # 组件包配置文件，描述组件信息和依赖。由OpenHarmony增加
|-- doc            # 项目文档目录
|-- examples       # 示例代码目录
|-- include        # 头文件目录
|-- package        # 打包相关文件目录
|-- src            # 源代码目录
|-- tests          # 测试代码目录
|-- tools          # 工具程序目录
```

## OpenHarmony如何集成dav1d
OpenHarmony通过部件AVCodec集成了dav1d的解码能力，即系统部件可通过BUILD.gn文件引用dav1d部件，以集成并使用其解码能力，方式如下：
```
// BUILD.gn
external_deps + = [dav1d:dav1d_ohos]
```

## 功能支持说明
**以下说明适用起始版本：** 6.1

(1) 应用在创建AV1解码器时，按照[平台统一规则](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/avcodec-support-formats.md)系统会优先创建硬件解码器，如无硬件解码器或则硬件解码器资源不足时创建软件解码器（当前AVCodec尚不支持AV1的硬解），如无解码能力则创建失败。

(2) AV1软解码为OpenHamony中的可选能力。厂商可通过配置文件开关AV1的软解码，配置文件路径可以如下：
```
//vendor/${pruduct_company}/${product_name}/config.json
```
配置方式可以如下，在av_codec部件节点的features中配置av_codec_support_av1_decoder：
```json
"multimedia:av_codec": {
    "features": {
        "av_codec_support_av1_decoder": false,
    }
}
```
(3) 按[Openharmony产品兼容性规范](https://www.openharmony.cn/systematic)，若支持AV1软件解码时，AV1解码建议规格至少为1080p(Main Profile(8bit、10bit)、High Profile(8bit、10bit)， Level 4.0)。厂商可以在AVCodec源码中修改AV1支持的Profile与Level，路径如下：
```
//foundation/multimedia/avcodec/services/engine/codec/video/av1decoder/av1decoder.cpp
```
在GetCodecCapability函数中，修改对出参capaArray(成员profileLevelsMap记录支持情况)的赋值，完成后重新编译：
```C++
void Av1Decoder::GetCodecCapability(std::vector<CapabilityData> &capaArray)
{
    ...
    CapabilityData capsData;
    ...
    // 赋值支持的profile，例如支持AV1 Main Profile，High Profile
    capsData.profile = {
        static_cast<int32_t>(AV1_PROFILE_MAIN),
        static_cast<int32_t>(AV1_PROFILE_HIGH)
    };
    std::vector<int32_t> levels；
    // 赋值profile下支持的level，例如Main Profile支持到level 4.0
    for (int32_t j = 0; j <= static_cast<int32_t>(AV1Level::AV1_LEVEL_4_0); j++) {
        levels.emplace_back(j);
    }
    capsData.profileLevelsMap.insert({static_cast<int32_t>(AV1_PROFILE_MAIN), levels}); // 赋值profile下支持的level
    ...
    capaArray.emplace_back(capsData);
}
```
上述Profile和Level均按照AV1标准进行了完整的定义，在AVCodec仓的[avcodec_info.h](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/inner_api/native/avcodec_info.h)中：
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
若需要应用开发者感知，还需要同时在[SDK仓](https://gitcode.com/openharmony/interface_sdk_c/blob/master/multimedia/av_codec/native_avcodec_base.h)和[AVCodec仓](https://gitcode.com/openharmony/multimedia_av_codec/blob/master/interfaces/kits/c/native_avcodec_base.h)的native_avcodec_base.h中定义:
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
完整的定义描述可见[OH_AV1Profile](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1profile)，[OH_AV1level](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/reference/apis-avcodec-kit/capi-native-avcodec-base-h.md#oh_av1level)

另外，若厂商需要支持professional profile(12bit)时，需要OH显示系统（中surfaceBuffer）支持12bit。

(4) 厂商适配了上述开关及能力后，应用开发者都可以通过AVCodec的接口查询到。详见[获取支持的编解码能力](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/obtain-supported-codecs.md)， [查询编解码档次和级别支持情况](https://gitcode.com/openharmony/docs/blob/master/zh-cn/application-dev/media/avcodec/obtain-supported-codecs.md#%E6%9F%A5%E8%AF%A2%E7%BC%96%E8%A7%A3%E7%A0%81%E6%A1%A3%E6%AC%A1%E5%92%8C%E7%BA%A7%E5%88%AB%E6%94%AF%E6%8C%81%E6%83%85%E5%86%B5)。

## License
详见仓库目录下的LICENSE文件