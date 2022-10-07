---
title: k8s 1.23.0+kubeadm+centos7
keywords: keynotes, architect, HA
permalink: keynotes_L4_architect_1_HA_8_8_k8s_1_23_0_kubeadm_centos7.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/8_k8s_1_23_0_kubeadm_centos7
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

在生产系统中，我们为了保证可用性，通常使用三个节点

## 2. 架构

![k3s](/pages/keynotes/L4_architect/1_HA/pics/7_k3s/how-it-works-k3s.svg)

整个架构和k8s架构简单，只不过rancher把api-server，scheduler，controller-manager和sqlite做了整合，打包成为一个进程，而agent则包含了k8s node节点上的所有功能，同样打包成了一个文件。

如果想实现高可用，我们就需要在数据库上下手，也就是要使用外部的数据库来实现高可用，替换掉只能本地运行的sqlite，目前来说，支持下面几种[数据库](https://rancher.com/docs/k3s/latest/en/installation/datastore/#external-datastore-configuration-parameters)

- Embedded [SQLite](https://www.sqlite.org/index.html)
- [PostgreSQL](https://www.postgresql.org/) (certified against versions 10.7 and 11.5)
- [MySQL](https://www.mysql.com/) (certified against version 5.7)
- [MariaDB](https://mariadb.org/) (certified against version 10.3.20)
- [etcd](https://etcd.io/) (certified against version 3.3.15)
- Embedded etcd for High Availability (experimental)

最后一个内置的高可用etcd正在开发当中，看来有望彻底和k8s靠拢。

### 3. 安装

