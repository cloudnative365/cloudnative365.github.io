---
title: thanos高可用
keywords: keynotes, L4_architect, monitoring, thanos_HA
permalink: keynotes_L4_architect_2_monitoring_26_thanos_HA.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/26_thanos_HA
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

thanos是无状态服务，所以并不存在传统集群中的数据同步，防脑裂，集群选举等问题。因为他的数据源是prometheus，数据最终会存储在对象存储（s3，minIO等）中。他就好像是一个后台应用，前台数据就是prometheus暴露出来的metrics，后台数据库就是对象存储。所以说，其实最好的方式是把thanos放在kubernetes中进行部署和管理是最佳实践。

## 2. 架构

![arch](/pages/keynotes/L4_architect/2_monitoring/pics/26_thanos_HA/thanos.jpg)

这张图我们在前面已经介绍过了，这次我们需要从高可用架构上注意一下红色框的位置是需要高可用的

+ grafana：前端是需要高可用的，由于用户有可能会创建很多dashboard，而存储的位置是pgsql，所以我们就需要在pgsql做一个HA就行了。需要注意的是，grafana的数据是存在数据库中的，但是插件是不存在数据库中的，我们需要在每个节点上进行安装（grafana-cli plugin install）
+ alertmanager：自带高可用，主要是防止重复报警或者漏发报警，我们后面讲。
+ prometheus：我们使用replica来表示他的高可用，而每个replica都需要一个thanos sidecar来辅助
+ minIO：上次介绍过了，统一入口只有一个，需要借助nginx来实现，当然，任何7层的负载均衡都可以实现。如果我们的环境中有存储和虚拟化（hpyervisor），还可以有更多玩法。
+ pgsql：上次也介绍过了，我们的pgsql不仅仅是为grafana的用户数据提供持久存储，还可以为zabbix，harbor，gitlab等大部分云原生软件提供持久存储，我们后面讲CI/CD时候还会用到。
+ consul：所有被监控的endpoint都会作为consul的service存储在consul中。

最后，我们这次使用kubernetes来构建这些无状态的应用，包括exporters，都放在k8s集群中管理。也就是说，grafana，alertmanager，prometheus，thanos都放在k8s集群中。minIO，pgsql，consul都作为独立进程来管理。而consul，minIO由于特性决定，我们可以使用本地磁盘作为存储介质。也就是说把consul，minIO应用部署在k8s集群中，而使用hostpath方式把本地磁盘给consul和minio作为存储介质，然后绑定应用到某个节点，来实现高可用。

## 3. 安装thanos

### 3.1. k8s集群

首先，我们要有一个k8s集群，请参考我们架构师第一部分的内容。

### 3.2. prometheus

### 3.3. thanos sidecar

