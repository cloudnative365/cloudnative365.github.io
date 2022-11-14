---
title: 各种beats
keywords: keynotes, senior, logging, beats
permalink: keynotes_L3_senior_6_logging_6_beats.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/6_beats
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

beats是用来安装在被采集端上，用来采集各种信息的程序。他们通常和被采集端装在同一个空间里面，比如一个虚拟机，一个实例或者一个pod中。以前的经典架构ELK是使用logstash来作为agent的，但是由于logstash实在太笨重，对于目前容器化环境不太友好，官方针对不同的功能进行了拆分和代码重构，也就是现在的beats。那么logstash就不再用了么？当然不是。logstash的功能包含了一条信息从产生到发送到ES的整个生命周期，比如，拿日志，过滤日志，添加内容，整理，变形到最后的传递给ES。由于beast承担了拿日志和传递日志的功能，中间的环节的功能，比如过滤，整理等，就不那么强大了。因为中间的这些过程，我们一般都是通过插件来完成的，这些插件的产生是由整个互联网的需求而产生，这些开发者又把插件贡献给了社区，因此logstash的功能就不停的壮大。所以logstash并不是被取代，而是被解耦！

因此，新的架构就变成了beats-->logstash-->elasticsearch-->kibana这种架构。

![image-20221112142258701](/pages/keynotes/L3_senior/6_logging/pics/6_beats/image-20221112142258701.png)

我们只有明白了这种架构才算是真正入门了ES的架构

## 2. beats

### 2.1. 各种beat

官方一共给出了7种[beats](https://github.com/elastic/beats)，这7种beats都是开源的，遵循apache协议

![image-20221112145509418](/pages/keynotes/L3_senior/6_logging/pics/6_beats/image-20221112145509418.png)

但是最常用的还是filebeats，毕竟大部分人用es还是用来存日志，虽然es本身也可以存metrics，甚至自己也有APM组件，但是使用的公司并不是很多。

### 2.2. filebeat

顾名思义，就是用来挖文件的agent

[filebeat](https://github.com/elastic/beats/tree/main/filebeat)是由golang语言开发的，性能非常强劲。但是日志这种东西，生成的速度非常之快，而filebeat挖掘的也快，因此，系统非常繁忙，产生的日志非常大量，但是都不是必须保留的，而把日志配置成滚动更新的。对于应用而言，写满了日志就换下一个文件，都写满了就开始写第一个。而filebeat就会不停的挖日志，而golang是多线程的，如果没有控制好，就会产生filebeats比系统还忙的情况，消耗大量资源。

我们来尝试一下，依然是在我的mac笔记本上，我们接着2_install_elk的环境来做。

+ 下载，解压beats到指定位置

  ``` bash
  mkdir /Users/jormun/Documents/Apps/filebeat
  cd /Users/jormun/Documents/Apps/src
  wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.17.7-darwin-x86_64.tar.gz
  tar xf filebeat-7.17.7-darwin-x86_64.tar.gz -C /Users/jormun/Documents/Apps/filebeat
  cd /Users/jormun/Documents/Apps/filebeat/filebeat-7.17.7-darwin-x86_64
  ```

+ 修改配置文件filebeat.yml

  ``` bash
  filebeat.inputs:
  - type: filestream
  
    # Unique ID among all inputs, an ID is required.
    id: my-filestream-id
  
    # Change to true to enable this input configuration.
    enabled: true
  ```
  
+ 启动filebeat

  ``` bash
  ./filebeat
  ```

+ 我们就可以在kibana中看到我们的日志了

  ![image-20221112163237894](/pages/keynotes/L3_senior/6_logging/pics/6_beats/image-20221112163237894.png)

## 3. logstash

### 3.1. Input

官方对于beats类的input单独给了一个input类，叫beats。logstash会监听在某一个端口之上，等待beats发过请求，然后做处理。和原来的file的input类似，也可以在传递给es之前做一系列的转化和过滤。

我们修改conf.d下的配置文件为

``` bash
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
}
```

启动logstash等待filebeats发送文件

### 3.2. Beats

filebeats默认是output到elasticsearch，我们把es相关的配置注释掉，改到发送给logstash

``` bash
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]

output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]
```

启动beats就可以看到数据了
