---
title: 教学大纲
keywords: lessons, syllabus
permalink: lessons_syllabus.html
sidebar: syllabus_sidebar
folder: lessons
---

## 课程概览

### 1. 基础课程

+ 容器虚拟化：容器化知识与Docker的使用
  + 容器化基础
  + 使用Docker
  + Docker的网络
  + Docker的存储卷
  + Docker资源限制
+ 容器与镜像：dockerfile和harbor
  + Docker镜像
  + 私有镜像仓库
  + Dockerfile
  + harbor
+ 容器运行时：
  + containerd
  + cri-o

### 2. 进阶课程

+ kubernetes入门讲座（CKA考试指南）
  1. kubernete课程简介
  2. kubernetes基础知识
  3. kubernetes架构和基础概念
  4. 使用kubeadm安装kubernetes
  5. 使用命令行定义资源
  6. 使用清单文件manifest文件定义资源
  7. 标签和标签选择器
  8. 深入理解Pod
  9. 深入理解Deployment
  10. 深入理解Service
  11. 存储卷
  12. configmap和secret
+ kubernetes从入门到精通
  1. 课程介绍
  2. 认识kubernetes
  3. kubernetes架构和基础概念
  4. 安装和配置kubernetes实验环境
  5. 使用命令行管理资源
  6. 使用清单文件管理资源
  7. POD
  8. 标签和标签选择器
  9. Deployment
  10. Service
  11. pv
  12. configmap和secret
  13. API
  14. STATEFULSET
  15. INGRESS
  16. 调度
  17. 日志和错误定位
  18. 自定义资源
  19. HELM
  20. SECURITY

### 3. 高级课程

+ etcd从入门到精通
  + etcd概述
  + 安装etcd集群
  + 使用kubeadm安装etcd集群
  + 扩展etcd节点
  + etcd性能调优
  + etcd备份和恢复
+ kubernetes集群管理
  + 使用kubeadm安装kubernetes集群（外挂etcd）
  + hard way of kubernetes
  + 管理kubernetes的master节点（kubeadm）
+ ingress控制器
  + ingress简介
  + 使用ingress-nginx
  + 使用ingress-envoy
+ helm管理应用
  + helm简介
  + helm安装与配置
  + 详解chart
+ kubernetes外挂存储
  + nfs-provisioner
+ rancher管理集群

### 4. 架构师课程

+ 高可用：搭建高可用的kubernetes
  + kubeadm
  + rke
  + kubespray
  + 二进制
  + minikube
  + kops
+ 监控：介绍多种监控方案
  + Prometheus：
  + Influxdb：
  + Jaeger从入门到精通：CNCF毕业项目之一，从服务跟踪到Jaeger
+ 日志：介绍多种日志采集方案
  + ELK系列产品：filebeat，apm，logstash，elasticsearch，kibana
  + Fluentd从入门到精通：CNCF毕业项目之一，轻量级数据收集工具
+ DNS解决方案：
  + DNS解析原理
  + CoreDNS：
+ 持久存储：介绍多种持久存储方案
  + Ceph
  + rbd-provisioner
  + rook
  + openEBS
+ 微服务治理：介绍多种微服务治理方案
  + Istio
  + linkerd
+ DevOps工具：介绍多种DevOps工具
  + Jenkins
  + Spinnaker
+ IaC：介绍多种IaC工具
  + Terraform

## 主讲人

张克：12年运维经验，曾经就职于诺和诺德，捷信，爱立信，渣打等知名外企，经常出差去欧洲各国和美国参加国际会议，对于国外的开源技术和先进的方法论了如指掌。热衷开源技术，善于总结，喜欢分享心得。

## 课程安排

每一套课程都有若干章节组成，每一个章节都由若干的小节组成，每个小节都有若干的段落组成，每个段落都由理论，实践两个部分组成。例如：1. 基础课程 由两个章节（Linux基础和虚拟化基础）组成，虚拟化基础由容器化基础和其他若干小节组成，容器化基础由容器化历史和容器化工具等若干段落组成，而容器化工具的介绍会分为理论和实践两部分视频组成。