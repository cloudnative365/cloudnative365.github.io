---
title: 使用Rancher管理我们的集群
keywords: keynotes, advanced, kubernetes_tools, install_kubernetes, ranche
permalink: keynotes_L2_advanced_2_kubernetes_tools_1_install_kubernetes_4_rancher.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/4_rancher
typora-root-url: ../../../../../../cloudnative365.github.io
---



## 1. 简介

Rancher被誉为容器化的一个非常完整的解决方案。他几乎集成了DevOps所有的工具，是一个非常好用的，支持不同架构的多kubernetes集成平台。

![Rancher Logo](https://rancher.com/imgs/rancher-logo-horiz-color.png)

业界对于Rancher非常看好，就在咱们这篇文章截稿的前一天，2020年3月17号，rancher完成D轮4000万美元融资。而在几个月前，CEO梁胜入选CNCF 2020技术监督委员会（TOC），成为了CNCF圆桌骑士之一。Rancher可谓是如火如荼，占领了容器管理软件市场的的量份额。



## 2. 功能

### 2.1. 第一个多容器云管理平台

没啥好说的，起步非常早，支持的平台非常多，我们今天的实验使用rancher来管理我们手动部署的rancher集群和在AWS上创建集群

![img](https://rancher.com/imgs/why-rancher/screen2-thumbnail.svg)

### 2.2. 让我们可以专注于想做的事情，不需要担心软件或者硬件

+ 降低kubernetes学习的成本（# Rancher来帮你搭建kubernetes）

+ 提高kubernetes的适应性（# Rancher来帮你操作kubernetes）

+ 让开发着专注写代码（# Rancher来帮你管理kubernetes）

+ 加速创新（# Rancher来帮你升级kubernetes）

### 2.3. 100%免费使用的开源软件，原生支持kubernetes

Rancher本身是一款开源软件，但是就好像Linux一样，即便是开源，也一样可以引领风骚。他免费的是软件，收费的是服务，如果企业想要选用Rancher作为自己的容器管理平台，要么，有牛人可以hold住这头小牛，要么就买rancher的服务。平台再好，不会使用，没有最佳实践，最后只能是给自己挖坑。

### 3. 安装Rancher

使用Docker的方式安装，简单粗暴，我们目前使用的是2.3.0版本

``` bash
docker run -d --restart=unless-stopped \
-p 80:80 -p 443:443 \
rancher/rancher:latest
```

完成后访问本地的https://localhost，出现更改密码的界面

![image-20200318115653807](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318115653807.png)

修改一下访问的url，这个一般是绑定DNS地址用的，我们默认就好了

![image-20200318115810545](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318115810545.png)

界面很清新

![image-20200318115831870](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318115831870.png)

点击右上角的Add Cluster

![image-20200318145119601](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318145119601.png)

会出现很多种方式

![image-20200318145241020](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318145241020.png)

我推荐的是使用Import an existing cluster方式

![image-20200318145404153](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318145404153.png)

还记得我们前几天用kubeadm创建集群的时候使用的apt方式么？这个在树莓派上同样适用

![IMG_4212](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/IMG_4212.png)

他会生成一些语句，我们只要在被导入的集群中运行命令就好了

![image-20200318150338407](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/image-20200318150338407.png)