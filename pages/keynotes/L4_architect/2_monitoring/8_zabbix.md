---
title: Zabbix5.0尝鲜
keywords: keynotes, architect, monitoring, complex_query
permalink: keynotes_L4_architect_2_monitoring_8_zabbix.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/8_zabbix
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

了解Zabbix 5.0

快速搭建Zabbix

## 1. 简介

Zabbix在开源监控界算是比较知名的，主要是他的社区活跃，监控模板众多。虽然在云原生时代，他的设计理念可能和目前容器化架构有一些隔阂，但是并不影响他在数据中心级别监控的地位。我们常用的功能，比如：分布式服务，分布式采集，画图功能，和强大的自定义功能，都是我们在数据中心级别监控中不可或缺的。

而我们使用zabbix的最重要的原因之一，或者说zabbix之所以还依然屹立不倒的原因之一，就是他的主动发现功能。尽管他没办法像prometheus一样主动去etcd或者consul中快速查找信息，但是他可以通过一些网络手段主动去扫描特定网段中的设备，一旦发现符合特征要求的主机，就把他加到我们的主机列表中，然后通过action对这些发现的主机进行操作，比如分配模板或者是分配到指定的组当中去。

我们这次使用的是zabbix 5.0 LTS版，这个版本是目前最新的版本，他在4.xx的基础上做了一些改动。比较大的改动，比如zabbix-agent换成了go语言来开发，更轻量了，模板也增加了很多原生支持的功能，比如vsphere的支持（目前只支持6.xx的版本，对于7.0的版本支持还不好）

## 2. Zabbix 5.0新特性

其实我犹豫了一下到底要不要讲Zabbix，因为毕竟这个是一个被各大机构讲烂了的话题，但是作为一套完整的监控教程，又不得不说一下，以后在生产上也是用的到的，正好5.0LTS版本刚不不久，我们就接着体验的由头来给大家上手一下新版的zabbix。

目前官方列出的新特性如下，转自官方网站https://www.zabbix.com/cn/whats_new_5_0

![image-20200816202811661](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816202811661.png)

我们可以看到，他支持一些新版本的系统了，比如CentOS/RHEL8，Ubuntu20.04和Suse15。这些都是2019年年底或者是2020年上半年才发布的操作系统。其实最令我兴奋的是，他开始支持ARM架构的CPU了（红色方框），也就是说，我们的监控可以部署在树莓派或者其他开发板上部署了。

也许有的朋友还不知道这意味着什么，大家有没有想过，如果我们在机柜上安装上一片树莓派，然后直接使用机柜来做我们zabbix的proxy-server或者zabbix-agent，这样是不是非常棒呢？如果我们想要升级，就直接在树莓派上安装一个docker和kubeproxy，然后让k8s来管理我们整个的监控系统，这样是不是能让我们的监控更加强壮呢？



![image-20200816203628508](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816203628508.png)

这个主要说的是我们可以在云上直接使用，比如AWS的market place中直接拖拽镜像来在云上部署，而docker和openshift则是通过docker镜像来实现一键部署的



![image-20200816204104500](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816204104500.png)

SAML的全称是安全断言标记语言（英语：Security Assertion Markup Language，简称*SAML*，发音sam-el）是一个基于XML的开源标准数据格式，它在当事方之间交换身份验证和授权数据，尤其是在身份提供者和服务提供者之间交换。我们可以简单理解为我们登陆时，账户的验证，授权等可以有更多的方式了。



![image-20200816204331592](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816204331592.png)

这边是对于安全方面改进，增加了数据库的传输加密，所有的组件都可以配置加密，包括agent，proxy，还有webhook



![image-20200816204544922](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816204544922.png)

密码从小圆点变成了锁。。。。



![image-20200816204640079](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816204640079.png)

zabbix的内置的是TimescaleDB，数据的分区功能对于查询热点数据的速度很有帮助，高性能和可扩展性也得到了加强，但是，和新时代的监控比起来依然很慢。



![image-20200816204902585](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816204902585.png)

新的agent由golang编写，更轻量了，速度更快，资源使用更小，可以直接从4.xx的agent升级到5.xx，不需要卸载，直接覆盖



![image-20200816205037997](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205037997.png)

这是一些使用上的改进



![image-20200816205141892](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205141892.png)

trigger更复杂了。。。



![image-20200816205216199](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205216199.png)

卖点功能更强大



![image-20200816205249594](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205249594.png)

界面改了一些，不过图形依然很low



![image-20200816205338652](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205338652.png)

支持自动开工单，不过要使用webhook的方式



![image-20200816205412593](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205412593.png)

支持的报警方式更多了



![image-20200816205446556](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200816205446556.png)

模板升级了，更多了，这也是运维喜欢使用zabbix的原因之一，监控模板非常丰富，基本满足我们的需求了

## 3. 快速搭建Zabbix

3.1. 官方文档：[点这里](https://www.zabbix.com/cn/download)

![image-20200817222548186](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200817222548186.png)

上面的是选择安装方式，但是并没有proxy-server的安装方式，我们后面会讲一下

![image-20200817222702312](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200817222702312.png)

下面是一个选择界面，通过选择不同的组合，下面会生成对应的安装方式。

3.2. 安装数据库

在新版中依然是MySQL或者PgSQL作为数据库，但是文档中没有写数据库的安装。

一般来说，这两种数据库的集群模式都是主备的模式，如果我们借助第三方工具可以实现主主模式，但是如果我们有大量的服务器的话（几千台）还是建议选择主备，主主模式如果同步不及时，就会影响数据的一致性。

如果想要减轻服务器的负担，可以考虑读写分离的方式，写请求全部去zabbix-server的主服务器去做，读请求全部分配到zabbix-server的备份服务器去做。

这个说来简单，实现的方式有两种，一种是在负载均衡器，比如：nginx上做，但是nginx的负载均衡是没有办法区分GET,PUT和UPDATE请求的，我们需要借助LUA插件来实现，或者干脆使用openresty。另外一种是让zabbix-server在连接数据库的时候，选择不同的库，zabbix-server的采集或者拉取功能全部去主库，而zabbix-server和grafana集成做展示的时候，让另外一台zabbix-server去备库拿信息。

![image-20200817225122944](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200817225122944.png)

当然，这个方式还有个要求，那就是高可用的要求，也就是说，我们还是需要借助nginx，只不过我们在upstream的时候会选择一个机器作为backup机器，只有当主机器down了的时候，我们才会去backup机器。

我们这里选择的是CentOS8作为操作系统，默认的CentOS8可以使用PgSQL10作为默认的版本，但是他也支持9.6和12版本

``` bash
$ dnf module list postgresql
上次元数据过期检查：0:23:14 前，执行于 2020年08月17日 星期一 10时35分00秒。
CentOS-8 - AppStream
Name         Stream   Profiles             Summary                              
postgresql   9.6      client, server [d]   PostgreSQL server and client module  
postgresql   10 [d]   client, server [d]   PostgreSQL server and client module  
postgresql   12       client, server [d]   PostgreSQL server and client module  

提示：[d]默认，[e]已启用，[x]已禁用，[i]已安装
```

【d】表示默认，也就是版本10，如果想要安装版本12，需要执行

``` bash
$ dnf install @postgresql:12
```

我们把contrib包也装上，这里面是一些工具

``` bash
$ dnf install postgresql-contrib
```

然后初始化一下，主要是为了创建数据文件，如果我们做坏了，直接删除数据文件，再初始化一下就好了

``` bash
$ postgresql-setup initdb
```

启动PgSQL，并且配置开机启动

``` bash
$ systemctl start postgresql
$ systemctl enable postgresql
```

3.3. 安装zabbix

配置repo

``` bash
rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-5.0-1.el8.noarch.rpm
```

安装组件

``` bash
$ dnf install zabbix-server-pgsql zabbix-web-pgsql zabbix-nginx-conf zabbix-agent
```

这里要注意，zabbix网站在国内访问时非常不稳定，有可能在选择对应的环境之后，下面的文档没有及时变化，一定要看好安装的包，这里安装的包如下：

+ zabbix-server-pgsql：zabbix-server主程序，如果是mysql版本的话名字叫zabbix-server-mysql
+ zabbix-web-pgsql：zabbix-server的web界面，如果是mysql版本的话叫zabbix-web-mysql
+ zabbix-nginx-conf: zabbix的nginx配置，由于zabbix的界面是php的，且会使用fpm模式，所以nginx需要额外配置一下，如果是apache的叫zabbix-apache-cong
+ zabbix-agent：这个就是agent了，在被监控端只安装这个就好了

3.4. 配置数据库

官网上是用sudo的方式来做的

``` bash
# sudo -u postgres createuser --pwprompt zabbix
# sudo -u postgres createdb -O zabbix zabbix
# zcat /usr/share/doc/zabbix-server-pgsql*/create.sql.gz | sudo -u zabbix psql zabbix
```

我这里是用root直接登录，所以命令变成了

``` bash
[root@zabbix ~]# su - postgres -c "createuser --pwprompt zabbix"
Enter password for new role:  # 这里输入zabbix
Enter it again:  # 重复输入zabbix
[root@zabbix ~]# su - postgres -c "createdb -O zabbix zabbix"
```

导入表结构

``` bash
# su - zabbix -c "zcat /usr/share/doc/zabbix-server-pgsql*/create.sql.gz | psql"
```

连到数据库看一下表是不是都导入成功了

``` bash
# su - zabbix -c "psql"

zabbix=> \c zabbix;
zabbix=> \d
                    List of relations
 Schema |            Name            |   Type   | Owner  
--------+----------------------------+----------+--------
 public | acknowledges               | table    | zabbix
 public | actions                    | table    | zabbix
 public | alerts                     | table    | zabbix
 public | application_discovery      | table    | zabbix
 public | application_prototype      | table    | zabbix

```

注意：这里的schema名字叫`public`，我们后面会用到

然后需要修改下pg_hba.conf文件，配置psql的访问权限

``` bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# zabbix
local   zabbix          zabbix                                  md5  # <= 这行是新加的
host    zabbix          zabbix          127.0.0.1/32            md5  # <= 这行是新加的

# "local" is for Unix domain socket connections only
local   all             all                                     peer

```

注意：一定要加在上面，因为这个文件是从上到下读取的，如果加在下面，就会被前面的规则匹配到，我们新加的就不生效了，然后需要重启数据库`systemctl restart postgresql`

3.5. 配置zabbix

修改/etc/zabbix/zabbix_server.conf文件

``` bash
DBPassword=zabbix #改成刚才配置数据库的时候配置的密码
```

配置/etc/nginx/conf.d/zabbix.conf, 把下面两行的`#`取消注释并且配置成我们需要的

``` bash
listen 80;
server_name example.com;
```

配置/etc/php-fpm.d/zabbix.conf，改成我们的时区

``` bash
php_value[date.timezone] = Asia/Shanghai
```

启动服务并且设置为开机启动

``` bash
systemctl restart zabbix-server zabbix-agent nginx php-fpm
systemctl enable zabbix-server zabbix-agent nginx php-fpm
```

检查一下端口，这个时候zabbix-server和fpm的端口没有，我们需要在后面的图形界面中配置

``` bash
netstat -untlp

0.0.0.0:80 <= # nginx的监听
0.0.0.0:10050 <= # zabbix-agent的监听
127.0.0.1:5432 <= # 数据库的监听
```

3.6. 图形界面配置

输入我们刚才在nginx中配置的`server_name`，http://zabbix.jormun.com，就可以看到欢迎界面了。

![image-20200818071805355](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818071805355.png)

如果是使用yum安装的，基本不会有啥问题

![image-20200818071855331](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818071855331.png)

这边只需要填写密码zabbix，然后会报错，报错后会让我们输入schema的名字

![image-20200818140016260](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818140016260.png)

host这里改成机器的名字，防止后面日志报错

![image-20200818140139023](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818140139023.png)

确认无误

![image-20200818140206637](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818140206637.png)

配置成功

![image-20200818140240830](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818140240830.png)

使用默认的用户名和密码登录 Admin/zabbix

![image-20200818140502450](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818140502450.png)

新版的zabbix就长这个样子了

![image-20200818140607229](/pages/keynotes/L4_architect/2_monitoring/pics/8_zabbix/image-20200818140607229.png)