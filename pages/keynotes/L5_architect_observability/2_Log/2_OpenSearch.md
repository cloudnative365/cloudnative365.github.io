---
title: fluentd基础概念
keywords: keynotes, architect, observability, fluentd, fluentd_basic
permalink: keynotes_L5_architect_observability_3_fluentd_1_fluentd_basic.html
sidebar: keynotes_L5_architect_observability
typora-copy-images-to: ./pics/1_fluentd_basic
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ 认识fluentd，搭建fluentd服务

## 1. fluentd

fluentd是一款开源的数据收集器，支持统一数据收集和使用，从而更好的利用和理解数据

``` bash
Fluentd is an open source data collector, which lets you unify the data collection and consumption for a better use and understanding of data.
```

使用fluentd之前

![img](/pages/keynotes/L5_architect_observability/3_Fluentd/pics/1_fluentd_basic/fluentd-before.png)



使用Fluentd之后

![img](/pages/keynotes/L5_architect_observability/3_Fluentd/pics/1_fluentd_basic/fluentd-architecture.png)

fluentd就像一个管道一样，把数据源统一收取，通过filter/buffer/routing等组件，对数据进行格式化之后发送到后端。我们可以从图上看出，fluentd支持非常多的input和output类型。实际上filter和buffer之类也是通过插件的形式对社区敞开大门，让更多的开发者可以参与到项目当中。

## 2. 主要功能

+ 统一日志格式为JSON

  fluentd尽量把所有的数据都转化为JSON格式，这样可以更容易的对多种数据源和输出源进行收集，过滤，缓存和输出

![img](/pages/keynotes/L5_architect_observability/3_Fluentd/pics/1_fluentd_basic/log-as-json.png)

+ 可插拔的架构

  软件自带的插件数量很少，大部分的插件都是来自于社区的贡献，这些插件随用随安装，这就使得整个程序非常精简，用到什么就安装什么，不用也可以随时卸载。

![img](/pages/keynotes/L5_architect_observability/3_Fluentd/pics/1_fluentd_basic/pluggable.png)

+ 需求资源很少

  这里的资源占用少是相对于传统的日志传输工具logstash来说的，因为logstash是Jruby开发的，开发语言是ruby，但是需要跑在JVM上，JVM这个内存老虎的厉害大家估计都有所感触。而fluentd的核心是C，而插件是ruby，既提高了效率，又对开发者非常的友好。当然，在fluentd刚出现的时代，golang+js或者golang+python这种组合并没有成熟。否则，怎么也轮不到ruby来做插件开发语言。

![img](/pages/keynotes/L5_architect_observability/3_Fluentd/pics/1_fluentd_basic/c-and-ruby.png)

+ 内置稳定性功能

  fluentd更多的是提供了一个框架，所以里面内置和非常多的功能，比如tail文件，http服务器，缓存，日志追踪，我们只需要关注我们的数据传输逻辑就好了，其他的就交给fluentd来做就可以。

![img](/pages/keynotes/L5_architect_observability/3_Fluentd/pics/1_fluentd_basic/reliable.png)