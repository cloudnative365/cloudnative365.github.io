---
title: thanos
keywords: keynotes, architect, monitoring, thanos
permalink: keynotes_L4_architect_2_monitoring_11_thanos.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/11_thanos
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

### 1.1. Federation（联邦）

federation是prometheus的机制之一，主要是为了满足集群场景中高可用需求。但是他有一个致命的缺点，这个要从他federation的机制说起。我们先看图

![See the source image](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/Prometheus_federation.png)

从图上可以看到，两台prometheus分别去采集4个分布在DC当中的prometheus的数据，在global的prometheus中我们应该这样配置

``` bash
- job_name: 'federate'
  scrape_interval: 15s

  honor_labels: true
  metrics_path: '/federate'

  params:
    'match[]':
      - '{job="prometheus"}'
      - '{__name__=~"job:.*"}'

  static_configs:
    - targets:
      - 'prometheus-ulsfo:9090'
      - 'prometheus-codfw:9090'
      - 'prometheus-eqiad:9090'
      - 'prometheus-esams:9090'
```

然后，大家想象一下这个场景，我们在前端使用grafana做展示，而grafana在选择数据源的时候只能选择一个地址，为了让prometheus集群中的两个实例都能够被grafana访问到，我们在prometheus之前配置了一个负载均衡器，比如nginx，让他监听在9090端口，使用nginx的反向代理功能，把请求代理至两个prometheus实例的9090端口上，实现了一定程度上的集群，如果一个prometheus实例挂了，那么，nginx会动态把这个后端摘掉，只把请求代理至另外一个实例。如果，过了一段时间，坏掉的实例恢复了，那么nginx会重新把这个实例加到集群中。这个时候，坏掉的prometheus在他坏掉的时候的数据是没办法收集到的。而这个时候nginx恰好把grafana的请求代理到了这个实例上，那么就会出现断点（gap）。

![gap](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/BqcdB.png)

而这个问题，通过传统的架构是没办法解决的，我们需要借助其他的机制。

### 1.2. 持久存储

使用过prometheus的朋友估计都知道，prometheus的数据并不是持久存储的，默认的retention参数`--storage.tsdb.retention.time`时间是15d。其实他还有很多别的机制比如文件大小`--storage.tsdb.retention.size`，这里就不详细说了。我们要说的是，prometheus本身是支持持久存储的。在[Integration](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)一章中有很多方式，他是通过remote write和remote read写入。他的本质是通过api的方式写入和读取，所以要求持久存储软件支持api方式写入。当然，也有Postgresql这种个例，是通过一个connector连接的。从节省的角度来说，我们可以选择只写的存储，比如ES，资料多，中文支持也好。如果是一个比较完整的方案，就要求读写。在云原生刚开始的时候，大家比较喜欢用clickhouse，因为他本身支持集群，还有人选择influxdb，因为他比较稳定。但是到了现在，我比较推荐cortex和thanos，因为这两个项目目前都是cncf的孵化项目，社区比较活跃，非常有潜力成为prometheus的流行发行版。

### 1.3. Thanos

最近很多朋友问我关于CNCF项目的分级问题，比如毕业的，孵化的或者沙箱的项目靠谱不靠谱。其实说实话，什么都靠谱什么都不靠谱。不靠谱是因为没有软件是完美的，即使商业软件也需要打补丁的。靠谱是因为软件的源代码开源，只要有开发能力，就没有办不成的事情。从另外一个角度来说，我们的k8s使用的注册中心etcd也是孵化软件，他本身也有很多的缺点。但是k8s一样风靡全世界，开源精神的另外一个境界就是他的不完美，不断的完善他才是开源精神的一种升华。

回头Thanos上，我们先来看一下thanos和prometheus的集成图

![image-20200910100749148](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200910100749148.png)

从图中可以看到，原来prometheus是通过集中式的一个prometheus去采集各个集群中的prometheus数据，然后通过federation的方式进行整合的。而thanos是通过为prometheus附加一个sidecar，然后sidecar直接读取prometheus的数据然后整合再由thanos querier进行整合。

## 2. 快速搭建

最近Rancher Lab发了一篇[文章](https://mp.weixin.qq.com/s/E7fF0SKdrQhTESBD3cYTmw)，我觉得写的是比较详细了，但是他的环境就让人望而却步，需要GKE，一个3节点k8s集群带ingress，还需要2个GCS存储桶。想还原这个真的需要投入很多。我们今天就使用1台机器进行demo，使用docker来隔离环境，使用MinIO来模拟S3存储桶。

### 2.1. 初始化Prometheus环境

我们假设有一个Prometheus服务器在EU1的集群环境。有两个Prometheus的实例在US1的集群中，抓取相同的目标

![image-20200911102235566](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911102235566.png)

+ prometheus的配置文件

  + EU1的配置文件prometheus0_eu1.yml

    ``` bash
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: eu1
        replica: 0
    
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['127.0.0.1:9090']
    ```

  + US1的配置文件prometheus0_us1.yml

    ``` bash
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: us1
        replica: 0
    
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['127.0.0.1:9091','127.0.0.1:9092']
    ```

    US1的配置文件prometheus1_us1.yml

    ``` bash
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: us1
        replica: 1
    
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['127.0.0.1:9091','127.0.0.1:9092']
    ```

+ 给prometheus准备数据文件的存储

  ``` bash
  mkdir -p /opt/{prometheus0_eu1_data,prometheus0_us1_data,prometheus1_us1_data}
  ```

+ 部署EU1

  ``` bash
  docker run -d --net=host --rm \
      -v /opt/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
      -v /opt/prometheus0_eu1_data:/prometheus \
      -u root \
      --name prometheus-0-eu1 \
      prom/prometheus:v2.14.0 \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/prometheus \
      --web.listen-address=:9090 \
      --web.enable-lifecycle \
      --web.enable-admin-api
  ```

+ 部署US1

  ``` bash
  docker run -d --net=host --rm \
      -v /opt/prometheus0_us1.yml:/etc/prometheus/prometheus.yml \
      -v /opt/prometheus0_us1_data:/prometheus \
      -u root \
      --name prometheus-0-us1 \
      prom/prometheus:v2.14.0 \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/prometheus \
      --web.listen-address=:9091 \
      --web.enable-lifecycle \
      --web.enable-admin-api
  ```

  ``` bash
  docker run -d --net=host --rm \
      -v /opt/prometheus1_us1.yml:/etc/prometheus/prometheus.yml \
      -v /opt/prometheus1_us1_data:/prometheus \
      -u root \
      --name prometheus-1-us1 \
      prom/prometheus:v2.14.0 \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/prometheus \
      --web.listen-address=:9092 \
      --web.enable-lifecycle \
      --web.enable-admin-api
  ```

### 2.2. 为prometheus附加sidecar

现在我们要为prometheus附加上sidecar了，只不过不是用pod方式，而是docker的--net方式模拟了一下sidecar

![image-20200911104548857](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911104548857.png)

+ 为 EU1的prometheus添加sidecar

  ``` bash
  docker run -d --net=host --rm \
      -v /opt/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
      --name prometheus-0-sidecar-eu1 \
      -u root \
      quay.io/thanos/thanos:v0.13.0 \
      sidecar \
      --http-address 0.0.0.0:19090 \
      --grpc-address 0.0.0.0:19190 \
      --reloader.config-file /etc/prometheus/prometheus.yml \
      --prometheus.url http://127.0.0.1:9090
  ```

+ 为US1的prometheus添加sidecar

  ``` bash
  docker run -d --net=host --rm \
      -v /opt/prometheus0_us1.yml:/etc/prometheus/prometheus.yml \
      --name prometheus-0-sidecar-us1 \
      -u root \
      quay.io/thanos/thanos:v0.13.0 \
      sidecar \
      --http-address 0.0.0.0:19091 \
      --grpc-address 0.0.0.0:19191 \
      --reloader.config-file /etc/prometheus/prometheus.yml \
      --prometheus.url http://127.0.0.1:9091
  ```

  ``` bash
  docker run -d --net=host --rm \
      -v /opt/prometheus1_us1.yml:/etc/prometheus/prometheus.yml \
      --name prometheus-1-sidecar-us1 \
      -u root \
      quay.io/thanos/thanos:v0.13.0 \
      sidecar \
      --http-address 0.0.0.0:19092 \
      --grpc-address 0.0.0.0:19192 \
      --reloader.config-file /etc/prometheus/prometheus.yml \
      --prometheus.url http://127.0.0.1:9092
  ```

+ 验证一下，由于sidecar的原因，我们在配置文件中修改的内容会直接体现到prometheus中

  + 修改prometheus0_en1.yml

    ``` bash
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: eu1
        replica: 0
    
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['127.0.0.1:9090']
      - job_name: 'sidecar'
        static_configs:
          - targets: ['127.0.0.1:19090']
    ```

  + 修改prometheus0_us1.yml

    ``` bash
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: us1
        replica: 0
    
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['127.0.0.1:9091','127.0.0.1:9092']
      - job_name: 'sidecar'
        static_configs:
          - targets: ['127.0.0.1:19091','127.0.0.1:19092']
    ```

  + 修改prometheus1_us1.yml

    ``` bash
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: us1
        replica: 1
    
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['127.0.0.1:9091','127.0.0.1:9092']
      - job_name: 'sidecar'
        static_configs:
          - targets: ['127.0.0.1:19091','127.0.0.1:19092']
    ```

+ 验证

  + 查看配置文件

    ![image-20200911110500231](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911110500231.png)

  + 查看参数`thanos_sidecar_prometheus_up`,一定要保证sidecar是启用的状态

    ![image-20200911110530041](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911110530041.png)

### 2.3. 通过Thanos查询

我们现在添加一个thanos查询来整合我们的数据，架构如下

![image-20200911111044507](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911111044507.png)

+ 添加thanos querier

  ``` bash
  docker run -d --net=host --rm \
      --name querier \
      quay.io/thanos/thanos:v0.13.0 \
      query \
      --http-address 0.0.0.0:29090 \
      --query.replica-label replica \
      --store 127.0.0.1:19190 \
      --store 127.0.0.1:19191 \
      --store 127.0.0.1:19192
  ```

+ 访问thanos的图形界面，和prometheus很像，但是多了几个功能

  + 在查询的时候有去重功能

    ![image-20200911111805575](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911111805575.png)

  + 还有一个new UI，这thanos自己做的，但是一样丑陋无比

    ![image-20200911111839473](/pages/keynotes/L4_architect/2_monitoring/pics/11_thanos/image-20200911111839473.png)