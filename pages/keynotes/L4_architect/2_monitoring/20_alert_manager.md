---
title: AlertManager
keywords: keynotes, architect, monitoring, AlertManager
permalink: keynotes_L4_architect_2_monitoring_20_alert_manager.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/20_alert_manager
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

alertmanager是prometheus的报警工具，但是他独立的进程，作为一个独立的报警软件存在，不仅仅能接受来自prometheus的报警，还能够接受多种其他的监控工具发来的报警。由于他是声明式的API，即使是postman或者curl也可以发送报警。或者搭配使用fluentd的alertmanager output插件，对于任何类型的input都可以实现报警功能。

## 2.安装

### 2.1. 安装alertmanager

``` bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
```



### 2.2. 配置prometheus

我们只需要在prometheus.yml文件中指定alertmanager

### 2.3. 配置alertmanager



## 3. 报警规则

alertmanager的报警规则需要借助于promsql来实现，如果