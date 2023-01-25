---
title: 生产级别的Opensearch 2.4
keywords: keynotes, architect, observability, log, opensearch, opensearch_in_prodution
permalink: keynotes_L5_architect_observability_2_log_1_2_opensearch_in_production.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_3_opensearch2_4_in_production
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 简介

OpenSearch的2.0版本对应了ElasticSearch的8.0版本，但是，功能上已经和ES算是两个不同的分支了。OpenSearch从1.0版本升级到2.0版本的目的和ES从7.0版本升级到8.0版本的原因基本相同。主要是由于lucence引擎从8升级到了9，所以两款软件都进行了大版本的升级。

当然，他们各自在功能上也都有了非常大的增加。目前ES依然是在商业功能上发力，而OpenSearch还是在努力的解决兼容性问题，从而占领更大的市场。

## 2. 组件和架构

OpenSearch 2.x和1.x的架构和组件完全一致，可以参考我上一篇文章

## 3. 安装和配置

### 3.1. 服务器

| 服务器IP     | 操作系统   | 组件               |
| ------------ | ---------- | ------------------ |
| 10.39.64.243 | CentOS 7.9 | OpenSearch，Kibana |
| 10.39.64.233 | CentOS 7.9 | OpenSearch，Kibana |
| 10.39.64.242 | CentOS 7.9 | OpenSearch，Kibana |

### 3.2. 准备环境

+ 创建相关的目录

  ``` bash
  mkdir /app/{opensearch,opensearch-dashboards}
  mkdir /app/opensearch/{data,logs}
  ```

+ 下载安装包

  ``` bash
  wget -c https://artifacts.opensearch.org/releases/bundle/opensearch/2.4.1/opensearch-2.4.1-linux-x64.tar.gz -P /app/opensearch
  wget -c https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.4.1/opensearch-dashboards-2.4.1-linux-x64.tar.gz -P /app/opensearch-dashboards
  ```

+ 解压

  ``` bash
  cd /app/opensearch
  tar xf opensearch-2.4.1-linux-x64.tar.gz
  cd /app/opensearch-dashboards
  tar xf opensearch-dashboards-2.4.1-linux-x64.tar.gz
  ```

+ 关闭swap

  ``` bash
  swapoff -a
  ```

+ 修改/etc/sysctl.conf

  ``` bash
  cat >/etc/sysctl.conf<<EOF
  vm.max_map_count=262144
  EOF
  sysctl -p
  ```

+ 创建opensearch用户

  ``` bash
  useradd opensearch
  chown -R opensearch:opensearch /data/opensearch
  chown -R opensearch:opensearch /data/opensearch-dashboards
  ```

  
