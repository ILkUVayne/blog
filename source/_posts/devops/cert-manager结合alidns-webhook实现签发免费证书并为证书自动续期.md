---
title: cert-manager结合alidns-webhook实现签发免费证书并为证书自动续期
date: 2024-03-17 21:44:09
categories:
- devops
tags:
- cert-manager
- alidns-webhook
- k3s
- linux
---

## 前提条件

- 本文是使用的k3s，且使用ingress-nginx(需要手动安装)
- 禁用了traefik，使用traefik可能会失败或者实现方式有所不同，请自寻查找资料
- 本文使用阿里云服务器，其他服务器需要自寻查找相应的webhook资料

## 配置cert-manager

### 1.创建cert-manager命名空间:

~~~bash
$ kubectl create namespace cert-manager
~~~

### 2.添加cert-manager Chart

~~~bash
$ helm repo add jetstack https://charts.jetstack.io
~~~

### 3.获取cert-manager Chart的最新信息

~~~bash
$ helm repo update
~~~

> cert-manager的版本需要和Kubernetes版本保持兼容。如果版本不匹配会失败。关于cert-manager和Kubernetes版本的对应关系，请参见[Supported Releases](https://cert-manager.io/docs/releases/)。

~~~bash
# installing CRDs with kubectl
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml

# installing cert-manager
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.0 \
 # --set installCRDs=true
~~~

### 4.验证

~~~bash
kubectl -n cert-manager get pods
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-6466f8f444-jrspl             1/1     Running   0          3m53s
cert-manager-cainjector-57ff68fc8-tp4n8   1/1     Running   0          3m53s
cert-manager-webhook-7bb75bd8dd-z667k     1/1     Running   0          3m53s
~~~

## 配置alidns-webhook

> 仓库地址：https://github.com/wjiec/alidns-webhook

### 1.安装alidns-webhook

~~~bash
# llyy替换为自己的域名
helm upgrade --install alidns-webhook alidns-webhook \
    --repo https://wjiec.github.io/alidns-webhook \
    --namespace cert-manager --create-namespace \
    --set groupName=acme.llyy.com
~~~

### 2.配置alidns-secret

配置阿里云RAM访问控制：https://ram.console.aliyun.com/users

在 **用户->创建用户** 后，获取对应的ID和secret，然后给这个用户 **添加权限->系统策略**，添加 **AliyunDNSFullAccess** 策略

创建Secret：

~~~yaml
# alidns-secret.yaml
# access-key-id 和 access-key-secret替换为 上面获取的
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: cert-manager
stringData:
  access-key-id: LTAI5 ... F2HCaR
  access-key-secret: ynCHESF ... 4BU0y
~~~

### 3.配置ClusterIssuer

注意 **groupName** 保持一致

~~~yaml
# ClusterIssuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
  namespace: cert-manager
spec:
  acme:
    # 替换为自己的邮箱
    email: 1193055746@qq.com
    # 正式地址
    server: https://acme-v02.api.letsencrypt.org/directory
      # 测试地址
      # server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - dns01:
        webhook:
          # 注意与cert-manager保持一致，不然会失败
          groupName: acme.llyy.com
          solverName: alidns
          config:
            region: ""
            accessKeyIdRef:
              name: alidns-secret
              key: access-key-id
            accessKeySecretRef:
              name: alidns-secret
              key: access-key-secret
~~~

### 4.配置Ingress，自动签发证书

~~~yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog
  namespace: blog
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  # 配置多个hosts 
  # 注意：之前如果已存在llyy-ink-tls ，需要删除 kubectl -n blog delete secrets llyy-ink-tls ，然后会自动重写生成新的
  #  tls:
  #    - secretName: llyy-ink-tls
  #      hosts:
  #        - www.llyy.ink
  #        - llyy.ink
  tls:
    - hosts:
      - www.llyy.ink
      secretName: llyy-ink-tls
  rules:
    - host: www.llyy.ink
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog
                port:
                  number: 80
    - host: llyy.ink
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog
                port:
                  number: 80
~~~

### 5.配置certificate，手动签发证书

~~~yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: llyy-ink
  namespace: blog
spec:
  secretName: llyy-ink-tls
  commonName: "llyy.ink"
  dnsNames:
    - "llyy.ink"
    - "*.llyy.ink"
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
~~~

## apply

### 1.apply alidns-secret,ClusterIssuer及ingress

~~~bash
$ kubectl apply -f alidns-secret.yaml
$ kubectl apply -f ClusterIssuer.yaml
$ kubectl apply -f ingress.yaml
~~~

### 2.验证

证书可能要过几分钟才会变成true，如果发现证书长时间未 Ready，可以参照官方文档 - [Troubleshooting Issuing ACME Certificates](https://cert-manager.io/docs/troubleshooting/acme/)，按证书申请流程进行逐层排查。同时也可以查看alidns-webhook容器的日志，获取详细错误信息。

~~~bash
$ kubectl get certificate --all-namespaces
NAMESPACE      NAME                 READY   SECRET               AGE
cert-manager   alidns-webhook-tls   True    alidns-webhook-tls   106m
cert-manager   alidns-webhook-ca    True    alidns-webhook-ca    106m
blog           llyy-ink-tls         True    llyy-ink-tls         37m

$ kubectl -n blog describe secrets llyy-ink-tls 
Name:         llyy-ink-tls
Namespace:    blog
Labels:       controller.cert-manager.io/fao=true
Annotations:  cert-manager.io/alt-names: www.llyy.ink
              cert-manager.io/certificate-name: llyy-ink-tls
              cert-manager.io/common-name: www.llyy.ink
              cert-manager.io/ip-sans: 
              cert-manager.io/issuer-group: cert-manager.io
              cert-manager.io/issuer-kind: ClusterIssuer
              cert-manager.io/issuer-name: letsencrypt
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
tls.crt:  3583 bytes
tls.key:  1679 bytes
~~~