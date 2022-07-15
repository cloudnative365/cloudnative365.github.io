---
title: Thanos前端缓存
keywords: keynotes, architect, observability, thanos, frontend
permalink: L5_architect_observability_thanos_frontend.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_5_Thanos_frontend
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Thanos-frontend

这个组件的主要作用就是提高查询的速度，只要有用户查询过某个时间段的数据，frontend就会把数据缓存起来，其他人再来查的时候就会直接返回给客户端，而不去后端请求了。

文档中提到了三个缓存的地方，IN-Memory，Memcached和Redis。这三个都是使用内存来存储的，如果我们有合适的环境，比如redis集群之类，就使用redis来做，如果没有，就使用IN-MEMORY来做就好了，毕竟memcached和redis的各项指标度对比来说都是没有什么优势的。

### 1.1. 配置thanos-frontend

配置/etc/systemd/system/thanos-frontend.service

``` bash
[Unit]
Description=thanos-frontend
Documentation=https://thanos.io/
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/sbin/thanos query-frontend \
          --http-address=0.0.0.0:10913 \
          --query-frontend.downstream-url=http://127.0.0.1:10903
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always
LimitNOFILE=20480000
[Install]
WantedBy=multi-user.target
```

如果我们不使用任何其他的配置，默认是使用IN-MEMORY

### 1.2. 配置查询地址

修改/etc/nginx/conf.d/thanos-frontend.conf

``` bash
server {
        listen 10913;
        server_name  10.0.0.11;

location / {
        proxy_pass http://thanos-frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        }
}

upstream thanos-frontend {
        server 10.0.1.11:10913;
        server 10.0.1.12:10913;
}
```

