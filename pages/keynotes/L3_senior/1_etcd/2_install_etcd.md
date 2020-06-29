---
title: 安装etcd
keywords: keynotes, senior, etcd, install_etcd
permalink: keynotes_L3_senior_1_etcd_2_install_etcd.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/2_install_etcd
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 安装单机版etcd
- 安装etcd集群
- 配置安全的etcd（配置SSL证书）

## 1. 环境

### 1.1. 软件版本

| 环境     | 版本                                          |
| -------- | --------------------------------------------- |
| 操作系统 | linux大部分发行版都可以（ubuntu/rhel/centos） |
| 内核版本 | 3.10和4.15                                    |
| etcd     | v3.4.9                                        |
| golang   | 1.14.3                                        |

### 1.2. 硬件规划

关于机器选项可以参考[这个](https://etcd.io/docs/v3.4.0/op-guide/hardware/#example-hardware-configurations)

Here are a few example hardware setups on AWS and GCE environments. As mentioned before, but must be stressed regardless, administrators should test an etcd deployment with a simulated workload before putting it into production.

Note that these configurations assume these machines are totally dedicated to etcd. Running other applications along with etcd on these machines may cause resource contentions and lead to cluster instability.

### Small cluster

A small cluster serves fewer than 100 clients, fewer than 200 of requests per second, and stores no more than 100MB of data.

Example application workload: A 50-node Kubernetes cluster

| Provider | Type                        | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
| :------- | :-------------------------- | :---- | :---------- | :------------------ | :-------------------- |
| AWS      | m4.large                    | 2     | 8           | 3600                | 56.25                 |
| GCE      | n1-standard-2 + 50GB PD SSD | 2     | 7.5         | 1500                | 25                    |

### Medium cluster

A medium cluster serves fewer than 500 clients, fewer than 1,000 of requests per second, and stores no more than 500MB of data.

Example application workload: A 250-node Kubernetes cluster

| Provider | Type                         | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
| :------- | :--------------------------- | :---- | :---------- | :------------------ | :-------------------- |
| AWS      | m4.xlarge                    | 4     | 16          | 6000                | 93.75                 |
| GCE      | n1-standard-4 + 150GB PD SSD | 4     | 15          | 4500                | 75                    |

### Large cluster

A large cluster serves fewer than 1,500 clients, fewer than 10,000 of requests per second, and stores no more than 1GB of data.

Example application workload: A 1,000-node Kubernetes cluster

| Provider | Type                         | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
| :------- | :--------------------------- | :---- | :---------- | :------------------ | :-------------------- |
| AWS      | m4.2xlarge                   | 8     | 32          | 8000                | 125                   |
| GCE      | n1-standard-8 + 250GB PD SSD | 8     | 30          | 7500                | 125                   |

### xLarge cluster

An xLarge cluster serves more than 1,500 clients, more than 10,000 of requests per second, and stores more than 1GB data.

Example application workload: A 3,000 node Kubernetes cluster

| Provider | Type                          | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
| :------- | :---------------------------- | :---- | :---------- | :------------------ | :-------------------- |
| AWS      | m4.4xlarge                    | 16    | 64          | 16,000              | 250                   |
| GCE      | n1-standard-16 + 500GB PD SSD | 16    | 60          | 15,000              | 250                   |

## 2. 安装单机版etcd

### 2.1. 二进制包安装etcd

+ 下载和解压

  [这里](https://github.com/etcd-io/etcd/releases/)有下载的向导，也可以点这里直接下载[v3.4.9](https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz)
  
  ``` bash
  #设置要下载的版本
  ETCD_VER=v3.4.9
  INSTALL_DIR=/opt
  
  # 也可以从google下载，鉴于国内无法访问，就注释掉了
  # GOOGLE_URL=https://storage.googleapis.com/etcd
  GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
  DOWNLOAD_URL=${GITHUB_URL}
  
  # 清理原来下载过的
  rm -f ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
  rm -rf ${INSTALL_DIR}/etcd && mkdir -p ${INSTALL_DIR}/etcd
  
  # 下载
  curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
  # 解压
  tar xzvf ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz -C ${INSTALL_DIR}/etcd --strip-components=1
  # 删除压缩包
  rm -f ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
  
  # 测试
  ${INSTALL_DIR}/etcd/etcd --version
  ${INSTALL_DIR}/etcd/etcdctl version
  
  ```

+ 启动etcd

  ``` bash
  ./etcd
  ```

### 2.2. 编译安装etcd

+ 官方文档在[这里](https://etcd.io/docs/v3.4.0/dl-build/)

+ 注意：编译安装的话需要安装1.14版本及以上的golang环境，下载golang的二进制包，配置GOROOT和GOPATH

+ 注意：官网要求golang版本是1.13以上（也就是1.14版本），目前的镜像只有RHEL8.2和AMAZON linux2的yum源中的golang是1.13版本，换句话说，截止2020年5月27日，如果想编译安装etcd，必须自己手动配置golang环境

+ 注意：目前版本上使用编译后的etcd创建集群的时候会出现错误，目前测试在AWS的linux2上会出现这个问题，所以只有二进制方式安装最保险

  ``` bash
  panic: runtime error: invalid memory address or nil pointer dereference
  ```

  

+ 下载golang

  ``` bash
  # 国内无法访问google，请使用下面的链接下载二进制包
  wget https://studygolang.com/dl/golang/go1.14.3.linux-amd64.tar.gz
  # 解压
  tar xf go1.14.3.linux-amd64.tar.gz
  # 验证
  ./go/bin/go version
  go version go1.14.3 linux/amd64
  ```

+ 配置go环境

  ``` bash
  cat << EOF > /etc/profile.d/golong.sh
  export GOROOT=/opt/go
  export PATH=$PATH:/opt/go/bin
  EOF
  
  source /etc/profile
  ```

+ 编译

  ``` bash
  $ git clone https://github.com/etcd-io/etcd.git
  $ cd etcd
  $ go env -w GOPROXY=https://goproxy.cn,direct
  $ go mod vendor
  $ ./build
  ```

+ 验证

  ``` bash
  # 在bin目录下面会多出两个可执行文件
  ls bin/
  etcd  etcdctl
  
  # 查看版本
  $ ./bin/etcd --version
  etcd Version: 3.5.0-pre
  Git SHA: 9b6c3e337
  Go Version: go1.14.3
  Go OS/Arch: linux/amd64
  
  $ ./bin/etcdctl version
  etcdctl version: 3.5.0-pre
  API version: 3.5
  ```

  



## 3. 安装etcd集群

如果是简单的demo，我们可以参考[官方文档](https://etcd.io/docs/v3.4.0/demo/)。这里介绍的是在生产环境上搭建etcd集群。

### 3.1. 准备环境

+ 配置时间服务，ntpd和chrony都可以

+ 为etcd创建独立的文件系统，在公有云环境中，系统基本都是镜像启动的，实例被干掉之后容易丢数据，而且速度不如外挂存储卷，且稳定性好。

  注意：不管我们的数据放在哪里，都有丢失的危险，一定要记得备份！etcd再轻量也是数据库，数据丢了，什么都没了

  ``` bash
  $ lvcreate -n lv_etcd -L 10G vg_system
  $ mkfs.xfs /dev/mapper/vg_system-lv_etcd
  $ mkdir -p /data/etcd
  ```

+ 权限控制

  ``` bash
  # 创建etcd用户，只用来跑程序
  $ useradd etcd
  # 修改etcd的主要组为adm，便于同属于adm组的管理员查看，并且指定不可以登录
  $ usermod -g adm -s nologin etcd
  # 修改数据的权限，etcd用户拥有所有的权限
  # etcd所在的组adm拥有读和执行的权限，方便管理员查看或者备份，也可以给他只读权限
  # 其他人员没有权限
  # root拥有所有权限
  $ chmod 740 /data/etcd
  $ chown etcd:adm /data/etcd
  $ mkdir /etc/etcd
  $ chmod 740 /data/etcd
  $ chown etcd:adm /etc/etcd
  ```

  

### 3.2. 二进制包安装etcd

+ 下载和解压

  ``` bash
  #设置要下载的版本
  ETCD_VER=v3.4.9
  INSTALL_DIR=/opt
  
  # 也可以从google下载，鉴于国内无法访问，就注释掉了
  # GOOGLE_URL=https://storage.googleapis.com/etcd
  GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
  DOWNLOAD_URL=${GITHUB_URL}
  
  # 清理原来下载过的
  rm -f ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
  rm -rf ${INSTALL_DIR}/etcd && mkdir -p ${INSTALL_DIR}/etcd
  
  # 下载
  curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
  # 解压
  tar xzvf ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz -C ${INSTALL_DIR}/etcd --strip-components=1
  # 删除压缩包
  rm -f ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
  
  # 测试
  ${INSTALL_DIR}/etcd/etcd --version
  ${INSTALL_DIR}/etcd/etcdctl version
  ```

+ 配置etcd的路径

  ``` bash
  cat << EOF > /etc/profile.d/etcd.sh
  export PATH=$PATH:/opt/etcd
  EOF
  
  source /etc/profile
  ```

+ 生成一个长一点的token保证安全

  ``` bash
  $ echo k8s-cluster|md5sum
  ea8cfe2bfe85b7e6c66fe190f9225838  -
  ```

+ 配置文件/etc/etcd/etcd.conf

  master1

  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=master1
  HOST_IP=10.0.1.204
  CLUSTER=master1=http://10.0.1.204:2380,master2=http://10.0.1.67:2380,master3=http://10.0.1.236:2380
  CLUSTER_STATE=new
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```
  
  master2
  
  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=master2
HOST_IP=10.0.1.67
  CLUSTER=master1=http://10.0.1.204:2380,master2=http://10.0.1.67:2380,master3=http://10.0.1.236:2380
CLUSTER_STATE=new
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```
  
  master3
  
  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=master3
  HOST_IP=10.0.1.236
  CLUSTER=master1=http://10.0.1.204:2380,master2=http://10.0.1.67:2380,master3=http://10.0.1.236:2380
  CLUSTER_STATE=new
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```
  

  
+ 编辑systemd服务文件 /usr/lib/systemd/etcd.service(rhel系列的)或者/lib/systemd/system/etcd.service(ubuntu系列)

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
            --data-dir ${DATA_DIR} \--name \"${HOST_NAME}\" \
            --initial-advertise-peer-urls http://${HOST_IP}:2380 \
            --listen-peer-urls http://${HOST_IP}:2380 \
            --advertise-client-urls http://${HOST_IP}:2379 \
            --listen-client-urls http://${HOST_IP}:2379 \
            --initial-cluster ${CLUSTER} \
            --initial-cluster-state ${CLUSTER_STATE} \
            --initial-cluster-token ${TOKEN}"
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```
  
+ 启动

  ``` bash
  systemctl daemon-reload
  systemctl start etcd
  ```

+ 查看状态

  ``` bash
  etcdctl endpoint health
  ```
  
  注意：etcdctl endpoint health命令不加参数的话，默认是访问本地的2379端口，也就是127.0.0.1：2379，但是咱们刚才配置集群的时候是没有监听本地端口的，所以要使用`--endpoint`命令指定端口。否则会报错
  
  ``` bash
  127.0.0.1:2379 is unhealthy: failed to commit proposal: context deadline exceeded
  ```
  
  可悲的是，如果使用endpoint参数，就必须使用https协议，也就是必须使用证书
  
  ``` bash
  etcdctl --endpoints=https://10.0.1.236:2379 --cacert=/etc/k8s/ssl/etcd-root-ca.pem --key=/etc/k8s/ssl/etcd-key.pem  --cert=/etc/k8s/ssl/etcd.pem  endpoint health 
  ```
  
  我们目前还没配置证书，只能从日志中查看etcd的状态是否正常
  
  ``` bash
  journalctl -u etcd
  ```
  
  而etcd的输出位置是没有参数去指定的，他的默认输出是stdout，会由journald来管理
  
  ``` bash
  Logging:
    --logger 'capnslog'
      Specify 'zap' for structured logging or 'capnslog'. [WARN] 'capnslog' will be deprecated in v3.5.
    --log-outputs 'default'
      Specify 'stdout' or 'stderr' to skip journald logging even when running under systemd, or list of comma separated output targets.
    --log-level 'info'
      Configures log level. Only supports debug, info, warn, error, panic, or fatal.
  ```
  
  

## 4. 配置安全的etcd

### 4.1. 制作证书

[制作证书](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)目前普遍使用这三种工具，`easyrsa`, `openssl` 或者 `cfssl`，我们这里使用cfssl，参考[github](https://github.com/coreos/docs/blob/master/os/generate-self-signed-certificates.md)

#### 4.1.1. 准备环境

+ 下载cfssl工具

  ``` bash
  curl -s -L -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  curl -s -L -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  chmod +x /usr/local/bin/{cfssl,cfssljson}
  ```

+ 测试一下

  ``` bash
  $ cfssl
  No command is given.
  Usage:
  Available commands:
  	serve
  	version
  	genkey
  	gencrl
  	ocsprefresh
  	selfsign
  	scan
  	print-defaults
  	revoke
  	bundle
  	sign
  	gencert
  	ocspdump
  	ocspserve
  	info
  	certinfo
  	ocspsign
  Top-level flags:
    -allow_verification_with_non_compliant_keys
      	Allow a SignatureVerifier to use keys which are technically non-compliant with RFC6962.
    -loglevel int
      	Log level (0 = DEBUG, 5 = FATAL) (default 1)
  ```

+ 创建工作目录

  ``` bash
  $ mkdir -p /etc/kubernetes/pki/etcd
  ```



#### 4.1.2. 创建CA相关证书

+ 创建CA配置文件（默认创建）

  ``` bash
  cd /etc/kubernetes/pki/etcd
  cfssl print-defaults config > ca-config.json
  cfssl print-defaults csr > ca-csr.json
  ```

+ 修改`ca-config.json`为

  ``` json
  {
      "signing": {
          "default": {
              "expiry": "43800h"
          },
          "profiles": {
              "server": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth"
                  ]
              },
              "client": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "client auth"
                  ]
              },
              "peer": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  ```

+ 修改`ca-csr.json`为

  ```json
  {
      "CN": "My own CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "US",
              "L": "CA",
              "O": "My Company Name",
              "ST": "San Francisco",
              "OU": "Org Unit 1",
              "OU": "Org Unit 2"
          }
      ]
  }
  ```

+ 创建CA的证书

  ``` bash
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
  ```

+ 会得到三个文件

  ``` bash
  ca-key.pem
  ca.csr
  ca.pem
  ```



#### 4.1.3. 创建服务器（server）相关证书

+ 生成配置文件

  ``` bash
  cfssl print-defaults csr > server.json
  ```

+ 修改server.json中CN和host的部分

  ``` json
  {
      "CN": "etcd",
      "hosts": [
          "127.0.0.1",
          "10.0.1.204",
          "10.0.1.67",
          "10.0.1.236"
      ],
      "key": {
          "algo": "ecdsa",
          "size": 256
      },
      "names": [
          {
              "C": "US",
              "L": "CA",
              "ST": "San Francisco"
          }
      ]
  }
  ```

+ 生成证书

  ``` bash
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
  ```

+ 同样是三个文件（这个是服务器启动时候的证书）

  ``` bash
  server-key.pem
  server.csr
  server.pem
  ```

#### 4.1.4. 创建服务器互相通讯（peer）的相关证书

+ 生成配置文件

  ``` bash
  cfssl print-defaults csr > peer.json
  ```

+ 修改server.json中CN和host的部分

  ``` json
  {
      "CN": "peer",
      "hosts": [
          "127.0.0.1",
          "10.0.1.204",
          "10.0.1.67",
          "10.0.1.236"
      ],
      "key": {
          "algo": "ecdsa",
          "size": 256
      },
      "names": [
          {
              "C": "US",
              "L": "CA",
              "ST": "San Francisco"
          }
      ]
  }
  ```

+ 生成证书

  ``` bash
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer members.json | cfssljson -bare members
  ```

+ 三个文件

  ``` bash
  peer-key.pem
  peer.csr
  peer.pem
  ```

#### 4.1.5. 创建客户端（client）的相关证书

+ 生成配置文件

  ``` bash
  cfssl print-defaults csr > client.json
  ```

+ 修改server.json中CN和host的部分

  注意：一般来说，如果etcd需要手动创建的话，架构上会把这三台etcd独立拿出来作为数据库来管理，所以客户端会的hosts是etcd之外的IP地址，但是我们这里实验是使用这三台etcd作为客户端的，所以地址还是这三台机器。如果实在不明白，就把所有的机器ip**和**DNS名称都写在这个hosts里面，或者让hosts留空(下面的例子)，防止出错，不过这样并不算最安全的选择。

  ``` json
  {
      "CN": "client",
      "hosts": [""],
      "key": {
          "algo": "ecdsa",
          "size": 256
      },
      "names": [
          {
              "C": "US",
              "L": "CA",
              "ST": "San Francisco"
          }
      ]
  }
  ```

+ 生成证书

  ``` bash
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
  ```

+ 又得到一组证书

  ``` bash
  client-key.pem
  client.csr
  client.pem
  ```

### 4.2. 配置etcd使用ssl证书

+ 把刚才生成的所有证书(在/etc/kubernetes/pki/etcd下的所有文件)都复制到另外的etcd机器上去。

+ 修改启动文件

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
            --peer-cert-file=/etc/kubernetes/pki/etcd/peer.pem \
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```

+ 修改配置文件，/etc/etcd/etcd.conf

  master1

  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=master1
  HOST_IP=10.0.1.204
  CLUSTER=master1=https://10.0.1.204:2380,master2=https://10.0.1.67:2380,master3=https://10.0.1.236:2380
  CLUSTER_STATE=new
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```

  master2

  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=master2
  HOST_IP=10.0.1.67
  CLUSTER=master1=https://10.0.1.204:2380,master2=https://10.0.1.67:2380,master3=https://10.0.1.236:2380
  CLUSTER_STATE=new
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```

  master3

  ``` bash
  DATA_DIR=/data/etcd
  HOST_NAME=master3
  HOST_IP=10.0.1.236
  CLUSTER=master1=https://10.0.1.204:2380,master2=https://10.0.1.67:2380,master3=https://10.0.1.236:2380
  CLUSTER_STATE=new
  TOKEN=ea8cfe2bfe85b7e6c66fe190f9225838
  ```

+ 修改权限

  ``` bash
  chmod 400 /etc/kubernetes/pki/etcd/*
  chown -R etcd:adm /etc/kubernetes/pki/etcd/
  ```

+ 重启启动集群会报错

  ``` bash
  error "tls: first record does not look like a TLS handshake
  ```

  删除数据文件，重新启动就好了


