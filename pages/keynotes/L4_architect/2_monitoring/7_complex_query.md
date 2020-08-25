---
title: prometheus复杂查询
keywords: keynotes, architect, monitoring, complex_query
permalink: keynotes_L4_architect_2_monitoring_7_complex_query.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/7_complex_query
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

了解比较复杂的PromSQL查询

## 1. 简介

我们今天主要就是要通过一个非常复杂的例子来帮大家了解我们日常生产系统中的查询是怎样组成的。我们的例子来自于我曾经做过的一个[报警指标](https://github.com/cloudnative365/PrivatePaaS/blob/master/monitoring/prometheus/prometheus-config-rules.yml)，里面的都是一些表达式。我们挑选CPU作为我们今天的例子。

``` bash
round((1 - avg(rate(node_cpu_seconds_total{mode="idle"}[15m])) by (instance)) * 100) > 80
```

这个指标里面涉及了我们几乎所有的知识点，他的主要目标是通过监控CPU空闲率来计算出现在CPU的繁忙程度，如果超过80%就需要报警了。

## 2. 查询

### 2.1. 查询普通的指标

如果我们只需要查询某一个指标的值，我们只需要把这个值放入查询框，此例中是`node_cpu_seconds_total`

![image-20200725222816264](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725222816264.png)

### 2.2. 选择（带标签的）查询

如果我们想选出某个标签等于特定值的结果，比如：`mode=“idle”`，就需要使用`node_cpu_seconds_total{mode="idle"}`

![image-20200725222844802](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725222844802.png)

### 2.3. 指定时间段

在指标的后面使用`[15m]`这种带时间参数的，就可以选出在15m之内的数值`node_cpu_seconds_total{mode="idle"}[15m]`

![image-20200725222943122](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725222943122.png)

感觉输出有点不太对劲啊，这是因为，如果要求一段时间内的值，这是需要使用一些修饰的，也就是rate和irate

### 2.4. rate和irate

rate结果如下

![image-20200725223103669](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725223103669.png)

irate结果如下

![image-20200725223125141](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725223125141.png)

irate和rate都会用于计算某个指标在一定时间间隔内的变化速率。但是它们的计算方法有所不同：irate取的是在指定时间范围内的最近两个数据点来算速率，而rate会取指定时间范围内所有数据点，算出一组速率，然后取平均值作为结果。

所以官网文档说：irate适合快速变化的计数器（counter），而rate适合缓慢变化的计数器（counter）。

根据以上算法我们也可以理解，对于快速变化的计数器，如果使用rate，因为使用了平均值，很容易把峰值削平。除非我们把时间间隔设置得足够小，就能够减弱这种效应。

#### 2.5. 求平均值的函数

使用avg()函数可以求函数值，当然，我们还可以有其他的比如min()，max()，sum()等

![image-20200725223308328](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725223308328.png)

感觉结果又不太正确了，这是因为这边结果需要加上by，这和sql中的group by的道理是一样的，avg函数需要按照某一个by的字段进行求平均数

![image-20200725223428061](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725223428061.png)

### 2.6. 求出我们要的值

系统的使用是由很多部分组成的，但是空闲的部分缺只有一个，所以我们想得到系统使用率就用1减去空闲的百分比就是我们的系统使用率了，而不用再把每个使用CPU的指标相加。

![image-20200725223746626](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725223746626.png)

### 2.7. 优化

我们发现其实我们得出的结果往往是带小数的，比如0.8032，但是在实际的使用中，如果我们把指标定为0.8就报警，如果CPU有一些抖动就会影响报警了，所以我们使用100去乘以结果，得到80.32，然后使用round()函数去四舍五入得到整数，再去和80比较，就会得到一个整数去和80比较，看上去就比较清晰。

![image-20200725223838650](/pages/keynotes/L4_architect/2_monitoring/pics/7_complex_query/image-20200725223838650.png)

## 3. grafana模板

我们通常在生产中很少会手动写模板，第一是因为每个版本（node_exporter，prometheus，grafana的版本）都可能会有一些指标的变化，会让我们曾经认为好用的模板变得一文不值。这个时候，社区的魅力就展现了出来，总有一些乐于分享的人，愿意把自己的成果分享出来，我们只要伸手就好了。但是伸手也需要一定的知识基础，我们讲过了prometheus和node_exporter之后，我们再来说说grafana。

### 3.1. 安装grafana

+ [下载地址](https://grafana.com/grafana/download)，我们可以看到他支持非常多的平台，甚至支持ARM，所以我理所当然的把grafana安装在了我的树莓派上。

+ 安装：

  ``` bash
  wget https://dl.grafana.com/oss/release/grafana-7.1.1-1.x86_64.rpm
  sudo yum install grafana-7.1.1-1.x86_64.rpm
  ```

+ 启动：

  ``` bash
  systemctl start grafana-server
  ```

+ 访问：打开浏览器，访问服务器地址的3000端口就可以看见界面了

### 3.2. grafana的一点说明

grafana是由grafana labs开发的软件，有完全免费的开源版本，还有基于saas的商业版本。grafana在云原生时代凭借出色的界面设计和非常流畅的用户体验让他在云原生的前期一路领先。

我们这次课程里面的grafana是一个比较简单的demo，主要是为了让大家对于grafana有一个直观的认识。我们后面会讲企业中，生产级别的grafana应该怎样搭建。

最后，grafana labs还有另外一款软件也是非常不错的日志收集展示工具，叫LOKI，如果我们有机会，会在日志的专题里面讲一下。

### 3.3. grafana连接prometheus