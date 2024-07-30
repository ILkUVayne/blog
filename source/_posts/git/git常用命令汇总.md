---
title: git常用命令汇总
date: 2024-07-30 15:54:01
categories:
  - git
tags:
  - git
  - logs
---

## 前言

git 使用中的一些命令技巧汇总。

## tag

- 添加一个tag

~~~bash
git tag -a v1.0.0 -m "添加tag"
~~~

- 推送创建的tag到远程仓库

~~~bash
# 推送某个tag
git push origin v1.0.0

# 推送所有tag
git push origin --tags
~~~
