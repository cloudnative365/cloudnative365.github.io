---
title: NFS
keywords: keynotes, architecture, storage, nas, nfs
permalink: keynotes_L7_architect_storage_1_storage_3_2_nfs.html
sidebar: keynotes_L7_architect_storage_sidebar
typora-copy-images-to: ./pics/3_2_nfs
typora-root-url: ../../../../../cloudnative365.github.io
---

## 2. GlusterFS安装

参考[官方文档](https://docs.gluster.org/en/latest/Install-Guide/Install/#setup-method-2-setting-up-on-physical-servers)，环境为两台CentOS 7.9，还需要参考CentOS的[文档](https://wiki.centos.org/SpecialInterestGroup/Storage/gluster-Quickstart)，我们先把yum源配置好

``` bash
yum -y install centos-release-gluster
```

### 2.1. 准备

我们最少的启动环境是两台，我们需要在每一台上做初始化工作。文件的大小随意，由于每个节点的数据是同步的，所以最好所有的节点盘的大小一致。

``` bash
# mkfs.xfs -i size=512 /dev/sdc1
# mkdir -p /bricks/brick1
# vi /etc/fstab
```

增加下面的内容

``` bash
/dev/sdc1 /bricks/brick1 xfs defaults 1 2
```

重新mount

``` bash
mount -a && mount
```

### 2.2. 安装

每台机器都要先安装

``` bash
yum -y install glusterfs-server
systemctl enable glusterd
```

配置两台机器互信，在服务器1上信任服务器2

``` bash
gluster peer probe 10.39.196.103
```

在服务器2上信任服务器1

``` bash
gluster peer probe 10.39.196.112
```

### 2.3. 配置

在所有的服务器上都配置

``` bash
mkdir /bricks/brick1/gv0
```

然后在任意一台服务器上配置

``` bash
gluster volume create gv0 replica 2 10.39.196.112:/bricks/brick1/gv0 10.39.196.103:/bricks/brick1/gv0
gluster volume start gv0
```

查看状态

``` bash
gluster volume info
```

### 2.4. 测试

``` bash
# mount -t glusterfs server1:/gv0 /mnt
# for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done
```

