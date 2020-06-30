---
title: 使用kubeadm安装kubernetes master
keywords: keynotes, senior, kubernetes, install_kubernetes_kubeadm
permalink: keynotes_L3_senior_2_kubernetes_1_install_kubernetes_kubeadm.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/1_install_kubernetes_kubeadm
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 使用kubeadm在已经建好的etcd集群上安装kubernetes

## 1. 简介

我们安装kubernetes高可用集群的方式非常的多，我们会在架构师课程中专门安排一个专题来说kubernetes的各种高可用安装方式，这个高级课程中，我们只说使用kubeadm安装的各种高可用集群。最简单的方式，请移步我的gitpage。

[用kubeadm搭建k8s高可用（yum版）](https://cloudnative365.github.io/keynotes_L4_architect_1_HA_1_k8s_cluster_kubeadm_yum.html)，[用kubeadm搭建k8s高可用（apt版）](https://cloudnative365.github.io/keynotes_L4_architect_1_HA_2_k8s_cluster_kubeadm_apt.html)

## 2. 架构与环境

### 2.1. 架构

+ 我们这次要说的是在外挂etcd的架构上安装kubernetes集群。架构图如下

![image-20200623144543807](/pages/keynotes/L3_senior/2_kubernetes/pics/1_install_kubernetes_kubeadm/image-20200623144543807.png)

+ etcd集群的安装方式我们前面已经说了，我们这里假设etcd集群已经安装好的，我们需要安装的是除了etcd的其他部分。

+ 机器

  | IP          | hostname | 用途                                   | 组件                                   |
  | ----------- | -------- | -------------------------------------- | -------------------------------------- |
  | 10.0.1.94   | lb       | loadbalance和jumpserver                | nginx                                  |
  | 10.0.11.202 | control1 | apiserver/controller-manager/scheduler | apiserver/controller-manager/scheduler |
  | 10.0.12.249 | control2 | control plane                          | apiserver/controller-manager/scheduler |
  | 10.0.13.82  | control3 | control plane                          | apiserver/controller-manager/scheduler |
  | 10.0.11.201 | etcd1    | etcd host                              | etcd                                   |
  | 10.0.12.248 | etcd2    | etcd host                              | etcd                                   |
  | 10.0.13.81  | etcd3    | etcd host                              | etcd                                   |
  | 10.0.12.135 | node1    | worker node                            | kubelet/kube-proxy                     |
  | 10.0.13.253 | node2    | worker node                            | kubelet/kube-proxy                     |

+ loadbalance的选型：支持4层负载均衡的均衡器都可以，但是如果在生产上一定要做两个负载均衡来保证

+ 使用kubeadm安装的kubernetes集群会把kube-api，kube-scheduler和kube-controller-manager都运行为容器，而kubelet和容器runtime（docker或者containerd）会托管给systemd

+ 为了满足国内朋友的需求，我会使用国内的环境来搭建，所以咱们会指定镜像仓库为国内的仓库（阿里云的仓库）

### 2.2. 软件环境

+ 需要准备已经安装好etcd的3台机器，参考[二进制安装etcd](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_2_install_etcd.html)，[kubeadm安装etcd](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_3_install_etcd_with_kubeadm.html)，etcd的版本是3.4.9

+ 同时，要检查etcd的证书

  ``` bash
  $ /etc/kubernetes/
  ├── manifests
  │   └── etcd.yaml
  └── pki
      ├── apiserver-etcd-client.crt
      ├── apiserver-etcd-client.key
      └── etcd
          ├── ca.crt
          ├── ca.key
          ├── healthcheck-client.crt
          ├── healthcheck-client.key
          ├── peer.crt
          ├── peer.key
          ├── server.crt
          └── server.key
  
  3 directories, 11 files
  ```

## 3. 安装

### 3.1. 安装nginx

负载均衡可以选择Nginx，Haproxy，lvs或者traefik甚至apache都可以，基本上所有的4层负载均衡或者7层负载均衡都可以，负载均衡的主要作用就是前端使用一个统一的IP地址，后端映射api-server。让每个node通讯的时候，都通过负载均衡器来调度请求。

这里，我们就使用最常见，最容器实现的nginx来做负载均衡。下面的操作需要在lb机器上做。

- 安装nginx

```bash
$ yum -y install nginx
```

- 我这边使用AWS的linux安装的

``` bash
amazon-linux-extras install nginx1.12
```

- 在`/etc/nginx/nginx.conf`里面添加一个include，让nginx读取目录下的配置文件

```bash
include /etc/nginx/conf.d/tcp.d/*.conf;
```

- 添加kubernetes的4层代理配置文件`/etc/nginx/conf.d/tcp.d/kube-api-server.conf`

```c
stream {
    log_format main '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log /var/log/nginx/k8s-access.log main;
    upstream k8s-apiserver {
        server 10.0.11.202:6443;
        server 10.0.12.249:6443;
        server 10.0.13.82:6443;
    }
    server {
        listen 10.0.1.94:6443;
        proxy_pass k8s-apiserver;
    }
}
```

- 查看端口是否在监听了

```bash
netstat -untlp|grep 6443
tcp        0      0 10.0.1.94:6443          0.0.0.0:*               LISTEN      3410/nginx: master
```

### 3.2. 安装docker

+ 私有云[点这里](https://developer.aliyun.com/mirror/docker-ce?spm=a2c6h.13651102.0.0.3e221b11OiZivH)

+ AWS

  ``` bash
  yum -y install docker
  ```

注意：一定要把docker的cgroups的方式和kubelet的cgroup方式修改成一致的，否则会报错

``` bash
# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://gvfjy25r.mirror.aliyuncs.com"]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
```

### 3.3. kubeadm，kubelet

+ 正常方式[点这里](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
+ 国内环境[点这里](https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.3e221b11OiZivH)

### 3.4. 准备kubeadm配置文件

+ 创建`kubeadm-config.yaml`

``` bash
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"
etcd:
    external:
        endpoints:
        - https://ETCD_0_IP:2379
        - https://ETCD_1_IP:2379
        - https://ETCD_2_IP:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
```

+ 修改`kubeadm-config.yaml`

``` bash
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "10.0.1.94:6443"
etcd:
    external:
        endpoints:
        - https://10.0.11.201:2379
        - https://10.0.12.248:2379
        - https://10.0.13.81:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
networking:
  podSubnet: "192.168.0.0/16"
imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
```

+ 初始化第一个节点

``` bash
kubeadm init --config kubeadm-config.yaml --upload-certs
```

+ 成功后会出现提示

``` bash
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.0.1.94:6443 --token nzjpz8.vkfaw9phnwh32jol \
    --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb \
    --control-plane --certificate-key 24a95f134489a05e39168c21135f7ea67152568fcc6e9d69105400fb1d008f81

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.1.94:6443 --token nzjpz8.vkfaw9phnwh32jol \
    --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb
```

+ 配置kubelet

``` bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

+ 在其他的master节点上执行

``` bash
  kubeadm join 10.0.1.94:6443 --token nzjpz8.vkfaw9phnwh32jol \
    --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb \
    --control-plane --certificate-key 24a95f134489a05e39168c21135f7ea67152568fcc6e9d69105400fb1d008f81
```

+ 在其他的worker节点上执行

``` bash
kubeadm join 10.0.1.94:6443 --token nzjpz8.vkfaw9phnwh32jol \
    --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb
```

