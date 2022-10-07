---
title: 使用kubespray创建集群
keywords: keynotes, architect, HA, kubespray_ubuntu
permalink: keynotes_L4_architect_1_HA_5_kubespray_ubuntu.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/5_kubespray_ubuntu
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 简介

kubespray是kubernetes官方推荐的部署方法之一，他代表了另外一个比较流行的kubernetes管理方法，利用其它自动化管理工具框架，通过编写执行流程来安装部署kubernetes。流行的框架就比如说ansible这种传统的，或者是terraform这种新兴的。而kubespray对于这些都有涉猎，他支持在公有云上安装kubernetes集群，而在IaaS层面上就是使用的terraform在AWS,GCE,Azure上面部署，在部署kubernetes的时候，就使用ansible这种编排工具来安装。其实，我感觉他更像是一个个小工具的集合，功能很全面，但是每个功能都是独立存在的，并没有太大的联系，从零构建kubernetes集群的时候，好像是一个个的pipeline。

## 2. 功能

- 可以部署在 **AWS, GCE, Azure, OpenStack, vSphere, Packet (bare metal), Oracle Cloud Infrastructure (实验阶段), or 裸机**
- 高可用集群
- 可以选择网络方案
- 支持大部分的Linux发行版
- 持续集成测试

## 3. 快速部署

+ 安装ansible和python3-pip

``` bash
sudo apt-get update && apt install -y ansible python3-pip
```

+ git项目到管理机

``` bash
git clone https://github.com/kubernetes-sigs/kubespray.git
```

+ 安装python需要的依赖包

``` bash
cd kubespray/
sudo pip3 install -r requirements.txt
```

+ 把配置的模板复制一份

``` bash
cp -rfp inventory/sample inventory/mycluster
```

+ 把所有节点的IP声明一下

``` bash
declare -a IPS=(10.0.11.219 10.0.12.225 10.0.13.101 10.0.12.155 10.0.13.217)
```

+ 生成配置文件

``` bash
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

+ 这两个文件就是所有的清单文件了，想要修改集群的信息，就在里面改，不修改的话，是两个master和3个worker

``` bash
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
```

+ 最后就是play了

``` bash
ansible-playbook --private-key /tmp/js.key -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

+ 等待一段时间，登录到第一台机器，也就是10.0.11.219上，使用kubectl命令

``` bash
root@node1:~# kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    master   35m   v1.16.8
node2   Ready    master   34m   v1.16.8
node3   Ready    <none>   33m   v1.16.8
node4   Ready    <none>   34m   v1.16.8
node5   Ready    <none>   33m   v1.16.8
```

## 4. 参考文档

kubespray中还有很多的小工具，比如在AWS使用terraform创建资源，[点这里](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform/aws)。我就不一一演示了，我想告诉他家的是一种思路，使用编排工具安装kubernetes的思路。而kubespray就是代表之一，如果想查看更多相关的信息，可以参考[代码仓库](https://github.com/kubernetes-sigs/kubespray)