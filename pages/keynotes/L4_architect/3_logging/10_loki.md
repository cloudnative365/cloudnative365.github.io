---
title: loki初体验
keywords: keynotes, architect, logging, loki
permalink: keynotes_L4_architect_3_logging_9_loki.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/9_loki
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

+ 认识loki，搭建loki服务
+ 使用promtail进行日志收集
+ 使用grafana进行日志展示

## 1. Loki

loki是grafana公司的另外一款主打产品，目前是grafana公司继prometheus，grafana，cortex之后的第四款比较知名的产品。这是一款支持垂直扩展，支持高可用，多租户日志聚合的系统。他和我们前面讲的ES就在于降低了成本，操作简单。Loki并没有对日志的内容进行索引，而是对于每个日志流进行打标签的工作。这样，对于小规模的日志存取都非常方便，也就更加适合于我们对于日志监控的要求，让产品定位在了日志监控。而ES在日志监控方面的功能虽然也非常强大，但是操作复杂度和昂贵的价格会让我们望而生畏，因此，轻量且开源的Loki让我们有了另外的选择。

但是Loki的缺点也是非常明显的，这也是大部分开源的云原生产品的共性：没有认证功能。如果想做认证，就需要通过其他的方式来实现，比如nginx，或者k8s ingress的认证，或者其他三方的工具来实现。

## 2. 架构图

一般来说，我们会通过logagent（比如promtail或者fluentd）挖掘日志，然后发送给Loki服务器，最后通过grafana进行展示。而主要的功能都集中在了loki服务器上

![modes_diagram](https://grafana.com/docs/loki/latest/architecture/modes_of_operation.png)

### 2.1. 组件

+ Distributer 分发器

