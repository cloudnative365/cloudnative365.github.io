---
title: 实验环境
keywords: keynotes, basic, labs, wmware_workstation
permalink: keynotes_L2_basic_4_labs_1_vmware_workstation.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/1_vmware_workstation
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 课程目标

使用vmware搭建实验环境

## 2. 简介

vmware算是x86虚拟化市场上的龙头老大了，他旗下的知名产品，比如x86虚拟化平台vsphere系列和windows桌面虚拟化软件workstation，算是最有名的两款产品。当然，他的其他产品也都是商用产品中的佼佼者，比如虚拟桌面系统vmware view，分布式存储vSAN，和容器化产品pivotal都是非常不错的虚拟化产品。但是vmware的特点就是好用，稳定，但是贵！

## 3. 准备hypervisor

安装包可以去官方网站[下载](https://www.vmware.com/cn/products/workstation-pro/workstation-pro-evaluation.html)，目前的版本最新是vmware workstation pro 16。当然，如果有愿意在Linux下用workstation的也可以，那么请使用Linux版。如果有同学使用的是Mac电脑，那么请搜索vmware fution，这是Mac版本的workstation。注意，他们都是收费的，请大家支持正版。

### 3.1. 安装vmware workstation/fustion

双击安装程序

### 3.2. 创建host

### 3.3. 网络模式

## 4. 安装系统

### 4.1. 安装介质

建议大家到国内的镜像源去找，因为镜像源既能提供ISO镜像，又能提供rpm或者apt的源，速度非常快。这些镜像站点是有每天的同步机制的，不用基本不用担心版本差异问题。比如[阿里的源](https://developer.aliyun.com/mirror/)，不仅提供下载，还把各种源的使用方法写的非常详细。比如：

+ ubuntu系统（CKA考试使用的都是ubuntu系统）：[镜像下载](http://mirrors.aliyun.com/ubuntu-releases/)
+ centos系统（我们日常工作有很大机会会用到centos）：[镜像下载](https://mirrors.aliyun.com/centos/)
+ centos for arm系统（我的实验环境，使用的是树莓派4）：[镜像下载](http://mirrors.huaweicloud.com/centos-altarch/8.1.1911/isos/armhfp/)

### 4.2. 安装ubuntu

#### 4.2.1. 通过镜像安装系统

#### 4.2.2. 配置系统

### 4.3. 安装CentOS

#### 4.3.1. 通过镜像安装系统

#### 4.3.2. 配置系统

### 4.4. 在树莓派上安装centos系统

#### 4.3.1. 通过镜像安装系统

#### 4.3.2. 配置系统