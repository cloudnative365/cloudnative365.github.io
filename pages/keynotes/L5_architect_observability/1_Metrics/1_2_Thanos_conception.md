---
title: Thano概念
keywords: keynotes, architect, observability, thanos, conception
permalink: L5_architect_observability_thanos_conception.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_1_Thanos_conception
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. 课程说明

### 1.1. 讲解顺序

Thanos比较难上手的原因是因为Thanos并不像prometheus一样，直接使用二进制文件一启动就可以玩起来了，他有非常多的组件，每一个组件的启动方式都是通过`thanos xxx --config xxx`这种形式来构成的。

而真正的产品中，都已经把这些功能打包成了镜像，我们没办法一层一层的剥开外壳，看到实质。所以我们会从内而外的展示一下thanos的各个组件是怎么协同工作的。我们会大致经过三个阶段。

+ 构建最简单的集群
+ 优化查询
+ 持久存储

期间的一些其他组件，比如注册中心的集群，持久存储的集群，数据库的集群，我们会在后面进行讲解。

### 1.2. 实验环境

在真实的生产环境中，一部分组件，比如对象存储和数据库，我们都可以直接使用公有云提供给我们的SaaS服务就好了。为了让大家理解的更加透彻，我们假设自己在私有云环境中，只有网络，服务器和存储三个部分。而这些我们都使用AWS的公有云环境进行模拟。

整个实验的最低要求是两个服务器，但是一些组件，比如服务注册中心consul和对象存储minio的集群分别需要三个和四个节点，所以建议大家使用四个节点。如果实在没有，就在一台机器上启动多个实例也可以，但是生产系统上还是要分开的。

我们这里为了模拟真实的生产环境，使用5台机器，一台作为跳转机和负载均衡，其他4台机器作为应用节点。由于我们的环境使用的是Terraform创建的，所以IP地址是随机生成的，可能每次生成的实验环境都不相同。为了让大家更直观的了解环境，我先把这几台机器的基本信息列出来。

| 节点  | 功能                                                         |
| ----- | ------------------------------------------------------------ |
| node0 | jumpserver/nginx                                             |
| node1 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/consul/postgresql |
| node2 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/consul/postgresql |
| node3 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/consul |
| node4 | prometheus/grafana/alertmanager/thanos-sidecar/thanos-query/thanos-storage-gateway/thanos-frontend/thanos-ruler/thanos-compact/minio/ |

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
mv prometheus.yml /etc/prometheus/yml
chown -R prometheus:prometheus /etc/prometheus/
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
ExecStart=/usr/local/sbin/prometheus \
          --config.file=/etc/prometheus/prometheus.yml \
          --web.console.templates=/var/lib/prometheus/console \
          --web.console.libraries=/var/lib/prometheus/console_libraries \
          --storage.tsdb.path=/app/prometheus/
TimeoutStopSec=20s
Restart=always
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
[root@ubmpaplp00215 ~]# cat /etc/systemd/system/node_exporter.service 
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/node_exporter 
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always

[Install]
WantedBy=multi-user.target
```

