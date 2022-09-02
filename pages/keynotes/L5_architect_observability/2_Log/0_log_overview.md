---
title: 日志概述
keywords: keynotes, architect, observability, log, log_overview
permalink: keynotes_L5_architect_observability_2_log_0_log_overview.html
sidebar: keynotes_L5_architect_observability
typora-copy-images-to: ./pics/0_log_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 日志系统

和监控系统类似，日志系统从架构上也分为采集，储存，展示和报警这几个部分。

### 1.1. 采集

采集上分为主动推送和被动采集两种方式

+ 主动推送：我们最常见的应该就是Linux的syslog服务了，log进程（syslog，rsyslog，jounal）通过syslog协议把本机产生的日志推送到syslog服务器。既然有syslog协议，那么也就有其他的协议，比如http协议或者tcp协议，这些方式基本都是由程序自己来实现，比如经典的ELK，promtail+loki。
+ 被动采集：这里的被动采集并不是单纯的被动采集，而是有一个中间层，比如：程序主动把日志写入kafka，然后fluentd去kafka拿日志，最后写入elasticsearch。

### 1.2. 储存

目前最流行的免费日志解决方案基本都是以ElasticSearch为底层存储来实现的。截止到这篇文章的成文时间（2022年7月26日），Elastic公司已经把ElasticSearch的版本推进到了8版本。而目前市面上常见的系统，基本还都是以7版本为主。

我们可以简单的把ES看做数据库，那么增删改查就成为了ES的主要功能。

### 1.3. 展示

不同的解决方案展示的工具不尽相同，但是也有一些兼容性比较好的工具，比如grafana，可以兼容常见工具，用来做统一展示是非常好的选择。如果想要更深层次的功能，比如数据分析，可能就要借助其他工具了，比如kibana。

但是，kibana上的这类功能，比如机器学习之类，都是需要购买商业版才能有的，这就大大限制了我们的发挥空间。

### 1.4. 报警

和展示功能一样，kibana上的报警功能同样需要购买商业版，这就让我们非常不爽。我们急需一种免费又好用的解决方案。

## 2. 常见的日志解决方案

### 2.1. ELK/EFK

这种是我们最常见到的解决方案，ELK是指ElasticSearch，Logstash和Kibana这三款由elastic公司提供的软件。ElasticSearch作为存储，Logstash作为日志收集工具，Kibana用来展示和报警。

Logstash被人诟病的地方在于他的运行机制是非常小众的Jruby，也就是开发语言是ruby，但是运行在JVM虚拟机上，这就导致了他运行起来非常笨重。特别是在云原生环境环境中，作为sidecar来运行的时候，只为了采集某一个小程序的日志，就需要运行一个有可能比程序本身还消耗资源的sidecar显然是不太合适的。

针对这种情况，我们开始寻求新的解决方案，那就是EFK，使用Fluentd来替代Logstash作为新的采集工具。Fluentd运行起来消耗的资源只有logstash百分之一甚至更少，这就使得他在容器环境中更加的便捷。

但是，fluentd并没有就此止步，他后来又推出了新的日志收集工具fluentbit，进一步缩小了日志收集工具的运行所需资源，变成了目前最为流行的fluentbit-->fluentd-->elasticsearch-->kibana。

### 2.2. Opensearch

ElasticSearch的免费版有两种，一个叫开源版，代码是公开的，一个叫社区版，免费试用。这就导致了很多商业软件和公有云厂商都来薅他的羊毛，特别是近年来的公有云厂商尤为疯狂。公有云厂商有完善的开发团队，通过对免费代码二次封装，很容易就包装成了自己的ES商业版并进行售卖。另外一个决定性因素就是ElasticSearch的开源版是基于apache协议的，他是完全免费试用并且针对二次封装等行为也不会收费。Elastic公司在2021年把矛头指向AWS，职责他剽窃ElasticSearch，并且在ES的7.10.3版本正式修改apache协议为SSPL协议，主要是针对公有云厂商，如果在公有云上使用开源版ES也同样需要付费给Elastic公司。

但是这种程序的行为并没有让AWS妥协，而是激发了新的产品，也就是OpenSearch。OpenSearch是基于ES的7.10.2版，也就是最后一个apache协议的版本独立了一个分支出来的。而AWS很早就有一套针对ES收费版功能的对策，那就是Opendistro。这套插件可以完全适配ES开源版并且经过了非常多的实践验证，这一次，aws把opendistro嵌入es7.10.2版本中，形成了一个新的es分支，那就是OpenSearch。而Kibana被branch之后叫做OpenSearch dashboard。

这套方案目前已经被AWS应用在了公有云上，在创建ES实例的时候可以选择ES或者OpenSearch了。也就是说这套方案已经相对成熟了，我们也可以看到虽然OpenSearch是完全免费的，但是他的更新速度和商业软件比，一点也不慢，从前阵子的Log4j的安全漏洞的例子来看，基本就是0day。而且目前也升级到了2.0版本，算是发展非常迅猛的一套产品了。

### 2.3. Loki

Loki是grafana lab的产品，基本上是Promtail-->Loki-->Grafana的架构来做的，同样是免费的解决方案。这套方案相比于上面两种就轻量多了，因为ES和OpenSearch主要是搜索引擎，日志搜索不过是他众多功能之一，而Loki本来就是为了存储日志而生，他没有使用lucene那种搜索引擎，所以Loki是没有索引概念的，他借鉴了prometheus的存储方式，使用类似时序数据库的方式来存储日志，这就让Loki本身的写日志非常快，而查询的话，相对ES来说还是慢了许多，更别提日志的分析功能了，所以这个方案本身就是为了方便日志的存储。

### 2.4. 其他开源解决方案

依然建议大家去CNCF的网站上去看一下![image-20220727212108528](/pages/keynotes/L5_architect_observability/2_Log/pics/0_log_overview/image-20220727212108528.png)

里面同样有很多不错的开源解决方案，比如graylog，sumo logic，这就需要我们大家去深挖了。
