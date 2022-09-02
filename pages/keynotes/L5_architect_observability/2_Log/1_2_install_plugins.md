---
title: 安装插件
keywords: keynotes, architect, logging, install_plugins
permalink: keynotes_L5_architect_observability_2_log_1_2_install_plugins.html
sidebar: keynotes_L5_architect_observability
typora-copy-images-to: ./pics/1_2_install_plugins.md
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

安装安全插件SearchGuard

## 1. 安装SearchGuard

![Logo](/pages/keynotes/L4_architect/3_logging/pics/3_install_plugins/sg_dlic_small.png)

在早期的ES版本（6.2之前）中，使用的认证插件是X-pack，这是一个官方的插件，但是在v6.2之后，ES自己支持了安全认证功能，但是要购买license。而另外一个比较知名的安全插件就是我们今天要说的SearchGuard了。SearchGuard在收费模式上和x-pack基本一致，也就是在粗粒度的安全级别上是免费的，比如index级别，但是到字段级别就需要收费了，我们一会再详细说。

### 1.1. 安装SearchGuard for ES

+ SearchGuard的[官方网站](https://search-guard.com/)，[官方文档](https://docs.search-guard.com/latest/)，[下载地址](https://docs.search-guard.com/latest/search-guard-versions)，demo文档

+ 下载页面上的版本和ES的版本基本对应，但是对于非常新的版本可能会有一定延迟

  ![image-20200826100253036](/pages/keynotes/L4_architect/3_logging/pics/3_install_plugins/image-20200826100253036.png)

+ 安装：

  在线安装

  ``` bash
  ./bin/elasticsearch-plugin install com.floragunn:search-guard-7:7.8.1-43.0 
  ```

  离线安装

  ``` bash
  # 下载Zip包，一共有三种下载，我们选Search Guard，里面包含了所有POC时候用到的包，如果不想试用，可以直接下载第二个Search Guard Admin
  ./bin/elasticsearch-plugin install -b file:///path/to/search-guard-suite-plugin-7.8.1-43.0.0.zip
  ```

+ demo installer

  ``` bash
  # 切换到Elasticsearc的安装目录
  cd ./plugins/search-guard-7/tools
  # 修改权限
  chmod 755 ./install_demo_configuration.sh
  # 运行命令
  ./install_demo_configuration.sh
  # 问到下面的问题的时候，按照这个回答
  Install demo certificates? [y/N] y
  Initialize Search Guard? [y/N] y
  Enable cluster mode? [y/N] n
  ```

+ 完成后，会在elasticsearch.yml文件中增加下面几行

  ``` bash
  ######## Start Search Guard Demo Configuration ########
  # WARNING: revise all the lines below before you go into production
  searchguard.ssl.transport.pemcert_filepath: esnode.pem
  searchguard.ssl.transport.pemkey_filepath: esnode-key.pem
  searchguard.ssl.transport.pemtrustedcas_filepath: root-ca.pem
  searchguard.ssl.transport.enforce_hostname_verification: false
  searchguard.ssl.http.enabled: true
  searchguard.ssl.http.pemcert_filepath: esnode.pem
  searchguard.ssl.http.pemkey_filepath: esnode-key.pem
  searchguard.ssl.http.pemtrustedcas_filepath: root-ca.pem
  searchguard.allow_unsafe_democertificates: true
  searchguard.allow_default_init_sgindex: true
  searchguard.authcz.admin_dn:
    - CN=kirk,OU=client,O=client,L=test, C=de
  
  searchguard.audit.type: internal_elasticsearch
  searchguard.enable_snapshot_restore_privilege: true
  searchguard.check_snapshot_restore_write_privileges: true
  searchguard.restapi.roles_enabled: ["SGS_ALL_ACCESS"]
  cluster.routing.allocation.disk.threshold_enabled: false
  cluster.name: searchguard_demo
  node.max_local_storage_nodes: 3
  xpack.security.enabled: false
  ######## End Search Guard Demo Configuration ########
  ```

+ 测试

  ``` bash
  # 启动ES
  systemctl start elasticsearch
  # 测试HTTPS
  curl -k https://localhost:9200/_searchguard/authinfo?pretty
  # 如果使用浏览器就需要接受一下自签证书
  ```

+ 登录界面的时候使用admin/admin登录

### 1.2. 安装SearchGuard for Kibana

+ 下载kibana插件，[下载地址](https://docs.search-guard.com/latest/search-guard-versions)，同样需要版本对应

+ 安装

  在线安装：

  ``` bash
  bin/kibana-plugin install https://url/to/search-guard-kibana-plugin-<version>.zip 
  ```

  下载安装：

  ``` bash
  bin/kibana-plugin install file:///path/to/search-guard-kibana-plugin-<version>.zip
  ```

+ 完成后修改/etc/kibana/kibana.yml

  ``` bash
  # Use HTTPS instead of HTTP
  elasticsearch.hosts: "https://localhost:9200"
  
  # Configure the Kibana internal server user
  elasticsearch.username: "kibanaserver"
  elasticsearch.password: "kibanaserver"
  
  # Disable SSL verification because we use self-signed demo certificates
  elasticsearch.ssl.verificationMode: none
  
  # Whitelist the Search Guard Multi Tenancy Header
  elasticsearch.requestHeadersWhitelist: [ "Authorization", "sgtenant" ]
  
  # X-Pack security needs to be disabled for Search Guard to work properly
  xpack.security.enabled: false
  ```

+ 启动kibana，这次启动时间比较长，kibana会把search guard的配置文件（<Elasticsearch directory>/plugins/search-guard-7/sgconfig）都加载到elasticsearch中，在elasticsearch中就可以看到一个叫SearchGuard的索引

  ``` bash
  systemctl start kibana
  ```

  + PS：如果不使用kibana，我们就需要使用命令来让demo生效<Elasticsearch directory>/plugins/search-guard-7/tools/sgadmin_demo.sh

+ 登录 http://localhost:5601/就会看到界面了，使用admin/admin登录

  ![image-20200826105021300](/pages/keynotes/L4_architect/3_logging/pics/3_install_plugins/image-20200826105021300.png)

### 1.3. Search Guard GUI

我们可以通过三种方式来管理SearchGuard，sgadmin命令行，REST API和Kibana上面的图形界面。当然，最简便的方式还是图形界面了，他可以配置，用户，角色和权限。点击左上角的展开图标。

![image-20200826105845354](/pages/keynotes/L4_architect/3_logging/pics/3_install_plugins/image-20200826105845354.png)

选择Search Guard Configuration出现配置界面

![image-20200826105939540](/pages/keynotes/L4_architect/3_logging/pics/3_install_plugins/image-20200826105939540.png)

在新版中，我们看到还有一个选项叫`Search Guard Signals`，这是新的报警功能。目前7.8.1版本只有Email，slack，Jira和pageduty几种报警方式

![image-20200826110912773](/pages/keynotes/L4_architect/3_logging/pics/3_install_plugins/image-20200826110912773.png)

### 1.4. 概念

+ Internal Users database：本地的账户。SG是支持其他认证方式的，比如LDAP等，详细配置看[这个](https://docs.search-guard.com/latest/demo-installer)。在sg里面的认证（authentication）简称authc，授权（authoriztion）叫authz
+ Tenants：相当于Openstack-M版中的租户或者R版中的project，AWS中的账户和子账户，confluence中的user space。多租户功能是收费的，免费的只能使用admin一个租户
+ Action Group：实际上就是组的概念，不过SG内置了很多的SGS开头的权限组，而action group就是把它们组合在一起，而这些权限又分为cluster，index和single级别。single级别就是针对某个文档或者字段的控制了，这个是收费功能。
+ Role：角色需要分别定义集群级别的，索引级别的，租户级别的权限，每个级别下面会有一个滑动按钮，叫`Advanced`点开就会出现一些细粒度的授权，这个里面的功能全部都是收费的。
+ Role mapping：由于角色和用户是多对多的关系，所以这里增加了一个Role mapping，把一个角色分配给很多的用户，而`Internal Users database`是把很多角色分配给一个用户。

### 1.5. Demo

#### 1.5.1. Demo 1：官方demo中提供的一些用户和权限

我们在运行了Demo脚本之后，会提供一些用户和权限，我们先来看看他们

+ 打开图形界面，点击`Search Guard Configuration`--> `Internal Users database`，我们可以看到非常多的用户

+ kibanaserver：我们在配置kibana的时候，使用的用户是kibanaserver，但是我们并没有发现这个用户，这是因为用户有两种，`reserved: true`，`reserved: false`，这个是在<Elasticsearch directory>/plugins/search-guard-7/sgconfig/sg_internal_users.yml中定义的

+ kibanaro：就是kibana read-only的缩写，这个账户可以看kibana界面，比如dashboard之类，但是无法管理索引，他有两个Backend Roles，`readall`和``，但是我们在配置文件（<Elasticsearch directory>/plugins/search-guard-7/sgconfig/sg_roles.yml）中并没有找到他们，因为他们是内建的角色，可以参考[这个](https://docs.search-guard.com/latest/roles-permissions)
+ logstash是类似的：只有一个组，这个是给logstash和beats的agent使用的，拥有logstash和beats的索引的所有权限
+ readall：可以读所有索引，但是不能写
+ snapshotrestore：可以创建和回复快照，主要是给自动化脚本用的

### 1.5.2. Demo 2：kibana用户权限

+ 我们创建一个用户，叫my-kibana，并且设置密码为my-kibana，不给任何权限
+ 登录kibana界面，会返回403错误
+ 修改my-kibana用户的角色，给他kibanauser的权限，就可以登录了，但是没办法看索引
+ 修改my-kibana用户的角色，给他readall的权限，就可以看discovery了

### 1.5.3. Demo 3：写权限

+ 我们配置一个fluent-bit的客户端，使用admin/admin账户来和ES通信
+ 我们把admin账户改成logstash账户，然后就会报错，没有权限
+ 我们把索引的名字改一下，从fluent-bit改成logstash，就没有问题了

### 1.5.4. Demo 4：创建自定义的角色

+ 我们为安全组创建一个权限，可以管理SG的审计日志，其他不允许看的权限，管理当然包括增删改查

+ 我们创建一个action group叫read-sg-audit，Type是Index，他的action group是SGS_INDICES_ALL，点击save

+ 创建一个Role，叫security-role，在index permissions选add，patterns手动输入sg7-*，action group选read-sg-audit，点击save

+ 创建用户security，为了看到效果，我们给他一个backend role叫kibanauser，点击save

+ 点击role mapping，选择security-role，Users选security，点击save

+ 使用security用户登录界面，这个用户只能看到和sg7-*有关的日志，其他的看不到

+ 使用http方式查询es的api

  ``` bash
  curl -k -u security:security 'https://localhost:9200/sg7-auditlog-2020.08.27/_doc/pRcQLnQB2xpX-EqCM2Nt?pretty'
  ```

+ 删除这个doc

  ``` bash
  curl -k -XDELETE -u security:security 'https://localhost:9200/sg7-auditlog-2020.08.27/_doc/pRcQLnQB2xpX-EqCM2Nt?pretty'
  ```

  

### 1.6. 收费模式

SG有三个版本，社区版（community edition），专业版（enterprise edition），合规版（compliance edition），具体参考[这个](https://search-guard.com/licensing/)

+ 社区版：免费使用，和收费版本的区别主要有三点
  + 权限控制的粒度：免费版，只支持集群和索引级别的控制，而收费版可以精细到文档和字段级别，最高级的还可以支持匿名字段和固定某个字段不可删除
  + 认证和授权的方式：免费版，支持内部用户认证，http认证，PKI认证，代理认证。授权只有内部授权，和proxy header的授权，而收费版支持的很多很多，详见上面的地址
  + 配置方式：免费版，只可以使用命令行来配置，收费版可以使用图形界面来配置
+ 专业版与合规版的区别主要是审计功能，专业版是没有审计功能的

### 1.7. sgadmin的使用

详细请参考[官方文档](https://docs.search-guard.com/latest/sgadmin-basic-usage)

## 2. 分词器系列

分词器是ES的特色之一，简单来说，分词器就是告诉ES怎样拆分一个句子中的词语，从而构建搜索的关键字的。比如，我们是小学生，那么分词器会把我们，小学生，作为搜索关键字来储存。一般来说，中文是需要中文的分词器的。而张三是小学生，就没办法识别，因为张三本身不是一个词，是一个名字，名字就无法被认为是一个词，而在实际生活中，明星的名字很有可能就是一个关键词，我们就需要手动为分词器添加关键字，也就是支持自定义关键字的分词器。比较常见的分词器有IK分词器，pinyin分词器。大家可以自己到[这里](https://github.com/medcl/)去下载，使用教程也比较详细，我们不着重说了。

