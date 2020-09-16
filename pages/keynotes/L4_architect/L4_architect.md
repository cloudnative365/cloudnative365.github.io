---
title: 架构师课程
keywords: keynotes, architect, 
permalink: keynotes_L4_architect.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/L4_architect
typora-root-url: ../../../cloudnative365.github.io
---

## 课程大纲

### 1. 高可用

+ kubeadm搭建k8s高可用（yum版）
+ kubeadm搭建k8s高可用（apt版）
+ kops安装kubernetes
+ rancher安装kubernetes
+ 安装minikube

### 2. 监控

+ 监控概述
+ Prometheus概述
+ exporter概述
+ PromQL查询
+ 常用的监控指标
+ 常见的监控函数
+ 使用Grafana展示监控数据
+ 搭建高可用Grafana
+ 使用alertmanager进行报警
+ 搭建高可用alertmanager
+ TIGK监控
+ Influxdb持久化prometheus数据
+ 高可用Influxdb与prometheus
+ thonas

### 3. 日志

+ 日志收集概述
+ ElasticSearch概述
+ logstash与日志收集agent
+ kibana展示数据与kibana集群
+ fluentd收集日志
+ fluentdbit收集日志
+ fluentd的其他用法

### 4. 负载均衡

+ 外部负载均衡方案
+ consul-template
+ 内部负载均衡方案
+ 从nginx到envoy

+ 使用公有云的负载均衡作为k8s的负载均衡

### 5. 持久存储

+ 分布式存储概述
+ nfs
+ Ceph
+ openEBS
+ Rook
+ LongHorn
+ 使用公有云的持久存储

### 6. 微服务治理

+ Istio

### 7. DevOps工具

+ Gitlab
+ Harbor
+ Jenkins
+ Spinnaker

### 8. IaC工具

+ Ansible
+ Terraform