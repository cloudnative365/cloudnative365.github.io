---
title: NAS概述
keywords: keynotes, architecture, storage, object_storage, nas_overview
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

但是由于NFS的广泛和易用，简单场景来做是非常方便的。并且由于是RWM的，所以经常用来做一些小型项目或者非大并发的场景
