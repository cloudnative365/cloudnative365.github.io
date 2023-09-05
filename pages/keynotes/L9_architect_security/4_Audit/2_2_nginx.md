---
title: nginx反向代理
keywords: keynotes, architecture, security, audit, nginx
permalink: keynotes_L9_architect_security_4_audit_2_2_nginx_proxy.html
sidebar: keynotes_L9_architect_sidebar
typora-copy-images-to: ./pics/2_2_nginx_proxy
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

### 1.1. 负载均衡

负载均衡（Load Balance），简称LB，其含义就是指将负载（工作任务）进行平衡、分摊到多个操作单元上进行运行，例如FTP服务器、Web服务器、企业核心应用服务器和其它主要任务服务器等，从而协同完成工作任务。

负载均衡构建在原有网络结构之上，它提供了一种透明且廉价有效的方法扩展服务器和网络设备的带宽、加强网络数据处理能力、增加吞吐量、提高网络的可用性和灵活性。

常见的企业级负载种类繁多，有硬件和软件两种。

+ 硬件：F5，深信服，华为，H3C都有自己的硬件负载均衡。
+ 软件：Nginx，Apache，Haproxy，LVS，Traefik，Envoy等等。

值得一提的是公有云，我们常见的带LB结尾的都是负载均衡类的。从技术实现的角度，大概是三种：

+ ELB：也叫弹性负载均衡或者传统负载均衡，基本上都是利用传统的开源工具改造而成。
+ ALB：application LB，专门来做7层负载均衡的。这类产品主要是用来做虚拟主机的代理。
+ NLB：Network LB，网络负载均衡用来做4层负载均衡，这类的产品速度上限高，因为他并不会做网络7层的拆包。

### 1.2. 反向代理

反向代理是负载均衡的功能之一。上面提到的软件负载均衡（Nginx，Apache，Haproxy，LVS，Traefik，Envoy等等）都具有反向代理功能。总的来说，反向代理有两种，一种是4层代理，一种是7层代理。而每种软件对于4层或者7层代理的方式不太一样。我们以Nginx为例，他可以同时实现4层和7层代理，同时把访问的日志发给splunk来做审计，实现了我们入口的统一管理和审计。

反向代理中可以实现的功能非常多，比如：后端健康检测，动态限流，重定向。如果这些功能都不能满足需求，我们还可以通过插件的方式来对反向代理进行功能的扩展，比如Nginx上可以通过LUA语言进行开发。

## 2. 架构图

![UserAccess.drawio](/pages/keynotes/L9_architect_security/4_Audit/pics/2_2_nginx_proxy/UserAccess.drawio.png)

对于外网用户而言，如果从互联网来访问内网资源，必然要经过一些安全产品，比如网络防火墙，DDOS和waf之类。通常WAF是自带负载均衡功能的，但是要注意，通常的WAF是指 WEB Application Firewall，也就是7层的负载均衡。如果需要4层的负载均衡，就需要Ngnix来帮忙了。

对于内网用户而言，访问可以被访问的服务，都需要通过Nginx来跳转，也就是反向代理功能，我们一般会使用nginx的stream或者upstream模块来分发流量。用户不用在意后端的服务器地址是什么，所有的健康监测，流量切换全部由Nginx来完成。既保护了后端的服务器，又给了用户非常好的体验。

## 3. 单机版

### 3.1.安装

一般发行版的源中都会带有安装包，在源的配置正确的情况下，我们可以直接使用`yum -y install nginx `或者`apt-get install nginx`来安装。也可以使用编译的方式来安装，`./configure`这种方式。编译安装主要用于我们自己开发了一些模块，或者一些社区的三方模块缺失的情况，我们可以在编译的时候，为nginx增加功能。但是大部分情况下，源安装包中提供的功能基本就够了。包名基本都是nginx-xxx的形式来命名的，通过`yum list nginx*`或者`yum list|grep nginx`来找到对应的包。比如stream模块就需要额外安装。

### 3.2. 配置

这种的配置教程一搜一大把，我这里要说的主要是对于nginx的配置文件的管理，让我们的配置更加清晰。

对于初次安装的系统，配置文件在/etc/nginx下面，主要的配置文件是nginx.conf。为了让配置更清晰，我们只在主配置文件中增加各个配置的位置，http段中放7层代理，stream段中放4层代理

``` bash
http {
...
    include /etc/nginx/conf.d/*.conf;
}
stream {
...
    include /etc/nginx/conf.d/*.stream;
}
```

指定新增的配置文件在/etc/nginx/conf.d下面

``` bash
# tree /etc/nginx/conf.d
/etc/nginx/conf.d
├── ansible.conf
├── gitlab.conf
├── proxy.stream
└── yum.conf

0 directories, 4 files
```

证书也创建独立的文件夹进行存放

``` bash
tree /etc/nginx/cert/
/etc/nginx/cert/
├── server.crt
└── server.key

0 directories, 2 files
```

## 4. 高可用架构

### 4.1. IPVS

高可用架构主要是通过漂IP的方式来实现了一个Active-Active的架构，我们最常用的方式就是使用keepalived，易用且架构简单。但是要注意，在多节点的情况下，keepalived非常容易脑裂，因为他是基于流言协议VRRP，这是一种不可靠的协议，通过多数通过的方式来决定是否需要来切断服务和连接。一但各个节点之间产生分歧，很可能就各自为政，产生脑裂。

但是在双节点之间，这个还是很好用的，我们需要增加一个触发器，一但出现情况，就直接干掉服务，防止脑裂。

![NginxHA.drawio](/pages/keynotes/L9_architect_security/4_Audit/pics/2_2_nginx_proxy/NginxHA.drawio.png)

### 4.2. 规划

我们的两台机器规划如下

| 机器名           | 网卡IP      | 虚拟IP      |
| ---------------- | ----------- | ----------- |
| bjpvlpitsng00057 | 10.11.79.18 | 10.11.79.19 |
| bjpvlpitsng00058 | 10.11.79.20 | 10.11.79.21 |

如果第一台机器挂了，那么第二台机器就会接管第一台机器

### 4.3. 安装

通过yum包安装

``` bash
# yum -y install keepalived
```

### 2.2. 配置

配置文件位置/etc/keepalived/keepalived.conf

+ global_defs主要用来配置全局属性，比如发邮件的配置
+ 里面包含了两段的配置VI_1和VI_2，分别对应VIP1和VIP2
+ 两台机器的配置是对应的，请参考下面的配置



在10.11.79.18上

``` bash
! Configuration File for keepalived

global_defs {
   notification_email {
     jialun.li@dxc.com
   }
   notification_email_from its@carizon.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id bjpvlpitsng00057
}
vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    timeout 5
    weight -20
}
vrrp_instance VI_1 {
    state MASTER
    interface ens34
    virtual_router_id 51
    priority 110
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.11.79.20
    }
    track_script {
        chk_nginx
    }
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens34
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.11.79.21
    }
    track_script {
        chk_nginx
    }
}
```

在10.11.79.19上

``` bash
! Configuration File for keepalived

global_defs {
   notification_email {
     jialun.li@dxc.com
   }
   notification_email_from its@carizon.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id bjpvlpitsng00058
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    timeout 5
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens34
    virtual_router_id 51
    priority 100
    advert_int 1
    smtp_alert
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.11.79.20
    }
    track_script {
        chk_nginx
    }
}
vrrp_instance VI_2 {
    state MASTER
    interface ens34
    virtual_router_id 52
    priority 110
    advert_int 1
    smtp_alert
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.11.79.21
    }
    track_script {
        chk_nginx
    }
}
```

检查脚本/etc/keepalived/nginx_check.sh，主要目的是检查nginx进程，如果nginx进程挂了，并且无法启动，那么就把keepalived也停掉，防止脑裂

``` bash
#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    systemctl restart nginx
    sleep 2
    counter=$(ps -C nginx --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
        systemctl stop keepalived
    fi
fi
```

# 
