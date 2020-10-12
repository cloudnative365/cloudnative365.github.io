---
title: systemd
keywords: keynotes, basic, linux, systemd
permalink: keynotes_L1_basic_3_linux_1_systemd.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/1_systemd
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

Systemd是一个系统管理守护进程、工具和库的集合，用于取代System V初始进程。Systemd的功能是用于集中管理和配置类UNIX系统。他使用systemctl作为管理命令。我们可以使用man systemd来查看详细的说明。

## 2. 服务

每种服务都对应一个`.service`文件，对于每种linux发行版，有可能service脚本文件的存储位置不一样。如果我们不熟悉的话，可以使用`systemctl status xxx`来查看状态信息，里面会显示脚本文件的位置。一般来说，会在`/usr/lib/systemd/systemd`中，`/lib/systemd/system/`中或者`/etc/systemd/system/`中。对于运行在系统上的进程，我们通常会给他配置一个`.service`文件，而一个常见的脚本文件如下

``` bash
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
```

再详细的可以看[中文版](http://www.jinbuguo.com/systemd/systemd.service.html)的

## 3. 管理服务

``` bash
# systemctl start/stop/enable/status/restart xxx
```

## 4. journald

systemd的另外一个功能就是日志管理，他可以根据不同service产生的日志进行管理，使用journalctl命令来管理。

``` bash
# 根据时间选择
journalctl -b -1
journalctl -b --since=”2014-10-24 16:38”
# 根据服务选择
journalctl -u gdm.service
# 根据PID选择
journalctl _PID=890
# 根据执行文件选择
journalctl /usr/bin/pulseaudio

```

