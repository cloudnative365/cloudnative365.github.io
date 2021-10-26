---
title: fluentd简单上手
keywords: keynotes, architect, logging, fluent-bit
permalink: keynotes_L4_architect_3_logging_13_fluentd_demo.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/13_fluentd_demo
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

使用fluent-bit收集服务器上的日志

## 1. 简介

fluent-bit可以说是fluentd的轻量重构版本，他的资源使用率只有fluentd的百分之一，可以说是非常的轻量。虽然项目本身并没有fluentd那样知名，但是由于他的轻量，非常适合作为sidecar或者agent运行在pod或者机器上是非常合适的，用来挖日志，而不做任何处理，如果有需要处理的就丢给运行在另外pod或者节点上的fluentd来统一做处理。从而实现了日志的ETL过程，最后由fluentd再发送给存储，比如ES或者HBase做最终处理。

``` bash
----------------------       -----------     -----------------
|         |           |      |         |     |               |
|container|fluent-bit |=====>| fluentd |====>| Elasticsearch |
|         |           |      |         |     |               |
----------------------       -----------     -----------------
```

或者是服务器

``` bash
----------------------      -----------     -----------------
| server             |      |         |     |               |
|     td-agent-bit   |====> | fluentd |====>| Elasticsearch |
|                    |      |         |     |               |
----------------------      -----------     -----------------
```

## 2. Demo

### 2.1. 安装

我们这里以centos为例来安装，我们可以选择使用yum源或者直接rpm安装

+ 下载地址：https://docs.fluentbit.io/manual/installation/linux/redhat-centos，这边只能通过配置yum repo的方式来安装，如果我们的机器无法连接外网，请找一台可以上网的机器，使用`yum install td-agent-bit --downloadonly --downloaddir=/tmp`把包下载之后再上传到需要安装的机器再安装。`yum install -y td-agent-bit-1.6.6-1.x86_64.rpm`

+ 如果可以使用外网，配置yum repo

  ``` bash
  [td-agent-bit]
  name = TD Agent Bit
  baseurl = https://packages.fluentbit.io/centos/7/$basearch/
  gpgcheck=1
  gpgkey=https://packages.fluentbit.io/fluentbit.key
  enabled=1
  ```

+ 安装

  ``` bash
  yum install -y td-agent-bit
  ```

+ 启动服务

  ``` bash
  systemctl start td-agent-bit && systemctl enable td-agent-bit
  ```

### 2.2. 配置

我们这里配置一下采集系统日志/var/log/messages。首先看配置文件的位置在`/etc/td-agent-bit`下面，有一个`td-agent-bit.conf`文件。

``` bash
[SERVICE]
# 这里面是全局配置
[INPUT]
# 数据源的位置
[OUTPUT]
# 输出日志的位置
```

我们只修改input和output，从本地文件，输出到es

``` bash
[INPUT]
  Name              tail
  Tag               syslog.messages
  Path              /var/log/messages

[OUTPUT]
  Name          es
  Match         syslog.messages
  Host          elasticsearch.jormun.com
  Port          9200
```

重启服务

``` bash
systemctl restart td-agent-bit
```

## 3. 生产级别的配置

上面的例子非常简单，因为没有任何额外的配置，但是这个显然不符合我们的要求，考虑到我们在前面讲agent的时候提到的需要考虑的问题，我们的配置应该如下

``` bash
[INPUT]
  Name              tail
  Tag               syslog.messages
  Path              /var/log/messages
  Parser            syslog-rfc5424
  DB                /var/log/td-agent-bit/syslog.messages.db
  Mem_Buf_Limit     5MB
  Refresh_Interval  5
  Skip_Long_Lines   On

[FILTER]
  Name              record_modifier
  Match             syslog.messages
  Record            hostname ${HOSTNAME}
  Record            agent_type td-agent-bit

[OUTPUT]
  Name          es
  Match         syslog.messages
  Host          elasticsearch.jormun.com
  Port          9200
  HTTP_User     fluentbit_system
  HTTP_Passwd   fluentbit_system
  Logstash_Format  On
  Logstash_Prefix  fluent-bit
```

