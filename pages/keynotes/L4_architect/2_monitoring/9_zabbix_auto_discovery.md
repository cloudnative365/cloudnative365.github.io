---
title: Zabbix5.0自动发现
keywords: keynotes, architect, monitoring, zabbix_operations
permalink: keynotes_L4_architect_2_monitoring_9_zabbix_operations.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/9_zabbix_operations
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

如何配置自动发现并且关联模板

zabbix与ldap集成

如何配置zabbix监控vCenter与ESXi

Zabbix与Grafana集成

Zabbix的Proxy模式

Zabbix企业级架构

## 1. 简介

zabbix的亮点之一就是自动发现，配置起来比较简单，而且通过zabbix-agent的配合，能够让机器实现自动化注册。他通过对指定的网段，进行指定方式的扫描，然后通过动作（action）来对识别到的设备进行下一步动作。我们这里说的动作基本就是两种，就是分类和关联模板。我们这次就来实现一个自动识别，分类然后关联模板的实验。

而对于我们常见的vmware虚拟出来的虚拟机，zabbix是原生支持通过调用ESXi或者vCenter的接口（sdk）来实现自动发现的。在4.4版本中zabbix要监控vmware需要导入第三方模板，但是5.0版本的模板中会自带vmware的模板，我们需要通过宏（macro）来传递vmware的地址和只读账号等信息，让zabbix能够正确读取vmware中的信息。有一点需要注意，到这篇文章发表的日期，也就是2020年8月下旬位置，zabbix 5.0LTS版本中的vmware模板识别vCenter 7.0版本是问题的，有一些指标无法正确识别，而ESXi的7.0版本是可以正确读取数据的。

## 2. 自动发现

### 2.1. 服务器自动发现（zabbix agent）

+ 客户端配置

  需要安装并启动zabbix-agent，请参考这个[地址](https://www.zabbix.com/download_agents)，我们这里用的是Linux客户端，配置下面的参数，并重启zabbix-agent

  ``` bash
  Server=XXX.XXX.XXX.XXX
  Hostname=XXXXXX
  ```

+ 服务器配置

  zabbix在安装完成之后，在Configuration -> Discovery选项卡中，默认有一个规则，是自动检测192.168.0.1到254网段的设备，间隔是1小时，检查的标准是检查Zabbix agent，但是没有启用。我们这里创建一个新的规则。

  点击右上角的`Create discovery rule`按钮，进入配置界面

  ![image-20200825230148559](/pages/keynotes/L4_architect/2_monitoring/pics/9_zabbix_operations/image-20200825230148559.png)

  + name：随便起一个名字

  + IP range：使用xxx.xxx.xxx.xxx-xxx的格式来写，例子中是10.109.224.1到10.109.224.254网段

  + Update interval：遍历网段的时间间隔，默认是1小时，我们可以改成5m

  + Check：列出了一些检查的条件，我们添加一个zabbix agent的check，key使用system.uname

    ​	![image-20200825230305802](/pages/keynotes/L4_architect/2_monitoring/pics/9_zabbix_operations/image-20200825230305802.png)

  + 完成后，下面的几个选项就会发生变化

  ![image-20200825230245512](/pages/keynotes/L4_architect/2_monitoring/pics/9_zabbix_operations/image-20200825230245512.png)

  + Device uniqueness criteria：设备唯一标识，我们选ip，这是这个设备注册在zabbix中的唯一标识
  + Host name：依然选IP，因为服务器会去探测host name的10050端口，如果有正向解析，也可以选DNS name
  + Visible name：这是在界面上显示的名字，我们可以选择DNS name

  这样，一个自动发现规则就配置好了，不过这样只能实现发现主机，我们在Monitoring--> Discovery下面就可以看到我们发现的主机了

  

### 2.2. 客户端主动注册

配置刚才的agent，请参考这个[地址](https://www.zabbix.com/download_agents)，我们这里使用的是Linux的客户端，其他的大同小异，需要修改配置文件/etc/zabbix/zabbix_agentd.conf

``` bash
Server=XXX.XXX.XXX.XXX
#主动模式下zabbix服务器的地址，如果是proxy模式，就要写proxy的地址
ServerActive=XXX.XXX.XXX.XXX
# hostname和刚才配置的自动发现规则中的hostname对应，注册的时候会把这个hostname注册过去
Hostname=XXXXXX
# 这个相当于我们的标签，可以配置很多个，用于发现的规则和分类
HostMetadata=linux zabbix.jormun
```

配置完成之后重启客户端，会立即去服务器注册。以后会根据`RefreshActiveChecks`定期发送check

+ 总结：从官方的角度来说，我们刚才说的是两种方式，一种是服务器发现，第二种是客户端主动注册，而在生产中，这两种方式是结合来使用的，也就是在负载不大的情况下，两种方式都会开启。但是，如果我们的负载非常庞大，就会让

### 2.2. 客户端主动注册

https://blog.csdn.net/diyuan8262/article/details/101501636

https://www.zabbix.com/forum/zabbix-help/35549-howto-use-zabbix-agent-in-discovered-vm-guests?t=43531

https://www.zabbix.com/documentation/current/manual/vm_monitoring

## 3. Zabbix和AD集成

### 3.1. 配置zabbix认证方式为AD

为了方便公司内部人员登录zabbix的界面，我们通常会和windows的AD集成，方便用户的管理。

调试的时候需要安装ldapsearch命令

``` bash
yum install openldap-clients
```

然后我们就可以使用ldap命令进行调试了

``` bash
ldapsearch -x 'memberof=CN=rol-infra-infra-s-g,OU=rol,OU=SecurityGroup,DC=UBRMB,DC=COM' -LLL -H ldaps://UBMPAPWP00001.ubrmb.com:636 -b 'dc=UBRMB,dc=COM' -D 'cn=s000006,ou=ServiceAccount,dc=UBRMB,dc=COM' -w 'RX6mxv5nyu' -o ldif-wrap=no
```

最后在界面上配置

![image-20200904153149077](/pages/keynotes/L4_architect/2_monitoring/pics/9_zabbix_operations/image-20200904153149077.png)

### 3.2. 配置ldap同步脚本

Zabbix的认证是去AD服务器认证的，但是账号还是需要手动创建，为了让我们的工作更加简便，我们会使用脚本的方式，定期把ldap中的用户信息导入到Zabbix中。这样，如果有新员工入职了，只要获得了AD的账户，并且获得了对应的AD权限，那么，他就可以登录Zabbix控制台，而不需要Zabbix管理员手动添加账户了。



## 4. Zabbix监控vCenter和ESXi

### 4.1. 配置Zabbix监控vCenter或者ESXi

Zabbix 5.0中，Zabbix的vmware监控模板对于vCenter7.0版本支持并不是很好，但是ESXi目前还好。如果一定要使用Zabbix监控ESXi，还是建议手动添加ESXi主机。

### 4.2. 对发现的主机自动分组和关联模板



## 5. Zabbix与Grafana集成



## 6. Zabbix的proxy模式



## 7. Zabbix企业级架构