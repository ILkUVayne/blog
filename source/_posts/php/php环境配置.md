---
title: php环境配置
date: 2024-04-28 15:19:28
categories:
- php
tags:
- env
- php
- install
- linux
---

## 安装依赖

> 依赖根据不同php版本,可能存在缺失，依据编译源码时的报错提示补充即可

~~~shell
sudo apt install -y gcc make openssl curl libbz2-dev libxml2-dev libjpeg-dev libpng-dev libfreetype6-dev libzip-dev libssl-dev libcurl4-openssl-dev libonig-dev zlib1g-dev

sudo ln -s /usr/lib/x86_64-linux-gnu/libssl.so /usr/lib

cd /usr/include
sudo ln -s x86_64-linux-gnu/curl
~~~

## 编译安装PHP

> 后续的扩展按照自己的需求安装即可

安装 PHP 使用编译安装的方式， 使用编译安装便于后续调试扩展增加。开发过程中遇到 `segmentation fault` 也可快速开启 `debug` 模式进行调试

~~~shell
git clone https://github.com/php/php-src.git
 
cd php-src
 
git checkout ${PHP_VERSION}
 
./buildconf
~~~

> 编译新版本或者 调整编译参数则需要先清除编译信息

~~~shell
make clean
~~~

> 如果需要编译安装其他扩展需查看官方配置选项，核心配置选项列表 https://www.php.net/manual/zh/configure.about.php

~~~shell
# 不同的版本可能使用的配置命令不一样
# 在php-src根目录执行 ./configure --help 命令可以查看所有的配置信息进行更改
# 例如：我想要查询gmp扩展怎么配置，使用 ./configure --help | grep gmp

./configure \
  --prefix=/usr/local/php/7.3 \
  --with-config-file-path=/etc/php/7.3 \
  --with-config-file-scan-dir=/etc/php/7.3/conf.d \
  --with-curl \
  --with-mysqli \
  --with-openssl \
  --with-pdo-mysql \
  --with-gd \
  --enable-fpm \
  --enable-bcmath \
  --enable-xml \
  --enable-zip \
  --enable-mbstring \
  --enable-sockets
  
  # sudo make -j4 && sudo make install 两条也可以命令一起执行
  
  sudo make -j4
  
  sudo make install
~~~

为 PHP 添加专属自定义 PATH 用于PHP可执行程序管理以及便于多版本切换，在 /etc/profile 文件末尾增加

> zsh 则再~/.zshrc 文件中添加

~~~shell
export PATH=/usr/local/php/bin:$PATH
~~~

如果需要 sudo 支持需要编辑 sudo 的 secure_path 在末尾追加

~~~shell
sudo visudo

...
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/usr/local/php/bin"
...
~~~

### 新增扩展

为已有的php编译安装其他扩展,有两种情况

- php官方集成的扩展，但是在编译php时没有启用这个扩展

这里以gd库为例：

编译安装

~~~shell
# 1. 进入php源码的扩展目录
cd php-src/ext/gd

# 生成 config.m4（部分版本需要手动拷贝配置文件,如果已存在，跳过这一步）
cp config0.m4 config.m4

# 生成编译文件
# 如果提示phpize命令不存在，可以使用绝对地址执行命令 /usr/local/php/bin/phpize
# 或者将 export PATH=/usr/local/php/bin/phpize:$PATH 添加到 /etc/profile 或者 ~/.zshrc
phpize

# 配置编译参数
./configure --with-php-config=$(which php-config)

# 编译安装
sudo make && sudo make install
~~~

启用扩展

~~~shell
# 在 /etc/php/{php version}/conf.d 创建扩展配置启用
cd /erc/php/8.2/conf.d
sudo vim 82-gd.ini
~~~

添加如下配置：

~~~
extension=gd.so
~~~

- php官方尚未集成的扩展

手动下载扩展包或者git clone扩展源码进行安装，然后具体的编译安装过程同上

## 多版本切换

将下面脚本添加至 `/usr/local/bin/usephp` 并添加可执行权限 `sudo chmod +x /usr/local/bin/usephp` ， 用于便捷的快速切换 PHP 版本(编译多个版本再不同的路径，使用命令脚本切换即可)

~~~shell
#!/bin/bash
 
version="版本未知！"
 
case $1 in
    "7.2"|"72")
        version="7.2"
        sudo ln -snf /usr/local/php/7.2/bin /usr/local/php/bin
    ;;
    "7.3"|"73")
        version="7.3"
        sudo ln -snf /usr/local/php/7.3/bin /usr/local/php/bin
    ;;
    "7.4"|"74")
        version="7.4"
        sudo ln -snf /usr/local/php/7.4/bin /usr/local/php/bin
    ;;
    "8.0"|"80")
        version="8.0"
        sudo ln -snf /usr/local/php/8.0/bin /usr/local/php/bin
    ;;
esac
 
echo "PHP 版本切换至 $version "
~~~

## 安装SWOOLE扩展
安装 SWOOLE 有很多种方式，详情参考 SWOOLE 官方文档。 此处使用编译安装的方式进行安装同理便于后续开启 debug、调整编译参数 编译选项

~~~shell
git clone https://github.com/swoole/swoole-src.git
 
cd swoole-src
 
git checkout ${SWOOLE_VERSION}
 
phpize
 
./configure \
--enable-openssl \
--enable-http2
 
make
 
sudo make install
~~~
将扩展添加至配置中

~~~shell
echo "extension=swoole.so" > /etc/php/7.3/conf.d/20-swoole.ini
~~~

检查 swoole 扩展是否被正确加载

~~~shell
php --ri swoole
~~~

## 安装sdebug扩展

~~~shell
git clone https://github.com/swoole/sdebug.git
 
cd sdebug
 
./rebuild.sh
 
echo "zend_extension=xdebug.so" > /etc/php/7.3/conf.d/99-sdebug.ini
~~~

检查 sdebug 扩展是否被正确加载

~~~shell
php --ri sdebug
~~~

~~~shell
# 添加一个 Xdebug 节点
{ echo "[Xdebug]"; \
# 启用远程连接
echo "xdebug.remote_enable = 1"; \
# 这个是多人调试，但是现在有些困难，就暂时不启动
echo ";xdebug.remote_connect_back = On"; \
# 自动启动远程调试
echo "xdebug.remote_autostart  = true"; \
# 这里 host 可以填前面取到的 IP ，也可以填写 host.docker.internal 。
echo "xdebug.remote_host = host.docker.internal"; \
# 这里端口固定填写 9000 ，当然可以填写其他的，需要保证没有被占用
echo "xdebug.remote_port = 9000"; \
# 这里固定即可
echo "xdebug.idekey=PHPSTORM"; \
# 把执行结果保存到 99-sdebug-enable.ini 里面去
} | tee /etc/php/7.3/conf.d/99-sdebug-enable.ini \
~~~

## 安装tideways_xhprof扩展
获取 PHP 程序详细运行日志函数调用链，分析程序执行过程快速定位程序性能问题

~~~shell
git clone https://github.com/tideways/php-xhprof-extension.git
 
cd php-xhprof-extension
 
phpize
 
./configure
 
make
 
sudo make install
 
echo "extension=tideways_xhprof.so" > /etc/php/7.3/conf.d/99-tideways_xhprof.ini
~~~

检查 tideways_xhprof 扩展是否被正确加载

~~~shell
php --ri tideways_xhprof
~~~

## 安装wrk
为什么需要安装 wrk， 我们开发中使用的是 Swoole 进行驱动 Swoole 的 Worker 是多进程单线程模型协程之间可能会有变量污染进程阻塞等问题，但是这些问题在单独测试某一个接口时是很难被发现的，所以我们在开发过程中需要对开发接口进行一定的压力测试来校验接口的健壮性、下逻辑的正确性以及简单是测试接口的性能。

docker 环境安装使用 Docker Hub

~~~shell
docker pull skandyla/wrk
docker run --rm -v $(pwd):/data skandyla/wrk -s \
  script.lua  -t5 -c10 -d30  https://www.example.com
wrk 参数列表
-c, --connections: total number of HTTP connections to keep open with
                   each thread handling N = connections/threads
 
-d, --duration:    duration of the test, e.g. 2s, 2m, 2h
 
-t, --threads:     total number of threads to use
 
-s, --script:      LuaJIT script, see SCRIPTING
 
-H, --header:      HTTP header to add to request, e.g. "User-Agent: wrk"
 
    --latency:     print detailed latency statistics
 
    --timeout:     record a timeout if a response is not received within
                   this amount of time.
~~~

wrk 不建议自行编译安装， 使用 docker 就足够了
