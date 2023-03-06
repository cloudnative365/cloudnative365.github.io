---
title: opensearch
keywords: keynotes, architect, observability, log, opensearch, opensearch_basic
permalink: keynotes_L5_architect_observability_2_log_1_1_opensearch.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_1_opensearch
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ 认识OpenSearch
+ OpenSearch和ElasticSearch的产品线比较
+ 体验OpenSearch
+ OpenSearch最佳实践
+ OpenSearch和其他日志类产品的比较

## 1. OpenSearch

### 1.1. 从ElasticSearch到OpenSearch

2021年1月15日，Elastic 公司 CEO Shay Banon 在公司官网发文表示，他们决定将 Elasticsearch 和 Kibana 的开源协议由 Apache 2.0 变更为 SSPL 与 Elastic License。PDF版本点[这里](https://storage.courtlistener.com/recap/gov.uscourts.cand.348946/gov.uscourts.cand.348946.1.0_2.pdf)

```
“亚马逊于 2015 年基于 Elasticsearch 推出自己的服务，还将其称为 Amazon Elasticsearch Service，这是很明显的商标侵权行为。NOT OK。”

“我在 2011 年借了一笔个人贷款来注册 Elasticsearch 商标...... 看到商标如此公然地滥用，我特别痛苦。亚马逊问题迫使我们提起诉讼。NOT OK。”

“商标问题让用户感到困惑，以为是 Elastic 和亚马逊之间有合作，这不是真的。NOT OK。”

“...... 多年来这种困惑仍然存在。NOT OK。”

“亚马逊针对 Elasticsearch 的 Open Distro 分支，进一步分裂了我们的社区，引发了相当多的混乱。NOT OK。”

“..... 最近，我们发现了更多挑战道德底线的例子。我们已经在专有功能方面上与众不同，现在这些设计却被视为来自亚马逊的灵感。NOT OK。”
```

SSPL 是由 MongoDB 制定的源代码许可。针对云服务提供商做出了限制，即要求云服务提供商在未对项目做出贡献的情况下，不得发布自己的开源产品即服务。SSPL 允许用户以自由且不受限制的方式使用并修改代码成果，唯一的要求是：如果将产品以作为一种服务进行交付，那么必须同时公开发布所有关于修改及 SSPL 之下管理层的源代码。

同年四月，那是一个春天，AWS在推出了OpenSearch，OpenSearch 项目由 OpenSearch (fork Elasticsearch 7.10.2) 和 OpenSearch Dashboards (fork Kibana 7.10.2) 组成，包括企业安全、告警、机器学习、SQL、索引状态管理等功能。OpenSearch 项目中的所有软件均采用了 Apache License 2.0 开源许可协议。

AWS 介绍称，他们推出的 OpenSearch 删除了 Elasticsearch 中受 Elastic 商业许可证限制的功能、代码和商标，以兼容 Apache License 2.0，自称这是每个人都可以构建和创新的基础，任何人无需签署 CLA (Contributor License Agreement) 即可为项目贡献代码。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/2021-opensearch-service-2-6882604.png)

### 1.2. RoadMap

OpenSearch的1.0版本GA是在2021年7月14日。

![Prospective OpenSearch Release Schedule](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/1-6882610.png)

| OpenSearch version                                           | Release highlights                                           | Release date     |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :--------------- |
| [1.1.0](https://github.com/opensearch-project/opensearch-build/tree/main/release-notes/opensearch-release-notes-1.1.0.md) | Adds cross-cluster replication, security for Index Management, bucket-level alerting, a CLI to help with upgrading from Elasticsearch OSS to OpenSearch, and enhancements to high cardinality data in the anomaly detection plugin. | 5 October 2021   |
| [1.0.1](https://github.com/opensearch-project/opensearch-build/tree/main/release-notes/opensearch-release-notes-1.0.1.md) | Bug fixes.                                                   | 1 September 2021 |
| [1.0.0](https://github.com/opensearch-project/opensearch-build/tree/main/release-notes/opensearch-release-notes-1.0.0.md) | General availability release. Adds compatibility setting for clients that require a version check before connecting. | 12 July 2021     |
| [1.0.0-rc1](https://github.com/opensearch-project/opensearch-build/tree/main/release-notes/opensearch-release-notes-1.0.0-rc1.md) | First release candidate.                                     | 7 June 2021      |
| [1.0.0-beta1](https://github.com/opensearch-project/opensearch-build/tree/main/release-notes/opensearch-release-notes-1.0.0-beta1.md) | Initial beta release. Refactors plugins to work with OpenSearch. | 13 May 2021      |

截止到今天，OpenSearch的最新版本是1.1版，是在2021年10月5日发行的。

关于项目今后的一些计划，我们可以关注[github](https://github.com/orgs/opensearch-project/projects/1)

![image-20211026145514433](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/image-20211026145514433-6882615.png)



## 2. OpenSearch和ElasticSearch的产品线比较

### 2.1. 项目的对比

我们可以参考这个[网站](https://www.libhunt.com/compare-OpenSearch-vs-elasticsearch)

![image-20211026162443499](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/image-20211026162443499-6882619.png)

### 2.2. OpenSearch的功能

OpenSearch提供了很多开源ES中不可用的功能

| **Features**                                                 | **Description**                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Advanced Security](https://github.com/opensearch-project/security) | Offers encryption, authentication, authorization, and auditing features. They include integrations with Active Directory, LDAP, SAML, Kerberos, JSON web tokens, and more. OpenSearch also provides fine-grained, role-based access control to indices, documents, and fields. |
| [SQL Query Syntax](https://github.com/opensearch-project/sql) | Provides the familiar SQL query syntax. Use aggregations, group by, and where clauses to investigate your data. Read data as JSON documents or CSV tables so you have the flexibility to use the format that works best for you. |
| [Reporting](https://github.com/opensearch-project/dashboards-reports) | Schedule, export, and share reports from dashboards, saved searches, alerts, and visualizations. |
| [Anomaly Detection](https://github.com/opensearch-project/anomaly-detection) | Use machine learning anomaly detection based on the [Random Cut Forest (RCF) algorithm](https://github.com/aws/random-cut-forest-by-aws) to automatically detect anomalies as your data is ingested. Combine with [alerting](https://github.com/opensearch-project/alerting) to monitor data in near real time and send alert notifications automatically. |
| [Index Management](https://github.com/opensearch-project/index-management) | Define custom policies to automate routine index management tasks, such as rollover and delete, apply them to indices and index patterns, and [transforms](https://opensearch.org/docs/im-plugin/index-transforms/index/). |
| [Performance Analyzer and RCA Framework](https://github.com/opensearch-project/performance-analyzer) | Query numerous cluster performance metrics and aggregations. Use PerfTop, the command line interface (CLI) to quickly display and analyze those metrics. Use the root cause analysis (RCA) framework to investigate performance and reliability issues in clusters. |
| [Asynchronous Search](https://github.com/opensearch-project/asynchronous-search) | Run complex queries without worrying about the query timing out with Asynchronous Search queries running in the background. Track query progress and retrieve partial results as they become available. |
| [Trace Analytics](https://github.com/opensearch-project/trace-analytics) | Ingest and visualize OpenTelemetry data for distributed applications. Visualize the flow of events between these applications to identify performance problems. |
| [Alerting](https://github.com/opensearch-project/alerting)   | Automatically monitor data and send alert notifications to stakeholders. With an intuitive interface and a powerful API, easily set up, manage, and monitor alerts. Craft highly specific alert conditions using OpenSearch’s full query language and scripting capabilities. |
| [k-NN search](https://github.com/opensearch-project/k-NN)    | Using machine learning, run the nearest neighbor search algorithm on billions of documents across thousands of dimensions with the same ease as running any regular OpenSearch query. Use aggregations and filter clauses to further refine similarity search operations. k-NN similarity search powers use cases such as product recommendations, fraud detection, image and video search, related document search, and more. |
| [Piped Processing Language](https://github.com/opensearch-project/piped-processing-language) | Provides a familiar query syntax with a comprehensive set of commands delimited by pipes (\|) to query data. |
| [Dashboard Notebooks](https://github.com/opensearch-project/dashboards-notebooks) | Combine dashboards, visualizations, text, and more to provide context and detailed explanations when analyzing data. |

### 2.3. 组件的对比

![OpenSearch Projects](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/22-6882624.png)

从图上可以看出，原来的ES实例，就是OpenSearch实例。而Kibana在OpenSearch体系里面叫做OpenSearch Dashboards。原来为ES开发的插件OpenDistro完全变成了OpenSearch的插件，来实现我们刚才说的那些功能，且完全免费。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/1HrKE3nPzZSjCRKplfJySHg-6882628.png)

以X-Pack为例，在OpenSearch中叫做OpenSearch-security，以plugin的形式随着二进制包一起被下载。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/1_u3TYQ5-_1IZJEiM_Umipg-6882633.png)

## 3. 体验OpenSearch

[官方](https://opensearch.org/docs/latest/#docker-quickstart)提供了Docker的方式，但是我们这里使用二进制包的方式，搭建一个单机的OpenSearch。

### 3.1.  准备

+ OpenSearch依赖于Java，尽管[官方](https://opensearch.org/docs/latest/opensearch/install/compatibility/)说JDK8和11都支持，但是使用JDK8会有Bug，我们就使用JDK11

+ 而操作系统依然是那些常见的Red Hat Enterprise Linux 7, 8; CentOS 7, 8; Amazon Linux 2; Ubuntu 16.04, 18.04, 20.04

+ 程序不能由root用户启动，所以需要创建一个用户

  ``` bash
  useradd -M -r opensearch
  ```

+ 不管使用什么操作系统，在正式安装之前，我们需要修改一些[参数](https://opensearch.org/docs/latest/opensearch/install/important-settings/)

  ``` bash
  # 修改/etc/sysctl.conf
  vm.max_map_count=262144
  # reload生效
  sudo sysctl -p
  # 修改JAVA的启动参数，一般配置成整个系统内存的一半
  OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
  # nofile参数根据不通的系统可能会有所不同，我们这里使用的是rhel/centos系统，需要配置下面两个地方
  vim /etc/security/limits.conf
  * soft nofile 65536
  * hard nofile 131072
  * soft nproc 65536
  * hard nproc 131072
  # 同时配置用户资源参数
  vim /etc/security/limits.d/20-nproc.conf
  opensearch soft nproc 65536
  ```

### 3.2. 安装

+ 下载安装包：https://opensearch.org/downloads.html

+ 解压安装包：

  ``` bash
  tar xf opensearch-1.0.0-linux-x64.tar.gz -C /data
  ```

+ 创建数据和日志目录

  ``` bash
  mkdir -pv /data/opensearch/{data,logs} 
  chown -R opensearch:opensearch /data/opensearch
  chown -R opensearch:opensearch /data/opensearch-1.1.0
  ```

+ 修改配置文件/data/opensearch-1.1.0/config/opensearch.yml：

  ```
  # 指定集群名称和主机名
  cluster.name: opensearch-cluster
  node.name: node01
   
  # 数据目录
  path.data: /data/opensearch/data
   
  # log目录
  path.logs: /data/opensearch/logs
   
  # 禁用交换内存
  bootstrap.memory_lock: true
   
  # 修改监听地址，外部机器也可以访问
  network.host: 0.0.0.0
   
  # 默认的端口号
  http.port: 9200
   
  # 设置单机模式运行
  discovery.type: single-node
  ```

### 3.3. 启动

+ 启动OpenSearch进程

  ```
  切换到opensearch用户启动
  su - opensearch
  ./opensearch-tar-install.sh
  ```

+ 验证

  ```
  curl -XGET https://localhost:9200 -u 'admin:admin' --insecure
  curl -XGET https://localhost:9200/_cat/plugins?v -u 'admin:admin' --insecure
  ```

+ 我们会发现，其实这个启动脚本已经默认把证书配置好了，下次再启动就直接使用bin/opensearch命令启动

### 3.4. opensearch-dashboard

+ 下载：https://opensearch.org/downloads.html

+ 解压

  ``` bash
  tar -zxvf opensearch-dashboards-1.1.0-linux-x64.tar.gz -C /data
  ```

+ 修改配置文件opensearch-dashboards-1.1.0/config/opensearch_dashboards.yml

  ``` bash
  # 添加监听地址，外部机器也可以访问
  server.host: 0.0.0.0
  opensearch.hosts: ["https://localhost:9200"]
  ```

+ 修改权限

  ``` bash
  chown -R opensearch:opensearch /data/opensearch-dashboards-1.1.0
  ```

+ 启动

  ``` bash
  切换到opensearch用户启动
  su - opensearch
  ./bin/opensearch-dashboards
  ```

+ 验证

  ![img](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/ZmFuZ3poZW5naGVpdGkw-6882638.png)

## 4. OpenSearch最佳实践

### 4.1. 集群

![multi-node cluster architecture diagram](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/cluster-6882642.png)

和ES一样，OpenSearch节点有多种功能，配置可以看[这里](https://opensearch.org/docs/latest/opensearch/cluster/)

| Node type         | Description                                                  | Best practices for production                                |
| :---------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `Master`          | Manages the overall operation of a cluster and keeps track of the cluster state. This includes creating and deleting indices, keeping track of the nodes that join and leave the cluster, checking the health of each node in the cluster (by running ping requests), and allocating shards to nodes. | Three dedicated master nodes in three different zones is the right approach for almost all production use cases. This configuration ensures your cluster never loses quorum. Two nodes will be idle for most of the time except when one node goes down or needs some maintenance. |
| `Master-eligible` | Elects one node among them as the master node through a voting process. | For production clusters, make sure you have dedicated master nodes. The way to achieve a dedicated node type is to mark all other node types as false. In this case, you have to mark all the other nodes as not master-eligible. |
| `Data`            | Stores and searches data. Performs all data-related operations (indexing, searching, aggregating) on local shards. These are the worker nodes of your cluster and need more disk space than any other node type. | As you add data nodes, keep them balanced between zones. For example, if you have three zones, add data nodes in multiples of three, one for each zone. We recommend using storage and RAM-heavy nodes. |
| `Ingest`          | Preprocesses data before storing it in the cluster. Runs an ingest pipeline that transforms your data before adding it to an index. | If you plan to ingest a lot of data and run complex ingest pipelines, we recommend you use dedicated ingest nodes. You can also optionally offload your indexing from the data nodes so that your data nodes are used exclusively for searching and aggregating. |
| `Coordinating`    | Delegates client requests to the shards on the data nodes, collects and aggregates the results into one final result, and sends this result back to the client. | A couple of dedicated coordinating-only nodes is appropriate to prevent bottlenecks for search-heavy workloads. We recommend using CPUs with as many cores as you can. |

### 4.2. SSL证书

在上面的Demo中，我们会发现使用这个脚本`opensearch-tar-install.sh`启动之后，配置文件中会默认多出几个关于SSL证书的选项，OpenSearch实际上是帮我们默认生成了一些证书，并且把他们添加到配置文件当中去了。也就是说，我们在访问默认的9200端口的时候，需要使用httpsx协议，且需要信任证书。并且，如果我们跳过这个启动脚本，直接在bin目录下用opensearch命令启动的话，会无法启动实例。

ssl证书的作用不仅仅是加密了访问端口，并且对于数据，也是使用这个证书来加密的，也就是说，如果我们要创建多节点的集群环境或者生成密码，加载配置文件等操作，也需要使用这些证书进行相应的操作。

### 4.3. 容量估算

容量估算的公式基本和ES的估算方式一样，且各个功能节点的预估也和ES一样。可以参考4.1中`Best practices for production`一列

### 4.4. 认证集成

和ES的x-pack一样，常见的认证方式都可以支持。

![image-20211027095159060](/pages/keynotes/L5_architect_observability/2_log/pics/1_1_opensearch/image-20211027095159060.png)

需要注意的是，权限的认证，授权是分开的，是基于RBAC的授权，且只能在配置文件中修改，然后同步到opensearch的隐藏表里面，opensearch-dashboard的图形界面里面是没办法配置的

## 5. OpenSearch vs Loki vs Splunk

### 5.1. 市面上的log产品

我们可以从这个[网站](https://stackshare.io/elk/alternatives)来看elk同类的产品，这个网站有免费，也有收费产品

![image-20211027114328790](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/image-20211027114328790-6882647.png)

如果要CNCF认证的产品，我们可以到[cncf网站](https://landscape.cncf.io/)去找，找到[logging](https://landscape.cncf.io/card-mode?category=logging&grouping=category)这一栏

![image-20211027114750971](/pages/keynotes/L5_architect_observability/2_Log/pics/1_1_opensearch/image-20211027114750971-6882652.png)

### 5.2. OpenSearch vs Loki vs Splunk

+ 产品比较

|           | OpenSearch | ElasticSearch | Loki        | Splunk             |
| --------- | ---------- | ------------- | ----------- | ------------------ |
| 定位      | 搜索引擎   | 搜索引擎      | 日志存储    | 安全工具           |
| 收费情况  | 免费       | 部分免费      | 免费        | 收费，按日志量收费 |
| 公司/社区 | AWS        | Elastic       | Grafana Lab | Splunk             |

OpenSearch和Elasticsearch本是同根生，在数据引擎方面基本一致，不一致的部分是一些插件，而真正满足客户需要的正是这些插件。ElasticSearch的插件基本是吸收了社区中比较优秀的代码，然后整合到了自己的产品中，比如xPack。而OpenSearch的插件是基于Open Distro，Open Distro是AWS自己基于免费版ElasticSearch而开发的一些功能。

Loki是Grafana Lab的一款产品，虽然影响力没有ES这么大，但是也是有自己的市场的，Loki主要解决了日志大量写的问题，因为Loki是没有索引的，所以他的读写效率都比较平均，适合海量存储。

Splunk的定位是安全产品，他的亮点在于对于日志的分析，他提供的大量模板让用户在应对安全问题时可以很快的找到切入点。

+ 性能比较

![image-20211027143002209](/pages/keynotes/L5_architect_observability/2_log/pics/1_1_opensearch/image-20211027143002209.png)

