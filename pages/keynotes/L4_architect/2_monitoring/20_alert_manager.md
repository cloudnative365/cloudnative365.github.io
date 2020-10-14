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

解压后，进入到文件夹

``` bash
tar xf alertmanager-0.21.0.linux-amd64.tar.gz
cd alertmanager-0.21.0.linux-amd64
```

我们可以看到三个文件，把它们分别放到系统的指定位置

``` bash
mv alertmanager /usr/local/sbin/
mv amtool /usr/local/sbin/
# 配置文件位置
mkdir /etc/alertmanager
mv alertmanager.yml /etc/alertmanager
# 数据文件位置
mkdir /var/lib/alertmanager
```

### 2.2. 配置alertmanager

修改启动文件/etc/systemd/system/node_exporter.service

``` bash
[Unit]
Description=alertmanager
Documentation=https://prometheus.io/docs/alerting/latest/overview/
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/alertmanager \
         --config.file=/etc/alertmanager/alertmanager.yml \
         --storage.path=/var/lib/alertmanager \
         --web.external-url=http://你的IP
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always

[Install]
WantedBy=multi-user.target
```

修改alertmanager的文件alertmanager.yml，他分为下面五个部分

+ global：主要是全局配置，比如smtp服务器，wechat的url之类
+ templates：可以指定我们发送时候使用的模板，一般是html格式的报警的时候会使用模板来发送报警，我们也可以自定模板，但是需要一些go template的知识
+ route：分发规则，根据报警的一些标志，发送给不同的receiver，使用不同的inhibbit_rule
+ receivers：发送报警的方式，支持邮件，pageduty，wechat或者直接发送到指定的webhoot
+ inhibbit_rules：对报警进行分类，如果某类的报警重复接受，会暂时抑制发送报警给用户
+ tls_config：https证书，这次咱们用不到

### 2.3. 配置prometheus

我们只需要在prometheus.yml文件中指定alertmanager

``` yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - 10.114.2.70:9093
       - 10.114.2.71:9093
```

## 3. 报警规则

alertmanager的报警规则需要借助于promsql来实现，而且，报警规则是在发送方配置的，也就是prometheus上来配置。为了日后好管理，我们会把报警根据exporter的种类进行分别的存放，在prometheus的配置文件中，使用通配符*来匹配某个目录下的所有规则，比如

``` yaml
rule_files:
    - "/etc/prometheus/rules/*.yml"
```

希望大家的知识比较扎实，这里我就简单的po出一个例子，然后在注释中做讲解

``` yaml
groups:
- name: OS
  rules:
   # 如果在15分钟内检测是内存使用量高于80%就报警
   # 使用这个死递归来让内存使用量增加
   # function a() {    $(a) ; }
   # a &
  - alert: MemoryUsage
    expr: round(((node_memory_MemTotal_bytes-node_memory_MemAvailable_bytes)/node_memory_MemTotal_bytes) * 100) > 80
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Memory of instance {{ $labels.instance }} is not enough"
      description: "Memory usage of {{ $labels.instance }} is too much for more than 15 minutes. (current value: {{ $value }}%"

  # 如果cpu的使用量连续15分钟高于80%就报警
  # 使用下面的命令来让CPU使用率增加
  # "cat /dev/urandom | md5sum"
  - alert: CPUUsage
    expr: round((1 - avg(rate(node_cpu_seconds_total{mode="idle"}[15m])) by (instance)) * 100) > 80
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "CPU usage of instance {{ $labels.instance }} is too hight"
      description: "CPU usage of {{ $labels.instance }} is too much for more than 15 minutes. (current value: {{ $value }}%"

  # 如果某个指定的硬盘设备使用量高于85%就报警
  # 使用下面的命令来让硬盘写满
  # dd if=/dev/zero of=test bs=1024M count=40
  - alert: RootfsUsage
    expr: round((node_filesystem_size_bytes{device="rootfs"}-node_filesystem_free_bytes{device="rootfs"})/node_filesystem_size_bytes{device="rootfs"} * 100) > 85
    for: 30s
    labels:
      serverity: warning
    annotations:
      summary: "Not enough space for root fs on {{ $labels.instance }}"
      description: "Not enough space for root fs on {{ $labels.instance }}. (current value: {{ $value }})%"

  # 进程是否启动，启动为1，不启动为0
  - alert: ProcessNodeExporter
    expr: up{job="node_exporter"} == 0
    for: 15s
    labels:
      serverity: critical
    annotations:
      summary: "Node Exporter on {{ $labels.instance }} is not running"
          description: "Node Exporter on {{ $labels.instance }} is not running"
```



