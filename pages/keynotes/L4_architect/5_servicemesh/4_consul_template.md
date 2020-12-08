---
title: consul template
keywords: keynotes, architect, servicemesh, consul_template
permalink: keynotes_L4_architect_5_servicemesh_1_consul_template.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_consul_template
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

使用consul template实现prometheus+consul服务自动注册

## 1. 概述

consul template是官方推出的一款用于制作consul service模板的工具，有点类似于python的jinja2框架，使用一些变量来固定我们注册服务的格式，从而简化我们注册服务的操作，固定了向API请求时候的格式，更加方便我们做自动化的管理。

## 2. 安装和配置

### 2.1. 安装

consul template和consul一样，也是一个命令集成了所有功能的模式

``` bash
wget https://releases.hashicorp.com/consul-template/0.25.1/consul-template_0.25.1_linux_amd64.zip
unzip 
mv consul-template /usr/local/sbin
```

验证安装

``` bash
consul-template -v
consul-template v0.25.1 (b11fa800)
```

### 2.2. 准备Json

下面是一个标准的JSON，这个是一个服务器的信息，我们需要把他注册到consul的服务中去，然后在让prometheus来直接读取

``` bash
{
  "ID": "node-exporter-10-114-2-129",
  "Name": "node-exporter-10-114-2-129",
  "Tags": [
    "node-exporter",
    "elasticsearch",
    "kibana"
  ],
  "Address": "10.114.2.129",
  "Port": 9100,
  "Meta": {
    "team": "cloud",
    "project": "monitoring",
    "cluster": "ubr-dc-prod"
  },
  "enable_tag_override": false,
  "Check": {
    "name": "health check of node_exporter",
    "HTTP": "http://10.114.2.129:9100/",
    "tls_skip_verify": true,
    "method": "GET",
    "Interval": "10s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}
```

一般来说，我们是通过curl命令来做的

``` bash
curl -XPUT --data @node_exporter-10-114-2-129.json http://10.114.2.68:8500/v1/agent/service/register
```

但是懒惰如我们，肯定是不会每次都去编辑一个文件，再用curl的方式把数据put给consul的，我们就需要一个模板，也就是consul template

``` bash
cat > tmpltest.ctmpl << EOF
{{range services}}
{{.Name}}
{{range .Tags}}
{{.}}{{end}}
{{end}}
EOF
```

我们先来试一下

``` bash
consul-template -consul-addr 10.114.2.68:8500 -template "tmpltest.ctmpl:result" -once
```

结果会把当前服务器中的service，都通过这个模板get出来

``` bash
consul


node-exporter-10-114-2-129

elasticsearch
kibana
node-exporter
```

