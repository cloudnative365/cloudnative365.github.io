---
title: 用kubeadm搭建k8s高可用（yum版）
keywords: keynotes, architect, solutions_design, HA, k8s_cluster_kubeadm_yum
permalink: keynotes_L4_architect_1_solutions_design_5_HA_1_k8s_cluster_kubeadm_yum.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_k8s_cluster_kubeadm_yum
typora-root-url: ../../../../../../cloudnative365.github.io


---

## 1. 介绍

### 1.1. kubeadm

前面讲kubeadm的时候，我们已经介绍过kubeadm的功能的，我么这里只说一下kubeadm创建集群时候的操作

### 2 架构图

### 2.1. 整体架构图

![ha-master-gce](/pages/keynotes/L4_architect/1_solutions_design/5_HA/pics/1_k8s_cluster_kubeadm_yum/ha-master-gce.png)

## 3. 资源清单

### 3.1. 测试环境

| 主机        | 组件           | CPU  | 内存 | 磁盘 | 操作系统      |
| ----------- | -------------- | ---- | ---- | ---- | ------------- |
| 10.0.11.228 | etcd1，master1 | 2C   | 4G   | 10G  | RHEL7/CentOS7 |
| 10.0.12.92  | etcd2，master2 | 2C   | 4G   | 10G  | RHEL7/CentOS7 |
| 10.0.13.58  | etcd3，master3 | 2C   | 4G   | 10G  | RHEL7/CentOS7 |
| 10.0.12.214 | worker1        | 2C   | 4G   | 10G  | RHEL7/CentOS7 |
| 10.0.13.21  | worker2        | 2C   | 4G   | 10G  | RHEL7/CentOS7 |
| 10.0.1.156  | loadbalancer   | 2C   | 4G   | 10G  | RHEL7/CentOS7 |



### 3.2. 生产环境

## 4. 安装与配置

### 4.1. 初始化

+ 修改机器名

  ```bash
  hostnamectl set-hostname XXX
  ```

+ 配置host文件/etc/hosts，在生产系统中，我们通常会使用DNS服务器，但是为了简化，我们这里就使用本地的解析

  ``` bash
  10.0.11.176 master1
  10.0.12.228 master2
  10.0.13.118 master3
  10.0.12.192 worker1
  10.0.13.76 worker2
  10.0.1.162 master-lb
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

  

  ***注意***：firewalld是一个管理工具，他管理的是linux的内核子系统netfilter，关闭firewalld并不意味着禁用了内核子系统，只不过丢给kube-proxy去管理，而kube-proxy是通过管理lvs来管理netfilter规则的，所以说，要关闭firewalld，防止他和lvs产生冲突。

+ 关闭selinux

  ``` bash
  #关闭当前selinux服务
  $ setenforce 0 
  #修改selinux配置文件，防止重启后再次开启
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

+ 桥接网络配置

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

  

### 4.2. 配置yum源

需要配置的有Docker，k8s和epel的源，我们都使用马云爸爸提供的源就好了。

+ CentOS7/RHEL7

Docker源

``` bash
# 安装所有yum的相关组件
$ yum install -y yum-utils
# 把repo文件添加到本地
$ yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

epel源

``` bash
$ rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
```

k8s源

``` bash
$ cat <<EOF >  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

+ CentOS8/RHEL8

Docker源

``` bash
# 安装所有yum的相关组件
$ dnf install -y yum-utils
# 把repo文件添加到本地
$ yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

epel源

``` bash
$ rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-8.noarch.rpm
```

k8s源

``` bash
$ cat <<EOF >  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```



### 4.3. 安装软件

+ Docker

```bash
$ yum -y install docker-ce
$ systemctl start docker
```

+ 注意1：Centos7或者RHEL7在安装docker的时候有可能会出现依赖的报错

```bash
错误：软件包：3:docker-ce-19.03.7-3.el7.x86_64 (docker-ce-stable)
          需要：container-selinux >= 2:2.74
错误：软件包：containerd.io-1.2.13-3.1.el7.x86_64 (docker-ce-stable)
          需要：container-selinux >= 2:2.74
 您可以尝试添加 --skip-broken 选项来解决该问题
```

只需要安装上这个包就可以了

```bash
$ rpm -ivh http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-3.el7.noarch.rpm
```

+ 注意2：RHEL8在安装docker的时候会报依赖的错误

``` bash
Error:
 Problem: package docker-ce-3:19.03.8-3.el7.x86_64 requires containerd.io >= 1.2.2-3, but none of the providers can be installed
  - cannot install the best candidate for the job
  - package containerd.io-1.2.10-3.2.el7.x86_64 is excluded
  - package containerd.io-1.2.13-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.3.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.el7.x86_64 is excluded
  - package containerd.io-1.2.4-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.5-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.6-3.3.el7.x86_64 is excluded
(try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

只需要安装上这个包就可以了

``` bash
$ dnf -y install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
```

+ kubectl，kubelet和kubeadm

```bash
$ yum -y install kubectl kubelet kubeadm
```

### 4.4. 修改Docker的源为国内的源

```bash
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://gvfjy25r.mirror.aliyuncs.com"]
}
EOF
```

记得`systemctl daemon-reload`和`systemctl restart docker`

### 4.5. 负载均衡器

负载均衡可以选择Nginx，Haproxy，lvs或者traefik甚至apache都可以，基本上所有的4层负载均衡或者7层负载均衡都可以，负载均衡的主要作用就是前端使用一个统一的IP地址，后端映射api-server。让每个node通讯的时候，都通过负载均衡器来调度请求。

这里，我们就使用最常见，最容器实现的nginx来做负载均衡。

``` bash
$ yum -y install nginx
```

