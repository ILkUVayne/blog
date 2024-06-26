---
title: linux重定向2>&1
date: 2024-04-12 14:09:20
categories:
- linux
tags:
- linux
---

## Linux中 0 1 2 代表的意义

在Linux系统中0 1 2是一个文件描述符

| 名称     | 代码  |
|--------|-----|
| 标准输入   | 0   |
| 标准输出   | 1   |
| 标准错误输出 | 2   |

## 2>&1的意义

表示将标准错误输出重定向到标准输出，'>&'是一个整体，不能单独使用'>','2>1'表示将错误输出重定向到名为'1'的文件中

## 具体用法

当我们需要对有标准错误输出的命令进行过滤查找等操作时，就可以先使用 `2&>1` 将标准错误输出重定向到标准输出,再执行对于操作

~~~bash
# 查询视频元数据，因为没有提供输出文件，会报错即标准错误输出
$ ffmpeg -i /mnt/f/video/1251203951-1-192.mp4 -hide_banner
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '/mnt/f/video/1251203951-1-192.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf58.29.100
    description     : Packed by Bilibili XCoder v2.0.2
  Duration: 00:03:34.74, start: 0.000000, bitrate: 1230 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709), 1280x720 [SAR 1:1 DAR 16:9], 1087 kb/s, 30 fps, 30 tbr, 16k tbn, 60 tbc (default)
    Metadata:
      handler_name    : VideoHandler
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 132 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
At least one output file must be specified

# 直接过滤，不会有变化
$ ffmpeg -i /mnt/f/video/1251203951-1-192.mp4 -hide_banner | grep -m 1 "Video"
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '/mnt/f/video/1251203951-1-192.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf58.29.100
    description     : Packed by Bilibili XCoder v2.0.2
  Duration: 00:03:34.74, start: 0.000000, bitrate: 1230 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709), 1280x720 [SAR 1:1 DAR 16:9], 1087 kb/s, 30 fps, 30 tbr, 16k tbn, 60 tbc (default)
    Metadata:
      handler_name    : VideoHandler
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 132 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
At least one output file must be specified

# 重定向错误输出，正常过滤
$ ffmpeg -i /mnt/f/video/1251203951-1-192.mp4 -hide_banner 2>&1 | grep -m 1 "Video"
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709), 1280x720 [SAR 1:1 DAR 16:9], 1087 kb/s, 30 fps, 30 tbr, 16k tbn, 60 tbc (default)
~~~
