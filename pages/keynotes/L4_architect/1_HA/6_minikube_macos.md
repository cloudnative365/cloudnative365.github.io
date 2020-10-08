---
title: MacOS上安装MiniKube
keywords: keynotes, architect, HA, minikube_macos
permalink: keynotes_L4_architect_1_HA_6_minikube_macos.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/6_minikube_macos
typora-root-url: ../../../../../cloudnative365.github.io
---



## 1. 环境的准备

### 1.1. 是否支持虚拟化

``` bash
sysctl -a | grep -E --color 'machdep.cpu.features|VMX'
```

如果结果中出现了VMX，就说明VT-x功能是打开的

### 1.2. 检查其他虚拟化软件

如果安装了HyperKit，VitrualBox或者VMwareFusion之外的其他hypervisor，有可能会产生冲突。

### 1.3. 安装docker-desktop

在[这里](https://www.docker.com/products/docker-desktop)下载，或者直接点击[这里](https://download.docker.com/mac/stable/Docker.dmg)，安装的时候各种下一步就可以了。

## 2 安装minikube

### 2.1. 安装brew，参考[这里](https://github.com/Homebrew/install)

``` bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

### 2.2. 修改brew的仓库为aliyun，参考[这里](https://developer.aliyun.com/mirror/homebrew)

``` bash
# 替换brew.git:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git
# 替换homebrew-core.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git
# 应用生效
brew update
# 替换homebrew-bottles:
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc
```

### 2.3. 安装kubectl

``` bash
brew install kubectl
```

或者

``` bash
brew install kubernetes-cli
```

+ 也可以从官方直接下载编译好的kubectl命令，参考[这里](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-macos)

### 2.4. 安装Hypervisor

我们可以选择[HyperKit](https://github.com/moby/hyperkit)(Docker的产品)， [VirtualBox](https://www.virtualbox.org/wiki/Downloads)(微软的产品)，[VMware Fusion](https://www.vmware.com/products/fusion)(VMware的产品)之一。我们不用手动安装，如果没有安装过这三个产品之一的话，下一步安装minikube的时候会直接安装的，我们无需做任何操作。

### 2.5. 安装minikube

``` bash
brew install minikube
```

+ 当然也可以使用编译好的包，参考[这个](https://kubernetes.io/docs/tasks/tools/install-minikube/#install-minikube)

``` bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 && chmod +x minikube
sudo mv minikube /usr/local/bin
```

## 3. 启动和停止

### 3.1. 启动

如果我们默认启动的话，他会去连Google的仓库，所以我们需要手动指定他的仓库是aliyun

``` bash
minikube start --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```

minikube是支持多种虚拟化引擎的，比如virtual-box，vmware fusion或者docker都可以，如果我们没有指定，那么默认是docker，如果我们想指定其他引擎，比如使用vmfusion，就需要使用

``` bash
minikube start --driver=<driver_name>
```



### 3.2. 查看状态

``` bash
minikube status
```

### 3.3. 停止

```
minikube stop
```

## 4. 部署一个应用

+ 创建nginx的deployment

``` bash
kubectl create deployment nginx --image=nginx
```

+ 暴露端口

``` bash
kubectl expose deployment nginx --type=LoadBalancer --port=80
service/nginx exposed
```

+ 查看服务

``` bash
kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP        12m
nginx        LoadBalancer   10.103.91.33   <pending>     80:31382/TCP   6s
```

+ 访问我们的服务

``` bash
minikube service nginx
|-----------|-------|-------------|---------------------------|
| NAMESPACE | NAME  | TARGET PORT |            URL            |
|-----------|-------|-------------|---------------------------|
| default   | nginx |             | http://192.168.64.2:31382 |
|-----------|-------|-------------|---------------------------|
```



