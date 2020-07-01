---
title: 安装ingress-nginx
keywords: keynotes, senior, kubernetes, install_ingress_nginx
permalink: keynotes_L3_senior_2_kubernetes_2_install_ingress_nginx.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/2_install_ingress_nginx
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 控制ingress控制器

## 1. 安装ingress控制器

[nginx官方文档](https://docs.nginx.com/nginx-ingress-controller/installation/?_ga=2.87051160.1809112633.1593511491-2130300903.1593399596)上提供了三种安装方法，镜像，manifest或者是helm。我们比较常用的是使用manifest或者helm，咱们这里使用manifest安装。

### 1.1. 使用manifest安装

+ 从git上下载

  ``` bash
  $ git clone https://github.com/nginxinc/kubernetes-ingress/
  $ cd kubernetes-ingress/deployments
  $ git checkout v1.7.2
  ```

+ 创建namespace

  ``` bash
  $ kubectl apply -f common/ns-and-sa.yaml
  ```

+ 配置RBAC

  ``` bash
  $ kubectl apply -f rbac/rbac.yaml
  ```

+ 创建TLS证书

  ``` bash
  $ kubectl apply -f common/default-server-secret.yaml
  ```

+ nginx的配置文件

  ``` bash
  $ kubectl apply -f common/nginx-config.yaml
  ```

+ 自定义资源

  ``` bash
  # VirtualServer
  $ kubectl apply -f common/vs-definition.yaml
  # VirtualServerRoute
  $ kubectl apply -f common/vsr-definition.yaml
  # TransportServer
  $ kubectl apply -f common/ts-definition.yaml
  ```

+ （可选）如果想配置四层（TCP，UDP）的负载均衡就运行下面的

  ``` bash
  # 自定义资源
  $ kubectl apply -f common/gc-definition.yaml
  # 全局配置
  $ kubectl apply -f common/global-configuration.yaml
  ```

+ 配置Ingress控制器

  ``` bash
  $ kubectl apply -f deployment/nginx-ingress.yaml
  ```

+ 或者选择DeamonSet的Ingress控制器

  ``` bash
  $ kubectl apply -f daemon-set/nginx-ingress.yaml
  ```

+ 创建nodePort方式的Service

  ``` bash
  $ kubectl create -f service/nodeport.yaml
  ```

  