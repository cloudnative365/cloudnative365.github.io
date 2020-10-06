---
title: consul高可用
keywords: keynotes, L4_architect, monitoring, consul_HA
permalink: keynotes_L4_architect_2_monitoring_27_consul_HA.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/27_consul_HA
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

consul从数据存储形式来看，是kv存储，但是我们更喜欢叫做注册中心。和etcd，consul，zookeeper一样，是目前比较流行的解决方案。我们上次搭建了一个单机版的，用他来做prometheus的注册中心。我们通过curl方式来注册和管理服务，然后我们只需要在prometheus中配置consul为数据源，prometheus就可以动态更新endpoint信息，而不用重新启动服务了。

## 2. 架构

### 2.1. 架构图

![consul_ha](/pages/keynotes/L4_architect/2_monitoring/pics/27_consul_HA/consul_ha.png)

### 3.2. server

由若干的server组成，其中一个是leader，其他的是follower，为了保证集群的基本功能，我们建议最少有3个server端，因为看名字就能看出来，consul集群是典型的raft方式，所以建议使用至少3个节点。

而在server运行的同时，还会有一个agent运行。但是这个agent不是负责服务发现或者设置和获取kv，而是负责自身节点和节点上服务运行状况进行健康检查的。

### 3.3. client

我们上次用单机的方式搭建了一个consul的server，其实consul还有另外一个方式，就是client。我们可以向server端发起请求，注册服务，也可以去找client，实际上client更像是一个proxy，或者是ES中的master节点。如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190318150146698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NoaXBzbHlj,size_16,color_FFFFFF,t_70)

## 4. 集群搭建

### 4.1. 准备

+ 下载

  ``` bash
  
  ```

+ 需要开放的端口

  + **8300**：服务端RPC，TCP。
  + **8301**：Serl LAN：处理LAN gossip，TCP UDP。
  + **8302**：Serl WAN：处理LAN gossip，TCP UDP。
  + **8500**：HTTP API，TCP.
  + **8600**：DNS，TCP,UDP

### 4.2. 启动集群

``` bash
consul agent -server=true -bootstrap-expect=3 -data-dir=/tmp/consul -node=consul -bind=xx.xx.xx.xx（本机Ip） -ui -client=0.0.0.0
```

