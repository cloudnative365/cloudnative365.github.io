---
title: Ceph高可用
keywords: keynotes, architecture, storage, object_storage, ceph_cluster
permalink: keynotes_L7_architect_storage_1_storage_1_2_ceph_cluster.html
sidebar: keynotes_L7_architect_storage_sidebar
typora-copy-images-to: ./pics/1_2_ceph_cluster
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

Ceph安装



## 1. Ceph安装

Ceph是集群架构的，[官方文档](https://docs.ceph.com/en/latest/install/get-packages/?spm=a2c6h.13651104.0.0.4b6622d15f37Ih)

[版本选择](https://docs.ceph.com/en/latest/releases/#active-releases)

### 1.1. RPM安装

导入仓库的key

``` bash
sudo rpm --import 'https://download.ceph.com/keys/release.asc'
```

配置yum源

``` bash
cat > /etc/yum.repos.d/ceph.repo << EOF
[ceph]
name=Ceph packages for $basearch
baseurl=https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/\$basearch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
EOF
```

修改hosts文件

``` bash
cat /etc/hosts
10.114.2.68 ceph-mon
10.114.2.129 ceph-node0
10.114.2.130 ceph-node1
10.114.2.131 ceph-node2
```

安装包

``` bash
ceph-deploy install ceph-node0 ceph-node1 ceph-node2
```

