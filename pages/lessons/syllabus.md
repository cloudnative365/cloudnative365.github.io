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

### 2. 进阶课程

+ kubernetes系列讲座：全面理解kubernetes知识，穿插CKAD和CKA考试的内容
  + kubernetes讲座（上）-- CKA考试指南
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
  + kubernetes讲座（下）-- kubernetes其他知识
    1. stateful控制器
    2. Ingress控制器
    3. 权限控制和RBAC
    4. 网络和网络策略
    5. 调度器
    6. 资源需求和限制
    7. 资源指标
+ kubernetes系列工具与生态：介绍CNCF旗下的项目和生产系统中会用到的工具
  + Helm：CNCF孵化项目之一
  + Harbor：
  + CoreDNS从入门到精通：CNCF毕业项目之一，从DNS到CoreDNS
  + etcd从入门到精通：CNCF毕业项目之一，从KV存储到etcd
  + Gitlab

### 3. 高级课程

+ 云原生Roadmap：介绍云原生的相关工具

  + Prometheus从入门到精通：CNCF毕业项目之一，从监控系统到Prometheus

  + Containerd从入门到精通：CNCF毕业项目之一，从容器化到Containerd

### 4. 架构师课程

+ 高可用：搭建高可用的kubernetes
+ 监控：介绍多种监控方案
  + Jaeger从入门到精通：CNCF毕业项目之一，从服务跟踪到Jaeger
+ 日志：介绍多种日志采集方案
  + Fluentd从入门到精通：CNCF毕业项目之一，轻量级数据收集工具
+ 负载均衡：介绍多种负载均衡方案
  + Envoy从入门到精通：CNCF毕业项目之一，从负载均衡到Envoy
+ 持久存储：介绍多种持久存储方案
  + ROOK
+ 微服务治理：介绍多种微服务治理方案
  + Istio
+ DevOps工具：介绍多种DevOps工具
  + Jenkins
  + Spinnaker
+ IaC：介绍多种IaC工具
  + Terraform

## 主讲人

张克：12年运维经验，曾经就职于诺和诺德，捷信，爱立信，渣打等知名外企，经常出差去欧洲各国和美国参加国际会议，对于国外的开源技术和先进的方法论了如指掌。热衷开源技术，善于总结，喜欢分享心得。

## 课程安排

每一套课程都有若干章节组成，每一个章节都由若干的小节组成，每个小节都有若干的段落组成，每个段落都由理论，实践两个部分组成。例如：1. 基础课程 由两个章节（Linux基础和虚拟化基础）组成，虚拟化基础由容器化基础和其他若干小节组成，容器化基础由容器化历史和容器化工具等若干段落组成，而容器化工具的介绍会分为理论和实践两部分视频组成。