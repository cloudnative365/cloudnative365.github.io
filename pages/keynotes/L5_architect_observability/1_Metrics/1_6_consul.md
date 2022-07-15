---
title: Consul集群
keywords: keynotes, architect, observability, thanos, consul
permalink: L5_architect_observability_thanos_consul.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_6_Thanos_consul
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Consul

consul是hashicorp公司的产品，是比较知名的服务注册中心。早期的kubernetes，service mesh的解决方案之一，consul template就是基于consul的。目前他和etcd，zookeeper一起作为三种最流行的服务注册中心的解决方案。

Prometheus可以通过拉取consul中的服务信息，从而把要监控的endpoint信息拿到prometheus当中。省去了我们手动修改prometheus的静态配置文件的麻烦，实现了服务动态注册服务，而不需要重新启动prometheus来更改被监控端。

Prometheus天生支持service discovery的模式，我们要用到的是consul_sd模块。通过consul来探测exporter是不是正常工作，而不是由prometheus来直接探测。consul集群架构如下

![consul_ha](/pages/keynotes/L5_architect_observability/1_Metrics/pics/1_6_Thanos_consul/consul_ha.png)

由于consul天生支持内网和外网探测两种方式，还自带集群模式，可以说是DC中prometheus的最好搭档。当然，还有一种模式也非常不错，就是通过DNS。

## 2. 搭建Consul集群

下载consul二进制包

``` bash
wget https://releases.hashicorp.com/consul/1.12.2/consul_1.12.2_linux_amd64.zip
unzip consul_1.12.2_linux_amd64.zip
mv consul /usr/local/sbin/
```

初始化目录

``` bash
mkdir -p /app/consul/{data,log}
mkdir /etc/consul
```

编辑/etc/systemd/system/consul-server.service

``` bash
[Unit]
Description=Consul service
Documentation=https://www.consul.io/docs/
After=network.target
    
[Service]
ExecStart=/usr/local/sbin/consul agent -config-dir /etc/consul
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
LimitNOFILE=20480000
    
[Install]
WantedBy=multi-user.target
```

编辑/etc/consul/server.json

``` bash
{
  "data_dir": "/app/consul/data",
  "log_file": "/app/consul/log/consul.log",
  "log_level": "INFO",
  "log_rotate_duration": "24h",
  "node_name": "node1",
  "server": true,
  "bootstrap_expect": 3,
  "client_addr": "0.0.0.0",
  "advertise_addr": "10.0.1.11",
  "bind_addr": "0.0.0.0",
  "start_join": ["10.0.1.11"],
  "datacenter": "internal",
  "ui": true,
  "limits":
  {
    "rpc_max_conns_per_client": 1000,
    "http_max_conns_per_client": 1000
  },
  "telemetry":
  {
    "prometheus_retention_time": "60s"
  }
}
```

在实际的生产上，我们还会考虑到安全的问题，比如通讯加密，认证token等等，有兴趣的同学可以参考这个https://learn.hashicorp.com/collections/consul/security，我们后面有时间会详细说。

## 3. 注册endpoint的信息

consul是使用api来进行管理的，虽然consul命令本身可以实现管理，但是本质依然是调用API，这个和kubernetes有一点像。我们就使用直接调用api的方式来把需要注册的信息传入consul中。

## 4. 配置Prometheus读取Consul的信息

Consul和Prometheus的结合方式是通过consul_sd来实现的。

``` bash
global:
  scrape_interval:     1m
  evaluation_interval: 1m
  external_labels:
    monitor: APSI
    replica: A
    
scrape_configs:
  - job_name: 'prometheus'
    file_sd_configs:
    - files:
      - /etc/prometheus/targets/prometheus/*.yml
      
  - job_name: 'consul-node-exporter'
    consul_sd_configs:
      - server: '10.0.1.11:8500'
        services: []
      - server: '10.0.1.12:8500'
        services: []
      - server: '10.0.1.13:8500'
        services: []
    relabel_configs:
      - source_labels: [__meta_consul_tags]
        regex: .*node-exporter.*
        action: keep
      - regex: __meta_consul_service_metadata_(.+)
        action: labelmap
```
