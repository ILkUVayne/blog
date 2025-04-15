---
title: github Actions报错问题
date: 2025-04-15 18:05:42
categories:
  - devops
tags:
  - k3s
  - github Actions
  - linux
---

## 前言

> ps: 集群环境: k3s

GitHub Actions (CI/CD) 报错及解决方案汇总。

## 0x01

### 报错信息

这段报错信息是在github的Actions workflows的build流程中的报错信息

~~~
error: error validating "deployment.yaml": error validating data: failed to download openapi: the server has asked for the client to provide credentials; if you choose to ignore these errors, turn validation off with --validate=false
~~~

该错误表明 Kubernetes 客户端 (kubectl) 无法通过身份验证访问集群的 OpenAPI 模式验证文件，导致 YAML 文件验证失败。

### 解决方案

- 登录到集群(k3s)所在服务器，检查是否能够访问集群

~~~bash
kubectl get pods                                     
E0415 17:37:06.276711 2274630 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0415 17:37:06.279792 2274630 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0415 17:37:06.282671 2274630 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0415 17:37:06.285694 2274630 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
E0415 17:37:06.288745 2274630 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
error: You must be logged in to the server (the server has asked for the client to provide credentials)
~~~

通过报错信息能够知道，kubeconfig 文件中引用的客户端证书已过期。

- 检查config(~/.kube/config)是否过期。

~~~bash
# notAfter 证书过期时间（在此时间后使用证书会报错）
kubectl config view --raw -o jsonpath='{.users[*].user.client-certificate-data}' | base64 -d | openssl x509 -enddate -noout
notAfter=Mar 16 04:44:34 2025 GMT
~~~

证书过期时间为 `2025.3.16 04:44:34` ，当前时间为 `2025.4.15`，证实客户端证书已过期。

- 更新证书

~~~bash
# 重启 K3s 服务以触发证书自动更新（推荐）
sudo systemctl restart k3s
# 验证新证书生效
kubectl get nodes
# 查看证书过期时间
kubectl config view --raw -o jsonpath='{.users[*].user.client-certificate-data}' | base64 -d | openssl x509 -enddate -noout
notAfter=Apr 15 09:49:54 2026 GMT
~~~

- 更新github Settings配置

更新 `Settings->Secrets adn variables->Actions` 中对应的kubeConfig密钥。rebuild action workflow验证即可。

