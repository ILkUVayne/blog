---
title: wsl2使用本地代理
date: 2024-03-25 13:11:26
categories:
- tools
tags:
- proxy
- wsl2
---

## 前言

> 本文代理工具使用v2rayN

当在使用wsl2的过程中需要从外网中下载更新资源（例如：github克隆代码，nvim下载更新插件等等）时，如果不使用代理会很慢或者失败。

## proxy shell

### 编写开启/关闭代理脚本

proxy

~~~shell
#!/bin/sh
hostip=$(cat /etc/resolv.conf | grep nameserver | awk '{ print $2 }')
wslip=$(hostname -I | awk '{print $1}')
# 这里填写主机代理的端口,v2rayN是10809
port=10809

PROXY_HTTP="http://${hostip}:${port}"

set_proxy(){
    export http_proxy=${PROXY_HTTP}
    export HTTP_PROXY=${PROXY_HTTP}

    export https_proxy=${PROXY_HTTP}
    export HTTPS_proxy=${PROXY_HTTP}
    git config --global http.proxy ${PROXY_HTTP}
    git config --global https.proxy ${PROXY_HTTP}
	echo "Current proxy:" $https_proxy
}

unset_proxy(){
    unset http_proxy
    unset HTTP_PROXY
    unset https_proxy
    unset HTTPS_PROXY
    git config --global --unset http.proxy
    git config --global --unset https.proxy
}

test_setting(){
    echo "Host ip:" ${hostip}
    echo "WSL ip:" ${wslip}
    echo "Current proxy:" $https_proxy
}

if [ "$1" = "set" ]
then
    set_proxy

elif [ "$1" = "unset" ]
then
    unset_proxy

elif [ "$1" = "test" ]
then
    test_setting
else
    echo "Unsupported arguments."
fi

~~~
将 **proxy** 脚本放入 **/usr/local/bin/**  中

设置可执行

~~~bash
sudo chmod 755 /usr/local/bin/proxy
~~~

### 使用

~~~bash
# 设置代理
source proxy set
# 清楚代理
source proxy unset
~~~
1
## 关闭防火墙

主机的防火墙如果没有关闭的话，wsl2中的代理可能无法生效

主机上 **Windows Defender** 设置：可以简单关闭公用网络的防火墙，在Shell中 ping 主机 ip，或运行 wget www.google.com，如果可以运行，说明上面几步成功，此时 Shell 可以链接到主机上的代理
