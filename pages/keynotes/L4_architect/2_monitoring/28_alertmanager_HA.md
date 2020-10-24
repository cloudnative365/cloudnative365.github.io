---
title: alertmanager高可用
keywords: keynotes, L4_architect, monitoring, alertmanager_HA
permalink: keynotes_L4_architect_2_monitoring_28_alertmanager_HA.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/28_alertmanager_HA
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. 介绍

alertmanager的高可用原理非常简单，就是通过各个节点暴露出来的端口来交换自己机器上收到的报警请求，看看是不是有重复报警。alertmanager服务自身默认监听在9093端口之上，在建立alertmanager集群的时候，要把集群中每个alertmanager的地址都写上，而集群是通过9094端口来通信的。最后，由于报警规则是在报警发送方，比如prometheus上定义的，所以alertmanager也不会频繁的重启来应用新的配置，也就没有-web-lifecycle选项。

## 2. 报警方式

我们的alertmanager不仅能够接受来自于prometheus的报警，还可以接受多种来源的报警。并且报警的方式也非常多样。他可以接受来自于prometheus，thanos，fluentd，telegraf等多种方式的报警请求，由于他的API是开放式API，我们还可以使用简单的http调用来实现自定义的报警。而报警的途径，可以通过邮件，微信，pageduty等多种方式报警。

## 3. 安装集群

+ 下载地址：https://prometheus.io/download/

  ``` bash
  wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
  ```

+ 解压alertmanager文件到/usr/local/sbin下

  ``` bash
  tar xf alertmanager-0.21.0.linux-amd64.tar.gz
  mv alertmanager-0.21.0.linux-amd64/alertmanager /usr/local/sbin/
  ```

+ 配置第一台机器的systemd文件，/etc/systemd/system/alertmanager.service

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
           --web.external-url=http://10.114.2.70:9093 \
           --cluster.listen-address=10.114.2.70:9094 \
           --cluster.peer=10.114.2.70:9094 \
           --cluster.peer=10.114.2.71:9094
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  ```
  
+ 配置第二台机器的systemd文件

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
           --web.external-url=http://10.114.2.71:9093
           --cluster.listen-address=10.114.2.71:9094 \
           --cluster.peer=10.114.2.70:9094 \
           --cluster.peer=10.114.2.71:9094
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  ```

+ 两台机器都要创建/var/lib/alertmanager目录，然后修改配置文件/etc/alertmanager/alertmanager.yml

  ``` yml
  global:
    # global parameter
    resolve_timeout: 5m
  
    # smtp parameter
    smtp_smarthost: xxxx.aliyun.com:465
    smtp_from: monitor@xxx.com
    smtp_auth_username: monitor@xxx.com
    smtp_auth_password: your_password
    smtp_require_tls: false
  
  route:
    group_by: ['alertname']
    group_wait: 10s
    group_interval: 10s
    repeat_interval: 1h
    receiver: 'test.email'
  
  receivers:
  - name: 'test.email'
    email_configs:
    - send_resolved: true
      to: '29371962@qq.com'
  
  inhibit_rules:
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'dev', 'instance']
  ```

+ 这个时候就可以启动了

  ``` bash
  systemctl start alertmanager
  ```

+ 同时修改prometheus的配置，指向这两台alertmanager

  ``` yml
  alerting:
    alertmanagers:
    - static_configs:
      - targets:
         - 10.114.2.70:9093
         - 10.114.2.71:9093
  
  # 我们顺路把rule文件路径也写上
  rule_files:
     - "/etc/prometheus/rules/node_exporter/*.yml"
  ```

## 4. 配置

### 4.1. 配置报警分发的规则

分发的规则是通过group by来区分的，就好像SQL中的group by是一样的，但是这里的group by是通过标签来区分的，同一类标签的报警只能发给同一类人，例子中是`alertname`。

``` bash
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'test.email'
```

最后写明的receiver内容需要和下面的内容匹配

### 4.2. 配置报警渠道reciever

reciever对应上面route的规则中的receiver，定义了一类的reciever，这一类reciever可以使用邮件，微信或者其他方式接收报警，我们的例子中只用了一个邮件

``` yml
receivers:
- name: 'test.email'
  email_configs:
  - send_resolved: true
    to: '29371962@qq.com'
```

send_resolved表示在报警消失时候是否发送已经恢复了的信息

### 4.3. 配置报警的抑制规则inhibit_rule

这个是说规定之中的同类型的报警不重复发

``` yml
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

上面是说如果标签alertname，dev和instance都相同的话，发送级别为critical的报警时，不会发级别为warning的报警

### 4.4. 内置变量

## 5. 规则

这里的规则实际上通过PromSQL的查询后，看看是否符合标准，符合报警规则就报警，不符合就不报警，所以前面prometheus查询的知识一定要牢固，我们这里给出了三个例子，分别是监控CPU，内存，硬盘的

``` yaml
groups:
- name: OS
  rules:

   # Alert for any instance that is unreachable for >5 minutes.
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

  # Alert for any instance that is unreachable for >5 minutes.
  # use "cat /dev/urandom | md5sum" to for CPU working with 100%
  - alert: CPUUsage
    expr: round((1 - avg(rate(node_cpu_seconds_total{mode="idle"}[15m])) by (instance)) * 100) > 80
    #expr: round((1 - irate(node_cpu_seconds_total{mode="idle"}) by (instance)) * 100) > 80
    for: 5s
    labels:
      severity: warning
    annotations:
      summary: "CPU usage of instance {{ $labels.instance }} is too hight"
      description: "CPU usage of {{ $labels.instance }} is too much for more than 15 minutes. (current value: {{ $value }}%"

  # Alert for any instance that has a median request latency >1s.
  # dd if=/dev/zero of=test bs=1024M count=40
  - alert: RootfsUsage
    expr: round((node_filesystem_size_bytes{device="rootfs"}-node_filesystem_free_bytes{device="rootfs"})/node_filesystem_size_bytes{device="rootfs"} * 100) > 85
    for: 30s
    labels:
      serverity: warning
    annotations:
      summary: "Not enough space for root fs on {{ $labels.instance }}"
      description: "Not enough space for root fs on {{ $labels.instance }}. (current value: {{ $value }})%"
```

## 6. 工具