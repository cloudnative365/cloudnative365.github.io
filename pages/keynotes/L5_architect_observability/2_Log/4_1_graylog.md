---
title: graylog初体验
keywords: keynotes, architect, observability, log, gray, gray_basic
permalink: keynotes_L5_architect_observability_2_log_4_1_graylog_basic.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/4_1_graylog_basic
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. Graylog

[Graylog](https://www.graylog.org/)是一款专门为日志相关需求而开发的软件。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/628d20c15e5d13777989e579_Graylog_Full Color.svg)

也就是说我们常用的日志功能他基本都具备了，比如聚合、分析、审计、展示和报警工具。因为他是专门为日志系统而开发的，所以他相对于传统的ELK系统要简单。注意，我说的是简单，而不是轻量，因为graylog是需要依托于ElasticSearch来做日志引擎的。因此，可以认为是ElasticSearch的功能扩展。相比于Loki来说，graylog并没有在本质上改变日志存储介质，只是在日志的存取的过程中，把用户常用功能全都包装在了graylog程序当中，让使用更加简单而已。

![img](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/graylog_functions.jpeg)

Graylog和Elastic类似，提供免费版（Open Source）和收费版（Operations和Security），而收费版包括公有云和自建两种方式。

![image-20220925154112779](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220925154112779.png)

比如报警，机器学习，预测，运维仪表盘和发送报告等功能都是需要收费的。也就是说，graylog的免费版只能实现日志的存储和查询。当然，如果有大神喜欢自己开发插件也是支持的，但是他的社区贡献者和ES相比还是逊色许多。

## 2. 功能与架构

Graylog server由三个部分组成，graylog程序，ES和mongoDB。

+ ES用来实现日志的存取，比较消耗内存。
+ 而MongoDB用来实现graylog本身的一些配置信息。因此MongoDB的资源消耗并不大，如果资源紧张可以和Graylog放在一起。而ES单独部署，这样会合理利用资源。
+ Graylog服务是一个带Web接口的服务，是一个计算密集型服务，建议把CPU多分一点。

![architec_small_setup](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/architec_small_setup.png)

Graylog相对于ELK来说，相当于在ES外面包了一层，取代了原来ELK架构中ES的位置，从而实现对日志的优化。

![architecturecomparison.png](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/architecturecomparison.png)



## 3. 单机版安装和配置

### 3.1. 环境

- 操作系统：Centos7
- Java > 1.8
- Elasticsearch 7.10.2
- MongoDB 4.4

### 3.2. 安装与配置

参考：https://docs.graylog.org/v1/docs/centos

#### 3.3.1. Java 1.8

+ 配置epel源

  ``` bash
  wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
  ```

+ 安装

  ``` bash
  yum install -y java
  ```

#### 3.3.2. Elasticsearch

+ 配置yum源

  ``` bash
  rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
  cat >/etc/yum.repos.d/es.repo<< EOF
  [elasticsearch]
  name=Elasticsearch repository for 7.x packages
  baseurl=https://artifacts.elastic.co/packages/7.x/yum
  gpgcheck=1
  gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
  enabled=0
  autorefresh=1
  type=rpm-md
  EOF
  ```

+ 安装

  ``` bash
  yum install --enablerepo=elasticsearch elasticsearch
  ```

+ 如果网络实在不好可以手动下载rpm包

  ``` bash
  wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-x86_64.rpm
  ```

+ 启动elasticseach

  ``` bash
  systemctl start elasticsearch
  ```

  

#### 3.3.2. MongoDB

+ 配置yum源

  ``` bash
  cat >/etc/yum.repos.d/mongodb-org-4.4.repo<< EOF
  [mongodb-org-4.4]
  name=MongoDB Repository
  baseurl=https://mirrors.aliyun.com/mongodb/yum/redhat/7Server/mongodb-org/4.4/x86_64/
  gpgcheck=1
  enabled=1
  gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
  EOF
  ```

+ 安装

  ``` bash
  yum install -y mongodb-org
  ```

+ 启动

  ``` bash
  systemctl start mongod
  ```


#### 3.3.3. Graylog

+ 配置yum源

  ``` bash
  rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-4.2-repository_latest.rpm
  ```

+ 安装

  ``` bash
  yum install graylog-server
  ```

+ 生成password_secret，这个相当于一个随机的token，如果在高可用环境下，每个节点都要使用相同的

  ``` bash
  yum install -y pwgem
  pwgen -N 1 -s 96
  Zhhoy87hfBzPdTsBCP4EhhZW4P2Mb8h4rjxfLz53789KxwZdn9qODkm7Xps1GfSfkk3ug7Z2vz5T04Uug57o42J4WPjHI5QG
  ```

+ 生成root_password_sha2，这个是admin用户的密码

  ``` bash
  echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
  Enter Password: admin
  8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
  ```

+ 配置/etc/graylog/server/server.conf

  ``` bash
  password_secret = Zhhoy87hfBzPdTsBCP4EhhZW4P2Mb8h4rjxfLz53789KxwZdn9qODkm7Xps1GfSfkk3ug7Z2vz5T04Uug57o42J4WPjHI5QG
  root_password_sha2 = 8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
  http_bind_address = 0.0.0.0:9000
  ```

+ 启动

  ``` bash
  systemctl start graylog-server.service
  ```

+ 最后使用admin/admin登录一下

  ![](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930150702978.png)

  ![image-20220930150738105](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930150738105.png)

## 4. 收集日志

### 4.1. 配置监听

+ 我们选择上面的System -> Input

  ![image-20220930160522591](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930160522591.png)

+ 我们这里使用docker来做一个测试，使用GELF方式

  ![image-20220930160905704](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930160905704.png)

+ 监听在12201端口上

  ![image-20220930161039864](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930161039864.png)

+ 看到running就可以了

  ![image-20220930161116901](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930161116901.png)

### 4.2. 配置docker发日志

+ 安装docker

  ``` bash
  yum install -y docker
  ```

+ 启动docker

  ``` bash
  systemctl start docker
  ```

+ 启动一个docker发送日志给graylog

  ``` bash
  docker run -d \
             --log-driver=gelf \
             --log-opt gelf-address=udp://localhost:12201 \
             --log-opt tag="log-test-container-A" \
             busybox sh -c 'while true; do echo "This is a log message from container A"; sleep 10; done;'
  
  docker run -d \
             --log-driver=gelf \
             --log-opt gelf-address=udp://localhost:12201 \
             --log-opt tag="log-test-container-B" \
             busybox sh -c 'while true; do echo "This is a log message from container B"; sleep 10; done;'
  ```

+ 过一会就看到日志过来了

  ![image-20220930172647025](/pages/keynotes/L5_architect_observability/2_Log/pics/4_1_graylog_basic/image-20220930172647025.png)
