---
title: node_exporter
keywords: keynotes, architect, monitoring, thanos
permalink: keynotes_L4_architect_2_monitoring_13_node_exporter.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/13_node_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

![See the source image](/pages/keynotes/L4_architect/2_monitoring/pics/13_node_exporter/OIP.UqkmSzvXxcoGowCmvlBxmwHaE7)

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

我们默认暴露的都是http的请求的，也没有验证，也就是明文传输，很容易存在安全隐患，如果需要对数据加密，我们就要使用参数`./node_exporter --web.config=web-config.yml`，这个文件的内容如下

``` bash
tls_server_config:
  # Certificate and key files for server to use to authenticate to client.
  cert_file: <filename>
  key_file: <filename>

  # Server policy for client authentication. Maps to ClientAuth Policies.
  # For more detail on clientAuth options: [ClientAuthType](https://golang.org/pkg/crypto/tls/#ClientAuthType)
  [ client_auth_type: <string> | default = "NoClientCert" ]

  # CA certificate for client certificate authentication to the server.
  [ client_ca_file: <filename> ]

  # Minimum TLS version that is acceptable.
  [ min_version: <string> | default = "TLS12" ]

  # Maximum TLS version that is acceptable.
  [ max_version: <string> | default = "TLS13" ]

  # List of supported cipher suites for TLS versions up to TLS 1.2. If empty,
  # Go default cipher suites are used. Available cipher suites are documented
  # in the go documentation:
  # https://golang.org/pkg/crypto/tls/#pkg-constants
  [ cipher_suites:
    [ - <string> ] ]

  # prefer_server_cipher_suites controls whether the server selects the
  # client's most preferred ciphersuite, or the server's most preferred
  # ciphersuite. If true then the server's preference, as expressed in
  # the order of elements in cipher_suites, is used.
  [ prefer_server_cipher_suites: <bool> | default = true ]

  # Elliptic curves that will be used in an ECDHE handshake, in preference
  # order. Available curves are documented in the go documentation:
  # https://golang.org/pkg/crypto/tls/#CurveID
  [ curve_preferences:
    [ - <string> ] ]

http_server_config:
  # Enable HTTP/2 support. Note that HTTP/2 is only supported with TLS.
  # This can not be changed on the fly.
  [ http2: <bool> | default = true ]

# Usernames and hashed passwords that have full access to the web
# server via basic authentication. If empty, no basic authentication is
# required. Passwords are hashed with bcrypt.
basic_auth_users:
  [ <string>: <secret> ... ]
```

这里面的密码是使用hash来加密的，我们常用的工具是htpasswd，比如，我们要加密密码Passw0rd

``` bash
htpasswd -nBC 10 "" | tr -d ':\n'
New password: 
Re-type new password: 
$2y$10$6oDcZssS/R6yPy8ixVx7ue8LiPX7CHpNtXxdVGlYkNgzW3CT48TfC%
```

使用不交互的方式，生成文件

``` bash
htpasswd -bc /application/prometheus/conf/htpasswd admin Passw0rd
```

在原有文件中添加一个用户

``` bash
htpasswd -b /application/prometheus/conf/htpasswd admin Passw0rd
```

不更新密码文件，只在屏幕上输出用户名和经过加密后的密码

``` bash
htpasswd -nb admin Passw0rd
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
          --collector.ntp \
          --collector.tcpstat
ExecReload=/bin/kill -HUP $MAINPID
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target
EOF
```

