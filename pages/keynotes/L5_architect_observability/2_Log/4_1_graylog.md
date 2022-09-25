---
title: graylog初体验
keywords: keynotes, architect, observability, log, gray, gray_basic
permalink: keynotes_L5_architect_observability_2_log_4_1_graylog_basic.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/4_1_graylog_basic
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Graylog

[Graylog](https://www.graylog.org/)是一款专门为日志相关需求而开发的软件。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/628d20c15e5d13777989e579_Graylog_Full Color.svg)

也就是说我们常用的日志功能他基本都具备了，比如聚合、分析、审计、展示和报警工具。因为他是专门为日志系统而开发的，所以他相对于传统的ELK系统要简单。注意，我说的是简单，而不是轻量，因为graylog是需要依托于ElasticSearch来做日志引擎的。因此，可以认为是ElasticSearch的功能扩展。相比于Loki来说，graylog并没有在本质上改变日志存储介质，只是在日志的存取的过程中，把用户常用功能全都包装在了graylog程序当中，让使用更加简单而已。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/graylog_functions.jpeg)

Graylog和Elastic类似，提供免费版（Open Source）和收费版（Operations和Security），而收费版包括公有云和自建两种方式。

![image-20220925154112779](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220925154112779.png)

比如报警，机器学习，预测，运维仪表盘和发送报告等功能都是需要收费的。也就是说，graylog的免费版只能实现日志的存储和查询。当然，如果有大神喜欢自己开发插件也是支持的，但是他的社区贡献者和ES相比还是逊色许多。

## 2. 功能与架构

Graylog server由三个部分组成，graylog程序，ES和mongoDB。

+ ES用来实现日志的存取，比较消耗内存。
+ 而MongoDB用来实现graylog本身的一些配置信息。因此MongoDB的资源消耗并不大，如果资源紧张可以和Graylog放在一起。而ES单独部署，这样会合理利用资源。
+ Graylog服务是一个带Web接口的服务，是一个计算密集型服务，建议把CPU多分一点。

![architec_small_setup](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/architec_small_setup.png)

Graylog相对于ELK来说，相当于在ES外面包了一层，取代了原来ELK架构中ES的位置，从而实现对日志的优化。

![architecturecomparison.png](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/architecturecomparison.png)



## 3. 单机版安装和配置

### 3.1. 环境

- 操作系统：Centos7
- Java 1.8：
- Elasticsearch 7.10.2/OpenSearch 1.3.5：https://artifacts.opensearch.org/releases/bundle/opensearch/1.3.5/opensearch-1.3.5-linux-x64.tar.gz
- MongoDB 4.4

### 3.2. Graylog

+ Graylog[下载地址](https://www.graylog.org/downloads-2)。我们[下载](https://downloads.graylog.org/releases/graylog/graylog-4.3.7.tgz)最新版。
