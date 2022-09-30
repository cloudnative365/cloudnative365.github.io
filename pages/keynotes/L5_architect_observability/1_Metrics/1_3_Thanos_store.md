---
title: Thanos存储
keywords: keynotes, architect, observability, thanos, store
permalink: L5_architect_observability_1_metrics_1_3_thanos_store.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_3_Thanos_store
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 对象存储

一般来说，我们将存储分为文件存储，块存储和对象存储。

+ 文件存储：一般都是POSIX协议的，比如我们操作系统上的XFS，EXT4

+ 块存储：一般都是由虚拟化层实现的，有可能是QEMU的driver或者Kernel自带的模块，比如AWS的EBS

+ 对象存储：对象存储通常向外提供API接口，系统通过网络向对象存储的接口传输数据。公有云的代表就是AWS的S3，私有云的代表就是MinIO。

需要说明的是Ceph，Ceph既能提供RBD块存储，也能提供CephFS的文件存储，还能提供对象存储RADOS，如果大家购买了红帽的订阅，不妨多花点钱使用Ceph，毕竟花钱的总是比不花钱的好一点。

另外，我们后面要说的k8s的对象存储也会涉及到Ceph，因为我们的Rook就是使用的Ceph。

## 2. 选择合适的存储类型

[官方文档](https://thanos.io/tip/thanos/storage.md/#supported-clients)中提供了一些选择，这些选择目前有Beta的，也有Stable的。如果在公有云环境，我建议大家使用公有云提供的对象存储，真的是又便宜又好用。我们这里用的MinIO是目前比较流行的私有云S3，他完全遵循S3协议，管理和使用方式和S3也基本相同。

### 2.1. 安装MinIO

下载MinIO服务器

``` bash
wget http://dl.minio.org.cn/server/minio/release/linux-amd64/minio #中国区
wget https://dl.min.io/server/minio/release/linux-amd64/minio # 国际区
mv minio /usr/local/sbin && chmod +x /usr/local/sbin/minio
```

下载MinIO客户端

``` bash
wget http://dl.minio.org.cn/client/mc/release/linux-amd64/mc #中国区
wget https://dl.min.io/client/mc/release/linux-amd64/mc #国际区
mv mc /usr/local/sbin && chmod +x /usr/local/sbin/mc
```

### 2.2. 配置

初始化需要的目录

``` bash
mkdir -p /app/minio/data && mkdir /etc/minio && mkdir /app/minio/run
```

四台机器配置一样 /etc/systemd/system/minio.service

``` bash
[Unit]
Description=Minio service
Documentation=https://docs.minio.io/

[Service]
WorkingDirectory=/app/minio/run/
EnvironmentFile=/etc/minio/minio.pw

ExecStart=/usr/local/sbin/minio server \
--config-dir /etc/minio \
--address :9000 \
--console-address :9001 \
http://10.0.1.11:9000/app/minio/data \
http://10.0.1.12:9000/app/minio/data \
http://10.0.1.13:9000/app/minio/data \
http://10.0.1.14:9000/app/minio/data

Restart=on-failure
RestartSec=5
LimitNOFILE=20480000
[Install]
WantedBy=multi-user.target
```

而minio.pw里面写上初始的密码

``` bash
MINIO_ACCESS_KEY=Minio
MINIO_SECRET_KEY=Passw0rd
```

### 2.3. 创建账户

在下载页面还有一个下载链接，是[下载](https://dl.min.io/client/mc/release/linux-amd64/mc)mc的，这个就是minio的客户端，我们可以不用把客户端装在服务器上，可以装在笔记本上。下载完成后是一个文件，给一个执行权限

``` bash
chmod +x mc
```

mc和aws-cli类似，权限控制也是由profile控制的，只不过在minio中，他叫config

``` bash
mc config host list
```

我们先添加一个host

``` bash
mc config host add monitor http://localhost:9001 Minio
mc: Configuration written to `/root/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/root/.mc/share`.
mc: Initialized share uploads `/root/.mc/share/uploads.json` file.
mc: Initialized share downloads `/root/.mc/share/downloads.json` file.
Enter Secret Key: 
Added `monitor` successfully.
```

这样就可以看到内容了

### 2.4. policy

和s3的权限控制类似，minio的用户也是通过policy来控制的，而写法也基本相同，都是json格式的，下面的例子是一个拥有所有权限的例子，all.json

``` json
{
  "Version": "2012-10-17",         
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [                      //  可以做出的行动（权限）
		"s3:ListAllMyBuckets",          //  查看所有的“桶”列表
		"s3:ListBucket",               //  查看桶内的对象列表
		"s3:GetBucketLocation",         
		"s3:GetObject",               //   下载对象
		"s3:PutObject",               //   上传对象
		"s3:DeleteObject"             //   删除对象
      ],
      "Resource": [
        "arn:aws:s3:::*"              // （应用到的资源，*表示所有，也可以用路径来控制范围。arn:aws:s3是命名空间）
      ]
    }
  ]
}
```

把这个policy添加到刚才的集群monitor当中

``` bash
mc admin policy add monitor all all.json
Added policy `all` successfully.
```

### 2.5. 用户

添加用户并且给密码

``` bash
mc admin user add monitor thanos Passw0rd
Added user `thanos` successfully.
```

把我们刚才的policy赋给用户UserName

``` bash
mc admin policy set monitor all user=thanos
Policy all is set on user `thanos`
```

### 2.4. nginx

增加minio的控制台配置

``` bash
server {
        listen 9000;
        server_name  10.0.0.11;

location / {
        proxy_pass http://minio;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        }
}

upstream minio {
        server 10.0.1.11:9000;
        server 10.0.1.12:9000;
        server 10.0.1.13:9000;
        server 10.0.1.14:9000;
}

server {
        listen 9001;
        server_name  10.0.0.11;

location / {
        proxy_pass http://minio-console;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        }
}

upstream minio-console {
        server 10.0.1.11:9001;
        server 10.0.1.12:9001;
        server 10.0.1.13:9001;
        server 10.0.1.14:9001;
}
```

## 3. Thanos Store

+ 准备目录

  ``` bash
  mkdir -p /app/thanos/store
  mkdir /etc/thanos
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
            --data-dir=/app/thanos/store \
            --objstore.config-file=/etc/thanos/thanos-minio.yml \
            --http-address=0.0.0.0:10905 \
            --grpc-address=0.0.0.0:10906 \
            --block-sync-concurrency=200 \
            --store.grpc.series-max-concurrency=200 \
            --chunk-pool-size=8GB \
            --max-time=30d
  ExecReload=/bin/kill -HUP 
  TimeoutStopSec=20s
  Restart=always
  LimitNOFILE=20480000
  [Install]
  WantedBy=multi-user.target
  ```

+ 对象存储配置文件/etc/thanos/thanos-minio.yml

  ``` bash
  type: S3
  config:
    bucket: "thanos"
    endpoint: "10.0.1.11:9000"
    access_key: "thanos"
    insecure: true
    secret_key: "Passw0rd"
    http_config:
      idle_conn_timeout: 5m
      response_header_timeout: 10m
      insecure_skip_verify: true
  ```

+ 启动服务

  ``` bash
  systemctl start thanos-store
  ```

  

## 4. Thanos compact

compact的作用就是定期把历史数据存入对象存储，其实他就像是一个cronjob，如果发现满足了条件，就会对对象存储中的数据进行整理

+ 初始化

``` bash
mkdir /app/thanos/compact
```

+ systemd配置:/etc/systemd/system/thanos-compact.service

```  bash
[Unit]
Description=thanos-compact
Documentation=https://thanos.io/
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/sbin/thanos compact \
          --data-dir=/app/thanos/compact \
          --objstore.config-file=/etc/thanos/thanos-minio.yml \
          --http-address=0.0.0.0:10912 \
          --wait \
          --block-sync-concurrency=30 \
          --compact.concurrency=6
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
```

## 5. Thanos sidecar

配置好之后，我们会发现对象存储里面并没有任何数据，官方文档中说

> **Compactor, Sidecar, Receive and Ruler are the only Thanos components which should have a write access to object storage, with only Compactor being able to delete data.**

所以我们修改一下sidecar配置，/etc/systemd/system/thanos-sidecar.service

``` bash
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
          --grpc-address=0.0.0.0:10902 \
          --objstore.config-file=/etc/thanos/thanos-minio.yml
ExecReload=/bin/kill -HUP 
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
```

## 6. thanos-query

修改query的配置文件，增加query对象存储的选项，/etc/systemd/system/thanos-query.service 

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
          --store=10.0.1.11:10902 \
          --store=10.0.1.12:10902 \
          --store=10.0.1.11:10906 \
          --store=10.0.1.12:10906 \
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

