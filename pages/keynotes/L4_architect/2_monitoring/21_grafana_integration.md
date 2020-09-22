---
title: grafana集成
keywords: keynotes, architect, monitoring, thanos
permalink: keynotes_L4_architect_2_monitoring_21_grafana_exporter.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/21_grafana_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

### 1.1. 数据展示

数据采集完毕了，终于到了可以展示的环节了。目前来说，常见的开源展示工具有

+ grafana：grafana lab的产品，兼容性特别好，适合做集成的展示
+ kibana：主要为ES服务，分析功能非常好，监控展示只不过是顺路而已
+ zabbix/nagios/cacti/Graphite/open-falcon/Chronograf：这些工具自带UI，但是比较简陋，功能上没问题，但是作为集成工具就不行了
+ kiali/jaeger：用于服务网格的展示

我们这里先说grafana，日志专题中讲kibana，微服务专题中讲kiali和jaeger

### 1.2. Grafana

grafana是由go和typescript语言编写的，他以界面的绚丽和集成功能的强大而受到开源社区的追捧

![image-20200921100533269](/pages/keynotes/L4_architect/2_monitoring/pics/21_grafana_exporter/image-20200921100533269.png)



### 1.3. grafana lab

grafana这个软件是由grafana labs开发和维护的，由他维护的开源项目中有下面几个

![image-20200920213606338](/pages/keynotes/L4_architect/2_monitoring/pics/21_grafana_exporter/image-20200920213606338.png)

+ grafana：监控指标展示工具
+ graphite：早期监控工具，是zabbix，nagios时代的产品
+ metricstank：为graphite提供了多租户管理
+ tanka：为kubernetes提供了扩展功能
+ cortex：为prometheus提供了高可用，多租户，持久化和快速部署prometheus等功能，和前面的thanos同时隶属于CNCF的孵化项目
+ Loki：日志收集系统，我们后面日志专题会说他
+ Prometheus：非常著名，不解释了



## 2. 搭建grafana

+ 下载地址：https://grafana.com/grafana/download

+ 在centos上安装

  ``` bash
  wget https://dl.grafana.com/oss/release/grafana-7.1.5-1.x86_64.rpm
  sudo yum install grafana-7.1.5-1.x86_64.rpm
  ```

+ 数据库，如果默认启动的话，grafana会默认使用sqlite数据库，但是这个方式实现grafana的HA就比较困难了。建议大家使用postgresql或者mysql，我们这节课是个Demo，所以不讲集群架构，咱们先使用单机的postgresql

  ``` bash
  yum install postgresql-12
  ```

+ 配置权限

  ``` bash
  
  ```

+ 启动grafana

  ``` bash
  systemctl start grafana-server
  ```

+ 默认密码是admin/admin

### 3. grafana插件

### 3.1. 插件的类型

