---
title: Helm简介
keywords: keynotes, senior, kubernetes, helm
permalink: keynotes_L3_senior_2_kubernetes_1_helm.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/1_helm
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 了解helm概念
- 了解helm v2与helm v3的区别
- 安装和使用helm v3

## 1. helm概念

我们在kubernetes集群中创建应用的时候，一般会使用清单文件manifest来规范我们的应用。典型的容器化应用程序会包含很多清单。Deployment、Service和ConfigMap的清单。我们可能还会创建一些Secret、Ingress和其他对象。每一个都需要一份清单。

从1.4版本开始，为了让软件的部署更加的规范化，Helm项目应运而生。Helm类似于yum或者apt这样的包管理器，而Chart类似于包。有了Helm，我们就可以打包所有这些清单文件，并使它们作为一个单独的tar包使用。我们可以将tar包放在一个存储库中，搜索该存储库，发现一个应用程序，然后使用一个命令部署并启动整个应用程序。

服务器运行在Kubernetes集群中，而客户端是本地的，也可以使本地笔记本电脑。我们可以使用客户端来连接到多个应用程序存储库。

## 2. helm v2与helm v3

### 2.1. Helm v2和Tiller

helm工具使用一系列YAML文件将Kubernetes应用程序打包成一个chart或包。这种方式允许用户之间简单的共享，使用模板方案进行优化，以及来源跟踪等。

![zgxbwiuexxtg-BasicFlow-HelmandTiller](/pages/keynotes/L3_senior/4_helm/pics/1_helm/zgxbwiuexxtg-BasicFlow-HelmandTiller.png)

### 2.2. 基础的Helm和Tiller的流程

Helm的v2版本是由两个组件构成的

+ 服务器端叫Tiller，运行在kubernetes集群之中
+ 客户端叫Helm，在我们本地的机器上运行

Helm v2会在集群中部署一个Tiller的pod，但是这样会引发很多安全和集群权限的问题。而Helm v3就不用部署这个pod了。

使用Helm客户端，我们就可以浏览包存储库（包含已发布的Chart），并在Kubernetes集群上部署这些Chart。kubernetes将下载Chart并将请求传递给Tiller来创建应用。这个应用是由运行在Kubernetes集群中的各种资源组成。

### 2.3. Helm v3

最近Helm彻底的翻修了一遍，流程和命令都有很大的改变。如果我们正在使用Helm的v2版本，那么可能需要花些时间去升级和集成这些改变。

最显著的变化之一就是Tiller的移除。这是一个一直存在的安全问题，因为pod需要提升权限才能部署Chart。在新版本中，这个功能被单独放在了命令行中，不再需要初始化才能使用。

在v2中，对chart和deployment的更新需要使用双向策略合并来完成。这需要将先前的manifest和预期的menifest进行比较，而不是在**helm**命令之外进行编辑。目前的检查是使用另外的方法，检查目前对象的状态。

还有其他的改变，比如软件安装不再自动生成名称。我们必须手动指定名称，要不就传递`--generated name`参数

## 3. 部署helm v3

由于helm的v2版本存在一些安全问题，所以全部转向v3是大势所趋，这里就不再说v2的内容了，我们直接部署v3。

+ 官方文档[在这里](https://helm.sh/docs/intro/install/)

+ 下载地址[在这里](https://github.com/helm/helm/releases/tag/v3.2.4)

### 3.1. 安装helm

我们这里以linux64版本举例，我们首先要保证kubectl命令能够正确连接到我们的kubernetes集群

+ 下载

  ``` bash
  $ wget https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz
  ```

+ 解压

  ``` bash
  $ tar xf https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz
  ```

+ 挪到环境变量所在的位置

  ``` bash
  mv linux-amd64/helm /usr/local/bin/helm
  chmod +x /usr/local/bin/helm
  ```

+ 验证是否可用

  ``` bash
  helm help
  The Kubernetes package manager
  
  Common actions for Helm:
  
  - helm search:    search for charts
  - helm pull:      download a chart to your local directory to view
  - helm install:   upload the chart to Kubernetes
  - helm list:      list releases of charts
  ...
  ```

+ 为helm添加一个仓库

  ``` bash
  $ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
  ```

+ 查看仓库里面的包

  ``` bash
  helm search repo stable
  
  NAME                                    CHART VERSION   APP VERSION                     DESCRIPTION
  stable/acs-engine-autoscaler            2.2.2           2.1.1                           DEPRECATED Scales worker nodes within agent pools
  stable/aerospike                        0.2.8           v4.5.0.5                        A Helm chart for Aerospike in Kubernetes
  stable/airflow                          4.1.0           1.10.4                          Airflow is a platform to programmatically autho...
  stable/ambassador                       4.1.0           0.81.0                          A Helm chart for Datawire Ambassador
  # ... and many more
  ```

  

### 3.2. 管理应用

+ 部署应用

  ``` bash
  $ helm repo update              # Make sure we get the latest list of charts
  $ helm install stable/mysql --generate-name
  Released smiling-penguin
  ```

+ 列出应用

  ``` bash
  $ helm ls
  NAME             VERSION   UPDATED                   STATUS    CHART
  smiling-penguin  1         Wed Sep 28 12:59:46 2016  DEPLOYED  mysql-0.1.0
  ```

+ 卸载应用

  ``` bash
  $ helm uninstall smiling-penguin
  Removed smiling-penguin
  ```

  