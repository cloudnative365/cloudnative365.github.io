---
title: Ceph概述
keywords: keynotes, architecture, storage, object_storage, ceph_overview
permalink: keynotes_L7_architect_storage_1_storage_1_1_ceph_overview.html
sidebar: keynotes_L7_architect_sidebar
typora-copy-images-to: ./pics/1_1_ceph_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Ceph概述

## 2. Ceph架构

## 3. Ceph初体验

### 3.1. 环境

|       | OS         | IP           | hostname             | Disk | Ceph   | function     |
| ----- | ---------- | ------------ | -------------------- | ---- | ------ | ------------ |
| node1 | CentOS 7.9 | 10.39.64.234 | ceph-ittools-prod-01 | 500G | 15.2.9 | master, data |
| node2 | CentOS 7.9 | 10.39.64.244 | ceph-ittools-prod-02 | 500G | 15.2.9 | master, data |
| node3 | CentOS 7.9 | 10.39.64.249 | ceph-ittools-prod-03 | 500G | 15.2.9 | master, data |

### 3.2. 初始化环境

+ 修改epel源，/etc/yum.repos.d/ceph.repo

  ``` bash
  [ceph]
  name=Ceph packages for $basearch
  baseurl=https://download.ceph.com/rpm-15.2.9/el7/$basearch
  enabled=1
  priority=2
  gpgcheck=1
  gpgkey=https://download.ceph.com/keys/release.asc
  
  [ceph-noarch]
  name=Ceph noarch packages
  baseurl=https://download.ceph.com/rpm-15.2.9/el7/noarch
  enabled=1
  priority=2
  gpgcheck=1
  gpgkey=https://download.ceph.com/keys/release.asc
  
  [ceph-source]
  name=Ceph source packages
  baseurl=https://download.ceph.com/rpm-15.2.9/el7/SRPMS
  enabled=0
  priority=2
  gpgcheck=1
  gpgkey=https://download.ceph.com/keys/release.asc
  ```

+ 修改/etc/hosts文件

  ``` bash
  10.39.64.234    ceph-ittools-prod-01
  10.39.64.244    ceph-ittools-prod-02
  10.39.64.249    ceph-ittools-prod-03
  ```

+ 安装cephadm，只在第一台机器

  ``` bash
  yum install -y ceph-deploy
  ```

+ 创建cephadm用户用来管理集群

  ``` bash
  groupadd -r -g 2024 cephadm && useradd -r -m -s /bin/bash -u 2024 -g 2024 cephadm && echo cephadm:Passw0rd | chpasswd
  ```

+ 并且给cephadm用户sudo权限

  ``` bash
  echo "cephadm ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ceph
  ```

+ 配置node1（管理节点）到其他节点的ssh免密登录（省略具体过程了）

  ``` bash
  ssh-keygen
  su - cephadm
  ssh-copy-id 10.39.64.234
  ssh-copy-id 10.39.64.244
  ssh-copy-id 10.39.64.249
  ```

+ 准备开始创建集群

  ``` bash
  mkdir ceph-cluster
  cd ceph-cluster/
  ```

### 3.3. 部署ceph

+ 在node1节点上生成配置文件

  ``` bash
  ceph-deploy new --cluster-network 10.39.64.224/27 --public-network 10.0.0.0/8 ceph-ittools-prod-01
  ```

+ 通过配置文件初始化三个节点上面的安装包

  ``` bash
  ceph-deploy install --no-adjust-repos --nogpgcheck ceph-ittools-prod-01 ceph-ittools-prod-01 ceph-ittools-prod-01
  ```

+ 初始化node1节点上的mon进程

  ``` bash
  ceph-deploy --overwrite-conf mon create-initial
  ```

+ 分发秘钥到其他节点

  ``` bash
  ceph-deploy admin ceph-ittools-prod-01 ceph-ittools-prod-02 ceph-ittools-prod-03
  ```

+ 看一下所有节点上的可用磁盘

  ``` bash
  ceph-deploy disk list ceph-ittools-prod-01 ceph-ittools-prod-02 ceph-ittools-prod-03
  ```

+ 格式化磁盘

  ``` bash
  ceph-deploy disk zap ceph-ittools-prod-01 /dev/sdb
  ceph-deploy disk zap ceph-ittools-prod-02 /dev/sdb
  ceph-deploy disk zap ceph-ittools-prod-03 /dev/sdb
  ```

  

