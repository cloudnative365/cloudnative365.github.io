---
title: 生产级别的Opensearch
keywords: keynotes, architect, logging, opensearch
permalink: keynotes_L4_architect_3_logging_11_opensearch_in_production.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/11_opensearch_in_production
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

+ OpenSearch插件
+ OpenSearch集群
+ OpenSearch安全

## 1. OpenSearch插件

从上一篇文章中，我们知道，OpenSearch插件的前身就是Open Distro。这些插件是默认安装的。他们被安装在`OPENSEARCH_HOME/plugins`目录下面，



## 2. OpenSearch集群

2.1. 节点的类型

| 节点类型        | 作用                                                         | 机器配置                     |
| --------------- | ------------------------------------------------------------ | ---------------------------- |
| master          | 索引的创建或删除<br />跟踪哪些节点是集群的一部分<br />决定哪些分片分配给相关的节点 | CPU 内存 消耗一般            |
| Master-eligible | 参与集群选举                                                 | CPU 内存 消耗一般            |
| data            | 存储索引数据<br />对文档进行增删改查,聚合操作                | 资源大户，主要消耗磁盘，内存 |
| Ingest(提取)    | Ingest节点和集群中的其他节点一样，但是它能够创建多个处理器管道，用以修改传入文档。类似 最常用的Logstash过滤器已被实现为处理器。<br />Ingest节点 可用于执行常见的数据转换和丰富。 处理器配置为形成管道。 在写入时，Ingest Node有20个内置处理器，例如grok，date，gsub，小写/大写，删除和重命名<br />在批量请求或索引操作之前，Ingest节点拦截请求，并对文档进行处理。 | CPU 内存 消耗一般            |
| Coordinating    | 协调节点，是一种角色，而不是真实的Elasticsearch的节点，你没有办法通过配置项来配置哪个节点为协调节点。集群中的任何节点，都可以充当协调节点的角色。当一个节点A收到用户的查询请求后，会把查询子句分发到其它的节点，然后合并各个节点返回的查询结果，最后返回一个完整的数据集给用户。在这个过程中，节点A扮演的就是协调节点的角色。 | 资源大户，主要消耗磁盘，内存 |





