---
title: 存储概述
keywords: keynotes, architecture, storage, overview
permalink: keynotes_L7_architect_storage_1_storage_0_storage_overview.html
sidebar: keynotes_L7_architect_sidebar
typora-copy-images-to: ./pics/0_storage_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 存储

我第一次基础计算机的时候，老师给我们讲计算机的组成，包括：控制器，运算器，存储器，输入设备，输出设备。存储器有ram和rom，RAM(random acess memory) 是随机读写设备，断电数据消失，也就是我们通常说的内存。ROM(read only memory) 是只读设备，断电数据不消失，比如磁盘，U盘。

画外音：这些概念都是来自于科学家冯诺依曼提出的理论，所以叫冯诺依曼架构，也叫普林斯顿结构。最早的电脑埃尼阿克就是遵照冯诺依曼架构来设计的。控制器，运算器我们统称为CPU，比较知名的CPU是intel的8086，我们也叫他X86。从第一台计算机诞生到现在，尽管计算机的体积已经变得非常小了，但是他的架构都是遵照冯诺依曼体系来生产的。无一例外！但是，目前正在研制的量子计算机就不是这个架构了！因为量子计算机的核心并不是晶体管了，这就会彻底推翻传统硬件行业，希望我们的国家可以在这一领域有所突破，实现弯道超车，站在世界的顶端。

磁盘是对于存储的一种抽象，文件系统是对于磁盘的一种抽象。我们平时所用的xfs，dfs是文件系统，更准确的说是文件管理系统，他们把磁盘抽象成文件，通过mount的形式映射给操作系统，操作系统通过系统调用来将数据写入这些介质中，从而实现持久化存储。

+ 在虚拟化时代，我们把磁盘虚拟成一个文件映射给虚拟机，从虚拟机看来，他使用的就是一块磁盘。

+ 在docker时代，我们使用-v选项来把文件夹映射给docker容器，docker会把内容直接写入系统文件夹。
+ 到了容器编排时代，那些大量创建和被销毁的容器对于存储的要求显然需要把数据存储在某个特定的位置，即使容器被销毁，数据还是可以长久保存的。对于编排来说，如果没有特殊配置，同一个pod显然不可能永远被调度到一个worker节点，使用docker那种文件映射的方式显然不可以，我们就需要一个解耦过的存储空间，比如：对象存储

## 2. 存储的类型

### 2.1. 文件存储，块存储，对象存储

这类的概念网上一查一大堆，我们没必要在这里赘述，我直接上结论。

+ 文件存储: 通常意义是支持POSIX接口，它跟传统的文件系统如ext4和xfs是一个类型的，但区别在于分布式存储提供了并行化的能力

+ 块存储: 这种接口通常以QEMU Driver或者Kernel Module的方式存在，这种接口需要实现Linux的Block Device的接口或者QEMU提供的Block Driver接口，如Sheepdog，AWS的EBS，青云的云硬盘和阿里云的盘古系统，还有Ceph的RBD(RBD是Ceph面向块存储的接口)

+ 对象存储: 也就是通常意义的键值存储，其接口就是简单的GET,PUT,DEL和其他扩展，如七牛、又拍，Swift，S3

  ![img](/pages/keynotes/L7_architect_storage/1_storage/pics/0_storage_overview/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcnkwNzEy,size_16,color_FFFFFF,t_70.png)

### 2.2. kubernetes持久存储

对于kubernetes来说，存储功能并不是kubernetes来提供的！kubernetes提供的是一个CSI(Container Storage Interface容器存储接口)，能够和这个接口兼容的都可以作为kubernetes的持久存储。

既然是接口，那么他就有个API（storage.k8s.io/v1），这个时候，最合适的存储就是对象存储，因为对象存储本身就是通过API来调用。当然，如果想使用块存储或者文件存储也可以，我们需要额外开发一个控制器，也就是开发CRD来实现。

### 2.3. kubernetes存储类型

对于kubernetes来说，从是否可以同时读写的角度，分为四类，但是常用的有三类，ReadWriteOnce，ReadOnly和ReadWriteMany

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

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

对于私有云来说，能同时读写的只有Glusterfs，NFS这两种和CephFS。当然，公有云是个例外，他可以直接提供对象存储服务，比如S3，OSS，OBS，这类的都是通过CSI来实现的，SC中的**provisioner**: kubernetes.io/glusterfs，就是指定存储来源的。

## 3. CNCF的存储项目

### 3.1. 常用的存储类型

上面我们介绍了存储的类型，对于我们日常在DC中常用的类型有

+ HostPath：简单直接，把文件存储到磁盘，但是如果pod被调度到另外的机器，数据就会找不到了
+ NFS：简单好配置，多读多写能满足大部分的存储使用场景，但是在生产系统上速度拉胯，如果没有靠谱的解决方案非常容易出事故
+ GlusterFS：和NFS类似，在生产系统上速度拉胯，国内用的比较少
+ CephFS：目前是红帽的项目，可以买订阅来获得官方支持，但是稳定性比硬件存储还是差很多，因为Ceph目前稳定性和硬件存储比起来还是有差距的

### 3.2. 存储管理类的项目

+ CNCF毕业的存储项目只有Rook一个，rook主要是对于ceph的封装。

+ rancher的longhorn和新上的kubeFS目前稳定性还是有待验证，建议大家慎重使用。

+ OpenEBS是对于NAS的支持，比如nfs和clusterFS，说实话，其实使用nfs-provisioner可能更方便一点。

+ MinIO是DC环境的S3，目前也是非常火热，轻量且好用，扩展性也很好，也是我比较推荐的方案。

rook和minio就是我们要详细讲解的项目了
