---
title: fluentd
keywords: keynotes, architect, logging, fluentd
permalink: keynotes_L4_architect_3_logging_12_fluentd.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/12_fluentd
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

![img](/pages/keynotes/L4_architect/3_logging/pics/12_fluentd/fluentd-before.png)



使用Fluentd之后

![img](/pages/keynotes/L4_architect/3_logging/pics/12_fluentd/fluentd-architecture.png)

fluentd就像一个管道一样，把数据源统一收取，通过filter/buffer/routing等组件，对数据进行格式化之后发送到后端。

## 2. 主要功能

+ 统一日志格式为JSON

  fluentd尽量把所有的数据都转化为JSON格式，这样可以更容易的对多种数据源和输出源进行收集，过滤，缓存和输出

![img](/pages/keynotes/L4_architect/3_logging/pics/12_fluentd/log-as-json.png)

+ 可插拔的架构

![img](/pages/keynotes/L4_architect/3_logging/pics/12_fluentd/pluggable.png)

+ 需求资源很少

![img](/pages/keynotes/L4_architect/3_logging/pics/12_fluentd/c-and-ruby.png)

+ 内置稳定性功能

![img](/pages/keynotes/L4_architect/3_logging/pics/12_fluentd/reliable.png)