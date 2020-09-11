---
title: thanos
keywords: keynotes, architect, monitoring, thanos
permalink: keynotes_L4_architect_2_monitoring_12_thanos_conception.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/12_thanos_conception
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 架构

我们先来看看官方架构图

![arch.jpg](/pages/keynotes/L4_architect/2_monitoring/pics/12_thanos_conception/arch.jpg)

+ 黄色的部分是prometheus，exporters和alertmanager的三大组件，是prometheus最原始的架构
+ 蓝色的部分中：Thanos Sidecar和Thanos Query就是我们上节课搭建的那两个组件
+ 深灰色的是存储部分，其中，Thanos ruler和Thanos Sidecar使用的是本地硬盘（其实也可以存储到对象存储上，但是没什么必要），而Thanos Storage Gateway和Thanos Compact可以去对象存储中获取数据。
+ Thanos Storage Gateway：我们在上一节课程中看到了，启动时候的参数`--store 127.0.0.1:19190`，这个是sidecar暴露出来的gRPG的API接口，不是Storage Gateway，如果想要使用别的存储，需要使用`thanos store `命令，并且指定配置文件的位置，使用`--objstore.config-file`，我们后面讲集成存储的时候再说。
+ Thanos Compact：他的作用是优化查询历史数据时候的查询速度。
+ Ruler：和Prometheus的ruler差不多，也是通过调用alertmanager来发送报警的

## 2. 安装和配置所有的组件

### 2.1. 准备对象存储

在我们正式安装所有的组件之前，我们还需要安装另外一个组件，这个组件thanos并不提供，他需要借助其他解决方案。[官方](https://thanos.io/tip/thanos/storage.md/)给出的方案有下面几种

![image-20200911172122008](/pages/keynotes/L4_architect/2_monitoring/pics/12_thanos_conception/image-20200911172122008.png)

我们可以看到目前稳定版本的依然是公有云三大巨头，而中国在这方面也不甘落后，阿里和腾讯都有支持，但是都是Beta版本。为了效果更好，我们使用另外一个开源方案，MinIO。这是一个开源的项目，他可以兼容S3协议，连接方式也和S3一样的。我们后面讲分布式存储的时候会说他，其他任何的软件，只要是支持持久存储到s3，我们都可以使用MinIO来替代。我们的体系中还有另外一个项目要使用到MinIO，那就是Spinnaker。

这里我们使用非常简单的方式启动一个S3

``` bash
docker
```

