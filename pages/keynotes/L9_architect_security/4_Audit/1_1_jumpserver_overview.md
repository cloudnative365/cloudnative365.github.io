---
title: jumpserver概述
keywords: keynotes, architecture, security, audit, jumpserver
permalink: keynotes_L9_architect_security_4_audit_1_1_jumpserver_overview.html
sidebar: keynotes_L9_architect_sidebar
typora-copy-images-to: ./pics/1_1_jumpserver_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

审计

## 2. 架构

对于一个

## 3. 单机版



## 4. 集群版

### 4.1. 集群架构

### 4.2. 

对于一个容纳500人同时在线的系统，可以参考下面的配置

| 功能     | 主机名称 | CPU（核） | 内存（GB） | 磁盘（GB） | 备注                    |
| -------- | -------- | --------- | ---------- | ---------- | ----------------------- |
| Nginx 01 |          | 2         | 4          | 40         | 堡垒机高可用环境Nginx01 |
| Nginx 02 |          | 2         | 4          | 40         | 堡垒机高可用环境Nginx02 |
| App 01   |          | 8         | 16         | 100        | 堡垒机高可用环境App01   |
| App 02   |          | 8         | 16         | 100        | 堡垒机高可用环境App02   |
| MySQL 01 |          | 4         | 8          | 200        | 堡垒机高可用环境MySQL01 |
| MySQL 02 |          | 4         | 8          | 200        | 堡垒机高可用环境MySQL02 |
| 共享存储 |          | 2         | 4          | 500        | 堡垒机高可用环境NFS01   |
| 远程应用 |          | 8         | 16         | 200        | 堡垒机高可用环境App03   |
