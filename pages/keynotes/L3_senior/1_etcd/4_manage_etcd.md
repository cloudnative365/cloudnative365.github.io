---
title: 管理etcd集群
keywords: keynotes, senior, etcd, manage_etcd
permalink: keynotes_L3_senior_1_etcd_4_manage_etcd.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/4_manage_etcd
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 向etcd集群中添加节点
- 从etcd集群中删除节点
- etcd集群的备份与恢复
- etcd集群的升级

## 1. 向etcd集群中添加节点

[官方文档](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#scaling-up-etcd-clusters)中不建议我们向生产系统中添加节点，因为添加etcd节点会损失集群的性能，增加节点并不会增加集群的性能或者是容量。一般来说不建议对etcd集群扩/缩容。更不要配置任何的etcd集群自动扩展策略。我们建议在生产系统中部署五个member的etcd集群。过程参考[github](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/runtime-configuration.md#remove-a-member)。

当然，如果有特殊需求，我们也可以部署7个节点，比如两地三中心的架构，三个DC中分别部署2/2/3个etcd集群。

### 1.1. 添加一个正常工作的etcd节点

一般来说，添加节点有两个步骤：

+ 我们可以通过 [HTTP members API](https://github.com/etcd-io/etcd/blob/master/Documentation/v2/members_api.md)， [gRPC members API](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto),或者是 `etcdctl member add` 命令.
+ 使用新的集群配置启动新的节点
+ 同步后，使用新的配置重新启动原有节点

### 1.2. 添加一个learner etcd节点

从etcd3.4版本之后，etcd知识把节点添加为learner节点（不会参与投票的节点）。这样做是为了让添加新节点更加的安全，较少添加节点过程中集群的宕机时间，我们建议在节点在完全同步数据之前作为learner节点。所以我们增加节点的步骤变成了下面几个

+ 我们可以通过 [HTTP members API](https://github.com/etcd-io/etcd/blob/master/Documentation/v2/members_api.md)， [gRPC members API](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto),或者是 `etcdctl member add --learner` 命令.
+ 使用新的集群配置启动新的节点
+ 使用新的配置重新启动原有节点
+ 使用 [gRPC members API](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto),或者是 `etcdctl member promote`命令来让learner节点成为有投票权的节点。这个时候etcd集群会检查一下promote的请求来确认这个操作是安全的。他会去确认learner节点确实已经同步了leader数据的节点，然后才让这个节点拥有投票权。





### 1.3. 如果添加节点出现问题

我们这里举例是同步出现问题，比如：

``` bash
$ etcd --name infra3 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: the member count is unequal
exit 1
```

我们需要清空数据文件，再更换一下节点的信息（节点名称和IP）

```bash
$ etcd --name infra4 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra4=http://10.0.1.14:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: unmatched member while checking PeerURLs
exit 1
```

### 1.4. 如果添加learner节点出现问题

+ 集群中多于1个learner会报错

``` bash
$ etcdctl member add infra4 --peer-urls=http://10.0.1.14:2380 --learner
Error: etcdserver: too many learner members in cluster
```

+ 数据不同步的情况下会报错

``` bash
$ etcdctl member promote 9bf1b35fc7761a23
Error: etcdserver: can only promote a learner member which is in sync with leader
```

+ 提升一个不是learner的节点会报错

``` bash
$ etcdctl member promote 9bf1b35fc7761a23
Error: etcdserver: can only promote a learner member
```

+ 提升一个不存在的learner节点会报错

``` bash
$ etcdctl member promote 12345abcde
Error: etcdserver: member not found
```

