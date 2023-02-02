---
title: 生产级别的Opensearch 2.4
keywords: keynotes, architect, observability, log, opensearch, opensearch_in_prodution
permalink: keynotes_L5_architect_observability_2_log_1_2_opensearch_in_production.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_3_opensearch2_4_in_production
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 简介

OpenSearch的2.0版本对应了ElasticSearch的8.0版本，但是，功能上已经和ES算是两个不同的分支了。OpenSearch从1.0版本升级到2.0版本的目的和ES从7.0版本升级到8.0版本的原因基本相同。主要是由于lucence引擎从8升级到了9，所以两款软件都进行了大版本的升级。

当然，他们各自在功能上也都有了非常大的增加。目前ES依然是在商业功能上发力，而OpenSearch还是在努力的解决兼容性问题，从而占领更大的市场。

## 2. 组件和架构

OpenSearch 2.x和1.x的架构和组件完全一致，可以参考我上一篇文章

## 3. 安装和配置

### 3.1. 服务器

| 服务器IP     | 操作系统   | 组件               |
| ------------ | ---------- | ------------------ |
| 10.39.64.243 | CentOS 7.9 | OpenSearch，Kibana |
| 10.39.64.233 | CentOS 7.9 | OpenSearch，Kibana |
| 10.39.64.242 | CentOS 7.9 | OpenSearch，Kibana |

### 3.2. 准备环境

+ 创建相关的目录

  ``` bash
  mkdir /app/{opensearch,opensearch-dashboards}
  mkdir /app/opensearch/{data,logs}
  ```

+ 下载安装包

  ``` bash
  wget -c https://artifacts.opensearch.org/releases/bundle/opensearch/2.4.1/opensearch-2.4.1-linux-x64.tar.gz -P /app/opensearch
  wget -c https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.4.1/opensearch-dashboards-2.4.1-linux-x64.tar.gz -P /app/opensearch-dashboards
  ```

+ 解压

  ``` bash
  cd /app/opensearch
  tar xf opensearch-2.4.1-linux-x64.tar.gz
  cd /app/opensearch-dashboards
  tar xf opensearch-dashboards-2.4.1-linux-x64.tar.gz
  ```

+ 关闭swap

  ``` bash
  swapoff -a
  ```

+ 修改/etc/sysctl.conf

  ``` bash
  cat >> /etc/sysctl.conf<<EOF
  vm.max_map_count=262144
  EOF
  sysctl -p
  ```

+ 创建opensearch用户

  ``` bash
  useradd opensearch
  chown -R opensearch:opensearch /app/opensearch
  chown -R opensearch:opensearch /app/opensearch-dashboards
  ```

+ 调整Java内存参数，官方建议修改环境变量`OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m`，我这里修改了/app/opensearch/opensearch-2.4.1/config/jvm.options

  ``` bash
  -Xms8g
  -Xmx8g
  ```


### 3.3. 证书

证书是安全插件所提供的功能，包放在`/app/opensearch/opensearch-2.4.1/plugins/opensearch-security`下面，配置证书的位置是在主配置文件opensearch.yml里面配置。

单机启动的时候会创建一些证书，我们可以把这些证书拷贝到其他节点上直接使用。但是在生产系统中，我们可能要使用管理员提供的证书。

#### 3.3.1.  生成证书

官方提供了一个[页面](https://opensearch.org/docs/latest/security-plugin/configuration/generate-certificates/)专门说自建证书。我们把主要步骤列在下面，需要注意的是，每个节点都需要创建自己的证书，然后使用公司的私钥去签一下。如果我们要使用公司的证书，需要把每台node的私钥生成的证书发给CA管理员去签署，而不是把域名丢给管理员，让管理员去签域名的证书。

证书一共有三组：

+ admin证书： 是用来做授权的时候数据加密的
+ node证书：是用来做数据同步时候数据加密的
+ client证书：是用来做做客户端和服务端数据加密的

下面了例子创建了三组证书，

``` bash
#!/bin/sh
# Root CA
openssl genrsa -out root-ca-key.pem 2048
openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=ROOT" -out root-ca.pem -days 730
# Admin cert
openssl genrsa -out admin-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem
openssl req -new -key admin-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=ADMIN" -out admin.csr
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out admin.pem -days 730
# Node cert 1
openssl genrsa -out node1-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node1-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node1-key.pem
openssl req -new -key node1-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=opensearch-ittools-prod-01" -out node1.csr
openssl x509 -req -in node1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node1.pem -days 730
# Node cert 2
openssl genrsa -out node2-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node2-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node2-key.pem
openssl req -new -key node2-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=opensearch-ittools-prod-02" -out node2.csr
openssl x509 -req -in node2.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node2.pem -days 730
# Node cert 3
openssl genrsa -out node3-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node3-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node3-key.pem
openssl req -new -key node3-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=opensearch-ittools-prod-03" -out node3.csr
openssl x509 -req -in node3.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node3.pem -days 730
# Client cert
openssl genrsa -out client-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in client-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out client-key.pem
openssl req -new -key client-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=CLIENT" -out client.csr
openssl x509 -req -in client.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out client.pem -days 730
# Cleanup
rm admin-key-temp.pem
rm admin.csr
rm node1-key-temp.pem
rm node1.csr
rm node2-key-temp.pem
rm node2.csr
rm client-key-temp.pem
rm client.csr
```

#### 3.3.2. 配置证书

Opensearch支持pkcs8和pkcs12类型的证书，但是配置是不一样的，详细的配置可以看[这里](https://opensearch.org/docs/latest/security-plugin/configuration/tls/)。

我们这里根据上面的结果，给出一个例子，certs目录就是`/app/opensearch/opensearch-2.4.1/config/certs`，需要自己手动创建

``` bash
######## Start OpenSearch Security Demo Configuration ########
# WARNING: revise all the lines below before you go into production
plugins.security.ssl.transport.pemcert_filepath: certs/node1.pem
plugins.security.ssl.transport.pemkey_filepath: certs/node1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/node1.pem
plugins.security.ssl.http.pemkey_filepath: certs/node1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.allow_unsafe_democertificates: true
plugins.security.allow_default_init_securityindex: true
plugins.security.authcz.admin_dn:
  - 'CN=ADMIN,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
plugins.security.nodes_dn:
  - 'CN=opensearch-ittools-prod-01,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=opensearch-ittools-prod-02,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=opensearch-ittools-prod-03,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opendistro-notifications-*", ".opendistro-notebooks", ".opendistro-asynchronous-search-response*", ".replication-metadata-store"]
node.max_local_storage_nodes: 3
######## End OpenSearch Security Demo Configuration ########
```

### 3.4. 修改配置文件

#### 3.4.1. OpenSearch

配置文件位置在`/opensearch/app/opensearch-1.2.0/config`下面

``` bash
jvm.options # 和Java的参数有关
jvm.options.d
log4j2.properties # 和log4j相关的配置
opensearch.keystore # 系统默认的密码文件
opensearch-observability # 和这个插件相关的配置
opensearch-reports-scheduler # 和这个插件相关的配置
opensearch.yml # opensearch的主配置文件
```

这次实验中的每个节点具备了node，master和coordinate所有的功能，配置文件中的选项配置如下。如果我们要针对节点做分工，给他们不同的职责，请参考[这里](https://opensearch.org/docs/latest/opensearch/cluster/#step-2-set-node-attributes-for-each-node-in-a-cluster)

``` bash
cluster.name: ma-ittools
node.name: opensearch-ittools-prod-01
node.attr.rack: Nutanix
path.data: /app/opensearch/data
path.logs: /app/opensearch/logs
bootstrap.memory_lock: true
network.host: 10.39.64.243
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["10.39.64.243:9300", "10.39.64.233:9300", "10.39.64.242:9300"]
cluster.initial_master_nodes: ["opensearch-ittools-prod-01", "opensearch-ittools-prod-02", "opensearch-ittools-prod-03"]
gateway.recover_after_nodes: 3
action.destructive_requires_name: true
```

所以说，一个node上完整的配置如下，例子中以node1为例，其他节点请修改节点响应信息。

``` bash
cluster.name: ma-ittools
node.name: opensearch-ittools-prod-01
node.attr.rack: Nutanix
path.data: /app/opensearch/data
path.logs: /app/opensearch/logs
bootstrap.memory_lock: true
network.host: 10.39.64.243
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["10.39.64.243:9300", "10.39.64.233:9300", "10.39.64.242:9300"]
cluster.initial_master_nodes: ["opensearch-ittools-prod-01", "opensearch-ittools-prod-02", "opensearch-ittools-prod-03"]
gateway.recover_after_nodes: 3
action.destructive_requires_name: true

######## Start OpenSearch Security Demo Configuration ########
# WARNING: revise all the lines below before you go into production
plugins.security.ssl.transport.pemcert_filepath: certs/node1.pem
plugins.security.ssl.transport.pemkey_filepath: certs/node1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/node1.pem
plugins.security.ssl.http.pemkey_filepath: certs/node1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.allow_unsafe_democertificates: true
plugins.security.allow_default_init_securityindex: true
plugins.security.authcz.admin_dn:
  - 'CN=ADMIN,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
plugins.security.nodes_dn:
  - 'CN=opensearch-ittools-prod-01,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=opensearch-ittools-prod-02,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=opensearch-ittools-prod-03,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opendistro-notifications-*", ".opendistro-notebooks", ".opendistro-asynchronous-search-response*", ".replication-metadata-store"]
node.max_local_storage_nodes: 3
######## End OpenSearch Security Demo Configuration ########
```

#### 3.4.2. OpenSearch-Dashboard

Dashboard本身是由nodejs开发的，我们可以看到非常清晰的npm的痕迹。配置文件在/app/opensearch-dashboards/opensearch-dashboards-2.4.1/config下面。

``` bash
opensearch.hosts: ["https://10.39.64.243:9200","https://10.39.64.233:9200","https://10.39.64.242:9200"]
opensearch.ssl.verificationMode: none
opensearch.username: "kibanaserver"
opensearch.password: "kibanaserver"
opensearch.requestHeadersWhitelist: [ authorization,securitytenant ]
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.preferred: ["Private", "Global"]
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
opensearch_security.cookie.secure: false
server.host: 10.39.64.243
server.port: 5601

#指定日志目录
logging.dest: /app/opensearch-dashboards/logs
```

我们注意到，实际上nodejs的配置文件中并没有指定数据目录，而日志默认是打到stdout上的，所以不太利于定位问题，我们可以选择把日志打到systemd上，用journalctl来看，也可以像我这样配置到某一个文件中。

opensearch-dashboard如果想创建多个节点，就在每个服务器上启动多个进程就好了，且配置文件一样，然后在前面创建一个负载均衡就可以了。

### 3.5. 启动进程

启动opensearch，需要在每个node上执行

``` bash
su - opensearch -c "/app/opensearch/opensearch-2.4.1/bin/opensearch" &
```

启动opensearch-dashboard

``` bash
su - opensearch -c "/app/opensearch-dashboards/opensearch-dashboards-2.4.1/bin/opensearch-dashboards" &
```

