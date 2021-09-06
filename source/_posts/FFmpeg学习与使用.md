---
title: FFmpeg学习与使用
date: 2021-07-25 11:16:39
categories: 
- 视频
tags:
- 视频
---


## 目的

本节主要记录下FFmpeg的使用方法以及如何用于视频项目上作一个简单的介绍

## 简介
FFmpeg是一款多媒体处理工具，内部包含了解码、编码、转码、解密的操作命令。正式由于FFmpeg太强大，目前大部分流媒体服务都会或多或小使用到FFmpeg的功能，所以我们有必要学习下FFmpeg的使用指令。

## 常用命令以及解析

### 主要成分：

1. libavformat：用于各种音视频封装格式的生成和解析，包括获取解码所需信息以生成解码上下文结构和读取音视频帧等功能；
1. libavcodec：用于各种类型声音/图像编解码；
1. libavutil：包含一些公共的工具函数；
1. libswscale：用于视频场景比例缩放、色彩映射转换；
1. libpostproc：用于后期效果处理；
1. ffmpeg：该项目提供的一个工具，可用于格式转换、解码或电视卡即时编码等；
1. ffsever：一个 HTTP 多媒体即时广播串流服务器；
1. ffplay：是一个简单的播放器，使用ffmpeg 库解析和解码，通过SDL显示；


### ffprobe
例子
```
ffprobe -rtsp_transport tcp -i "rtsp://10.128.184.34:554/01?Short=1&Token=vEKOlIYbbCFseXHTjO3kbtEKZGZ4eyRExIkPU+yWWPA=&DomainCode=6a7c1a077a004406a344a1f260d86e41&UserId=6&"
```

-rtsp_transport tcp/udp：选择以TCP还是UDP方式打开 
·-i filename：指定输入文件名，rtsp、rtmp、摄像头地址等。
-show_format filename：展示格式
-show_frames filename：显示帧信息

### ffplay

播放文件
```
ffplay xxx.mp3 -ast 1 -loop 10
```
-loop num：循环次数

-[ast|vst] 1：选择播放[音频|视频]

### FFmpeg

FFmpeg参数太多，以下是我举例了项目中实际用到的命令。

```
ffmpeg -re -loglevel quiet {[decoder]}  {[tcp]} -i "{[rtsp]}"
```
·-re：代表按照帧率发送，尤其在作为推流工具的时候一定要加入该参数，否则ffmpeg会按照最高速率向流媒体服务器不停地发送数据

-loglevel [debug|quiet|error]：日志级别记录

·-i filename：指定输入文件名，rtsp、rtmp、摄像头地址等。

截图
```
ffmpeg  -re -loglevel quiet {[tcp]} -i \"{[rtsp]}\" -an {[size]}  -vframes 1 -y -f image2 {[pattern]}.jpg
```
-an：视频静音

·-y：覆盖已有文件。

·-f fmt：指定格式（音频或者视频格式）。


推流转码成h264
```
ffmpeg -re -loglevel debug  -rtsp_transport tcp -i "rtsp://10.128.184.34:554/01?Short=1&Token=yWSkzfBj71Yja67n9Y65SXDN97TCCOzBveNwOB53Lh8=&DomainCode=6a7c1a077a004406a344a1f260d86e41&UserId=6&" -an -s 1280x720  -vcodec smart_h264 -f flv rtmp://192.168.193.168:1935/hls/7a92e703d492aac22614ba670c85018f?secret=035c73f7-bb6b-4889-a715-d9eb2d1925ccSimp
```

-analyzeduration：设置码流分析时间

-probesize：探测时长，这个设置的时间越长，视频打开得越慢

·-vcodec [codec|smart_h264]：强制使用codec编解码方式（'copy'代表不进行重新编码）,smart_h264实现h264编码

·-s size：指定分辨率（320×240）。


```
{[ffmpeg]} -loglevel error {[decoder]} -i {[rtsp]} {[copy]} {[encoder]} -an -threads 10 -tune zerolatency -crf {[crf]} -g 1 -r {[frameRate]} -preset ultrafast -vcodec smart_h264 -f flv rtmp://192.168.193.168:1935/hls/7a92e703d492aac22614ba670c85018f?secret=035c73f7-bb6b-4889-a715-d9eb2d1925ccSimp
```


crf num：
为恒定质量（无比特率目标）和受限质量（最大比特率目标）模式设置质量/大小折衷。有效范围是0到63，数字越大表示质量越低，输出大小越小。仅在设置时使用；默认情况下，仅使用比特率目标

threads num：选定的编解码器实现支持多线程，则设置要使用的线程数。

tune [zerolatency|fastdecode|psnr|ssim]：主要配合视频类型和视觉优化的参数。zerolatency零延迟，用在需要非常低的延迟的情况下，比如电视电话会议的编码；fastdecode可以快速解码的参数； psnr为提高psnr做了优化的参数；ssim为提高ssim做了优化的参数； 

preset type：预设类型。

r num：帧率

g size：设置GOP的大小


## 项目中如何是使用

由于ffmpeg太强大，在视频网关的项目中，我们经常会使用到ffmpeg。主要使用到有两方面，一个是截图，一个是将h265的视频数据格式转码成h264或者更改视频的分辨率功能等。为什么在项目中会使用到FFmpeg呢，其实像海康和大华第三方的厂商都会有对应的sdk,但是像截图之类的功能不能很好支持各种分辨率的截图，所以喔们考虑到使用FFmpeg来结合起来，以下是关于项目的流程图。

![FFmpeg使用流程](/images/video/流程图.jpg)

其实在项目上，我们一般会使用FFmpeg来进行截图，如上述说因为能够定制化配置截图得大小以及色差得调节等，而第三方厂商sdk一般都是默认尺寸，扩展性不高（如海康和大华）。而且，使用FFmpeg得扩展性会比较高。但是FFmpeg并不是万能，对于有一些私有的RTSP等可能也会存在失败的情况，所以在项目中如果FFmpeg截图不成功都会采用对应厂商sdk进行兜底截图，保证图片能够生成并且可用。

## 参考

1. [FFmpeg官网](http://www.ffmpeg.org/documentation.html)