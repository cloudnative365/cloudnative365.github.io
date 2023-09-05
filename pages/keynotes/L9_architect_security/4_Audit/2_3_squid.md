---
title: squid上网代理
keywords: keynotes, architecture, security, audit, squid
permalink: keynotes_L9_architect_security_4_audit_2_2_squid_proxy.html
sidebar: keynotes_L9_architect_sidebar
typora-copy-images-to: ./pics/2_2_squid_proxy
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

### 1.1. 上网代理

上网代理一般是指从公司内部网络访问互联网的统一出口，可以代理http，https或者FTP服务，通过对访问的统一管理，来达到安全审计的目的。

目前代理服务器主流厂商主要有ISA、CCPorxy(软件)、Bluecoat、Websense、NetAPP以及网康、深信服等，内网安全设备主要有Bluecoat、IronPort以及网康、瑞星、H3C、华为等。其中两者融合的安全代理服务器的厂商其实很少，目前的主流品牌主要有网康的NPS，深信服的SG，Bluecoat的Proxy SG + Proxy AV。

如果公司并不打算购买硬件的产品，那么我们可以使用开源的方案来替代，比如，我们今天要说的squid。

![How Does a Squid Proxy Server Work - Developers, Designers & Freelancers -  FreelancingGig](/pages/keynotes/L9_architect_security/4_Audit/pics/2_3_squid_proxy/How-does-Squid-Proxy-Server-works.png)

### 1.2. squid

squid是一款古老且稳定的上网代理，被很多的软件仓库收录。因此，我们可以很方便的从ubuntu，rhel这些常见的Linux发行版的软件仓库中直接使用apt或者yum来安装。

![Trying out squid proxy with HTTP & HTTPs in Ubuntu — Part 1 | by Dushan  Silva | Mar, 2021 | Medium | Medium](/pages/keynotes/L9_architect_security/4_Audit/pics/2_3_squid_proxy/0*sdEnr1nP8XiaUcpp.png)

### 1.3. 企业级squid

由于squid本身并不支持集群，所以我们需要在前面加一层反向代理来实现集群功能。同时，为了控制上网，我们需要认证的用户才能上网。并且，上网动作本身也需要审计。因此，我们有了下面的架构。

![squid.drawio](/pages/keynotes/L9_architect_security/4_Audit/pics/2_3_squid_proxy/squid.drawio.png)![squid.drawio](/pages/keynotes/L9_architect_security/4_Audit/pics/2_3_squid_proxy/squid.drawio.png)

### 1.4. 单网卡还是双网卡

看了上面的图，细心的朋友会发现，1.1中的图是双网卡，但是1.3中却只有一个IP（两个IP是说两个服务器，而不是一个服务器两个IP）。其实这是两种不同的架构，使用那种架构，取决于各位的服务器环境。

+ 单网卡：这个好理解，就是说机器能不能上网，取决于网络上有没有给你配置SNAT规则。SQUID机器的IP是配置了SNAT规则的，所以可以上网，其他的机器没有SNAT规则，就需要访问squid代理服务器来上网。因此一个网卡就够了。
+ 双网卡：网卡分内网和外网，内网的网卡和外网的网卡不在一个网络中（通常是2个vlan），通过网卡转发的方式，将请求转发到互联网上。这就需要squid服务器**打开内核参数ipv4_forward**。并且**配置两个网关（Linux默认只有一个网关，另外的网关需要手动添加静态路由）**，这种方式比较复杂，且不太常用，

## 2. 安装和配置单网卡架构

### 2.1. 服务器

| IP           | 功能   | 备注             |
| ------------ | ------ | ---------------- |
| 10.11.79.18  | nginx1 | VIP：10.11.79.20 |
| 10.11.79.19  | nginx2 | VIP：10.11.79.21 |
| 10.11.149.11 | squid1 |                  |
| 10.11.149.12 | squid2 |                  |

### 2.2. 安装与配置

#### 2.2.1. 参考

+ RHEL9的配置：https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/deploying_different_types_of_servers/configuring-the-squid-caching-proxy-server_deploying-different-types-of-servers#setting-up-squid-as-a-caching-proxy-with-ldap-authentication_configuring-the-squid-caching-proxy-server
+ 开源社区文档：http://www.squid-cache.org/Doc/

#### 2.2.2. 准备

+ 保证squid的两台服务器是可以访问互联网的。企业中服务器直接访问互联网基本都是NAT出去的，具体的可以和网络工程师去沟通。
+ 在1.3中标明了两台机器的网络访问。下面的端口需要保持联通
  + 客户端到nginx的8080端口HTTP/HTTPS协议畅通
  + nginx到squid服务器3128端口TCP协议畅通

#### 2.2.3. 安装

配置好yum repo之后

``` bash
# dnf -y install squid
# systemctl start squid
# systemctl enable squid
```

#### 2.2.4. 简单配置

其实目前我们已经可以通过代理服务器上网了，找一台不能上网的linux服务器，配置环境变量

``` bash
# export HTTP_PROXY=http://10.11.149.11:3128
```

然后找个能响应的页面，比如baidu

``` bash
# curl -I http://www.baidu.com
```

如果返回码是200，那就是没问题了

#### 2.2.5. 只允许特定网络访问

我们发现在默认配置中有下面一下配置                                                                                                                                                               

### 2.3. Nginx做高可用代理

nginx就使用上一篇文章中的高可用nginx做4层转发就好了

``` bash
```



### 2.4. LDAP认证

### 2.5. 日志发给splunk

## 3. 客户端配置

### 3.1. Linux服务器配置代理

### 3.2. 浏览器配置代理

### 3.3. Docker配置代理

### 3.4. Kubernete的pod配置代理
