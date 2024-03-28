---
title: 修改git提交的commit用户
date: 2024-03-28 13:52:50
categories:
- git
tags:
- git
- commit
---

## 前言

当你查看你的github contributions的时候，发现没有熟悉的绿点了，你的代码提交记录没有显示出来，查看git提交日志才发现commit user设置错了。啊啊啊，不能装逼了，好难受，必须改回来！！！

## 克隆仓库

克隆需修改仓库裸露的Git存储库

~~~bash
$ git clone --bare git@github.com:ILkUVayne/gii.git
~~~

进入裸Git仓库

~~~bash
$ cd gii.git
~~~

## 编写更新脚本

> 本文使用脚本进行修改。如果要使用git命令进行修改，请自行查找修改资料

amend.sh

~~~bash
#!/bin/sh

git filter-branch --env-filter '

OLD_EMAIL="xxx@163.com"
CORRECT_NAME="xxx"
CORRECT_EMAIL="xxx@qq.com"

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
export GIT_COMMITTER_NAME="$CORRECT_NAME"
export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
export GIT_AUTHOR_NAME="$CORRECT_NAME"
export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
~~~

将下列参数替换为自己的

- OLD_EMAIL：旧邮箱，需要被修改掉的commit 邮箱
- CORRECT_NAME： 新用户名
- CORRECT_EMAIL： 新邮箱

执行脚本,更新commit用户

~~~bash
sh ./amend.sh
~~~

## 推送远程仓库

将本地修改推送到远程仓库，并删除本地裸Git库

~~~bash
$ git push --force --tags origin 'refs/heads/*'
$ cd ..
# 删除裸露的Git存储库
$ rm -rf gii.git
~~~

完成之后可以使用 `git log` 验证或者查看github contributions是否正常显示


