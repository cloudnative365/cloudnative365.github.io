---
title: thanos概念
keywords: keynotes, architect, monitoring, thanos_conception
permalink: keynotes_L4_architect_2_monitoring_12_thanos_conception.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/12_thanos_conception
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 架构

我们先来看看官方架构图

![arch.jpg](/pages/keynotes/L4_architect/2_monitoring/pics/12_thanos_conception/arch.jpg)

+ 黄色的部分是prometheus，exporters和alertmanager的三大组件，是prometheus最原始的架构
+ 蓝色的部分中：Thanos Sidecar和Thanos Query就是我们上节课搭建的那两个组件
+ 深灰色的是存储部分，其中，Thanos ruler和Thanos Sidecar使用的是本地硬盘（其实也可以存储到对象存储上，但是没什么必要），而Thanos Storage Gateway和Thanos Compact可以去对象存储中获取数据。
+ Thanos Storage Gateway：我们在上一节课程中看到了，启动时候的参数`--store 127.0.0.1:19190`，这个是sidecar暴露出来的gRPG的API接口，不是Storage Gateway，如果想要使用别的存储，需要使用`thanos store `命令，并且指定配置文件的位置，使用`--objstore.config-file`，我们后面讲集成存储的时候再说。
+ Thanos Compact：他的作用是优化查询历史数据时候的查询速度。
+ Ruler：和Prometheus的ruler差不多，也是通过调用alertmanager来发送报警的

## 2. 安装和配置所有的组件

### 2.1. 准备对象存储

在我们正式安装所有的组件之前，我们还需要安装另外一个组件，这个组件thanos并不提供，他需要借助其他解决方案。[官方](https://thanos.io/tip/thanos/storage.md/)给出的方案有下面几种

![image-20200911172122008](/pages/keynotes/L4_architect/2_monitoring/pics/12_thanos_conception/image-20200911172122008.png)

我们可以看到目前稳定版本的依然是公有云三大巨头，而中国在这方面也不甘落后，阿里和腾讯都有支持，但是都是Beta版本。为了效果更好，我们使用另外一个开源方案，MinIO。这是一个开源的项目，他可以兼容S3协议，连接方式也和S3一样的。我们后面讲分布式存储的时候会说他，其他任何的软件，只要是支持持久存储到s3，我们都可以使用MinIO来替代。我们的体系中还有另外一个项目要使用到MinIO，那就是Spinnaker。

这里我们使用非常简单的方式启动一个S3

``` bash
docker run -p 9000:9000 --name minio \
  -e "MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" \
  -e "MINIO_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" \
  -v /mnt/data:/data \
  -v /mnt/config:/root/.minio \
  minio/minio server /data
```

起来之后就可以看到图形界面了，而在启动时候的两个key和AWS上的两个KEY作用一样，后面配置文件需要用到。

![image-20200913225005302](/pages/keynotes/L4_architect/2_monitoring/pics/12_thanos_conception/image-20200913225005302.png)

我们创建一个bucket，叫thanos

#### 2.2.  Sidecar

在容器时代非常常见的一种扩展容器的方式，如果把thanos部署在容器当中，那么sidecar组件就要部署为sidecar形式。sidecar的作用有两个

+ 把prometheus的数据备份到对象存储中
+ 为Thanos的其他组件提供一个可以访问prometheus的接口，这个接口是使用gRPC API的

注意：如果想让Sidecar可以重新加载Prometheus节点，记得要在prometheus启动的时候加上`--web.enable-lifecycle`选项

#### 2.2.1. 启动参数

如果使用命令直接启动的话使用下面的配置

``` bash
thanos sidecar \
    --tsdb.path            /var/prometheus \          # 需要指定prometheus的数据文件地址
    --prometheus.url       "http://localhost:9091" \  # 暴露查询API的地址
    --objstore.config-file bucket_config.yaml \       # 对象存储的信息
```

部署这个对于prometheus来说，基本不会有任何的影响。同时，他还可以起到备份的作用，如果我们不想备份，就不需要`--objstore.config-file`选项了。

#### 2.2.2. Store API

Sidecar组件还包含了一个Store API，他暴露为gRPG形式的，可以让我们查询到存储在Prometheus中的metrics数据。在刚才的基础上，我们的sidecar参数就会扩展为

``` bash
thanos sidecar \
    --tsdb.path                 /var/prometheus \
    --objstore.config-file      bucket_config.yaml \
    --prometheus.url            http://localhost:9090 \
    --http-address              0.0.0.0:19191 \ # 这个是用来查询sidecar中的数据的
    --grpc-address              0.0.0.0:19090   # 这个是用来查询prometheus中的数据的
```

#### 2.2.3. 上传以前的metrics

如果需要上传以前的数据，我们需要使用`--shipper.upload-compacted`选项，他会把所有的prometheus的数据都上传（从prometheus启动之后的，存储在本地硬盘上的数据都上传）。而且删除的时候需要手动删除，除非我们需要验证从前的数据，否则一定要慎重打开这个选项。

#### 2.2.4. External Labels

prometheus中允许配置`external labels`。这样就可以定义它在全局中的角色了。由于Thanos的目的是为了整合不同实例中的数据，所以定义external label就变得至关重要。

比如，在我们的prometheus中就可以这样定义

``` yaml
global:
  external_labels:
    region: eu-west
    monitor: infrastructure
    replica: A
```

### 2.3. Querier/Query

#### 2.3.1. 启动选项

我们为一个或者多个Prometheus实例配置的Sidecar之后，我们就可以使用query组件来通过PromQL查询来同时查询所有的metrics。

这个组件是无状态的，可以水平扩展的，他可以部署任意的副本，一旦连接上Sidecar之后，他会自动的检测哪个Prometheus服务器需要被连接来做查询。

Query同样使用Prometheus官方的HTTP接口，因此他同样支持外部工具，比如Grafana。他还同时提供了一个类似于Prometheus的界面来查询和存储状态。

我们可以这样启动一个Query

``` bash
thanos query \
    --http-address 0.0.0.0:19192 \ # 图形界面的地址
    --store        1.2.3.4:19090 \ # 静态gRPC地址，用于查询
    --store        1.2.3.5:19090 \ # 还可以设置多个
    --store        dnssrv+_grpc._tcp.thanos-store.monitoring.svc #还支持dns的方式暴露
```

#### 2.3.2. 对Prometheus集群架构中的数据进行去重

刚才在sidecar中说到了external_label，这里对于集群的角色分类，就是靠的这个标签。我们刚才配置了一个标签

``` yaml
global:
  external_labels:
    region: eu-west
    monitor: infrastructure
    replica: A
```

这里我们就针对replica来去重，使用`--query.replica-label`参数

``` bash
thanos query \
    --http-address        0.0.0.0:19192 \
    --store               1.2.3.4:19090 \
    --store               1.2.3.5:19090 \
    --query.replica-label replica
    --query.replica-label replicaX 
```

#### 2.3.3. 不同组件的通信

不同节点间通信的唯一渠道就是我们的gRPC storeAPI。我们通常会这样配置

``` bash
thanos query \
    --http-address 0.0.0.0:19192 \
    --grpc-address 0.0.0.0:19092 \
    --store        1.2.3.4:19090 \
    --store        1.2.3.5:19090 \
    --store        dns+rest.thanos.peers:19092
```

### 2.4. Store Gateway

这个组件是用来为prometheus提供持久存储的，他同样需要暴露一个http地址和grpc地址来允许thanos集群来访问

``` bash
thanos store \
    --data-dir             /var/thanos/store \
    --objstore.config-file bucket_config.yaml \
    --http-address         0.0.0.0:19191 \
    --grpc-address         0.0.0.0:19090   
```

### 2.5. Compactor

这个组件是用来加速历史数据查询的，sidecar和Store Gate都会把一些历史数据写入对象存储当中，但是这些对象存储中的数据都算是历史数据，为了加速对他们的查询，就需要compactor组件了，他会优化对象存储中的数据，来加速查询

``` bash
thanos compact \
    --data-dir             /var/thanos/compact \
    --objstore.config-file bucket_config.yaml \
    --http-address         0.0.0.0:19191
```

### 2.6. ruler

和prometheus的ruler类似，我们后面讲报警再说

### 2.7. 对象存储

一个正常的s3存储的配置如下

``` yaml
type: S3
config:
  bucket: ""
  endpoint: ""
  region: ""
  access_key: ""
  insecure: false
  signature_version2: false
  secret_key: ""
  put_user_metadata: {}
  http_config:
    idle_conn_timeout: 1m30s
    response_header_timeout: 2m
    insecure_skip_verify: false
  trace:
    enable: false
  part_size: 134217728
  sse_config:
    type: ""
    kms_key_id: ""
    kms_encryption_context: {}
    encryption_key: ""
```

最少也需要提供这四个选项`bucket`, `endpoint`, `access_key`, 和 `secret_key`。也就是minIO的四个选项，bucket_config.yaml文件内容如下

``` bash
type: S3
config:
  bucket: ""
  endpoint: "localhost:9000"
  access_key: "AKIAIOSFODNN7EXAMPLE"
  insecure: false
  signature_version2: false
  secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

