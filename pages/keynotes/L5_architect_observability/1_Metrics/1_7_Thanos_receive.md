---
title: Thanos接受请求
keywords: keynotes, architect, observability, thanos, receive
permalink: L5_architect_observability_thanos_receive.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_7_Thanos_receive
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Thanos receive

到了这里，我们要开始把thanos从sidecar模式转向receive模式了。我们再复习一下receive模式。

![Deployment_with_Receive](/pages/keynotes/L5_architect_observability/1_Metrics/pics/1_7_Thanos_receive/Deployment_with_Receive.png)

这个模式中，thanos不再是通过sidecar的模式来主动抓取数据了，而是要求prometheus通过remote write的方式向thanos receive传递数据，传递过来的数据直接写入对象存储。

## 2. thanos receive

编辑/etc/systemd/system/thanos-receive.service

``` bash
[Unit]
Description=thanos-receive
Documentation=https://thanos.io/
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/sbin/thanos receive \
          --tsdb.path=/app/thanos/receive \
          --grpc-address=0.0.0.0:10908 \
          --http-address=0.0.0.0:10909 \
          --receive.replication-factor=2 \
          --label=receive_replica="A" \
          --receive.local-endpoint=10.0.1.11:10908 \
          --receive.hashrings-file=/app/thanos/receive/hashring.json \
          --remote-write.address=0.0.0.0:10912 \
          --objstore.config-file=/etc/thanos/thanos-minio.yml
ExecReload=/bin/kill -HUP
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
```

编辑/app/thanos/receive/hashring.json

``` bash
[
    {
        "endpoints": [
            "10.0.1.11:10908",
            "10.0.1.12:10908"
        ]
    }
]
```

记得要在两台服务器上都配置上

## 3. remote write

remote write是prometheus持久化数据的方案之一，prometheus通过定期发送监控数据到远程的介质中，来实现数据持久化。远程的介质分为可读可写和只写这两种方式，只写是说prometheus端是没办法直接读取的，但是不代表远程介质本身没办法读，比如influxdb。他们有着一个共同点，那就是都支持远程API写入的方式。当然，prometheus本身也可以作为远程存储的介质之一，从一个prometheus写入另外一个prometheus，但是我们有另外的好方法，这就是federation，我们在前面介绍过，这里就不说了。

我们配置prometheus的systemd文件，增加一个remote write的配置

``` bash
[Unit]
Description=prometheus
Documentation=https://prometheus.io/
After=network.target
[Service]
Type=simple
ExecStartPre=/usr/local/sbin/promtool check config /etc/prometheus/prometheus.yml
ExecStart=/usr/local/sbin/prometheus \
          --config.file=/etc/prometheus/prometheus.yml \
          --web.listen-address=127.0.0.1:9090 \
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

