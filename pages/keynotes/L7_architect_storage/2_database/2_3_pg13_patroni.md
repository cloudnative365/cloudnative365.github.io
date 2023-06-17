---
title: patroni
keywords: keynotes, architect, storage, database, postgresql
permalink: keynotes_L7_architect_storage_2_database_2_3_postgresql_13_HA.html
sidebar: keynotes_L7_architect_sidebar
typora-copy-images-to: ./pics/2_3_postgresql_13_HA
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

截止到2023年4月7日，目前pgsql的最新版本已经到15了。对于高可用的方案又有很多完善且好用的方案，今天我们来介绍的，就是基于pigsty的partoni方案

### 1.1. 特性

Pigsty 是一个更好的本地开源 RDS for PostgreSQL 替代：

- 开箱即用的 [PostgreSQL](https://www.postgresql.org/) 发行版，深度整合地理时序分布式向量等核心扩展： [PostGIS](https://postgis.net/), [TimescaleDB](https://www.timescale.com/), [Citus](https://www.citusdata.com/)，[PGVector](https://github.com/pgvector/pgvector)……
- 基于现代的 [Prometheus](https://prometheus.io/) 与 [Grafana](https://grafana.com/) 技术栈，提供令人惊艳，无可比拟的数据库观测能力：[演示站点](http://demo.pigsty.cc/)
- 基于 [patroni](https://patroni.readthedocs.io/en/latest/), [haproxy](http://www.haproxy.org/), 与[etcd](https://etcd.io/)，打造故障自愈的高可用架构：硬件故障自动切换，流量无缝衔接。
- 基于 [pgBackRest](https://pgbackrest.org/) 与可选的 [MinIO](https://min.io/) 集群提供开箱即用的 PITR 时间点恢复，为软件缺陷与人为删库兜底。
- 基于 [Ansible](https://www.ansible.com/) 提供声明式的 API 对复杂度进行抽象，以 **Database-as-Code** 的方式极大简化了日常运维管理操作。
- Pigsty用途广泛，可用作完整应用运行时，开发演示数据/可视化应用，大量使用 PG 的软件可用 [Docker](https://www.docker.com/) 模板一键拉起。
- 提供基于 [Vagrant](https://www.vagrantup.com/) 的本地开发测试沙箱环境，与基于 [Terraform](https://www.terraform.io/) 的云端自动部署方案，开发测试生产保持环境一致。

### 1.2. 介绍

**彻底释放世界上最先进的关系型数据库的力量!**

PostgreSQL 是一个足够完美的数据库内核，但它需要更多工具与系统的配合，才能成为一个足够好的数据库服务（RDS），而 Pigsty 帮助 PostgreSQL 完成这一步飞跃。

Pigsty 深度整合 PostgreSQL 生态的核心扩展插件：您可以使用 **PostGIS** 处理地理空间数据，使用 **TimescaleDB** 分析时序/事件流数据，使用 **Citus** 原地进行分布式水平扩展，使用 **PGVector** 存储并搜索 AI Embedding，以及其他海量扩展插件。 Pigsty 确保这些插件可以协同工作，提供开箱即用的分布式的时序地理空间向量数据库能力。此外，Pigsty 还提供了运行企业级 RDS 服务的所需软件，打包所有依赖为离线软件包，所有组件均可在无需互联网访问的情况下一键完成安装部署，进入生产可用状态。

在 Pigsty 中功能组件被抽象 [**模块**](https://github.com/Vonng/pigsty/blob/master/docs/ARCH#modules)，可以自由组合以应对多变的需求场景。[`INFRA`](https://github.com/Vonng/pigsty/blob/master/docs/INFRA.md) 模块带有完整的现代监控技术栈，而 [`NODE`](https://github.com/Vonng/pigsty/blob/master/docs/NODE.md) 模块则将节点调谐至指定状态并纳入监控。 在多个节点上安装 [`PGSQL`](https://github.com/Vonng/pigsty/blob/master/docs/PGSQL.md) 模块会自动组建出一个基于主从复制的高可用数据库集群，而同样的 [`ETCD`](https://github.com/Vonng/pigsty/blob/master/docs/ETCD.md) 模块则为数据库高可用提供共识与元数据存储。可选的 [`MINIO`](https://github.com/Vonng/pigsty/blob/master/docs/MINIO.md)模块可以用作图像视频等大文件存储并可选用为数据库备份仓库。 与 PG 有着极佳相性的 [`REDIS`](https://github.com/Vonng/pigsty/blob/master/docs/REDIS.md) 亦为 Pigsty 所支持，更多的模块（如`GPSQL`, `MYSQL`, `KAFKA`）将会在后续加入，你也可以开发自己的模块并自行扩展 Pigsty 的能力。

![pigsty-distro](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/226076217-77e76e0c-94ac-4faa-9014-877b4a180e09.jpg)

### 1.3. 可观测性

**使用现代开源可观测性技术栈，提供无与伦比的监控最佳实践！**

Pigsty 提供了基于开源的 Grafana / Prometheus 可观测性技术栈做监控的最佳实践：Prometheus 用于收集监控指标，Grafana 负责可视化呈现，Loki 用于日志收集与查询，Alertmanager 用于告警通知。 PushGateway 用于批处理任务监控，Blackbox Exporter 负责检查服务可用性。整套系统同样被设计为一键拉起，开箱即用的 INFRA 模块。

Pigsty 所管理的任何组件都会被自动纳入监控之中，包括主机节点，负载均衡 HAProxy，数据库 Postgres，连接池 Pgbouncer，元数据库 ETCD，KV缓存 Redis，对象存储 MinIO，……，以及整套监控基础设施本身。大量的 Grafana 监控面板与预置告警规则会让你的系统观测能力有质的提升，当然，这套系统也可以被复用于您的应用监控基础设施，或者监控已有的数据库实例或 RDS。

无论是故障分析还是慢查询优化、无论是水位评估还是资源规划，Pigsty 为您提供全面的数据支撑，真正做到数据驱动。在 Pigsty 中，超过三千类监控指标被用于描述整个系统的方方面面，并被进一步加工、聚合、处理、分析、提炼并以符合直觉的可视化模式呈现在您的面前。从全局大盘总揽，到某个数据库实例中单个对象（表，索引，函数）的增删改查详情都能一览无余。您可以随意上卷下钻横向跳转，浏览系统现状与历史趋势，并预测未来的演变。详见[公开演示](http://demo.pigsty.cc/)。

![pigsty-dashboard](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/198838834-1bd30b7e-47c9-4e35-90cb-5a75a2e6f6c6.jpg)

### 1.4. 久经考验的可靠性

**开箱即用的高可用与时间点恢复能力，确保你的数据库坚如磐石！**

对于软件缺陷或人为误操作造成的删表删库，Pigsty 提供了开箱即用的 PITR 时间点恢复能力，无需额外配置即默认启用。只要存储空间管够，基于 `pgBackRest` 的基础备份与 WAL 归档让您拥有快速回到过去任意时间点的能力。您可以使用本地目录/磁盘，亦或专用的 MinIO 集群或 S3 对象存储服务保留更长的回溯期限，丰俭由人。

更重要的是，Pigsty 让高可用与故障自愈成为 PostgreSQL 集群的标配，基于 `patroni`, `etcd`, 与 `haproxy` 打造的故障自愈架构，让您在面对硬件故障时游刃有余：主库故障自动切换的 RTO < 30s，一致性优先模式下确保数据零损失 RPO = 0。只要集群中有任意实例存活，集群就可以对外提供完整的服务，而客户端只要连接至集群中的任意节点，即可获得完整的服务。

Pigsty 内置了 HAProxy 负载均衡器用于自动流量切换，提供 DNS/VIP/LVS 等多种接入方式供客户端选用。故障切换与主动切换对业务侧除零星闪断外几乎无感知，应用不需要修改连接串重启。极小的维护窗口需求带来了极大的灵活便利：您完全可以在无需应用配合的情况下滚动维护升级整个集群。硬件故障可以等到第二天再抽空善后处置的特性，让研发，运维与 DBA 都能安心睡个好觉。 许多大型组织与核心机构已经在生产环境中长时间使用 Pigsty ，最大的部署有 25K CPU 核心与 200+ PostgreSQL 实例，在这一部署案例中， Pigsty 在三年内经历了数十次硬件故障与各类事故，但依然可以保持 99.999% 以上的整体可用性。

![pigsty-ha](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/206971583-74293d7b-d29a-4ca2-8728-75d50421c371.gif)

### 1.5. 简单易用可维护

**Infra as Code, 数据库即代码，声明式的API将数据库管理的复杂度来封装。**

Pigsty 使用声明式的接口对外提供服务，将系统的可控制性拔高到一个全新水平：用户通过配置清单告诉 Pigsty "我想要什么样的数据库集群"，而不用去操心到底需要怎样去做。从效果上讲，这类似于 K8S 中的 CRD 与 Operator，但 Pigsty 可用于任何节点上的数据库与基础设施：不论是容器，虚拟机，还是物理机。

无论是创建/销毁集群，添加/移除从库，还是新增数据库/用户/服务/扩展/黑白名单规则，您只需要修改配置清单并运行 Pigsty 提供的幂等剧本，而 Pigsty 负责将系统调整到您期望的状态。 用户无需操心配置的细节，Pigsty将自动根据机器的硬件配置进行调优，您只需要关心诸如集群叫什么名字，有几个实例放在哪几台机器上，使用什么配置模版：事务/分析/核心/微型，这些基础信息，研发也可以自助服务。但如果您愿意跳入兔子洞中，Pigsty 也提供了丰富且精细的控制参数，满足最龟毛 DBA 的苛刻定制需求。

除此之外，Pigsty 本身的安装部署也是一键傻瓜式的，所有依赖被预先打包，在安装时可以无需互联网访问。而安装所需的机器资源，也可以通过 Vagrant 或 Terraform 模板自动获取，让您在十几分钟内就可以从零在本地笔记本或云端虚拟机上拉起一套完整的 Pigsty 部署。本地沙箱环境可以跑在1核2G的微型虚拟机中，提供与生产环境完全一致的功能模拟，可以用于开发、测试、演示与学习。

![pigsty-iac](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/206972039-e13746ab-72ae-4cab-8de7-7b2ef543f3e5.gif)

### 1.6. 扎实的安全性

**加密备份一应俱全，只要硬件与密钥安全，您无需操心数据库的安全性。**

每套 Pigsty 部署都会创建一套自签名的 CA 用于证书签发，所有的网络通信都可以使用 SSL 加密。数据库密码使用合规的 `scram-sha-256` 算法加密存储，远端备份会使用 `AES-256` 算法加密。此外还针对 PGSQL 提供了一套开箱即用的的访问控制体系，足以应对绝大多数应用场景下的安全需求。

Pigsty 针对 PostgreSQL 提供了一套开箱即用，简单易用，精炼灵活的，便于扩展的[访问控制体系](https://github.com/Vonng/pigsty/blob/master/docs/PGSQL-ACL.md)，包括职能分离的四类默认角色：读(DQL) / 写(DML) / 管理(DDL) / 离线(ETL) ，与四个默认用户：dbsu / replicator / monitor / admin。 所有数据库模板都针对这些角色与用户配置有合理的默认权限，而任何新建的数据库对象也会自动遵循这套权限体系，而客户端的访问则受到一套基于最小权限原则的设计的 HBA 规则组限制，任何敏感操作都会记入日志审计。

任何网络通信都可以使用 SSL 加密，需要保护的敏感管理页面与API端点都受到多重保护：使用用户名与密码进行认证，限制从管理节点/基础设施节点IP地址/网段访问，要求使用 HTTPS 加密网络流量。Patroni API 与 Pgbouncer 因为性能因素默认不启用 SSL ，但亦提供安全开关便于您在需要时开启。 合理配置的系统通过等保三级毫无问题，只要您遵循安全性最佳实践，内网部署并合理配置安全组与防火墙，数据库安全性将不再是您的痛点。

![pigsty-dashboard2](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/198838841-b0796703-03c3-483b-bf52-dbef9ea10913.gif)

### 1.7. 广泛的应用场景

**使用预置的Docker模板，一键拉起使用PostgreSQL的海量软件！**

在各类数据密集型应用中，数据库往往是最为棘手的部分。例如 Gitlab 企业版与社区版的核心区别就是底层 PostgreSQL 数据库的监控与高可用，如果您已经有了足够好的本地 PG RDS，又为什么要用软件自带的土法手造组件掏钱？

Pigsty 提供了 Docker 模块与大量开箱即用的 Compose 模板。您可以使用 Pigsty 管理的高可用 PostgreSQL （以及 Redis 与 MinIO ）作为后端存储，以无状态的模式一键拉起这些软件： Gitlab、Gitea、Wiki.js、Odoo、Jira、Confluence、Habour、Mastodon、Discourse、KeyCloak 等等。如果您的应用需要一个靠谱的 PostgreSQL 数据库， Pigsty 也许是最简单的获取方案。

Pigsty 也提供了与 PostgreSQL 紧密联系的应用开发工具集：PGAdmin4、PGWeb、ByteBase、PostgREST、Kong、以及 EdgeDB、FerretDB、Supabase 这些使用 PostgreSQL 作为存储的"上层数据库"。更奇妙的是，您完全可以基于 Pigsty 内置了的 Grafana 与 Postgres ，以低代码的方式快速搭建起一个交互式的数据应用来，甚至还可以使用 Pigsty 内置的 ECharts 面板创造更有表现力的交互可视化作品。

![pigsty-app](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/198838829-f0ea4af2-d33f-4978-a31a-ed81897aa8d1.gif)

### 1.8. 开源免费的自由软件

**Pigsty是基于 AGPLv3 开源的自由软件，由热爱 PostgreSQL 的社区成员用热情浇灌**

Pigsty 是完全开源免费的自由软件，它允许您在缺乏数据库专家的情况下，用几乎接近纯硬件的成本来运行企业级的 PostgreSQL 数据库服务。作为对比，公有云厂商提供的 RDS 会收取底层硬件资源几倍到十几倍不等的溢价作为 "服务费"。

很多用户选择上云，正是因为自己搞不定数据库；很多用户使用 RDS，是因为别无他选。我们将打破云厂商的垄断，为用户提供一个云中立的，更好的 RDS 开源替代： Pigsty 紧跟 PostgreSQL 上游主干，不会有供应商锁定，不会有恼人的 "授权费"，不会有节点数量限制，不会收集您的任何数据。您的所有的核心资产 —— 数据，都能"自主可控"，掌握在自己手中。

Pigsty 本身旨在用数据库自动驾驶软件，替代大量无趣的人肉数据库运维工作，但再好的软件也没法解决所有的问题。总会有一些的冷门低频疑难杂症需要专家介入处理。这也是为什么我们也提供专业的订阅服务，来为有需要的企业级用户使用 PostgreSQL 提供兜底。几万块的订阅咨询费不到顶尖 DBA 每年工资的几十分之一，让您彻底免除后顾之忧，把成本真正花在刀刃上。当然对于社区用户，我们亦用爱发电，提供免费的支持与日常答疑。

![pigsty-rds-cost](https://user-images.githubusercontent.com/8587410/225852971-577be00f-b2df-427c-a590-f8b4c5a63a4b.png)

## 2. 一键部署生产级别的高可用集群

### 2.1. 简单Demo

项目的第一层目录中包含了一个叫pigsty的目录，默认使用了4台机器来创建集群，我们第一次可以创建一个10.10.10.0/24的网络，比如：10.0.0.0/8的VPC中包含一个10.10.10.0/24的子网，创建4台机器并且指定他们的IP地址，从10.10.10.10-10.10.10.13，并且预留2个VIP。

在10.10.10.10上配置10.10.10.10可以免密ssh登录到10.10.10.10-10.10.10.13这4台机器。然后下载整个工程

``` bash
bash -c "$(curl -fsSL http://download.pigsty.cc/get)";
cd ~/pigsty; ./bootstrap; ./configure; ./install.yml;
```

这样就可以简单实现一个demo环境了，视频可以看这个https://asciinema.org/a/568771

### 2.2. 分析Demo环境

整个的环境配置都是由下面的文件来配置的，也就是说，最后一步的install.yml是读取下面的文件来进行配置的，而前面的bootstrap和configure都是读取其他配置，从而生成pigsty.yml. 因此，我们想要创建环境，并不需要bootstrap和configure，直接手动修改pigsty.yml就好了。下面我们看一下这个文件。

``` yml
---
#==============================================================#
# File      :   pigsty.yml
# Desc      :   Pigsty Local Sandbox 4-node Demo Config
# Ctime     :   2020-05-22
# Mtime     :   2023-03-31
# Docs      :   https://vonng.github.io/pigsty/#/CONFIG
# Author    :   Ruohang Feng (rh@vonng.com)
# License   :   AGPLv3
#==============================================================#


all:
  children:

    # infra cluster for proxy, monitor, alert, etc..
    infra: { hosts: { 10.10.10.10: { infra_seq: 1 } }}

    # minio cluster, s3 compatible object storage
    minio: { hosts: { 10.10.10.10: { minio_seq: 1 } }, vars: { minio_cluster: minio } }

    # etcd cluster for ha postgres
    etcd: { hosts: { 10.10.10.10: { etcd_seq: 1 } }, vars: { etcd_cluster: etcd } }

    # postgres example cluster: pg-meta
    pg-meta:
      hosts: { 10.10.10.10: { pg_seq: 1, pg_role: primary } }
      vars:
        pg_cluster: pg-meta
        pg_users:
          - {name: dbuser_meta     ,password: DBUser.Meta     ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: pigsty admin user }
          - {name: dbuser_view     ,password: DBUser.Viewer   ,pgbouncer: true ,roles: [dbrole_readonly] ,comment: read-only viewer for meta database }
          - {name: dbuser_grafana  ,password: DBUser.Grafana  ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for grafana database    }
          - {name: dbuser_bytebase ,password: DBUser.Bytebase ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for bytebase database   }
          - {name: dbuser_kong     ,password: DBUser.Kong     ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for kong api gateway    }
          - {name: dbuser_gitea    ,password: DBUser.Gitea    ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for gitea service       }
          - {name: dbuser_wiki     ,password: DBUser.Wiki     ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for wiki.js service     }
        pg_databases:
          - { name: meta ,baseline: cmdb.sql ,comment: pigsty meta database ,schemas: [pigsty] ,extensions: [{name: postgis, schema: public}, {name: timescaledb}]}
          - { name: grafana  ,owner: dbuser_grafana  ,revokeconn: true ,comment: grafana primary database }
          - { name: bytebase ,owner: dbuser_bytebase ,revokeconn: true ,comment: bytebase primary database }
          - { name: kong     ,owner: dbuser_kong     ,revokeconn: true ,comment: kong the api gateway database }
          - { name: gitea    ,owner: dbuser_gitea    ,revokeconn: true ,comment: gitea meta database }
          - { name: wiki     ,owner: dbuser_wiki     ,revokeconn: true ,comment: wiki meta database }
        pg_hba_rules:
          - {user: dbuser_view , db: all ,addr: infra ,auth: pwd ,title: 'allow grafana dashboard access cmdb from infra nodes'}
        pg_vip_enabled: true
        pg_vip_address: 10.10.10.2/24
        pg_vip_interface: eth1
        node_crontab:  # make a full backup 1 am everyday
          - '00 01 * * * postgres /pg/bin/pg-backup full'

    # pgsql 3 node ha cluster: pg-test
    pg-test:
      hosts:
        10.10.10.11: { pg_seq: 1, pg_role: primary }   # primary instance, leader of cluster
        10.10.10.12: { pg_seq: 2, pg_role: replica }   # replica instance, follower of leader
        10.10.10.13: { pg_seq: 3, pg_role: replica, pg_offline_query: true } # replica with offline access
      vars:
        pg_cluster: pg-test           # define pgsql cluster name
        pg_users:  [{ name: test , password: test , pgbouncer: true , roles: [ dbrole_admin ] }]
        pg_databases: [{ name: test }]
        pg_vip_enabled: true
        pg_vip_address: 10.10.10.3/24
        pg_vip_interface: eth1
        node_tune: tiny
        pg_conf: tiny.yml
        node_crontab:  # make a full backup on monday 1am, and an incremental backup during weekdays
          - '00 01 * * 1 postgres /pg/bin/pg-backup full'
          - '00 01 * * 2,3,4,5,6,7 postgres /pg/bin/pg-backup'

    redis-ms: # redis classic primary & replica
      hosts: { 10.10.10.10: { redis_node: 1 , redis_instances: { 6501: { }, 6502: { replica_of: '10.10.10.10 6501' } } } }
      vars: { redis_cluster: redis-ms ,redis_password: 'redis.ms' ,redis_max_memory: 64MB }

    redis-meta: # redis sentinel x 3
      hosts: { 10.10.10.11: { redis_node: 1 , redis_instances: { 6001: { } ,6002: { } , 6003: { } } } }
      vars: { redis_cluster: redis-meta ,redis_password: 'redis.meta' ,redis_mode: sentinel ,redis_max_memory: 16MB }

    redis-test: # redis native cluster: 3m x 3s
      hosts:
        10.10.10.12: { redis_node: 1 ,redis_instances: { 6501: { } ,6502: { } ,6503: { } } }
        10.10.10.13: { redis_node: 2 ,redis_instances: { 6501: { } ,6502: { } ,6503: { } } }
      vars: { redis_cluster: redis-test ,redis_password: 'redis.test' ,redis_mode: cluster, redis_max_memory: 32MB }


  vars:                               # global variables
    version: v2.0.2                   # pigsty version string
    admin_ip: 10.10.10.10             # admin node ip address
    region: default                   # upstream mirror region: default|china|europe
    node_tune: tiny                   # use tiny template for NODE  in demo environment
    pg_conf: tiny.yml                 # use tiny template for PGSQL in demo environment
    infra_portal:                     # domain names and upstream servers
      home         : { domain: h.pigsty }
      grafana      : { domain: g.pigsty ,endpoint: "${admin_ip}:3000" , websocket: true }
      prometheus   : { domain: p.pigsty ,endpoint: "${admin_ip}:9090" }
      alertmanager : { domain: a.pigsty ,endpoint: "${admin_ip}:9093" }
      blackbox     : { endpoint: "${admin_ip}:9115" }
      loki         : { endpoint: "${admin_ip}:3100" }
      minio        : { domain: sss.pigsty  ,endpoint: "${admin_ip}:9001" ,scheme: https ,websocket: true }
      postgrest    : { domain: api.pigsty  ,endpoint: "127.0.0.1:8884" }
      pgadmin      : { domain: adm.pigsty  ,endpoint: "127.0.0.1:8885" }
      pgweb        : { domain: cli.pigsty  ,endpoint: "127.0.0.1:8886" }
      bytebase     : { domain: ddl.pigsty  ,endpoint: "127.0.0.1:8887" }
      gitea        : { domain: git.pigsty  ,endpoint: "127.0.0.1:8889" }
      wiki         : { domain: wiki.pigsty ,endpoint: "127.0.0.1:9002" }
    nginx_navbar:                    # application nav links on home page
      - { name: PgAdmin4   , url : 'http://adm.pigsty'  , comment: 'PgAdmin4 for PostgreSQL'  }
      - { name: PGWeb      , url : 'http://cli.pigsty'  , comment: 'PGWEB Browser Client'     }
      - { name: ByteBase   , url : 'http://ddl.pigsty'  , comment: 'ByteBase Schema Migrator' }
      - { name: PostgREST  , url : 'http://api.pigsty'  , comment: 'Kong API Gateway'         }
      - { name: Gitea      , url : 'http://git.pigsty'  , comment: 'Gitea Git Service'        }
      - { name: Minio      , url : 'http://sss.pigsty'  , comment: 'Minio Object Storage'     }
      - { name: Wiki       , url : 'http://wiki.pigsty' , comment: 'Local Wikipedia'          }
      - { name: Explain    , url : '/pigsty/pev.html'   , comment: 'pgsql explain visualizer' }
      - { name: Package    , url : '/pigsty'            , comment: 'local yum repo packages'  }
      - { name: PG Logs    , url : '/logs'              , comment: 'postgres raw csv logs'    }
      - { name: Schemas    , url : '/schema'            , comment: 'schemaspy summary report' }
      - { name: Reports    , url : '/report'            , comment: 'pgbadger summary report'  }
    node_timezone: Asia/Hong_Kong     # use Asia/Hong_Kong Timezone
    node_ntp_servers:                 # NTP servers in /etc/chrony.conf
      - pool cn.pool.ntp.org iburst
      - pool ${admin_ip} iburst       # assume non-admin nodes does not have internet access
    pgbackrest_method: minio          # pgbackrest repo method: local,minio,[user-defined...]
...
```

下面是各个组件的架构

![pigsty-sandbox](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/218279650-5d5e8b09-8907-42bf-a48c-4c28bcc73ddd.jpg)

我们对照着各个组件的功能来看

| 组件名称 | 软件名称                                                     | 主机名 | IP地址      | VIP        |
| -------- | ------------------------------------------------------------ | ------ | ----------- | ---------- |
| PGSQL    | Patroni, Pgbouncer, HAproxy, PgBackrest                      | pgsql1 | 10.10.10.11 | 10.10.10.3 |
| PGSQL    | Patroni, Pgbouncer, HAproxy, PgBackrest                      | pgsql2 | 10.10.10.12 | 10.10.10.3 |
| PGSQL    | Patroni, Pgbouncer, HAproxy, PgBackrest                      | pgsql3 | 10.10.10.13 | 10.10.10.3 |
| INFRA    | Local yum repo, Prometheus, Grafana, Loki, AlertManager, PushGateway, Blackbox Exporter | infra  | 10.10.10.10 | 10.10.10.2 |
| NODE     | Tune node into the desired state, name, timezone, NTP, ssh, sudo, haproxy, docker, promtail. | node   | 10.10.10.10 | 10.10.10.2 |
| ETCD     | Distributed key-value store will be used as DCS for high-available Postgres clusters. | etcd   | 10.10.10.10 | 10.10.10.2 |
| REDIS    | Redis servers in standalone master-replica, sentinel, cluster mode with Redis exporter. | redis  | 10.10.10.10 | 10.10.10.2 |
| MINIO    | S3 compatible simple object storage server, can be used as an optional backup center for Postgres. | minio  | 10.10.10.10 | 10.10.10.2 |

我们来对照上面的信息分析一下：

+ PGSQL：1主1备1读，标准的生产环境最小配置，如果做读写分离或者需要在工作时间备份，我们可以适当增加读节点
  + Patroni, Pgbouncer：高可用的核心组件，用来切换主从
  + HAproxy：这个HAproxy是给VIP 10.10.10.3用的，用来对外提供服务
  + PgBackrest：备份工具
+ INFRA：上面是一些功能性的东西，企业中很可能会有其他集中管理的平台来提供服务，特别是有好多套PGSQL的时候，这些是可以复用的，没必要重复做很多套
  + Local yum repo：在我们执行bootstrap的时候，会下载一个叫做pigsty.tgz的包，里面就是rpm包，我们可以提前把这放到其他文件服务器上，我们可以通过手动指定yum源的方式，具体请参考[这个](https://vonng.github.io/pigsty/#/PARAM?id=repo)
  + Prometheus：监控数据库
  + Grafana：展示工具
  + Loki：日志数据库
  + AlertManager：报警工具
  + PushGateway：作为metric的跳板
  + Blackbox Exporter：TCP/HTTP的探针，并且暴露metrics
+ NODE：更多的是作为控制节点，来管理整个集群
  + NTP：NTP服务器，让整个集群来这里同步，我们可以在下面的配置中配置ntp服务器指向公司内部的地址
  + ssh：通过ssh登录到被管控的节点
  + haprxoy：作为管理节点的IP，所有的管理类域名都指向这个IP
  + docker：启动各种服务用的
  + promtail：日志采集Agent，给Loki传日志
+ ETCD：作为集群的仲裁，Partoni会把被管控的PGSQL状态同步到etcd中，从而防止脑裂，建议在生产系统中使用3个etcd组成集群，防止etcd单点故障，从而导致集群选举出现问题
+ REDIS：缓存服务器
+ MINIO：支持S3协议的对象存储，企业环境中一般也会提供，我们可以找管理员来申请一个bucket来给pgsql备份用
+ ntp：由于企业内部的机器不一定可以和外网相连，特别是数据库服务器，一般来说无法访问外网，因此一般的企业环境都会提供有ntp服务器，我们可以根据我们的需要修改`node_ntp_servers`选项
+ node_timezone：时区也别忘了改到自己的
+ region: default，可以改成China，提高下载速度
+ infra_portal和nginx_navbar：都是域名，建议大家提前规划好域名，方便我们创建服务

### 2.3. 生产环境

#### 2.3.1. 准备

我们假设在企业的环境中，有一些服务可以利用管理员提供给我们的，如ntp，dns，yum repo，file server with http，Prometheus, Grafana, Loki, AlertManager

这里要说一下的是yum repo和文件服务器

#### 2.3.2. 规划

为了让我们深入的研究每一个组件，我们对每一个组件尽量单独的配置一个机器，方便我们单独配置研究

| 组件名称 | 软件名称                                                     | 主机名 | IP地址      | VIP        |
| -------- | ------------------------------------------------------------ | ------ | ----------- | ---------- |
| PGSQL    | Patroni, Pgbouncer, HAproxy, PgBackrest                      | pgsql1 | 10.10.10.11 | 10.10.10.3 |
| PGSQL    | Patroni, Pgbouncer, HAproxy, PgBackrest                      | pgsql2 | 10.10.10.12 | 10.10.10.3 |
| PGSQL    | Patroni, Pgbouncer, HAproxy, PgBackrest                      | pgsql3 | 10.10.10.13 | 10.10.10.3 |
| INFRA    | Local yum repo, Prometheus, Grafana, Loki, AlertManager, PushGateway, Blackbox Exporter | infra  | 10.10.10.10 | 10.10.10.2 |
| NODE     | Tune node into the desired state, name, timezone, NTP, ssh, sudo, haproxy, docker, promtail. | node   | 10.10.10.10 | 10.10.10.2 |
| ETCD     | Distributed key-value store will be used as DCS for high-available Postgres clusters. | etcd   | 10.10.10.10 | 10.10.10.2 |
| REDIS    | Redis servers in standalone master-replica, sentinel, cluster mode with Redis exporter. | redis  | 10.10.10.10 | 10.10.10.2 |
| MINIO    | S3 compatible simple object storage server, can be used as an optional backup center for Postgres. | minio  | 10.10.10.10 | 10.10.10.2 |

对于每一个组件，都有非常多的参数，为了准确的对每个参数进行控制，我们可以先复制一份[完整的](https://github.com/Vonng/pigsty/blob/master/files/pigsty/full.yml)，再根据需要进行修改。我们根据这次的需求修改如下

``` yml
---
#==============================================================#
# File      :   full.yml
# Desc      :   pigsty demo config with full default values
# Ctime     :   2020-05-22
# Mtime     :   2023-03-31
# Docs      :   https://vonng.github.io/pigsty/#/CONFIG
# Author    :   Ruohang Feng (rh@vonng.com)
# License   :   AGPLv3
#==============================================================#


#==============================================================#
#                        Sandbox (4-node)                      #
#==============================================================#
# admin user : vagrant  (nopass ssh & sudo already set)        #
# 1.  meta    :    10.10.10.10     (2 Core | 4GB)    pg-meta   #
# 2.  node-1  :    10.10.10.11     (1 Core | 1GB)    pg-test-1 #
# 3.  node-2  :    10.10.10.12     (1 Core | 1GB)    pg-test-2 #
# 4.  node-3  :    10.10.10.13     (1 Core | 1GB)    pg-test-3 #
# (replace these ip if your 4-node env have different ip addr) #
# VIP 2: (l2 vip is available inside same LAN )                #
#     pg-meta --->  10.10.10.2 ---> 10.10.10.10                #
#     pg-test --->  10.10.10.3 ---> 10.10.10.1{1,2,3}          #
#==============================================================#


all:

  ##################################################################
  #                            CLUSTERS                            #
  ##################################################################
  # meta nodes, nodes, pgsql, redis, pgsql clusters are defined as
  # k:v pair inside `all.children`. Where the key is cluster name
  # and value is cluster definition consist of two parts:
  # `hosts`: cluster members ip and instance level variables
  # `vars` : cluster level variables
  ##################################################################
  children:                                 # groups definition

    # infra cluster for proxy, monitor, alert, etc..
    infra: { hosts: { 10.10.10.10: { infra_seq: 1 } } }

    # etcd cluster for ha postgres
    etcd: { hosts: { 10.10.10.10: { etcd_seq: 1 } }, vars: { etcd_cluster: etcd } }
    etcd: { hosts: { 10.10.10.10: { etcd_seq: 2 } }, vars: { etcd_cluster: etcd } }
    etcd: { hosts: { 10.10.10.10: { etcd_seq: 3 } }, vars: { etcd_cluster: etcd } }

    # minio cluster, s3 compatible object storage
    # minio: { hosts: { 10.10.10.10: { minio_seq: 1 } }, vars: { minio_cluster: minio } }

    #----------------------------------#
    # pgsql cluster: pg-meta (CMDB)    #
    #----------------------------------#
    pg-meta:
      hosts: { 10.10.10.10: { pg_seq: 1, pg_role: primary , pg_offline_query: true } }
      vars:
        pg_cluster: pg-meta
        pg_databases:                       # define business databases on this cluster, array of database definition
          - name: meta                      # REQUIRED, `name` is the only mandatory field of a database definition
            baseline: cmdb.sql              # optional, database sql baseline path, (relative path among ansible search path, e.g files/)
            pgbouncer: true                 # optional, add this database to pgbouncer database list? true by default
            schemas: [pigsty]               # optional, additional schemas to be created, array of schema names
            extensions:                     # optional, additional extensions to be installed: array of `{name[,schema]}`
              - { name: postgis , schema: public }
              - { name: timescaledb }
            comment: pigsty meta database   # optional, comment string for this database
            #owner: postgres                # optional, database owner, postgres by default
            #template: template1            # optional, which template to use, template1 by default
            #encoding: UTF8                 # optional, database encoding, UTF8 by default. (MUST same as template database)
            #locale: C                      # optional, database locale, C by default.  (MUST same as template database)
            #lc_collate: C                  # optional, database collate, C by default. (MUST same as template database)
            #lc_ctype: C                    # optional, database ctype, C by default.   (MUST same as template database)
            #tablespace: pg_default         # optional, default tablespace, 'pg_default' by default.
            #allowconn: true                # optional, allow connection, true by default. false will disable connect at all
            #revokeconn: false              # optional, revoke public connection privilege. false by default. (leave connect with grant option to owner)
            #register_datasource: true      # optional, register this database to grafana datasources? true by default
            #connlimit: -1                  # optional, database connection limit, default -1 disable limit
            #pool_auth_user: dbuser_meta    # optional, all connection to this pgbouncer database will be authenticated by this user
            #pool_mode: transaction         # optional, pgbouncer pool mode at database level, default transaction
            #pool_size: 64                  # optional, pgbouncer pool size at database level, default 64
            #pool_size_reserve: 32          # optional, pgbouncer pool size reserve at database level, default 32
            #pool_size_min: 0               # optional, pgbouncer pool size min at database level, default 0
            #pool_max_db_conn: 100          # optional, max database connections at database level, default 100
          #- { name: grafana  ,owner: dbuser_grafana  ,revokeconn: true ,comment: grafana primary database }
          #- { name: bytebase ,owner: dbuser_bytebase ,revokeconn: true ,comment: bytebase primary database }
          #- { name: kong     ,owner: dbuser_kong     ,revokeconn: true ,comment: kong the api gateway database }
          #- { name: gitea    ,owner: dbuser_gitea    ,revokeconn: true ,comment: gitea meta database }
          #- { name: wiki     ,owner: dbuser_wiki     ,revokeconn: true ,comment: wiki meta database }
        pg_users:                           # define business users/roles on this cluster, array of user definition
          - name: dbuser_meta               # REQUIRED, `name` is the only mandatory field of a user definition
            password: DBUser.Meta           # optional, password, can be a scram-sha-256 hash string or plain text
            login: true                     # optional, can log in, true by default  (new biz ROLE should be false)
            superuser: false                # optional, is superuser? false by default
            createdb: false                 # optional, can create database? false by default
            createrole: false               # optional, can create role? false by default
            inherit: true                   # optional, can this role use inherited privileges? true by default
            replication: false              # optional, can this role do replication? false by default
            bypassrls: false                # optional, can this role bypass row level security? false by default
            pgbouncer: true                 # optional, add this user to pgbouncer user-list? false by default (production user should be true explicitly)
            connlimit: -1                   # optional, user connection limit, default -1 disable limit
            expire_in: 3650                 # optional, now + n days when this role is expired (OVERWRITE expire_at)
            expire_at: '2030-12-31'         # optional, YYYY-MM-DD 'timestamp' when this role is expired  (OVERWRITTEN by expire_in)
            comment: pigsty admin user      # optional, comment string for this user/role
            roles: [dbrole_admin]           # optional, belonged roles. default roles are: dbrole_{admin,readonly,readwrite,offline}
            parameters: {}                  # optional, role level parameters with `ALTER ROLE SET`
            pool_mode: transaction          # optional, pgbouncer pool mode at user level, transaction by default
            pool_connlimit: -1              # optional, max database connections at user level, default -1 disable limit
          - {name: dbuser_view     ,password: DBUser.Viewer   ,pgbouncer: true ,roles: [dbrole_readonly], comment: read-only viewer for meta database}
          #- {name: dbuser_grafana  ,password: DBUser.Grafana  ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for grafana database   }
          #- {name: dbuser_bytebase ,password: DBUser.Bytebase ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for bytebase database  }
          #- {name: dbuser_gitea    ,password: DBUser.Gitea    ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for gitea service      }
          #- {name: dbuser_wiki     ,password: DBUser.Wiki     ,pgbouncer: true ,roles: [dbrole_admin]    ,comment: admin user for wiki.js service    }
        pg_services:                        # extra services in addition to pg_default_services, array of service definition
          # standby service will route {ip|name}:5435 to sync replica's pgbouncer (5435->6432 standby)
          - name: standby                   # required, service name, the actual svc name will be prefixed with `pg_cluster`, e.g: pg-meta-standby
            port: 5435                      # required, service exposed port (work as kubernetes service node port mode)
            ip: "*"                         # optional, service bind ip address, `*` for all ip by default
            selector: "[]"                  # required, service member selector, use JMESPath to filter inventory
            dest: default                   # optional, destination port, default|postgres|pgbouncer|<port_number>, 'default' by default
            check: /sync                    # optional, health check url path, / by default
            backup: "[? pg_role == `primary`]"  # backup server selector
            maxconn: 3000                   # optional, max allowed front-end connection
            balance: roundrobin             # optional, haproxy load balance algorithm (roundrobin by default, other: leastconn)
            options: 'inter 3s fastinter 1s downinter 5s rise 3 fall 3 on-marked-down shutdown-sessions slowstart 30s maxconn 3000 maxqueue 128 weight 100'
        pg_hba_rules:
          - {user: dbuser_view , db: all ,addr: infra ,auth: pwd ,title: 'allow grafana dashboard access cmdb from infra nodes'}
        pg_vip_enabled: true
        pg_vip_address: 10.10.10.2/24
        pg_vip_interface: eth1
        node_crontab:  # make a full backup 1 am everyday
          - '00 01 * * * postgres /pg/bin/pg-backup full'

    #----------------------------------#
    # pgsql cluster: pg-test (3 nodes) #
    #----------------------------------#
    # pg-test --->  10.10.10.3 ---> 10.10.10.1{1,2,3}
    pg-test:                          # define the new 3-node cluster pg-test
      hosts:
        10.10.10.11: { pg_seq: 1, pg_role: primary }   # primary instance, leader of cluster
        10.10.10.12: { pg_seq: 2, pg_role: replica }   # replica instance, follower of leader
        10.10.10.13: { pg_seq: 3, pg_role: replica, pg_offline_query: true } # replica with offline access
      vars:
        pg_cluster: pg-test           # define pgsql cluster name
        pg_users:  [{ name: test , password: test , pgbouncer: true , roles: [ dbrole_admin ] }]
        pg_databases: [{ name: test }] # create a database and user named 'test'
        node_tune: tiny
        pg_conf: tiny.yml
        pg_vip_enabled: true
        pg_vip_address: 10.10.10.3/24
        pg_vip_interface: eth1
        node_crontab:  # make a full backup on monday 1am, and an incremental backup during weekdays
          - '00 01 * * 1 postgres /pg/bin/pg-backup full'
          - '00 01 * * 2,3,4,5,6,7 postgres /pg/bin/pg-backup'

    #----------------------------------#
    # redis ms, sentinel, native cluster
    #----------------------------------#
    redis-ms: # redis classic primary & replica
      hosts: { 10.10.10.10: { redis_node: 1 , redis_instances: { 6501: { }, 6502: { replica_of: '10.10.10.10 6501' } } } }
      vars: { redis_cluster: redis-ms, redis_password: 'redis.ms' ,redis_max_memory: 64MB }

    redis-meta: # redis sentinel x 3
      hosts: { 10.10.10.11: { redis_node: 1 , redis_instances: { 6001: { } ,6002: { } , 6003: { } } } }
      vars: { redis_cluster: redis-meta, redis_password: 'redis.meta' ,redis_mode: sentinel ,redis_max_memory: 16MB }

    redis-test: # redis native cluster: 3m x 3s
      hosts:
        10.10.10.12: { redis_node: 1 ,redis_instances: { 6501: { } ,6502: { } ,6503: { } } }
        10.10.10.13: { redis_node: 2 ,redis_instances: { 6501: { } ,6502: { } ,6503: { } } }
      vars: { redis_cluster: redis-test ,redis_password: 'redis.test' ,redis_mode: cluster, redis_max_memory: 32MB }


  ####################################################################
  #                             VARS                                 #
  ####################################################################
  vars:                               # global variables


    #================================================================#
    #                         VARS: INFRA                            #
    #================================================================#

    #-----------------------------------------------------------------
    # META
    #-----------------------------------------------------------------
    version: v2.0.2                   # pigsty version string
    admin_ip: 10.10.10.10             # admin node ip address
    region: default                   # upstream mirror region: default,china,europe
    proxy_env:                        # global proxy env when downloading packages
      no_proxy: "localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,*.pigsty,*.aliyun.com,mirrors.*,*.myqcloud.com,*.tsinghua.edu.cn"
      # http_proxy:  # set your proxy here: e.g http://user:pass@proxy.xxx.com
      # https_proxy: # set your proxy here: e.g http://user:pass@proxy.xxx.com
      # all_proxy:   # set your proxy here: e.g http://user:pass@proxy.xxx.com

    #-----------------------------------------------------------------
    # CA
    #-----------------------------------------------------------------
    ca_method: create                 # create,recreate,copy, create by default
    ca_cn: pigsty-ca                  # ca common name, fixed as pigsty-ca
    cert_validity: 7300d              # cert validity, 20 years by default

    #-----------------------------------------------------------------
    # INFRA_IDENTITY
    #-----------------------------------------------------------------
    #infra_seq: 1                     # infra node identity, explicitly required
    infra_portal:                     # infra services exposed via portal
      home         : { domain: h.pigsty }
      grafana      : { domain: g.pigsty ,endpoint: "${admin_ip}:3000" ,websocket: true }
      prometheus   : { domain: p.pigsty ,endpoint: "${admin_ip}:9090" }
      alertmanager : { domain: a.pigsty ,endpoint: "${admin_ip}:9093" }
      blackbox     : { endpoint: "${admin_ip}:9115" }
      loki         : { endpoint: "${admin_ip}:3100" }

    #-----------------------------------------------------------------
    # REPO
    #-----------------------------------------------------------------
    repo_enabled: true                # create a yum repo on this infra node?
    repo_home: /www                   # repo home dir, `/www` by default
    repo_name: pigsty                 # repo name, pigsty by default
    repo_endpoint: http://${admin_ip}:80 # access point to this repo by domain or ip:port
    repo_remove: true                 # remove existing upstream repo
    repo_upstream:                    # where to download #
      - { name: base           ,description: 'EL 7 Base'         ,module: node  ,releases: [7    ] ,baseurl: { default: 'http://mirror.centos.org/centos/$releasever/os/$basearch/'                    , china: 'https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/'       , europe: 'https://mirrors.xtom.de/centos/$releasever/os/$basearch/'           }}
      - { name: updates        ,description: 'EL 7 Updates'      ,module: node  ,releases: [7    ] ,baseurl: { default: 'http://mirror.centos.org/centos/$releasever/updates/$basearch/'               , china: 'https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/'  , europe: 'https://mirrors.xtom.de/centos/$releasever/updates/$basearch/'      }}
      - { name: extras         ,description: 'EL 7 Extras'       ,module: node  ,releases: [7    ] ,baseurl: { default: 'http://mirror.centos.org/centos/$releasever/extras/$basearch/'                , china: 'https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/'   , europe: 'https://mirrors.xtom.de/centos/$releasever/extras/$basearch/'       }}
      - { name: epel           ,description: 'EL 7 EPEL'         ,module: node  ,releases: [7    ] ,baseurl: { default: 'http://download.fedoraproject.org/pub/epel/$releasever/$basearch/'            , china: 'https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/$basearch/'            , europe: 'https://mirrors.xtom.de/epel/$releasever/$basearch/'                }}
      - { name: centos-sclo    ,description: 'EL 7 SCLo'         ,module: node  ,releases: [7    ] ,baseurl: { default: 'http://mirror.centos.org/centos/$releasever/sclo/$basearch/sclo/'             , china: 'https://mirrors.aliyun.com/centos/$releasever/sclo/$basearch/sclo/'          , europe: 'https://mirrors.xtom.de/centos/$releasever/sclo/$basearch/sclo/'    }}
      - { name: centos-sclo-rh ,description: 'EL 7 SCLo rh'      ,module: node  ,releases: [7    ] ,baseurl: { default: 'http://mirror.centos.org/centos/$releasever/sclo/$basearch/rh/'               , china: 'https://mirrors.aliyun.com/centos/$releasever/sclo/$basearch/rh/'            , europe: 'https://mirrors.xtom.de/centos/$releasever/sclo/$basearch/rh/'      }}
      - { name: baseos         ,description: 'EL 8+ BaseOS'      ,module: node  ,releases: [  8,9] ,baseurl: { default: 'https://dl.rockylinux.org/pub/rocky/$releasever/BaseOS/$basearch/os/'         , china: 'https://mirrors.aliyun.com/rockylinux/$releasever/BaseOS/$basearch/os/'      , europe: 'https://mirrors.xtom.de/rocky/$releasever/BaseOS/$basearch/os/'     }}
      - { name: appstream      ,description: 'EL 8+ AppStream'   ,module: node  ,releases: [  8,9] ,baseurl: { default: 'https://dl.rockylinux.org/pub/rocky/$releasever/AppStream/$basearch/os/'      , china: 'https://mirrors.aliyun.com/rockylinux/$releasever/AppStream/$basearch/os/'   , europe: 'https://mirrors.xtom.de/rocky/$releasever/AppStream/$basearch/os/'  }}
      - { name: extras         ,description: 'EL 8+ Extras'      ,module: node  ,releases: [  8,9] ,baseurl: { default: 'https://dl.rockylinux.org/pub/rocky/$releasever/extras/$basearch/os/'         , china: 'https://mirrors.aliyun.com/rockylinux/$releasever/extras/$basearch/os/'      , europe: 'https://mirrors.xtom.de/rocky/$releasever/extras/$basearch/os/'     }}
      - { name: epel           ,description: 'EL 8+ EPEL'        ,module: node  ,releases: [  8,9] ,baseurl: { default: 'http://download.fedoraproject.org/pub/epel/$releasever/Everything/$basearch/' , china: 'https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/Everything/$basearch/' , europe: 'https://mirrors.xtom.de/epel/$releasever/Everything/$basearch/'     }}
      - { name: powertools     ,description: 'EL 8 PowerTools'   ,module: node  ,releases: [  8  ] ,baseurl: { default: 'https://dl.rockylinux.org/pub/rocky/$releasever/PowerTools/$basearch/os/'     , china: 'https://mirrors.aliyun.com/rockylinux/$releasever/PowerTools/$basearch/os/'  , europe: 'https://mirrors.xtom.de/rocky/$releasever/PowerTools/$basearch/os/' }}
      - { name: crb            ,description: 'EL 9 CRB'          ,module: node  ,releases: [    9] ,baseurl: { default: 'https://dl.rockylinux.org/pub/rocky/$releasever/CRB/$basearch/os/'            , china: 'https://mirrors.aliyun.com/rockylinux/$releasever/CRB/$basearch/os/'         , europe: 'https://mirrors.xtom.de/rocky/$releasever/CRB/$basearch/os/'        }}
      - { name: grafana        ,description: 'Grafana'           ,module: infra ,releases: [7,8,9] ,baseurl: { default: 'https://packages.grafana.com/oss/rpm'                                         , china: 'https://mirrors.tuna.tsinghua.edu.cn/grafana/yum/rpm' }}
      - { name: prometheus     ,description: 'Prometheus'        ,module: infra ,releases: [7,8,9] ,baseurl: { default: 'https://packagecloud.io/prometheus-rpm/release/el/$releasever/$basearch' }}
      - { name: nginx          ,description: 'Nginx Repo'        ,module: infra ,releases: [7,8,9] ,baseurl: { default: 'https://nginx.org/packages/centos/$releasever/$basearch/'                }}
      - { name: docker-ce      ,description: 'Docker CE'         ,module: infra ,releases: [7,8,9] ,baseurl: { default: 'https://download.docker.com/linux/centos/$releasever/$basearch/stable'                  , china: 'https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/$releasever/$basearch/stable'                     , europe: 'https://mirrors.xtom.de/docker-ce/linux/centos/$releasever/$basearch/stable'       }}
      - { name: pgdg15         ,description: 'PostgreSQL 15'     ,module: pgsql ,releases: [7,8,9] ,baseurl: { default: 'https://download.postgresql.org/pub/repos/yum/15/redhat/rhel-$releasever-$basearch'     , china: 'https://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/15/redhat/rhel-$releasever-$basearch'     , europe: 'https://mirrors.xtom.de/postgresql/repos/yum/15/redhat/rhel-$releasever-$basearch' }}
      - { name: pgdg-common    ,description: 'PostgreSQL Common' ,module: pgsql ,releases: [7,8,9] ,baseurl: { default: 'https://download.postgresql.org/pub/repos/yum/common/redhat/rhel-$releasever-$basearch' , china: 'https://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/common/redhat/rhel-$releasever-$basearch' , europe: 'https://mirrors.xtom.de/postgresql/repos/yum/common/redhat/rhel-$releasever-$basearch' }}
      - { name: pgdg-extras    ,description: 'PostgreSQL Extra'  ,module: pgsql ,releases: [7,8,9] ,baseurl: { default: 'https://download.postgresql.org/pub/repos/yum/common/pgdg-rhel$releasever-extras/redhat/rhel-$releasever-$basearch' , china: 'https://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/common/pgdg-rhel$releasever-extras/redhat/rhel-$releasever-$basearch' , europe: 'https://mirrors.xtom.de/postgresql/repos/yum/common/pgdg-rhel$releasever-extras/redhat/rhel-$releasever-$basearch' }}
      - { name: pgdg-el8fix    ,description: 'PostgreSQL EL8FIX' ,module: pgsql ,releases: [  8  ] ,baseurl: { default: 'https://download.postgresql.org/pub/repos/yum/common/pgdg-centos8-sysupdates/redhat/rhel-8-x86_64/' , china: 'https://mirrors.tuna.tsinghua.edu.cn/postgresql/repos/yum/common/pgdg-centos8-sysupdates/redhat/rhel-8-x86_64/' , europe: 'https://mirrors.xtom.de/postgresql/repos/yum/common/pgdg-centos8-sysupdates/redhat/rhel-8-x86_64/' }}
      - { name: timescaledb    ,description: 'TimescaleDB'       ,module: pgsql ,releases: [7,8,9] ,baseurl: { default: 'https://packagecloud.io/timescale/timescaledb/el/$releasever/$basearch'  }}
      - { name: citus          ,description: 'Citus Community'   ,module: pgsql ,releases: [7    ] ,baseurl: { default: 'https://repos.citusdata.com/community/el/$releasever/$basearch'          }}
    repo_packages:                    # which packages to be included
      - grafana loki logcli promtail prometheus2 alertmanager pushgateway blackbox_exporter node_exporter redis_exporter
      - nginx nginx_exporter wget createrepo_c sshpass ansible python3 python3-pip python3-requests python3-jmespath mtail dnsmasq docker-ce docker-compose-plugin etcd
      - lz4 unzip bzip2 zlib yum dnf-utils pv jq git ncdu make patch bash lsof wget uuid tuned chrony perf flamegraph nvme-cli numactl grubby sysstat iotop htop modulemd-tools
      - netcat socat rsync ftp lrzsz s3cmd net-tools tcpdump ipvsadm bind-utils telnet audit ca-certificates openssl openssh-clients readline vim-minimal haproxy redis
      - postgresql15* postgis33_15* citus_15* pglogical_15* pg_squeeze_15* wal2json_15* pg_repack_15* pgvector_15* timescaledb-2-postgresql-15* timescaledb-tools libuser openldap-compat
      - patroni patroni-etcd pgbouncer pgbadger pgbackrest tail_n_mail pgloader pg_activity
      - orafce_15* mysqlcompat_15 mongo_fdw_15* tds_fdw_15* mysql_fdw_15 hdfs_fdw_15 sqlite_fdw_15 pgbouncer_fdw_15 pg_dbms_job_15
      - pg_stat_kcache_15* pg_stat_monitor_15* pg_qualstats_15 pg_track_settings_15 pg_wait_sampling_15 system_stats_15 logerrors_15 pg_top_15
      - plprofiler_15* plproxy_15 plsh_15* pldebugger_15 plpgsql_check_15*  pgtt_15 pgq_15* pgsql_tweaks_15 count_distinct_15 hypopg_15
      - timestamp9_15* semver_15* prefix_15* rum_15 geoip_15 periods_15 ip4r_15 tdigest_15 hll_15 pgmp_15 extra_window_functions_15 topn_15
      - pg_comparator_15 pg_ivm_15* pgsodium_15* pgfincore_15* ddlx_15 credcheck_15 safeupdate_15
      - pg_fkpart_15 pg_jobmon_15 pg_partman_15 pg_permissions_15 pgaudit17_15 pgexportdoc_15 pgimportdoc_15 pg_statement_rollback_15*
      - pg_cron_15 pg_background_15 e-maj_15 pg_catcheck_15 pg_prioritize_15 pgcopydb_15 pg_filedump_15 pgcryptokey_15
    repo_url_packages:                # extra packages from url
      - https://github.com/Vonng/pg_exporter/releases/download/v0.5.0/pg_exporter-0.5.0.x86_64.rpm
      - https://github.com/cybertec-postgresql/vip-manager/releases/download/v2.1.0/vip-manager_2.1.0_Linux_x86_64.rpm
      - https://github.com/dalibo/pev2/releases/download/v1.8.0/index.html
      - https://dl.min.io/server/minio/release/linux-amd64/archive/minio-20230324214123.0.0.x86_64.rpm
      - https://dl.min.io/client/mc/release/linux-amd64/archive/mcli-20230323200304.0.0.x86_64.rpm

    #-----------------------------------------------------------------
    # INFRA_PACKAGE
    #-----------------------------------------------------------------
    infra_packages:                   # packages to be installed on infra nodes
      - grafana,loki,prometheus2,alertmanager,pushgateway,blackbox_exporter,nginx_exporter,redis_exporter,pg_exporter
      - nginx,ansible,python3-requests,redis,mcli,logcli,postgresql15
    infra_packages_pip: ''            # pip installed packages for infra nodes

    #-----------------------------------------------------------------
    # NGINX
    #-----------------------------------------------------------------
    nginx_enabled: true               # enable nginx on this infra node?
    nginx_sslmode: enable             # nginx ssl mode? disable,enable,enforce
    nginx_home: /www                  # nginx content dir, `/www` by default
    nginx_port: 80                    # nginx listen port, 80 by default
    nginx_ssl_port: 443               # nginx ssl listen port, 443 by default
    nginx_navbar:                     # nginx index page navigation links
      - { name: CA Cert ,url: '/ca.crt'   ,desc: 'pigsty self-signed ca.crt'   }
      - { name: Package ,url: '/pigsty'   ,desc: 'local yum repo packages'     }
      - { name: PG Logs ,url: '/logs'     ,desc: 'postgres raw csv logs'       }
      - { name: Reports ,url: '/report'   ,desc: 'pgbadger summary report'     }
      - { name: Explain ,url: '/pigsty/pev.html' ,desc: 'postgres explain visualizer' }

    #-----------------------------------------------------------------
    # DNS
    #-----------------------------------------------------------------
    dns_enabled: true                 # setup dnsmasq on this infra node?
    dns_port: 53                      # dns server listen port, 53 by default
    dns_records:                      # dynamic dns records resolved by dnsmasq
      - "${admin_ip} h.pigsty a.pigsty p.pigsty g.pigsty"
      - "${admin_ip} api.pigsty adm.pigsty cli.pigsty ddl.pigsty lab.pigsty git.pigsty sss.pigsty wiki.pigsty"

    #-----------------------------------------------------------------
    # PROMETHEUS
    #-----------------------------------------------------------------
    prometheus_enabled: true          # enable prometheus on this infra node?
    prometheus_clean: true            # clean prometheus data during init?
    prometheus_data: /data/prometheus # prometheus data dir, `/data/prometheus` by default
    prometheus_sd_interval: 5s        # prometheus target refresh interval, 5s by default
    prometheus_scrape_interval: 10s   # prometheus scrape & eval interval, 10s by default
    prometheus_scrape_timeout: 8s     # prometheus global scrape timeout, 8s by default
    prometheus_options: '--storage.tsdb.retention.time=15d' # prometheus extra server options
    pushgateway_enabled: true         # setup pushgateway on this infra node?
    pushgateway_options: '--persistence.interval=1m' # pushgateway extra server options
    blackbox_enabled: true            # setup blackbox_exporter on this infra node?
    blackbox_options: ''              # blackbox_exporter extra server options
    alertmanager_enabled: true        # setup alertmanager on this infra node?
    alertmanager_options: ''          # alertmanager extra server options
    exporter_metrics_path: /metrics   # exporter metric path, `/metrics` by default
    exporter_install: none            # how to install exporter? none,yum,binary
    exporter_repo_url: ''             # exporter repo file url if install exporter via yum

    #-----------------------------------------------------------------
    # GRAFANA
    #-----------------------------------------------------------------
    grafana_enabled: true             # enable grafana on this infra node?
    grafana_clean: true               # clean grafana data during init?
    grafana_admin_username: admin     # grafana admin username, `admin` by default
    grafana_admin_password: pigsty    # grafana admin password, `pigsty` by default
    grafana_plugin_cache: /www/pigsty/plugins.tgz # path to grafana plugins cache tarball
    grafana_plugin_list:              # grafana plugins to be downloaded with grafana-cli
      - volkovlabs-echarts-panel
      - marcusolsson-treemap-panel
    loki_enabled: true                # enable loki on this infra node?
    loki_clean: false                 # whether remove existing loki data?
    loki_data: /data/loki             # loki data dir, `/data/loki` by default
    loki_retention: 15d               # loki log retention period, 15d by default


    #================================================================#
    #                         VARS: NODE                             #
    #================================================================#

    #-----------------------------------------------------------------
    # NODE_IDENTITY
    #-----------------------------------------------------------------
    #nodename:           # [INSTANCE] # node instance identity, use hostname if missing, optional
    node_cluster: nodes   # [CLUSTER] # node cluster identity, use 'nodes' if missing, optional
    nodename_overwrite: true          # overwrite node's hostname with nodename?
    nodename_exchange: false          # exchange nodename among play hosts?
    node_id_from_pg: true             # use postgres identity as node identity if applicable?

    #-----------------------------------------------------------------
    # NODE_DNS
    #-----------------------------------------------------------------
    node_default_etc_hosts:           # static dns records in `/etc/hosts`
      - "${admin_ip} h.pigsty a.pigsty p.pigsty g.pigsty"
    node_etc_hosts: []                # extra static dns records in `/etc/hosts`
    node_dns_method: add              # how to handle dns servers: add,none,overwrite
    node_dns_servers: ['${admin_ip}'] # dynamic nameserver in `/etc/resolv.conf`
    node_dns_options:                 # dns resolv options in `/etc/resolv.conf`
      - options single-request-reopen timeout:1

    #-----------------------------------------------------------------
    # NODE_PACKAGE
    #-----------------------------------------------------------------
    node_repo_method: local           # how to setup node repo: none,local,public
    node_repo_remove: true            # remove existing repo on node?
    node_repo_local_urls:             # local repo url, if node_repo_method = local
      - http://${admin_ip}/pigsty.repo
    node_packages: [ ]                # packages to be installed current nodes
    node_default_packages:            # default packages to be installed on all nodes
      - lz4,unzip,bzip2,zlib,yum,pv,jq,git,ncdu,make,patch,bash,lsof,wget,uuid,tuned,chrony,perf,nvme-cli,numactl,grubby,sysstat,iotop,htop,yum,yum-utils
      - wget,netcat,socat,rsync,ftp,lrzsz,s3cmd,net-tools,tcpdump,ipvsadm,bind-utils,telnet,dnsmasq,audit,ca-certificates,openssl,openssh-clients,readline,vim-minimal
      - node_exporter,etcd,mtail,python3-idna,python3-requests,haproxy

    #-----------------------------------------------------------------
    # NODE_TUNE
    #-----------------------------------------------------------------
    node_disable_firewall: true       # disable node firewall? true by default
    node_disable_selinux: true        # disable node selinux? true by default
    node_disable_numa: false          # disable node numa, reboot required
    node_disable_swap: false          # disable node swap, use with caution
    node_static_network: true         # preserve dns resolver settings after reboot
    node_disk_prefetch: false         # setup disk prefetch on HDD to increase performance
    node_kernel_modules: [ softdog, br_netfilter, ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh ]
    node_hugepage_count: 0            # number of 2MB hugepage, take precedence over ratio
    node_hugepage_ratio: 0            # node mem hugepage ratio, 0 disable it by default
    node_overcommit_ratio: 0          # node mem overcommit ratio, 0 disable it by default
    node_tune: oltp                   # node tuned profile: none,oltp,olap,crit,tiny
    node_sysctl_params: { }           # sysctl parameters in k:v format in addition to tuned

    #-----------------------------------------------------------------
    # NODE_ADMIN
    #-----------------------------------------------------------------
    node_data: /data                  # node main data directory, `/data` by default
    node_admin_enabled: true          # create a admin user on target node?
    node_admin_uid: 88                # uid and gid for node admin user
    node_admin_username: dba          # name of node admin user, `dba` by default
    node_admin_ssh_exchange: true     # exchange admin ssh key among node cluster
    node_admin_pk_current: true       # add current user's ssh pk to admin authorized_keys
    node_admin_pk_list: []            # ssh public keys to be added to admin user

    #-----------------------------------------------------------------
    # NODE_TIME
    #-----------------------------------------------------------------
    node_timezone: ''                 # setup node timezone, empty string to skip
    node_ntp_enabled: true            # enable chronyd time sync service?
    node_ntp_servers:                 # ntp servers in `/etc/chrony.conf`
      - pool pool.ntp.org iburst
    node_crontab_overwrite: true      # overwrite or append to `/etc/crontab`?
    node_crontab: [ ]                 # crontab entries in `/etc/crontab`

    #-----------------------------------------------------------------
    # HAPROXY
    #-----------------------------------------------------------------
    haproxy_enabled: true             # enable haproxy on this node?
    haproxy_clean: false              # cleanup all existing haproxy config?
    haproxy_reload: true              # reload haproxy after config?
    haproxy_auth_enabled: true        # enable authentication for haproxy admin page
    haproxy_admin_username: admin     # haproxy admin username, `admin` by default
    haproxy_admin_password: pigsty    # haproxy admin password, `pigsty` by default
    haproxy_exporter_port: 9101       # haproxy admin/exporter port, 9101 by default
    haproxy_client_timeout: 24h       # client side connection timeout, 24h by default
    haproxy_server_timeout: 24h       # server side connection timeout, 24h by default
    haproxy_services: []              # list of haproxy service to be exposed on node

    #-----------------------------------------------------------------
    # NODE_EXPORTER
    #-----------------------------------------------------------------
    node_exporter_enabled: true       # setup node_exporter on this node?
    node_exporter_port: 9100          # node exporter listen port, 9100 by default
    node_exporter_options: '--no-collector.softnet --no-collector.nvme --collector.ntp --collector.tcpstat --collector.processes'

    #-----------------------------------------------------------------
    # PROMTAIL
    #-----------------------------------------------------------------
    promtail_enabled: true            # enable promtail logging collector?
    promtail_clean: false             # purge existing promtail status file during init?
    promtail_port: 9080               # promtail listen port, 9080 by default
    promtail_positions: /var/log/positions.yaml # promtail position status file path


    #================================================================#
    #                        VARS: DOCKER                            #
    #================================================================#
    docker_enabled: false             # enable docker on this node?
    docker_cgroups_driver: systemd    # docker cgroup fs driver: cgroupfs,systemd
    docker_registry_mirrors: []       # docker registry mirror list
    docker_image_cache: /tmp/docker   # docker image cache dir, `/tmp/docker` by default


    #================================================================#
    #                         VARS: ETCD                             #
    #================================================================#
    #etcd_seq: 1                      # etcd instance identifier, explicitly required
    #etcd_cluster: etcd               # etcd cluster & group name, etcd by default
    etcd_safeguard: false             # prevent purging running etcd instance?
    etcd_clean: true                  # purging existing etcd during initialization?
    etcd_data: /data/etcd             # etcd data directory, /data/etcd by default
    etcd_port: 2379                   # etcd client port, 2379 by default
    etcd_peer_port: 2380              # etcd peer port, 2380 by default
    etcd_init: new                    # etcd initial cluster state, new or existing
    etcd_election_timeout: 1000       # etcd election timeout, 1000ms by default
    etcd_heartbeat_interval: 100      # etcd heartbeat interval, 100ms by default


    #================================================================#
    #                         VARS: MINIO                            #
    #================================================================#
    #minio_seq: 1                     # minio instance identifier, REQUIRED
    minio_cluster: minio              # minio cluster name, minio by default
    minio_clean: false                # cleanup minio during init?, false by default
    minio_user: minio                 # minio os user, `minio` by default
    minio_node: '${minio_cluster}-${minio_seq}.pigsty' # minio node name pattern
    minio_data: '/data/minio'         # minio data dir(s), use {x...y} to specify multi drivers
    minio_domain: sss.pigsty          # minio external domain name, `sss.pigsty` by default
    minio_port: 9000                  # minio service port, 9000 by default
    minio_admin_port: 9001            # minio console port, 9001 by default
    minio_access_key: minioadmin      # root access key, `minioadmin` by default
    minio_secret_key: minioadmin      # root secret key, `minioadmin` by default
    minio_extra_vars: ''              # extra environment variables
    minio_alias: sss                  # alias name for local minio deployment
    minio_buckets: [ { name: pgsql }, { name: infra },  { name: redis } ]
    minio_users:
      - { access_key: dba , secret_key: S3User.DBA, policy: consoleAdmin }
      - { access_key: pgbackrest , secret_key: S3User.Backup, policy: readwrite }


    #================================================================#
    #                         VARS: REDIS                            #
    #================================================================#
    #redis_cluster:        <CLUSTER> # redis cluster name, required identity parameter
    #redis_node: 1            <NODE> # redis node sequence number, node int id required
    #redis_instances: {}      <NODE> # redis instances definition on this redis node
    redis_fs_main: /data              # redis main data mountpoint, `/data` by default
    redis_exporter_enabled: true      # install redis exporter on redis nodes?
    redis_exporter_port: 9121         # redis exporter listen port, 9121 by default
    redis_exporter_options: ''        # cli args and extra options for redis exporter
    redis_safeguard: false            # prevent purging running redis instance?
    redis_clean: true                 # purging existing redis during init?
    redis_rmdata: true                # remove redis data when purging redis server?
    redis_mode: standalone            # redis mode: standalone,cluster,sentinel
    redis_conf: redis.conf            # redis config template path, except sentinel
    redis_bind_address: '0.0.0.0'     # redis bind address, empty string will use host ip
    redis_max_memory: 1GB             # max memory used by each redis instance
    redis_mem_policy: allkeys-lru     # redis memory eviction policy
    redis_password: ''                # redis password, empty string will disable password
    redis_rdb_save: ['1200 1']        # redis rdb save directives, disable with empty list
    redis_aof_enabled: false          # enable redis append only file?
    redis_rename_commands: {}         # rename redis dangerous commands
    redis_cluster_replicas: 1         # replica number for one master in redis cluster


    #================================================================#
    #                         VARS: PGSQL                            #
    #================================================================#

    #-----------------------------------------------------------------
    # PG_IDENTITY
    #-----------------------------------------------------------------
    pg_mode: pgsql          #CLUSTER  # pgsql cluster mode: pgsql,citus,gpsql
    # pg_cluster:           #CLUSTER  # pgsql cluster name, required identity parameter
    # pg_seq: 0             #INSTANCE # pgsql instance seq number, required identity parameter
    # pg_role: replica      #INSTANCE # pgsql role, required, could be primary,replica,offline
    # pg_instances: {}      #INSTANCE # define multiple pg instances on node in `{port:ins_vars}` format
    # pg_upstream:          #INSTANCE # repl upstream ip addr for standby cluster or cascade replica
    # pg_shard:             #CLUSTER  # pgsql shard name, optional identity for sharding clusters
    # pg_group: 0           #CLUSTER  # pgsql shard index number, optional identity for sharding clusters
    # gp_role: master       #CLUSTER  # greenplum role of this cluster, could be master or segment
    pg_offline_query: false #INSTANCE # set to true to enable offline query on this instance

    #-----------------------------------------------------------------
    # PG_BUSINESS
    #-----------------------------------------------------------------
    # postgres business object definition, overwrite in group vars
    pg_users: []                      # postgres business users
    pg_databases: []                  # postgres business databases
    pg_services: []                   # postgres business services
    pg_hba_rules: []                  # business hba rules for postgres
    pgb_hba_rules: []                 # business hba rules for pgbouncer
    # global credentials, overwrite in global vars
    pg_replication_username: replicator
    pg_replication_password: DBUser.Replicator
    pg_admin_username: dbuser_dba
    pg_admin_password: DBUser.DBA
    pg_monitor_username: dbuser_monitor
    pg_monitor_password: DBUser.Monitor
    pg_dbsu_password: ''              # dbsu password, empty string means no dbsu password by default

    #-----------------------------------------------------------------
    # PG_INSTALL
    #-----------------------------------------------------------------
    pg_dbsu: postgres                 # os dbsu name, postgres by default, better not change it
    pg_dbsu_uid: 26                   # os dbsu uid and gid, 26 for default postgres users and groups
    pg_dbsu_sudo: limit               # dbsu sudo privilege, none,limit,all,nopass. limit by default
    pg_dbsu_home: /var/lib/pgsql      # postgresql home directory, `/var/lib/pgsql` by default
    pg_dbsu_ssh_exchange: true        # exchange postgres dbsu ssh key among same pgsql cluster
    pg_version: 15                    # postgres major version to be installed, 15 by default
    pg_bin_dir: /usr/pgsql/bin        # postgres binary dir, `/usr/pgsql/bin` by default
    pg_log_dir: /pg/log/postgres      # postgres log dir, `/pg/log/postgres` by default
    pg_packages:                      # pg packages to be installed, `${pg_version}` will be replaced
      - postgresql${pg_version}*
      - pgbouncer pg_exporter pgbadger vip-manager patroni patroni-etcd pgbackrest
    pg_extensions:                    # pg extensions to be installed, `${pg_version}` will be replaced
      - postgis33_${pg_version}* pg_repack_${pg_version} wal2json_${pg_version} pgvector_${pg_version}* timescaledb-2-postgresql-${pg_version} citus*${pg_version}*

    #-----------------------------------------------------------------
    # PG_BOOTSTRAP
    #-----------------------------------------------------------------
    pg_safeguard: false               # prevent purging running postgres instance? false by default
    pg_clean: true                    # purging existing postgres during pgsql init? true by default
    pg_data: /pg/data                 # postgres data directory, `/pg/data` by default
    pg_fs_main: /data                 # mountpoint/path for postgres main data, `/data` by default
    pg_fs_bkup: /data/backups         # mountpoint/path for pg backup data, `/data/backup` by default
    pg_storage_type: SSD              # storage type for pg main data, SSD,HDD, SSD by default
    pg_dummy_filesize: 64MiB          # size of `/pg/dummy`, hold 64MB disk space for emergency use
    pg_listen: '0.0.0.0'              # postgres/pgbouncer listen addresses, comma separated list
    pg_port: 5432                     # postgres listen port, 5432 by default
    pg_localhost: /var/run/postgresql # postgres unix socket dir for localhost connection
    pg_namespace: /pg                 # top level key namespace in etcd, used by patroni & vip
    patroni_enabled: true             # if disabled, no postgres cluster will be created during init
    patroni_mode: default             # patroni working mode: default,pause,remove
    patroni_port: 8008                # patroni listen port, 8008 by default
    patroni_log_dir: /pg/log/patroni  # patroni log dir, `/pg/log/patroni` by default
    patroni_ssl_enabled: false        # secure patroni RestAPI communications with SSL?
    patroni_watchdog_mode: off        # patroni watchdog mode: automatic,required,off. off by default
    patroni_username: postgres        # patroni restapi username, `postgres` by default
    patroni_password: Patroni.API     # patroni restapi password, `Patroni.API` by default
    patroni_citus_db: postgres        # citus database managed by patroni, postgres by default
    pg_conf: oltp.yml                 # config template: oltp,olap,crit,tiny. `oltp.yml` by default
    pg_max_conn: auto                 # postgres max connections, `auto` will use recommended value
    pg_shared_buffer_ratio: 0.25      # postgres shared buffer ratio, 0.25 by default, 0.1~0.4
    pg_rto: 30                        # recovery time objective in seconds,  `30s` by default
    pg_rpo: 1048576                   # recovery point objective in bytes, `1MiB` at most by default
    pg_libs: 'timescaledb, pg_stat_statements, auto_explain'  # extensions to be loaded
    pg_delay: 0                       # replication apply delay for standby cluster leader
    pg_checksum: false                # enable data checksum for postgres cluster?
    pg_pwd_enc: scram-sha-256         # passwords encryption algorithm: md5,scram-sha-256
    pg_encoding: UTF8                 # database cluster encoding, `UTF8` by default
    pg_locale: C                      # database cluster local, `C` by default
    pg_lc_collate: C                  # database cluster collate, `C` by default
    pg_lc_ctype: en_US.UTF8           # database character type, `en_US.UTF8` by default
    pgbouncer_enabled: true           # if disabled, pgbouncer will not be launched on pgsql host
    pgbouncer_port: 6432              # pgbouncer listen port, 6432 by default
    pgbouncer_log_dir: /pg/log/pgbouncer  # pgbouncer log dir, `/pg/log/pgbouncer` by default
    pgbouncer_auth_query: false       # query postgres to retrieve unlisted business users?
    pgbouncer_poolmode: transaction   # pooling mode: transaction,session,statement, transaction by default
    pgbouncer_sslmode: disable        # pgbouncer client ssl mode, disable by default

    #-----------------------------------------------------------------
    # PG_PROVISION
    #-----------------------------------------------------------------
    pg_provision: true                # provision postgres cluster after bootstrap
    pg_init: pg-init                  # provision init script for cluster template, `pg-init` by default
    pg_default_roles:                 # default roles and users in postgres cluster
      - { name: dbrole_readonly  ,login: false ,comment: role for global read-only access     }
      - { name: dbrole_offline   ,login: false ,comment: role for restricted read-only access }
      - { name: dbrole_readwrite ,login: false ,roles: [dbrole_readonly] ,comment: role for global read-write access }
      - { name: dbrole_admin     ,login: false ,roles: [pg_monitor, dbrole_readwrite] ,comment: role for object creation }
      - { name: postgres     ,superuser: true  ,comment: system superuser }
      - { name: replicator ,replication: true  ,roles: [pg_monitor, dbrole_readonly] ,comment: system replicator }
      - { name: dbuser_dba   ,superuser: true  ,roles: [dbrole_admin]  ,pgbouncer: true ,pool_mode: session, pool_connlimit: 16 ,comment: pgsql admin user }
      - { name: dbuser_monitor ,roles: [pg_monitor] ,pgbouncer: true ,parameters: {log_min_duration_statement: 1000 } ,pool_mode: session ,pool_connlimit: 8 ,comment: pgsql monitor user }
    pg_default_privileges:            # default privileges when created by admin user
      - GRANT USAGE      ON SCHEMAS   TO dbrole_readonly
      - GRANT SELECT     ON TABLES    TO dbrole_readonly
      - GRANT SELECT     ON SEQUENCES TO dbrole_readonly
      - GRANT EXECUTE    ON FUNCTIONS TO dbrole_readonly
      - GRANT USAGE      ON SCHEMAS   TO dbrole_offline
      - GRANT SELECT     ON TABLES    TO dbrole_offline
      - GRANT SELECT     ON SEQUENCES TO dbrole_offline
      - GRANT EXECUTE    ON FUNCTIONS TO dbrole_offline
      - GRANT INSERT     ON TABLES    TO dbrole_readwrite
      - GRANT UPDATE     ON TABLES    TO dbrole_readwrite
      - GRANT DELETE     ON TABLES    TO dbrole_readwrite
      - GRANT USAGE      ON SEQUENCES TO dbrole_readwrite
      - GRANT UPDATE     ON SEQUENCES TO dbrole_readwrite
      - GRANT TRUNCATE   ON TABLES    TO dbrole_admin
      - GRANT REFERENCES ON TABLES    TO dbrole_admin
      - GRANT TRIGGER    ON TABLES    TO dbrole_admin
      - GRANT CREATE     ON SCHEMAS   TO dbrole_admin
    pg_default_schemas: [ monitor ]   # default schemas to be created
    pg_default_extensions:            # default extensions to be created
      - { name: adminpack          ,schema: pg_catalog }
      - { name: pg_stat_statements ,schema: monitor }
      - { name: pgstattuple        ,schema: monitor }
      - { name: pg_buffercache     ,schema: monitor }
      - { name: pageinspect        ,schema: monitor }
      - { name: pg_prewarm         ,schema: monitor }
      - { name: pg_visibility      ,schema: monitor }
      - { name: pg_freespacemap    ,schema: monitor }
      - { name: postgres_fdw       ,schema: public  }
      - { name: file_fdw           ,schema: public  }
      - { name: btree_gist         ,schema: public  }
      - { name: btree_gin          ,schema: public  }
      - { name: pg_trgm            ,schema: public  }
      - { name: intagg             ,schema: public  }
      - { name: intarray           ,schema: public  }
      - { name: pg_repack }
    pg_reload: true                   # reload postgres after hba changes
    pg_default_hba_rules:             # postgres default host-based authentication rules
      - {user: '${dbsu}'    ,db: all         ,addr: local     ,auth: ident ,title: 'dbsu access via local os user ident'  }
      - {user: '${dbsu}'    ,db: replication ,addr: local     ,auth: ident ,title: 'dbsu replication from local os ident' }
      - {user: '${repl}'    ,db: replication ,addr: localhost ,auth: pwd   ,title: 'replicator replication from localhost'}
      - {user: '${repl}'    ,db: replication ,addr: intra     ,auth: pwd   ,title: 'replicator replication from intranet' }
      - {user: '${repl}'    ,db: postgres    ,addr: intra     ,auth: pwd   ,title: 'replicator postgres db from intranet' }
      - {user: '${monitor}' ,db: all         ,addr: localhost ,auth: pwd   ,title: 'monitor from localhost with password' }
      - {user: '${monitor}' ,db: all         ,addr: infra     ,auth: pwd   ,title: 'monitor from infra host with password'}
      - {user: '${admin}'   ,db: all         ,addr: infra     ,auth: ssl   ,title: 'admin @ infra nodes with pwd & ssl'   }
      - {user: '${admin}'   ,db: all         ,addr: world     ,auth: cert  ,title: 'admin @ everywhere with ssl & cert'   }
      - {user: '+dbrole_readonly',db: all    ,addr: localhost ,auth: pwd   ,title: 'pgbouncer read/write via local socket'}
      - {user: '+dbrole_readonly',db: all    ,addr: intra     ,auth: pwd   ,title: 'read/write biz user via password'     }
      - {user: '+dbrole_offline' ,db: all    ,addr: intra     ,auth: pwd   ,title: 'allow etl offline tasks from intranet'}
    pgb_default_hba_rules:            # pgbouncer default host-based authentication rules
      - {user: '${dbsu}'    ,db: pgbouncer   ,addr: local     ,auth: peer  ,title: 'dbsu local admin access with os ident'}
      - {user: 'all'        ,db: all         ,addr: localhost ,auth: pwd   ,title: 'allow all user local access with pwd' }
      - {user: '${monitor}' ,db: pgbouncer   ,addr: intra     ,auth: pwd   ,title: 'monitor access via intranet with pwd' }
      - {user: '${monitor}' ,db: all         ,addr: world     ,auth: deny  ,title: 'reject all other monitor access addr' }
      - {user: '${admin}'   ,db: all         ,addr: intra     ,auth: pwd   ,title: 'admin access via intranet with pwd'   }
      - {user: '${admin}'   ,db: all         ,addr: world     ,auth: deny  ,title: 'reject all other admin access addr'   }
      - {user: 'all'        ,db: all         ,addr: intra     ,auth: pwd   ,title: 'allow all user intra access with pwd' }

    #-----------------------------------------------------------------
    # PG_BACKUP
    #-----------------------------------------------------------------
    pgbackrest_enabled: true          # enable pgbackrest on pgsql host?
    pgbackrest_clean: true            # remove pg backup data during init?
    pgbackrest_log_dir: /pg/log/pgbackrest # pgbackrest log dir, `/pg/log/pgbackrest` by default
    pgbackrest_method: local          # pgbackrest repo method: local,minio,[user-defined...]
    pgbackrest_repo:                  # pgbackrest repo: https://pgbackrest.org/configuration.html#section-repository
      local:                          # default pgbackrest repo with local posix fs
        path: /pg/backup              # local backup directory, `/pg/backup` by default
        retention_full_type: count    # retention full backups by count
        retention_full: 2             # keep 2, at most 3 full backup when using local fs repo
      minio:                          # optional minio repo for pgbackrest
        type: s3                      # minio is s3-compatible, so s3 is used
        s3_endpoint: sss.pigsty       # minio endpoint domain name, `sss.pigsty` by default
        s3_region: us-east-1          # minio region, us-east-1 by default, useless for minio
        s3_bucket: pgsql              # minio bucket name, `pgsql` by default
        s3_key: pgbackrest            # minio user access key for pgbackrest
        s3_key_secret: S3User.Backup  # minio user secret key for pgbackrest
        s3_uri_style: path            # use path style uri for minio rather than host style
        path: /pgbackrest             # minio backup path, default is `/pgbackrest`
        storage_port: 9000            # minio port, 9000 by default
        storage_ca_file: /etc/pki/ca.crt  # minio ca file path, `/etc/pki/ca.crt` by default
        bundle: y                     # bundle small files into a single file
        cipher_type: aes-256-cbc      # enable AES encryption for remote backup repo
        cipher_pass: pgBackRest       # AES encryption password, default is 'pgBackRest'
        retention_full_type: time     # retention full backup by time on minio repo
        retention_full: 14            # keep full backup for last 14 days

    #-----------------------------------------------------------------
    # PG_SERVICE
    #-----------------------------------------------------------------
    pg_weight: 100                    # relative load balance weight in service, 100 by default, 0-255
    pg_service_provider: ''           # dedicate haproxy node group name, or empty string for local nodes by default
    pg_default_service_dest: pgbouncer # default service destination if svc.dest='default'
    pg_default_services:              # postgres default service definitions
      - { name: primary ,port: 5433 ,dest: default  ,check: /primary   ,selector: "[]" }
      - { name: replica ,port: 5434 ,dest: default  ,check: /read-only ,selector: "[]" , backup: "[? pg_role == `primary` || pg_role == `offline` ]" }
      - { name: default ,port: 5436 ,dest: postgres ,check: /primary   ,selector: "[]" }
      - { name: offline ,port: 5438 ,dest: postgres ,check: /replica   ,selector: "[? pg_role == `offline` || pg_offline_query ]" , backup: "[? pg_role == `replica` && !pg_offline_query]"}
    pg_vip_enabled: false             # enable a l2 vip for pgsql primary? false by default
    pg_vip_address: 127.0.0.1/24      # vip address in `<ipv4>/<mask>` format, require if vip is enabled
    pg_vip_interface: eth0            # vip network interface to listen, eth0 by default
    pg_dns_suffix: ''                 # pgsql dns suffix, '' by default
    pg_dns_target: auto               # auto, primary, vip, none, or ad hoc ip

    #-----------------------------------------------------------------
    # PG_EXPORTER
    #-----------------------------------------------------------------
    pg_exporter_enabled: true              # enable pg_exporter on pgsql hosts?
    pg_exporter_config: pg_exporter.yml    # pg_exporter configuration file name
    pg_exporter_cache_ttls: '1,10,60,300'  # pg_exporter collector ttl stage in seconds, '1,10,60,300' by default
    pg_exporter_port: 9630                 # pg_exporter listen port, 9630 by default
    pg_exporter_params: 'sslmode=disable'  # extra url parameters for pg_exporter dsn
    pg_exporter_url: ''                    # overwrite auto-generate pg dsn if specified
    pg_exporter_auto_discovery: true       # enable auto database discovery? enabled by default
    pg_exporter_exclude_database: 'template0,template1,postgres' # csv of database that WILL NOT be monitored during auto-discovery
    pg_exporter_include_database: ''       # csv of database that WILL BE monitored during auto-discovery
    pg_exporter_connect_timeout: 200       # pg_exporter connect timeout in ms, 200 by default
    pg_exporter_options: ''                # overwrite extra options for pg_exporter
    pgbouncer_exporter_enabled: true       # enable pgbouncer_exporter on pgsql hosts?
    pgbouncer_exporter_port: 9631          # pgbouncer_exporter listen port, 9631 by default
    pgbouncer_exporter_url: ''             # overwrite auto-generate pgbouncer dsn if specified
    pgbouncer_exporter_options: ''         # overwrite extra options for pgbouncer_exporter

...
```

### 2.4. 多套pigsty

在企业中，DBA管理的集群肯定不止一套。我们发现，其实有些组件是可以重复利用的，比如granfa，prometheus等，因此我们没必要重复创建。我们为了统一展示，可以把一部分重复的功能用1台机器统一管理。

![image-20230617115934631](/pages/keynotes/L7_architect_storage/2_database/pics/2_3_postgresql_13_HA/image-20230617115934631.png)
