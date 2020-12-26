---
title: 日志收集工具
keywords: keynotes, architect, logging, 6_log_agent
permalink: keynotes_L4_architect_3_logging_6_log_agent.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/6_log_agent
typora-root-url: ../../../../../cloudnative365.github.io
---

## 学习目标

了解常用的日志收集工具

搭建fluentd和fluent-bit

## 1. 日志收集

日志收集从日志系统搭建的角度是第二步，在日志收集的过程中是第一步。日志需要被发送或者采集到统一的位置才可能被利用起来。日志统一发送的目的有两个，可能是直接发送到了存储，比如：logstash直接发送到ES。也有可能是发送到了代理节点，然后在代理节点做处理之后才向存储发送，比如：filebeat发送日志到logstash，经过整理之后发送给ES。

那么，问题来了。

1. 如果日志是主动发送的，怎样保证日志在发送的过程中是不丢数据的？我们知道，rsyslog或者是syslog协议是UDP的，也就是说，如果发送数据，发送端只管发，而不管服务端时候能接收到。所以这种方式不太适合比较重要的日志，比如应用日志，HTTP请求等等，我们会用另外一种方式来处理这些日志。这种方式比较适合那种非常大量的，而且无法安装agent的场景，比如：交换机，路由器日志。
2. 如果日志是通过log agent从设备上抽取的（俗称叫挖日志），那么在抽取的过程中，需要指定日志的位置，在读取的过程中需要设置内存的大小和一次性发送的数量，而且在发送的过程中如果有time out，或者404之类的情况，可以选择把没有发送的日志存起来，或者记录下发送日志的位置，下次再重新发送。
3. 从服务端的角度，如果某一个日志接收器（比如ES或者collector挂了），agent会重新向另外的日志接收器发送日志，直到成功，指针才会转移向下一段日志。而日志接收器之间也需要一些同步来保证数据的一致性和单一性，不会出现丢日志或者日志重复的情况。比如ES或者kafka这种本身有集群机制的接收器可能好理解，如果我们的接收器是fluentd或者logstash这种的中转站应该如何处理呢？
4. 日志是有格式的，syslog有syslog的格式，比如：syslog-rfc5424，syslog-rfc3164，nginx有nginx的格式，如果是自定的log，就需要自定义一个日志格式，自定义日志格式就需要使用正则表达式来把一段string类型的日志变成对应的格式。比如：` ^\<(?<pri>[0-9]{1,5})\>1 (?<time>[^ ]+) (?<host>[^ ]+) (?<ident>[^ ]+) (?<pid>[-0-9]+) (?<msgid>[^ ]+) (?<extradata>(\[(.*)\]|-)) (?<message>.+)$`

我们带着上面的思考去重新审视我们后面要做的事情，就可以清晰的理解到一些日志系统的真谛，从而升华我们在应对日志系统架构问题时候的境界。也许目前我们的眼光并没有这么犀利和透彻，所以我建议大家在学完之后再回头来看一下这些问题，他们会全部迎刃而解。

## 2. 常见的agent

每种agent实际上都对应一个日志的存放机制，比如ES，Loki，HBase等。但是，大部分agent不会只支持某一种数据库。如果把agent比作一个管道的话，那么一段承接着我们要采集的日志，叫input，而另一端是把日志输出的媒介，叫output。input可以有很多种，output也有很多种。而在传输过程中的一些处理，我们习惯叫他filter。而有些agent对filter做了更加详细的分类，比如fluntd，他的filter有filter，formatter和parser。

### 2.1. fluentd

我们这套课程的核心其实是fluentd，因为我们讲的是云原生，而fluentd是CNCF早期的毕业项目，毕业时间是2019年4月，毕业时间距离kubernetes和prometheus的毕业时间只是滞后了一点点。我们后面会对他做非常详细的讲解。

### 2.2. fluent-bit

这款产品和fluentd的母公司Treasure Data的另外一款产品，尽管没有托管在cncf，但是他依然是完全免费开源使用的，在资源消耗上比fluentd要小很多，但是插件没有fluentd丰富，用来做一些简单的日志收集是非常适合的。我们后面也会着重讲解。

### 2.3. logstash

这是非常经典的ELK架构中的L，logstash依托于ES的烘托，也算是个不错的选择，但是由于资源消耗问题一直被社区诟病，因为他的基于JVM开发的，使用Jruby语言构建的软件，java的效率大家有目共睹，再加上JRUBY这种反人类的架构，让原来的ELK架构演变为EFK架构，大家纷纷使用Fluentd来取代Logstash来收集日志。

### 2.4. Filebeats和各种beat

ES的公司把ES的生态包括ES都统称为elastic stack，而他们的agent，除了logstash，都统称为xxxbeat。我们前面讲ES产品的时候说过了，比如：filebeat，windlogbeat。而logstash被定义为`采集、转换、充实，然后输出。`也就是对日志做处理的工具。

### 2.5. Promtail

grafana lab的产品，主要是和日志工具Loki所对应的一款日志收集工具。他的功能只限于日志收集，他所能兼容的input和output的产品类型非常有限，但是和Loki的兼容性是非常好的，也是目前唯一一个和Loki兼容性好的工具。官方文档中提到了fluentd也可以通过加载动态链接库`.so`的方式来支持Loki作为output，但是咱们的课程里面不会涉及，有兴趣的可以自己试一下。

### 2.6. flume

大数据项目中常用的一种方式，他的优点在于分布式。因为他可以依托于zookeeper来做集群，所以他可以近乎无限的扩展节点，且稳定性很高。但是基本上只有一些涉及到HDFS或者HBase的项目会用到这个工具。咱们课程中不会涉及到他。

### 2.7. 其他产品

其实还有很多商用的产品，比如splunk，sysdig还有IBM的logdns都有自己的解决方案，而且适配性也很好，但是缺点都是一样的，那就是收费。但是，如果没有太大的数据量，但是对于日志有高要求的公司，使用产品也是不错的选择。

## 3. 安装fluentd

我们就来先亲身体验一下fluentd。文档地址：https://docs.fluentd.org/。这次我们使用CENTOS8的版本来安装。

### 3.1. 准备

+ ntp服务器，可以选择chrony或者ntpd，私有云可以用vmware-tools，公有云比如AWS就用AWS自己的ntp服务器

+ 修改打开文件的个数，一般默认是1024，我们建议修改到65535

  ``` bash
  # 查看现在的
  ulimit -n
  ```

  修改`/etc/security/limits.conf`文件

  ``` bash
  root soft nofile 65536
  root hard nofile 65536
  * soft nofile 65536
  * hard nofile 65536
  ```

  PS: 如果我们使用安装包安装的话，安装包会修改这个参数

+ 优化内核参数

  ``` bash
  net.core.somaxconn = 1024
  net.core.netdev_max_backlog = 5000
  net.core.rmem_max = 16777216
  net.core.wmem_max = 16777216
  net.ipv4.tcp_wmem = 4096 12582912 16777216
  net.ipv4.tcp_rmem = 4096 12582912 16777216
  net.ipv4.tcp_max_syn_backlog = 8096
  net.ipv4.tcp_slow_start_after_idle = 0
  net.ipv4.tcp_tw_reuse = 1
  net.ipv4.ip_local_port_range = 10240 65535
  ```

### 3.2. 安装

请看文档中关于[安装](https://docs.fluentd.org/installation/install-by-rpm)这一节，安装中我们会发现有两个版本3和4，具体区别可以看[这里](https://docs.fluentd.org/quickstart/td-agent-v2-vs-v3-vs-v4)。主要区别还是对于ARM芯片的支持。我们的实验环境的基于X86的，那么我们选哪个都可以，为了显示兼容性，我们选用最新的4版本。

``` bash
# td-agent 4
$ curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent4.sh | sh

# td-agent 3
$ curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent3.sh | sh
```

如果不使用安装脚本，还可以使用rpm包的方式来安装

``` bash
wget http://packages.treasuredata.com.s3.amazonaws.com/4/redhat/8/x86_64/td-agent-4.0.1-1.el8.x86_64.rpm
```



### 3.3. 结构和目录

安装好之后，我们可以找到一个叫做td-agent的systemd配置文件，启动软件

``` bash
systemctl start td-agent
systemctl enable td-agent
```

他的配置文件位于/etc/td-agent下面，有两个，一个是td-agent.conf，是主配置文件，另外的`plugin`是插件，我们以后用到的各种扩展功能都是放在这里面的。

而主要的文件都是在`/opt/td-agent/`下面，比如bin，lib，share，include这些

### 3.4. 安装插件

我们可以去查询[官网](https://www.fluentd.org/plugins)去找相关的插件，其实那些都是常用的，而所有的插件列表在[这里](https://www.fluentd.org/plugins/all)，在册的目前有657个插件。安装插件需要使用gem命令，这个命令在`/opt/td-agent/bin/gem`，我们可以直接使用gem install命令来安装插件。比如要安装kafka插件

``` bash
# 官方文档
gem install fluent-plugin-kafka
# 实际上的操作
/opt/td-agent/bin/gem install fluent-plugin-kafka
Fetching fluent-plugin-kafka-0.15.3.gem
Successfully installed fluent-plugin-kafka-0.15.3
Parsing documentation for fluent-plugin-kafka-0.15.3
Installing ri documentation for fluent-plugin-kafka-0.15.3
Done installing documentation for fluent-plugin-kafka after 0 seconds
1 gem installed
```

当然，每种插件的格式是不一样的，大家最好去git上查看，比如kafka插件的[地址](https://github.com/fluent/fluent-plugin-kafka)，也有git上没有说明的，那就要去看源码中的注释了，我们以后会经常来实验一些常用插件

### 3.5. 读取文件内容，输出到终端上

这里做一个小Demo，虽然小，但是这是大家用来trouble shooting的常用手段，用来判断能不能读取到数据源中的数据，能读取，在看输出格式，配置文件如下

``` bash
<source>
  @type tail
  @id demo
  <parse>
    @type apache2
  </parse>
  path /var/log/httpd/access_log
  tag apache2log
</source>

<match apache2log>
  @type stdout
</match>
```

这个时候我们访问http的日志就会被输出到屏幕上，当然，也可以match到文件中

``` bash
<match pattern>
  @type file
  path /var/log/fluent/apache2demo
</match>
```

path是文件夹的名字，生成的文件会有很多种，我们讲到这个的时候再说，如果是文件的话会生成这样的文件

``` bash
buffer.b5b7602e55a2c26f5f505af9dae5a1622.log  buffer.b5b7602e55a2c26f5f505af9dae5a1622.log.meta
```

感觉上这就是一个数据库，有meta，还有内容