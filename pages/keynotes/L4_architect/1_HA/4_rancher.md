---
title: 使用Rancher管理我们的集群
keywords: keynotes, advanced, kubernetes_tools, install_kubernetes, ranche
permalink: keynotes_L2_advanced_2_kubernetes_tools_1_install_kubernetes_4_rancher.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/4_rancher
typora-root-url: ../../../../../cloudnative365.github.io
---



## 1. 简介

Rancher被誉为容器化的一个非常完整的解决方案。他几乎集成了DevOps所有的工具，是一个非常好用的，支持不同架构的多kubernetes集成平台。

![Rancher Logo](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/rancher-logo-horiz-color.png)

业界对于Rancher非常看好，就在咱们这篇文章截稿的前一天，2020年3月17号，rancher完成D轮4000万美元融资。而在几个月前，CEO梁胜入选CNCF 2020技术监督委员会（TOC），成为了CNCF圆桌骑士之一。Rancher可谓是如火如荼，占领了容器管理软件市场的的量份额。



## 2. 功能

### 2.1. 第一个多容器云管理平台

没啥好说的，起步非常早，支持的平台非常多，我们今天的实验使用rancher来管理我们手动部署的rancher集群和在AWS上创建集群

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/screen2-thumbnail.svg)

### 2.2. 让我们可以专注于想做的事情，不需要担心软件或者硬件

+ 降低kubernetes学习的成本（# Rancher来帮你搭建kubernetes）

+ 提高kubernetes的适应性（# Rancher来帮你操作kubernetes）

+ 让开发着专注写代码（# Rancher来帮你管理kubernetes）

+ 加速创新（# Rancher来帮你升级kubernetes）

### 2.3. 100%免费使用的开源软件，原生支持kubernetes

Rancher本身是一款开源软件，但是就好像Linux一样，即便是开源，也一样可以引领风骚。他免费的是软件，收费的是服务，如果企业想要选用Rancher作为自己的容器管理平台，要么，有牛人可以hold住这头小牛，要么就买rancher的服务。平台再好，不会使用，没有最佳实践，最后只能是给自己挖坑。

## 3. 安装与配置rancher
+ 安装
``` bash
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:latest
```
+ 配置

+ 访问 https://你的IP ，出现更改密码的界面

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/2222971f8e2c62a347e1201584675924.png)

+ 修改一下访问的url，这个一般是绑定DNS地址用的，我们默认就好了

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/22279f246d0eb877a600a01584675900.png)

+ 界面很清新

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222beb66fcdb52323f11601584675997.png)

+ 点击右上角的Add Cluster

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222784b060f5140cff4b101584676012.png)

+ 会出现很多种方式

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/2225f60535b4bd66a902301584676030.png)

+ 我推荐的是使用Import an existing cluster方式

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222806558c6f3bc47759801584676057.png)

+ 还记得我们前几天用kubeadm创建集群的时候使用的apt方式么？这个在树莓派上同样适用

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222485f8af7c4f4d66f9301584676090.png)

+ 他会生成一些语句，我们只要在被导入的集群中运行命令就好了

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222bb554ec600be5143d001584676094.png)

## 4. 选择AWS作为provider快速创建集群

+ 创建集群，选择AWS

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222a0c7fb2a181f85335e01584676380.png)

+ 填写一些信息

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222841979c6940b0ba38901584676858.png)

+ 点击`Add Node Template`，添加节点的模板

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/22257b57203122012a83101584688457.png)

+ 第一次使用的话需要填入AWS的认证信息

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/2223eb57203122012a83101584688652.png)

+ 选择VPC和网络

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222ca2d72f2e0bc64267601584688790.png)

+ 选择安全组

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/22211da2a3c54f2cdcd8101584688818.png)

+ 其他的一些信息

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222713bb9ffac58de4fb101584688982.png)

+ 创建完成之后就可以使用模板了

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/22289060dd95943d4ca0101584689078.png)

+ 其他选项默认就好，云供应商选AWS，这样方便使用一些AWS的功能，比如使用nlb作为集群的负载均衡器等

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222415dec83d923b6ba5101584689203.png)

+ 后面就是一段时间的耐心等待了

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/222b7af706dfe7f48dae801584689252.png)

+ 完成的状态下，就可以点进去看状态了

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/1_install_kubernetes/pics/4_rancher/22272247fe6d39d4beb0401584697776.png)

其他的功能大家就看图形界面自己发掘吧，我就不给大家剧透了。毕竟我们不是做产品宣传。