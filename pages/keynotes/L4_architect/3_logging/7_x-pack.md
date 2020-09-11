---
title: x-pack
keywords: keynotes, architect, logging, 7_x-pack
permalink: keynotes_L4_architect_3_logging_7_x-pack.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/7_x-pack
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

在早起的ES版本中，我是说6.2版本之前，ES的安全功能是需要额外安装插件的，也就是我们熟知的x-pack，需要通过`elasticsearch-plugin install`的方式来安装。在6.2版本之后，官方把这个功能整合进了ES，我们可以通过配置文件直接开启，而不需要额外安装plugin了。在官方文档中，也在标题部分抹去了x-pack的痕迹，而全部都叫做[安全](https://www.elastic.co/guide/en/security/current/index.html)功能（SIEM）。 而安全功能实际上整合了SIEM线程探测（SIEM threat detection features），端点保护（endpoint prevention ）和响应能力（response capabilities）。

### 1.1. 官方版本的区别

elastic stack免费版提供的安全功能是非常有限的，在官方文档中的描述叫核心安全功能，而收费的叫高级安全功能，请参考[官方网站](https://www.elastic.co/cn/subscriptions)

![image-20200904094700864](/pages/keynotes/L4_architect/3_logging/pics/7_x-pack/image-20200904094700864.png)

其实我觉得他已经把一些亮点都总结的很好了

+ 基础级：免费
+ 黄金级：亮点在于kibana的监控和报警功能
+ 白金级：亮点在于安全功能，也就是x-pack的全套功能
+ 企业级：所有功能

### 1.2. 安全组件的工作流

![Elastic Security workflow](/pages/keynotes/L4_architect/3_logging/pics/7_x-pack/workflow.png)

## 1. 开启安全认证

