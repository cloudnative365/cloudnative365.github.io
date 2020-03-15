---
title: 用kubeadm搭建k8s高可用（apt版）
keywords: keynotes, architect, solutions_design, HA, k8s_cluster_kubeadm_apt
permalink: keynotes_L4_architect_1_solutions_design_1_HA_2_k8s_cluster_kubeadm_apt.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/2_k8s_cluster_kubeadm_apt
typora-root-url: ../../../../../../cloudnative365.github.io



---

## 1. 介绍

### 1.1. kubeadm

前面讲kubeadm的时候，我们已经介绍过kubeadm的功能的，我么这里只说一下kubeadm创建集群时候的操作

### 2 架构图

### 2.1. 整体架构图

![ha-master-gce](/pages/keynotes/L4_architect/1_solutions_design/5_HA/pics/2_k8s_cluster_kubeadm_apt/ha-master-gce.png)

## 3. 资源清单

### 3.1. 测试环境

| 机器名    | IP        | 组件                  | CPU  | 内存 | 磁盘 | 操作系统                      |
| --------- | --------- | --------------------- | ---- | ---- | ---- | ----------------------------- |
| master1   | 10.1.1.11 | etcd1，master1        | 2C   | 4G   | 64G  | RHEL7/CentOS7                 |
| master2   | 10.1.1.12 | etcd2，master2        | 2C   | 4G   | 64G  | ubuntu16/ubuntu18/raspberryPi |
| master3   | 10.1.1.13 | etcd3，master3        | 2C   | 4G   | 64G  | ubuntu16/ubuntu18/raspberryPi |
| worker1   | 10.1.1.21 | worker1               | 2C   | 4G   | 128G | ubuntu16/ubuntu18/raspberryPi |
| worker2   | 10.1.1.22 | worker2               | 2C   | 4G   | 128G | ubuntu16/ubuntu18/raspberryPi |
| mgtserver | 10.1.1.10 | loadbalancer（nginx） | 1C   | 2G   | 64G  | ubuntu16/ubuntu18/raspberryPi |
| master    | 10.1.1.10 | 虚拟IP                |      |      |      |                               |



### 3.2. 生产环境

## 4. 安装与配置

### 4.1. 初始化

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



+ 配置host文件/etc/hosts，在生产系统中，我们通常会使用DNS服务器，但是为了简化，我们这里就使用本地的解析

``` bash
10.1.1.11 master1
10.1.1.12 master2
10.1.1.13 master3
10.1.1.21 worker1
10.1.1.22 worker2
10.1.1.10 mgtserver
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

### 4.2. 配置apt源

+ (可选，apt的国外源也OK，就是稍微有点慢)更换apt源为国内源

修改/etc/apt/sources.list.d中的内容，把原来的注释掉，添加国内源，例如，ubuntu更换阿里云的源，[看这里](https://developer.aliyun.com/mirror/ubuntu)。

我使用的是树莓派，所以我更换了下面的源

``` bash
deb https://mirrors.aliyun.com/raspbian/raspbian/ buster main non-free contrib
deb-src https://mirrors.aliyun.com/raspbian/raspbian/ buster main non-free contrib
```

+ 添加docker国内源，例如，添加阿里云的docker源，可以[参考这个](https://developer.aliyun.com/mirror/docker-ce)

我使用的是官方的源

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

### 4.3. 安装软件

+ Docker

``` bash
$ apt-get -y install docker-ce
```

+ kubectl, kubelet和kubeadm

``` bash
$ apt-get -y install kubectl kubelet kubeadm
```

### 4.4. 修改Docker的源为国内的源

``` bash
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
$ apt-get install -y nginx
```

在`/etc/nginx/nginx.conf`里面添加一个include，让nginx读取目录下的配置文件

``` bash
include /etc/nginx/conf.d/tcp.d/*.conf;
```

添加kubernetes的4层代理配置文件`kube-api-server.conf`

``` bash
cat <<EOF >/etc/nginx/conf.d/tcp.d/kube-api-server.conf
stream {
    log_format main '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log /var/log/nginx/k8s-access.log main;
    upstream k8s-apiserver {
        server 10.1.1.11:6443;
        server 10.1.1.12:6443;
        server 10.1.1.13:6443;
    }
    server {
        listen 10.1.1.10:6443;
        proxy_pass k8s-apiserver;
    }
}
EOF
```

