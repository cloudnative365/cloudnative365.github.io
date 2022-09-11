---
title: loki初体验
keywords: keynotes, architect, observability, log, loki, loki_basic
permalink: keynotes_L5_architect_observability_2_log_3_1_loki_basic.html
sidebar: keynotes_L5_architect_observability
typora-copy-images-to: ./pics/3_1_loki_basic
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

+ 认识loki，搭建loki服务
+ 使用promtail进行日志收集
+ 使用grafana进行日志展示

## 1. Loki

loki是grafana公司的另外一款主打产品，目前是grafana公司继prometheus，grafana，cortex之后的第四款比较知名的产品。这是一款支持垂直扩展，支持高可用，多租户日志聚合的系统。他和我们前面讲的ES就在于降低了成本，操作简单。Loki并没有对日志的内容进行索引，而是对于每个日志流进行打标签的工作。这样，对于小规模的日志存取都非常方便，也就更加适合于我们对于日志监控的要求，让产品定位在了日志监控。而ES在日志监控方面的功能虽然也非常强大，但是操作复杂度和昂贵的价格会让我们望而生畏，因此，轻量且开源的Loki让我们有了另外的选择。

但是Loki的缺点也是非常明显的，这也是大部分开源的云原生产品的共性：没有认证功能。如果想做认证，就需要通过其他的方式来实现，比如nginx，或者k8s ingress的认证，或者其他三方的工具来实现。

## 2. 架构图

一般来说，我们会通过logagent（比如promtail或者fluentd）挖掘日志，然后发送给Loki服务器，最后通过grafana进行展示。而主要的功能都集中在了loki服务器上

![modes_diagram](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/modes_of_operation.png)



### 2.1. 组件

![agdfg](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/ZmFuZ3poZW5naGVpdGk.png)

+ Distributer 分发器，负责接收来自于客户端的日志，通过批处理来构建压缩数据
+ Ingester捕获器，负责构建和刷新chunks，满足特定条件在刷新到存储中去
+ Querier查询器，根据时间和标签选择器，去索引查找并且匹配到指定的数据，同时去Ingester里面查找尚未刷新到存储中的数据

### 2.2. 逻辑架构图

Loki同样是使用标签作为索引，一切的数据都是通过筛选标签来得到结果的。实际上，我们真正需要安装的只有grafana，loki和promtail

![轻量级日志采集系统Loki+grafana搭建](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/20210329233601933.png)



+ Grafana：用于最终的展示，并且借助于官方网站上的[dashboard12559](https://grafana.com/grafana/dashboards/12559)来实现nginx日志的展示与监控

  ![asdf](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/image.jpeg)

+ Loki：我们上面介绍了Loki的主要组件，但是实际上Loki还包含了query frontend和存储相关的组件，数据的存储方式可以选择本地，也可以选择公有云存储或对象存储

+ promtail：日志的agent，一般安装在需要采集日志的主机上来挖掘日志。

## 3. 单机安装

为了实现我们上面看到的效果，我们需要借助于nginx+GeoIP。我们选用的是AWS，实例是Amazon Linux2，选用Centos或者RHEL原理基本相同，需要注意的是，nginx需要一个叫做ngx_http_geoip_module.so的模块，如果我们使用的是系统自带的epel源的话，是没办法安装这个模块的，只能用编译的方式来安装这个模块。或者我们可以直接使用nginx官方提供的源。

### 3.1. 安装Nginx和GEOIP

配置nginx源,/etc/yum.repos.d/nginx.repo，我使用的是amazon linux2，所以baseurl我修改如下

``` bash
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/7/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

安装相关包

``` bash
yum install -y nginx  # nginx主程序
yum install -y nginx-module-geoip # geoip模块
yum install -y GeoIP-data # geoip的数据文件
```

配置nginx文件，在全局配置下加载模块

``` bash
load_module /etc/nginx/modules/ngx_http_geoip_module.so;
load_module /etc/nginx/modules/ngx_stream_geoip_module.so;
```

在http段中，增加日志格式并且加载地理文件

``` bash
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  /var/log/nginx/access.log  main;

    log_format json_analytics escape=json '{'
                    '"msec": "$msec", ' # request unixtime in seconds with a milliseconds resolution
                    '"connection": "$connection", ' # connection serial number
                    '"connection_requests": "$connection_requests", ' # number of requests made in connection
                    '"pid": "$pid", ' # process pid
                    '"request_id": "$request_id", ' # the unique request id
                    '"request_length": "$request_length", ' # request length (including headers and body)
                    '"remote_addr": "$remote_addr", ' # client IP
                    '"remote_user": "$remote_user", ' # client HTTP username
                    '"remote_port": "$remote_port", ' # client port
                    '"time_local": "$time_local", '
                    '"time_iso8601": "$time_iso8601", ' # local time in the ISO 8601 standard format
                    '"request": "$request", ' # full path no arguments if the request
                    '"request_uri": "$request_uri", ' # full path and arguments if the request
                    '"args": "$args", ' # args
                    '"status": "$status", ' # response status code
                    '"body_bytes_sent": "$body_bytes_sent", ' # the number of body bytes exclude headers sent to a client
                    '"bytes_sent": "$bytes_sent", ' # the number of bytes sent to a client
                    '"http_referer": "$http_referer", ' # HTTP referer
                    '"http_user_agent": "$http_user_agent", ' # user agent
                    '"http_x_forwarded_for": "$http_x_forwarded_for", ' # http_x_forwarded_for
                    '"http_host": "$http_host", ' # the request Host: header
                    '"server_name": "$server_name", ' # the name of the vhost serving the request
                    '"request_time": "$request_time", ' # request processing time in seconds with msec resolution
                    '"upstream": "$upstream_addr", ' # upstream backend server for proxied requests
                    '"upstream_connect_time": "$upstream_connect_time", ' # upstream handshake time incl. TLS
                    '"upstream_header_time": "$upstream_header_time", ' # time spent receiving upstream headers
                    '"upstream_response_time": "$upstream_response_time", ' # time spend receiving upstream body
                    '"upstream_response_length": "$upstream_response_length", ' # upstream response length
                    '"upstream_cache_status": "$upstream_cache_status", ' # cache HIT/MISS where applicable
                    '"ssl_protocol": "$ssl_protocol", ' # TLS protocol
                    '"ssl_cipher": "$ssl_cipher", ' # TLS cipher
                    '"scheme": "$scheme", ' # http or https
                    '"request_method": "$request_method", ' # request method
                    '"server_protocol": "$server_protocol", ' # request protocol, like HTTP/1.1 or HTTP/2.0
                    '"pipe": "$pipe", ' # "p" if request was pipelined, "." otherwise
                    '"gzip_ratio": "$gzip_ratio", '
                    '"http_cf_ray": "$http_cf_ray",'
                    '"geoip_country_code": "$geoip_country_code"'
                    '}';

    access_log /var/log/nginx/json_access.log json_analytics;

    geoip_country /usr/share/GeoIP/GeoIP.dat;
    geoip_city /usr/share/GeoIP/GeoIPCity.dat;
    
    ...
```

启动nginx

``` bash
systemctl start nginx
```

### 3.2. 安装grafana

目前版本是7.5，我们选用最新的就可以

``` bash
yum install -y https://dl.grafana.com/oss/release/grafana-7.5.1-1.x86_64.rpm
```

记得安装插件

``` bash
/usr/sbin/grafana-cli plugins install grafana-worldmap-panel
```

启动即可，访问IP：3000，默认密码是admin/admin

### 3.3. 安装loki

``` bash
wget https://github.com/grafana/loki/releases/download/v2.2.0/loki-linux-amd64.zip # 下载
unzip loki-linux-amd64.zip # 解压
mv loki-linux-amd64 /usr/local/sbin/loki # 挪到$PATH下
mkdir /etc/loki # 创建配置文件目录
wget https://raw.githubusercontent.com/grafana/loki/master/cmd/loki/loki-local-config.yaml -O /etc/loki/loki-local-config.yaml # 下载样例
nohup loki -config.file=/etc/loki/loki-local-config.yaml & # 直接启动就好
```

他会监听在本地的3100端口上，我们这个时候就可以用grafana使用loki作为数据源了

### 3.4. 安装promtail

``` bash
wget https://github.com/grafana/loki/releases/download/v2.2.0/promtail-linux-amd64.zip # 下载
unzip loki-linux-amd64.zip # 解压
mv promtail-linux-amd64 /usr/local/sbin/promtail # 挪到$PATH下
mkdir /etc/promtail # 创建配置文件目录
```

官方提供了配置文件样例

``` bash
wget https://raw.githubusercontent.com/grafana/loki/master/cmd/promtail/promtail-local-config.yaml
```

但是这里我们简化一下，为了更好的出图

``` bash
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
    - job_name: nginx_location_json 
      pipeline_stages:
      - replace:
          expression: '(?:[0-9]{1,3}\.){3}([0-9]{1,3})'
          replace: '***'
      static_configs:
      - targets:
         - localhost
        labels:
         job: nginx_access_log
         host: appfelstrudel
         agent: promtail
         __path__: /var/log/nginx/json_access.log
```

启动

``` bash
nohup promtail -config.file=/etc/promtail/promtail-local-config.yaml &
```

### 3.5.  看图喽

我们这个时候就可以切换到我们的界面上看到效果了

![c8bb18c9-8aed-4992-82ba-86ab11c0dfd4](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/c8bb18c9-8aed-4992-82ba-86ab11c0dfd4.png)



## 4. 报警

### 4.1. grafana报警

我们可以在某个指标上直接报警，这个是通过grafana来实现的，但是这个报警严重依赖于grafana，功能非常受限

![123](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/5354deaf-05e6-4291-b984-6d2eb34bc162.png)

### 4.2. alertmanager报警

这个是我们比较推荐的方式

``` bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
tar xf alertmanager-0.21.0.linux-amd64.tar.gz
mv alertmanager-0.21.0.linux-amd64/alertmanager /usr/local/sbin/alertmanager
```

修改一下配置文件/etc/alertmanager/alertmanager.yml

``` bash
global:
  # global parameter
  resolve_timeout: 5m

  # smtp parameter
  smtp_smarthost: smtpdm.xxx.com:465
  smtp_from: monitor@notify.xxx.com
  smtp_auth_username: monitor@notify.xxx.com
  smtp_auth_password: XXXXXX
  smtp_require_tls: false

route:
  receiver: default-reciever
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  group_by: ["alertname"]

receivers:
- name: 'default-reciever'
  email_configs:
  - send_resolved: true
    to: 'jormun@xxx.com'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'env', 'instance', 'zone']
```

启动

``` bash
nohup alertmanager --config.file=/etc/alertmanager/alertmanager.yml &
```

配置报警规则，在loki的配置/etc/loki/loki-local-config.yaml文件中有这么一段

``` bash
ruler:
  storage:
    type: local
    local:
      directory: /tmp/loki/rules
  rule_path: /tmp/loki/rules-temp
  alertmanager_url: http://localhost:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
```

这就规定了报警规则的文件需要在/tmp/loki/rules下面，我们需要在/tmp/loki/rules下面创建一个文件夹，然后再写规则才能生效。。。囧。比如：/tmp/loki/rules/demo/example.yml

``` bash
groups:
  - name: sample_alert
    rules:
      - alert: sample_alert
        expr: 'sum by (filename) (count_over_time({filename=~".*json_access.log.*"} |~ "404" [1m]) > 0)'
        annotations:
          message: "404 alert"
        for: 1m
        labels:
          severity: critical
```

上面的报警规则是说如果日志中在一分钟内有一个404就来报警（为了好展示效果），我们可以随便访问一个不存在的地址，然后就会收到下面的报警了

![343348c8-0eba-49f4-bb66-958a8980b99d](/pages/keynotes/L5_architect_observability/2_log/pics/3_1_loki/343348c8-0eba-49f4-bb66-958a8980b99d.png)

