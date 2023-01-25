---
title: postgresql@13高可用
keywords: keynotes, architect, storage, database, postgresql
permalink: keynotes_L7_architect_storage_2_database_2_3_postgresql_13_HA.html
sidebar: keynotes_L7_architect_sidebar
typora-copy-images-to: ./pics/2_3_postgresql_13_HA
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

截止到目前2022年12月04日，当前比较流行的pgsql的版本是13版本，目前使用这个版本的pgsql的数据库是AWX和Gitlab。这两个应用是使用operator来安装的，如果使用operator默认提供的pgsql，就是使用的13版本。为了让我们的系统production ready，所以我选择把两个数据库外置，并且使用高可用的架构。

一般来说，我们会选择主备的架构，但是这种架构会导致备节点的资源利用率非常低，所以这次我们就模拟一下环境有限的情况。利用两台机器装两个实例，对于A B机器来说，AWX实例在A是主，B是备，Gitlab实例在B是主，A是备的架构来安装这两套pgsql环境，实现资源的有效利用。

## 2. 环境

### 2.1. 信息

操作系统都是CentOS 7.9，selinux和防火墙记得都关闭

| IP地址       | VIP                                    | 机器名                  | 备注                             |
| ------------ | -------------------------------------- | ----------------------- | -------------------------------- |
| 10.39.64.246 | 10.39.64.230（AWX数据库连接这个IP）    | postgre-ittools-prod-01 | AWX 数据库主库，Gitlab数据库备库 |
| 10.39.64.227 | 10.39.64.231（Gitlab数据库连接这个IP） | postgre-ittools-prod-02 | Gitlab 数据库主库，AWX数据库备库 |

### 2.2. 安装postgresql

去[这个地址](https://www.postgresql.org/download/linux/redhat/)选择必要的版本，其实12-15的版本，yum安装的方式基本一样，就是版本有区别

``` bash
# Install the repository RPM:
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install PostgreSQL:
sudo yum install -y postgresql13-server
```

## 3. 安装单机版

### 3.1. pgsql

+ 官方的文档中直接使用下面的就启动了

  ``` bash
  # Optionally initialize the database and enable automatic start:
  sudo /usr/pgsql-13/bin/postgresql-13-setup initdb
  sudo systemctl enable postgresql-13
  sudo systemctl start postgresql-13
  ```

  但是要注意，这种方式启动的数据库，数据和备份文件都放在`/var/lib/pgsql/13`下，这个是因为`/usr/pgsql-13/bin/postgresql-13-setup`脚本中有一个变量`SERVICE_NAME`这个需要在initdb的时候指定，然后才会去`/usr/lib/systemd/system/${SERVICE_NAME}.service`下面找对应的启动脚本中指定的数据目录来初始化数据库目录

  ``` bash
  .
  .
  .
  # PGVERSION is the full package version, e.g., 13.0
  # Note: the specfile inserts the correct value during package build
  PGVERSION=13
  .
  .
  .
  # The second parameter is the new database version, i.e. $PGMAJORVERSION in this case.
  # Use  "postgresql-$PGMAJORVERSION" service, if not specified.
  SERVICE_NAME="$2"
  if [ x"$SERVICE_NAME" = x ]
  then
      SERVICE_NAME=postgresql-$PGMAJORVERSION
  fi
  .
  .
  .
  # Find the unit file for new version.
  if [ -f "/etc/systemd/system/${SERVICE_NAME}.service" ]
  then
      SERVICE_FILE="/etc/systemd/system/${SERVICE_NAME}.service"
  elif [ -f "/usr/lib/systemd/system/${SERVICE_NAME}.service" ]
  then
      SERVICE_FILE="/usr/lib/systemd/system/${SERVICE_NAME}.service"
  else
      echo "Could not find systemd unit file ${SERVICE_NAME}.service"
      exit 1
  fi
  ```

  所以，我们要使用我们自己定义的`.service`脚本来管理我们的数据库

+ 我们来复制一个awx的

  ``` bash
  cp /usr/lib/systemd/system/postgresql-13.service /usr/lib/systemd/system/postgresql-awx.service
  ```

+ 修改数据库目录

  ``` bash
  Environment=PGDATA=/app/pgsql/awx/data/
  ```

+ reload配置，创建目录，修改数据目录的权限

  ``` bash
  systemctl daemon-reload
  mkdir -p /app/pgsql/awx/{data,backup,archive}
  chown -R postgres:postgres /app/pgsql
  ```

+ 然后初始化，相关的数据文件就会出现在/app/pgsql/data下面了

  ``` bash
  /usr/pgsql-13/bin/postgresql-13-setup initdb postgresql-awx
  ```

+ 启动之前看一下端口，因为我们每台机器有两个端口对外服务，所以一定要注意端口冲突。

  配置文件位置/app/pgsql/awx/data/postgresql.conf

  ``` bash
  port = 5432
  ```

+ 启动

  ``` bash
  systemctl start postgresql-awx
  ```

### 3.2. keepalived

对于VIP，我们依然选择keepalived来做，其实最好的方式还是有负载均衡最好，通过另外一个位置来探测数据库的服务状态。或者使用pg pool来做前端会更加好。我们这里就使用最简单的ipvs方式来做高可用，毕竟没有这么高的业务要求。

