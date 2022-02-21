---
title: Prometheus概览
keywords: keynotes, senior, logging, fluentd_overview
permalink: keynotes_L3_senior_6_logging_2_fluentd_overview.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/2_fluentd_overview
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

+ 认识fluentd，搭建fluentd服务

## 1. fluentd

fluentd是一款开源的数据收集器，支持统一数据收集和使用，从而更好的利用和理解数据

```
Fluentd is an open source data collector, which lets you unify the data collection and consumption for a better use and understanding of data.
```

使用fluentd之前

![img](/pages/keynotes/L3_senior/6_logging/pics/2_fluentd_overview/fluentd-before.png)

使用Fluentd之后

![img](/pages/keynotes/L3_senior/6_logging/pics/2_fluentd_overview/fluentd-architecture.png)

fluentd就像一个管道一样，把数据源**统一**收取，通过filter/buffer/routing等组件，对数据进行格式化之后发送到后端。

## 2. 主要功能

- 统一日志格式为JSON

  fluentd尽量把所有的数据都转化为JSON格式，这样可以更容易的对多种数据源和输出源进行收集，过滤，缓存和输出

![img](/pages/keynotes/L3_senior/6_logging/pics/2_fluentd_overview/log-as-json.png)

- 可插拔的架构

  一个完整的日志传输过程包概，Input,Filter,Parser,Formater,Buffer，这些组件都支持自定义，我们可以Input各种类型的日志，经过一系列可以定制的转换，输出到各种类型的output，这些东西都是以插件相识存在的，我们可以通过td-agent-gem plugin list来查看安装了什么样的组件。

![img](/pages/keynotes/L3_senior/6_logging/pics/2_fluentd_overview/pluggable.png)

- 需求资源很少

  Fluentd的核心是C语言，而插件是由Ruby开发的，消耗资源比Logstash那种JRuby的方式要小的多的多

![img](/pages/keynotes/L3_senior/6_logging/pics/2_fluentd_overview/c-and-ruby.png)

- 内置稳定性功能

  Fluentd的引擎包含了Buffer，日志，debug，metrics等各种功能，保证整个程序的稳定性。

## 2. 使用Fluentd挖日志

### 2.1. 单机安装fluentd

[官方文档](https://docs.fluentd.org/installation/install-by-rpm)中提到了，目前Fluentd有两个版本V3和V4，我们这次使用redhat系统来做demo，选择v4的fluentd来安装。而程序的名字叫td-agent

``` bash
# td-agent 4
$ curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent4.sh | sh
```

这个脚本中其实就是添加了一个yum源，然后yum install了一下

``` bash
echo "=============================="
echo " td-agent Installation Script "
echo "=============================="
echo "This script requires superuser access to install rpm packages."
echo "You will be prompted for your password by sudo."

# clear any previous sudo permission
sudo -k

# run inside sudo
sudo sh <<SCRIPT

  # add GPG key
  rpm --import https://packages.treasuredata.com/GPG-KEY-td-agent

  # add treasure data repository to yum
  cat >/etc/yum.repos.d/td.repo <<'EOF';
[treasuredata]
name=TreasureData
baseurl=http://packages.treasuredata.com/4/redhat/\$releasever/\$basearch
gpgcheck=1
gpgkey=https://packages.treasuredata.com/GPG-KEY-td-agent
EOF

  # update your sources
  yum check-update

  # install the toolbelt
  yes | yum install -y td-agent

SCRIPT

# message
echo ""
echo "Installation completed. Happy Logging!"
echo ""
```

安装完成之后，我们可以在系统的systemd中找到这个服务

``` bash
systemctl status td-agent
systemctl start td-agent
```

### 2.2. 使用Fluentd代替logstash挖日志