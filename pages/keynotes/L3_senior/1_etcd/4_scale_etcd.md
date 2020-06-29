---
title: 伸缩etcd集群
keywords: keynotes, senior, etcd, scale_etcd
permalink: keynotes_L3_senior_1_etcd_4_scale_etcd.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/4_scale_etcd
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 向etcd集群中添加节点
- 从etcd集群中删除节点
- etcd集群的备份与恢复
- etcd集群的升级



## 1. 环境

| IP          | HOST   |
| ----------- | ------ |
| 10.0.11.36  | infra0 |
| 10.0.12.21  | infra1 |
| 10.0.13.126 | infra2 |
| 10.0.12.157 | infra3 |
| 10.0.13.118 | infra4 |

且infra0，infra1，infra2是创建好的etcd集群，方法请看[这里](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_2_install_etcd.html)



## 2. 向etcd集群中添加节点

[官方文档](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#scaling-up-etcd-clusters)中不建议我们向生产系统中添加节点，因为添加etcd节点会损失集群的性能，增加节点并不会增加集群的性能或者是容量。一般来说不建议对etcd集群扩/缩容。更不要配置任何的etcd集群自动扩展策略。我们建议在生产系统中部署五个member的etcd集群。过程参考[github](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/runtime-configuration.md#remove-a-member)。

当然，如果有特殊需求，我们也可以部署7个节点，比如两地三中心的架构，三个DC中分别部署2/2/3个etcd集群。

### 2.1. 添加一个正常工作的etcd节点

一般来说，添加节点有两个步骤：

+ 我们可以通过 [HTTP members API](https://github.com/etcd-io/etcd/blob/master/Documentation/v2/members_api.md)， [gRPC members API](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto),或者是 `etcdctl member add` 命令。当然我们肯定是需要使用etcdctl命令的
+ 使用新的集群配置启动新的节点
+ 同步后，使用新的配置重新启动原有节点

**注意1**：如果原来的etcd使用的是ssl证书在签署的时候，指定了hosts的地址的证书是不能够使用的，比如：

``` bash
cfssl certinfo -csr server.csr
.
.
  "IPAddresses": [
    "127.0.0.1",
    "10.0.11.36",
    "10.0.12.21",
    "10.0.13.126"
  ]
}
```

这样的证书在添加了新的member之后是无法正常通信的，需要重新签署证书，把新的节点加入到证书中。

**注意2**：由于签署的证书只用于数据传输，不用于数据加密，所以即使更换CA，重新签署证书，也是没有问题的。但是，如果我们首次启动etcd集群的时候使用的是非加密方式，后面改成SSL方式启动etcd集群的时候就会报错

**注意3**：json文件无法使用`hosts: [""] `这种方式来通配所有地址（理论上可以行，实际操作上报错）

**注意4**：json文件中可以使用域名来通配一类host，理论上可行，但是没有测试

#### 2.1.1. 环境准备

请参考[环境准备](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_2_install_etcd.html#31-%E5%87%86%E5%A4%87%E7%8E%AF%E5%A2%83)和[下载和解压](https://cloudnative365.github.io/keynotes_L3_senior_1_etcd_2_install_etcd.html#32-%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%8C%85%E5%AE%89%E8%A3%85etcd)

#### 2.1.2. 添加节点

+ 检查节点健康状态

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ 在任意一个etcd节点上执行

  ``` bash
  $ etcdctl member add infra3 --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" --peer-urls=https://10.0.12.157:2380
  
  Member 6a5e9d82c1c1c471 added to cluster f36e4227f36fdc03
  
  ETCD_NAME="infra3"
  ETCD_INITIAL_CLUSTER="infra2=https://10.0.13.126:2380,infra3=https://10.0.12.157:2380,infra1=https://10.0.12.21:2380,infra0=https://10.0.11.36:2380"
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.12.157:2380"
  ETCD_INITIAL_CLUSTER_STATE="existing"
  ```

+ 再查看集群状态

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, unstarted, , https://10.0.12.157:2380, , false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ 在新节点创建配置文件/etc/etcd/etcd.conf

  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=infra3
  HOST_IP=10.0.12.157
  CLUSTER=infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380,infra3=https://10.0.12.157:2380
  CLUSTER_STATE=existing
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```

+ 修改systemd文件/lib/systemd/system/etcd.service

  ``` bash
  [Unit]
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target
  
  [Service]
  Type=notify
  WorkingDirectory=/data/etcd
  EnvironmentFile=-/etc/etcd/etcd.conf
  User=etcd
  # set GOMAXPROCS to number of processors
  ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/etcd/etcd \
            --data-dir ${DATA_DIR} \
            --name ${HOST_NAME} \
            --initial-advertise-peer-urls https://${HOST_IP}:2380 \
            --listen-peer-urls https://${HOST_IP}:2380 \
            --advertise-client-urls https://${HOST_IP}:2379 \
            --listen-client-urls https://127.0.0.1:2379,https://${HOST_IP}:2379 \
            --listen-metrics-urls=http://127.0.0.1:2381 \
            --initial-cluster ${CLUSTER} \
            --initial-cluster-state ${CLUSTER_STATE} \
            --initial-cluster-token ${TOKEN} \
            --client-cert-auth \
            --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
            --cert-file=/etc/kubernetes/pki/etcd/server.pem \
            --key-file=/etc/kubernetes/pki/etcd/server-key.pem \
            --peer-client-cert-auth \
            --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
            --peer-cert-file=/etc/kubernetes/pki/etcd/members.pem \
            --peer-key-file=/etc/kubernetes/pki/etcd/members-key.pem"
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```

+ 通过scp方式把etcd的pki同步到新机器上并修改权限

  ``` bash
  chmod 400 /etc/kubernetes/pki/etcd/*
  chown -R etcd:adm /etc/kubernetes/pki/etcd/
  ```

+ 在新节点上启动etcd

  ``` bash
  systemctl start etcd
  ```

+ 如果报错，就吧数据文件`/data/etcd/member`删除再重启

+ 这时候再list member

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ **注意**：修改每个节点上的/etc/etcd/etcd.conf都需要修改为对应的配置

#### 2.1.3. 添加一个learner etcd节点

从etcd3.4版本之后，etcd知识把节点添加为learner节点（不会参与投票的节点）。这样做是为了让添加新节点更加的安全，较少添加节点过程中集群的宕机时间，我们建议在节点在完全同步数据之前作为learner节点。所以我们增加节点的步骤变成了下面几个

+ 我们可以通过 [HTTP members API](https://github.com/etcd-io/etcd/blob/master/Documentation/v2/members_api.md)， [gRPC members API](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto),或者是 `etcdctl member add --learner` 命令.
+ 使用新的集群配置启动新的节点
+ 使用新的配置重新启动原有节点
+ 使用 [gRPC members API](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto),或者是 `etcdctl member promote`命令来让learner节点成为有投票权的节点。这个时候etcd集群会检查一下promote的请求来确认这个操作是安全的。他会去确认learner节点确实已经同步了leader数据的节点，然后才让这个节点拥有投票权。

#### 2.1.4. 添加learner etcd节点具体操作如下

+ 检查节点健康状态

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ 在任意一个etcd节点上执行

  ```  bash
  $ etcdctl member add infra4 --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" --peer-urls=https://10.0.13.118:2380 --learner
  Member d11c0954aee6f056 added to cluster f36e4227f36fdc03
  
  ETCD_NAME="infra4"
  ETCD_INITIAL_CLUSTER="infra2=https://10.0.13.126:2380,infra3=https://10.0.12.157:2380,infra1=https://10.0.12.21:2380,infra4=https://10.0.13.118:2380,infra0=https://10.0.11.36:2380"
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.13.118:2380"
  ETCD_INITIAL_CLUSTER_STATE="existing"
  ```

+ 再查看集群状态

  ``` bash
  etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  
  249badc16d370eb0, unstarted, , https://10.0.13.118:2380, , true
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ 在新节点创建配置文件/etc/etcd/etcd.conf

  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=infra4
  HOST_IP=10.0.13.118
  CLUSTER=infra0=https://10.0.11.36:2380,infra1=https://10.0.12.21:2380,infra2=https://10.0.13.126:2380,infra3=https://10.0.12.157:2380,infra4=https://10.0.13.118:2380
  CLUSTER_STATE=existing
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```

+ 修改systemd文件/lib/systemd/system/etcd.service

  ``` bash
  [Unit]
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target
  
  [Service]
  Type=notify
  WorkingDirectory=/data/etcd
  EnvironmentFile=-/etc/etcd/etcd.conf
  User=etcd
  # set GOMAXPROCS to number of processors
  ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/etcd/etcd \
            --data-dir ${DATA_DIR} \
            --name ${HOST_NAME} \
            --initial-advertise-peer-urls https://${HOST_IP}:2380 \
            --listen-peer-urls https://${HOST_IP}:2380 \
            --advertise-client-urls https://${HOST_IP}:2379 \
            --listen-client-urls https://127.0.0.1:2379,https://${HOST_IP}:2379 \
            --listen-metrics-urls=http://127.0.0.1:2381 \
            --initial-cluster ${CLUSTER} \
            --initial-cluster-state ${CLUSTER_STATE} \
            --initial-cluster-token ${TOKEN} \
            --client-cert-auth \
            --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
            --cert-file=/etc/kubernetes/pki/etcd/server.pem \
            --key-file=/etc/kubernetes/pki/etcd/server-key.pem \
            --peer-client-cert-auth \
            --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
            --peer-cert-file=/etc/kubernetes/pki/etcd/members.pem \
            --peer-key-file=/etc/kubernetes/pki/etcd/members-key.pem"
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```

+ 通过scp方式把etcd的pki同步到新机器上并修改权限

  ``` bash
  $ chmod 400 /etc/kubernetes/pki/etcd/*
  $ chown -R etcd:adm /etc/kubernetes/pki/etcd/
  ```

+ 启动etcd

  ``` bash
  $ systemctl start etcd
  ```

+ 再查看集群状态，发现infra4的learner是true

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  12d20db544264be2, started, infra4, https://10.0.13.118:2380, https://10.0.13.118:2379, true
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ 查看集群状态，集群状态的显示结果为`The items in the lists are endpoint, ID, version, db size, is leader, is learner, raft term, raft index, raft applied index, errors.`

  ``` bash
  $ etcdctl endpoint status --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" --endpoints=https://127.0.0.1:2379 --cluster
  https://10.0.13.118:2379, 12d20db544264be2, 3.4.9, 16 kB, false, true, 4232, 26, 26,
  https://10.0.13.126:2379, 466035bfe5ac7a64, 3.4.9, 20 kB, false, false, 4232, 26, 26,
  https://10.0.12.157:2379, 6a5e9d82c1c1c471, 3.4.9, 20 kB, false, false, 4232, 26, 26,
  https://10.0.12.21:2379, 784e0050552d81cd, 3.4.9, 20 kB, false, false, 4232, 26, 26,
  https://10.0.11.36:2379, f7d6895384ae86d2, 3.4.9, 20 kB, true, false, 4232, 26, 26,
  ```

+ 当信息同步之后，我们就可以开始promote了

  ``` bash
  $ etcdctl member promote 12d20db544264be2 --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"
  Member 12d20db544264be2 promoted in cluster f36e4227f36fdc03
  ```

+ 再查看集群状态

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  12d20db544264be2, started, infra4, https://10.0.13.118:2380, https://10.0.13.118:2379, false
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ **注意**：修改每个节点上的/etc/etcd/etcd.conf都需要修改为对应的配置

### 2.2. 如果添加节点出现问题

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

### 2.3. 如果添加learner节点出现问题

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

## 3. 移除节点

+ 查看集群状态

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  12d20db544264be2, started, infra4, https://10.0.13.118:2380, https://10.0.13.118:2379, false
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ 从集群中删除节点infra4

  ``` bash
  $ etcdctl member remove 12d20db544264be2 --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem"
  Member 12d20db544264be2 removed from cluster f36e4227f36fdc03
  ```

+ 查看状态

  ``` bash
  $ etcdctl member --cacert="/etc/kubernetes/pki/etcd/ca.pem" --cert="/etc/kubernetes/pki/etcd/members.pem" --key="/etc/kubernetes/pki/etcd/members-key.pem" list
  466035bfe5ac7a64, started, infra2, https://10.0.13.126:2380, https://10.0.13.126:2379, false
  6a5e9d82c1c1c471, started, infra3, https://10.0.12.157:2380, https://10.0.12.157:2379, false
  784e0050552d81cd, started, infra1, https://10.0.12.21:2380, https://10.0.12.21:2379, false
  f7d6895384ae86d2, started, infra0, https://10.0.11.36:2380, https://10.0.11.36:2379, false
  ```

+ **注意**：修改每个节点上的/etc/etcd/etcd.conf都需要修改为对应的配置

