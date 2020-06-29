---
title: 监控概述
keywords: keynotes, architect, monitoring, monitoring_overview
permalink: keynotes_L4_architect_2_monitoring_1_monitoring_overview.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_monitoring
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 监控系统

### 1.1. 什么是监控系统

这让我想起了多年前面试的时候的囧事，面试官问我：“你知道什么监控系统？”。我那个时候刚考完RHCE，对于系统架构还处于懵懂状态，我面对这个问题的第一个反应就是这个。

![img](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/273480.jpg)

这个算是监控的一部分，不过是机房级别的监控，和我们要讨论的监控从哲学角度是一个东西，但是从软件角度却大相径庭。

### 1.2. 从运维到监控

我们运维的最重要职责之一就是要保证我们的环境稳定，为了让我们知道环境稳定与否，我们就需要去确认。当然不是人工去确认，而是使用一些摄像头去24小时监控，在我们的监控系统中，我们叫他Agent。而Agent会吧他看到的东西主动或者被动的传递给监控服务器，服务器会有自己的逻辑去判断我们采集来的数据是否符合标准，符合标准的不做任何处理，不符合标准的就需要去报警，来通知运维人员去处理。

我们这里面涉及到的几个概念可以归纳如下

| 图中显示的监控系统     | 一般监控系统   | prometheus监控系统 |
| ---------------------- | -------------- | ------------------ |
| 摄像头                 | agent          | exporter           |
| 好多屏幕组成的监控平台 | server         | prometheus         |
| 值班的人               | 报警系统       | alertmanager       |
| 打电话                 | 发送报警到手机 | 发送报警到微信     |
| 维修工人               | 运维工程师     | 运维工程师         |

### 1.3. 数据采集方式

一般来说，服务器获取数据的方式有两种，一种是推，一种是拉。推是由agent主动向server报告，而拉是由服务器去客户端取数据，而prometheus主要是拉数据。

从1.2的图中可以看出，prometheus的agent叫exporter，也就是打报告的人，这个打报告的人会把系统的状态通过特定端口暴露出来，也就是写成报告，让prometheus去采集，这是一种拉的方式。

### 1.4. 架构图

![Prometheus architecture](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/architecture.png)

从图中我们可以看到我们刚才讲到的拉的方式，而prometheus还有一个组件叫Pushgateway，他是负责收集一次性的指标的，但是他的方式是推，也就是任务执行完成后，会把数据推送到Pushgateway上，再由Prometheus去拉。

在右上角还有一个组件叫AlertManager，他主要是来负责报警的，他支持把报警推送到多个平台，甚至是微信。

### 1.5. 软件清单

因此，我们需要安装的程序如下

+ exporters：有很多的exporter，或者我们可以开发自己的exporter
+ Prometheus Server：时序数据库，可以通过http方式来管理
+ alertmanager：报警系统，但是报警规则要在Prometheus上定义
+ pushgateway：接受一次性metrics推送，然后暴露出来
+ grafana：成图工具，非常绚丽的界面

### 1.6. Prometheus的优点（缺点）

+ 契合google SRE的理念，与kubernetes集成效果卓越，是云原生系统中监控的不二选择

+ 粒度小，可以精确到1~5的采集精度（消耗资源巨大，对硬件性能要求很高）
+ 插件丰富，各种eporter，甚至可以自定义（模板少，需要自己根据环境定义自己的指标）
+ 基于数学计算模型设计，有大量的函数，可以实现复杂的逻辑（对于新手不太友好，入门门槛高）
+ 基于http的存取方式让他非常容易和其他系统集成（有些功能需要和其他的软件一起才能实现）
+ 图形美观（需要和grafana集成）
+ （数据不太适合持久化存储）



## 2. 常用的监控数据库

### 2.1. 监控数据的存储软件

说到监控数据的存储软禁，无非就是在说数据库。咱们现在常见的大部分数据库都是用来存储业务数据的关系型数据库，比如Oracle，MySQL等等。但是对于咱们promethe的监控指标的存储，我们不需要非常复杂的依赖关系，甚至不需要满足三范式。我们只需要根据等量的时间间隔，在某个时间点去采集一下数据就好了，针对这种场景，时序数据库就是最好的选择了。当然，还有很多的监控软件并没有选择时序数据库，他们选择自己创造了自己的数据库。

### 2.2. 常见的数据库产品

数据库从不同的角度可以有多种分类，比如从满足范式的程度，我们可以分为SQL，NoSQL和NewSQL。而我们这里的分类如下：

| Category   | Subcategory                                             | Examples                                                     |
| ---------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| Databases  | Time-series时序数据库                                   | Timescale，KDB，AWS Timestream，OpenTSDB，Prometheus，GridDB，Influxdb |
|            | Industrial IoT data historian工业物联网数据库           | OSI-PI，WonderWare，Rockwell                                 |
|            | Relational关系型数据库                                  | Postgres，MySQL，MariaDB，Oracle，AWS RDS                    |
|            | Document文档数据库                                      | MongoDB，AWS DynamoDB                                        |
|            | Memory内存数据库                                        | Redis                                                        |
|            | Other其他                                               | Cassandra，Neo4j，AWS Elasticache                            |
| Monitoring | Open source infrastructure monitoring开源基础设施监控   | Prometheus，Nagios，Zabbix                                   |
|            | Closed-source infrastructure monitoring闭源基础设施监控 | Datadog，SignalFX，New Relic                                 |
|            | App Performance Management（APM）应用性能管理           | NewRelic，AppDynamics，Datadog，SignalFX                     |
|            | Log Management日志管理                                  | Splunk，Elastic，Sumo Login，Datadog                         |

而在CNCF的landscape里面的可以看到更多的产品

+ 应用数据库，https://landscape.cncf.io/category=database&format=card-mode&grouping=category

![image-20200605110311468](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/image-20200605110311468.png)

+ 监控，https://landscape.cncf.io/category=observability-and-analysis&format=card-mode&grouping=category

![image-20200605110423481](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/image-20200605110423481.png)

这么多软件一个屏幕已经放不开了，我是缩小之后才放在文章中的。图中用蓝框画出来的软件是注册在CNCF下面的软件。粗略的看一下就会发现不管是什么软件都在和云计算挂钩，即使oracle或者DB2这种老牌的数据库在这个时代也会推出一些云功能，或者向云原生靠拢。

### 2.3. 时序数据库

2017年时序数据库忽然火了起来。

+ 2017年2月，Facebook开源了beringei时序数据库
+ 2017年4月，基于PostgreSQL打造的时序数据库TimeScaleDB也开源了
+ 2016年7月，百度云在其天工物联网平台上发布了国内首个多租户的分布式时序数据库产品TSDB
+ opentsdb是基于Hbase的时序数据库，早在2011年就有了1.0版本，但是真正成熟是在2014年5月之后的2.0版本
+ GridDB是在2013年release的，是由C++写成的
+ kdb+（基于q或者k语言的db, 增强版，也简称kdb）被官方称为世界上最快的时间序列数据库

+ Timestream是AWS上的一款SaaS产品，同样是针对物联网的时序数据库
+ Influxdb是由Go语言开发的时序数据库，性能非常稳定，社区版免费试用，商业版支持集群，但是需要收费，我们这个专题会使用Influxdb作为prometheus数据持久化的解决方案，同时，我会教大家怎样使用合理运用架构来使用社区版建立集群。

## 3. 为什么需要监控

### 3.1. 报警

我们前面聊到了监控的作用，就是“插眼”（引用自英雄联盟）。让我们可以持续的观察想要观察的东西，而如果发现了异常的状态之后的动作才是我们需要监控的最终目的。

+ 通常情况下，如果监控到了异常的行为，我们会“报警”（alert），让相关人员知道，然后采取人工的动作。

+ 但是，我们前面学过了kubernetes，kubernetes的pod中有probe的机制，这个机制确实发现了异常之后直接采取action，而不去通知运维工程师。这种就是自动修复的动作。

而我们监控的终极目标就是要自动修复！早先，阿里巴巴每年双11活动的时候，会产生巨大的系统压力，阿里集团各个系统的人员都会集中到阿里巴巴集团的杭州总部待命，在园区各个地方搭起帐篷待命。

![img](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/Img425971672.jpg)

而从2017年开始，自从阿里集团开始使用公有云作为基础设施来提供服务之后，就实现了“喝茶过双十一”。那么，可想而知，如果某一个环节出现了问题，比如资源不足，应用宕机，其实都是由定义好的规则自动修复的。而监控资源，监控应用的监控系统就是整个环节中的第一个重要步骤了，我们会在监控中定义好“底线”，也就是阈值，一旦系统触发了底线，我们的监控就会采取相应的措施，比如：等待3秒，如果不恢复，就告诉自动修复的程序去修复，如果再无法修复，就告诉运维工程师，如果运维工程师不响应，就给经理发消息。

### 3.2. 查看趋势

查看数据库目前的数据量，以及增长速度。又例如每日活跃用户的数量增长的速度。跨时间范围的比较，或者是观察实验组与控制组之间的区别

![See the source image](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/68747470733a2f2f71696974612d696d6167652d73746f72652e73332e616d617a6f6e6177732e636f6d2f302f3130303337372f31363035396662332d316466342d383938312d656534622d3530656663326233356465352e706e67.png)

当然Prometheus提供的图形界面基本沦为了运维工程师调试PromSQL的的工具，真正给老板看的图应该是这样的。

![See the source image](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/grafana-esxi-022.png)

### 3.3. 构建监控台页面

我们上面展示了一些个指标，但是真正有用的指标，或者“黄金指标”一般是说

+ 延迟

  服务处理某个请求所需要的时间。这里区分成功请求和失败请求很重要。例如，某个由于数据库连接丢失或者其他后端问题造成的HTTP 500错误可能延迟很低。计算总体延迟时，如果将500回复的延迟也计算在内，可能会产生误导性的结果。但是，“慢”错误要比“快”错误更糟！因此，监控错误回复的延迟是很重要的。

+ 流量

  使用系统中的某个高层次的指标针对系统负载需求所进行的度量。对Web服务器来说，该指标通常是每秒HTTP请求数量，同时可能按请求类型分类（静态请求与动态请求）。针对音频流媒体系统来说，这个指标可能是网络I/O速率，或者并发会话数量。针对键值对存储系统来说，指标可能是每秒交易数量，或每秒的读取操作数量。

+ 错误

  请求失败的速率，要么是显式失败（例如HTTP 500），要么是隐式失败（例如HTTP 200 回复中包含了错误内容），或者是策略原因导致的失败（例如，如果要求回复在1s内发出，任何超过1s的请求就都是失败请求）。当协议内部的错误代码无法表达全部的失败情况时，可以利用其他信息，如内部协议，来跟踪一部分特定故障情况。监控方式也非常不一样：在负载均衡器上检测HTTP 500请求可能足够抓住所有的完全失败的请求，但是只有端到端的系统才能检测到返回错误内容这种故障类型。

+ 饱和度

  服务容量有多“满”。通常是系统中目前最为受限的某种资源的某个具体指标的度量。（在内存受限的系统中，即为内存；在I/O受限的系统中，即为I/O）。这里要注意，很多系统在达到100%利用率之前性能会严重下降，增加一个利用率目标也是很重要的。

  在复杂系统中，饱和度可以配合其他高层次的负载度量来使用：该服务是否可以正常处理两倍的流量，是否可以应对10%的额外流量，或者甚至应对当前更少的流量？对没有请求复杂度变化的简单服务来说（例如，“返回一个随机数”服务，或者是“返回一个全球唯一的单向递增整数”服务），根据负载测试中得到的一个固定数值可能就足够了。但是正如前文所述，大部分服务都需要使用某种间接指标，例如CPU利用率，或者网络带宽等来代替，因为这些指标通常有一个固定的已知的上限。延迟增加是饱和度的前导现象。99% 的请求延迟（在某一个小的时间范围内，例如一分钟）可以作为一个饱和度早期预警的指标。

+ 最后，饱和度同样也需要进行预测，例如“看起来数据库会在4个小时内填满硬盘”。

如果我们度量所有这4个黄金指标，同时在某个指标出现故障时发出警报（或者对于饱和度来说，快要发生故障时），能做到这些，服务的监控就基本差不多了。

### 3.4. 临时性的回溯分析

我们还有的时候需要查看很久之前的数据，但是这个功能其实Prometheus做的并不好，他默认会保留1个月的数据，想要查看更早的数据就需要借助于其他工具。因为Prometheus是时序数据库，也就是按照顺序存储数据，没有分区，热点数据或者压缩功能，这也是Prometheus的痛点之一。当然，我们后面会介绍怎样解决这个问题。

![See the source image](/pages/keynotes/L4_architect/2_monitoring/pics/1_monitoring/GrafanaMonomicrolithsSlide4.png)