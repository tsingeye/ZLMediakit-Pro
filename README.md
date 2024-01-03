# 一、概述
zlmediakit pro版本支持基于ffmpeg的转码能力，在开源版本强大功能的基础，是开源ZLMediakit的超集，新增支持如下能力：

- 1、音视频间任意转码(包括h265/h264/opus/g711/aac等)。
- 2、基于配置文件的转码，支持设置比特率，codec类型等参数。
- 3、基于http api的动态增减转码，支持设置比特率，分辨率倍数，codec类型、滤镜等参数。
- 4、支持硬件、软件自适应转码。
- 5、支持按需转码，有人观看才转码。
- 6、支持负载过高时，转码主动降低帧率且不花屏。
- 7、支持滤镜，支持添加osd文本以及logo角标等能力。

<img src="https://upload-images.jianshu.io/upload_images/8409177-05174aef527943e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/200" alt="图片.png" style="zoom:50%;" />

# 二、转码实现原理

- 视频转码原理
  <img src="https://upload-images.jianshu.io/upload_images/8409177-c5e3cfc6ba45ae48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="图片.png" style="zoom:50%;" />



- 音频转码原理
  <img src="https://upload-images.jianshu.io/upload_images/8409177-c50aedc47037188c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="图片.png" style="zoom: 67%;" />


# 三、使用方法
目前zlmediakit pro转码能力支持两种使用方式，第一种是基于配置文件方式，在设置好配置文件后，所有流都支持转码为目标编码格式直播流，第二种模式基于http api方式，此方式更灵活，功能强大，可以指定更多转码相关参数。

## 3.1 基于配置文件的转码
```ini
[transcode]
#转码stream_id后缀,为空时关闭转码
suffix=
#默认转码视频目标codec，支持H264/H265/JPEG/copy 
vcodec=H264
#默认转码音频目标codec，支持mpeg4-generic/PCMA/PCMU/opus/copy
acodec=mpeg4-generic
#是否开启ffmpeg日志
enable_ffmpeg_log=0
# h264解码器白名单
decoder_h264=h264_cuvid,h264_qsv,h264_videotoolbox,h264_nvmpi,h264_bm,libopenh264
# h265解码器白名单
decoder_h265=hevc_cuvid,hevc_qsv,hevc_videotoolbox,hevc_nvmpi,hevc_bm
# h264编码器白名单
encoder_h264=h264_nvenc,h264_qsv,h264_videotoolbox,h264_nvmpi,h264_bm,libx264,libopenh264
# h265编码器白名单
encoder_h265=hevc_nvenc,hevc_qsv,hevc_videotoolbox,hevc_nvmpi,hevc_bm,libx265
```

在上述配置文件中，如果用户配置好`suffix`，那么zlmediakit将统一把所有直播流转码为目标编码格式，用户通过访问新的流地址即可确保为预期编码格式视频。

例如源视频地址为：rtmp://127.0.0.1/live/test, 那么转码后地址即为：rtmp://127.0.0.1/live/test_H264。

当配置文件修改为`suffix=null`时，转码后流会直接替换原始流(不会有`_suffix`后缀)；替换模式下，建议`rtsp.directProxy/rtmp.directProxy`都设置为0。

如果源视频编码格式与目标编码格式一致，那么zlmediakit为了确保性能最优，将直接拷贝流数据(不会编码)。

基于配置文件方式的转码使用最简单，可以使用于安防行业H265视频无法webrtc/mse播放的场景。

## 3.2 基于http api的转码
zlmediakit同时还提供基于http api的转码方式，这种方式支持的功能更强大，使用更灵活，同时支持一个流转码成多个目标流(比如说不同分辨率的场景)。

- 请求地址：`/index/api/setupTranscode`

- 请求参数：

  |        参数        |   参数类型   |                      释意                       | 是否必选 |
  |:--------:|:---------------------------------------------:| :----------------------------------------------------------: | :------: |
  | `secret`           | `string` |                api操作密钥(配置文件配置)                |    Y     |
  | `vhost`            | `string` |          流的虚拟主机，例如`__defaultVhost__`          |    Y     |
  | `app`              | `string` |                 流的应用名，例如live                  |    Y     |
  | `stream`           | `string` |                 流的id名，例如test                  |    Y     |
  | `name`             | `string` |       转码名(后缀)，功能类似配置文件transcode.suffix        |    Y     |
  | `add`              |  `int`   |                1：添加转码; 0: 删除转码                |    Y     |
  | `video_codec`      | `string` |       视频转码的codec,支持H264/H265/JPEG/copy        |    Y     |
  | `video_bitrate`    |  `int`   |                   转码后视频的比特率                   |    Y     |
  | `video_scale`      | `float`  |             转码视频宽高拉伸比例，取值范围0.1~10             |    Y     |
  | `audio_codec`      | `string` | 音频转码codec，支持mpeg4-generic/PCMA/PCMU/opus/copy |    Y     |
  | `audio_bitrate`    |  `int`   |                   转码后音频比特率                    |    Y     |
  | `audio_samplerate` |  `int`   |                   转码后音频采样率率                   |    Y     |
  | `filter`           | `string` |        avfilter滤镜参数,用法与ffmpeg -vf 参数一致        |    Y     |
  | `force`            |  `bool`  |          是否强制转码，强制转码时不管目标编码是否一致，默认否           |    N    |
  | `decoder_threads`  |  `int`   |           解码线程数，默认2个，最大16个，音频强制为1个            |    N     |
  | `encoder_threads`  |  `int`   |           编码线程数，默认4个，最大16个，音频强制为1个            |  N     |
  | `hw_decoder`       |  `bool`  |                是否启用硬件解码器，默认启用                 |    N    |
  | `hw_encoder`       |  `bool`  |                是否启用硬件编码器，默认启用                 |    N     |
  | `decoder_list`     | `string` |     视频ffmpeg解码器列表，例如: h264_cuvid,h264_qsv     |    N     |
  | `encoder_list`     | `string` |     视频ffmpeg编码器列表，例如: hevc_nvenc,hevc_qsv     |    N     |
  | `gpu_index`        |  `int`   |                硬件编解码gpu索引号，默认0                |    N     |
  | `enable_hls`       |  `bool`  |             转码后是否转换成hls-mpegts协议              |    N     |
  | `enable_hls_fmp4`  |  `bool`  |              转码后是否转换成hls-fmp4协议               |    N     |
  | `enable_mp4`       |  `bool`  |                 转码后是否允许mp4录制                  |    N     |
  | `enable_rtsp`      |  `bool`  |                 转码后是否转rtsp协议                  |    N     |
  | `enable_rtmp`      |  `bool`  |               转码后是否转rtmp/flv协议                |    N     |
  | `enable_ts`        |  `bool`  |             转码后是否转http-ts/ws-ts协议             |    N     |
  | `enable_fmp4`      |  `bool`  |           转码后是否转http-fmp4/ws-fmp4协议           |    N     |
  | `hls_demand`       |  `bool`  |                转码后该协议是否有人观看才生成                |    N     |
  | `rtsp_demand`      |  `bool`  |                转码后该协议是否有人观看才生成                |    N     |
  | `rtmp_demand`      |  `bool`  |                转码后该协议是否有人观看才生成                |    N     |
  | `ts_demand`        |  `bool`  |                转码后该协议是否有人观看才生成                |    N     |
  | `fmp4_demand`      |  `bool`  |                转码后该协议是否有人观看才生成                |    N     |
  | `enable_audio`     |  `bool`  |                 转码后转协议时是否开启音频                 |    N     |
  | `add_mute_audio`   |  `bool`  |               转码后无音频是否添加静音aac音频               |    N     |
  | `mp4_save_path`    | `string` |            转码后mp4录制文件保存根目录，置空使用默认             |    N     |
  | `mp4_max_second`   |  `int`   |               转码后mp4录制切片大小，单位秒                |    N     |
  | `mp4_as_player`    |  `bool`  |            转码后MP4录制是否当作观看者参与播放人数计数            |    N     |
  | `hls_save_path`    | `string` |            转码后hls文件保存保存根目录，置空使用默认             |    N     |
  | `modify_stamp`     |  `int`   |    转码后该流是否开启时间戳覆盖(0:绝对时间戳/1:系统时间戳/2:相对时间戳)    |    N     |
  | `auto_close`       |  `bool`  |          转码后无人观看是否自动关闭流(不触发无人观看hook)          |    N     |



- 响应：

   ```json
   {
      "code" : 0,
      "msg" : "success"
   }
   ```



## 3.3 使用http api获取转码信息

- 请求接口：/index/api/getMediaInfo
- 请求回复：请查看`transcode`字段

```json
{
  "aliveSecond": 88,
  "app": "live",
  "bytesSpeed": 330246,
  "code": 0,
  "createStamp": 1691902256,
  "isRecordingHLS": true,
  "isRecordingMP4": false,
  "originSock": {
    "identifier": "2-51",
    "local_ip": "192.168.31.101",
    "local_port": 8000,
    "peer_ip": "192.168.31.101",
    "peer_port": 61801
  },
  "originType": 8,
  "originTypeStr": "rtc_push",
  "originUrl": "rtc://127.0.0.1/live/test?app=live&stream=test&type=push&session=1-50",
  "readerCount": 0,
  "schema": "rtsp",
  "stream": "test",
  "totalReaderCount": 0,
  "tracks": [
    {
      "codec_id": 0,
      "codec_id_name": "H264",
      "codec_type": 0,
      "fps": 30.0,
      "frames": 2648,
      "gop_interval_ms": 2012,
      "gop_size": 60,
      "height": 556,
      "key_frames": 51,
      "loss": 0.0,
      "ready": true,
      "width": 990
    },
    {
      "channels": 1,
      "codec_id": 4,
      "codec_id_name": "PCMU",
      "codec_type": 1,
      "frames": 4434,
      "loss": 0.0,
      "ready": true,
      "sample_bit": 16,
      "sample_rate": 8000
    }
  ],
  "transcode": [
    {
      "name": "codec",                     // 转码名称
      "setting": {                         // 转码配置信息
        "adecoder_threads": 1,           // 音频解码器线程数
        "aencoder_threads": 1,           // 音频编码器线程数
        "hw_decoder": true,              // 启动硬件解码器
        "hw_encoder": true,              // 启动硬件编码器
        "target_acodec": "mpeg4-generic",// 目标音频编码格式
        "target_vcodec": "H265",         // 目标视频编码格式
        "vdecoder_threads": 4,           // 视频解码器线程数
        "vencoder_threads": 8,           // 视频编码器线程数
        "force": false,                  // 是否强制转码
        "filter": "",                     // 滤镜参数
        "decoder_list" : ["h264_cuvid", "h264_qsv"],  // 解码器列表
        "encoder_list" : ["hevc_nvenc", "hevc_qsv"]   // 编码器列表
      },
      "adec": "pcm_mulaw",       // 音频解码器名称
      "aenc": "aac",             // 音频编码器名称
      "aenc_ctx": {              // 音频AVCodecContext信息
        "bit_rate": 32000,     // 比特率
        "channels": 1,         // 通道数
        "frame_number": 4055,  // 已编码帧数
        "frame_size": 1024,    // 每帧采样数
        "sample_fmt": "fltp",  // 音频编码输入格式
        "sample_rate": 48000   // 编码器采样率
      },
      "vdec": "h264",               // 视频解码器名称
      "venc": "hevc_videotoolbox",  // 视频编码器名称
      "venc_ctx": {                 // 视频AVCodecContext信息
        "bit_rate": 1000000,      // 比特率
        "fps": 20,                // 帧率
        "frame_number": 2595,     // 已编码帧数
        "gop": 60,								// gop大小
        "has_b_frames": 0,        // 是否编码b帧
        "height": 556,            // 视频高度
        "pix_fmt": "nv12",        // 编码器输入图片格式
        "width": 990              // 视频宽度
      }
    }
  ],
  "vhost": "__defaultVhost__"
}
```
# 欢迎联系试用：
Q&WX 18675721
## 产品价格及内容：
产品包含全部商业级源代码，支持后续迭代升级，另外可提供二进制可执行程序。
|类型|价格|备注|
|:--------:|:--------:|:--------:|
|源代码|3W|不含税|
|二进制程序|1W|不含税|
备注：税率：1%普票，3%、6%、13%专票
