---
title: 方法论：我们需要哪些系统
keywords: keynotes, architect, solutions_design, cncf_roadmap
permalink: keynotes_L4_architect_1_solutions_design_cncf_roadmap.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/cncf_roadmap
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. 介绍

### 1.1. kubeadm

前面讲kubeadm的时候，我们已经介绍过kubeadm的功能的，我么这里只说一下kubeadm创建集群时候的操作

### 2 架构图

### 2.1. 整体架构图

![ha-master-gce](/../../../../Desktop/ha-master-gce.png)

## 3. 资源清单

### 3.1. 测试环境

| 主机       | 组件           | CPU  | 内存 | 磁盘 | 操作系统           |
| ---------- | -------------- | ---- | ---- | ---- | ------------------ |
| 10.0.11.18 | etcd1，master1 | 2C   | 4G   | 10G  | RHEL7/8，CentOS7/8 |
| 10.0.12.55 | etcd2，master2 | 2C   | 4G   | 10G  | RHEL7/8，CentOS7/8 |
| 10.0.13.22 | etcd3，master3 | 2C   | 4G   | 10G  | RHEL7/8，CentOS7/8 |
| 10.0.12.16 | worker1        | 2C   | 4G   | 10G  | RHEL7/8，CentOS7/8 |
| 10.0.13.37 | worker2        | 2C   | 4G   | 10G  | RHEL7/8，CentOS7/8 |
| 10.0.1.157 | loadbalancer   | 2C   | 4G   | 10G  | RHEL7/8，CentOS7/8 |



### 3.2. 生产环境

## 4. 安装与配置

### 4.1. 初始化

+ 修改机器名

  ```bash
  hostnamectl set-hostname XXX
  ```

+ 配置host文件/etc/hosts，在生产系统中，我们通常会使用DNS服务器，但是为了简化，我们这里就使用本地的解析

  ``` bash
  10.0.11.18 master1
  10.0.12.55 master2
  10.0.13.22 master3
  10.0.12.16 worker1
  10.0.13.37 worker2
  10.0.1.157 master-lb
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

***下面的步骤在负载均衡节点master-lb上做***

+ 安装nginx

``` bash
$ yum -y install nginx
```

+ 在`/etc/nginx/nginx.conf`里面添加一个include，让nginx读取目录下的配置文件

``` bash
include /etc/nginx/conf.d/tcp.d/*.conf;
```

+ 添加kubernetes的4层代理配置文件`/etc/nginx/conf.d/tcp.d/kube-api-server.conf`

``` bash
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
```

+ 查看端口是否在监听了

``` bash
netstat -untlp|grep 6443
tcp        0      0 10.0.1.157:6443         0.0.0.0:*               LISTEN      15787/nginx: master
```

***上面的步骤在负载均衡节点master-lb上做***

### 4.6. 使用kubeadm初始化第一个master节点

+ 检查网络是否通畅(使用telnet也可以)

``` bash
nc -v 10.0.1.157 6443
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 10.0.1.157:6443.
```

+ 把LOAD_BALANCER_DNS:LOAD_BALANCER_PORT替换成刚才nginx的IP和端口

``` bash
kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
```



``` bash
kubeadm init --control-plane-endpoint "10.0.1.157:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```

成功之后，会有下面的提示，找个小本本记下来吧

``` bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.0.1.157:6443 --token yww7fd.oxn7vj6484ye5amq \
    --discovery-token-ca-cert-hash sha256:94f5eaf1fcd4c32d52084515d5917561fd94b2e5489798e3d2a7edd615fe892b \
    --control-plane --certificate-key ac9317ed86ae2132d172f3dae66163f156aa13eaba299c6d00c7228abbaec9d4

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.1.157:6443 --token yww7fd.oxn7vj6484ye5amq \
    --discovery-token-ca-cert-hash sha256:94f5eaf1fcd4c32d52084515d5917561fd94b2e5489798e3d2a7edd615fe892b
```

### 4.7. 初始化master2和master3

``` bash
kubeadm join 10.0.1.157:6443 --token yww7fd.oxn7vj6484ye5amq \
    --discovery-token-ca-cert-hash sha256:94f5eaf1fcd4c32d52084515d5917561fd94b2e5489798e3d2a7edd615fe892b \
    --control-plane --certificate-key ac9317ed86ae2132d172f3dae66163f156aa13eaba299c6d00c7228abbaec9d4
```

### 4.8. 初始化worker1和worker2

``` bash
kubeadm join 10.0.1.157:6443 --token yww7fd.oxn7vj6484ye5amq \
    --discovery-token-ca-cert-hash sha256:94f5eaf1fcd4c32d52084515d5917561fd94b2e5489798e3d2a7edd615fe892b
```

### 4.9. 选择一个网络方案

我们这里就选用flannel了

``` bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 4.10. 验证一下成果

``` bash
[root@master-lb .kube]# kubectl get pod -A
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-6955765f44-6w7hn          1/1     Running   0          59m
kube-system   coredns-6955765f44-vpdfg          1/1     Running   0          59m
kube-system   etcd-master1                      1/1     Running   0          59m
kube-system   etcd-master2                      1/1     Running   0          49m
kube-system   etcd-master3                      1/1     Running   0          48m
kube-system   kube-apiserver-master1            1/1     Running   0          59m
kube-system   kube-apiserver-master2            1/1     Running   0          49m
kube-system   kube-apiserver-master3            1/1     Running   0          47m
kube-system   kube-controller-manager-master1   1/1     Running   1          59m
kube-system   kube-controller-manager-master2   1/1     Running   0          49m
kube-system   kube-controller-manager-master3   1/1     Running   0          47m
kube-system   kube-flannel-ds-amd64-fkgbb       1/1     Running   0          11s
kube-system   kube-flannel-ds-amd64-l6rx4       1/1     Running   0          11s
kube-system   kube-flannel-ds-amd64-lhg9p       1/1     Running   0          11s
kube-system   kube-flannel-ds-amd64-rjr9x       1/1     Running   0          11s
kube-system   kube-flannel-ds-amd64-sb955       1/1     Running   0          11s
kube-system   kube-proxy-2h4dt                  1/1     Running   0          49m
kube-system   kube-proxy-67f8b                  1/1     Running   0          59m
kube-system   kube-proxy-f9774                  1/1     Running   0          50m
kube-system   kube-proxy-sw4k5                  1/1     Running   0          48m
kube-system   kube-proxy-tst5s                  1/1     Running   0          51m
kube-system   kube-scheduler-master1            1/1     Running   1          59m
kube-system   kube-scheduler-master2            1/1     Running   0          49m
kube-system   kube-scheduler-master3            1/1     Running   0          47m
```



