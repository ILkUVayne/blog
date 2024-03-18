---
title: github Action结合k3s部署hexo
date: 2024-03-08 16:54:00
categories:
- devops
tags:
- k3s
- github Actions
---

## 准备工作

- 一个云服务器（这里使用阿里云的服务器）
- 一个镜像仓库（这里使用阿里云的个人镜像仓库）
- github代码仓库（托管代码）
- 域名（同样是阿里云购买的域名）
- 一个配置好的hexo博客
- 阿里云服务器需要配置安全组（开放80/443/6443等端口）

## 配置服务器环境

### 下载k3s

#### 1.官方下载

国内下载可能会失败或者非常慢（因为有些资源在外网）

~~~bash
curl -sfL https://get.k3s.io | sh -
~~~

#### 2.国内源安装

~~~bash
# --docker 使用docker作为容器运行时
# --disable traefik 禁用traefik，使用ingress-nginx
# --tls-san "xxx.xxx.xxx.158" 服务器公网ip 如果是阿里云服务器，需要添加安全组（6443端口默认未开启）
# --write-kubeconfig ~/.kube/config 修改配置文件路径，和k8s保持一致
curl –sfL \                         
     https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | \
     INSTALL_K3S_MIRROR=cn sh -s - \
     --system-default-registry "registry.cn-hangzhou.aliyuncs.com" \
     --write-kubeconfig ~/.kube/config \
     --write-kubeconfig-mode 666 \
     --disable traefik \
     --tls-san "xxx.xxx.xxx.158" \
     --docker
~~~

### 安装helm

[helm官方安装](https://helm.sh/zh/docs/intro/install/)

国内服务器安装较慢或者失败，采用第三方包管理方式安装

~~~bash
# 使用Apt (Debian/Ubuntu)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
~~~

验证：

~~~bash
$ helm version
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /root/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /root/.kube/config
version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f0ff63856811846ce18f3bdc93d2b4d54b", GitTreeState:"clean", GoVersion:"go1.21.7"}

# 修改配置文件权限，消除警告
$ chmod -R 600   ~/.kube/config
$ helm version                 
version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f0ff63856811846ce18f3bdc93d2b4d54b", GitTreeState:"clean", GoVersion:"go1.21.7"}
~~~

### 重启服务

~~~bash
sudo systemctl daemon-reload
sudo systemctl restart k3s 

# 然后查看节点是否启动正常
sudo k3s kubectl get no
~~~

### 配置kubectl命令补全

~~~bash
# bash
source <(kubectl completion bash)

# zsh
source <(kubectl completion zsh)
~~~

### 安装ingress-nginx

ingress-nginx需要从谷歌k8s仓库拉取镜像，国内无法拉取，采取将镜像拉取到本地的方式安装,本地下载方式查看 {% post_link devops/创建github镜像仓库 %}

下载官方**deploy.yaml**

> ingress-nginx 版本需要和k8s对应，具体版本查看官方：https://github.com/kubernetes/ingress-nginx

~~~bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
~~~

替换镜像为本地镜像：

~~~bash
# vim 编辑替换镜像
$ vim  deploy.yaml
~~~

apply 应用

~~~bash
$ kubectl apply -f deploy.yaml 
~~~
### 配置tls证书

实现方式： {% post_link devops/cert-manager结合alidns-webhook实现签发免费证书并为证书自动续期 %}

## 配置github Action

Github Actions 是由 Github 官方提供的一套 CI 方案,通过 Actions，我们可以在每次提交代码的时候，让 GitHub 自动帮我们完成编译->打包->测试->部署的整个过程。

### 编写Actions配置

本博客仓库只有mian分支，所以只会在 mian 收到 push 时触发Action。触发的过程如下：

1. 先安装hexo-cli,然后通过hexo-cli将博客编译成静态文件
2. 然后docker根据Dockerfile构建出镜像，并推送到镜像仓库（这里使用阿里云的个人镜像仓库）
3. 最后通知k3s拉取仓库镜像并通过 kubectl 将我的应用配置部署到 k3s 服务上

~~~yaml
name: main

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # node 编译
      - name: build
        uses: actions/setup-node@v4
      - run: |
          npm i -g hexo-cli
          npm i
          hexo clean
          hexo g

      # docker build，并push
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          registry: registry.cn-shanghai.aliyuncs.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: registry.cn-shanghai.aliyuncs.com/my_hb/my-blog:${{ github.sha }}

      # 让K8s应用deployment
      - name: replace deployment env
        run: |
          sed -i 's/{TAG}/${{ github.sha }}/g' deployment.yaml
      - name: deploy to cluster
        uses: steebchen/kubectl@master
        with:
          config: ${{ secrets.KUBE_CONFIG_DATA }}
          command: apply -f deployment.yaml
      - name: verify deployment
        uses: steebchen/kubectl@master
        with:
          config: ${{ secrets.KUBE_CONFIG_DATA }}
          command: rollout status -n blog deployment/blog
~~~

我们用到了3个secrets，分别是镜像仓账号密码和 kubeconfig，这个需要在github对应的代码仓库 **settings->Secrets and variables->Repository secrets** 中添加。

镜像仓库账号密码需要在对应的仓库中的基本信息中获取，密码可以在访问凭证中修改

kubeconfig存放在 **~/.kube/config** 中，拷贝出来（不要修改原配置）保存成文件，将其中的 **server: https://127.0.0.1:6443** 修改为公网ip **server: https://xxx.xxx.xx.xx:6443**,再通过cat kubeconfig | base64得到正确的 config 信息。

### docker打包

对于 hexo 博客而言，通过hexo g编译成静态文件后我们就可以直接通过访问public文件夹中的 index.html 文件来预览我们的博客了。我们需要将hexo-cli编译得到的静态文件**public**打包到一个 nginx 镜像中，并暴露 80 端口来为提供 http 请求提供服务。并推送到指定的镜像仓库中。

~~~dockerfile
FROM nginx:1.17.7-alpine
EXPOSE 80
EXPOSE 443
COPY ./public /usr/share/nginx/html
~~~

### 连接k3s集群，部署应用

我们需要通知 k3s 部署服务或者升级服务。通过在设置好 kubeconfig 文件，我们便能使用 kubectl 远程访问 apiserver。在通过kubectl apply -f deployment.yaml部署上最新的配置文件即可

~~~yaml
      - name: replace deployment env
        # 替换deployment.yaml {TAG}占位符的内容为镜像实际tag ${{ github.sha }}
        run: |
          sed -i 's/{TAG}/${{ github.sha }}/g' deployment.yaml
      - name: deploy to cluster
        uses: steebchen/kubectl@master
        with:
          config: ${{ secrets.KUBE_CONFIG_DATA }}
          command: apply -f deployment.yaml
      - name: verify deployment
        uses: steebchen/kubectl@master
        with:
          config: ${{ secrets.KUBE_CONFIG_DATA }}
          command: rollout status -n blog deployment/blog
~~~

## apply应用

### deployment.yaml

~~~yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog
  namespace: blog
spec:
  ingressClassName: nginx
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
---
apiVersion: v1
kind: Service
metadata:
  name: blog
  namespace: blog
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: blog
  sessionAffinity: ClientIP
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog
  namespace: blog
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: blog
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
        - image: registry.cn-shanghai.aliyuncs.com/my_hb/my-blog:{TAG}
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: 80
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 2
            successThreshold: 1
            timeoutSeconds: 2
          name: blog
          ports:
            - containerPort: 80
              name: 80tcp02
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: 80
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 2
            successThreshold: 2
            timeoutSeconds: 2
          resources: {}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities: {}
            privileged: false
            readOnlyRootFilesystem: false
            runAsNonRoot: false
          stdin: true
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          tty: true
          volumeMounts:
            - mountPath: /usr/share/nginx/html/db.json
              name: db
            - mountPath: /usr/share/nginx/html/Thumbs.json
              name: thumbs
      imagePullSecrets:
        - name: myregistrykey
      dnsConfig: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - hostPath:
            path: /data/db.json
            type: ""
          name: db
        - hostPath:
            path: /data/Thumbs.json
            type: ""
          name: thumbs
~~~

### 创建namespace

~~~bash
$ kubectl create namespace blog
~~~

验证：

~~~bash
$ kubectl get namespaces                                 
NAME              STATUS   AGE
kube-system       Active   3d19h
kube-public       Active   3d19h
kube-node-lease   Active   3d19h
default           Active   3d19h
blog              Active   2d15h
~~~

### 创建镜像仓库拉取Secret

我们要拉取的镜像为私有仓库，所以我们需要使用imagePullSecrets并创建Secret

登录云服务器，并登录阿里云Docker Registry

~~~bash
$ docker login --username=账号 registry.cn-shanghai.aliyuncs.com
~~~

创建名字为myregistrykey的Secret

~~~bash
# 需要设置namespace
# 仓库地址、账号、密码、邮箱替换为实际的值
kubectl create secret docker-registry --namespace blog myregistrykey \
  --docker-server=仓库地址 \
  --docker-username=账号 \
  --docker-password=密码 \
  --docker-email=邮箱
~~~

检查是否成功

~~~bash
# .dockerconfigjson 保存的就是账号信息，可以使用 echo 'eyJh...' | base64 --decode 验证
$ kubectl -n blog get secrets myregistrykey --output yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXR ... jMlE9In19fQ==
kind: Secret
metadata:
  creationTimestamp: "2024-03-09T13:24:37Z"
  name: myregistrykey
  namespace: blog
  resourceVersion: "57891"
  uid: 18bd76 ... f6c66f
type: kubernetes.io/dockerconfigjson
~~~

### 验证是否部署成功

查看pod状态

~~~bash
kubectl -n blog get pods                               
NAME                    READY   STATUS    RESTARTS   AGE
blog-6df748f77c-6vdpw   1/1     Running   0          42h
blog-6df748f77c-fq86r   1/1     Running   0          42h
~~~