---
title: 日志系统概览
keywords: keynotes, senior, logging, logging_overview
permalink: keynotes_L3_senior_6_logging_1_logging_overview.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/1_logging_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 日志系统

日志系统从我们学习Linux就一直伴随我们，我们做问题定位基本都是靠日志系统来实现的。实际上，我们常见的是系统日志，而真正的日志在广义上包括系统日志和应用日志。现在炒的火热的大数据分析，其实很多都是分析日志数据。一个银行系统，储存了这么多的用户信息，交易信息，也不过几百T的数据。而互联网公司，数据量都是EB，PB级别的。为什么这么多呢？因为主要都是日志，我们在界面上的每个操作，其实都已经被系统记录下来，以日志的方式储存起来，精确度越高，日志量越大。比如我们常见的推荐系统，你点击的每个页面都会被系统拿去做用户行为分析，甚至你在页面上停留的秒数都会被记录下来，所以，我们的日志量非常庞大。

我们常说的日志系统其实由下面几个部分组成：

+ agent：日志收集器，主要负责传输日志。不一定是特定的程序，也许是一个rsyslog的客户端。需要注意的是，我们使用rsyslog的时候，通常使用的是514端口，使用的协议是UDP。UDP是一个不可靠的传输协议，所以真正的生产上，我们不建议使用rsyslog直接传输日志，而是先写到本地，然后由agent收集之后，传送到服务器端。
+ server：对agent传输过来的日志进行整理，或者对于用户的请求进行处理，他一般监听在http或者tcp的某个端口上。
+ 数据库：存储数据的软件，负责组织数据在硬盘，内存中的组织形式
+ 搜索引擎：通常是指我们查询的时候使用的一系列算法的组合。
+ 展示工具：负责展示我们的搜索结果

## 2. 日志系统与搜索引擎

日志的格式基本都是以文本为主，所以搜索引擎首当其冲作为检索的工具是比较合适的，我们常见的开源搜索引擎有

+ lucene：apache，只针对搜索引擎进行了实现
+ ElasticSearch：基于lucence，Java，实现了server，数据库，搜索引擎
+ Solr：基于lucence

但是，搜索引擎是日志系统的一部分，也是比较核心的部分，但是，**搜索引擎不等于日志系统**。日志系统是包含了搜索引擎在内的非常多的组件组合而成的，只不过有些产品，以搜索引擎为核心，配套了很多的周边，形成了一套日志收集系统，比如我们下面要说的elasticsearch就是搜索引擎，他和周边很多产品才构成了我们的日志系统。

所以，我们的题目叫日志系统，而不是搜索引擎，和其他课程讲elasticsearch大数据开发相关知识不同，我们讲的是构建一套生产级别的日志系统，开发的事情还是交给程序员吧，我们运维的主要任务是提供稳定的环境。

## 3. Elastic Stack

### 3.1. Elastic的产品线

目前为止，最新版本是7.9.0，一般来说，如果我们使用的是某一个版本，比如7.9.0，那么相应的其他的产品最好也升级，或者使用相同的版本，否则，日志中会报错。我们先来看一看es的产品线。

原址：https://www.elastic.co/cn/downloads/

![image-20200825171346720](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/image-20200825171346720.png)

![image-20200825171452720](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/image-20200825171452720.png)

+ ElasticSearch：主要产品

+ Kibana：主要的展示平台，使用nodejs开发

+ Beats：相当于agent，里面包含了相当多的beats。下面会解释
+ Logstash：可以当agent用，但是比agent增加了很多数据转换的功能
+ APM：监控系统，可观测性系统，收费功能
+ Elastic企业搜索：SaaS产品，包含全套组件，官方的cloud
+ Elastic Cloud：云托管，比如阿里云，腾讯云
+ Elastic Cloud Enterprise：把SaaS产品下载到自己的机房
+ Elastic Cloud on Kubernetes：在kubernetes上运行Cloud
+ ES-Hadoop：从Hadoop中把数据导入到ES
+ JDBC 客户端：es的java连接，jar包，收费功能
+ ODBC客户端：主要给windows用的
+ Elastic Cloud Control (ecctl)：ES的命令行工具，beta测试
+ X-Pack：早期的（6.2之前的）一些工具集合，大部分收费功能
+ Elastic代理：相当于一个反向代理，beta测试，收费功能

其实早起的ES，只有ES，kibana和logstash产品，也就是我们通常说的ELK，而随着业务的需求，很多爱好者使用插件的方式，为ES添加了很多功能，而这些功能被ES慢慢吸收，用来增强自己的产品线，有的就干脆独立出来做成一款产品，可见ES还是非常受到开源社区的关注的。

### 3.2. beats

beats作为ES的客户端，也就是我们常说的agent。ES把logstash定位为数据转换的工具，而对于不同的场景做了很多的beats，给logstash减减肥。我们只需要安装特定的beats就可以实现某一项工作。

原址：https://www.elastic.co/cn/downloads/beats

![image-20200825172059895](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/image-20200825172059895.png)

+ filebeats：挖日志的
+ Packetbeat：研究网络的情况，丢包率等等
+ winlogbeat：挖window日志的
+ metricbeat：挖掘监控指标的
+ Heartbeat：看设备是否在线
+ Auditbeat：挖审计日志的
+ Functionbeat：监控AWS lamda的
+ Journalbeat：挖掘journal日志的

### 3.3. 定价

大家可以自行参考自己关心的功能，我们这里主要说一下安全的功能。原址：https://www.elastic.co/cn/pricing/

![image-20200825175543107](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/image-20200825175543107.png)

![image-20200825175713628](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/image-20200825175713628.png)

![image-20200825175742172](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/image-20200825175742172.png)

由于企业里面是需要用到认证功能的，所以我们需要安装其他的插件来解决这个问题，我们比较常用的就是searchguard，我们后面会有一章来介绍他。

## 4. Loki

Loki是grafana lab的另外一款产品，他专注的是日志的检索功能，他可以和grafana很好的结合，做出非常漂亮的dashborad。这一款非常轻量的产品，功能，稳定和安全性肯定不能和elasticsearch来比，但是他叫做日志聚合系统（log aggregation），主打的是和grafana很好的结合，完善监控日志的功能。如果作为监控系统使用，我觉得Loki比ES更加适合。

### 4.1. Loki架构图

+ 物理架构

  ![See the source image](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/OIP.hlteEwpZJ72zcHXEzDmgRwHaDJ)

  物理上架构非常简单，日志是通过Promtail组件来挖取的，然后promtail会把日志发送到loki的服务器，最后通过grafana进行展示

+ 逻辑架构

  ![See the source image](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/loki-arch.png)

逻辑架构就比较复杂了，我们后面讲这个时候再一点点说

### 4.2. 展示工具

既然是grafana的产品，不给他单独做一个dashboard就不太合适了，他的dashboard和ES差不多，同样是提供一个输入框，下面可以展示结果。

![See the source image](/pages/keynotes/L3_senior/6_logging/pics/1_logging_overview/loki_grafana_filtering.png)

