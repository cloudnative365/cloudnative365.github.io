---
title: 查询
keywords: keynotes, L4_architect, 1_solutions_design, 2_monitoring, 4_querying
permalink: keynotes_L4_architect_2_monitoring_4_querying.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/4_querying
typora-root-url: ../../../../../cloudnative365.github.io

---

## 简介

Prometheus提供函数式的表达式语言PromQL(Prometheus Query Language) ，可以使用户实时地查找和聚合时间序列数据。表达式计算结果可以在图表中展示，也可以在Prometheus表达式浏览器中以表格形式展示，或者作为数据源, 以HTTP API的方式提供给外部系统使用。

## 1. HTTP API

在prometheus服务器上，我们可以通过访问/api/v1来查询prometheus状态，也就是http://localhost:9090/api/v1

``` bash
curl http://localhost:9090/api/v1
<a href="/api/v1/">Moved Permanently</a>
```

不过我们这样是没办法查询到数据的，毕竟这是接口的地址，但是出现这个输出就说明接口是可以访问的

### 1.1. 接口的格式

API的返回值是JSON格式的。每个请求成功的返回值都是以`2xx`开头的编码。

到达API处理的无效请求，返回一个JSON错误对象，并返回下面的错误码：

- `400 Bad Request`。当参数错误或者丢失时。
- `422 Unprocessable Entity`。当一个表达式不能被执行时。
- `503 Service Unavailable`。当查询超时或者中断时。

### 1.2. 表达式查询

查询语言表达式可以在**瞬时向量**或者**范围向量**中执行。

+ Instant queries(即时查询)：

  > GET /api/v1/query
  >
  > POST /api/v1/query

  URL查询参数：

  - `query=<string>`: Prometheus表达式要查询的字符串。
  - `time=<rfc3339 | unix_timestamp>`: 执行这个查询的时间戳，可选项，缺省则表示当前服务器时间
  - `timeout=<duration>`: 执行的超时时间设置，默认由`-query.timeout`参数设置

  查询结果如下

  ``` bash
  {
   "resultType": "matrix" | "vector" | "scalar" | "string",
   "result": <value>
  }
  ```

  比如：下面例子执行的是在时刻是`2015-07-01T20:10:51.781Z`的`up`表达式：

  ``` json
  $ curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
  {
   "status": "success",
   "data":{
      "resultType": "vector",
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

+ 范围查询

  > GET /api/v1/query_range
  >
  > POST /api/v1/query_range

  URL查询参数

  - `query=<string>`: Prometheus表达式查询字符串。
  - `start=<rfc3339 | unix_timestamp>`: 开始时间戳。
  - `end=<rfc3339 | unix_timestamp>`: 结束时间戳。
  - `step=<duration | float>`: 查询时间步长，范围时间内每step秒执行一次。
  - `timeout=<duration>`: 执行超时时间设置，默认由`-query.timeout`标志设置

  下面查询结果格式的`data`部分：

  ``` json
  {
      "resultType": "matrix",
      "result": <value>
  }
  ```

  下面例子评估的查询条件`up`，且30s范围的查询，步长是15s。

  ``` json
  $ curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
  {
     "status" : "success",
     "data" : {
        "resultType" : "matrix",
        "result" : [
           {
              "metric" : {
                 "__name__" : "up",
                 "job" : "prometheus",
                 "instance" : "localhost:9090"
              },
              "values" : [
                 [ 1435781430.781, "1" ],
                 [ 1435781445.781, "1" ],
                 [ 1435781460.781, "1" ]
              ]
           },
           {
              "metric" : {
                 "__name__" : "up",
                 "job" : "node",
                 "instance" : "localhost:9091"
              },
              "values" : [
             [ 1435781430.781, "0" ],
                 [ 1435781445.781, "0" ],
                 [ 1435781460.781, "1" ]
              ]
           }
        ]
     }
  }
  ```

### 1.3. 查询元数据

+ 通过标签匹配器找到度量指标列表

  下面例子返回了度量指标列表 且不返回时间序列数据值。

  > GET /api/v1/series

  URL查询参数：

  - `match[]=<series_selector>`: 选择器是series_selector。这个参数个数必须大于等于1.
  - `start=<rfc3339 | unix_timestamp>`: 开始时间戳。
  - `end=<rfc3339 | unix_timestamp>`: 结束时间戳。

  返回结果的`data`部分，是由key-value键值对的对象列表组成的。

  下面这个例子返回时间序列数据, 选择器是`up`或者`process_start_time_seconds{job="prometheus"}`

  ``` bash
  $ curl -g 'http://localhost:9090/api/v1/series?match[]=up&match[]=process_start_time_seconds{job="prometheus"}'
  {
     "status" : "success",
     "data" : [
        {
           "__name__" : "up",
           "job" : "prometheus",
           "instance" : "localhost:9090"
        },
        {
           "__name__" : "up",
           "job" : "node",
           "instance" : "localhost:9091"
        },
        {
           "__name__" : "process_start_time_seconds",
           "job" : "prometheus",
           "instance" : "localhost:9090"
        }
     ]
  }
  ```

+ 查询标签值

  下面这个例子，返回了带有指定标签的标签值列表

  > GET /api/v1/label//values

  这个返回JSON结果的`data`部分是带有label_name=job的值列表：

  ``` bash
  $ curl http://localhost:9090/api/v1/label/job/values
  {
     "status" : "success",
     "data" : [
        "node",
        "prometheus"
     ]
  }
  ```

+ 删除时间序列

  下面的例子，是从Prometheus服务中删除匹配的度量指标和标签列表：

  > DELETE /api/v1/series

  URL查询参数

  + `match[]=<series_selector>`: 删除符合series_selector匹配器的时间序列数据。参数个数必须大于等于1.

  返回JSON数据中的`data`部分有以下的格式

  ``` json
   {
      "numDeleted": <number of deleted series>
   }
  ```

  下面的例子删除符合度量指标名称是`up`或者时间序为`process_start_time_seconds{job="prometheus"}`：

  ``` bash
  $ curl -XDELETE -g 'http://localhost:9090/api/v1/series?match[]=up&match[]=process_start_time_seconds{job="prometheus"}'
  {
     "status" : "success",
     "data" : {
        "numDeleted" : 3
     }
  }
  ```

  

## 2. 表达式语言数据类型

在Prometheus的表达式语言PromQL中，所有的表达式或者子表达式都可以归为四种类型：

+ `string` ：字符串类型
+ `scalar` ：标量（浮点值）

- `instant vector` ：瞬时向量，它是指在同一时刻，抓取的所有metrics指标数据。这些metrics指标数据的key都是相同的，也就是时间戳是相同的。
- `range vector`：范围向量，它是指在任何一个时间范围之内，抓取的所有metrics指标数据。

### 2.1. 字符串

字符串需要用单引号`'`、双引号`"`或者反引号表示

PromQL遵循着与Go语言相同的转义规则。使用单引号，双引号中，反斜杠`\`作为转义字符，它后面还可以跟着`a, b, f, n, r, t, v`或者`\`。 也可以使用八进制(\nnn)或者十六进制(\xnn, \unnnn和\Unnnnnnnn)提供特定字符。

在反引号内不处理转义字符。与Go不同，PromQL不会丢弃反引号中的换行符。例如：

```
"this is a string"
'these are unescaped: \n \\ \t'
`these are not unescaped: \n ' " \t`
```

![image-20200612214927733](/pages/keynotes/L4_architect/2_monitoring/pics/4_querying/image-20200612214927733.png)

### 2.2. 标量

标量浮点值可以直接写成这样的形式 `[-](digits)[.(digits)]`

```
-2.43
```

![image-20200612215918944](/pages/keynotes/L4_architect/2_monitoring/pics/4_querying/image-20200612215918944.png)

### 2.3. 瞬时向量选择器

瞬时向量选择器可以对一组时间序列数据进行筛选，并给出结果中的每个结果键值对（时间戳-样本值）。最简单的形式是，只有一个metric名称被指定。在一个瞬时向量中这个结果包含有这个metrics指标名称的所有样本数据的key-value。

下面这个例子选择所有时间序列度量名称为`http_requests_total`的样本数据：

> http_requests_total

通过在度量指标后面增加{}一组标签可以进一步地过滤这些时间序列数据。

下面这个例子选择了度量指标名称为`http_requests_total`，且一组标签为`job=prometheus`, `group=canary`:

> http_requests_total{job=”prometheus”,group=”canary”}

可以采用不匹配的标签值也是可以的，或者用正则表达式不匹配标签。标签匹配操作如下所示：

- `=`: 精确地匹配标签给定的值
- `!=`: 不等于给定的标签值
- `=~`: 正则表达匹配给定的标签值
- `!=`: 给定的标签值不符合正则表达式

例如：度量指标名称为`http_requests_total`，正则表达式匹配标签`environment`为`staging, testing, development`的值，且http请求方法不等于`GET`。

> http_requests_total{environment=~”staging|testing|development”, method!=”GET”}

匹配空标签值的标签匹配器也可以选择没有设置任何标签的所有时间序列数据。正则表达式完全匹配。

向量选择器必须指定一个度量指标名称或者至少不能为空字符串的标签值。以下表达式是非法的:

> {job=~”.*”} #Bad!

上面这个例子既没有度量指标名称，标签选择器也可以正则匹配空标签值，所以不符合向量选择器的条件

相反地，下面这些表达式是有效的，第一个一定有一个字符。第二个有一个有用的标签method

> {job=~”.+”} # Good!{job=~”.*”, method=”get”} # Good!

标签匹配器能够被应用到度量指标名称，使用`__name__`标签筛选度量指标名称。例如：表达式`http_requests_total`等价于`{__name__="http_requests_total"}`。 其他的匹配器，如：`= ( !=, =~, !~)`都可以使用。下面的表达式选择了度量指标名称以`job:`开头的时间序列数据：

> {**name**=~”^job:.*”} #

### 2.4. 范围向量选择器

范围向量类似于瞬时向量, 所不同在于，它们从当前实例来选择样本范围区间。在语法上，时间长度被追加在向量选择器尾部的方括号[]之中，用以指定对于每个样本范围区间中的每个元素应该抓取的时间范围样本区间。

时间长度有一个数值决定，后面可以跟下面的单位：

- `s` - seconds
- `m` - minutes
- `h` - hours
- `d` - days
- `w` - weeks
- `y` - years

在下面这个例子中, 选择过去5分钟内，度量指标名称为`http_requests_total`， 标签为`job="prometheus"`的时间序列数据:

> http_requests_total{job=”prometheus”}[5m]

### 2.5. 偏移修饰符

这个`offset`偏移修饰符允许在查询中改变单个瞬时向量和范围向量中的时间偏移

例如，下面的表达式返回相对于当前时间的前5分钟时的时刻, 度量指标名称为`http_requests_total`的时间序列数据：

> http_requests_total offset 5m

注意：`offset`偏移修饰符必须直接跟在选择器后面，例如：

> sum(http_requests_total{method=”GET”} offset 5m) // GOOD.

然而，下面这种情况是不正确的

> sum(http_requests_total{method=”GET”}) offset 5m // INVALID.

offset偏移修饰符在范围向量上和瞬时向量用法一样的。下面这个返回了相对于当前时间的前一周时，过去5分钟的度量指标名称为`http_requests_total`的速率：

> rate(http_requests_total[5m] offset 1w)

### 2.6. 子查询

子查询可以在一个给定的范围和结果内进行查询。子查询的结果是一个范围向量

> <instant_query> '[' <range> ':' [<resolution>] ']' [ offset <duration> ]

### 2.7. 注释

PromQL支持行首以`#`开头的行为注释行

```
    # This is a comment
```

### 2.8. 运算符和函数

这两个也是查询中需要用到的，内容我们会在后面详细说