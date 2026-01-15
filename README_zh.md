# dav1d
原始仓来源：https://code.videolan.org/videolan/dav1d

仓库包含第三方开源软件dav1d，dav1d是av1标准的视频编解码器。在OpenHarmony中，dav1d主要是媒体子系统的基础组件，提供av1码流的解码能力。
## 目录结构
```
//third_party_dav1d
|-- BUILD.gn
|-- bundle.json
|-- av1
|-- doc
|-- examples
|-- examples
|-- include
|-- package
|-- patches
|-- src
|-- tests
|-- tools
```
## OpenHarmony如何使用dav1d
OpenHarmony的系统部件需要在BUILG.gn中引用dav1d部件以使用libav1。
```
// BUILD.gn
external_deps + = [dav1d:dav1d_ohos]
```
### 使用dav1d库解码步骤
（1）解码器初始化
```
DAV1D_API void dav1d_default_settings(Dav1dSettings *s)
s：解码器参数设置
```
（2）打开解码器实例
```
DAV1D_API int dav1d_open(Dav1dContext **c_out, const Dav1dSettings *s)
c_out：要打开的解码器实例。c_out将被设置为新分配的解码器实例
s：解码器配置
注意：解码完成后必须使用dav1d_close()释放实例
返回值：0解码成功，非0解码失败，参考DAV1D_ERR中错误码的说明
```
（3）输送数据进行解码
```
DAV1D_API int dav1d_send_data(Dav1dContext *c, Dav1dData *in)
c：输入解码器实例
in：输入比特流数据。成功后，数据输送至库中
返回值：0解码成功，数据已输送
       DAV1D_ERR(EAGAIN)：数据未被输送。在函数输送新数据之前，应调用dav1d_get_picture()来获取一个或多个帧
       其他DAV1D_ERR错误码：解码过程中出现错误，或由于传入的参数无效。该数据的所有权仍归调用者所有
```
（4）获取解码后图像
```
DAV1D_API int dav1d_get_picture(Dav1dContext *c, Dav1dPicture *out)
c：输入解码器实例
out：输出帧。调用者取得返回引用的所有权
返回值：0解码成功，返回一帧
       DAV1D_ERR(EAGAIN)：数据不足，无法输出帧。应调用dav1d_send_data()输入新数据
       其他DAV1D_ERR错误码：解码过程中出错或入参无效
注意：如需排空解码器内部缓存的帧（即在流结束时），请循环调用本函数，直到返回DAV1D_ERR(EAGAIN)为止
```
（5）销毁解码器
```
DAV1D_API void dav1d_close(Dav1dContext **c_out)
c_out：要关闭的解码器实例。c_out将被设置为NULL
```
## 功能支持说明
OpenHarmony目前仅集成了dav1d的解码能力，用于解析av1的码流，暂不支持视频编码功能。
## License
详见仓库目录下的LICENSE.md文件