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

git 使用中的一些命令或技巧汇总。

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

- 其他

~~~bash
# 查看tag列表
git tag

# 匹配tag
git tag -l "v1.*"
~~~

## rm

- 文件从暂存区域移除

文件从暂存区域移除，但仍然希望保留在当前工作目录中。例如：我忘记添加`.gitignore`，把某些不想的提交的文件推送到了仓库（如`.idea`），此时就需要把对应文件或文件夹从暂存区移除，再提交到远程仓库。

~~~bash
# git rm --cached file_path
# 若是文件夹则需要使用 -r 递归删除
git rm -r --cached .idea/
git commit -m "xxxx"
git push
~~~