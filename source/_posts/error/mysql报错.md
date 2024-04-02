---
title: mysql报错
date: 2024-04-02 16:21:44
categories:
- error
tags:
- mysql
- docker
---

## 前言

mysql使用中的报错汇总，以及相应解决方法

## lost connection to mysql server

> 环境：wsl2
> docker version: 20.10.7
> mysql: 8.0

navicat连接数据库失败的详细错误信息：

~~~bash
2013 - lost connection to mysql server at 'reading initial communication packet', system error: 0 "internal error/check (not system error)"
~~~

解决方法如下：

### DNS地址有误

> 该方法适用于之前是能够正常连接的，然后突然连接失败的情况。

/etc/resolv.conf 的DNS地址有误，尝试重启系统，会自动更新DNS地址，重新连接即可。

### 添加连接权限

如果为8.0及以上版本；需要注意，该版本密码认证机制已经升级，有些客户端未能兼容，请使用新的认证方式修改Mysql密码，还有就是，所登录的用户是否允许任意主机连接

查看mysql数据库中的user表信息：

~~~bash
# exec进入容器， ceca1ab50e6e 为mysql容器id,使用 docker ps 查询
$ docker exec -it ceca1ab50e6e bash
# 命令行登录mysql
$ mysql -uroot -p

# 切换数据库
mysql> use mysql;
# 查询user信息
# 仅允许本地连接：root@localhost
# 允许任意连接：root@%
# 如果发现为：root@localhost， 不要直接修改此表，可以新建一个用户，并赋予权限
mysql> select user,host from user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| root             | %         |
| user             | %         |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+

# 新建用户，其中的 user1 替换为自己实际要创建的登录用户和密码
mysql> CREATE USER 'user1'@'%' IDENTIFIED WITH mysql_native_password BY 'user1';
# 赋予权限，使用新建的用户进行登录即可
mysql> GRANT ALL PRIVILEGES ON *.* TO 'user1'@'%';
~~~
