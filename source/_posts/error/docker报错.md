---
title: docker报错
date: 2024-04-02 15:25:30
categories:
- error
tags:
- ubuntu
- docker
---

## 前言

docker使用中的报错汇总，以及相应解决方法

## Error response from daemon

> 环境：wsl2
> docker version: 20.10.7

docker下载镜像报错，详细错误信息

~~~bash
Error response from daemon: Get "https://registry-1.docker.io/v2/": dial tcp: lookup registry-1.docker.io on 127.0.0.53:53: read udp 127.0.0.1:48086->127.0.0.53:53: read: connection refused
~~~

解决方法：

### DNS地址有误

> 该方法适用于之前是能够正常拉取的，然后突然失败的情况。

/etc/resolv.conf 的DNS地址有误，尝试重启系统，会自动更新DNS地址