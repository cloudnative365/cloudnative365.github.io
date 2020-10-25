---
title: thanos高可用
keywords: keynotes, L4_architect, monitoring, thanos_HA
permalink: keynotes_L4_architect_2_monitoring_26_thanos_HA.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/26_thanos_HA
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

thanos是无状态服务，所以并不存在传统集群中的数据同步，防脑裂，集群选举等问题。因为他的数据源是prometheus，数据最终会存储在对象存储（s3，minIO等）中。他就好像是一个后台应用，前台数据就是prometheus暴露出来的metrics，后台数据库就是对象存储。所以说，其实最好的方式是把thanos放在kubernetes中进行部署和管理是最佳实践。

## 2. 架构

![arch](/pages/keynotes/L4_architect/2_monitoring/pics/26_thanos_HA/thanos.jpg)

这张图我们在前面已经介绍过了，这次我们需要从高可用架构上注意一下红色框的位置是需要高可用的

+ grafana：前端是需要高可用的，由于用户有可能会创建很多dashboard，而存储的位置是pgsql，所以我们就需要在pgsql做一个HA就行了。需要注意的是，grafana的数据是存在数据库中的，但是插件是不存在数据库中的，我们需要在每个节点上进行安装（grafana-cli plugin install）
+ alertmanager：自带高可用，主要是防止重复报警或者漏发报警，我们后面讲。
+ prometheus：我们使用replica来表示他的高可用，而每个replica都需要一个thanos sidecar来辅助
+ minIO：上次介绍过了，统一入口只有一个，需要借助nginx来实现，当然，任何7层的负载均衡都可以实现。如果我们的环境中有存储和虚拟化（hpyervisor），还可以有更多玩法。
+ pgsql：上次也介绍过了，我们的pgsql不仅仅是为grafana的用户数据提供持久存储，还可以为zabbix，harbor，gitlab等大部分云原生软件提供持久存储，我们后面讲CI/CD时候还会用到。
+ consul：所有被监控的endpoint都会作为consul的service存储在consul中。

在实际工作环境中，我们建议使用kubernetes来构建这些无状态的应用，包括exporters，都放在k8s集群中管理。也就是说，grafana，alertmanager，prometheus，thanos都放在k8s集群中。minIO，pgsql，consul都作为独立进程来管理。而consul，minIO由于特性决定，我们可以使用本地磁盘作为存储介质。也就是说把consul，minIO应用部署在k8s集群中，而使用hostpath方式把本地磁盘给consul和minio作为存储介质，然后绑定应用到某个节点，来实现高可用。

## 3. 安装

这次我们使用systemd的方式来安装，由于是在一台机器上安装所有组件，所以，提前规划好我们的端口

| Component      | Interface               | Port  |
| -------------- | ----------------------- | ----- |
| Prometheus     | HTTP                    | 9090  |
| Sidecar        | gRPC                    | 10901 |
| Sidecar        | HTTP                    | 10902 |
| Query          | gRPC                    | 10903 |
| Query          | HTTP                    | 10904 |
| Store          | gRPC                    | 10905 |
| Store          | HTTP                    | 10906 |
| Receive        | gRPC (store API)        | 10907 |
| Receive        | HTTP (remote write API) | 10908 |
| Receive        | HTTP                    | 10909 |
| Rule           | gRPC                    | 10910 |
| Rule           | HTTP                    | 10911 |
| Compact        | HTTP                    | 10912 |
| Query Frontend | HTTP                    | 10913 |

### 3.1. prometheus

我们分别在两台机器172.16.0.4/5上安装prometheus

+ 下载

  ``` bash
  wget https://github.com/prometheus/prometheus/releases/download/v2.21.0/prometheus-2.21.0.linux-amd64.tar.gz
  ```

+ 解压

  ``` bash
  tar xf prometheus-2.21.0.linux-amd64.tar.gz
  ```

+ 准备目录

  ``` bash
  # 把prometheus文件放到$PATH下
  mv prometheus /usr/local/sbin
  # 两个web的库文件放到/var/lib/prometheus下面
  mv console /var/lib/prometheus
  mv console_libraries /var/lib/prometheus
  # 配置文件主目录
  mkdir /etc/prometheus/
  # 其他配置文件目录
  # 用来存放采集目标的
  mkdir /etc/prometheus/target/consul
  mkdir /etc/prometheus/target/mysql
  # 用来存放规则的
  mkdir /etc/prometheus/rules/consul
  mkdir /etc/prometheus/target/mysql
  mkdir /etc/prometheus/target/node_exporter
  # 用来存放数据的
  mkdir /app/prometheus/
  ```
  
+ systemd文件

  ``` bash
  [Unit]
  Description=prometheus
  Documentation=https://prometheus.io/
  After=network.target
  [Service]
  Type=simple
  ExecStart=/usr/local/sbin/prometheus \
            --config.file=/etc/prometheus/prometheus.yml \
            --web.listen-address=:9090 \
            --web.enable-lifecycle \
            --web.enable-admin-api \
            --web.console.templates=/var/lib/prometheus/console \
            --web.console.libraries=/var/lib/prometheus/console_libraries \
            --storage.tsdb.path=/app/prometheus/ \
            --storage.tsdb.min-block-duration=2h \
            --storage.tsdb.max-block-duration=2h \
            --log.level=info
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  [Install]
  WantedBy=multi-user.target
  ```

+ 配置文件`prometheus.yml`

  ``` bash
  global:
    scrape_interval:     15s
    evaluation_interval: 15s
    # 这个是thanos在做聚合的时候需要用的，我们这里以replica作为聚合标准，在另外一个prometheus节点上使用replica：B
    external_labels:
      region: dc-1
      replica: A
  
  alerting:
    alertmanagers:
    - static_configs:
      - targets:
         - 10.114.2.70:9093
         - 10.114.2.71:9093
  
  # 配置文件必须要指定到文件级别，指定目录不可以，可以使用通配符进行匹配
  rule_files:
     - "/etc/prometheus/rules/node_exporter/*.yml"
  
  scrape_configs:
  
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
      
  # 配置consul源，让prometheus动态获取采集目标
    - job_name: 'consul-node-exporter'
      consul_sd_configs:
        - server: '10.114.2.68:8500'
          services: []  
      relabel_configs:
      - source_labels: [__meta_consul_tags]
          regex: .*node-exporter.*
          action: keep
        - regex: __meta_consul_service_metadata_(.+)
          action: labelmap
  ```
  
+ 启动服务

  ``` bash
  systemctl start prometheus
  systemctl enable prometheus
  ```

### 3.2. thanos sidecar

sidecar需要在每个prometheus上都配置一个，如果使用k8s就要使用sidecar方式。我们这次用二进制方式来做。

+ 下载地址：https://github.com/thanos-io/thanos/releases

  ``` bash
  wget https://github.com/thanos-io/thanos/releases/download/v0.15.0/thanos-0.15.0.linux-amd64.tar.gz
  ```

+ 下载完成后解压出一个文件，放到可执行目录中去

  ``` bash
  tar xf thanos-0.15.0.linux-amd64.tar.gz -C /usr/local/sbin
  ```

+ 准备目录

  ``` bash
  # thanos配置文件的目录
  mkdir /etc/thanos/
  # 创建配置文件
  touch /etc/thanos/thanos-sidecar.yml
  ```

+ 配置systemd文件/etc/systemd/system/thanos-sidecar.service

  ``` bash
  [Unit]
  Description=thanos-sidecar
  Documentation=https://thanos.io/
  After=network.target
  [Service]
  Type=simple
  ExecStart=/usr/local/sbin/thanos sidecar \
            --tsdb.path=/app/prometheus \
            --objstore.config-file=/etc/thanos/thanos-sidecar.yml \
            --prometheus.url=http://localhost:9090 \
            --http-address=0.0.0.0:10901 \
            --grpc-address=0.0.0.0:10902
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  [Install]
  WantedBy=multi-user.target
  ```

+ 对象存储我们暂时使用本地文件系统，当然，我们也可以使用对象存储，那就是minIO

  ``` bash
  type: FILESYSTEM
  config:
    directory: "/app/thanos/sidecar/"
  ```

+ 启动服务

  ``` bash
  systemctl start thanos-sidecar
  systemctl enable thanos-sidecar
  ```

### 3.3. thanos-storage

+ 准备目录

  ``` bash
  mkdir /app/thanos/data
  touch /etc/thanos/thanos-store.yml
  ```

+ systemd文件/etc/systemd/system/thanos-store.service

  ``` bash
  [Unit]
  Description=thanos-store
  Documentation=https://thanos.io/
  After=network.target
  [Service]
  Type=simple
  ExecStart=/usr/local/sbin/thanos store \
            --data-dir=/app/thanos/data \
            --objstore.config-file=/etc/thanos/thanos-store.yml \
            --http-address=0.0.0.0:10905 \
            --grpc-address=0.0.0.0:10906
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  [Install]
  WantedBy=multi-user.target
  ```

+ 对象存储配置文件/etc/thanos/thanos-store.yml

  ``` bash
  type: FILESYSTEM
  config:
    directory: "/app/thanos/data/"
  ```

+ 启动服务

  ``` bash
  systemctl start thanos-store
  systemctl enable thanos-store
  ```

### 3.4. thanos-query

+ 配置systemd文件/etc/systemd/system/thanos-query.service

  ``` bash
  [Unit]
  Description=thanos-query
  Documentation=https://thanos.io/
  After=network.target
  [Service]
  Type=simple
  ExecStart=/usr/local/sbin/thanos query \
            --http-address=0.0.0.0:10903 \
            --grpc-address=0.0.0.0:10904 \
            --store=10.114.2.66:10902 \
            --store=10.114.2.69:10902 \
            --query.replica-label=replica
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  [Install]
  WantedBy=multi-user.target
  ```

+ 启动服务

  ``` bash
  systemctl start thanos-store
  systemctl enable thanos-store
  ```

### 3.5. thanos-ruler

+ 准备目录

  ``` bash
  mkdir /app/thanos/ruler
  mkdir -p /etc/thanos/rules/node_exporter
  touch /etc/thanos/thanos-ruler.yml
  ```

+ 配置systemd文件/etc/systemd/system/thanos-ruler.service

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
      --eval-interval=30s \
      --rule-file=/etc/thanos/rules/*/*.yml \
      --alert.query-url=http://0.0.0.0:9090 \
      --alertmanagers.url=http://10.114.2.70:9093,http://10.114.2.71:9093 \
      --query=10.114.2.66:10903 \
      --query=10.114.2.69:10903 \
      --objstore.config-file=/etc/thanos/thanos-ruler.yml \
      --label=region="dc-1" \
      --label=replica="A"
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  [Install]
  WantedBy=multi-user.target
  ```

+ 创建rule，/etc/thanos/rules/node_exporter/os.yml，这个是node_exporter用来采集系统数据的，报警有三个，内存，cpu和磁盘使用率

  ``` bash
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

+ 启动服务

  ``` bash
  systemctl start thanos-ruler
  systemctl enable thanos-ruler
  ```

  