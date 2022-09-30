---
title: telegraf
keywords: keynotes, architect, observability, telegraf
permalink: L5_architect_observability_1_metrics_2_2_telegraf.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/2_2_telegraf
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 介绍

influxdb是一个非常成熟的产品，并且可以作为商用产品出售，就表示他的技术其实是靠谱的。influxdb的收费模式是influxdb收费，而采集工具telegraf是不收费的。由于telegraf是不支持开放式插件的，他只会把一些社区贡献的代码整合到产品中，所以他的稳定性也就得到了保障。他的局限性就在于他outpu的数据基本都是

