---
title: node_exporter
keywords: keynotes, senior, metrics, node_exporter
permalink: keynotes_L3_senior_5_metrics_9_node_exporter.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/9_node_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

![See the source image](/pages/keynotes/L3_senior/5_metrics/pics/9_node_exporter/OIP.UqkmSzvXxcoGowCmvlBxmwHaE7)

node_exporter是我们最常用的exporter之一，我们可以把他认为是一个agent，需要被安装在操作系统之上，然后才能采集到系统的数据。采集Linux类的exporter被称为node_exporter，如果我们使用sidecar的方式来运行的话，他就可以采集pod的信息。

我们在前面的指标采集一章中，已经使用node_exporter做了一个demo，他的采集的指标比zabbix多了好多。其实，这只是他的冰山一角，因为我们配置exporter的时候，使用的是默认启动的方式，他还有很多开关，需要添加参数才能生效。同时，他还可以指定那类指标不收集，以此来节省我们的资源消耗。

## 2. 详解node_exporter

node_exporter是prometheus官方提供的agent，项目被托管在prometheus的账号之下。我们也可以通过官方提供的链接[下载](https://prometheus.io/download/)最新的版本。在官方的[github](https://github.com/prometheus/node_exporter)里面已经提供了非常详细使用说明，我们这里带大家梳理一下

### 2.1. 默认启用的参数

[官方地址](https://github.com/prometheus/node_exporter#enabled-by-default)，这些参数就是我们前面做实验的时候默认收集的指标

| Name         | Description                                                  | OS                                                           |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| arp          | Exposes ARP statistics from `/proc/net/arp`.                 | Linux                                                        |
| bcache       | Exposes bcache statistics from `/sys/fs/bcache/`.            | Linux                                                        |
| bonding      | Exposes the number of configured and active slaves of Linux bonding interfaces. | Linux                                                        |
| boottime     | Exposes system boot time derived from the `kern.boottime` sysctl. | Darwin, Dragonfly, FreeBSD, NetBSD, OpenBSD, Solaris         |
| conntrack    | Shows conntrack statistics (does nothing if no `/proc/sys/net/netfilter/` present). | Linux                                                        |
| cpu          | Exposes CPU statistics                                       | Darwin, Dragonfly, FreeBSD, Linux, Solaris                   |
| cpufreq      | Exposes CPU frequency statistics                             | Linux, Solaris                                               |
| diskstats    | Exposes disk I/O statistics.                                 | Darwin, Linux, OpenBSD                                       |
| edac         | Exposes error detection and correction statistics.           | Linux                                                        |
| entropy      | Exposes available entropy.                                   | Linux                                                        |
| exec         | Exposes execution statistics.                                | Dragonfly, FreeBSD                                           |
| filefd       | Exposes file descriptor statistics from `/proc/sys/fs/file-nr`. | Linux                                                        |
| filesystem   | Exposes filesystem statistics, such as disk space used.      | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD                   |
| hwmon        | Expose hardware monitoring and sensor data from `/sys/class/hwmon/`. | Linux                                                        |
| infiniband   | Exposes network statistics specific to InfiniBand and Intel OmniPath configurations. | Linux                                                        |
| ipvs         | Exposes IPVS status from `/proc/net/ip_vs` and stats from `/proc/net/ip_vs_stats`. | Linux                                                        |
| loadavg      | Exposes load average.                                        | Darwin, Dragonfly, FreeBSD, Linux, NetBSD, OpenBSD, Solaris  |
| mdadm        | Exposes statistics about devices in `/proc/mdstat` (does nothing if no `/proc/mdstat` present). | Linux                                                        |
| meminfo      | Exposes memory statistics.                                   | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD                   |
| netclass     | Exposes network interface info from `/sys/class/net/`        | Linux                                                        |
| netdev       | Exposes network interface statistics such as bytes transferred. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD                   |
| netstat      | Exposes network statistics from `/proc/net/netstat`. This is the same information as `netstat -s`. | Linux                                                        |
| nfs          | Exposes NFS client statistics from `/proc/net/rpc/nfs`. This is the same information as `nfsstat -c`. | Linux                                                        |
| nfsd         | Exposes NFS kernel server statistics from `/proc/net/rpc/nfsd`. This is the same information as `nfsstat -s`. | Linux                                                        |
| pressure     | Exposes pressure stall statistics from `/proc/pressure/`.    | Linux (kernel 4.20+ and/or [CONFIG_PSI](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/accounting/psi.txt)) |
| rapl         | Exposes various statistics from `/sys/class/powercap`.       | Linux                                                        |
| schedstat    | Exposes task scheduler statistics from `/proc/schedstat`.    | Linux                                                        |
| sockstat     | Exposes various statistics from `/proc/net/sockstat`.        | Linux                                                        |
| softnet      | Exposes statistics from `/proc/net/softnet_stat`.            | Linux                                                        |
| stat         | Exposes various statistics from `/proc/stat`. This includes boot time, forks and interrupts. | Linux                                                        |
| textfile     | Exposes statistics read from local disk. The `--collector.textfile.directory` flag must be set. | *any*                                                        |
| thermal_zone | Exposes thermal zone & cooling device statistics from `/sys/class/thermal`. | Linux                                                        |
| time         | Exposes the current system time.                             | *any*                                                        |
| timex        | Exposes selected adjtimex(2) system call stats.              | Linux                                                        |
| udp_queues   | Exposes UDP total lengths of the rx_queue and tx_queue from `/proc/net/udp` and `/proc/net/udp6`. | Linux                                                        |
| uname        | Exposes system information as provided by the uname system call. | Darwin, FreeBSD, Linux, OpenBSD                              |
| vmstat       | Exposes statistics from `/proc/vmstat`.                      | Linux                                                        |
| xfs          | Exposes XFS runtime statistics.                              | Linux (kernel 4.4+)                                          |
| zfs          | Exposes [ZFS](http://open-zfs.org/) performance statistics.  | [Linux](http://zfsonlinux.org/), Solaris                     |

可以看到，每个指标都有他对应的操作系统，并不是每种系统都适用的。如果不想收集某个类型的指标，就使用`--no-collector.<name>`参数，比如

``` bash
./node_exporter --no-collector.time
```



### 2.2. 默认不启用的参数

默认不启用的参数需要通过`--collector.<name>`参数来启用，官方提供的不启用的参数如下

| Name         | Description                                                  | OS                 |
| ------------ | ------------------------------------------------------------ | ------------------ |
| buddyinfo    | Exposes statistics of memory fragments as reported by /proc/buddyinfo. | Linux              |
| devstat      | Exposes device statistics                                    | Dragonfly, FreeBSD |
| drbd         | Exposes Distributed Replicated Block Device statistics (to version 8.4) | Linux              |
| interrupts   | Exposes detailed interrupts statistics.                      | Linux, OpenBSD     |
| ksmd         | Exposes kernel and system statistics from `/sys/kernel/mm/ksm`. | Linux              |
| logind       | Exposes session counts from [logind](http://www.freedesktop.org/wiki/Software/systemd/logind/). | Linux              |
| meminfo_numa | Exposes memory statistics from `/proc/meminfo_numa`.         | Linux              |
| mountstats   | Exposes filesystem statistics from `/proc/self/mountstats`. Exposes detailed NFS client statistics. | Linux              |
| ntp          | Exposes local NTP daemon health to check [time](https://github.com/prometheus/node_exporter/blob/master/docs/TIME.md) | *any*              |
| processes    | Exposes aggregate process statistics from `/proc`.           | Linux              |
| qdisc        | Exposes [queuing discipline](https://en.wikipedia.org/wiki/Network_scheduler#Linux_kernel) statistics | Linux              |
| runit        | Exposes service status from [runit](http://smarden.org/runit/). | *any*              |
| supervisord  | Exposes service status from [supervisord](http://supervisord.org/). | *any*              |
| systemd      | Exposes service and system status from [systemd](http://www.freedesktop.org/wiki/Software/systemd/). | Linux              |
| tcpstat      | Exposes TCP connection status information from `/proc/net/tcp` and `/proc/net/tcp6`. (Warning: the current version has potential performance issues in high load situations.) | Linux              |
| wifi         | Exposes WiFi device and station statistics.                  | Linux              |
| perf         | Exposes perf based metrics (Warning: Metrics are dependent on kernel configuration and settings). | Linux              |

如果大家使用的是Centos系，或者ubuntu系的，我们建议大家打开ntp，mountstats，systemd，ntp，tcpstat这几个选项

### 2.3. 文本收集器

使用`--collector.textfile.directory`选项可以把文本中的内容也用metrics的方式收集起来，但是用的不多，大家了解就好

### 2.4. 在prometheus中过滤collector

我们还可以在prometheus中过滤要采集的选项，在exporter的配置文件中使用下面的配置

``` yaml
  params:
    collect[]:
      - foo
      - bar
```

这个功能可以减少prometheus的压力，但是并不会减少node_exporter采集到的数据。有点鸡肋。。。既然不要这个，就不要采集了嘛。。。

### 2.5. TLS的配置

我们默认暴露的都是http的请求的，也没有验证，也就是明文传输，很容易存在安全隐患，如果需要对数据加密，我们就要使用参数`./node_exporter --web.config=web-config.yml`。这个文件中含有TLS或者basic auth等选项的配置。我们先来配置TLS

+ 生成证书

  ``` bash
  # mkdir -p prometheus-tls
  # cd prometheus-tls
  # prometheus-tls openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=CN/ST=Beijing/L=Beijing/O=jormun.com/CN=localhost"
  Generating a RSA private key
  ...........................................+++++
  ..+++++
  writing new private key to 'node_exporter.key'
  -----
  # prometheus-tls ls
  node_exporter.crt  node_exporter.key
  ```

+ 配置web-config.yml文件

  ``` yml
  tls_server_config:
    cert_file: node_exporter.crt
    key_file: node_exporter.key
  ```

  

### 2.6. 配置http的basic认证

同样是修改`--web.config=web-config.yml`中的web-config.yml文件中的参数。我们需要使用一些工具来生成密码。比如，我们的用户名为prometheus，密码为node_exporter。

+ 注意：这个功能是在node_exporter 1.0.0之后才有的功能，老版本想要认证，就需要借助反向代理，比如nginx上的认证

我们先配置node_exporter，这里面的密码是使用hash来加密的，我们常用的工具是htpasswd

``` bash
htpasswd -nBC 12 '' | tr -d ':\n'
New password: 
Re-type new password: 
$2y$12$YND5ZlkC/pbUh4pvKnE1wehfpnha2DRdR.GrmRSQAcLff1.gOmh0.
```

使用不交互的方式，生成文件

``` bash
htpasswd -bcBC 12 /application/prometheus/conf/htpasswd prometheus node_exporter
```

在原有文件中添加一个用户

``` bash
htpasswd -bBC 12 /application/prometheus/conf/htpasswd prometheus node_exporter
```

不更新密码文件，只在屏幕上输出用户名和经过加密后的密码

``` bash
htpasswd -nbBC 12 prometheus node_exporter
```

修改web-config.yml的配置如下

``` bash
basic_auth_users:
  prometheus: $2y$12$c97Lg30Imrl6hXZJ8h9IC.KQ0YhL8rQtHCuKlfrgSYv6RmmUbBlXO
```

#### 2.6.1. 配置prometheus使用scrapy

修改prometheus的配置如下

``` bash
global:
  scrape_interval:     15s 
  evaluation_interval: 15s 

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    scheme: http
    basic_auth:
      username: prometheus
      password: node_exporter
    static_configs:
    - targets: ['localhost:9100']
```



#### 2.6.2. 在consul中注册node_exporter

如果要把node_exporter注册在consul中，从而给prometheus作为数据源的话，我们在前面介绍过了。在添加认证之后的node_exporter则需要在注册服务的同时，在check选项中，加入header字段。但是，header字段是通过username:password的方式同时传递给header的，因此，我们需要使用这个方式，把prometheus:node_exporter同时做base64处理。

``` bash
perl -e 'use MIME::Base64; print encode_base64("prometheus:node_exporter")'
cHJvbWV0aGV1czpub2RlX2V4cG9ydGVy
```

然后在注册的时候，json文件如下

``` bash
{
  "ID": "node-exporter-10-114-2-73",
  "Name": "node-exporter-10-114-2-73",
  "Tags": [
    "node-exporter",
    "nginx",
    "keepalived"
  ],
  "Address": "10.114.2.73",
  "Port": 9100,
  "Meta": {
    "team": "cloud",
    "project": "monitoring"
  },
  "EnableTagOverride": false,
  "Check": {
    "HTTP": "http://10.114.2.73:9100/metrics",
    "header": {"Authorization":["Basic cHJvbWV0aGV1czpub2RlX2V4cG9ydGVy"]},
    "Interval": "10s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}
```



### 2.6. systemd例子

为了便于管理，我们使用systemd来管理这个服务

``` bash
cat > /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target
[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/node_exporter \
          --web.config=web-config.yml \
          --collector.ntp \
          --collector.mountstats \
          --collector.systemd \
          --collector.tcpstat
ExecReload=/bin/kill -HUP $MAINPID
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
EOF
```

