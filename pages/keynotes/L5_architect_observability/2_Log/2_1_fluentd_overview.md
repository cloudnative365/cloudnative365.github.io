---
title: fluentd概览
keywords: keynotes, architect, observability, log, fluentd, fluentd_overview
permalink: keynotes_L5_architect_observability_2_log_2_1_fluentd_basic.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/2_1_fluentd_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ 认识fluentd，搭建fluentd服务

## 1. fluentd

fluentd是一款开源的数据收集器，支持统一数据收集和使用，从而更好的利用和理解数据

``` bash
Fluentd is an open source data collector, which lets you unify the data collection and consumption for a better use and understanding of data.
```

使用fluentd之前

![img](/pages/keynotes/L5_architect_observability/2_log/pics/2_1_fluentd_overview/fluentd-before.png)



使用Fluentd之后

![img](/pages/keynotes/L5_architect_observability/2_log/pics/2_1_fluentd_overview/fluentd-architecture.png)

fluentd就像一个管道一样，把数据源统一收取，通过filter/buffer/routing等组件，对数据进行格式化之后发送到后端。我们可以从图上看出，fluentd支持非常多的input和output类型。实际上filter和buffer之类也是通过插件的形式对社区敞开大门，让更多的开发者可以参与到项目当中。

## 2. 主要功能

+ 统一日志格式为JSON

  fluentd尽量把所有的数据都转化为JSON格式，这样可以更容易的对多种数据源和输出源进行收集，过滤，缓存和输出

![img](/pages/keynotes/L5_architect_observability/2_log/pics/2_1_fluentd_overview/log-as-json.png)

+ 可插拔的架构

  软件自带的插件数量很少，大部分的插件都是来自于社区的贡献，这些插件随用随安装，这就使得整个程序非常精简，用到什么就安装什么，不用也可以随时卸载。

![img](/pages/keynotes/L5_architect_observability/2_log/pics/2_1_fluentd_overview/pluggable.png)

+ 需求资源很少

  这里的资源占用少是相对于传统的日志传输工具logstash来说的，因为logstash是Jruby开发的，开发语言是ruby，但是需要跑在JVM上，JVM这个内存老虎的厉害大家估计都有所感触。而fluentd的核心是C，而插件是ruby，既提高了效率，又对开发者非常的友好。当然，在fluentd刚出现的时代，golang+js或者golang+python这种组合并没有成熟。否则，怎么也轮不到ruby来做插件开发语言。

![img](/pages/keynotes/L5_architect_observability/2_log/pics/2_1_fluentd_overview/c-and-ruby.png)

+ 内置稳定性功能

  fluentd更多的是提供了一个框架，所以里面内置和非常多的功能，比如tail文件，http服务器，缓存，日志追踪，我们只需要关注我们的数据传输逻辑就好了，其他的就交给fluentd来做就可以。

![img](/pages/keynotes/L5_architect_observability/2_log/pics/2_1_fluentd_overview/reliable.png)

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