---
title: MinIO概述
keywords: keynotes, architecture, storage, object_overview
permalink: keynotes_L7_architect_storage_1_storage_2_1_minio_overview.html
sidebar: keynotes_L7_architect_storage_sidebar
typora-copy-images-to: ./pics/2_1_object_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. MinIO概述

MinIO 是一款高性能、分布式的对象存储系统. 它是一款软件产品, 可以100%的运行在标准硬件。即X86等低成本机器也能够很好的运行MinIO。

MinIO与传统的存储和其他的对象存储不同的是：它一开始就针对性能要求更高的私有云标准进行软件架构设计。因为MinIO一开始就只为对象存储而设计。所以他采用了更易用的方式进行设计，它能实现对象存储所需要的全部功能，在性能上也更加强劲，它不会为了更多的业务功能而妥协，失去MinIO的易用性、高效性。 这样的结果所带来的好处是：它能够更简单的实现局有弹性伸缩能力的原生对象存储服务。

MinIO在传统对象存储用例（例如辅助存储，灾难恢复和归档）方面表现出色。同时，它在机器学习、大数据、私有云、混合云等方面的存储技术上也独树一帜。当然，也不排除数据分析、高性能应用负载、原生云的支持。

在中国：阿里巴巴、腾讯、百度、中国联通、华为、中国移动等等9000多家企业也都在使用MinIO产品

### 1.1. 高性能

![High Performance](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/high-performance.svg)

MinIO 是全球领先的对象存储先锋，目前在全世界有数百万的用户. 在标准硬件上，读/写速度上高达183 GB / 秒 和 171 GB / 秒。
对象存储可以充当主存储层，以处理Spark、Presto、TensorFlow、H2O.ai等各种复杂工作负载以及成为Hadoop HDFS的替代品。
MinIO用作云原生应用程序的主要存储，与传统对象存储相比，云原生应用程序需要更高的吞吐量和更低的延迟。而这些都是MinIO能够达成的性能指标。

### 1.2. 可扩展性

![Scalability](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/scalability.svg)

MinIO利用了Web缩放器的来之不易的知识，为对象存储带来了简单的缩放模型。 这是我们坚定的理念 “简单可扩展.” 在 MinIO, 扩展从单个群集开始，该群集可以与其他MinIO群集联合以创建全局名称空间, 并在需要时可以跨越多个不同的数据中心。 通过添加更多集群可以扩展名称空间, 更多机架，直到实现目标。

### 1.3. 云原生支持

![Cloud Native](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/cloud-native.svg)

MinIO 是在过去4年的时间内从0开始打造的一款软件 ，符合一切原生云计算的架构和构建过程，并且包含最新的云计算的全新的技术和概念。 其中包括支持Kubernetes 、微服和多租户的的容器技术。使对象存储对于 Kubernetes更加友好。

### 1.4. 开放全部源代码 + 企业级支持

![Open Source + Enterprise Grade](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/open-source-enterprise.svg)

MinIO 基于Apache V2 license 100% 开放源代码 。 这就意味着 MinIO的客户能够自动的、无限制、自由免费使用和集成MinIO、自由的创新和创造、 自由的去修改、自由的再次发行新的版本和软件. 确实, MinIO 强有力的支持和驱动了很多世界500强的企业。 此外，其部署的多样性和专业性提供了其他软件无法比拟的优势。

### 1.5. 与Amazon S3 兼容

![Amazon S3 Compatibility](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/s3-compatibility.svg)

亚马逊云的 S3 API（接口协议） 是在全球范围内达到共识的对象存储的协议，是全世界内大家都认可的标准。 MinIO 在很早的时候就采用了 S3 兼容协议，并且MinIO 是第一个支持 S3 Select 的产品. MinIO对其兼容性的全面性感到自豪， 并且得到了 750多个组织的认同, 包括Microsoft Azure使用MinIO的S3网关 - 这一指标超过其他同类产品的总和。

### 1.6. 简单

![img](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/simplicity.gif)

极简主义是MinIO的指导性设计原则。简单性减少了出错的机会，提高了正常运行时间，提供了可靠性，同时简单性又是性能的基础。 只需下载一个二进制文件然后执行，即可在几分钟内安装和配置MinIO。 配置选项和变体的数量保持在最低限度，这样让失败的配置概率降低到接近于0的水平。 MinIO升级是通过一个简单命令完成的，这个命令可以无中断的完成MinIO的升级，并且不需要停机即可完成升级操作 - 降低总使用和运维成本。

## 2. 社区

### 2.1. Git

Git地址：https://github.com/minio/minio。遵循[AGPL-3.0 license](https://github.com/minio/minio/blob/master/LICENSE)。从2016年2月8日第一个Release到现在2023年1月30日总共Stars：37.2K，Forks：4.4K，release了大小381个版本。可以说社区是非常活跃的。

### 2.2. 网站

国际站：https://min.io/。国内站：https://www.minio.org.cn/。国内站的很多东西是从国际站机器翻译过来的，而且会有版本的延迟。

## 3. 初体验

在1.6章那个动态gif其实已经把过程给大家了，这里咱们再重现一次

``` bash
wget   http://dl.minio.org.cn/server/minio/release/linux-amd64/minio
chmod +x minio
mv minio /usr/local/sbin

mkdir /data/minio
chmod -R 775 /data/minio
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=Passw0rd
export MINIO_KMS_SECRET_KEY=my-minio-encryption-key:bXltaW5pb2VuY3J5cHRpb25rZXljaGFuZ2VtZTEyMwo=

minio server /data/minio
```

然后我们就可以登陆web界面来进行管理了，http://你的IP:9000。用户名是$MINIO_ROOT_USER，密码是$MINIO_ROOT_PASSWORD。

![image-20230130093530752](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/image-20230130093530752.png)

登陆后就可以看到一些我们熟悉的概念了，比如bucket，AK，IAM policy之类，我们可以试着创建桶并且给与特定用户一些权限，这些和AWS的S3完全兼容且一致。

## 4. 总结

### 4.1. 管理方式

MinIO除了拥有图形界面，还有一个命令行工具叫mc。界面上可以做的，命令行工具完全可以实现，界面上不能做的，命令行也可以做。

``` bash
wget    http://dl.minio.org.cn/client/mc/release/linux-amd64/mc
chmod +x mc
mv mc /usr/local/sbin
```

当然，对象存储最重要的功能之一就是要支持程序直接访问，SDK目前支持Java, Go, Node.js, Python, .NET, Haskell

### 4.2. 常用的功能

从图形界面上，我们可以看到还有很多常用的功能

![image-20230130094419245](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/image-20230130094419245.png)

我们企业级别能用到的功能，我们后面会说的，比如：

+ Site Replication，这个是集群功能，分片，高可用之类的都是集群功能。
+ Identify，管理用户，组和他们的权限，并且可以和其他认证系统，比如ldap之类的集成。
+ Notifications，这个是报警功能。
+ Tier，可以把其他的对象存储通过MinIO进行整合。

### 4.3. systemd管理

为了方便我们后面的实验，我们用systemd来对MinIO服务进行管理，修改`/etc/systemd/system/minio.service`文件

``` bash
[Unit]
Description=MinIO
Documentation=https://min.io/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/sbin/minio

[Service]
WorkingDirectory=/usr/local

User=minio
Group=minio
ProtectProc=invisible

EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/sbin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target

# Built for ${project.name}-${project.version} (${project.name})
```

为了使用minio用户启动服务，我们还需要创建minio用户并且给文件夹对应的权限

``` bash
useradd minio
chown minio:minio -R /data/minio
```

我们的配置文件用环境变量文件/etc/default/minio来存储

``` bash
# MINIO_ROOT_USER and MINIO_ROOT_PASSWORD sets the root account for the MinIO server.
# This user has unrestricted permissions to perform S3 and administrative API operations on any resource in the deployment.
# Omit to use the default values 'minioadmin:minioadmin'.
# MinIO recommends setting non-default values as a best practice, regardless of environment

MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=Passw0rd

# MINIO_VOLUMES sets the storage volume or path to use for the MinIO server.

MINIO_VOLUMES="/data/minio"

# MINIO_SERVER_URL sets the hostname of the local machine for use with the MinIO Server
# MinIO assumes your network control plane can correctly resolve this hostname to the local machine

# Uncomment the following line and replace the value with the correct hostname for the local machine.

#MINIO_SERVER_URL="http://minio.example.net"

# Set all MinIO server options
# #
# # The following explicitly sets the MinIO Console listen address to
# # port 9001 on all network interfaces. The default behavior is dynamic
# # port selection.
#
MINIO_OPTS="--console-address 10.39.64.234:9001"
```

