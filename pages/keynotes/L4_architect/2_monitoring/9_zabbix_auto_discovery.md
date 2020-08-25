---
title: Zabbix5.0自动发现
keywords: keynotes, architect, monitoring, zabbix_operations
permalink: keynotes_L4_architect_2_monitoring_9_zabbix_operations.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/9_zabbix_operations
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

如何配置自动发现并且关联模板

zabbix与ldap集成

如何配置zabbix监控vCenter与ESXi

Zabbix与Grafana集成

Zabbix的Proxy模式

Zabbix企业级架构

## 1. 简介

zabbix的亮点之一就是自动发现，配置起来比较简单，而且通过zabbix-agent的配合，能够让机器实现自动化注册。他通过对指定的网段，进行指定方式的扫描，然后通过动作（action）来对识别到的设备进行下一步动作。我们这里说的动作基本就是两种，就是分类和关联模板。我们这次就来实现一个自动识别，分类然后关联模板的实验。

而对于我们常见的vmware虚拟出来的虚拟机，zabbix是原生支持通过调用ESXi或者vCenter的接口（sdk）来实现自动发现的。在4.4版本中zabbix要监控vmware需要导入第三方模板，但是5.0版本的模板中会自带vmware的模板，我们需要通过宏（macro）来传递vmware的地址和只读账号等信息，让zabbix能够正确读取vmware中的信息。有一点需要注意，到这篇文章发表的日期，也就是2020年8月下旬位置，zabbix 5.0LTS版本中的vmware模板识别vCenter 7.0版本是问题的，有一些指标无法正确识别，而ESXi的7.0版本是可以正确读取数据的。

## 2. 自动发现

zabbix在安装完成之后，在Configuration -> Discovery选项卡中，默认有一个规则，是自动检测192.168.0.1到254网段的设备，间隔是1小时，检查的标准是检查Zabbix agent，但是没有启用。我们这里创建一个新的规则。

点击右上角的`Create discovery rule`按钮



## 3. Zabbix和AD集成

### 3.1. 配置zabbix认证方式为AD

为了方便公司内部人员登录zabbix的界面，我们通常会和windows的AD集成，方便用户的管理。

### 3.2. 配置ldap同步脚本

Zabbix的认证是去AD服务器认证的，但是账号还是需要手动创建，为了让我们的工作更加简便，我们会使用脚本的方式，定期把ldap中的用户信息导入到Zabbix中。这样，如果有新员工入职了，只要获得了AD的账户，并且获得了对应的AD权限，那么，他就可以登录Zabbix控制台，而不需要Zabbix管理员手动添加账户了。



## 4. Zabbix监控vCenter和ESXi

### 4.1. 配置Zabbix监控vCenter或者ESXi

Zabbix 5.0中，Zabbix的vmware监控模板对于vCenter7.0版本支持并不是很好，但是ESXi目前还好。如果一定要使用Zabbix监控ESXi，还是建议手动添加ESXi主机。

### 4.2. 对发现的主机自动分组和关联模板



## 5. Zabbix与Grafana集成



## 6. Zabbix的proxy模式



## 7. Zabbix企业级架构