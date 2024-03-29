---
title: apt命令报错
date: 2024-03-29 17:02:33
categories:
- error
tags:
- ubuntu
- apt
---

## 前言

> 本文的Ubuntu版本: Ubuntu 20 LTS

在使用ubuntu的过程中，我们一般都会使用官方的apt命令工具来管理软件包和库的安装、更新和卸载，由于各种原因，使用过程中会产生报错。为了方便查找，在这个做统一归纳汇总。

## 403 Forbidden

执行 `sudo apt update` 命令更新apt包时，报错 **403 Forbidden**

### 报错描述

~~~bash
$ sudo apt update
Hit:1 http://mirrors.aliyun.com/ubuntu focal InRelease
Hit:2 http://mirrors.aliyun.com/ubuntu focal-updates InRelease
Hit:3 http://mirrors.aliyun.com/ubuntu focal-backports InRelease
Hit:4 http://mirrors.aliyun.com/ubuntu focal-security InRelease
Hit:5 https://download.docker.com/linux/ubuntu focal InRelease
Get:6 https://deb.nodesource.com/node_18.x focal InRelease [4583 B]
Err:7 http://ppa.launchpad.net/dawidd0811/neofetch/ubuntu focal InRelease
  403  Forbidden [IP: 185.125.190.52 80]
Hit:8 http://ppa.launchpad.net/ondrej/php/ubuntu focal InRelease
Reading package lists... Done
E: Failed to fetch http://ppa.launchpad.net/dawidd0811/neofetch/ubuntu/dists/focal/InRelease  403  Forbidden [IP: 185.125.190.52 80]
E: The repository 'http://ppa.launchpad.net/dawidd0811/neofetch/ubuntu focal InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
~~~

从报错信息中能够发现：neofetch软件的仓库更新失败了，报了403

### 解决方法

解决方法也很简单，删除403报错的源即可

~~~bash
$ cd /etc/apt/sources.list.d
# 删除403报错的源
$ sudo rm -rf dawidd0811-ubuntu-neofetch-focal.list
# 重新更新即可
$ sudo apt update
~~~