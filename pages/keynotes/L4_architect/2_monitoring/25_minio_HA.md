---
title: MinIO高可用
keywords: keynotes, L4_architect, monitoring, minio_HA
permalink: keynotes_L4_architect_2_monitoring_25_minio_HA.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/25_minio_HA
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

### 1.1. 介绍

minIO是目前一个比较轻量级的分布式文件系统，遵循apache 2.0开源协议。我们可以把他视为数据中心版的S3（S3是AWS的一个云存储服务，相当于阿里云的oss），他是目前兼容性比较好的对象存储。他本身是由golang开发的，所以运行效率可以和ceph媲美。目前社区也非常活跃，git的代码提交者中，我们可以看到很多中国程序员的身影。

### 1.2. 订阅

对于存储这种非常重要的部分，大部分公司都会非常慎重，毕竟数据无价嘛，但是与其购买昂贵的存储设备，倒不如尝试分布式存储，毕竟在中国的大部分互联网公司都已经采用了这种方案，而且运行起来高效而且节省开支。如果使用ceph，我们可以去买redhat的订阅，如果使用minIO，可以使用minIO的[订阅](https://min.io/pricing)。

![image-20201004222453147](/pages/keynotes/L4_architect/2_monitoring/pics/25_minio_HA/image-20201004222453147.png)

### 1.3. 功能与集成

既然称minIO是s3，那么s3的一些特性，minIO是完全具备的。并且，并不是只有商业版才有，而是开源版就具备了。比如：Bucket的版本控制，Bucket的生命周期管理，多租户，对外暴露API，支持Veeam备份，角色访问控制ARN，还可以发消息给中间件（redis，ES，kafka）。这些功能我们会在对象存储那一个专题中详细讲解。

## 2. 架构

由于是非常轻量级的软件，所以架构上也没有这么复杂，他使用操作系统的文件系统作为存储介质，我们在向任意节点写数据的时候，minIO会自动同步数据到另外的节点，而机制叫做erasure code（[纠删码](https://docs.min.io/docs/minio-erasure-code-quickstart-guide.html)）来保证集群的稳定，保证数据可用，所以我们建议至少使用4个节点来构建集群。

如果一个N节点的分布式MinIO，只要有N/2节点在线，数据就是安全的。但是要保证至少有N/2+1个节点来创建新的对象。比如：我们的集群有4个节点，每个节点上一块盘，就算有2两个节点宕机，这个集群仍然是可读的，但是，我们需要3个节点才能让集群写数据。这就是为什么我们要有4个节点来构建集群。

早期版本中，每个租户至少有4个盘，最多有16个盘，这个是纠删码的限制，而新版本中是没有[限制](https://docs.min.io/docs/minio-server-limits-per-tenant.html)的。如果想要实现多租户，就需要借助于kubernetes来构建多个MinIO实例，或者启动多个实例来实现多租户。也就是说，一个进程对应一个实例，一个实例对应一个租户。

我们这次实验由下面这四台机器构成

| 节点   | IP             | data             |
| ------ | -------------- | ---------------- |
| minio1 | 192.168.133.71 | /data/minio/data |
| minio2 | 192.168.133.72 | /data/minio/data |
| minio3 | 192.168.133.73 | /data/minio/data |
| minio4 | 192.168.133.74 | /data/minio/data |

## 3. 搭建集群

### 3.1. 准备环境

+ 修改主机名

  ``` bash
  hostnamectl set-hostname minio1
  ```

+ 修改hosts文件

  ``` bash
  cat >> /etc/hosts <<EOF
  192.168.133.71 minio1
  192.168.133.72 minio2
  192.168.133.73 minio3
  192.168.133.74 minio4
  EOF
  ```

+ 修改系统最大文件数

  ``` bash
  echo "*   soft    nofile  65535" >> /etc/security/limits.conf
  echo "*   hard    nofile  65535" >> /etc/security/limits.conf
  ```

+ 创建数据目录，如果有条件，建议MinIO的盘使用单独的SSD盘来做，使用2块盘来做RAID0性能会更好

  ``` bash
  mkdir -p /data/minio/{run,data} && mkdir -p /etc/minio
  ```

+ 下载minio到/data/minio/run目录下

  ``` bash
  (cd /data/minio/run && wget https://dl.min.io/server/minio/release/linux-amd64/minio)
  ```

### 3.2. MinIO集群

+ 启动脚本/data/minio/run/run.sh

  ``` bash
  cat > /data/minio/run/run.sh <<EOF
  #!/bin/bash
  export MINIO_ACCESS_KEY=Minio
  export MINIO_SECRET_KEY=Passw0rd
  
  /data/minio/run/minio server --config-dir /etc/minio \
  http://192.168.133.71/data/minio/data \
  http://192.168.133.72/data/minio/data \
  http://192.168.133.73/data/minio/data \
  http://192.168.133.74/data/minio/data
  EOF
  ```

+ systemd配置文件minio.service

  ``` bash
  cat > /usr/lib/systemd/system/minio.service <<EOF
  [Unit]
  Description=Minio service
  Documentation=https://docs.minio.io/
  
  [Service]
  WorkingDirectory=/data/minio/run/
  ExecStart=/data/minio/run/run.sh
  
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

+ 修改权限

  ``` bash
  chmod +x /usr/lib/systemd/system/minio.service && chmod +x /data/minio/run/minio && chmod +x /data/minio/run/run.sh
  ```

+ 启动集群试试吧

  ``` bash
  systemctl daemon-reload
  systemctl enable minio && systemctl start minio
  ```

### 3.3. 使用nginx做代理

我们使用nginx来做负载均衡，当然，其他http代理也可以

``` bash
upstream minio{
        server 192.168.133.71:9000;
        server 192.168.133.72:9000;
        server 192.168.133.73:9000;
        server 192.168.133.74:9000;
}
server {
        listen 9000;
        server_name minio;
        location / {
                proxy_pass http://minio;
                proxy_set_header Host $http_host;
                client_max_body_size 1000m;
        }
}
```



