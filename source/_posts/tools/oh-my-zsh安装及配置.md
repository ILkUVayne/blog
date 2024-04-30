---
title: oh-my-zsh安装及配置
date: 2024-04-30 15:00:50
categories:
- tools
tags:
- zsh
- linux
- ubuntu
---

## 前言

Oh My Zsh是一个开源、社区驱动的框架，用于管理您的Zsh配置。本文主要用于归纳一些常用插件以及配置

## 安装

### zsh

> 以ubuntu为例,其他的发行版本请自行查阅对应的软件下载方式

大部分linux默认的shell是Bash，我们要使用Oh My Zsh的话，需要先安装zsh.

~~~bash
sudo apt update
sudo apt install zsh
# 切换到zsh
zsh
~~~

### ohmyzsh

> 本文档只做参考,可能未能及时更新,下载链接以官方仓库的文档为准

使用官方的Github仓库: [ohmyzsh](https://github.com/ohmyzsh/ohmyzsh) 中的下载方式下载

~~~bash
# curl
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# wget
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# fetch
sh -c "$(fetch -o - https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
~~~

如果您在印度或中国这样的国家，则可能需要使用此URL来阻止raw.githubusercontent.com

~~~bash
# curl
sh -c "$(curl -fsSL https://install.ohmyz.sh/)"

# wget
sh -c "$(wget -O- https://install.ohmyz.sh/)"

# fetch
sh -c "$(fetch -o - https://install.ohmyz.sh/)"
~~~

安装完成之后,会在 `~/.zshrc` 生成配置文件 `.zshrc`, 后续的配置都将使用该文件

## 配置

所有修改的配置都需要重新执行 `source ~/.zshrc` 才能生效

### 主题

ohmyzsh内置了很多主题: [themes地址](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes),替换配置文件中的`ZSH_THEME`的值为你想要的主题名字即可.

如果官方提供的主题无法满足需求,可以查看外部主题:[External-themes](https://github.com/ohmyzsh/ohmyzsh/wiki/External-themes),是由第三方实现的主题,参考对应主题仓库的文档安装,可能需要安装一些特殊的字体或者图标,同样替换配置文件中的`ZSH_THEME`的值为你想要的主题名字即可.

### 插件

下载完插件后,需要在配置文件中 `.zshrc` 的 `plugins` 中添加对应的插件名称

#### 自动补全插件

~~~bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
~~~

#### 代码高亮插件

~~~bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
~~~

### 其他配置

> 一般放在配置文件末尾

- kubectl命令补全

~~~shell
source <(kubectl completion zsh)
~~~

- 在git检出和提交时都不自动转换行尾

~~~shell
git config --global core.autocrlf false
~~~