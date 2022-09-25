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

设备的自动发现和注册

SNMP和SNMP trap

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

Zabbix的认证是去AD服务器认证的，但是账号还是需要手动创建，为了让我们的工作更加简便，我们会使用脚本的方式，定期把ldap中的用户信息导入到Zabbix中。这样，如果有新员工入职了，只要获得了AD的账户，并且获得了对应的AD权限，那么，他就可以登录Zabbix控制台，而不需要Zabbix管理员手动添加账户了。在5.0之前的版本，我们经常使用一个ldap的同步脚本[dnaeon/zabbix-ldap-sync](https://github.com/dnaeon/zabbix-ldap-sync)。这个脚本目前已经被zabbix收录在[zabbix-tooling](https://github.com/zabbix-tooling)这个仓库中，这个仓库中目前收录了一些社区贡献的代码。



## 4. 设备的自动发现和注册

### 4.1. 配置Zabbix自动发现规则

### 4.2. 对发现的主机自动分组和关联模板

## 5. SNMP和SNMP trap

通过SNMP监控是Zabbix的亮点之一，目前2021年的流行架构中，使用zabbix来做LLD（Low Level Discovery）依然是非常流行的做法，他的优点在于zabbix中有非常丰富的模板，我们可以非常轻松的实现全网扫描和SNMP发现，进而实现对于一些机房设备的监控。因为这些机房设备基本上是不支持安装agent的，所以我们只能通过SNMP（Simple Network Management Protocol）来监控设备，这些设备都使用统一的规则来暴露自己的指标，这些指标遵循管理信息结构SMI（Structure of Management Information），SMI定义了SNMP框架所用信息的组织和标识，为MIB定义管理对象及使用管理对象提供模板。而MIB定义了可以通过SNMP进行访问的管理对象的集合。SNMP中的MIB是一种树状数据库，MIB管理的对象，就是树的端节点，每个节点都有唯一位置和唯一名字.IETF规定管理信息库对象识别符（OID，Object Identifier）唯一指定，其命名规则就是父节点的名字作为子节点名字的前缀。

### 5.1. SNMP监控

zabbix中提供了非常丰富的SNMP监控模板，在界面上有三处SNMP模板非常集中的地方

+ Templates/Network devices
+ Templates/Server hardware
+ Templates/Module/Template Module Generic SNMP

这些都是Zabbix给我们准备好的模板。这些模板调用的MIB库其实都是从操作系统上调用的，以Centos7/RHEL7为例，他存放的位置是/usr/share/snmp/mibs。如果我们想添加一下第三方或者是自定义的MIB，需要我们在/etc/snmp/snmpd.conf中引用一下，加上这一行

``` bash
mibs : /usr/share/snmp/mibs/*
```

### 5.2. SNMP  trap

trap是由被监控端主动向监控端发送SNMP消息的方式。如果是snmp v2需要我们在客户端配置好community，如果是v3，可能还要配置其他的参数。详细请参考各个设备的说明书。在服务器端，我们需要安装相关组件。

```
yum install -y net-snmp net-snmp-utils net-snmp-perl
```

注意，不同的操作系统类型可能有不一样的地方，比如在RHEL7上，net-snmp-perl是需要特殊的订阅才可能安装，我们可能需要去其他的地方去下载对应的包

``` bash
yum -y install http://www.rpmfind.net/linux/centos/7.9.2009/updates/x86_64/Packages/net-snmp-perl-5.7.2-49.el7_9.1.x86_64.rpm
```

调整/etc/snmp/snmptrapd.conf的配置，假设被监控设备的community是device@123

``` bash
authCommunity log,execute,net device@123
traphandle default /usr/sbin/snmptthandler
```

Zabbix在实现snmptrap有两个条件，第一个是能收集和记录snmptrap信息的程序，这个程序就是snmptrapd，第二个是需要把这些信息发送到Zabbix server，这个软件是snmptt，当然也可以是perl脚本，本篇文章以snmptt为例，需要我们提前配置好epel源

``` bash
yum intall snmptt
```

修改/etc/snmp/snmptt.ini

``` ini
date_time_format=  %Y/%m/%d %H:%M:%S 
net_snmp_perl_enable = 1 
translate_log_trap_oid = 2
```

修改/etc/snmp/snmptt.conf，把数据处理格式改一下

``` bash
EVENT general .* "General event" Normal 
FORMAT ZBXTRAP $aA $ar 
```

在zabbix上开启snmptrap，修改`/etc/zabbix/zabbix_server.conf `

``` bash
StartSNMPTrapper=1
```

重启相关服务

``` bash
service snmptt restart 
service snmptrapd restart 
service zabbix-server restart 
```

创建log文件

``` bash
mkdir /var/log/snmptt 
touch /var/log/snmptt/snmptt.log 
chown snmptt:snmptt 
```





## 6. Zabbix与Grafana集成

grafana是通过把zabbix作为数据源来把数据集成到grafana当中的。在grafana默认的配置当中，zabbix并不是标准的数据源，我们需要

## 7. Zabbix的proxy模式



## 8. Zabbix企业级架构