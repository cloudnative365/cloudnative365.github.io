---
title: database_exporter
keywords: keynotes, architect, monitoring, database_exporter
permalink: keynotes_L4_architect_2_monitoring_16_database_exporter.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/16_database_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

数据库的exporter级别的监控基本都是通过创建可读账号，然后登录数据库去拿一些数据来完成监控的。我们常见的数据库在[官方的网站](https://prometheus.io/docs/instrumenting/exporters/#databases)上记录的export如下

![image-20200918095642019](/pages/keynotes/L4_architect/2_monitoring/pics/16_database_exporter/image-20200918095642019.png)

我们就挑几个在企业中经常用到的来说一说

## 2. oracle_exporter

先来说说企业中用的最多的一种关系型数据库oracle，我们使用的是[这个](https://github.com/iamseth/oracledb_exporter)。

### 2.1. Docker

+ 创建一个oracle实验环境

  ``` bash
  docker run -d --name oracle -p 1521:1521 wnameless/oracle-xe-11g-r2:18.04-apex
  ```

+ 创建oracle_exporter

  + 一般的镜像

    ``` bash
    docker run -d --name oracledb_exporter --link=oracle -p 9161:9161 -e DATA_SOURCE_NAME=system/oracle@oracle/xe iamseth/oracledb_exporter
    ```

  + alpine的镜像，exporter的版本大于0.2.1才支持

    ``` bash
    docker run -d --name oracledb_exporter --link=oracle -p 9161:9161 -e DATA_SOURCE_NAME=system/oracle@oracle/xe iamseth/oracledb_exporter:alpine
    ```

### 2.2. 二进制包

+ 创建一个oracle实验环境

  ``` bash
  docker run -d --name oracle -p 1521:1521 wnameless/oracle-xe-11g-r2:18.04-apex
  ```

+ 安装oracle客户端，如果使用docker镜像，镜像中会包含oracle的依赖库，我们使用二进制方式的话就需要手动配置

  ``` bash
  rpm -ivh https://download.oracle.com/otn_software/linux/instantclient/19800/oracle-instantclient19.8-basiclite-19.8.0.0.0-1.x86_64.rpm
  ```

+ 配置oracle环境变量，根据下载的版本不一样，文件夹可能会有小变动，但是基本都是在/usr/lib/oracle下，如果是二进制包的，就去找解压地址

  ``` bash
  ORACLE_HOME=/usr/lib/oracle/19.8/client64/
  LD_LIBRARY_PATH=/usr/lib/oracle/19.8/client64/lib
  ```

+ 下载二进制包并解压

+ 配置各种参数

  + 直接连接oracle listener的SID

    ``` bash
    export DATA_SOURCE_NAME=system/password@oracle-sid
    ```

  + 或者直接使用URL

    ``` bash
    export DATA_SOURCE_NAME=user/password@//myhost:1521/service
    ```

+ 启动exporter

  ``` bash
  export DATA_SOURCE_NAME=system/oracle@localhost:1521/xe; /path/to/binary/oracledb_exporter --log.level error --web.listen-address 0.0.0.0:9161
  ```

+ 创建systemd的service文件

  ``` bash
  [Unit]
  Description=Service for oracle telemetry client
  After=network.target
  [Service]
  Type=oneshot
  #User=oracledb_exporter
  ExecStart=/path/of/the/oracledb_exporter --log.level error --web.listen-address 0.0.0.0:9161
  [Install]
  WantedBy=multi-user.target
  ```

### 2.3. 自定义metrics

这里说的自定义metrics是说去数据库中，通过select语句查询之后，把返回的结果作为metrics暴露出来，主要用来监控标准模板中不提供的一些参数，比如：业务指标，或者标准模板中不提供的一些功能，比如：查询最耗时的SQL语句等等。

在exporter中有一个启动参数叫`--custom.metrics string`，这个参数用来指定自定义文件的位置，文件需要是一个TOML格式的文件，比如c_metrics.toml

``` bash
[[metric]]
context = "test"
request = "SELECT 1 as value_1, 2 as value_2 FROM DUAL"
metricsdesc = { value_1 = "Simple example returning always 1.", value_2 = "Same but returning always 2." }
```

我们启动的时候就需要使用

``` bash
export DATA_SOURCE_NAME=system/oracle@localhost:1521/xe; /path/to/binary/oracledb_exporter --log.level error --web.listen-address 0.0.0.0:9161 --custom.metrics /path/to/c_metrics.toml
```

启动之后，我们可以看到这样的结果

``` bash
...
# HELP oracledb_test_value_1 Simple example returning always 1.
# TYPE oracledb_test_value_1 gauge
oracledb_test_value_1 1
# HELP oracledb_test_value_2 Same but returning always 2.
# TYPE oracledb_test_value_2 gauge
oracledb_test_value_2 2
...
```

这就是我们刚才自定义的数据了。其实我们如果使用二进制包，他的包里面是有一个例子的，叫`default-metrics.toml`。我们默认的数据就是使用这个来获取数据的。

## 3. mysql_exporter

我们先启动一个测试用的mysql

``` bash
docker run --name mysql-server -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=mysql docker.io/mysql
```

登录到容器当中

``` bash
#看下容器的ID
~]# docker ps
CONTAINER ID        IMAGE                                   COMMAND                  CREATED             STATUS              PORTS                                      NAMES
7aa221d1c9bb        docker.io/mysql                         "docker-entrypoint..."   7 minutes ago       Up 7 minutes        0.0.0.0:3306->3306/tcp, 33060/tcp          mysql-server
# 登录过去
docker exec -it 7aa221d1c9bb sh
```

配置一下访问权限

``` bash
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

mysql> select host,user from user;
+-----------+------------------+
| host      | user             |
+-----------+------------------+
| %         | root             |
| localhost | mysql.infoschema |
| localhost | mysql.session    |
| localhost | mysql.sys        |
| localhost | root             |
+-----------+------------------+
5 rows in set (0.00 sec)

mysql> CREATE USER 'readonly'@'%' IDENTIFIED BY 'readonly';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT SELECT ON *.* TO 'readonly'@'%';
Query OK, 0 rows affected (0.06 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

mysql> select host,user from user;
+-----------+------------------+
| host      | user             |
+-----------+------------------+
| %         | readonly         |
| %         | root             |
| localhost | mysql.infoschema |
| localhost | mysql.session    |
| localhost | mysql.sys        |
| localhost | root             |
+-----------+------------------+
6 rows in set (0.00 sec)

```

### 3.1. 使用docker的方式运行

+ 下载镜像

  ``` bash
  docker pull prom/mysqld-exporter
  ```

  

+ 看一下mysql的网络IP，默认是连接到了bridge上

  ``` bash
  # docker network ls
  NETWORK ID          NAME                DRIVER              SCOPE
  9386ac1896df        bridge              bridge              local
  a16288c35865        host                host                local
  2a88d9207c1a        none                null                local
  
  # docker inspect 9386ac1896df|grep default
              "Driver": "default",
              "com.docker.network.bridge.default_bridge": "true",
  
  ```

+ 启动exporter

  ``` bash
  docker run -d \
    -p 9104:9104 \
    --network 9386ac1896df  \
    -e DATA_SOURCE_NAME="readonly:readonly@(127.0.0.1:3306)/" \
    prom/mysqld-exporter
  ```

+ 看指标

  ``` bash
  curl localhost:9104/metrics
  ```

## 4. mssql_exporter



## 5. pgsql_exporter



## 6. elasticsearch_exporter



## 7. redis_exporter