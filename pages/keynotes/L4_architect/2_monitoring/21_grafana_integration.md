---
title: grafana集成
keywords: keynotes, architect, monitoring, thanos
permalink: keynotes_L4_architect_2_monitoring_21_grafana_exporter.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/21_grafana_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

数据采集完毕了，终于到了可以展示的环节了。目前来说，常见的开源展示工具有

+ grafana：grafana lab的产品，兼容性特别好，适合做集成的展示
+ kibana：主要为ES服务，分析功能非常好，监控展示只不过是顺路而已
+ zabbix/nagios/cacti/Graphite/open-falcon/Chronograf：这些工具自带UI，但是比较简陋，功能上没问题，但是作为集成工具就不行了
+ kiali/jaeger：用于服务网格的展示

我们这里先说grafana，日志专题中讲kibana，微服务专题中讲kiali和jaeger

## 2. 搭建grafana

