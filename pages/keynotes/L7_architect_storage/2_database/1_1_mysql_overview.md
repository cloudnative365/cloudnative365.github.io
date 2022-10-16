---
title: mysql概述
keywords: keynotes, architect, storage, database, mysql_overview
permalink: keynotes_L7_architect_storage_2_database_1_1_mysql_overview.html
sidebar: keynotes_L7_architect_sidebar
typora-copy-images-to: ./pics/1_1_mysql_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

虽然社区纷纷转向其他开源数据库，但是也有很多不好迁移，不想迁移的软件依旧使用mysql，所以mysql咱们也稍微说一下。

mysql目前的版本基本都是5.7了，也有小概率碰上5.5或者5.6的。像centos7默认安装的mariadb，版本是5，其实对应mysql就是5.5。

一般来说，如果我们需要在机器上安装5.7，只能选择二进制包或者yum的方式来安装，并且记得不要使用系统自带的那个，安装完了还得yum remove掉。

## 2. 安装

### 2.1. yum安装

+ 通过yum安装

  ``` bash
  wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
  yum -y install mysql57-community-release-el7-10.noarch.rpm
  yum -y install mysql-community-server --nogpgcheck
  ```

+ 配置，默认数据库不启动，所以可以在启动前修改配置，比如数据的位置啥的

  ``` bash
  /etc/mysql.d/mysql.cnf
  ```

### 2.2. 简单配置

+ 找到密码

  mysql每个版本的密码方式都不一样，5.7的方式是在yum安装后，在/var/log/mysql/mysql.log里面找一下，最后会找到一个密码

+ 连接数据库

  ``` bash
  mysql -uroot -p
  ```

+ 简单初始化

  ``` bash
  # 5.7初始化数据库之后会默认密码是严格的策略，我们手动调整一下
  set global validate_password_policy=LOW;
  # 修改密码
  SET PASSWORD FOR 'root'@'localhost' = PASSWORD('Passw0rd');
  FLUSH PRIVILEGES;
  
  # 如果是其他的库可以这么弄
  USE mysql;
  
  UPDATE user SET password = PASSWORD('Passw0rd') WHERE user = 'root' AND host = 'localhost';
  ```

  
