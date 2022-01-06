---
title: Rancher简介
keywords: keynotes, senior, kubernetes, Rancher
permalink: keynotes_L3_senior_2_kubernetes_4_Rancher.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/4_Rancher
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 了解常见的kubernetes管理工具
- rancher简介
- 安装Rancher
- 使用rancher部署一个集群

## 1. kubernetes管理工具

我们这里说的管理工具，其实在CNCF项目的[landscape](https://landscape.cncf.io/category=platform&format=card-mode&grouping=category)中是有四个分类的

+ [Certified Kubernetes - Distribution](https://landscape.cncf.io/category=certified-kubernetes-distribution&format=card-mode&grouping=category) 发行商，大部分都是在kubenetes中加入了各种工具，或者自定义资源，让kubernetes的功能更加完善，更加趋于PaaS平台
+ [Certified Kubernetes - Hosted](https://landscape.cncf.io/category=certified-kubernetes-hosted&format=card-mode&grouping=category) 主持kubernetes，实际上就是一些公有云厂商，可以把kubernetes直接运行在他们的平台上
+ [Certified Kubernetes - Installer](https://landscape.cncf.io/category=certified-kubernetes-installer&format=card-mode&grouping=category)这个好理解，就是安装kubernetes的一些工具。
+ [PaaS/Container Service](https://landscape.cncf.io/category=paa-s-container-service&format=card-mode&grouping=category)PaaS云，可以把代码直接运行在云上打包运行为容器而不用关心操作系统。

个人意见，其实在管理工具方面，我推荐的是两个，一个是红帽的openshift，他是一个非常完善的PaaS平台。而另外一个就是Rancher，我觉得从功能和界面来看，他的潜力是非常大的。

## 2. Rancher简介

### 2.1. Rancher和它的项目们

Rancher是公司的名字，也是他主要项目的名字，Rancher项目本身是一个带有图形界面的工具，但是，他也提供了命令行方式和API接口，是一个github上的开源项目，遵循apache2.0协议，大家可以免费下载和使用。当然，他免费的是软件，收费的是服务，如果企业想要选用Rancher作为自己的容器管理平台，要么，有牛人可以hold住这头小牛，要么就买rancher的服务。平台再好，不会使用，没有最佳实践，最后只能是给自己挖坑。

Rancher被誉为容器化的一个非常完整的解决方案。他几乎集成了DevOps所有的工具，是一个非常好用的，支持不同架构的多kubernetes集成平台。

业界对于Rancher非常看好，就在咱们这篇文章截稿的前一天，2020年3月17号，rancher完成D轮4000万美元融资。而在几个月前，CEO梁胜入选CNCF 2020技术监督委员会（TOC），成为了CNCF圆桌骑士之一。Rancher可谓是如火如荼，占领了容器管理软件市场的的量份额。

同时，rancher公司的其他项目也非常有潜力，比如：

+ kubernetes发行版rke
+ 适用于物联网的k3s和k3os
+ 最近GA的、在CNCF孵化的项目longhorn

我们的课程会在后面的架构师课程中专门放一个rancher专题，我们这里会带大家感受一下使用图形界面，通过点点点去管理kubernetes。

### 2.2. 功能

+ 第一个多容器云管理平台

+ 让我们可以专注于想做的事情，不需要担心软件或者硬件
  + 降低kubernetes学习的成本（# Rancher来帮你搭建kubernetes）
  + 提高kubernetes的适应性（# Rancher来帮你操作kubernetes）
  + 让开发着专注写代码（# Rancher来帮你管理kubernetes）
  + 加速创新（# Rancher来帮你升级kubernetes）

## 3. 安装Rancher

+ 我们使用最简单的docker方式来安装rancher

  ``` bash
  
  ```

+ 访问http://你的IP，会让我们重置密码

  ![image-20200629161404349](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629161404349.png)

+ 输入URL地址，以后就会使用https访问了

  ![image-20200629162023678](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162023678.png)

+ 现在我们登陆为admin用户

  ![image-20200629162103821](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162103821.png)

+ 点击add cluster

  ![image-20200629162134099](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162134099.png)

+ 选择AWS（或者任意你喜欢的，不过我们这里使用AWS为例子）

  ![image-20200629162352617](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162352617.png)

+ 添加templete

  ![image-20200629162624337](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162624337.png)

+ 输入key的信息

  ![image-20200629162829503](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162829503.png)

+ 选择vpc和subnet

  ![image-20200629162921280](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162921280.png)

+ 选择新建安全组

  ![image-20200629162949511](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629162949511.png)

+ 选择镜像的信息，不选就是用ubuntu

  ![image-20200629163135620](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629163135620.png)

+ 起个名字就可以create了

  ![image-20200629163224379](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629163224379.png)

+ 填写必须的信息，如果想要创建集群的话，就在这里多创建几个模板，让每个模板在不同的region，不同的vpc，不同的sg，然后分配不同的职责，从而构建集群。

  ![image-20200629164104658](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629164104658.png)

+ 创建就好了

  ![image-20200629164143052](/pages/keynotes/L3_senior/2_kubernetes/pics/4_Rancher/image-20200629164143052.png)

