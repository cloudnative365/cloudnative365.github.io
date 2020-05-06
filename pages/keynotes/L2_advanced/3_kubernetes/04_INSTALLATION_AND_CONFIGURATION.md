---
title: INSTALLATION_AND_CONFIGURATION
keywords: keynotes, L2_advanced, kubernetes, CKA, INSTALLATION_AND_CONFIGURATION
permalink: keynotes_L2_advanced_1_kubernetes_1_CKA_3_INSTALLATION_AND_CONFIGURATION.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/3_INSTALLATION_AND_CONFIGURATION
typora-root-url: ../../../../../cloudnative365.github.io
---

## 学习目标

+ 了解常见的部署kubernetes集群的方式

- 学习kubeadm安装kubernetes集群之后的系统架构
- 下载和配置kubernetes相关工具
- 使用kubeadm安装kubernetes集群

## 1. 部署kubernetes集群的方式

### 1.1. kubeadm

自1.4版以来，Kubeadm是Kubernetes发行工具。该工具有助于在现有基础架构上引导最佳实践的Kubernetes集群。但是，Kubeadm无法为你配置基础结构。它的主要优势是能够在任何地方启动最少可行的Kubernetes集群。但是，附件和网络设置都不在Kubeadm的范围内，因此你将需要手动或使用其他工具来安装它。尽管如此，亲儿子就是亲儿子，有了kubernetes官方的支持，不怕没有市场。

![See the source image](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/OIP.cR34-5z71jjWK_Jx7eqlYwHaId)

## 1.2. rancher

国人最喜欢的工具之一，友好的图形界面，起步也非常早，最近Rancher的创始人更是成为了CNCF的TOC之一，可谓如日中天。他的功能也非常完善，从一键部署k8s到后期的集群管控，一气呵成。而且非常接地气的在改进自己的版本，如今，已经到了2.4时代，是很多刚开始接触k8s的人的不二之选。但是，由于他在k8s上包装了非常多自己的东西，所以很多底层的东西并不很透明，需要对Rancher有非常深入的研究才能在生产系统上驾驭。

![img](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/rancher.png)

### 1.3. Kops

Kops可帮助你从命令行创建，销毁，升级和维护生产级别的高可用性Kubernetes集群。当前已支持AWS，GCE提供beta测试支持，VMware vSphere提供alpha测试，并计划提供其他平台支持。Kops允许控制整个Kubernetes集群生命周期；从基础架构配置到集群删除。后期之秀，但是生的晚了一点。虽然很有后劲，但是其他的软件也不是停滞不前。所以说排在第三位吧。

![image-20200420145624898](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145624898-8571316.png)

### 1.4. kubespray

Kubespray为Kubernetes部署和配置提供了一组Ansible角色。Kubespray可以使用AWS，GCE，Azure，OpenStack或裸机基础架构即服务（IaaS）平台。Kubespray是具有开放开发模型的开源项目。对于已经了解Ansible的人们来说，该工具是一个不错的选择，因为无需使用其他工具进行预配和编排。

![image-20200420145611037](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145611037-8571303.png)





### 1.5. minikube

Minikube允许在本地安装和试用Kubernetes。该工具是开始Kubernetes的一个很好起点。可在虚拟机（VM）内轻松启动单节点Kubernetes集群。Minikube在Windows，Linux和OSX上可用。可在短短5分钟内，将能够使用Kubernetes的主要功能。而且只需一个命令即可直接启动Minikube仪表板。这个东西简单方面，适合在本地测试，但是上生产还是不要了。

![See the source image](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/OIP.qxsK_a2YWWt6vi_7kXdtoQHaCG)



### 1.6. 各个公有云厂商的工具

各大云厂商在k8s上都提供了比较成熟的解决方案，比如AWS的EKS，GCE的GKE，Azure的AKS，他们把k8s和自己的云产品结合起来，提高客户的粘性，让客户更多的依赖自己的平台。但是底层依然是原生的k8s，只不过通过annotation的方式和自己的平台进行结合，提供一些k8s不提供的功能，比如公网DNS直接映射到ingress等等。

### 1.7. 其他的方案

![image-20200420145641248](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145641248-8571353.png)

![image-20200420145659128](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145659128-8571374.png)



## 2. 使用kubeadm安装的集群架构

我们知道，一般的集群架构是这个样子的

![See the source image](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/kubernetes.jpg)

但是，我们知道，etcd，api-server，scheduler和controller-manager这四个组件实际上就是4个进程，我们其实也可以把他运行为容器，也就是使用docker来管理，而在集群中，创建容器是由kubelet调用docker来完成的，所以说，我们就需要在master上也安装上kubelet和docker。而kubeadm在初始化集群的时候，就会使用kubelet的静态pod功能，把关键进程以容器的方式启动。但是这种方式有个缺陷，如果kubelet挂了，那么集群就会不正常，所以，我们需要把kubelet托管给操作系统的systemd来管理。最终，就是下面这个样子了。

![image-20200505184348280](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200505184348280.png)

当然，我们可以把所有的工具都使用kubelet的静态pod的方式，把他托管给kubernetes集群，比如kube-proxy，CoreDNS之类

### 什么叫静态POD呢

我们前面讲过了kubelet的作用，他的作用是监控节点上pod的状态，然后根据pod的类型不同，汇报的对象也不同。我们这里说的pod，一般来说，有两类

+ 一个是有控制器管理的pod，这类pod是生产上最常用的pod，控制器会负责监控容器的整个生命周期
+ 一个是没有控制器管理的POD。而没有控制器管理的pod就由本地的kubelet进行管理。但是这类pod的创建有两种方式
  + 是由API请求生成的pod，这类pod是由调度器根据一定的算法来分配的，这种叫自主式pod
  + 还有一种，是我们定义在kubelet的配置中的，这种pod只会启动在有这个配置的kubelet的那台机器上，这种叫静态pod

## 3. 下载和配置kubernetes相关工具



![See the source image](/pages/keynotes/L2_advanced/3_kubernetes/pics/3_INSTALLATION_AND_CONFIGURATION/kube-install.png)

我们需要在每个节点上安装的软件有docker，kubelet，kubeadm，kubectl，并且我们还需要在安装软件之前执行一些初始化操作。

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### 3.1. 初始化

+ 修改机器名

``` BASH
hostnamectl set-hostname XXX
```

+ 关闭防火墙

``` bash
#停止当前防火墙服务
systemctl stop firewalld.service
#禁用防火墙启动
systemctl disable firewalld.service
#查看防火墙状态
firewall-cmd --state
```

+ （可省略）配置host文件/etc/hosts，在生产系统中，我们通常会使用DNS服务器，但是为了简化，我们这里就使用本地的解析

``` bash
10.0.1.1 master
10.0.1.2 worker1
10.0.1.3 worker2
```

+ （有的系统已经配置好了，检查一下就好）桥接网络配置

``` bash
# 加载模块
$ modprobe br_netfilter
# 验证模块是否生效
$ lsmod | grep br_netfilter
br_netfilter           24576  0
bridge                172032  1 br_netfilter

#新建k8s.conf文件，并添加以下内容，这个是防止由于 iptables 被绕过而导致流量无法正确路由的问题。
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

#执行修改的桥接网络设置
$ sysctl -p /etc/sysctl.d/k8s.conf

# 验证桥接的参数
$ ls /proc/sys/net/bridge
bridge-nf-call-arptables  bridge-nf-call-iptables        bridge-nf-filter-vlan-tagged
bridge-nf-call-ip6tables  bridge-nf-filter-pppoe-tagged  bridge-nf-pass-vlan-input-dev
```

### 3.2. 安装docker

+ 添加docker国内源，例如，添加阿里云的docker源，可以[参考这个](https://developer.aliyun.com/mirror/docker-ce)

+ 安装

  ``` bash
  $ apt-get -y install docker-ce
  ```

  + 如果大家使用的是公有云，请使用

    ``` bash
    $ apt-get -y install docker.io
    ```

  + 如果是其他的操作系统可以尝试

    ``` bash
    $ apt-get -y install docker
    ```

+ 更换镜像仓库为国内的

  ``` bash
  cat > /etc/docker/daemon.json <<EOF
  {
    "registry-mirrors": ["https://gvfjy25r.mirror.aliyuncs.com"]
  }
  EOF
  ```

  记得`systemctl daemon-reload`和`systemctl restart docker`

### 3.3. 安装kubeadm，kubelet，kubectl

+ 我使用的是官方的源

  ``` bash
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
  
  cat <<EOF >/etc/apt/sources.list.d/docker.list
  deb [arch=armhf] https://download.docker.com/linux/raspbian buster stable
  EOF
  
  apt-get update
  ```

+ 使用kubernetes国内源

  ``` bash
  apt-get update && apt-get install -y apt-transport-https
  
  curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
  
  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
  deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
  EOF
  
  apt-get update
  ```

+ 安装

  ``` bash
  $ apt-get -y install kubectl kubelet kubeadm
  ```

  

## 4. 使用kubeadm安装kubernetes集群

### 4.1. 初始化master节点

执行kubeadm init

``` bash
kubeadm init --apiserver-advertise-address=10.0.1.1 --pod-network-cidr=10.244.0.0/16 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

.
.
.
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

执行下面的命令，让kubectl可以正常访问集群

``` bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.2. 初始化worker节点

把上面输出的内容，在worker节点上执行，来加入到集群当中

``` bash
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```



## 5. 选择网络方案

在初始化Kubernetes集群之前，必须考虑网络并避免IP冲突。在不同的开发级别和特性集中，有几种Pod网络选择。

许多项目都会提到容器网络接口（CNI），这是一个CNCF项目。目前有几个容器运行时使用CNI。CNI作为处理网络资源部署管理和清理的标准，会越来越流行。

![image-20200420145238669](/pages/keynotes/L2_advanced/1_CKA/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145238669.png)

![image-20200420145320616](/pages/keynotes/L2_advanced/1_CKA/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145320616.png)

![image-20200420145336238](/pages/keynotes/L2_advanced/1_CKA/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145336238.png)

![image-20200420145353789](/pages/keynotes/L2_advanced/1_CKA/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145353789.png)

![image-20200420145412668](/pages/keynotes/L2_advanced/1_CKA/pics/3_INSTALLATION_AND_CONFIGURATION/image-20200420145412668.png)

