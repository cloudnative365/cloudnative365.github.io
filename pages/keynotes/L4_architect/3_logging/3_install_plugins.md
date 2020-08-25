---
title: 日志系统概述
keywords: keynotes, architect, logging, logging_overview
permalink: keynotes_L4_architect_3_logging_1_logging_overview.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_logging_overview
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

了解日志系统

熟悉常见的搜索引擎

学习云原生架构下的日志系统

Elastic Stack

Loki

## 1. 日志系统

日志系统从我们学习Linux就一直伴随我们，我们做问题定位基本都是靠日志系统来实现的。实际上，我们常见的是系统日志，而真正的日志在广义上包括系统日志和应用日志。现在炒的火热的大数据分析，其实很多都是分析日志数据。一个银行系统，储存了这么多的用户信息，交易信息，也不过几百T的数据。而互联网公司，数据量都是EB，PB级别的。为什么这么多呢？因为主要都是日志，我们在界面上的每个操作，其实都已经被系统记录下来，以日志的方式储存起来，精确度越高，日志量越大。比如我们常见的推荐系统，你点击的每个页面都会被系统拿去做用户行为分析，甚至你在页面上停留的秒数都会被记录下来，所以，我们的日志量非常庞大。

我们常说的日志系统其实由下面几个部分组成：

+ agent：日志收集器，主要负责传输日志。不一定是特定的程序，也许是一个rsyslog的客户端。需要注意的是，我们使用rsyslog的时候，通常使用的是514端口，使用的协议是UDP。UDP是一个不可靠的传输协议，所以真正的生产上，我们不建议使用rsyslog直接传输日志，而是先写到本地，然后由agent收集之后，传送到服务器端。
+ server：对agent传输过来的日志进行整理，或者对于用户的请求进行处理，他一般监听在http或者tcp的某个端口上。
+ 数据库：存储数据的软件，负责组织数据在硬盘，内存中的组织形式
+ 搜索引擎：通常是指我们查询的时候使用的一系列算法的组合。
+ 展示工具：负责展示我们的搜索结果

## 2. 搜索引擎

我们常见的搜索引擎有

+ lucene

## 3. 云原生架构下的日志系统



## 4. Elastic Stack



## 5. Loki