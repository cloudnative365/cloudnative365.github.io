---
title: kind
keywords: keynotes, architect, k8s_tools, kind
permalink: keynotes_L4_architect_2_k8s_tools_1_1_kind.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_1_kind
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 简介

Kind

## 2. 功能

## 3. 安装和部署

### 3.1. runtime

我们可以直接使用yum或者apt-get来安装docker，只要我们的操作系统是最新版基本都没什么问题。安装docker不是关键，关键的是containerd，凡是比较新的docker都是docker+containerd的架构，而docker对于k8s来说直接就寄了，k8s要用的是containerd。

+ centos

  ``` bash
  yum -y install docker
  ```

+ ubuntu

  ``` bash
  apt-get install docker.io
  ```

### 3.2. 安装Kind

+ 下载二进制包并且丢到$PATH下面就行，[官方文档](https://github.com/kubernetes-sigs/kind)

  ``` bash
  curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.16.0/kind-$(uname)-amd64"
  chmod +x ./kind
  mv ./kind /usr/local/sbin
  ```

### 3.3. 安装Kubectl

+ 由于kind只负责创建集群，管理工具需要我们自己下载，我们就手动下载kubectl，[官方文档](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

  ``` bash
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x kubectl
  mv kubectl /usr/local/sbin
  ```

  

## 4. 创建集群

最简单的方式就是`kind create cluster`

## 5. 相关工具

### 5.1. Helm

+ 官方的源国内无法访问，我们直接使用脚本方式下载和安装，如果git的raw地址访问不了，就直接用网页打开，把内容粘贴下来执行

  ``` bash
  $ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  $ chmod 700 get_helm.sh
  $ ./get_helm.sh
  ```

### 5.2. Ingress
