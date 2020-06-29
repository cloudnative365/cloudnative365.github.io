---
title: etcd备份与恢复
keywords: keynotes, senior, etcd, backup_and_restore_etcd
permalink: keynotes_L3_senior_1_etcd_6_backup_and_restore_etcd.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/6_backup_and_restore_etcd
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

灾备原理

怎样备份

怎样恢复

## 1. 灾备原理

etcd设计用于承受机器故障。etcd集群自动从临时故障（例如，机器重新启动）中恢复，并且对N个成员的集群最多可容忍`（N-1）/2`个永久故障。当一个成员永久性地失败时，无论是由于硬件故障还是磁盘损坏，它都将失去对集群的访问。如果集群永久失去超过`（N-1）/2`个成员，那么它将无法使用，永久地失去仲裁能力。一旦仲裁丢失，群集就无法达成共识，因此无法继续接受更新。

为了从灾难性故障中恢复，etcd v3提供了快照和恢复功能，以便在不丢失etcd v3 key数据的情况下重新创建集群。

也就是说，etcd的备份实际上是备份了某个key在某个时间的状态，也就是备份的内容是整个的keyspace

## 2. 备份

刚才说过了，备份实际上就是对于keyspace的snapshot。而恢复集群首先需要的是从etcd集群成员中对于keyspace的snapshot。快照可以是使用`etcdctl snapshot save`命令导出的文件，或者是从etcd数据文件夹中`member/snap/db`复制出来的db文件。我们可以使用下面的命令对etcd数据库打快照。

``` bash
$ ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshot.db
```

当然，我们这里使用的是带ssl证书的，应该使用下面的命令

``` bash
ETCDCTL_API=3 etcdctl --endpoints https://10.0.13.126:2379 --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" snapshot save /tmp/snapshot.db
{"level":"info","ts":1592547629.6369183,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/tmp/snapshot.db.part"}
{"level":"info","ts":"2020-06-19T06:20:29.643Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1592547629.6432784,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://10.0.13.126:2379"}
{"level":"info","ts":"2020-06-19T06:20:29.644Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1592547629.6491425,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://10.0.13.126:2379","size":"20 kB","took":0.012103202}
{"level":"info","ts":1592547629.6492367,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/tmp/snapshot.db"}
Snapshot saved at /tmp/snapshot.db
```

这样我们就得到了snapshot文件，`/tmp/snapshot.db`

## 3. 恢复

要恢复集群，只需要一个快照“db”文件。使用`etcdctl snapshot restore`的群集还原将创建新的etcd数据目录；所有成员都应使用同一快照还原。还原将覆盖某些快照元数据（特别是成员ID和群集ID）；该成员将丢失其以前的标识。此元数据覆盖可防止新成员无意中加入现有群集。因此，要从快照启动群集，还原必须启动新的逻辑群集。

注意

+ 快照完整性可以在还原时进行验证。如果使用`etcdctl snapshot save`拍摄快照，则它将具有由`etcdctl snapshot restore`检查的完整性哈希。
+ 如果快照是从数据目录复制的，则不存在完整性哈希，它只能使用`--skip hash check`进行还原。

#### 步骤如下

+ 我们首先向etcd中放点数据

``` bash
$ etcdctl put name jormun --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"

$ etcdctl get name --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"
name
jormun
```

+ 然后备份一下

``` bash
$ ETCDCTL_API=3 etcdctl --endpoints https://10.0.13.126:2379 --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" snapshot save /tmp/snap.db
{"level":"info","ts":1592548731.7236383,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/tmp/snap.db.part"}
{"level":"info","ts":"2020-06-19T06:38:51.727Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1592548731.7275999,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://10.0.13.126:2379"}
{"level":"info","ts":"2020-06-19T06:38:51.730Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1592548731.7342563,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://10.0.13.126:2379","size":"20 kB","took":0.010554193}
{"level":"info","ts":1592548731.7343948,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/tmp/snap.db"}
Snapshot saved at /tmp/snap.db
```

+ 然后给name新的值

``` bash
$ etcdctl put name jormun_new --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"
OK

$ etcdctl get name --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"
name
jormun_new
```

+ 现在我们停掉三台etcd

``` bash
$ systemctl stop etcd
```

+ 然后在三个节点上分别恢复

+ **注意**：还原将使用etcd的群集配置标志使用新的群集配置初始化新群集的新成员，但保留etcd keyspace的内容。从上一个示例继续，下面为三个集群成员创建新的etcd数据目录（infra0.etcd、infra1.etcd、infra2.etcd），且就在执行命令的目录，也就是当前目录`pwd`下创建文件夹。我是在/data目录下执行的

``` bash
$ ETCDCTL_API=3 etcdctl snapshot restore /tmp/snap.db \
   --name infra0 \
   --initial-cluster infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380 \
   --initial-cluster-token ea8cfe2bfe85b7e6c66fe190f9225838 \
   --initial-advertise-peer-urls https://10.0.11.36:2380 \
   --cacert="/etc/kubernetes/pki/etcd/ca.pem" \
   --cert="/etc/kubernetes/pki/etcd/members.pem" \
   --key="/etc/kubernetes/pki/etcd/members-key.pem"
   
{"level":"info","ts":1592805749.46045,"caller":"snapshot/v3_snapshot.go:296","msg":"restoring snapshot","path":"/tmp/snap.db","wal-dir":"infra0.etcd/member/wal","data-dir":"infra0.etcd","snap-dir":"infra0.etcd/member/snap"}
{"level":"info","ts":1592805749.467868,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"f36e4227f36fdc03","local-member-id":"0","added-peer-id":"466035bfe5ac7a64","added-peer-peer-urls":["https://10.0.13.126:2380"]}
{"level":"info","ts":1592805749.4684293,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"f36e4227f36fdc03","local-member-id":"0","added-peer-id":"784e0050552d81cd","added-peer-peer-urls":["https://10.0.12.21:2380"]}
{"level":"info","ts":1592805749.4684544,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"f36e4227f36fdc03","local-member-id":"0","added-peer-id":"f7d6895384ae86d2","added-peer-peer-urls":["https://10.0.11.36:2380"]}
{"level":"info","ts":1592805749.4759514,"caller":"snapshot/v3_snapshot.go:309","msg":"restored snapshot","path":"/tmp/snap.db","wal-dir":"infra0.etcd/member/wal","data-dir":"infra0.etcd","snap-dir":"infra0.etcd/member/snap"}
```

+ 这个时候会在/data下面生成一个新的文件，叫infra0.etcd

``` bash
$ ls /data/
etcd  infra0.etcd
```

+ 修改启动文件/etc/etcd/etcd.conf，修改数据文件位置

``` bash
# DATA_DIR=/data/etcd
DATA_DIR=/data/infra0.etcd
```

+ 还原将使用etcd的群集配置标志使用新的群集配置初始化新群集的新成员，但保留etcd keyspace的内容。从上一个示例继续，需要为每个集群成员创建新的etcd数据目录（/data/infra0.etcd，/data/infra1.etcd，/data/infra2.etcd）。我们把刚才备份的数据文件放到每一台机器的/tmp下，叫snap.db。
+ on infra1

``` bash
ETCDCTL_API=3 etcdctl snapshot restore /tmp/snap.db \
   --name infra1 \
   --initial-cluster infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380 \
   --initial-cluster-token ea8cfe2bfe85b7e6c66fe190f9225838 \
   --initial-advertise-peer-urls https://10.0.12.21:2380 \
   --cacert="/etc/kubernetes/pki/etcd/ca.pem" \
   --cert="/etc/kubernetes/pki/etcd/members.pem" \
   --key="/etc/kubernetes/pki/etcd/members-key.pem"
```

+ on infra2

``` bash
ETCDCTL_API=3 etcdctl snapshot restore /tmp/snap.db \
   --name infra2 \
   --initial-cluster infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380 \
   --initial-cluster-token ea8cfe2bfe85b7e6c66fe190f9225838 \
   --initial-advertise-peer-urls https://10.0.13.126:2380 \
   --cacert="/etc/kubernetes/pki/etcd/ca.pem" \
   --cert="/etc/kubernetes/pki/etcd/members.pem" \
   --key="/etc/kubernetes/pki/etcd/members-key.pem"
```

+ 修改infra1和infra2的配置文件/etc/etcd/etcd.conf

``` bash
DATA_DIR=/data/infra1.etcd
HOST_NAME=infra1
HOST_IP=10.0.12.21
CLUSTER=infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380
CLUSTER_STATE=new
TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
```

``` bash
DATA_DIR=/data/infra2.etcd
HOST_NAME=infra2
HOST_IP=10.0.13.126
CLUSTER=infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380
CLUSTER_STATE=new
TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
```

+ 修改权限（我这里是按照前面的步骤做的，有可能权限不一样，请根据情况而定）

``` bash
chown -R etcd:adm infra0.etcd
```

``` bash
chown -R etcd:adm infra1.etcd
```

``` bash
chown -R etcd:adm infra2.etcd
```

+ 每台机器上都启动etcd

``` bash
systemctl start etcd
```

+ 此时在查数据库，数据已经回来了

``` bash
etcdctl get name --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"
name
jormun
```

