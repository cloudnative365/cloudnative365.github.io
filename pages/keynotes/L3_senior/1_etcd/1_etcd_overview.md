---
title: etcd概述
keywords: keynotes, senior, etcd, etcd_overview
permalink: keynotes_L3_senior_1_etcd_1_etcd_overview.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/1_etcd_overview
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- etcd发展历程
- etcd的架构
- etcd的内部实现机制
- etcd的使用场景

## 1. etcd简介

etcd原来是CoreOS公司的产品，他最初适用于解决集群管理系统中OS升级的分布式并发控制及配置文件的存储于并发问题而设计的。所以，etcd被设计为高可用、强一致性的小型k-v数据存储服务。

目前，这个项目已经贡献给了CNCF，目前处于孵化状态。被AWS、Google、Microsoft、Alibaba等大型互联网公司广泛使用。当然，应用最广泛的还是在云原生技术，也就是kubernetes的数据库来使用，用来存储整个集群的状态。还有其他的项目，比如 [locksmith](https://github.com/coreos/locksmith), [vulcand](https://github.com/vulcand/vulcand), [Doorman](https://github.com/youtube/doorman),consul都在使用它

+  2013 年 6 月份由 CoreOS 公司向 GitHub 中提交了第一个版本的初始代码。

+ 2014 年的 6 月，社区发生了一件事情，Kubernetes v0.4 版本发布，它使用了 etcd 0.2 版本作为实验核心元数据的存储服务，自此 etcd 社区得到了飞速的发展。

+ 2015 年 2 月份，etcd 发布了第一个正式的稳定版本 2.0。在 2.0 版本中，etcd 重新实践了 Raft 一致性算法，另外用户提供了一个简单的树形数据视图，在 2.0 版本中 etcd 支持每秒超过 1000 次的写入性能，满足了当时绝大多数的应用场景需求。2.0 版本发布之后，经过不断的迭代与改进，其原有的数据存储方案逐渐成为了新世纪的性能瓶颈，因此 etcd 启动了 v3 版本的方案设计。

+ 2017 年 1 月，etcd 发布了 3.1 版本，基本上标志着 v3 版本方案的全面成熟。在 v3 版本中 etcd 提供了一套全新的 API，并且重新实践了更有效的一致性读取方法，同时提供了一个 gRPC 接口。gRPC 的 proxy 用于扩展 etcd 的读取性能，同时在 v3 版本的方案中包含了大量的 GC 的优化，极大地提高了 etcd 的性能。在该版本中 etcd 可以支持每秒超过 10000 次的写入。

+ 2018 年，CNCF 基金会下的众多项目都使用了 etcd 作为其核心的数据存储。据不完全统计，使用 etcd 的项目超过了 30 个，在同年 11 月份，etcd 项目自身也成为了 CNCF 旗下的孵化项目。进入 CNCF 基金会后，etcd 拥有了超过 400 个贡献组，其中包含了来自 AWS、Google、Alibaba 等 8 个公司的 9 个项目维护者。

+ 2019 年，etcd 即将发布全新的 3.4 版本，该版本由 Google、Alibaba 等公司联合打造，将进一步改进 etcd 的性能及稳定性，以满足在超大型公司使用中苛刻的场景要求。
+ 到这篇文章写成的时候，是2020年6月，目前etcd的版本已经到了3.4.9



## 2. 架构

### 2.1. 架构解析

etcd 是一个分布式的、可靠的 key-value 存储系统，它用于存储分布式系统中的关键数据

> etcd is a distributed reliable key-value store for the most critical data of a distributed system

![image-20200611110309495](/pages/keynotes/L3_senior/1_etcd/pics/1_etcd_overview/image-20200611110309495.png)

一个 etcd 集群，通常会由 3 个， 5 个等奇数节点组成，多个节点之间，通过一个叫做 Raft 一致性算法的方式完成分布式一致性协同，算法会选举出一个主节点作为 leader，由 leader 负责数据的同步与数据的分发，当 leader 出现故障后，系统会自动地选取另一个节点成为 leader，并重新完成数据的同步与分发。客户端在众多的 leader 中，仅需要选择其中的一个就可以完成数据的读写。

在 etcd 整个的架构中，有一个非常关键的概念叫做 quorum，quorum 的定义是 =（n+1）/2，也就是说超过集群中半数节点组成的一个团体，在 3 个节点的集群中，etcd 可以容许 1 个节点故障，也就是只要有任何 2 个节点重合，etcd 就可以继续提供服务。同理，在 5 个节点的集群中，只要有任何 3 个节点重合，etcd 就可以继续提供服务。这也是 etcd 集群高可用的关键。

当我们在允许部分节点故障之后，继续提供服务，这里就需要解决一个非常复杂的问题，即分布式一致性。在 etcd 中，该分布式一致性算法由 [Raft](https://raft.github.io/) 一致性算法完成，这个算法本身是比较复杂的，我们这里就不展开详细介绍了。

但是这里面有一个关键点，它基于一个前提：任意两个 quorum 的成员之间一定会有一个交集，也就是说只要有任意一个 quorum 存活，其中一定存在某一个节点，它包含着集群中最新的数据。正是基于这个假设，这个一致性算法就可以在一个 quorum 之间采用这份最新的数据去完成数据的同步，从而保证整个集群**向前衍进**的过程中其数据保持一致。

### 2.2. API接口

虽然 etcd 内部的机制比较复杂，但是 etcd 给客户提供的接口是比较简单的。如下图所示，我们可以通过 etcd 提供的客户端去访问集群的数据，也可以直接通过 http 的方式，类似像 curl 命令直接访问 etcd。在 etcd 内部，其数据表示也是比较简单的，我们可以直接把 etcd 的数据存储理解为一个有序的 map，它存储着 key-value 数据。同时 etcd 为了方便客户端去订阅资料的数据，也支持了一个 watch 机制，我们可以通过 watch 实时地拿到 etcd 中数据的增量更新，从而保持与 etcd 中的数据同步。![image-20200611112120489](/pages/keynotes/L3_senior/1_etcd/pics/1_etcd_overview/image-20200611112120489.png)

接下来我们看一下 [etcd 提供的接口](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md#key-value-api)，这里将 etcd 的接口分为了 5 组：

- 第一组是 Put(key, value) 与 Delete(key)。

  上图可以看到 put 与 delete 的操作都非常简单，只需要提供一个 key 和一个 value，就可以向集群中写入数据了，那么在删除数据的时候，只需要提供 key 就可以了；

- 第二组是Get(key)/Get(keyForm, keyEnd)

  这是一组查询操作。查询操作 etcd 支持两种类型的查询：第一种是指定单个 key 的查询，第二种是指定的一个 key 的范围；

- 第三组是Watch(key/keyPrefix)

  etcd 启动了 Watch 的机制，也就是我们前面提到的用于实现增量的数据更新，watch 也是有两种使用方法，第一种是指定单个 key 的 Watch，另一种是指定一个 key 的前缀。在实际应用场景的使用过程中，经常采用第二种；

- 第四组是Transactions(if/ then /else osp).Commit()

  API 是 etcd 提供的一个事务操作，可以通过指定某些条件，当条件成立的时候执行一组操作。当条件不成立的时候执行另外一组操作；

- 第五组是 Leases: Grant/Revoke/KeepAlive

  Leases 接口是分布式系统中常用的一种设计模式，后面会具体介绍。

## 3. 内部实现机制

### 2.3. etcd 的数据版本号机制

要正确使用 etcd 的 API，必须要知道内部对应数据版本号的基本原理。 

- term: 全局单调递增，64bits

  代表的是整个集群 Leader 的标志。当集群发生 Leader 切换，比如说 Leader 节点故障，或者说 Leader 节点网络出现问题，再或者是将整个集群停止后再次拉起，这个时候都会发生 Leader 的切换。当 Leader 切换的时候，term 的值就会 +1。

- revision：全局单调递增，64bits

  代表的是全局数据的版本。当数据发生变更，包括创建、修改、删除，revision 对应的都会 +1。在任期内，revision 都可以保持全局单调递增的更改。正是 revision 的存在才使得 etcd 既可以支持数据的 MVCC(Mutil-Version Concurrency Control多版本并发控制)，也可以支持数据的 Watch。

- 对于每一个 KeyValue 数据，etcd 中都记录了三个版本：
  - 第一个版本叫做 create_revision，是 KeyValue 在创建的时候生成的版本号；
  - 第二个叫做 mod_revision，是其数据被操作的时候对应的版本号；
  - 第三个 version 就是一个计数器，代表了 KeyValue 被修改了多少次。

![image-20200611113556869](/pages/keynotes/L3_senior/1_etcd/pics/1_etcd_overview/image-20200611113556869.png)

在同一个 Leader 任期之内，我们发现所有的修改操作，其对应的 term 值都等于 2，始终保持不变，而 revision 则保持单调递增。 

当重启集群之后，我们会发现所有的修改操作对应的 term 值都变成了 3。在新的任期内，所有的 term 值都等于3，且不会发生变化。而对应的 revision 值同样保持单调递增。

从一个更大的维度去看，可以发现：在多个任期内，其数据对应的 revision 值会保持全局的单调递增。

### 3.2. etcd mvcc和streaming watch

也就是 etcd 多版本号的并发控制(mvcc)以及 watch 的使用方法。

首先，在 etcd 中，支持对同一个 Key 发起多次数据修改。因为已经知道每次数据修改都对应一个版本号，多次修改就意味着一个 key 中存在多个版本，在查询数据的时候可以通过不指定版本号查询，这时 etcd 会返回该数据的最新版本。当我们指定一个版本号查询数据后，可以获取到一个 Key 的历史版本。

因为每次操作修改数据的时候都会对应一个版本号。 在 watch 的时候指定数据的版本，创建一个 watcher，并通过这个 watcher 提供的一个数据管道，能够获取到指定的 revision 之后所有的数据变更。如果指定的 revision 是一个旧版本，可以立即拿到从旧版本到当前版本所有的数据更新。并且，watch 的机制会保证 etcd 中，该 Key 的数据发生后续的修改后，依然可以从这个数据管道中拿到数据增量的更新。 

为了更好地理解 etcd 的多版本控制以及 watch 的机制，可以简单的介绍一下 etcd 的内部实现。

![image-20200611132528566](/pages/keynotes/L3_senior/1_etcd/pics/1_etcd_overview/image-20200611132528566.png)

在 etcd 中所有的数据都存储在一个 b+tree 中。b+tree 是保存在磁盘中，并通过 mmap 的方式映射到内存用来查询操作。

在 b+tree 中（如图所示灰色部分），维护着 revision 到 value 的映射关系。也就是说当指定 revision 查询数据的时候，就可以通过该 b+tree 直接返回数据。当我们通过 watch 来订阅数据的时候，也可以通过这个 b+tree 维护的 revision 到 value 映射关系，从而通过指定的 revision 开始遍历这个 b+tree，拿到所有的数据更新。

同时在 etcd 内部还维护着另外一个 b+tree。它管理着 key 到 revision 的映射关系。当需要查询 Key 对应数据的时候，会通过蓝色方框的 b+tree，将 key 翻译成 revision。再通过灰色框 b+tree 中的 revision 获取到对应的 value。至此就能满足客户端不同的查询场景了。

这里需要提两点：

- 一个数据是有多个版本的；
- 在 etcd 持续运行过程中会不断的发生修改，意味着 etcd 中内存及磁盘的数据都会持续增长。这对资源有限的场景来说是无法接受的。因此在 etcd 中会**周期性的运行一个 Compaction 的机制**来清理历史数据。对于一个 Key 的历史版本数据，可以选择清理掉。

### 3.3. etcd mini-transactions（MTR）

在理解了 mvcc 机制及 watch 机制之后，来介绍一下 etcd 提供的 mini-transactions 机制。etcd 的 transaction 机制比较简单，基本可以理解为一段 if-else 程序，在 if 中可以提供多个操作。

![image-20200611133134896](/pages/keynotes/L3_senior/2_etcd/pics/2_etcd_overview/image-20200611133134896.png)

在上图的示例中，if 里面写了两个条件。当 Value(key1) 大于“bar”，并且 Version(key1) 的版本等于 2 的时候，执行 Then 里面指定的操作：修改 Key2 的数据为 valueX，同时删除 Key3 的数据。如果不满足条件，则执行另外一个操作：Key2 修改为 valueY。

在 etcd 内部会保证整个事务操作的原子性。也就是说 If 操作所有的比较条件，其看到的视图，一定是一致的。同时它能够确保在争执条件中，多个操作的原子性不会出现 etc 仅执行了一半的情况。

通过 etcd 提供的事务操作，我们可以在多个竞争中去保证数据读写的一致性，比如说前面已经提到过的 Kubernetes 项目，它正是利用了 etcd 的事务机制，来实现多个 KubernetesAPI server 对同样一个数据修改的一致性。

![image-20200611133427250](/pages/keynotes/L3_senior/1_etcd/pics/1_etcd_overview/image-20200611133427250.png)

### 3.4. etcd lease 的概念及用法

lease 是分布式系统中一个常见的概念，用于代表一个租约。通常情况下，在分布式系统中需要去检测一个节点是否存活的时候，就需要租约机制。

![image-20200611133525836](/pages/keynotes/L3_senior/1_etcd/pics/1_etcd_overview/image-20200611133525836.png)

上图示例中首先创建了一个 10s 的租约，如果创建租约后不做任何的操作，那么 10s 之后，这个租约就会自动过期。 

在这里，接着将 key1 和 key2 两个 key value 绑定到这个租约之上，这样当租约过期时，etcd 就会自动清理掉 key1 和 key2 对应的数据。

如果希望这个租约永不过期，比如说需要检测分布式系统中一个进程是否存活，那么就会在这个分布式进程中去访问 etcd 并且创建一个租约，同时在该进程中去调用 KeepAlive 的方法，与 etcd 保持一个租约不断的续约。试想一下，如果这个进程挂掉了，这时就没有办法再继续去开发 Alive。租约在进程挂掉的一段时间就会被 etcd 自动清理掉。所以可以通过这个机制来判定节点是否存活。

在 etcd 中，允许将多个 key 关联在统一的 lease 之上，这个设计是非常巧妙的，事实上最初的设计也不是这个样子。通过这种方法，将多个 key 绑定在同一个lease对象，可以大幅地减少 lease 对象刷新的时间。试想一下，如果大量的 key 都需要绑定在同一个 lease 对象之上，每一个 key 都去更新这个租约的话，这个 etcd 会产生非常大的压力。通过支持加多个 key 绑定在同一个 lease 之上，它既不失灵活性同时能够大幅改进 etcd 整个系统的性能。

