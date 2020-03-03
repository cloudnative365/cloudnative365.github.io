---
title: kubernetes架构
keywords: keynotes, L2_advanced, 1_kubernetes, 4_learn_kubernetes, 1_Architecture
permalink: keynotes_L2_advanced_1_kubernetes_4_learn_kubernetes_1_Architecture.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/1_Architecture
typora-root-url: ../../../../../../cloudnative365.github.io

---

我们以往在实现负责安装部署应用程序的时候，像nginx，mysql，或者一个网站架构，手动去做是非常繁琐的。所以我们后来有了运维工具，Ansible，puppet，而ansible这种工具其实就是一种应用编排工具，服务的安装，配置，启动，通过我们定义的playbook，通过一些参数的定义，逻辑的处理，完成对多种应用程序的组合部署。