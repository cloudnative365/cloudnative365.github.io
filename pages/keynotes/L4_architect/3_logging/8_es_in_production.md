---
title: 生产级别的es
keywords: keynotes, architect, logging, 8_es_in_production
permalink: keynotes_L4_architect_3_logging_8_es_in_production.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/8_es_in_production
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

在ES集群上扩展master node

在ES集群上扩展data node

在ES集群上扩展ingest node

在ES上备份特定的index

## 1. 概述

生产级别的ES，顾名思义，是用在我们生产系统上的ES。在生产上的ES就要考虑我们贯穿整个架构师课程的观念了，监控，集群，备份和安全。安全的配置咱们前面说过了，唯一没说的是和ldap或者ad的集成，由于免费版本并不提供AD的集成，所以没办法给大家做演示。

### 1.1. 监控

监控有两种方式，一种是使用prometheus的elasticsearch_exporter，或者使用kibana的监控。鉴于kibana的监控功能实在是弱爆了，还是建议大家使用prometheus+alertmanager的方式。

### 1.2. 集群

高可用方面，elasticsearch本身是自带高可用的，集群之间有分片，多副本，数据同步等机制。

负载均衡方面有两种，一种是master+data的方式。master作为请求的分发器和数据的聚合器和客户端直接打交道，所有的请求由master进行分发和聚合。而data节点用来纯粹的索引和存储数据。

如果数据再增多，就会用到ingest节点，这个节点主要用来对数据进行预处理的。在他们真正的被存放在ES之前进行一定的聚合。这个功能默认在所有节点是启用的，也就是说所有的节点都可以做聚合。我们为了提高效率通常会找一个或者几个节点专门做这个工作，然后把其他节点的这个选项配置为false`node.ingest: false`。也就是说这个节点会作为一个集群的门户接受请求。

``` bash
----------   ----------   --------
|        |   |        |   |      |
| ingest |==>| master |==>| data |
|        |   |        |   |      |
----------   ----------   --------
```

### 1.3. 备份

官方文档中，对于备份，只提供了snapshot级别的，针对index的snapshot。并没有提及把整个集群进行备份的说明，换句话说，如果要做节点级别的备份，就需要对整个磁盘进行备份了，或者借助三方的软件进行备份，比如veeam。

### 1.4. 安全

安全方面我们前面讲过了x-pack和searchguard两种。从功能来讲，两者基本相同，特别免费和收费的功能同样相同，我还是建议使用x-pack，他毕竟是es官方的产品，各方面来说，兼容性都是非常好的。可以唯一不同的就是收费版的价格了，那么大家有机会和厂商合作的时候再详细了解好了。

### 1.5. kibana高可用

kibana本身是无状态的，他的数据是存储在ES中的，会有一个索引叫`.kibana`，所以如果直接使用kibana是不需要额外配置高可用的。其实kibana本身是有一个组件kibana-proxy的，也就是用他来作为负载均衡器使用，但是这个功能是收费的，只有和kibana，es的集成认证，没有其他功能，所以我们通常会使用一些负载均衡软件，比如nginx来做负载均衡。

## 2. 配置生产级别的ES

