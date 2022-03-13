---
title: Prometheus进阶
keywords: keynotes, senior, metrics, prometheus_advanced
permalink: keynotes_L3_senior_5_metrics_2_prometheus_advanced.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/2_prometheus_advanced
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. API

prometheus提供了RESTFUL风格的API，我们可以通过API对集群进行增删改查。实际上，大部分和prometheus的交互都是通过API来进行的。

### 1.1.  查询

我们先来看一个查询的例子

``` bash
curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
{
   "status" : "success",
   "data" : {
      "resultType" : "vector",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "value": [ 1435781451.781, "1" ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9100"
            },
            "value" : [ 1435781451.781, "0" ]
         }
      ]
   }
}
```

先来看输入：

+ curl：没有加任何的参数就是使用get方式，当然，我们也可以使用post方式
+ /api/v1/query是我们的API的地址，表示我们和这个API来进行交互
+ ?query=up：这是我们要查询的metric指标名字
+ &time=2015-07-01T20:10:51.781Z：这个位置指定的是要查询的时间点

再来看输出：

+ status：显示的是查询结果
+ data是我们要的数据
+ resultType：就是我们前面说过的数据类型
+ result的类型是一个array，数组中的每一个元素都是一个符合条件的查询结果

当然，我们做展示的时候肯定不会给用户看这些枯燥的数据，我们需要一个标准的软件可以通过读取这些数据来生成漂亮的视图，比如grafana。最后，常见的查询方式请看[官方文档](https://prometheus.io/docs/prometheus/latest/querying/api/)

### 1.2. 管理

和其他的数据库一样，prometheus中大部分的配置是可以进行动态修改的，比如我们的target，我们就可以通过修改配置文件的方式来修改配置，而对于prometheus只需要reload，而不是restart。[官方文档](https://prometheus.io/docs/prometheus/latest/management_api/)

比如：查看健康状态healthy

``` bash
GET /-/healthy
```

查看是否正常启动readiness

``` bash
GET /-/ready
```

重新加载配置，必须启用`--web.enable-lifecycle`选项

``` bash
PUT /-/reload
POST /-/reload
```

退出

``` bash
PUT /-/quit
POST /-/quit
```

## 2. 存储

### 2.1. 本地存储

当我们把数据存储在本地的时候，prometheus会把采集的数据每两个小时的数据组合成一个组，每两小时的数据会被打成一个chunck，也就是一个文件夹，而不足两小时的数据就会放在wal里面

``` bash
./data
├── 01BKGV7JBM69T2G1BGBGM6KB12
│   └── meta.json
├── 01BKGTZQ1SYQJTR4PB43C8PD98
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
├── 01BKGTZQ1HHWHV8FBJXW1Y3W0K
│   └── meta.json
├── 01BKGV7JC0RY8A6MACW02A2PJD
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
├── chunks_head
│   └── 000001
└── wal
    ├── 000000002
    └── checkpoint.00000001
        └── 00000000
```

本地存储的局限之处在于他不能做集群或者做复制，如果实例挂了，在实例停止到实例恢复工作之前的数据就丢失了，最明显的就是在画图软件的图上会显示一个断点。最坏的情况，如果节点挂了，并且无法恢复，那么之前的数据也就都丢失了。这个时候我们就需要其他的方式来保证数据的可靠性，比如数据快照，RAID或者存储级别的高可用。

但是我们最常用的方式还是通过远程读写来把数据存放在其他具有高可用特性的媒介当中，比如我们可以通过远程读写。

### 2.2. 集成远程存储

prometheus本身是支持远程读写功能的，叫remote read和remote write。

![Remote read and write architecture](/pages/keynotes/L3_senior/5_metrics/pics/2_prometheus_advanced/remote_integrations.png)

[官方文档](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)中提供了一系列可以远程读写的数据源

![image-20220309232257114](/pages/keynotes/L3_senior/5_metrics/pics/2_prometheus_advanced/image-20220309232257114.png)

但是要注意，并不是每种方案都经过严格的验证。特别是开源的软件，大家在选择的时候要特别小心不要被坑住，在正式投入生产之前一定要做充分的验证。而且，并不是每种存储都支持直接写入。我们在架构师的课程里面会挑出几种来做几套生产级别的解决方案。