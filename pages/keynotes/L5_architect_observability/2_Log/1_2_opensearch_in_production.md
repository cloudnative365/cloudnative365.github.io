---
title: 生产级别的Opensearch
keywords: keynotes, architect, observability, log, opensearch, opensearch_in_prodution
permalink: keynotes_L5_architect_observability_2_log_1_2_opensearch_in_production.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/1_2_opensearch_in_production
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ OpenSearch插件
+ OpenSearch集群
+ OpenSearch安全

## 1. OpenSearch插件

从上一篇文章中，我们知道，OpenSearch插件的前身就是Open Distro。这些插件是默认安装的。他们被安装在`OPENSEARCH_HOME/plugins`目录下面。这些插件很大程度上丰富了整个系统的功能，但是到目前位置，社区上的很多插件还是适配ES的，而不是OpenSearch。如果我们强行把ES上的插件搬到OpenSearch用，大概率会起不来。但是有AWS这棵大树，OpenSearch不是一个人在奋斗，相信在不久的将来，大量开发者就会转向OpenSearch。

从这次Log4j的风波就可以看出来，AWS对于OpenSearch的支持力度可是杠杠的，补丁基本上是0day就放出了，让运维升着放心，让开发用着安心。

## 2. OpenSearch集群

### 2.1. 节点的类型

| 节点类型        | 作用                                                         | 机器配置                     |
| --------------- | ------------------------------------------------------------ | ---------------------------- |
| master          | 索引的创建或删除<br />跟踪哪些节点是集群的一部分<br />决定哪些分片分配给相关的节点 | CPU 内存 消耗一般            |
| Master-eligible | 参与集群选举                                                 | CPU 内存 消耗一般            |
| data            | 存储索引数据<br />对文档进行增删改查,聚合操作                | 资源大户，主要消耗磁盘，内存 |
| Ingest(提取)    | Ingest节点和集群中的其他节点一样，但是它能够创建多个处理器管道，用以修改传入文档。类似 最常用的Logstash过滤器已被实现为处理器。<br />Ingest节点 可用于执行常见的数据转换和丰富。 处理器配置为形成管道。 在写入时，Ingest Node有20个内置处理器，例如grok，date，gsub，小写/大写，删除和重命名<br />在批量请求或索引操作之前，Ingest节点拦截请求，并对文档进行处理。 | CPU 内存 消耗一般            |
| Coordinating    | 协调节点，是一种角色，而不是真实的Elasticsearch的节点，你没有办法通过配置项来配置哪个节点为协调节点。集群中的任何节点，都可以充当协调节点的角色。当一个节点A收到用户的查询请求后，会把查询子句分发到其它的节点，然后合并各个节点返回的查询结果，最后返回一个完整的数据集给用户。在这个过程中，节点A扮演的就是协调节点的角色。 | 资源大户，主要消耗磁盘，内存 |

### 2.2. 常见的部署方式

常见的集群部署方式可以归为两大类，一类是使用SaaS服务，比如AWS的OpenSearch服务，一类是自己搭建。而自己搭建的集群也有两种大类，一个是部署在虚拟机或者实例上，一个是在k8s集群中部署。

在实例上部署：

![image-20211102114231598](/pages/keynotes/L5_architect_observability/2_log/pics/1_2_opensearch_in_production/image-20211102114231598.png)

在k8s上部署

![image-20211102105336099](/pages/keynotes/L5_architect_observability/2_log/pics/1_2_opensearch_in_production/image-20211102105336099.png)

![image-20211102105344439](/pages/keynotes/L5_architect_observability/2_log/pics/1_2_opensearch_in_production/image-20211102105344439.png)

## 3. 安装集群

我们这里使用二进制包的方式来部属，让大家更容易理解架构

### 3.1. 准备安装

#### 3.1.1. 下载安装包

进入[下载界面](https://opensearch.org/downloads.html)，下载最新版[OpenSearch](https://artifacts.opensearch.org/releases/bundle/opensearch/1.2.0/opensearch-1.2.0-linux-x64.tar.gz)和[OpenSearch-Dashboard](https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/1.2.0/opensearch-dashboards-1.2.0-linux-x64.tar.gz)，如果要下载历史版本，就要去[github](https://github.com/opensearch-project)去下载了。

``` bash
wget https://artifacts.opensearch.org/releases/bundle/opensearch/1.2.0/opensearch-1.2.0-linux-x64.tar.gz
wget https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/1.2.0/opensearch-dashboards-1.2.0-linux-x64.tar.gz
```

#### 3.1.2. 初始化操作系统

+ 文件系统：（个人习惯，仅供参考）我习惯创建三个文件架来分别存放二进制包（app），数据文件（data）和日志（logs）

  ``` bash
  # 二进制包的大小基本不会变
  # 日志是有rotation机制的，大小也基本不会变
  # 数据盘比较重要，为了保证性能，在有条件的情况下，建议使用SSD硬盘
  mkdir /opensearch/{app,data,logs}
  ```

+ 解压二进制包到app目录下

  ``` bash
  tar xf opensearch-1.2.0-linux-x64.tar.gz -C /opensearch/app/
  tar xf opensearch-dashboards-1.2.0-linux-x64.tar.gz -C /opensearch/app
  ```

+ 配置Linux内核参数，修改/etc/sysctl.conf

  ``` bash
  vm.max_map_count=262144
  ```

  执行`sudo sysctl -p`重新加载内核

+ 调整Java内存参数，官方建议修改环境变量`OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m`，我这里修改了/opensearch/app/opensearch-1.2.0/config/jvm.option

  ``` bash
  -Xms24g
  -Xmx24g
  ```

+ 修改打开文件数，`nofile 65536`，我们这里使用的是RHEL7，修改`/etc/security/limits.d/20-nproc.conf`，增加下面一行

  ``` bash
  opensearch      soft    nproc   65536
  ```

+ 创建opensearch用户，整个opensearch和opensearch-dashboard都需要使用opensearch用户来运行

  ``` bash
  useradd opensearch
  chown opensearch:opensearch /opensearch/*
  ```

### 3.2. 证书

证书是安全插件所提供的功能，包放在`/opensearch/app/opensearch-1.2.0/plugins/opensearch-security`下面，配置证书的位置是在主配置文件opensearch.yml里面配置。

单机启动的时候会创建一些证书，我们可以把这些证书拷贝到其他节点上直接使用。但是在生产系统中，我们可能要使用管理员提供的证书。

#### 3.2.1.  生成证书

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
openssl req -new -key node1-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=node1.example.com" -out node1.csr
openssl x509 -req -in node1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node1.pem -days 730
# Node cert 2
openssl genrsa -out node2-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node2-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node2-key.pem
openssl req -new -key node2-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=node2.example.com" -out node2.csr
openssl x509 -req -in node2.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node2.pem -days 730
# Node cert 3
openssl genrsa -out node3-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node3-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node3-key.pem
openssl req -new -key node3-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=node3.example.com" -out node3.csr
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

#### 3.2.2. 配置证书

Opensearch支持pkcs8和pkcs12类型的证书，但是配置是不一样的，详细的配置可以看[这里](https://opensearch.org/docs/latest/security-plugin/configuration/tls/)。

我们这里根据上面的结果，给出一个例子，certs目录就是`/opensearch/app/opensearch-1.2.0/config/certs`，需要自己手动创建

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
  - 'CN=ubr-dc-data1.opensearch.ubrmb.com,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=ubr-dc-data2.opensearch.ubrmb.com,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=ubr-dc-data3.opensearch.ubrmb.com,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opendistro-notifications-*", ".opendistro-notebooks", ".opendistro-asynchronous-search-response*", ".replication-metadata-store"]
node.max_local_storage_nodes: 3
######## End OpenSearch Security Demo Configuration ########
```

### 3.3. 修改配置文件

#### 3.3.1. OpenSearch

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
cluster.name: ubr-dc-internal
node.name: ubr-dc-data1
node.attr.rack: RHV
path.data: /opensearch/data
path.logs: /opensearch/logs
bootstrap.memory_lock: true
network.host: 10.114.2.129
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["10.114.2.129:9300", "10.114.2.130:9300", "10.114.2.131:9300"]
cluster.initial_master_nodes: ["ubr-dc-data1", "ubr-dc-data2", "ubr-dc-data3"]
gateway.recover_after_nodes: 3
action.destructive_requires_name: true
```

所以说，一个node上完整的配置如下，例子中以node1为例，其他节点请修改节点响应信息。

``` bash
cluster.name: ubr-dc-internal
node.name: ubr-dc-data1
node.attr.rack: RHV
path.data: /opensearch/data
path.logs: /opensearch/logs
bootstrap.memory_lock: true
network.host: 10.114.2.129
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["10.114.2.129:9300", "10.114.2.130:9300", "10.114.2.131:9300"]
cluster.initial_master_nodes: ["ubr-dc-data1", "ubr-dc-data2", "ubr-dc-data3"]
gateway.recover_after_nodes: 3
action.destructive_requires_name: true

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
  - 'CN=ubr-dc-data1.opensearch.ubrmb.com,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=ubr-dc-data2.opensearch.ubrmb.com,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=ubr-dc-data3.opensearch.ubrmb.com,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opendistro-notifications-*", ".opendistro-notebooks", ".opendistro-asynchronous-search-response*", ".replication-metadata-store"]
node.max_local_storage_nodes: 3
```

#### 3.3.2. OpenSearch-Dashboard

Dashboard本身是由nodejs开发的，我们可以看到非常清晰的npm的痕迹。配置文件在/opensearch/app/opensearch-dashboards-1.2.0-linux-x64/config下面。

``` bash
opensearch.hosts: ["https://10.114.2.129:9200","https://10.114.2.130:9200","https://10.114.2.131:9200"]
opensearch.ssl.verificationMode: none
opensearch.username: "kibanaserver"
opensearch.password: "kibanaserver"
opensearch.requestHeadersWhitelist: [ authorization,securitytenant ]
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.preferred: ["Private", "Global"]
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
opensearch_security.cookie.secure: false
server.host: 10.114.2.129
server.port: 5601

#指定日志目录
logging.dest: /opensearch/logs/opensearch
```

我们注意到，实际上nodejs的配置文件中并没有指定数据目录，而日志默认是打到stdout上的，所以不太利于定位问题，我们可以选择把日志打到systemd上，用journalctl来看，也可以像我这样配置到某一个文件中。

opensearch-dashboard如果想创建多个节点，就在每个服务器上启动多个进程就好了，且配置文件一样，然后在前面创建一个负载均衡就可以了。

### 3.4. 启动进程

启动opensearch，需要在每个node上执行

``` bash
su - opensearch -c "/opensearch/app/opensearch-dashboards-1.2.0-linux-x64/bin/opensearch-dashboards" &
```

启动opensearch-dashboard

``` bash
su - opensearch -c "/opensearch/app/opensearch-dashboards-1.2.0-linux-x64/bin/opensearch-dashboards" &
```

## 4.  企业级安全配置

### 4.1. 安全插件

opensearch的认证功能同样依靠[security插件](https://opensearch.org/docs/latest/security-plugin/index/)，他支持非常多的认证和授权方式。

| Feature                                                      | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Node-to-node encryption                                      | Encrypts traffic between nodes in the OpenSearch cluster.    |
| HTTP basic authentication                                    | A simple authentication method that includes a user name and password as part of the HTTP request. |
| Support for Active Directory, LDAP, Kerberos, SAML, and OpenID Connect | Use existing, industry-standard infrastructure to authenticate users, or create new users in the internal user database. |
| Role-based access control                                    | Roles define the actions that users can perform: the data they can read, the cluster settings they can modify, the indices to which they can write, and so on. Roles are reusable across users, and users can have multiple roles. |
| Index-level, document-level, and field-level security        | Restrict access to entire indices, certain documents within an index, or certain fields within documents. |
| Audit logging                                                | These logs let you track access to your OpenSearch cluster and are useful for compliance purposes or after unintended data exposure. |
| Cross-cluster search                                         | Use a coordinating cluster to securely send search requests to remote clusters. |
| OpenSearch Dashboards multi-tenancy                          | Create shared (or private) spaces for visualizations and dashboards. |

我们常用的有

+ Node-to-node encryption：这个就需要用到我们上面做证书时候的client certificate了，数据发送端在发送数据前需要用证书认证一下。
+ HTTP basic authentication： 我们用call API的方式查看数据的时候需要带上用户名和密码。
+ Support for Active Directory, LDAP, Kerberos, SAML, and OpenID Connect：用户登录的时候，去其他认证中心去认证后才能访问
+ Audit logging：其实我们已经打开了审计功能，`plugins.security.audit.type: internal_opensearch`,其他的配置可以参考[官方文档](https://opensearch.org/docs/latest/security-plugin/audit-logs/index/)

剩下的功能我们后面会说到

### 4.2. AD认证

在很多企业用，用户都是用windows的AD来管理的，所以我们这里就以AD认证为例，来展示一下怎样去AD中认证。

#### 4.2.1. 准备工作

不管什么样的linux系统上的什么软件，大部分用到ldap功能的，基本都用到操作系统的ldap依赖，如果我们发现无法使用ldapsearch命令，那基本上就是ldap依赖没安装，以centos/rhel为例

``` bash
# 看看哪个包中有ldapsearch命令
yum whatprovides ldapsearch
# 安装ldapsearch客户端
yum whatprovides openldap-clients
```

有些公司还对ldap认证进行了证书认证，需要我们把证书导入linux系统才能使用ldapsearch命令

``` bash
# 把证书放到/etc/pki/ca-trust/source/anchors下，然后执行
update-ca-trust
```

#### 4.2.2. 配置

下面是我的一个配置文件`/opensearch/app/opensearch-1.2.0/plugins/opensearch-security/securityconfig/config.yml`的例子

``` bash
config:
  dynamic:
    http:
      anonymous_auth_enabled: false
      xff:
        enabled: false
    authc: # 认证的部分
      kerberos_auth_domain:
        .
      basic_internal_auth_domain:
        description: "Authenticate via HTTP Basic against internal users database"
        http_enabled: true
        transport_enabled: true
        order: 0 # 这里写0表示第一个来这里认证
        http_authenticator:
          type: basic
          challenge: true
        authentication_backend:
          type: intern
      proxy_auth_domain:
        .
      jwt_auth_domain:
        .
      clientcert_auth_domain:
        .
      ldap:
        description: "Authenticate via LDAP or Active Directory"
        http_enabled: true
        transport_enabled: true
        order: 1 # 这个改成1
        http_authenticator:
          type: basic
          challenge: true
        authentication_backend:
          type: ldap
          config:
            enable_ssl: false
            enable_start_tls: false
            enable_ssl_client_auth: false
            verify_hostnames: false
            hosts:
            - UBMPAPWP00001.ubrmb.com:389
            bind_dn: "CN=s000064,OU=ServiceAccount,DC=UBRMB,DC=COM"
            password: "xxxx"
            userbase: "OU=UBRMB,DC=UBRMB,DC=COM"
            usersearch: '(sAMAccountName={0})'
            username_attribute: sAMAccountName
    authz:
      roles_from_myldap: # 这里是role的名字，可以在前台看到
        description: "Authorize via LDAP or Active Directory"
        http_enabled: true
        transport_enabled: true
        authorization_backend:
          type: ldap
          config:
            enable_ssl: false
            enable_start_tls: false
            enable_ssl_client_auth: false
            verify_hostnames: false
            hosts:
            - UBMPAPWP00001.ubrmb.com:389
            bind_dn: "CN=s000064,OU=ServiceAccount,DC=UBRMB,DC=COM"
            password: "xxxx"
            rolebase: "OU=SecurityGroup,DC=UBRMB,DC=COM"
            rolesearch: '(uniquemember={0})'
            userroleattribute: null
            userrolename: memberOf
            rolename: cn
            resolve_nested_roles: true
            userbase: "OU=Users,DC=UBRMB,DC=COM"
            usersearch: '(cn={0})'
      roles_from_another_ldap:
        description: "Authorize via another Active Directory"
        http_enabled: false
        transport_enabled: false
        authorization_backend:
          type: ldap
```

### 4.3. 让配置生效

#### 4.3.1. 官方文档

参考：https://opensearch.org/docs/latest/security-plugin/configuration/security-admin/

所有的安全插件的配置，包括用户，角色和权限都存在OpenSearch集群中叫`.opendistro_security`的index中。把这些配置都放在index中可以保证我们改变配置之后不需要重启集群并且防止我们在每个节点上重复修改配置。

想要初始化`.opendistro_security`index，我们就需要运行`plugins/opensearch-security/tools/securityadmin.sh`命令，这个脚本会加载我们的初始化配置到index当中，初始化配置文件在`plugins/opensearch-security/securityconfig`当中。`.opendistro_security`index初始化完毕，就可以使用OpenSearch-Dashboard或者使用REST API来管理用户，角色和权限。

+ 备份配置文件

  ``` bash
  ./securityadmin.sh -backup my-backup-directory \
    -icl \
    -nhnv \
    -cacert ../../../config/root-ca.pem \
    -cert ../../../config/kirk.pem \
    -key ../../../config/kirk-key.pem
  ```

+ 指定加载某个配置文件

  ``` bash
  ./securityadmin.sh -f ../securityconfig/internal_users.yml \
    -t internalusers \
    -icl \
    -nhnv \
    -cacert ../../../config/root-ca.pem \
    -cert ../../../config/kirk.pem \
    -key ../../../config/kirk-key.pem
  ```

#### 4.3.2. 修改admin密码

回到我们的系统中来，我们要先修改internal_users.yml文件，但是我们发现密码是hash过的，使用下面的命令来

+ 生成hash密码

  ``` bash
  chmod +x /opensearch/app/opensearch-1.2.0/plugins/opensearch-security/tools/hash.sh
  /opensearch/app/opensearch-1.2.0/plugins/opensearch-security/tools/hash.sh
  [Password:]
  $2y$12$fTYA9oV8pRMjq2vqJSoQS.OUYNsTY8xH62Z74mGgTCqNHMGzFOkO6
  ```

+ internal_users.yml文件

  ``` bash
  cp /opensearch/app/opensearch-1.2.0/plugins/opensearch-security/securityconfig/internal_users.yml{,.$(date +%F)}
  vim /opensearch/app/opensearch-1.2.0/plugins/opensearch-security/securityconfig/internal_users.yml
  ```

+ 找到admin相关的配置

  ``` bash
  admin:
    hash: "$2y$12$fTYA9oV8pRMjq2vqJSoQS.OUYNsTY8xH62Z74mGgTCqNHMGzFOkO6"
    reserved: true
    backend_roles:
    - "admin"
    description: "admin user"
  ```

+ 应用配置

  ``` bash
  /opensearch/app/opensearch-1.2.0/plugins/opensearch-security/tools/securityadmin.sh \
  -f /opensearch/app/opensearch-1.2.0/plugins/opensearch-security/securityconfig/internal_users.yml \
  -t internalusers \
  -icl \
  -nhnv \
  -cacert /opensearch/app/opensearch-1.2.0/config/certs/root-ca.pem \
  -cert /opensearch/app/opensearch-1.2.0/config/certs/admin.pem \
  -key /opensearch/app/opensearch-1.2.0/config/certs/admin-key.pem \
  -h 10.114.2.129
  ```

  

