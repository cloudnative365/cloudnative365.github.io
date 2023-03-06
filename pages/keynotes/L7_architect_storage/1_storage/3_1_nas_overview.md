---
title: NAS概述
keywords: keynotes, architecture, storage, nas, nas_overview
permalink: keynotes_L7_architect_storage_1_storage_3_1_nas_overview.html
sidebar: keynotes_L7_architect_storage_sidebar
typora-copy-images-to: ./pics/3_1_minio_cluster
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. kubernetes存储卷

我们前面学过了kubernetes存储卷相关的内容，其中一个概念是RWO，ROM，RWM

| Volume Plugin        |     ReadWriteOnce     |     ReadOnlyMany      |                        ReadWriteMany                         | ReadWriteOncePod      |
| :------------------- | :-------------------: | :-------------------: | :----------------------------------------------------------: | --------------------- |
| AWSElasticBlockStore |           ✓           |           -           |                              -                               | -                     |
| AzureFile            |           ✓           |           ✓           |                              ✓                               | -                     |
| AzureDisk            |           ✓           |           -           |                              -                               | -                     |
| CephFS               |           ✓           |           ✓           |                              ✓                               | -                     |
| Cinder               |           ✓           |           -           | ([if multi-attach volumes are available](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/features.md#multi-attach-volumes)) | -                     |
| CSI                  | depends on the driver | depends on the driver |                    depends on the driver                     | depends on the driver |
| FC                   |           ✓           |           ✓           |                              -                               | -                     |
| FlexVolume           |           ✓           |           ✓           |                    depends on the driver                     | -                     |
| GCEPersistentDisk    |           ✓           |           ✓           |                              -                               | -                     |
| Glusterfs            |           ✓           |           ✓           |                              ✓                               | -                     |
| HostPath             |           ✓           |           -           |                              -                               | -                     |
| iSCSI                |           ✓           |           ✓           |                              -                               | -                     |
| NFS                  |           ✓           |           ✓           |                              ✓                               | -                     |
| RBD                  |           ✓           |           ✓           |                              -                               | -                     |
| VsphereVolume        |           ✓           |           -           |              - (works when Pods are collocated)              | -                     |
| PortworxVolume       |           ✓           |           -           |                              ✓                               | -                     |

从上面我们可以看到，私有云中比较常见的基于网络的RWM的解决方案只有ClusterFS和NFS。对于这些，我们统一叫他们NAS（网络存储）。对于ClusterFS来说，他天生支持高可用，而NFS并没有什么稳定的开源集群解决方案。但是由于他的广泛性和配置简单等，我们还是经常拿他来做k8s的持久存储。

这里值得一提的是另外一些很好的免费使用的产品，比如已经贡献给CNCF的OpenEBS，Rancher公司的Longhorn等，这些都是非常不错的解决方案。

## 2. NFS

NFS本身并不支持多节点协同工作，也就是说他并不具备天然的高可用。为了让NFS高可用，我们就需要借助其他的体系，比如pacemaker和drbd之类来模仿高可用。但是模仿出来的高可用，最多就是主备模式，如果心跳机制检测到主节点挂了，就把服务切到备节点。这种方式本身并不是非常稳定，如果是高并发的场景是必然会有数据丢失或者是性能瓶颈的。

但是由于NFS的广泛和易用，简单场景来做是非常方便的。并且由于是RWM的，所以经常用来做一些小型项目或者非大并发的场景。

kubernetes直接使用nfs是没有问题的，如果想动态的申请nfs空间，也就是说通过pv和pvc的方式来使用的话，就需要一个工具，确切的说是一个provisioner。由于他的代码在[github](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)上被多次挪动，所以我们通常叫他nfs-provisioner。

``` bash
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/exported/path
```

## 3. Glusterfs

Glusterfs应该算是比较早期的分布式存储了，但是国内使用的并不多，国内主要还是使用nfs，要么就是对象存储。Glusterfs的有点在于他的稳定性，国外的圈子用的比较多

## 4. PortworxVolume

这也是kubernetes官方推荐的方案之一，算是后期之秀，但是距离老牌的产品距离还是有的

## 5. OpenEBS

OpenEBS本身是基于iSCSI协议来做的，但是国内的广泛程度远不如Rook

## 6. 公有云厂商

AAG对于RWO，ROM，RWM都有对应的方案，国内的除了阿里云还算靠谱，其他的一定要慎重。

## 7. 传统存储厂商

如果选择传统存储厂商来做的话，一定要综合考虑。

+ 存储本身的支持
+ 虚拟化层的支持，我就见到过RHV平台无法透传FC协议实现存储RWM导致架构高可用上的缺陷
+ kubernetes以及CSI对于上面的支持

## 8. NAS性能：NFS，Samba和GlusterFS

简而言之：对于小型文件写入，Samba的速度比NFS和GlusterFS快得多。

- GlusterFS复制2：**32-35秒**，高CPU负载
- GlusterFS单：**14-16秒**，高CPU负载
- GlusterFS + NFS客户端：**16-19秒**，高CPU负载
- NFS内核服务器+ NFS客户端（同步）：**32-36秒**，非常低的CPU负载
- NFS内核服务器+ NFS客户端（异步）：**3-4秒**，非常低的CPU负载
- Samba：**4到7秒**，中等CPU负载
- 直接磁盘：**<1**秒
