---
title: blackbox_exporter
keywords: keynotes, architect, monitoring, blackbox_exporter
permalink: keynotes_L4_architect_2_monitoring_18_blackbox_exporter.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/18_blackbox_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

blackbox_exporter是用来探测HTTP, HTTPS, DNS, TCP和ICMP的。用来探测这些服务或者端口是否正常运行。我们也可以认为他是prometheus功能的扩展。

## 2. 安装

下载地址：https://github.com/prometheus/blackbox_exporter/releases。当然，我们也可以选择使用容器化安装。

``` bash
docker run --rm -d -p 9115:9115 --name blackbox_exporter -v `pwd`:/config prom/blackbox-exporter:master --config.file=/config/blackbox.yml
```

但是我们为了便于理解，使用二进制方式来安装。

### 2.1. 配置服务

下载并且解压二进制包

``` bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
tar xf blackbox_exporter-0.18.0.linux-amd64.tar.gz
```



