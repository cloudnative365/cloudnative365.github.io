---
title: grafana
keywords: keynotes, senior, metrics, grafana
permalink: keynotes_L3_senior_5_metrics_4_grafana.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/4_grafana
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. 概述

### 1.1. 数据展示

数据采集完毕了，终于到了可以展示的环节了。目前来说，常见的开源展示工具有

+ grafana：grafana lab的产品，兼容性特别好，适合做集成的展示
+ kibana：主要为ES服务，分析功能非常好，监控展示只不过是顺路而已
+ zabbix/nagios/cacti/Graphite/open-falcon/Chronograf：这些工具自带UI，但是比较简陋，功能上没问题，但是作为集成工具就不行了
+ kiali/jaeger：用于服务网格的展示

我们这里先说grafana，日志专题中讲kibana，微服务专题中讲kiali和jaeger

### 1.2. Grafana

grafana是由go和typescript语言编写的，他以界面的绚丽和集成功能的强大而受到开源社区的追捧

![image-20200921100533269](/pages/keynotes/L3_senior/5_metrics/pics/4_grafana/image-20200921100533269.png)



### 1.3. grafana lab

grafana这个软件是由grafana labs开发和维护的，由他维护的开源项目中有下面几个

![image-20200920213606338](/pages/keynotes/L3_senior/5_metrics/pics/4_grafana/image-20200920213606338.png)

+ grafana：监控指标展示工具
+ graphite：早期监控工具，是zabbix，nagios时代的产品
+ metricstank：为graphite提供了多租户管理
+ tanka：为kubernetes提供了扩展功能
+ cortex：为prometheus提供了高可用，多租户，持久化和快速部署prometheus等功能，和前面的thanos同时隶属于CNCF的孵化项目
+ Loki：日志收集系统，我们后面日志专题会说他
+ Prometheus：非常著名，不解释了



## 2. 搭建grafana

+ 下载地址：https://grafana.com/grafana/download

+ 在centos上安装

  ``` bash
  wget https://dl.grafana.com/oss/release/grafana-7.1.5-1.x86_64.rpm
  sudo yum install grafana-7.1.5-1.x86_64.rpm
  ```

+ 数据库，如果默认启动的话，grafana会默认使用sqlite数据库，但是这个方式实现grafana的HA就比较困难了。建议大家使用postgresql或者mysql，我们这节课是个Demo，所以不讲集群架构，咱们先使用单机的postgresql

  ``` bash
  yum install postgresql-12
  ```

+ 配置权限

  ``` bash
  
  ```

+ 启动grafana

  ``` bash
  systemctl start grafana-server
  ```

+ 默认密码是admin/admin

## 3. grafana插件

### 3.1. 插件的类型

## 4. ldap认证

nginx支持多种的认证方式，从[官方手册](https://grafana.com/docs/grafana/latest/auth/overview/)中可以看到，本地认证，ldap，AD，oauth2等，还有可以从共有云直接认证。需要注意的是，有一种叫做enhanced ldap认证，他比普通认证多的功能是同步。ldap认证类似于zabbix的ldap认证，认证过程是去ldap服务器。但是，但是如果想要授权，需要其他配置，在grafana中叫做mapping。而zabbix就需要手动配置了

### 4.1. 配置ldap认证

在主配置文件目录下创建新的ldap认证文件/etc/grafana/ldap.toml，或者手动在grafana.ini文件中指定，并且修改grafana.ini文件如下

``` bash
[auth.ldap]
# 启动ldap认证
enabled = true

# ldap配置文件位置 (默认是: `/etc/grafana/ldap.toml`)
config_file = /etc/grafana/ldap.toml

# 允许用户注册（需要ldap认证是ok的）。如果配置为false，只允许实现配置的用户登录。
allow_sign_up = true
```

然后配置`/etc/grafana/ldap.toml`文件来配置映射

``` ini
[[servers]]
# LDAP服务器信息在这里
host = "ldap.jormun.com"
port = 636
use_ssl = true
start_tls = false
ssl_skip_verify = false
bind_dn = "cn=s000064,ou=ServiceAccount,dc=JORMUN,dc=COM"
bind_password = 'YourBindPassword'
search_filter = "(sAMAccountName=%s)"
search_base_dns = ["dc=UBRMB,dc=COM"]

[servers.attributes]
# 这边是用户的属性映射
name = "givenName"
surname = "sn"
username = "cn"
member_of = "memberOf"
email =  "mail"

[[servers.group_mappings]]
# 这边登录上来的都是org_id为4的用户，都是org为4的用户的admin
group_dn = "CN=rol-infra-infra-s-g,OU=rol,OU=SecurityGroup,DC=JORMUN,DC=COM"
org_role = "Admin"
grafana_admin = false
org_id = 4

[[servers.group_mappings]]
# 其他登录上来的都放到org的id为5的org，但是其实不写也可以
group_dn = "*"
org_role = "Viewer"
org_id = 5
```



## 5. 权限控制

### 5.1. 权限控制的粒度

组织org是organization的缩写，在grafana里面的orgs可以认为是一个公司，而一个公司下面有很多的team，每个team中有user，而user是有角色的，他们有三种角色admin，editor和viewer。我们在创建好grafna的时候，是默认有一个组织的，叫main org，里面只有一个用户admin。

为了让大家更好理解，我们创建一个组织org叫Cloud team。由于我们前面集成了LDAP，我们可以把一整个组都映射给这个org（Cloud team），让他们作为cloud team的admin。然后退出登录，用一个在ldap中mapping的用户来登录，登录后会显示这个org下的配置，当然，第一个登录上来是什么都没有的。

## 6. 配置HTTPS

配置https有两种方式，一种是在反向代理服务器，比如nginx上配置ssl证书，实现https。另外一种是直接在grafana服务器上配置https证书

### 6.1. 在nginx上配置ssl证书

首先找到nginx的主配置文件nginx.conf，在http段中加入`include /etc/nginx/conf.d/http.d/*.conf;`。然后增加配置文件include /etc/nginx/conf.d/http.d/grafana.conf

``` bash
server {
        listen 443;
        server_name  grafana.monitor.ubrmb.com;
        ssl on;
        ssl_certificate /etc/nginx/ssl/monitor.ubrmb.crt;  
        ssl_certificate_key /etc/nginx/ssl/monitor.ubrmb.rsa;  
        ssl_session_timeout 5m;  
        ssl_protocols SSLv2 SSLv3 TLSv1;  
        ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;  
        ssl_prefer_server_ciphers on; 

location / {
        proxy_pass http://grafana;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        }
}

upstream grafana {
        ip_hash;
        server 10.114.2.70:3000;
        server 10.114.2.71:3000;
}
```

重启服务

``` bash
systemctl restart nginx
```

### 5.2. 在grafana配置上配置ssl证书