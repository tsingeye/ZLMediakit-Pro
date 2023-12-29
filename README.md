# 一、概述
zlmediakit pro版本⽀持基于ffmpeg的转码能⼒，在开源版本强⼤功能的基础上，新增⽀持如下能⼒：
1. ⾳视频间任意转码(包括h265/h264/opus/g711/aac等)；
2. 基于配置⽂件的转码，⽀持设置⽐特率，codec类型等参数；
3. 基于http api的动态增减转码，⽀持设置⽐特率，分辨率倍数，codec类型、滤镜等参数；
4. ⽀持硬件、软件⾃适应转码;
5. ⽀持按需转码，有⼈观看才转码;
6. ⽀持负载过⾼时，转码主动降低帧率且不花屏;
7. ⽀持滤镜，⽀持添加osd⽂本以及logo⻆标等能⼒。


# 二、转码实现原理
- 视频转码原理
- 音频转码原理

# 三、使用方法
⽬前zlmediakit pro转码能⼒⽀持两种使⽤⽅式：
第⼀种是基于配置⽂件⽅式，在设置好配置⽂件后，所有流都⽀
持转码为⽬标编码格式直播流。
第⼆种模式基于http api⽅式，此⽅式更灵活，功能强⼤，可以指定更多转码相关
参数。
## 3.1 基于配置文件的转码
```ini
[transcode]
#转码stream_id后缀,为空时关闭转码
suffix=
#默认转码视频⽬标codec，⽀持H264/H265/JPEG/copy
vcodec=H264
#默认转码⾳频⽬标codec，⽀持mpeg4-generic/PCMA/PCMU/opus/copy
acodec=mpeg4-generic
#是否开启ffmpeg⽇志
enable_ffmpeg_log=0
# h264解码器⽩名单
decoder_h264=h264_cuvid,h264_qsv,h264_videotoolbox,h264_nvmpi,h264_bm,libopenh264
# h265解码器⽩名单
decoder_h265=hevc_cuvid,hevc_qsv,hevc_videotoolbox,hevc_nvmpi,hevc_bm
# h264编码器⽩名单
encoder_h264=h264_nvenc,h264_qsv,h264_videotoolbox,h264_nvmpi,h264_bm,libx264,libopenh2
64
# h265编码器⽩名单
encoder_h265=hevc_nvenc,hevc_qsv,hevc_videotoolbox,hevc_nvmpi,hevc_bm,libx265
```

在上述配置⽂件中，如果⽤户配置好 suffix ，那么zlmediakit将统⼀把所有直播流转码为⽬标编码格式，⽤户通
过访问新的流地址即可确保为预期编码格式视频。
例如源视频地址为：rtmp://127.0.0.1/live/test, 那么转码后地址即为：rtmp://127.0.0.1/live/test_H264。
当配置⽂件修改为 suffix=null 时，转码后流会直接替换原始流(不会有 _suffix 后缀)；替换模式下，建议
rtsp.directProxy/rtmp.directProxy 都设置为0。
如果源视频编码格式与⽬标编码格式⼀致，那么zlmediakit为了确保性能最优，将直接拷⻉流数据(不会编码)。
基于配置⽂件⽅式的转码使⽤最简单，可以使⽤于安防⾏业H265视频⽆法webrtc/mse播放的场景。

## 3.2 基于Http API的转码
zlmediakit同时还提供基于http api的转码⽅式，这种⽅式⽀持的功能更强⼤，使⽤更灵活，同时⽀持⼀个流转码
成多个⽬标流(⽐如说不同分辨率的场景)。

- 请求地址：/index/api/setupTranscode
- 请求参数： 
