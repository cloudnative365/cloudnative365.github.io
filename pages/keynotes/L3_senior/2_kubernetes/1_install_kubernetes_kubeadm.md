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
  | 10.0.1.196  | lb       | loadbalance和jumpserver                | nginx                                  |
  | 10.0.11.233 | control1 | apiserver/controller-manager/scheduler | apiserver/controller-manager/scheduler |
  | 10.0.12.101 | control2 | control plane                          | apiserver/controller-manager/scheduler |
  | 10.0.13.172 | control3 | control plane                          | apiserver/controller-manager/scheduler |
  | 10.0.11.233 | etcd1    | etcd host                              | etcd                                   |
  | 10.0.12.101 | etcd2    | etcd host                              | etcd                                   |
  | 10.0.13.172 | etcd3    | etcd host                              | etcd                                   |
  | 10.0.12.109 | node1    | worker node                            | kubelet/kube-proxy                     |
  | 10.0.13.202 | node2    | worker node                            | kubelet/kube-proxy                     |

+ loadbalance的选型：支持4层负载均衡的均衡器都可以，但是如果在生产上一定要做两个负载均衡来保证

+ 使用kubeadm安装的kubernetes集群会把kube-api，kube-scheduler和kube-controller-manager都运行为容器，而kubelet和容器runtime（docker或者containerd）会托管给systemd

+ 为了满足国内朋友的需求，我会使用国内的环境来搭建，所以咱们会指定镜像仓库为国内的仓库（阿里云的仓库）

### 2.2. 软件环境

+ 需要准备已经安装好etcd的3台机器，参考[二进制安装etcd](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_2_install_etcd.html)，[kubeadm安装etcd](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_3_install_etcd_with_kubeadm.html)，etcd的版本是3.4.9

+ 同时，要检查etcd的证书

  ``` bash
  /etc/kubernetes/pki
  ├── apiserver-etcd-client.crt
  ├── apiserver-etcd-client.key
  └── etcd
      ├── ca.crt
      ├── healthcheck-client.crt
      ├── healthcheck-client.key
      ├── peer.crt
      ├── peer.key
      ├── server.crt
      └── server.key
  ```

## 3. 安装

### 3.1. 安装nginx

负载均衡可以选择Nginx，Haproxy，lvs或者traefik甚至apache都可以，基本上所有的4层负载均衡或者7层负载均衡都可以，负载均衡的主要作用就是前端使用一个统一的IP地址，后端映射api-server。让每个node通讯的时候，都通过负载均衡器来调度请求。

这里，我们就使用最常见，最容器实现的nginx来做负载均衡。

- 安装nginx

```bash
$ yum -y install nginx
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

- 查看端口是否在监听了

```bash
netstat -untlp|grep 6443
tcp        0      0 10.0.1.157:6443         0.0.0.0:*               LISTEN      15787/nginx: master
```