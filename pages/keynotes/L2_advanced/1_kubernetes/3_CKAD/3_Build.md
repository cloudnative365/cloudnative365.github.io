---
title: 创建Kubernetes资源
keywords: keynotes, L2_advanced, 1_kubernetes, 3_CKAD, 3_Build
permalink: keynotes_L2_advanced_1_kubernetes_3_CKAD_3_Build.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/3_Build
typora-root-url: ../../../../../../cloudnative365.github.io

---

## 1. 课程目标

- 学习有关运行环境和容器的选项
- 应用的容器化
- 管理本地的仓库
- 部署多容器的pod
- 配置readinessProbes（readiness探针）
- 配置livenessProbes（liveness探针）

## 2. 创建资源

### 2.1. 容器的选择

目前有很多的组织都在竞相开发容器。随着各大社区的开放和互相兼容，作为一个编排工具，kubernetes的开发方向也向着兼容大部分的容器而努力。最早的和最健壮的当属Docker。随着Docker的逐渐完善，kubernetes也逐渐通过支持其他厂商的软件的功能来完善其自身容器的创建和部署，这使得新项目和功能的开发变得更加流行。而在其他容器引擎变得成熟的过程中，kubernetes也变得更加开放和独立。

容器的运行环境是为容器化的应用程序提供请求解析的。Docker是kubernetes的默认引擎，同时，CRI-O, rkt和其他的引擎都是由社区支持的。

