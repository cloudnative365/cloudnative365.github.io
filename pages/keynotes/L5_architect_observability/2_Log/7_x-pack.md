---
title: x-pack
keywords: keynotes, architect, logging, 7_x-pack
permalink: keynotes_L4_architect_3_logging_7_x-pack.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/7_x-pack
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 概述

在早起的ES版本中，我是说6.2版本之前，ES的安全功能是需要额外安装插件的，也就是我们熟知的x-pack，需要通过`elasticsearch-plugin install`的方式来安装。在6.2版本之后，官方把这个功能整合进了ES，我们可以通过配置文件直接开启，而不需要额外安装plugin了。在官方文档中，也在标题部分抹去了x-pack的痕迹，而全部都叫做[安全](https://www.elastic.co/guide/en/security/current/index.html)功能（SIEM）。 而安全功能实际上整合了SIEM线程探测（SIEM threat detection features），端点保护（endpoint prevention ）和响应能力（response capabilities）。

### 1.1. 官方版本的区别

elastic stack免费版提供的安全功能是非常有限的，在官方文档中的描述叫核心安全功能，而收费的叫高级安全功能，请参考[官方网站](https://www.elastic.co/cn/subscriptions)

![image-20200904094700864](/pages/keynotes/L4_architect/3_logging/pics/7_x-pack/image-20200904094700864.png)

其实我觉得他已经把一些亮点都总结的很好了

+ 基础级：免费
+ 黄金级：亮点在于kibana的监控和报警功能
+ 白金级：亮点在于安全功能，也就是x-pack的全套功能
+ 企业级：所有功能

### 1.2. 安全组件的工作流

![Elastic Security workflow](/pages/keynotes/L4_architect/3_logging/pics/7_x-pack/workflow.png)

## 1. 开启安全认证

在ES的7.x版本中，basic认证是免费的功能，我们只需要在elasticsearch的配置文件中添加下面的配置就可以了

``` bash
# x-pack security configuration
xpack.security.enabled: true
xpack.license.self_generated.type: basic
```

但是，这样会启动不了，会报错

``` bash
[1] bootstrap checks failed
[1]: Transport SSL must be enabled if security is enabled on a [basic] license. Please set [xpack.security.transport.ssl.enabled] to [true] or disable security by setting [xpack.security.enabled] to [false]
```

提示我们配置ssl

``` bash
xpack.security.transport.ssl.enabled: true
```

但是启动了ssl之后，需要我们生成证书

``` bash
elasticsearch-certutil ca
.
.
.
Please enter the desired output file [elastic-stack-ca.p12]: 
Enter password for elastic-stack-ca.p12 : 
```

ca证书的生成位置在`/usr/share/elasticsearch/elastic-stack-ca.p12`，然后通过这个ca签署证书

``` bash
elasticsearch-certutil cert --ca /usr/share/elasticsearch/elastic-stack-ca.p12
.
.
.
Enter password for CA (/usr/share/elasticsearch/elastic-stack-ca.p12) : 
Please enter the desired output file [elastic-certificates.p12]: 
Enter password for elastic-certificates.p12 : 
# 证书生成的位置
Certificates written to /usr/share/elasticsearch/elastic-certificates.p12
```

我们把证书放到配置文件目录`/etc/elasticsearch/certs`下面

``` bash
mkdir /etc/elasticsearch/certs
mv /usr/share/elasticsearch/elastic-certificates.p12 /etc/elasticsearch/certs
```

最后，所以我们的配置文件最后是这个样子的

``` bash
# x-pack security configuration
xpack.security.enabled: true
xpack.license.self_generated.type: basic
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/certs/elastic-certificates.p12
```

由于我刚才在生成证书的时候还配置了密码，所以还需要执行下面的命令（如果生成证书的时候没有密码，下面的就省略）

``` bash
elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

## 2. 配置用户

使用命令行配置默认的用户

``` bash
elasticsearch-setup-passwords interactive
.
.
.
Changed password for user [apm_system]
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

看名字就能知道是干什么的了吧

| 用户名                 | 作用                                                    |
| ---------------------- | ------------------------------------------------------- |
| elastic                | 超级用户                                                |
| apm_system             | 给apm用的                                               |
| kibana_system          | 负责kibana连接elasticsearch                             |
| kibana                 | 旧版本中使用的，马上要被废弃掉，使用kibana_system就好了 |
| logstash_system        | 给logstash用的                                          |
| beats_system           | 给各种beats用的                                         |
| remote_monitoring_user | 给metricsbeats用的                                      |

## 3. kibana配置

可以在配置文件中写密码，但是不太建议这么做

``` bash
elasticsearch.username: 
elasticsearch.password:
```

通常我们建议使用keystore，也就是kibana.keystore

``` bash
kibana-keystore create --allow-root
kibana-keystore add elasticsearch.username --allow-root
kibana-keystore add elasticsearch.password --allow-root
# 删除
kibana-keystore remove xxx
```

再次登录kibana界面，使用elastic用户进入控制台就好了

## 4. 配置Elasticsearch和kibana的ssl通信

