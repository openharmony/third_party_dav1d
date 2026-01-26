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
### Decoding Steps Using dav1d
（1）Decoder Initialization
```
DAV1D_API void dav1d_default_settings(Dav1dSettings *s)
s: Input settings context.

```
（2）open decoder instance
```
DAV1D_API int dav1d_open(Dav1dContext **c_out, const Dav1dSettings *s)
c_out：The decoder instance to open. c_out will be set to the allocated context.
s：Input settings context.
note：The context must be freed using dav1d_close() when decoding is finished.
return：0 on success, or < 0 (a negative DAV1D_ERR code) on error.

```
（3）Decode one frame
```
DAV1D_API int dav1d_send_data(Dav1dContext *c, Dav1dData *in)
c：Input decoder instance.
in：Input bitstream data. On success, ownership of the reference is passed to the library.
return：0: Success, and the data was consumed.
DAV1D_ERR(EAGAIN)：The data can't be consumed. dav1d_get_picture() should be called to get one or more frames before the function can consume new data.
Other negative DAV1D_ERR codes: Error during decoding or because of invalid passed-in arguments. The reference remains owned by the caller.

```
（4）Get the decoded image
```
DAV1D_API int dav1d_get_picture(Dav1dContext *c, Dav1dPicture *out)
c：Input decoder instance.
out：Output frame. The caller assumes ownership of the returned reference.
return：0: Success, and a frame is returned.
DAV1D_ERR(EAGAIN): Not enough data to output a frame. dav1d_send_data() should be called with new input.
Other negative DAV1D_ERR codes: Error during decoding or because of invalid passed-in arguments.
note：To drain buffered frames from the decoder (i.e. on end of stream), call this function until it returns DAV1D_ERR(EAGAIN).

```
（5）Destroy the decoder
```
DAV1D_API void dav1d_close(Dav1dContext **c_out)
c_out：The decoder instance to close. c_out will be set to NULL.

```
## Feature Support
OpenHarmony currently integrates the decoding capability of dav1d, used to parse AV1 bitstreams.
OpenHarmony does not mandate that device manufacturers support AV1 decoding, nor does it impose any compulsory profile or level requirements.
## License
See the LICENSE file in the root directory for details.