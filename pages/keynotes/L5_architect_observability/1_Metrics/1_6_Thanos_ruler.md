---
title: Thanos报警
keywords: keynotes, architect, observability, thanos, ruler
permalink: L5_architect_observability_thanos_ruler.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_6_Thanos_ruler
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Alertmanager集群

AlertManager是天生支持集群模式的，他们的集群是通过两个实例之间的http通讯来进行信息传递的。alertmanager的功能我们在alertmanager的单机版中已经说过了，这里就不赘述了。我们直接开始搭建alertmanager集群。

我们在10.0.1.13和10.0.1.14上安装和配置alertmanager。

+ 初始化安装

  下载地址：https://prometheus.io/download/

  ``` bash
  wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz
  tar xf alertmanager-0.24.0.linux-amd64.tar.gz
  cd alertmanager-0.24.0.linux-amd64
  mv alertmanager amtool /usr/local/sbin
  mkdir /etc/alertmanager
  mv alertmanager.yml /etc/alertmanager/
  ```

+ 修改systemd文件，/etc/systemd/system/alertmanager.service，两台机器都这么配置，从而互相监听

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
         --cluster.listen-address=0.0.0.0:9094 \
         --cluster.peer=10.0.1.13:9094 \
         --cluster.peer=10.0.1.14:9094
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always

[Install]
WantedBy=multi-user.target
```

+ 修改alertmanager.yml

``` bash
global:
  # global parameter
  resolve_timeout: 5m

  # smtp parameter
  smtp_smarthost: smtp.qq.com:465
  smtp_from: 29371962@qq.com
  smtp_auth_username: 29371962@qq.com
  smtp_auth_password: qjrqmfxodukpbihc
  smtp_require_tls: false

route:
  receiver: default-reciever
  group_wait: 15s
  group_interval: 45s
  repeat_interval: 24h
  group_by: [alertname]
  routes:
  # below configuration is for DBAs, we have oracle, mysql, sql-server databases
  - receiver: 'mssql-dba-qa'
    continue: true
    match:
      dbserver: "mysql"
      
receivers:
- name: 'default-reciever'
  webhook_configs:
  - send_resolved: false
    url: 'http://127.0.0.1:15000'
- name: 'mssql-dba-qa'
  email_configs:
  - send_resolved: true
    to: '29371962@qq.com'
    
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'env', 'project']
```



## 2. 配置报警

### 2.1. 报警的过程

报警规则的匹配有下面几个步骤

+ ruler，不管是prometheus的ruler还是thanos的ruler，他们会使用配置文件中的匹配规则去对应的时序数据库中匹配，如果匹配到了就会把匹配到的规则发给alertmanager
+ alertmanager拿到了需要报警的请求之后，通过router中的规则，对应标签分发报警到对应的receiver。
+ alertmanager的receiver中记录的不同的报警渠道，比如微信或者SNS
+ alertmanager会根据hibite的规则对同类型的报警进行比对，如果满足发送条件，会把报警进行发送

### 2.2.  Thanos-ruler

thanos的rule和prometheus的rule功能完全一样，但是为了集群化，我们也会使用两个同样的实例。如果两个实例同时发现了报警，还是和query一样会drop掉label之后合并报警，最后会把报警同时发给alertmanager。

配置/etc/systemd/system/thanos-ruler.service

``` bash
[Unit]
Description=thanos-ruler
Documentation=https://thanos.io/
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/sbin/thanos rule \
    --http-address=0.0.0.0:10910 \
    --grpc-address=0.0.0.0:10911 \
    --data-dir=/app/thanos/ruler \
    --rule-file=/etc/thanos/rules/*/*.yml \
    --alert.query-url=http://10.0.0.11:10903 \ # 查询的URL
    --alertmanagers.config-file=/etc/thanos/thanos-ruler-alertmanager.yml \
    --query.config-file=/etc/thanos/thanos-ruler-query.yml \
    --objstore.config-file=/etc/thanos/thanos-minio.yml \
    --web.external-prefix=http://10.0.0.11:10910 \ # 对外暴露的url
    --label=cluster="aws" \
    --label=replica="A" \ # 另外的集群写B
    --alert.label-drop="replica"
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
```

/etc/thanos/thanos-ruler-alertmanager.yml文件

``` bash
alertmanagers:
- http_config:
  static_configs: ['10.0.1.13:9093','10.0.1.14:9093']
  scheme: http
  timeout: 30s
```

/etc/thanos/thanos-ruler-query.yml文件

``` bash
- http_config:
  static_configs: ["10.0.0.11:10903"]
  scheme: http
```

### 2.3. 报警规则

``` bash
groups:
- name: linux_OS
  rules:
  - alert: NodeExporterIsUp
    expr: up{job="consul-node-exporter",project="monitoring"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Node_exporter on {{ $labels.instance }} is unreachable"
      description: "Node_exporter on {{ $labels.instance }} is unreachable for 1m"

   # Alert for any instance that is unreachable for >5 minutes.
   # function a() {    $(a) ; }
   # a &
  - alert: MemoryUsage
    expr: round(((node_memory_MemTotal_bytes-node_memory_MemAvailable_bytes{project="monitoring"})/node_memory_MemTotal_bytes{project="monitoring"}) * 100) > 80
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Memory of instance {{ $labels.instance }} is not enough"
      description: "Memory usage of {{ $labels.instance }} is too much for more than 15 minutes. (current value: {{ $value }}%"

  # Alert for any instance that is unreachable for >5 minutes.
  # use "cat /dev/urandom | md5sum" to for CPU working with 100%
  - alert: CPUUsage
    expr: round((1 - avg(rate(node_cpu_seconds_total{mode="idle",project="monitoring"}[15m])) by (instance)) * 100) > 80
    #expr: round((1 - irate(node_cpu_seconds_total{mode="idle"}) by (instance)) * 100) > 80
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "CPU usage of instance {{ $labels.instance }} is too hight"
      description: "CPU usage of {{ $labels.instance }} is too much for more than 15 minutes. (current value: {{ $value }}%"

  # Alert for any instance that has a median request latency >1s.
  # dd if=/dev/zero of=test bs=1024M count=40
  - alert: FileSysystemUsage
    expr: round((node_filesystem_size_bytes{job="consul-node-exporter",device!="tmpfs",project="monitoring"}-node_filesystem_free_bytes{job="consul-node-exporter",device!="tmpfs",project="monitoring"})/node_filesystem_size_bytes{job="consul-node-exporter",device!="tmpfs",project="monitoring"} * 100) > 80
    for: 30s
    labels:
      serverity: warning
    annotations:
      summary: "Not enough space for file system on {{ $labels.instance }}"
      description: "Not enough space for file system {{ $labels.mountpoint }} fs on {{ $labels.instance }}. (current value: {{ $value }})%"
```

