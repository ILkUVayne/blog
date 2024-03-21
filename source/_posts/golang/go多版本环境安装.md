---
layout: post/go
title: go多版本环境安装
date: 2024-03-19 15:17:16
categories:
- golang
tags:
- env
- golang
- install
---

## 序言

> 本文是Linux环境，其他环境可能不适用或有差别，理论上本文的配置方法也可能适用一些其他的语言，细节方面可能存在差异，例如php等

为了应对同时维护多个项目，且不同项目的go版本不一样的情况，所以采用这种配置方式


## 下载安装go

> 在go官网：https://go.dev/ 上下载对应的版本

### 1.分别下载1.22.1和1.21两个版本

~~~bash
# 1.22.1
$ wget https://go.dev/dl/go1.22.1.linux-amd64.tar.gz
# 1.21
$ wget https://go.dev/dl/go1.22.1.linux-amd64.tar.gz

# 查看
$ ll
total 130M
-rw-r--r-- 1 root root 64M Aug  8  2023 go1.21.0.linux-amd64.tar.gz
-rw-r--r-- 1 root root 66M Mar  6 01:44 go1.22.1.linux-amd64.tar.gz
~~~

### 2.解压

这里我们根据版本分别解压到 **/usr/local/go_src/22** 和 **/usr/local/go_src/21**

~~~bash
# 创建对应的解压源码文件夹
$ cd /usr/local
$ mkdir go_src && cd go_src && mkdir 22 && mkdir 21

# 返回压缩包所在文件夹，分别解压到对应文件夹
$ cd ~/golong_src
$ tar -C /usr/local/go_src/22 -xzf go1.22.1.linux-amd64.tar.gz
$ tar -C /usr/local/go_src/21 -xzf go1.21.0.linux-amd64.tar.gz

# 验证
$ cd /usr/local/go_src/21/go/bin && ./go version
go version go1.21.0 linux/amd64

$ cd /usr/local/go_src/22/go/bin && ./go version
go version go1.22.1 linux/amd64
~~~

## 多版本切换

现在我们已经有了两个版本的go,我们需要编写一个多版本切换脚本

将下面脚本添加至 **/usr/local/bin/usego** 并添加可执行权限 **sudo chmod 755 /usr/local/bin/usego** ， 用于便捷的快速切换 GO 版本(多个版本在不同的路径，使用命令脚本切换即可)

需要添加其他的版本时，按照上一步的方式下载解压到指定路径，并更新 **usego** 脚本即可

~~~shell
#!/bin/bash

version="版本未知！"

case $1 in
    "1.21"|"21")
        version="go 1.21"
        sudo ln -snf /usr/local/go_src/21/go /usr/local/go
    ;;
    "1.22"|"22")
        version="go 1.22"
        sudo ln -snf /usr/local/go_src/22/go /usr/local/go
    ;;
esac

echo "GO 版本切换至 $version "
~~~

添加 **/usr/local/go** 的环境变量

> bash添加方式：export PATH=$PATH:/usr/local/go/bin

~~~bash
# zsh 在~/.zshrc的配置文件中末尾添加下面信息
export PATH=$PATH:/usr/local/go/bin

# 添加完成后
$ source ~/.zshrc
~~~

## 验证

使用 **usego** 切换版本，并验证

~~~bash
$ cat <<EOF > test_version.go
package main
import (
    "fmt"
    "runtime"
)
func main() {
    fmt.Println(runtime.Version())
}
EOF
 
# 使用go 1.22
$ usego 22 && go version
GO 版本切换至 go 1.22 
go version go1.22.1 linux/amd64
$ go run test_version.go
go1.22.1

# 使用go 1.21
$ usego 21 && go version
GO 版本切换至 go 1.21 
go version go1.21.0 linux/amd64
$ go run test_version.go 
go1.21.0
~~~