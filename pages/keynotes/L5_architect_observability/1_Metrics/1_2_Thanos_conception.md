---
title: Thanos概念
keywords: keynotes, architect, observability, thanos, conception
permalink: L5_architect_observability_1_metrics_1_2_thanos_conception.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_2_Thanos_conception
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 课程说明

### 1.1. 讲解顺序

Thanos比较难上手的原因是因为Thanos并不像prometheus一样，直接使用二进制文件一启动就可以玩起来了，他有非常多的组件，每一个组件的启动方式都是通过`thanos xxx --config xxx`这种形式来构成的。

而真正的产品中，都已经把这些功能打包成了镜像，我们没办法一层一层的剥开外壳，看到实质。所以我们会从内而外的展示一下thanos的各个组件是怎么协同工作的。我们会大致经过三个阶段。

+ 构建最简单的集群
+ 优化查询
+ 持久存储

期间的一些其他组件，比如注册中心(consul)的集群，持久存储(minio)的集群，数据库(pgsql，repmgr)的集群，我们会在后面进行讲解。

### 1.2. 实验环境

在真实的生产环境中，一部分组件，比如对象存储和数据库，我们都可以直接使用公有云提供给我们的SaaS服务就好了。为了让大家理解的更加透彻，我们假设自己在私有云环境中，只有网络，服务器和存储三个部分。而这些我们都使用AWS的公有云环境进行模拟。

整个实验的最低要求是两个服务器，但是一些组件，比如服务注册中心consul和对象存储minio的集群分别需要三个和四个节点，所以建议大家使用四个节点。如果实在没有，就在一台机器上启动多个实例也可以，但是生产系统上还是要分开的。

我们这里为了模拟真实的生产环境，使用5台机器，一台作为跳转机和负载均衡，其他4台机器作为应用节点。由于我们的环境使用的是Terraform创建的，所以IP地址是随机生成的，可能每次生成的实验环境都不相同。为了让大家更直观的了解环境，我先把这几台机器的基本信息列出来。

| 节点  | IP地址    | 功能                                                         |
| ----- | --------- | ------------------------------------------------------------ |
| node0 | 10.0.0.11 | jumpserver/nginx                                             |
| node1 | 10.0.1.11 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/consul/postgresql |
| node2 | 10.0.1.12 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/consul/postgresql |
| node3 | 10.0.1.13 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/consul |
| node4 | 10.0.1.14 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/ |

### 1.3. 端口

由于实验环境机器有限，我们在一台机器上使用下面的端口给Thanos用

| Component      | Interface               | Port  |
| -------------- | ----------------------- | ----- |
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

## 2. 最初的架构

经典的组合，Prometheus+Grafana+AlertManager+Node_Exporter，我们就不过多赘述，直接上手。

### 2.1. Prometheus

创建用户用来专门跑prometheus进程，这个是为了安全起见。

``` bash
useradd -s /sbin/nologin prometheus
```

从下载并且解压prometheus

``` bash
wget https://github.com/prometheus/prometheus/releases/download/v2.35.0/prometheus-2.35.0.linux-amd64.tar.gz
tar xf prometheus-2.35.0.linux-amd64.tar.gz
cd prometheus-2.35.0.linux-amd64
```

我们可以直接启动，但是为了符合linux的操作习惯，我喜欢把这些文件整理一下

``` bash
mv prometheus promtool /usr/local/sbin/
chown -R prometheus:prometheus /usr/local/sbin/prometheus /usr/local/sbin/promtool

mkdir /var/lib/prometheus
mv console_libraries consoles /var/lib/prometheus
chown -R prometheus:prometheus /var/lib/prometheus

mkdir /etc/prometheus/
mv prometheus.yml /etc/prometheus/prometheus.yml
chown -R prometheus:prometheus /etc/prometheus/

mkdir /app/prometheus/
chown -R prometheus:prometheus /app/prometheus/
```

而prometheus自身配置如下

``` bash
/etc/prometheus/prometheus.yml 
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  external_labels:
    replica: A

# Alertmanager configuration
#alerting:
#  alertmanagers:
#  - scheme: https
#    static_configs:
#    - targets:
#        - alertmanager.monitor.ubrmb.com

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
#rule_files:
#   - "/etc/thanos/rules/node_exporter/*.yml"
#   - "/etc/thanos/rules/blackbox_exporter/*.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'prometheus'
    file_sd_configs:
    - files:
      - /etc/prometheus/targets/prometheus/*.yml
```



给他一个启动文件，配置的话都写在systemd文件里面

``` bash
cat /etc/systemd/system/prometheus.service
[Unit]
Description=prometheus
Documentation=https://prometheus.io/
After=network.target
[Service]
Type=simple
User=prometheus
ExecStartPre=/usr/local/sbin/promtool check config /etc/prometheus/prometheus.yml
ExecStart=/usr/local/sbin/prometheus \
          --config.file=/etc/prometheus/prometheus.yml \
          --web.listen-address=0.0.0.0:9090 \
          --web.enable-lifecycle \
          --web.enable-admin-api \
          --web.console.templates=/var/lib/prometheus/console \
          --web.console.libraries=/var/lib/prometheus/console_libraries \
          --web.external-url=https://prometheus.monitor.ubrmb.com \
          --storage.tsdb.path=/app/prometheus/ \
          --storage.tsdb.min-block-duration=2h \
          --storage.tsdb.max-block-duration=2h \
          --storage.tsdb.retention.time=30d \
          --log.level=info
ExecReload=/bin/curl -X POST http://127.0.0.1:9090/-/reload
TimeoutStopSec=20s
Restart=always
LimitNOFILE=20480000
[Install]
WantedBy=multi-user.target
```

启动进程`systemctl start prometheus`之后就可以通过浏览器访问了。

### 2.2. node_exporter

同样创建node_exporter用户专门用来跑node_exporter进程

下载并且解压

``` bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
tar xf node_exporter-1.3.1.linux-amd64.tar.gz
cd node_exporter-1.3.1.linux-amd64
```

同样整理一下

``` bash
mv node_exporter /usr/local/sbin/
```

给他一个启动文件

``` bash
cat /etc/systemd/system/node_exporter.service 
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/node_exporter \
          --collector.mountstats \
          --collector.systemd \
          --collector.tcpstat
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always

[Install]
WantedBy=multi-user.target
```

## 3. Thanos基础组件

Thanos只需要两个组件就可以简单形成一个集群，Thanos-query和Thanos-sidecar，sidecar用来抽象数据层，query来查询抽象出来的数据层，从而用来提供查询接口。

下载thanos

``` bash
wget https://github.com/thanos-io/thanos/releases/download/v0.26.0/thanos-0.26.0.linux-amd64.tar.gz
tar xf thanos-0.26.0.linux-amd64.tar.gz
cd thanos-0.26.0.linux-amd64
mv thanos /usr/local/sbin
```

### 3.1. Thanos-sidecar

名如其意，他就是sidecar，和prometheus的sidecar一样，我们这里先用systemd来模拟一下

``` bash
cat /etc/systemd/system/thanos-sidecar.service
[Unit]
Description=thanos-sidecar
Documentation=https://thanos.io/
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/sbin/thanos sidecar \
          --tsdb.path=/app/prometheus \
          --prometheus.url=http://localhost:9090 \
          --http-address=0.0.0.0:10901 \
          --grpc-address=0.0.0.0:10902
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
```

### 3.2. Thanos-query

query用来查询所有可能的数据接口，比如sidecar或storage-gateway，我们这里并没有把数据进行远程写，所以我们只需要查询sidecar就好了

``` bash
  cat /etc/systemd/system/thanos-query.service 
[Unit]
Description=thanos-query
Documentation=https://thanos.io/
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/sbin/thanos query \
          --http-address=0.0.0.0:10903 \
          --grpc-address=0.0.0.0:10904 \
          --store=10.0.1.11:10902 \
          --store=10.0.1.12:10902 \
          --query.timeout=10m \
          --query.max-concurrent=200 \
          --query.max-concurrent-select=40 \
          --query.replica-label=replica
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always
LimitNOFILE=20480000
[Install]
WantedBy=multi-user.targe
```

## 4. 高可用

利用nginx的代理功能来实现高可用和认证功能，为thanos-query服务增加反向代理配置

``` bash
server {
        listen 10903;
        server_name  10.0.0.11;

location / {
        proxy_pass http://thanos;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        }
}

upstream thanos {
        server 10.0.1.11:10903;
        server 10.0.1.12:10903;
}
```

