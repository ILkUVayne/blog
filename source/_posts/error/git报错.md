---
title: git报错
date: 2024-05-23 16:32:49
categories:
- error
tags:
- git
---

## 前言

git使用中的报错汇总，以及相应解决方法

## Permanently added the ECDSA

执行push操作时，出现的警告

### 报错描述

~~~bash
Warning: Permanently added the ECDSA host key for IP address '[140.82.116.35]:443' to the list of known hosts.
~~~

### 解决方法

> 参考文档出处：http://www.cnblogs.com/xiangyangzhu/

将如下ip添加到 `/etc/hosts` 中即可

~~~editorconfig
140.82.116.35   github.com
~~~