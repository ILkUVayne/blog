---
title: github添加ssh
date: 2024-03-22 12:55:55
categories:
- git
tags:
- ssh
- config
- linux
- git
---

## 前言

> 文本使用的linux环境，windows环境可能有所不同，请自行查阅相关资料

由于国内访问GitHub不太稳定，时好时不好，在使用http的方式推送/拉取代码的时候就会很慢或者失败。所以就需要配置ssh，避免频繁输入账号密码的同时还能解决代码推送的问题

## 生成ssh密钥

先在本地使用命令行工具生成ssh密钥

~~~bash
# 这里comment邮箱替换为自己的
ssh-keygen -t ed25519 -C "1193055746@qq.com"
# 后续全部回车即可，成功后会在 ~/.ssh/ 生成对于密钥（可能在其他位置，会在生成过程中提示）
~~~

## 将 SSH 密钥添加到 ssh-agent

~~~bash
# 在后台启动 ssh 代理
eval "$(ssh-agent -s)"
# 将 SSH 私钥添加到 ssh-agent 生成的公钥 id_ed25519
ssh-add ~/.ssh/id_ed25519
~~~

将 SSH 公钥添加到 GitHub 上的帐户

在 **github->Settings->SSH and GPG keys** 中添加生成的ssh公钥，位于 **~/.ssh/id_ed25519.pub**

## 测试

测试是否配置成功

~~~bash
ssh -T git@github.com
Hi ILkUVayne! You've successfully authenticated, but GitHub does not provide shell access.
~~~

## 番外：一些常用命令

~~~bash
# 设置全局username
$ git config --global user.name "username"
# 设置全局user email
$ git config --global user.email "1193055746@qq.com"
# 清除.idea缓存
git rm -r --cached .idea
# 制作一个裸露的Git存储库 git@github.com:ILkUVayne/gii.git 替换为需要制作的仓库
$ git clone --bare git@github.com:ILkUVayne/gii.git
~~~

## FAQ

### fatal: Could not read from remote repository.

如果正确配置了ssh,可能是网络问题，我的解决办法是重启电脑（遇到过两次，都是这么解决的），如果任然无法解决，可能是默认22端口不可用

### ssh: connect to host github.com port 22: Connection timed out

可能是22端口不可用，尝试换个端口

~~~bash
ssh -T -p 443 git@ssh.github.com
Hi ILkUVayne! You've successfully authenticated, but GitHub does not provide shell access.
~~~

修改端口

~~~bash
# 创建config
vim ~/.ssh/config
~~~

添加如下配置到config中

~~~bash
# 修改端口号为443
Host github.com
User git
Hostname ssh.github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_ed25519
Port 443
~~~
