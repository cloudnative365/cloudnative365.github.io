---
title: etcd调优
keywords: keynotes, senior, etcd, etcd_tuning
permalink: keynotes_L3_senior_1_etcd_5_etcd_tuning.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/5_etcd_tuning
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

理解 etcd 性能

etcd 性能优化

## 1. 理解etcd性能

![image-20200618174108466](/pages/keynotes/L3_senior/1_etcd/pics/5_etcd_tuning/image-20200618174108466.png)

上图是一个标准的 etcd 集群架构简图。可以将 etcd 集群划分成几个核心的部分：例如蓝色的 Raft 层、红色的 Storage 层，Storage 层内部又分为 treeIndex 层和 boltdb 底层持久化存储 key/value 层。它们的每一层都有可能造成 etcd 的性能损失。

+ Raft 。Raft 需要通过网络同步数据，网络 IO 节点之间的 RTT 和 / 带宽会影响 etcd 的性能。除此之外，WAL 也受到磁盘 IO 写入速度影响。

+ Storage 。磁盘 IO fdatasync 延迟会影响 etcd 性能，索引层锁的 block 也会影响 etcd 的性能。除此之外，boltdb Tx 的锁以及 boltdb 本身的性能也将大大影响 etcd 的性能。

+ 其他方面。etcd 所在宿主机的内核参数和 gRPC api 层的延迟，也将影响 etcd 的性能。

## 2. etcd 性能优化

### 2.1. server端性能优化

#### 2.1.1. 硬件方面

server 端在硬件上需要足够的 CPU 和 Memory 来保障 etcd 的运行。其次，作为一个非常依赖于磁盘 IO 的数据库程序，etcd 需要 IO 延迟和吞吐量非常好的 ssd 硬盘，etcd 是一个分布式的 key/value 存储系统，网络条件对它也很重要。最后在部署上，需要尽量将它独立的部署，以防止宿主机的其他程序会对 etcd 的性能造成干扰。可以参考[官方文档](https://etcd.io/docs/v3.4.0/op-guide/hardware/)。

#### 2.1.2. 软件方面

etcd 软件分成很多层，下面根据不同层次进行性能优化的简单介绍。因为我对于开发不是很擅长，想深度了解的同学可以自行访问下面的 GitHub pr 来获取具体的修改代码。

- 首先是针对于 etcd 的**内存索引层**优化：优化内部锁的使用减少等待时间。 原来的实现方式是遍历内部引 BTree 使用的内部锁粒度比较粗，这个锁很大程度上影响了 etcd 的性能，新的优化减少了这一部分的影响，降低了延迟。具体可参照如下链接：https://github.com/coreos/etcd/pull/9511
- 针对于 **lease 规模**使用的优化：优化了 lease revoke 和过期失效的算法，将原来遍历失效 list 时间复杂度从 O(n) 降为 O(logn)，解决了 lease 规模化使用的问题。具体可参照如下链接：[https://github.com/coreos/etcd/pull/9418](https://github.com/coreos/etcd/pull/9418)
- 最后是针对于**后端 boltdb** 的使用优化：将后端的 batch size limit/interval 进行调整，这样就能根据不同的硬件和工作负载进行动态配置，这些参数以前都是固定的保守值。具体可参照如下链接：https://github.com/etcd-io/etcd/commit/3faed211e535729a9dc36198a8aab8799099d0f3
- 还有一点是由谷歌工程师优化的完全并发读特性：优化调用 boltdb tx 读写锁使用，提升读性能。具体可参照如下链接：https://github.com/etcd-io/etcd/pull/10523

#### 2.1.3. Raft log retention

`etcd --snapshot-count`配置压缩前要保存在内存中的应用Raft条目数。当-`--snapshot-count`达到设定的数值时，服务器会将快照数据保存到磁盘上，然后截断旧条目。当速度较慢的follower节点在压缩索引之前请求日志时，leader会发送快照，强制follower覆盖其状态。

如果把 `--snapshot-count`参数调整高一点，在快照之前会在内存中保留更多Raft条目，从而导致经常性的更高内存使用率，[recurrent higher memory usage](https://github.com/kubernetes/kubernetes/issues/60589#issuecomment-371977156)，这是一个issue。由于leader保留最新的Raft条目的时间更长，所以慢的follower在leader快照之前有更多的时间追赶。`--snapshot-count` 的数值应该在较高的内存使用率和较慢的follower的较高可用性之间进行权衡。

从v3.2开始，默认值--snapshot count从10000更改为100000。

在性能方面，`--snapshot-count`大于100000可能会影响写入的吞吐量。内存中对象的数量增加会减慢Go语言GC标记阶段的`runtime.scanobject`，这也是一个issue， [Go GC mark phase `runtime.scanobject`](https://golang.org/src/runtime/mgc.go)。并且不频繁的内存回收会使分配变慢。性能因工作负载和系统环境而异。但是，通常情况下，过于频繁的压缩会影响群集可用性和写入吞吐量。过于频繁的压缩也有害于给垃圾收集器施加过大的压力。具体的结果可以看[slideshare](https://www.slideshare.net/mitakeh/understanding-performance-aspects-of-etcd-and-raft)。

#### 2.1.4. Auto Compaction自动压缩

etcd默认会为key保留一个小时，也就是一个小时之后，key的空间就会被自动压缩

``` bash
# keep one hour of history
$ etcd --auto-compaction-retention=1
```

+ 在v3.0.0和v3.1.0版本，`--auto-compaction-retention=10`在v3版本的key-value上运行，每10小时进行一次压缩。压缩功能只支持阶段性压缩。压缩器会每五分钟记录一次最新的revision，知道达到第一个压缩点（比如10个小时）。为了保留上次压缩时期的键值历史记录，他使用压缩器件之前，从每5分钟收集的revision记录中获取最后的revision。当`--auto-compaction-retention=10`的时候，压缩器使用revision 100作为压缩的revision，其中revison 100是10小时前回去的最新版本。如果压缩成功或请求的修订已被压缩，则他将重置时段计时器并以新的历史revision记录重新开始（例如，重新启动revision收集并压缩下一个10小时的时段）。如果压缩失败，5分钟后会重试。

+ 在3.2.0版本中，压缩器每小时运行一次。压缩器只支持阶段性压缩。压缩器同样是五分钟运行一次，来记录最新一次的revision。在每个小时内，它使用压缩期间之前，每5分钟收集来的revision记录中获取的最后一个revision。也就是说，每小时，压缩程序都会丢弃压缩期间之前创建的历史数据。压缩时期的保持期会移到下一小时。例如，当每小时写入次数为100并且`--auto compression retention=10`时，v3.1每10小时压缩一次revision 1000、2000和3000次，而v3.2.x、v3.3.0、v3.3.1和v3.3.2每1小时压缩revision 1000、1100和1200次。如果压缩成功或请求的revision已被压缩，它将重置时段计时器，并从历史revision记录中删除已使用的压缩revision（例如，从以前收集的revision开始下一个revision收集和压缩）。如果压缩失败，将在5分钟后重试。

+ 在v3.3.0、v3.3.1和v3.3.2中，`--auto-compaction-mode=revision --auto-compaction-retention=1000`在"latest revision"上自动压缩-每5分钟压缩1000（当最新revision为30000时，在revision29000上压缩）。例如，`--auto-compaction-mode=periodic --auto-compaction-retention=72h`在压缩的时候，每7.2小时有72小时的保持时间窗口。例如，`--auto-compaction-mode=periodic --auto-compaction-retention=30m`，每3分钟用30分钟保持时间窗口自动压实。定期压缩机继续记录给定压缩周期的每1/10的最新revision（例如，`--auto-compaction-mode=periodic --auto-compaction-retention=10h`为1小时）。对于给定压缩周期的每1/10，compactor使用压缩周期之前获取的最后一个修订版来丢弃历史数据。每给定压实周期的1/10，压实周期的保留窗口移动一次。例如，当每小时写入次数为100并且`--auto-compaction-retention=10`时，v3.1每10小时压缩一次revision1000、2000和3000，而v3.2.x、v3.3.0、v3.3.1和v3.3.2每1小时压缩一次revision1000、1100和1200。此外，当每分钟写入1000、v3.3.0、v3.3.1和v3.3.2时，对于粒度更细的每3分钟，`--auto-compaction-mode=periodic --auto-compaction-retention=30m`压缩版本30000、33000和36000。

+ 当 `--auto-compaction-retention=10h`的时候, etcd 会先等待10小时进行第一次压缩，然后每小时压缩（1/10个10小时）以此类推

  ``` bash
  0Hr  (rev = 1)
  1hr  (rev = 10)
  ...
  8hr  (rev = 80)
  9hr  (rev = 90)
  10hr (rev = 100, Compact(1))
  11hr (rev = 110, Compact(10))
  ...
  ```

  不管压缩成功与否，这个进程会在给定的1/10个压缩时间段中不断的重复。如果压缩成功，他会把已经压缩的revijsion从历史revision记录中移除。

+ 在v3.3.3版本中，`--auto-compaction-mode=revision --auto-compaction-retention=1000`在`latest revision`上每5分钟就自动压缩（latest revsion为30000时，在29000的revision上压缩）。在以前`--auto-compaction-mode=periodic --auto-compaction-retention=72h`每7.2小时自动压缩，72小时保留窗口一次。现在，每隔1小时就会发生一次压缩，但保留时间仍为72小时。以前，`--auto-compaction-mode=periodic --auto-compaction-retention=30m`，自动压缩时，每3分钟保留30分钟的保留窗口。现在，每30分钟执行一次压缩，但保留时间仍为30分钟。当给定周期小于1小时时，阶段性压缩器会记录每个压缩周期的最新revision，如果给定周期大于1小时（比如： 在1小时内，当`--auto-compaction-mode=periodic --auto-compaction-retention=24h`的时候）。对于每个压实周期或1小时，压缩器会使用在压缩周期之前获取的最新revision来丢弃历史数据。压缩期的保留窗口会在每个给定的压实期或小时内移动。例如，当每小时写入为100并且`--auto-compaction-mode=periodic --auto-compaction-retention=24h`的时候，v3.2.x，v3.3.0，v3.3.1和v3.3.2每2个小时压缩revision2400，2640和2880，而v3.3.3或更高版本每1小时压缩2400、2500、2600版本。此外，当`--auto-compaction-mode=periodic --auto-compaction-retention=30m`且每分钟写入量约为1000，v3.3.0，v3.3.1和v3.3.2压缩30000、33000和36000时， v3.3.3或更高版本每3分钟压缩一次revision30000、60000和90000，每30分钟压缩一次。

#### 2.1.5. Defragmentation去碎片化

压缩键空间后，后端数据库可能会出现内部碎片。 任何内部碎片都是后端可以免费使用但仍消耗存储空间的空间。 通过在后端数据库中留有空隙，内部压缩旧修订会碎片化etcd。 碎片空间可供etcd使用，但对主机文件系统不可用。 换句话说，删除应用程序数据不会回收磁盘上的空间。

碎片整理过程会将此存储空间释放回文件系统。 对每个成员进行碎片整理，以便避免群集范围内的延迟峰值。

要对etcd成员进行碎片整理，请使用`etcdctl defrag`命令：

``` bash
$ etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
```

**请注意**，对活动成员进行碎片整理会阻止系统在重建其状态时读取和写入数据。就是说如果有节点在恢复数据，这个操作会打断的。

如果要对集群做整理，就需要指定--endpoints或者使用--cluster选项

```bash
$ etcdctl defrag --cluster
Finished defragmenting etcd member[http://127.0.0.1:2379]
Finished defragmenting etcd member[http://127.0.0.1:22379]
Finished defragmenting etcd member[http://127.0.0.1:32379]
```

如果etcd没在运行，就需要指定etcd的--data-dir

``` bash
$ etcdctl defrag --data-dir <path-to-etcd-data-dir>
```

#### 2.1.5. Space quota空间限额

etcd中的空间配额可确保集群以可靠的方式运行。 如果没有空间配额，则如果密钥空间过大，etcd可能会遭受性能不佳的影响，或者它可能仅会耗尽存储空间，从而导致无法预测的集群行为。 如果任何成员的密钥空间的后端数据库超出了空间配额，则etcd会发出一个群集范围的警报，该警报会将群集置于维护模式，该模式仅接受密钥读取和删除。 只有在释放键空间中足够的空间并对后端数据库进行碎片整理以及清除空间配额警报之后，集群才能恢复正常运行。但是生产系统上几乎没有人这么做。

默认情况下，etcd设置适合大多数应用程序的保守的空间配额，但是可以在命令行上以字节为单位进行配置：

``` bash
# set a very small 16MB quota
$ etcd --quota-backend-bytes=$((16*1024*1024))
```

可以使用循环来触发他

``` bash
# fill keyspace
$ while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024  | ETCDCTL_API=3 etcdctl put key  || break; done
...
Error:  rpc error: code = 8 desc = etcdserver: mvcc: database space exceeded
# confirm quota space is exceeded
$ ETCDCTL_API=3 etcdctl --write-out=table endpoint status
+----------------+------------------+-----------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        |  VERSION  | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | bf9071f4639c75cc | 2.3.0+git | 18 MB   | true      |         2 |       3332 |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
# confirm alarm is raised
$ ETCDCTL_API=3 etcdctl alarm list
memberID:13803658152347727308 alarm:NOSPACE
```

删除过多的键空间数据并对后端数据库进行碎片整理将使群集重新回到配额限制内：

``` bash
# get current revision
$ rev=$(ETCDCTL_API=3 etcdctl --endpoints=:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
# compact away all old revisions
$ ETCDCTL_API=3 etcdctl compact $rev
compacted revision 1516
# defragment away excessive space
$ ETCDCTL_API=3 etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
# disarm alarm
$ ETCDCTL_API=3 etcdctl alarm disarm
memberID:13803658152347727308 alarm:NOSPACE
# test puts are allowed again
$ ETCDCTL_API=3 etcdctl put newkey 123
OK
```



### 2.2. client端性能优化

+ 针对于 Put 操作避免使用大 value，精简精简再精简，例如 K8s 下的 crd 使用；

+ 其次，etcd 本身适用及存储一些不频繁变动的 key/value 元数据信息。因此客户端在使用上需要避免创建频繁变化的 key/value。这一点例如 K8s下对于新的 node 节点的心跳数据上传就遵循了这一实践；

+ 最后，我们需要避免创建大量的 lease，尽量选择复用。例如在 K8s下，event 数据管理：相同 TTL 失效时间的 event 同样会选择类似的 lease 进行复用，而不是创建新的 lease。