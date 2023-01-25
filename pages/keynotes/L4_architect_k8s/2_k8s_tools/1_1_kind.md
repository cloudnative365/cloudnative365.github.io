---
title: kind
keywords: keynotes, architect, k8s_tools, kind
permalink: keynotes_L4_architect_2_k8s_tools_1_1_kind.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_1_kind
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 简介

Kind是kubernetes官方推荐的，快速构建k8s环境的工具之一

## 2. 功能

## 3. 安装和部署

### 3.1. runtime

我们可以直接使用yum或者apt-get来安装docker，只要我们的操作系统是最新版基本都没什么问题。安装docker不是关键，关键的是containerd，凡是比较新的docker都是docker+containerd的架构，而docker对于k8s来说直接就寄了，k8s要用的是containerd。另外，我们最好安装比较新的docker版本。

+ centos8

  ``` bash
  dnf -y install docker
  ```

+ centos7，由于自带版本太低，建议直接使用社区版，建议使用[国内的源](https://developer.aliyun.com/mirror/docker-ce?spm=a2c6h.13651102.0.0.3e221b111ICv0A)

  ``` bash
  # step 1: 安装必要的一些系统工具
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  # Step 2: 添加软件源信息
  sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  # Step 3
  sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
  # Step 4: 更新并安装Docker-CE
  sudo yum makecache fast
  sudo yum -y install docker-ce
  # Step 4: 开启Docker服务
  sudo service docker start
  ```
  
+ ubuntu

  ``` bash
  apt-get install docker.io
  ```

### 3.2. 安装Kind

+ 下载二进制包并且丢到$PATH下面就行，[官方文档](https://github.com/kubernetes-sigs/kind)

  ``` bash
  curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.16.0/kind-$(uname)-amd64"
  chmod +x ./kind
  mv ./kind /usr/local/sbin
  ```

### 3.3. 安装Kubectl

+ 由于kind只负责创建集群，管理工具需要我们自己下载，我们就手动下载kubectl，[官方文档](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

  ``` bash
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x kubectl
  mv kubectl /usr/local/sbin
  ```


## 4. 创建集群

最简单的方式就是`kind create cluster`，我们使用的是带[ingress](https://kind.sigs.k8s.io/docs/user/ingress/)的，所以要用下面的来创建

``` bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

然后创建nginx-ingress

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

验证，通过创建ingress服务来验证下集群是正常的

``` yaml
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: foo
spec:
  containers:
  - name: foo-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=foo"
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
  # Default port used by the image
  - port: 5678
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: bar
spec:
  containers:
  - name: bar-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=bar"
---
kind: Service
apiVersion: v1
metadata:
  name: bar-service
spec:
  selector:
    app: bar
  ports:
  # Default port used by the image
  - port: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: foo-service
            port:
              number: 5678
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: bar-service
            port:
              number: 5678
---
```

如果集群正常的话，可以看到下面的输出

``` bash
# should output "foo"
curl localhost/foo
# should output "bar"
curl localhost/bar
```

## 5. 相关工具

### 5.1. Helm

+ 官方的源国内无法访问，我们直接使用脚本方式下载和安装，如果git的raw地址访问不了，就直接用网页打开，把内容粘贴下来执行

  ``` bash
  $ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  $ chmod 700 get_helm.sh
  $ ./get_helm.sh
  ```

### 5.2. kustomize

+ 有些operater是通过这种方式来安装的，有点类似于make && make install，但是这里用的是[kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/)命令

  ``` bash
  curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
  ```

  
