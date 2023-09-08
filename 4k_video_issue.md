
# 4k 视频播放问题

## 问题描述

目前（20230905）收到了多个在播放过程中有问题的视频源

1. [video_unplayable.mp4] syncronize 时间过长，导致 gstreamer 假死
2. [video_shutdown.mp4] 播放在2分钟内不定时卡死
3. [video_stuck.mp4] 播放数分钟后视频逐渐变卡顿

可以在 [https://drive.google.com/drive/folders/1Iu1HFzDx-8YJ1U4WTyGQxe41lPF1J0Yv](https://drive.google.com/drive/folders/1Iu1HFzDx-8YJ1U4WTyGQxe41lPF1J0Yv?usp=sharing) 中下载

## 问题排查

排查命令示例

```shell
# 通用示例
GST_DEBUG_DUMP_DOT_DIR=xxx GST_DEBUG_FILE=xxx \
gst-launch-1.0 filesrc location=/media/debian/4A21-0000/video/video/test_animal_4k60fps.mp4 ! qtdemux name=demux \
demux.video_0 ! queue ! h264parse ! decodebin ! videoconvert ! videoscale ! glimagesink \
demux.audio_0 ! queue !  aacparse ! avdec_aac ! audioconvert ! audioresample ! alsasink \
--gst-debug-level=3 --gst-debug-no-color

# 软解测试
# fpsdisplaysink 在 gst-plugins-bad 包中
gst-launch-1.0 filesrc location=/media/debian/4A21-0000/video/problem/video_shutdown.mp4 ! qtdemux ! queue ! h264parse ! avdec_h264 ! videoconvert ! videoscale ! fpsdisplaysink video-sink=glimagesink sync=false


# 使用 ffmpeg 检查文件本身有何问题
ffmpeg -v warning -i .\video_normal.mp4 -f null  - 2>error.log
```

### GStreamer 假死

无法解视频流，但是解音频流没有问题，是视频流问题

### video_shutdown.mp4 播放卡死

解视频流会在两分钟内卡死，而解音频流并没有影响，是视频流的问题

使用 ffmpeg 探测错误

```shell
ffmpeg -v error  -i filename.mp4 -f null - 2>error.log
```

该文件有如下报错

```log
[null @ 00000204e213b340] Application provided invalid, non monotonically increasing dts to muxer in stream 0: 4121 >= 4121
```

根据[FFmpeg - What does non monotonically increasing dts mean?](https://stackoverflow.com/questions/46231348)中的回答， null 类型仍然会检查时间戳的单调性，这是个历史遗留问题，并不意味着错误的发生。

使用 fakesink，sync=false 未见异常
使用 glimagesink，sync=false 未见异常
使用 fakesink，sync=true 未见异常
**使用 glimagesink，sync=true 异常**

正常播放时：gst-launch-1.0 占用率60%，xorg 占用率20%
卡死时：gst-launch-1.0 的 cpu 占用率约34%，xorg 占用率 为0
未见明显的内存变化

--gst-debug-level=3 未见相关日志
--gst-debug-level=4 未见相关日志
--gst-debug-level=5 难以调试【todo】

可以观察到，日志等级不超过 4 时，glimagesink 卡住后不再打印日志

注意到出错视频的分辨率为 3996x2160，试用其他为此分辨率的源，播放正常，排除分辨率问题

附上 pipeline![0.00.00.825889235-gst-launch.PAUSED_PLAYING.dot.png](https://cdn.nlark.com/yuque/0/2023/png/21774188/1693908764846-f553325d-49c0-4d31-86ed-c9558ff5bf7c.png#averageHue=%23f8f6f6&clientId=ua73e26dd-3f7a-4&from=drop&id=u699da37f&originHeight=492&originWidth=5648&originalType=binary&ratio=1&rotation=0&showTitle=false&size=202387&status=done&style=none&taskId=ud2bb9ea3-7521-4d0f-962d-a74279adfe0&title=)

使用 ffprobe 观察播放正常的视频与问题视频，那么剩下的不一样的地方是，问题视频的 tbn 为 90k

DTS(Decoding Time Stamp, 解码时间戳)，表示压缩帧的解码时间。

PTS(Presentation Time Stamp, 显示时间戳)，表示将压缩帧解码后得到的原始帧的显示时间。

tbn 90k 是非常常见的 timebase ，没问题。

检索`bin gstbin.c:4074:bin_query_duration_fold:<glimagesinkbin0> got duration 171898000000`可以看出，自276508行开始，日志里就开始循环出现同样的记录，并且间隔时间非常短，大约100行日志就会重复一次。gstreamer的时间戳40s，减去前面的2s启动时间，正好是程序卡死的时间38s。

自 276413 行之后，线程就一直卡在 0x2ac405a000 中跳不出去

gobject 程序调试

```shell
set env LD_PRELOAD /home/debian/Desktop/gobject-list/libgobject-list.so
r filesrc location=/media/debian/4A21-0000/video/problem/video_shutdown.mp4 ! qtdemux ! queue ! h264parse ! omxh264dec ! glimagesink
```
