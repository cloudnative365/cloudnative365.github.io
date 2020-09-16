---
title: exporter概览
keywords: keynotes, architect, monitoring, exporter_overview
permalink: keynotes_L4_architect_2_monitoring_3_exporter_overview.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/3_exporter_overview
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. exporter概览

前面我们尝试过了使用prometheus server抓举数据，只不过数据是我们随机生成的，而且数据和服务器是在一台机器。而我们实际的生产中最常使用的endpoint叫exporter，exporter可以被认为是prometheus的客户端或者agent，只不过这个agent并不是推送数据到prometheus服务器，而是暴露（expose）指标（metrics）到某一个端口，这个端口是可以通过http的方式访问到的。而我们通过scrapy的方式就可以抓取到。

官方网站一共给我们提供了下面几种exporter的[下载](https://prometheus.io/download/)

+ blackbox_exporter
+ consul_exporter
+ graphite_exporter
+ haproxy_exporter
+ memcached_exporter
+ mysqld_exporter
+ node_exporter
+ statsd_exporter

而实际上，我们是可以自己定义exporter的，在官网上[列出](https://prometheus.io/docs/instrumenting/exporters/)的第三方exporter不计其数。

如果这还不够，我们还可以开发[自己的exporter](https://prometheus.io/docs/instrumenting/writing_exporters/)，当然，要遵循一定的[规则](https://prometheus.io/docs/instrumenting/exposition_formats/)。

最后，如果exporters依然不能满足需求，那么我们就[自己写吧](https://prometheus.io/docs/instrumenting/clientlibs/)，官方声明支持Go，java，scala，python和ruby的客户端，而实际上非官方的还有10多种语言可以来开发。**总有一款适合你！**

## 2. node_exporter

这么多的客户端我们不能一一尝试，我这里用node_exporter来给大家做一个演示。这个主要是来监控主机的，支持linux和mac，没有windows。我们下面的演示是在mac上做的，在linux上也是一样的。

### 2.1. 启动node_exporter

+ 下载地址：[node_exporter-1.0.0.darwin-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.0.0/node_exporter-1.0.0.darwin-amd64.tar.gz)

+ 解压

  ``` bash
  tar xf node_exporter-1.0.0.darwin-amd64.tar.gz
  cd node_exporter-1.0.0.darwin-amd64
  ```

+ 启动

  他默认是不提供配置文件的，所以直接运行就可以了

  ``` bash
  ./node_exporter
  ```

  当然，不是没有配置，我们可以使用`node_exporter -h`来查看详细的配置，或者使用`--web.config=""`来指定TLS证书的位置。

+ 打开浏览器，访问http://localhost:9100/metrics。可以看到下面的信息。

![image-20200610103335829](/pages/keynotes/L4_architect/2_monitoring/pics/3_exporter_overview/image-20200610103335829.png)

看起来好像和prometheus服务器暴露出来的信息的格式是差不多的，只不过我们没办法进行数据的查询，我们需要通过prometheus抓取之后才能够进行计算。

### 2.2. 配置Prometheus抓取这个endpoint上的数据

我们还是使用前面的prometheus服务器

+ 修改prometheus.yml，添加一个新的job

  ``` yaml
    - job_name: 'laptop_exporter'
      scrape_interval: 5s
      static_configs:
      - targets: ['localhost:9100']
        labels:
          host: 'my_laptop'
  ```

+ 重新启动prometheus并且访问http://localhost:9090

+ 在console选项卡界面，输入我们要查询的项`node_filesystem_size_bytes`

  ![image-20200610104211803](/pages/keynotes/L4_architect/2_monitoring/pics/3_exporter_overview/image-20200610104211803.png)

## 3. mysql_exporter

另外一个常见的exporter是mysql，这个exporter是通过登录mysql数据库内部，从内部取得一些指标后，暴露出来的。

### 2.1. 安装mysql

这个就不演示了，我们只要通过mysql命令行可以登录就可以了，我们这里是本地演示，如果是远程的话需要打开mysql的远程登录

``` bash
# /usr/local/mysql/bin/mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 8.0.20 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

### 2.2. 启动node_exporter

+ 下载地址：[mysqld_exporter-0.12.1.darwin-amd64.tar.gz](https://github.com/prometheus/mysqld_exporter/releases/download/v0.12.1/mysqld_exporter-0.12.1.darwin-amd64.tar.gz)

+ 解压：

  ``` bash
  tar xf mysqld_exporter-0.12.1.darwin-amd64.tar.gz
  cd mysqld_exporter-0.12.1.darwin-amd64
  ```

+ 启动

  ``` bash
  $ export DATA_SOURCE_NAME='root:Passw0rd@(localhost:3306)/'
  $ ./mysqld_exporter
  INFO[0000] Starting mysqld_exporter (version=0.12.1, branch=HEAD, revision=48667bf7c3b438b5e93b259f3d17b70a7c9aff96)  source="mysqld_exporter.go:257"
  INFO[0000] Build context (go=go1.12.7, user=root@0b3e56a7bc0a, date=20190729-12:36:41)  source="mysqld_exporter.go:258"
  INFO[0000] Enabled scrapers:                             source="mysqld_exporter.go:269"
  INFO[0000]  --collect.global_status                      source="mysqld_exporter.go:273"
  INFO[0000]  --collect.global_variables                   source="mysqld_exporter.go:273"
  INFO[0000]  --collect.slave_status                       source="mysqld_exporter.go:273"
  INFO[0000]  --collect.info_schema.innodb_cmp             source="mysqld_exporter.go:273"
  INFO[0000]  --collect.info_schema.innodb_cmpmem          source="mysqld_exporter.go:273"
  INFO[0000]  --collect.info_schema.query_response_time    source="mysqld_exporter.go:273"
  INFO[0000] Listening on :9104                            source="mysqld_exporter.go:283"
  ```

+ 打开浏览器，访问http://localhost:9104/metrics，可以看到下面的信息

  ![image-20200610113154431](/pages/keynotes/L4_architect/2_monitoring/pics/3_exporter_overview/image-20200610113154431.png)

### 2.3. 配置Prometheus抓取这个endpoint上的数据

我们还是使用前面的prometheus服务器

+ 修改prometheus.yml，添加一个新的job

  ``` bash
    - job_name: 'mysql_on_laptop_exporter'
      static_configs:
      - targets: ['localhost:9104']
        labels:
          app: 'mysql_on_laptop'
  ```

- 重新启动prometheus并且访问http://localhost:9090

+ 在console选项卡界面，输入我们要查询的项`mysql_version_info`

![image-20200610113711240](/pages/keynotes/L4_architect/2_monitoring/pics/3_exporter_overview/image-20200610113711240.png)