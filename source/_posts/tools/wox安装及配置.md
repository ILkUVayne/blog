---
title: wox安装及配置
date: 2024-04-23 15:33:16
categories:
- tools
tags:
- wox
- launcher
- tools
---

## 前言

wox是一个工作简单的跨平台启动器，可以用于linux、mac和windows，本文基于windows安装和配置（主要是我的个人电脑是windows系统 \owo/ ）

## 下载

> [wox仓库地址](https://github.com/Wox-launcher/Wox)

github下载地址：[wox releases](https://github.com/Wox-launcher/Wox/releases)

在releases页面下载对应系统的软件，并安装

## 配置

右下角托盘点击wox图标打开设置 **settings**：

![setting2](/images/tools/wox/setting2.png)

或者点击 **open** 输入 **settings** 打开设置：

![setting1](/images/tools/wox/setting1.png)

设置界面如下：

![setting3](/images/tools/wox/setting3.png)

根据自己的需求进行修改即可

### 主题配置文件地址

可以根据自己的需求创建新的主题文件，然后在设置界面应用即可，需要理解一下主题文件的编写逻辑，我使用的主题文件参考：[BlurPink](https://github.com/ILkUVayne/.config/blob/main/windows/wox/conf/BlurPink.xaml)

~~~
# windows主题配置文件路径
# 11930 表示当前账号文件夹
# app-1.3.524 软件版本
C:\Users\11930\AppData\Local\Wox\app-1.3.524\Themes
~~~