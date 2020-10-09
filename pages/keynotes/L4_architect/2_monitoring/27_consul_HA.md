---
title: consul高可用
keywords: keynotes, L4_architect, monitoring, consul_HA
permalink: keynotes_L4_architect_2_monitoring_27_consul_HA.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/27_consul_HA
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

consul从数据存储形式来看，是kv存储，但是我们更喜欢叫做注册中心。和etcd，consul，zookeeper一样，是目前比较流行的解决方案。我们上次搭建了一个单机版的，用他来做prometheus的注册中心。我们通过curl方式来注册和管理服务，然后我们只需要在prometheus中配置consul为数据源，prometheus就可以动态更新endpoint信息，而不用重新启动服务了。

## 2. 架构

### 2.1. 架构图

![consul_ha](/pages/keynotes/L4_architect/2_monitoring/pics/27_consul_HA/consul_ha.png)

### 3.2. server

由若干的server组成，其中一个是leader，其他的是follower，为了保证集群的基本功能，我们建议最少有3个server端，因为看名字就能看出来，consul集群是典型的raft方式，所以建议使用至少3个节点。

而在server运行的同时，还会有一个agent运行。但是这个agent不是负责服务发现或者设置和获取kv，而是负责自身节点和节点上服务运行状况进行健康检查的。

### 3.3. client

我们上次用单机的方式搭建了一个consul的server，其实consul还有另外一个方式，就是client。我们可以向server端发起请求，注册服务，也可以去找client，实际上client更像是一个proxy，或者是ES中的master节点。如下图所示：

![20190318150146698](/pages/keynotes/L4_architect/2_monitoring/pics/27_consul_HA/20190318150146698.png)

## 4. 集群搭建

### 4.1. 准备

+ 下载地址：[点这里](https://www.consul.io/downloads)。

  ``` bash
  wget https://releases.hashicorp.com/consul/1.8.4/consul_1.8.4_linux_amd64.zip
  ```

+ 下载完成后解压

  ``` bash
  unzip consul_1.8.4_linux_amd64.zip
  ```

+ 由于consul是go语言编写的，解压之后只有一个文件，我们可以把他放在/usr/local/bin下面

  ``` bash
  mv consul /usr/local/bin/
  ```

+ 需要开放的端口

  + **8300**：服务端RPC，TCP。
  + **8301**：Serl LAN：处理LAN gossip，TCP UDP。
  + **8302**：Serl WAN：处理LAN gossip，TCP UDP。
  + **8500**：HTTP API，TCP.
  + **8600**：DNS，TCP,UDP

+ 创建目录

  ``` bash
  mkdir -p /data/consul/{data,log}
  mkdir /etc/consul
  ```

### 4.2. 启动集群

+ 在第一个节点172.16.0.1上

  + 创建配置文件`/etc/consul/server.json`

    ``` json
    {
    "data_dir": "/data/consul/data",
    "log_file": "/data/consul/log/consul.log",
    "log_level": "INFO",
    "log_rotate_duration": "24h",
    "node_name": "node1",
    "server": true,
    "bootstrap_expect": 3,
    "client_addr": "0.0.0.0",
    "advertise_addr": "172.16.0.1"
    }
    ```

  + 创建systemd文件`/lib/systemd/system/consul-server.service`

    ``` bash
    [Unit]
    Description=Consul service
    Documentation=https://www.consul.io/docs/
    
    [Service]
    ExecStart=/usr/local/bin/consul agent -ui -config-dir /etc/consul
    KillSignal=SIGINT
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    ```

  + 启动服务

    ``` bash
    systemctl start consul-server
    ```

+ 在第二个节点172.16.0.2上

  + 创建配置文件`/etc/consul/server.json`

    ``` json
    {
    "data_dir": "/data/consul/data",
    "log_file": "/data/consul/log/consul.log",
    "log_level": "INFO",
    "log_rotate_duration": "24h",
    "node_name": "node2",
    "server": true,
    "bootstrap_expect": 3,
    "client_addr": "0.0.0.0",
    "advertise_addr": "172.16.0.2"
    }
    ```

  + 创建systemd文件`/lib/systemd/system/consul-server.service`

    ``` bash
    [Unit]
    Description=Consul service
    Documentation=https://www.consul.io/docs/
    
    [Service]
    ExecStart=/usr/local/bin/consul agent -ui -config-dir /etc/consul
    KillSignal=SIGINT
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    ```

  + 启动服务

    ``` bash
    systemctl start consul-server
    ```

  + 加入第一个节点的集群

    ``` bash
    consul join 172.16.0.1
    ```

+ 在第三个节点172.16.0.3上

  + 创建配置文件`/etc/consul/server.json`

    ``` json
    {
    "data_dir": "/data/consul/data",
    "log_file": "/data/consul/log/consul.log",
    "log_level": "INFO",
    "log_rotate_duration": "24h",
    "node_name": "node3",
    "server": true,
    "bootstrap_expect": 3,
    "client_addr": "0.0.0.0",
    "advertise_addr": "172.16.0.3"
    }
    ```

  + 创建systemd文件`/lib/systemd/system/consul-server.service`

    ``` bash
    [Unit]
    Description=Consul service
    Documentation=https://www.consul.io/docs/
    
    [Service]
    ExecStart=/usr/local/bin/consul agent -ui -config-dir /etc/consul
    KillSignal=SIGINT
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    ```

  + 启动服务

    ``` bash
    systemctl start consul-server
    ```

  + 加入第一个节点的集群

    ``` bash
    consul join 172.16.0.1
    ```

+ 检查一下consul的成员

  ``` bash
  # consul members
  Node   Address          Status  Type    Build  Protocol  DC   Segment
  node1  172.16.0.1:8301  alive   server  1.8.4  2         dc1  <all>
  node2  172.16.0.2:8301  alive   server  1.8.4  2         dc1  <all>
  node3  172.16.0.3:8301  alive   server  1.8.4  2         dc1  <all>
  ```

### 4.3. 负载均衡

+ 在172.16.0.10上为consul-server配置nginx的tcp代理

  ``` bash
  # 服务端RPC，TCP
  upstream consul-server-8300 {
          server 172.16.0.1:8300;
          server 172.16.0.2:8300;
          server 172.16.0.3:8300;
  }
  server {
          listen 172.16.0.10:8300;
          proxy_pass consul-server-8300;
  }
  
  # Serl LAN：处理LAN gossip，TCP UDP。
  upstream consul-server-8301 {
          server 172.16.0.1:8301;
          server 172.16.0.2:8301;
          server 172.16.0.3:8301;
  }
  server {
          listen 172.16.0.10:8301;
          proxy_pass consul-server-8301;
  }
  
  # Serl WAN：处理LAN gossip，TCP UDP。
  upstream consul-server-8302 {
          server 172.16.0.1:8302;
          server 172.16.0.2:8302;
          server 172.16.0.3:8302;
  }
  server {
          listen 172.16.0.10:8302;
          proxy_pass consul-server-8302;
  }
  ```

+ 在172.16.0.10上为consul-client配置nginx的tcp代理

  ``` bash
  # 客户端HTTP API，TCP.
  upstream consul-client-8500 {
          server 172.16.0.4:8500;
          server 172.16.0.5:8500;
  }
  server {
          listen 172.16.0.10:8500;
          proxy_pass consul-client-8500;
  }
  
  # 客户端DNS，TCP,UDP
  upstream consul-client-8600 {
          server 172.16.0.4:8600;
          server 172.16.0.5:8600;
  }
  server {
          listen 172.16.0.10:8600;
          proxy_pass consul-client-8600;
  }
  ```

### 4.4. 添加client

客户端可以添加无数个，每台机器都这样配置就好了

+ 创建配置文件`/etc/consul/client.json`

  ``` json
  {
  "data_dir": "/data/consul/data",
  "log_file": "/data/consul/log/consul.log",
  "log_level": "INFO",
  "log_rotate_duration": "24h",
  "node_name": "node4",
  "server": false,
  "client_addr": "0.0.0.0",
  "advertise_addr": "172.16.0.4",
  "start_join": ["172.16.0.10"] # 如果不用负载均衡的话，也可以把所有server的ip地址都写上
  }
  ```

+ 创建systemd文件`/lib/systemd/system/consul-client.service`

  ``` bash
  [Unit]
  Description=Consul service
  Documentation=https://www.consul.io/docs/
  
  [Service]
  ExecStart=/usr/local/bin/consul agent -ui -config-dir /etc/consul
  KillSignal=SIGINT
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
  ```

+ 启动服务

  ``` bash
  systemctl start consul-client
  ```

+ node5也如法炮制，最后看一下我们所有的节点

  ``` bash
  # consul members
  Node   Address          Status  Type    Build  Protocol  DC   Segment
  node1  172.16.0.1:8301  alive   server  1.8.4  2         dc1  <all>
  node2  172.16.0.2:8301  alive   server  1.8.4  2         dc1  <all>
  node3  172.16.0.3:8301  alive   server  1.8.4  2         dc1  <all>
  node4  172.16.0.4:8301  alive   client  1.8.4  2         dc1  <default>
  node5  172.16.0.5:8301  alive   client  1.8.4  2         dc1  <default>
  ```

## 5. 添加endpoint

我们前面为client添加了nginx的反向代理，所以说我们直接向client发起请求就好了。注意，`-ui`选项是为了帮助我们查看状态的，我们是可以不打开的，我们仅通过命令行来控制我们的服务。

``` bash
# 注册服务，
curl --request PUT --data @你要注册的服务.json http://172.16.0.10:8500/v1/agent/service/register
# 查询服务
curl http://172.16.0.10:8500/v1/catalog/service/你的服务名
# 取消注册
curl -XPUT http://172.16.0.10:8500/v1/agent/service/deregister/你的服务名
```

### 5.1. prometheus选取服务

对于不同类型的服务，我们使用tag的方式进行区别，然后在prometheus中通过标签选择的方式，选出我们需要的endpoint，比如，我们要筛选出node-exporter的tag的，然后进行分类

``` yaml
  - job_name: 'consul-node-exporter'
    consul_sd_configs:
      - server: '172.16.0.10:8500'
        services: []  
    relabel_configs:
      - source_labels: [__meta_consul_tags]
        regex: .*node-exporter.*  # 这是一个特殊的正则，用来匹配我们的服务
        action: keep
      - regex: __meta_consul_service_metadata_(.+)
        action: labelmap
```

### 5.2. node-exporter

修改node-exporter-demo.json

``` json
{
  "ID": "node-exporter",
  "Name": "node-exporter-172-16-0-1",
  "Tags": [
    "node-exporter"  // 这个就是我们需要筛选的内容
  ],
  "Address": "172.16.0.1",
  "Port": 9100,
  "Meta": {
    "app": "elasticsearch",
    "team": "bigdata",
    "project": "log-analyze"
  },
  "EnableTagOverride": false,
  "Check": {
    "HTTP": "http://172.16.0.1:9100/metrics",
    "Interval": "10s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}
```

使用curl方式注册服务

``` bash
curl --request PUT --data @node-exporter-demo.json http://172.16.0.10:8500/v1/agent/service/register
```

### 5.3. 结果

+ 查询consul

  ``` bash
  # curl http://172.16.0.10:8500/v1/catalog/service/node-exporter-172-16-0-1?pretty
  [
      {
          "ID": "91969b25-8494-33d8-2e41-59c048eb994e",
          "Node": "node5",
          "Address": "172.16.0.5",
          "Datacenter": "dc1",
          "TaggedAddresses": {
              "lan": "172.16.0.5",
              "lan_ipv4": "172.16.0.5",
              "wan": "172.16.0.5",
              "wan_ipv4": "172.16.0.5"
          },
          "NodeMeta": {
              "consul-network-segment": ""
          },
          "ServiceKind": "",
          "ServiceID": "node-exporter",
          "ServiceName": "node-exporter-172-16-0-1",
          "ServiceTags": [
              "node-exporter"
          ],
          "ServiceAddress": "172.16.0.1",
          "ServiceTaggedAddresses": {
              "lan_ipv4": {
                  "Address": "172.16.0.1",
                  "Port": 9100
              },
              "wan_ipv4": {
                  "Address": "172.16.0.1",
                  "Port": 9100
              }
          },
          "ServiceWeights": {
              "Passing": 10,
              "Warning": 1
          },
          "ServiceMeta": {
              "app": "elasticsearch",
              "project": "log-analyze",
              "team": "bigdata"
          },
          "ServicePort": 9100,
          "ServiceEnableTagOverride": false,
          "ServiceProxy": {
              "MeshGateway": {},
              "Expose": {}
          },
          "ServiceConnect": {},
          "CreateIndex": 1384,
          "ModifyIndex": 1384
      }
  ]
  ```

+ 查看prometheus的结果

  ![image-20201009173834806](/pages/keynotes/L4_architect/2_monitoring/pics/27_consul_HA/image-20201009173834806.png)

  

instance就对应我们json文件中的address:port