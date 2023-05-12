---
title: MinIO概述
keywords: keynotes, architecture, storage, object_overview
permalink: keynotes_L7_architect_storage_1_storage_2_1_minio_overview.html
sidebar: keynotes_L7_architect_storage_sidebar
typora-copy-images-to: ./pics/2_1_object_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. S3协议

### 1.1. 什么是S3

```
Amazon Simple Storage Service（Amazon S3）是一种对象存储服务，提供行业领先的可扩展性、数据可用性、安全性和性能。各种规模和行业的客户都可以使用 Amazon S3 存储和保护任意数量的数据，用于数据湖、网站、移动应用程序、备份和恢复、归档、企业应用程序、IoT 设备和大数据分析。Amazon S3 提供了管理功能，使您可以优化、组织和配置对数据的访问，以满足您的特定业务、组织和合规性要求。
--https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/Welcome.html
```

上面是AWS的官方介绍，我们可以看出，S3实际上是AWS的一个SaaS的存储服务，AWS提供存储类，存储管理，访问控制，数据处理，存储日志记录和监控等等功能来让这个服务更加强大。

### 2.1. AWS S3的功能

#### 存储类

Amazon S3 提供一系列适合不同使用案例的存储类。例如，您可以将任务关键型生产数据存储在 S3 Standard 中以便频繁访问，通过将不经常访问的数据存储在 S3 Standard-IA 或 S3 One Zone-IA 中以最低的成本归档数据来节省成本，并在 S3 Glacier Instant Retrieval、S3 Glacier Flexible Retrieval、和 S3 Glacier Deep Archive 中以最低的成本归档数据。

您可以在 S3 Intelligent-Tiering 中存储具有不断变化或未知访问模式的数据，该分层可在访问模式发生变化时自动在四个访问层之间移动数据，从而优化存储成本。这四个访问层包括两个低延迟访问层（针对频繁和不频繁访问进行了优化），以及两个为异步访问很少访问的数据而设计的 opt-in archive 访问层。

有关更多信息，请参阅[使用 Amazon S3 存储类](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/storage-class-intro.html)。有关 S3 Glacier Flexible Retrieval 的更多信息，请参阅 [*Amazon S3 Glacier 开发人员指南*](https://docs.aws.amazon.com/amazonglacier/latest/dev/introduction.html)。

#### 存储管理

Amazon S3 具有存储管理功能，您可以使用这些功能来管理成本、满足法规要求、减少延迟并保存数据的多个不同副本以满足合规性要求。

- [S3 生命周期](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html) - 配置生命周期策略以管理您的对象，并在其整个生命周期内经济高效地存储。您可以将对象转换为其他 S3 存储类，也可以使其生命周期结束的对象过期。
- [S3 对象锁定](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html) - 可以在固定的时间段内或无限期地阻止删除或覆盖 Amazon S3 对象。可以使用对象锁定来帮助您满足需要*一次写入多次读取* *（WORM）* 存储的法规要求，或只是添加另一个保护层来防止对象被更改和删除。
- [S3 复制](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html) - 将对象及其各自的元数据和对象标签复制到同一或不同的 AWS 区域 目标存储桶中的一个或多个目标存储桶，以减少延迟、合规性、安全性和其他使用案例。
- [S3 分批操作](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html) - 通过单个 S3 API 请求或在 Amazon S3 控制台中单击几次，大规模管理数十亿个对象。您可以使用分批操作来执行诸如**复制**、**调用 AWS Lambda 函数**, 和**恢复**数百万或数十亿对象。

#### 访问管理

Amazon S3 提供了用于审核和管理对存储桶和数据元的访问的功能。默认情况下，S3 存储桶和对象都是私有的。您只能访问您创建的 S3 资源。要授予支持您特定使用案例的细粒度资源权限或审核 Amazon S3 资源的权限，您可以使用以下功能。

- [S3 阻止公有访问](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html) - 阻止对 S3 存储桶和对象的公有访问。默认情况下，在账户和桶级别打开 “阻止公共访问” 设置。
- [AWS Identity and Access Management（IAM）](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-access-control.html)– IAM 是一种 Web 服务，可帮助您安全地控制对 AWS 资源（包括 Amazon S3 资源）的访问。借助 IAM，您可以集中管理控制用户可访问哪些 AWS 资源的权限。可以使用 IAM 来控制谁通过了身份验证（准许登录）并获得授权（拥有权限）来使用资源。
- [桶策略](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html) - 使用基于 IAM 的策略语言为 S3 桶及其中的对象配置基于资源的权限。
- [Amazon S3 访问点](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points.html) – 使用专用访问策略配置命名网络终端节点，以便大规模管理对 Amazon S3 中共享数据集的访问。
- [访问控制列表（ACL）](https://docs.aws.amazon.com/AmazonS3/latest/userguide/acls.html)- 向授权用户授予单个桶和对象的读写权限。作为一般规则，我们建议您使用基于 S3 资源的策略（桶策略和接入点策略）或 IAM 策略进行访问控制，而不是 ACL。ACL 是一种访问控制机制，早于基于资源的策略和 IAM。有关何时使用 ACL 而不是基于资源的策略或 IAM 策略的更多信息，请参阅[访问策略指南](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/access-policy-alternatives-guidelines.html).
- [S3 Object Ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)（S3 对象所有权）— 禁用 ACL 并获取桶中每个对象的所有权，从而简化了对存储在 Amazon S3 中的数据的访问管理。作为桶所有者，您自动拥有并完全控制桶中的每个对象，并且数据的访问控制是基于策略而进行。
- [S3 访问分析器](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-analyzer.html) - 评估和监控您的 S3 桶访问策略，确保这些策略仅提供对 S3 资源的预期访问。

#### 数据处理

要转换数据并触发工作流以大规模自动执行各种其他处理活动，您可以使用以下功能。

- [S3 Object Lambda](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transforming-objects.html) - 您可以将自己的代码添加到 S3 GET、HEAD 和 LIST 请求中，以便在数据返回到应用程序时修改和处理数据。筛选行、动态调整图像大小、编辑机密数据等。
- [事件通知](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html) - 当您的 S3 资源进行更改时，触发使用 Amazon Simple Notification Service (Amazon SNS)、Amazon Simple Queue Service (Amazon SQS) 和的工作流程和 AWS Lambda。

#### 存储日志记录和监控

Amazon S3 提供日志记录和监控工具，您可以使用这些工具来监控和控制 Amazon S3 资源的使用情况。更多信息，请参阅[监控工具](https://docs.aws.amazon.com/AmazonS3/latest/userguide/monitoring-automated-manual.html)。

###### 自动监控工具

- [Amazon S3 的 Amazon CloudWatch 指标](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cloudwatch-monitoring.html)- 跟踪 S3 资源的运行状况，并在估计费用达到用户定义的阈值时配置计费警报。
- [AWS CloudTrail](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cloudtrail-logging.html) - 在 Amazon S3 中记录用户采取的行动、角色或 AWS 服务。CloudTrail 日志为您提供了 S3 存储桶级别和对象级操作的详细 API 跟踪。

###### 手动监控工具

- [服务器访问日志](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html)- 详细地记录对存储桶提出的各种请求。您可以使用服务器访问日志对许多使用案例进行安全和访问审计，了解客户群或了解您的 Amazon S3 账单。
- [AWS Trusted Advisor](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html) - 通过使用 AWS 最佳实践检查以确定优化 AWS 基础架构、提高安全性和性能、降低成本以及监控服务配额。然后，您可以按照建议优化服务和资源。

#### 分析和见解

Amazon S3 提供的功能可帮助您了解存储使用情况，从而使您能够更好地了解、分析和大规模优化存储。

- [Amazon S3 Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html) - 了解、分析和优化您的存储。S3 Storage Lens 提供了超过 29 个使用率和活动指标以及交互式仪表板，用于汇总整个组织、特定客户、AWS 区域、存储桶或前缀。
- [存储类分析](https://docs.aws.amazon.com/AmazonS3/latest/userguide/analytics-storage-class.html)- 分析存储访问模式，以决定何时需要将数据移动到更经济高效的存储类别。
- [带清单报告的 S3 清单](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html)- 审核和报告对象及其相应的元数据，并配置其他 Amazon S3 功能，在清单报告中采取措施。例如，您可以报告对象的复制和加密状态。有关清单报告中每个对象可用的所有元数据的列表，请参阅 [Amazon S3 清单](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/storage-inventory.html#storage-inventory-contents)。

#### 强一致性

Amazon S3 为所有 AWS 区域 中的 Amazon S3 存储桶中对象的 PUT 和 DELETE 提供了强大的先写后读一致性。这个行为既适用于到新对象的写入，也适用于覆盖现有对象的 PUT 和 DELETE。此外，针对 Amazon S3 Select、Amazon S3 访问控制列表 (ACL)、Amazon S3 对象标签和对象元数据（例如 HEAD 对象）的读取操作具有严格一致性。有关更多信息，请参阅[Amazon S3 数据一致性模型](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)。

### 1.3. S3的局限性

诚然，S3非常强大，他的可靠度高达99.995%，是非常好用的存储服务。但是，他只限于公有云环境，如果我们私有云环境想使用S3，就需要通过外网接口，专线或者VPN的方式来访问S3服务。

![image-20230310234730456](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/image-20230310234730456.png)

在实际的生产当中，我们我们也会有不能上公有云的场景，我们就需要在数据中心部署兼容S3的存储服务。

+ 使用可以提供兼容S3服务的硬件存储

  使用硬件大部分时候会比软件更稳定，而且效率更高。如果我们购买了一些兼容S3的硬件存储，比如EMC（EXF900以及更高级的），NetApp（ONTAP 9.12.1之后的）

+ 使用软件提供兼容S3的服务

  常见的两种，一种是Ceph-S3，另外一种就是目前非常流行的MinIO

## 2. MinIO概述

MinIO 是一款高性能、分布式的对象存储系统. 它是一款软件产品, 可以100%的运行在标准硬件。即X86等低成本机器也能够很好的运行MinIO。

MinIO与传统的存储和其他的对象存储不同的是：它一开始就针对性能要求更高的私有云标准进行软件架构设计。因为MinIO一开始就只为对象存储而设计。所以他采用了更易用的方式进行设计，它能实现对象存储所需要的全部功能，在性能上也更加强劲，它不会为了更多的业务功能而妥协，失去MinIO的易用性、高效性。 这样的结果所带来的好处是：它能够更简单的实现局有弹性伸缩能力的原生对象存储服务。

MinIO在传统对象存储用例（例如辅助存储，灾难恢复和归档）方面表现出色。同时，它在机器学习、大数据、私有云、混合云等方面的存储技术上也独树一帜。当然，也不排除数据分析、高性能应用负载、原生云的支持。

在中国：阿里巴巴、腾讯、百度、中国联通、华为、中国移动等等9000多家企业也都在使用MinIO产品

### 2.1. 高性能

![High Performance](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/high-performance.svg)

MinIO 是全球领先的对象存储先锋，目前在全世界有数百万的用户. 在标准硬件上，读/写速度上高达183 GB / 秒 和 171 GB / 秒。
对象存储可以充当主存储层，以处理Spark、Presto、TensorFlow、H2O.ai等各种复杂工作负载以及成为Hadoop HDFS的替代品。
MinIO用作云原生应用程序的主要存储，与传统对象存储相比，云原生应用程序需要更高的吞吐量和更低的延迟。而这些都是MinIO能够达成的性能指标。

### 2.2. 可扩展性

![Scalability](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/scalability.svg)

MinIO利用了Web缩放器的来之不易的知识，为对象存储带来了简单的缩放模型。 这是我们坚定的理念 “简单可扩展.” 在 MinIO, 扩展从单个群集开始，该群集可以与其他MinIO群集联合以创建全局名称空间, 并在需要时可以跨越多个不同的数据中心。 通过添加更多集群可以扩展名称空间, 更多机架，直到实现目标。

### 2.3. 云原生支持

![Cloud Native](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/cloud-native.svg)

MinIO 是在过去4年的时间内从0开始打造的一款软件 ，符合一切原生云计算的架构和构建过程，并且包含最新的云计算的全新的技术和概念。 其中包括支持Kubernetes 、微服和多租户的的容器技术。使对象存储对于 Kubernetes更加友好。

### 2.4. 开放全部源代码 + 企业级支持

![Open Source + Enterprise Grade](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/open-source-enterprise.svg)

MinIO 基于Apache V2 license 100% 开放源代码 。 这就意味着 MinIO的客户能够自动的、无限制、自由免费使用和集成MinIO、自由的创新和创造、 自由的去修改、自由的再次发行新的版本和软件. 确实, MinIO 强有力的支持和驱动了很多世界500强的企业。 此外，其部署的多样性和专业性提供了其他软件无法比拟的优势。

### 2.5. 与Amazon S3 兼容

![Amazon S3 Compatibility](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/s3-compatibility.svg)

亚马逊云的 S3 API（接口协议） 是在全球范围内达到共识的对象存储的协议，是全世界内大家都认可的标准。 MinIO 在很早的时候就采用了 S3 兼容协议，并且MinIO 是第一个支持 S3 Select 的产品. MinIO对其兼容性的全面性感到自豪， 并且得到了 750多个组织的认同, 包括Microsoft Azure使用MinIO的S3网关 - 这一指标超过其他同类产品的总和。

### 2.6. 简单

![img](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/simplicity.gif)

极简主义是MinIO的指导性设计原则。简单性减少了出错的机会，提高了正常运行时间，提供了可靠性，同时简单性又是性能的基础。 只需下载一个二进制文件然后执行，即可在几分钟内安装和配置MinIO。 配置选项和变体的数量保持在最低限度，这样让失败的配置概率降低到接近于0的水平。 MinIO升级是通过一个简单命令完成的，这个命令可以无中断的完成MinIO的升级，并且不需要停机即可完成升级操作 - 降低总使用和运维成本。

## 2. 社区

### 3.1. Git

Git地址：https://github.com/minio/minio。遵循[AGPL-3.0 license](https://github.com/minio/minio/blob/master/LICENSE)。从2016年2月8日第一个Release到现在2023年1月30日总共Stars：37.2K，Forks：4.4K，release了大小381个版本。可以说社区是非常活跃的。

### 3.2. 网站

国际站：https://min.io/。国内站：https://www.minio.org.cn/。国内站的很多东西是从国际站机器翻译过来的，而且会有版本的延迟。

## 4. 初体验

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

## 5. 总结

### 5.1. 管理方式

MinIO除了拥有图形界面，还有一个命令行工具叫mc。界面上可以做的，命令行工具完全可以实现，界面上不能做的，命令行也可以做。

``` bash
wget    http://dl.minio.org.cn/client/mc/release/linux-amd64/mc
chmod +x mc
mv mc /usr/local/sbin
```

当然，对象存储最重要的功能之一就是要支持程序直接访问，SDK目前支持Java, Go, Node.js, Python, .NET, Haskell

### 5.2. 常用的功能

从图形界面上，我们可以看到还有很多常用的功能

![image-20230130094419245](/pages/keynotes/L7_architect_storage/1_storage/pics/2_1_object_overview/image-20230130094419245.png)

我们企业级别能用到的功能，我们后面会说的，比如：

+ Site Replication，这个是集群功能，分片，高可用之类的都是集群功能。
+ Identify，管理用户，组和他们的权限，并且可以和其他认证系统，比如ldap之类的集成。
+ Notifications，这个是报警功能。
+ Tier，可以把其他的对象存储通过MinIO进行整合。

### 5.3. systemd管理

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

