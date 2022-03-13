---
title: Prometheus概览
keywords: keynotes, senior, metrics, prometheus_overview
permalink: keynotes_L3_senior_5_metrics_1_prometheus_overview.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/1_prometheus_overview
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. Prometheus快速搭建

### 1.1. 下载

[下载地址](https://prometheus.io/download/)，这里面的地址都是来自于github，我们也可以去github中的[release](https://github.com/prometheus/prometheus/releases)直接下载。

我们这里使用的是2.18.1版本

``` bash
wget https://github.com/prometheus/prometheus/releases/download/v2.18.1/prometheus-2.18.1.linux-amd64.tar.gz
```

### 1.2. 安装

不管哪个版本，我都建议看[安装文档](https://prometheus.io/docs/prometheus/latest/getting_started/)来安装。这里我们为了讲课方便，我会把安装步骤写下来。

首先解压他

``` bash
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

### 1.3. 配置Prometheus监控他自己

Prometheus收集来自从targets的endpoints中的metrics。因为Prometheus同样会暴露一些关于自己的指标，他也可以抓取并且监控他自己的监控。

虽然Prometheus服务器只收集自身的数据在实践中不是很有用，但它是一个很好的开始示例。将以下基本Prometheus配置保存为名为`Prometheus.yml`

``` yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

### 1.4. 启动Prometheus

使用新创建的配置文件来启动Prometheus，进入到包含Prometheus二进制文件的目录运行

``` bash
# Start Prometheus.
# By default, Prometheus stores its database in ./data (flag --storage.tsdb.path).
./prometheus --config.file=prometheus.yml
```



## 2. 架构

### 2.1. 架构图

![Prometheus architecture](/pages/keynotes/L3_senior/5_metrics/pics/1_prometheus_overview/architecture.png)

### 2.2. 一些概念



## 3. 初步认识Prometheus

### 3.1. 使用表达式浏览

我们来看一下Prometheus收集的关于他自己的数据。使用Prometheus内建的浏览器，打开浏览器输入 http://localhost:9090/graph然后选择Console来浏览“Graph”标签。

我们可以从这个地址来采集metric [localhost:9090/metrics](http://localhost:9090/metrics), Prometheus所暴露的他自己的指标中，有一个metric叫 `prometheus_target_interval_length_seconds` (两次抓取指标的时间间隔). 我们把他输入到查询框中，然后执行Execute

![image-20200607222651752](/pages/keynotes/L3_senior/5_metrics/pics/1_prometheus_overview/image-20200607222651752.png)



他会返回一系列不同的时间（每个后面都有一个value），所有的metrics都叫 `prometheus_target_interval_length_seconds`但是标签不同。这些标签标示不同的延迟百分比和目标组的间隔时间

如果我们只对“99th percentile latencies”这一行感兴趣，我们就可以这样来取出这一行

```
prometheus_target_interval_length_seconds{quantile="0.99"}
```

如果想统计返回的时间序列的个数，就使用count函数

```
count(prometheus_target_interval_length_seconds)
```

### 3.2. 使用图形接口

使用图形接口的话，就输入http://localhost:9090/graph然后使用“Graph”标签

比如，输入下面的表达式可以显示出最近每分钟产生的chunks的数量

```
rate(prometheus_tsdb_head_chunks_created_total[1m])
```

![image-20200608103548306](/pages/keynotes/L3_senior/5_metrics/pics/1_prometheus_overview/image-20200608103548306.png)

### 3.3. 启动一些样例的target

我们可以创建一些样例的targets给Prometheus去抓取

Go客户端的库里面包含了一个样例，为三个不同延迟的服务暴露了虚构的RPC延迟

确保我们有[Go语言编译器](https://golang.org/doc/install)和[Go语言编译环境](https://golang.org/doc/code.html)（配置`GOPATH`）

从Prometheus网站下载Go客户端的库来运行这三个样例

```
# 下载客户端的库并且编译
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go get -d
go build

# 在三个端口启动这三个样例
./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

现在我们就有一些样例的数据暴露在端口上， http://localhost:8080/metrics, http://localhost:8081/metrics, 和 http://localhost:8082/metrics.

### 3.4. 配置Prometheus去监控这些样例

现在我们配置Prometheus去抓取这些新的目标。我们在这三个后端节点放在一个job中，叫做 `example-random`。我们前两个节点想象为生产的节点，第三个节点是一个金丝雀的实例。我们可以在Prometheus中添加多个端点到一个组，然后为每个组添加额外的标签。在下面的例子当中，我们会给第一组添加`group="production"`标签，给第二组添加`group="canary"`标签。

我们需要在`prometheus.yml` 文件的`scrape_configs`添加job的定义，然后重启Prometheus实例

```
scrape_configs:
  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

去浏览器的表达式中查看，确认Prometheus已经拥有这些端点暴露出来的时间序列，比如 `rpc_durations_seconds` 

![image-20200608173434649](/pages/keynotes/L3_senior/5_metrics/pics/1_prometheus_overview/image-20200608173434649.png)

### 3.5. 配置把抓取的数据聚合到新的时间序列

在我们刚才的例子成功的条件下，查询那些由上千条时间序列聚合而成的数据的时候可能很慢，他们需要一条一条的计算。为了让他更有效率，Prometheus允许我们通过规则预先设定一个表达式rate(rpc_durations_seconds_count[5m]）来创建一个新的时间段。比如，我们对于所有的实例中，平均每秒钟的RPC请求 (`rpc_durations_seconds_count`) 感兴趣，并且观察的维度（groub by）是`job`和`service`，每五分钟计算一次，我们就可以写成下面的形式

```
avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

![image-20200608173621247](/pages/keynotes/L3_senior/5_metrics/pics/1_prometheus_overview/image-20200608173621247.png)

### 3.6. 创建新的规则

然后，我们想把刚才的时间序列变成一个新的metrics，叫做`job_service:rpc_durations_seconds_count:avg_rate5m`，就创建一个规则文件，并且保存成`prometheus.rules.yml`

```
groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

为了让Prometheus识别刚才的规则，需要在`prometheus.yml`中添加一个 `rule_files` 。配置文件如下：

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  external_labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

用新的配置文件重启Prometheus并且确保使用新的metrics名字的时间序列`job_service:rpc_durations_seconds_count:avg_rate5m`可以被浏览器查询

![image-20200608175311646](/pages/keynotes/L3_senior/5_metrics/pics/1_prometheus_overview/image-20200608175311646.png)

## 4. 概念解释

### 4.1. 数据模型

Prometheus从根本上存储的所有数据都是[时间序列](http://en.wikipedia.org/wiki/Time_series): 具有时间戳的数据流只属于单个度量指标和该度量指标下的多个标签维度。除了存储时间序列数据外，Prometheus也可以利用查询表达式存储5分钟的返回结果中的时间序列数据

#### 4.1.1. metrics和labels(度量指标名称和标签)

每一个时间序列数据由metric度量指标名称和它的标签labels键值对集合唯一确定。

这个metric度量指标名称指定监控目标系统的测量特征（如：`http_requests_total`- 接收http请求的总计数）. metric度量指标命名ASCII字母、数字、下划线和冒号，他必须配正则表达式`[a-zA-Z_:][a-zA-Z0-9_:]*`。

标签开启了Prometheus的多维数据模型：对于相同的度量名称，通过不同标签列表的结合, 会形成特定的度量维度实例。(例如：所有包含度量名称为`/api/tracks`的http请求，打上`method=POST`的标签，则形成了具体的http请求)。这个查询语言在这些度量和标签列表的基础上进行过滤和聚合。改变任何度量上的任何标签值，则会形成新的时间序列图

标签label名称可以包含ASCII字母、数字和下划线。它们必须匹配正则表达式`[a-zA-Z_][a-zA-Z0-9_]*`。带有`_`下划线的标签名称被保留内部使用。

标签labels值包含任意的Unicode码。

#### 4.1.2. 有序的采样值

有序的采样值形成了实际的时间序列数据列表。每个采样值包括：

- 一个64位的浮点值
- 一个精确到毫秒级的时间戳一个样本数据集是针对一个指定的时间序列在一定时间范围的数据收集。这个时间序列是由{=, …}

+ 指定度量名称和度量指标下的相关标签值，则确定了所关心的目标数据，随着时间推移形成一个个点，在图表上实时绘制动态变化的线条’

#### 4.1.3. Notation(符号)

表示一个度量指标和一组键值对标签，需要使用以下符号：

> [metric name]{[label name]=[label value], …}

例如，度量指标名称是`api_http_requests_total`， 标签为`method="POST"`, `handler="/messages"` 的示例如下所示：

> api_http_requests_total{method=”POST”, handler=”/messages”}

这些命名和OpenTSDB使用方法是一样的

### 4.2. metrics类型

Prometheus客户库提供了四个核心的metrics类型。这四种类型目前仅在客户库和wire协议中区分。Prometheus服务还没有充分利用这些类型。不久的将来就会发生改变。

#### 4.2.1. Counter(计数器)

*counter* 是一个累计度量指标，它是一个只能递增的数值。计数器主要用于统计服务的请求数、任务完成数和错误出现的次数等等。计数器是一个递增的值。反例：统计goroutines的数量。计数器的使用方式在下面的各个客户端例子中：

客户端使用计数器的文档：

- [Go](http://godoc.org/github.com/prometheus/client_golang/prometheus#Counter)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Counter.java)
- [Python](https://github.com/prometheus/client_python#counter)
- [Ruby](https://github.com/prometheus/client_ruby#counter)

#### 4.2.2. Gauge(测量器)

*gauge*是一个度量指标，它表示一个既可以递增, 又可以递减的值。

测量器主要测量类似于温度、当前内存使用量等，也可以统计当前服务运行随时增加或者减少的Goroutines数量

客户端使用计量器的文档：

- [Go](http://godoc.org/github.com/prometheus/client_golang/prometheus#Gauge)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Gauge.java)
- [Python](https://github.com/prometheus/client_python#gauge)
- [Ruby](https://github.com/prometheus/client_ruby#gauge)

#### 4.2.3. Histogram(柱状图)

*histogram*，是柱状图，在Prometheus系统中的查询语言中，有三种作用：

1. 对每个采样点进行统计，打到各个分类值中(bucket)
2. 对每个采样点值累计和(sum)
3. 对采样点的次数累计和(count)

度量指标名称: `[basename]`的柱状图, 上面三类的作用度量指标名称

- [basename]_bucket{le=”上边界”}, 这个值为小于等于上边界的所有采样点数量
- [basename]_sum
- [basename]_count

小结：所以如果定义一个度量类型为Histogram，则Prometheus系统会自动生成三个对应的指标

使用[histogram_quantile()](https://prometheus.io/docs/querying/functions/#histogram_quantile)函数, 计算直方图或者是直方图聚合计算的分位数阈值。 一个直方图计算[Apdex值](http://en.wikipedia.org/wiki/Apdex)也是合适的, 当在buckets上操作时，记住直方图是累计的。详见[直方图和总结](https://prometheus.io/docs/practices/histograms)

客户库的直方图使用文档：

- [Go](http://godoc.org/github.com/prometheus/client_golang/prometheus#Histogram)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Histogram.java)
- [Python](https://github.com/prometheus/client_python#histogram)
- [Ruby](https://github.com/prometheus/client_ruby#histogram)

#### 4.2.4. 总结

类似*histogram*柱状图，*summary*是采样点分位图统计，(通常的使用场景：请求持续时间和响应大小)。 它也有三种作用：

1. 对于每个采样点进行统计，并形成分位图。（如：正态分布一样，统计低于60分不及格的同学比例，统计低于80分的同学比例，统计低于95分的同学比例）
2. 统计班上所有同学的总成绩(sum)
3. 统计班上同学的考试总人数(count)

带有度量指标的`[basename]`的`summary` 在抓取时间序列数据展示。

- 观察时间的φ-quantiles (0 ≤ φ ≤ 1), 显示为`[basename]{分位数="[φ]"}`
- `[basename]_sum`， 是指所有观察值的总和
- `[basename]_count`, 是指已观察到的事件计数值

详见[histogram和summaries](https://prometheus.io/docs/practices/histograms)

有关`summaries`的客户端使用文档：

- [Go](http://godoc.org/github.com/prometheus/client_golang/prometheus#Summary)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Summary.java)
- [Python](https://github.com/prometheus/client_python#summary)
- [Ruby](https://github.com/prometheus/client_ruby#summary)

### 4.3. Jobs和Instances(任务和实例)

就Prometheus而言，pull拉取采样点的端点服务称之为**instance**。多个这样pull拉取采样点的instance, 则构成了一个**job**

例如, 一个被称作**api-server**的任务有四个相同的实例。

- job: `api-server`
  - instance 1：`1.2.3.4:5670`
  - instance 2：`1.2.3.4:5671`
  - instance 3：`5.6.7.8:5670`
  - instance 4：`5.6.7.8:5671`

#### 4.3.1. 自动化生成的标签和时间序列

当Prometheus拉取一个目标, 会自动地把两个标签添加到度量名称的标签列表中，分别是：

- **job**: 目标所属的配置任务名称**api-server**。
- **instance**: 采样点所在服务: `host:port`

如果以上两个标签二者之一存在于采样点中，这个取决于`honor_labels`配置选项。详见[文档](https://prometheus.io/docs/operating/configuration/#)

对于每个采样点所在服务instance，Prometheus都会存储以下的度量指标采样点：

- `up{job="[job-name]", instance="instance-id"}`: up值=1，表示采样点所在服务健康; 否则，网络不通, 或者服务挂掉了
- `scrape_duration_seconds{job="[job-name]", instance="[instance-id]"}`: 尝试获取目前采样点的时间开销
- `scrape_samples_post_metric_relabeling{job="", instance=""}`: 表示度量指标的标签变化后，标签没有变化的度量指标数量。
- `scrape_samples_scraped{job="", instance=""}`: 这个采样点目标暴露的样本点数量

`up`度量指标对服务健康的监控是非常有用的。