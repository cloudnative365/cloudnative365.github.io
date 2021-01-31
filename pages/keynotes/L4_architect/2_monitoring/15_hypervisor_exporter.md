---
title: hypervisor_exporter
keywords: keynotes, architect, monitoring, 15_hypervisor_exporter
permalink: keynotes_L4_architect_2_monitoring_15_hypervisor_exporter.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/15_hypervisor_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

hypervisor主要是指服务器虚拟化的管理器，我们比较常见的IaaS有下面几种

+ vmware(vCenter + ESXi): vmware是商用软件，vCenter自身提供了一个监控工具，叫vRealize，这个工具需要购买Enterprise Licens才能使用。
+ rhv：红帽的IaaS解决方案之一，开源版的叫ovirt，底层使用的是KVM，管理工具叫rhv-m
+ hyperV：windows的虚拟化方案，商业解决方案中最便宜的，只要购买Windows正版授权，hyperV也免费使用。
+ Openstack：OpenStack目前在企业中应用的成功案例不多，但是一些公司，比如华为，阿里之类的公司基于Openstack的某个版本做了一下二次开发之后的产品倒是非常常见

Windows的exporter很有意思，因为他不仅可以监控Windows系统本身，连上面一些常用的服务也可以监控起来，比如sql-server，iis，hyperv，exchange, .NET信息等等

## 2. 监控方案

对于这类的监控，exporter基本都是去调用对应hypervisor的接口来实现监控的。基本都是由第三方工具来维护的，所以说解决方案不止一个，好的软件层出不穷，如果我们觉得都不好，还可以自己写一个。

### 2.1. vmware

+ 下载地址：[点这个](https://github.com/pryorda/vmware_exporter)

+ 由于这个是python做的，建议大家直接使用docker镜像

  ``` bash
  docker run -it --rm  -p 9272:9272 -e VSPHERE_USER=${VSPHERE_USERNAME} -e VSPHERE_PASSWORD=${VSPHERE_PASSWORD} -e VSPHERE_HOST=${VSPHERE_HOST} -e VSPHERE_IGNORE_SSL=True -e VSPHERE_SPECS_SIZE=2000 --name vmware_exporter pryorda/vmware_exporter
  ```

  从命令可以看出来，我们必须需要三个信息

  + VSPHERE_USER：只读账号，去找VM管理员要把
  + VSPHERE_PASSWORD：账号的密码
  + VSPHERE_HOST：一般来说，都是https://vmware-host。
    + 注意1：他会默认去找https://vmware-host/sdk，如果我们的路径前端有代理，或者是修改过路径，就需要修改这个参数
    + 注意2：这地址可以是vCenter的地址，也可以是ESXi的地址，区别就在于能够读取信息的多少

+ 如果大家一定要直接安装，最好使用pip安装`pip install vmware_exporter`，然后使用`vmware_exporter -c /path/to/your/config`来启动他。

+ 配置文件如下，有三个例子

  ``` bash
  default:
      vsphere_host: "vcenter"
      vsphere_user: "user"
      vsphere_password: "password"
      ignore_ssl: False
      specs_size: 5000
      fetch_custom_attributes: True
      fetch_tags: True
      fetch_alarms: True
      collect_only:
          vms: True
          vmguests: True
          datastores: True
          hosts: True
          snapshots: True
  
  esx:
      vsphere_host: vc.example2.com
      vsphere_user: 'root'
      vsphere_password: 'password'
      ignore_ssl: True
      specs_size: 5000
      fetch_custom_attributes: True
      fetch_tags: True
      fetch_alarms: True
      collect_only:
          vms: False
          vmguests: True
          datastores: False
          hosts: True
          snapshots: True
  
  limited:
      vsphere_host: slowvc.example.com
      vsphere_user: 'administrator@vsphere.local'
      vsphere_password: 'password'
      ignore_ssl: True
      specs_size: 5000
      fetch_custom_attributes: True
      fetch_tags: True
      fetch_alarms: False
      collect_only:
          vms: False
          vmguests: False
          datastores: True
          hosts: False
          snapshots: False
  ```

+ 配置prometheus，同样是三个例子

  ``` bash
    - job_name: 'vmware_vcenter'
      metrics_path: '/metrics'
      static_configs:
        - targets:
          - 'vcenter.company.com'
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: localhost:9272
  
    - job_name: 'vmware_esx'
      metrics_path: '/metrics'
      file_sd_configs:
        - files:
          - /etc/prometheus/esx.yml
      params:
        section: [esx]
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: localhost:9272
  
  # Example of Multiple vCenter usage per #23
  
  - job_name: vmware_export
      metrics_path: /metrics
      static_configs:
      - targets:
        - vcenter01
        - vcenter02
        - vcenter03
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: exporter_ip:9272
  ```

### 2.2. rhv

+ 下载地址：[点这个](https://github.com/czerwonk/ovirt_exporter)

+ 使用命令的方式

  ``` bash
  ./ovirt_exporter -api.url ${RHV_HOST} -api.username ${RHV_USERNAME} -api.password ${RHV_PASSWORD} -api.insecure-cert
  ```

+ 注意：在实际的生产系统当中，我们通常是会做统一登录认证的，比如AD，ldap等，如果是集成了AD认证，那么${RHV_USERNAME}就是yourAccount@yourdomain.com@yourdomain.com。

### 2.3. hyperV

+ 主要还是使用windows-exporter中的hyperV选项

### 2.4. Openstack

+ 下载地址：[点这里](https://github.com/openstack-exporter/openstack-exporter)