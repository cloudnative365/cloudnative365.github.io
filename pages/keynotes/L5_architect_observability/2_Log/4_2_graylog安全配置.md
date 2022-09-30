---
title: graylog安全配置
keywords: keynotes, architect, observability, log, gray, gray_security
permalink: keynotes_L5_architect_observability_2_log_4_1_graylog_security.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/4_2_graylog_security
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 引言

作为一款日志产品，就不得不在意他的安全相关的配置。主要有两个方面的安全，数据存储的安全和数据在传输过程中的安全。

## 2. 架构图



## 3. 单机版安装和配置

### 3.1. 环境

- 操作系统：Centos7
- Java > 1.8：我们使用OpenSearch自带的JRE
- OpenSearch 1.3.5：如果使用ES的话千万不要使用高于7.10.2的版本
- MongoDB 4.4：如果配置的EPEL源，可以直接yum install mongodb

### 3.2. 下载地址

+ OpenSearch: [下载页](https://opensearch.org/downloads.html)，[下载地址](https://artifacts.opensearch.org/releases/bundle/opensearch/1.3.5/opensearch-1.3.5-linux-x64.tar.gz)
+ MongoDB: [下载页](https://www.mongodb.com/try/download/community?tck=docs_server)，[Server](https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.16.tgz)，[Shell](https://fastdl.mongodb.org/linux/mongodb-shell-linux-x86_64-rhel70-4.4.16.tgz)

+ Graylog：[下载页](https://www.graylog.org/downloads-2)，[下载地址](https://downloads.graylog.org/releases/graylog/graylog-4.3.7.tgz)

### 3.3. 安装与配置

#### 3.3.1. 准备

+ 创建graylog用户，opensearch用户，mongo用户来分别运行他们的程序，同时制定他们的日志目录和数据目录，如果有条件，可以为三个应用挂载独立的分区

  ``` bash
  ```

  

#### 3.3.2. OpenSearch

+ 解压并创建目录

  ``` bash
  cd /data/graylog
  tar xf opensearch-1.3.5-linux-x64.tar.gz
  cd opensearch-1.3.5
  mkdir data
  ```

+ 修改配置文件/data/graylog/opensearch-1.3.5/config/opensearch.yml，只改这两个，其他默认就好

  ``` bash
  path.data: /data/graylog/opensearch-1.3.5/data
  path.logs: /data/graylog/opensearch-1.3.5/logs
  ```

+ 启动一下看看`/data/graylog/opensearch-1.3.5/opensearch-tar-install.sh`，如果没有报错再继续

+ 配置systemd：/etc/systemd/system/opensearch.service

  ``` bash
  [Unit]
  Description=OpenSearch
  Documentation=https://www.elastic.co
  Wants=network-online.target
  After=network-online.target
  
  [Service]
  Environment=OPENSEARCH_HOME=/data/graylog/opensearch-1.3.5
  Environment=PID_DIR=/data/graylog/opensearch-1.3.5
  WorkingDirectory=/data/graylog/opensearch-1.3.5
  User=graylog
  Group=graylog
  ExecStart=/data/graylog/opensearch-1.3.5/bin/opensearch -p ${PID_DIR}/opensearch.pid --quiet
  StandardOutput=journal
  StandardError=inherit
  LimitNOFILE=65535
  LimitNPROC=4096
  LimitAS=infinity
  LimitFSIZE=infinity
  TimeoutStopSec=0
  KillSignal=SIGTERM
  KillMode=process
  SendSIGKILL=no
  SuccessExitStatus=143
  TimeoutStartSec=75
  [Install]
  WantedBy=multi-user.target
  ```

+ 然后就可以启动了

  ``` bash
  systemctl start opensearch
  ```

+ 看到下面的内容就算成功了

  ``` bash
  curl -k -u admin:admin https://localhost:9200
  {
    "name" : "joomoo-dev-02",
    "cluster_name" : "opensearch",
    "cluster_uuid" : "nj9xq7icTN65XBeD2uQHJA",
    "version" : {
      "distribution" : "opensearch",
      "number" : "1.3.5",
      "build_type" : "tar",
      "build_hash" : "6cabc6aacc030bcaab30aeacf7529bbe7415c61b",
      "build_date" : "2022-08-30T17:40:02.136297Z",
      "build_snapshot" : false,
      "lucene_version" : "8.10.1",
      "minimum_wire_compatibility_version" : "6.8.0",
      "minimum_index_compatibility_version" : "6.0.0-beta1"
    },
    "tagline" : "The OpenSearch Project: https://opensearch.org/"
  }
  ```

#### 3.3.2. MongoDB

+ 安装依赖包

  ``` bash
  yum install -y libcurl openssl xz-libs
  ```

+ 解压并创建目录

  ``` bash
  mkdir /data/graylog/mongodb/logs
  mkdir /data/graylog/mongodb/data
  ```

+ 配置文件

  ``` bash
  wget https://raw.githubusercontent.com/mongodb/mongo/master/rpm/mongod.conf -P /data/graylog/mongodb/
  ```

+ 鉴于国内环境复杂，我们也可以直接使用下面的

  ``` bash
  cat > /data/graylog/mongodb/mongod.conf <<EOF
  systemLog:
    destination: file
    logAppend: true
    path: /data/graylog/mongodb/logs/mongod.log
    
  storage:
    dbPath: /data/graylog/mongodb/data
    journal:
      enabled: true
  
  processManagement:
    fork: true  # fork and run in background
    pidFilePath: /data/graylog/mongodb/mongod.pid
    timeZoneInfo: /usr/share/zoneinfo
  
  net:
    port: 27017
    bindIp: 127.0.0.1
  
  #security:
  EOF
  ```

+ 配置systemd

  ``` bash
  wget https://raw.githubusercontent.com/mongodb/mongo/master/rpm/mongod.service  -P /etc/systemd/system/
  ```

+ 同样可以直接抄这个

  ``` bash
  cat > /usr/lib/systemd/system/mongod.service <<'EOF'
  [Unit]
  Description=MongoDB Database Server
  Documentation=https://docs.mongodb.org/manual
  After=network-online.target
  Wants=network-online.target
  
  [Service]
  User=graylog
  Group=graylog
  Environment="OPTIONS=-f /data/graylog/mongodb/mongod.conf"
  EnvironmentFile=-/etc/sysconfig/mongod
  ExecStart=/data/graylog/mongodb/bin/mongod $OPTIONS
  ExecStartPre=/usr/bin/mkdir -p /var/run/mongodb
  ExecStartPre=/usr/bin/chown graylog:graylog /var/run/mongodb
  ExecStartPre=/usr/bin/chmod 0755 /var/run/mongodb
  PermissionsStartOnly=true
  PIDFile=/data/graylog/mongodb/mongod.pid
  Type=forking
  # file size
  LimitFSIZE=infinity
  # cpu time
  LimitCPU=infinity
  # virtual memory size
  LimitAS=infinity
  # open files
  LimitNOFILE=64000
  # processes/threads
  LimitNPROC=64000
  # locked memory
  LimitMEMLOCK=infinity
  # total threads (user+kernel)
  TasksMax=infinity
  TasksAccounting=false
  # Recommended limits for mongod as specified in
  # https://docs.mongodb.com/manual/reference/ulimit/#recommended-ulimit-settings
  
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

+ 可以启动了`systemctl start mongod`

+ 安装mongo-shell

  ``` bash
  tar xf mongodb-shell-linux-x86_64-rhel70-4.4.16.tgz
  mv mongodb-linux-x86_64-rhel70-4.4.16/bin/mongo /usr/sbin
  ```

+ 创建账号，mongo安装完成之后默认是不需要认证的，我们需要手动创建。直接运行mongo命令进入mongo-shell

  ``` bash
  > show dbs
  admin   0.000GB
  config  0.000GB
  local   0.000GB
  > use admin
  switched to db admin
  > db.createUser({user:"admin",pwd:"admin",roles:["root"]})
  Successfully added user: { "user" : "admin", "roles" : [ "root" ] }
  > db.auth("admin","admin")
  1
  > 
  ```

+ 修改启动命令，增加认证选项--auth

  ``` bash
  Environment="OPTIONS=-f /data/graylog/mongodb/mongod.conf --auth"
  ```

+ 再启动

  ``` bash
  systemctl daemon-reload
  systemctl restart mongod
  ```

+ 链接mongodb

  ``` bash
  mongo -u admin -p admin
  ```

+ 为mongodb创建TLS证书，参考https://blog.csdn.net/HappyLearnerL/article/details/124052816

+ 生成根证书

  ``` bash
  openssl genrsa -out ca.key 2048
  openssl req -new -key ca.key -out ca.csr
  ```

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/601d45357d6e417ab1f903d40eec7111.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGFwcHlMZWFybmVyTA==,size_16,color_FFFFFF,t_70,g_se,x_16)

  ``` bash
  openssl x509 -req -days 36500 -in ca.csr -signkey ca.key -out ca.pem
  ```

+ 生成服务器证书

  ``` bash
  2、生成服务器端PEM文件
  1、生成server私钥(server.key)（不加密）
  openssl genrsa -out server.key 2048
  2、生成server证书签名请求(server.csr)
  openssl req -new -key server.key -out server.csr
  输入内容和上图保持一致
  3、使用ca证书签署服务端csr以生成服务端证书(server.cert)
  openssl ca -days 36500 -in server.csr -out server.crt -cert ca.pem -keyfile ca.key
  # 如果报错，就直接创建一下文件
  touch /etc/pki/CA/index.txt
  touch /etc/pki/CA/serial && echo 01 > /etc/pki/CA/serial
  ```

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/51cb49d6aaf2404e86fa1bca015e49f0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGFwcHlMZWFybmVyTA==,size_20,color_FFFFFF,t_70,g_se,x_16)

  删掉server.crt中的certificate信息

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/959752763e254de4b590d71b95dc1a16.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGFwcHlMZWFybmVyTA==,size_20,color_FFFFFF,t_70,g_se,x_16)

  5、合并证书和私钥成PEM文件 ，构建命令如下：
  cat server.key server.crt > server.pem

+ 添加证书信任

  ``` bash
  cp root-ca.pem /etc/pki/ca-trust/source/anchors/
  update-ca-trust
  ```

  

#### 3.3.4. Graylog

+ 解压

  ``` bash
  tar xf graylog-4.3.7.tgz
  ```

+ 生成密码

  ``` bash
  # 如果没有安装pwgen命令，需要使用epel源，然后yum install pwgen
  # 生成password_secret密码
  pwgen -N 1 -s 96
  tWmmJmUpdefwLAO1xFmVTdeRM8IeDX29gwmIOktctkr3s8vkPtA2PTB1fPoZzzL2XxEk0rZRPRXi4bUYLpFFlaVrC8FFZdH8
  # 生成root_password_sha2，web界面登录的密码
  echo -n"Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
  Enter Password: 
  admin
  8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
  ```

+ 配置文件

  ``` bash
  mkdir etc
  cp graylog.conf.example server.conf/server.conf
  ```

+ 修改server.conf

  ``` bash
  http_bind_address = 10.39.64.182:9000
  
  password_secret = tWmmJmUpdefwLAO1xFmVTdeRM8IeDX29gwmIOktctkr3s8vkPtA2PTB1fPoZzzL2XxEk0rZRPRXi4bUYLpFFlaVrC8FFZdH8
  root_password_sha2 = 8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
  
  elasticsearch_hosts = https://admin:admin@127.0.0.1:9200
  elasticsearch_discovery_default_scheme = https
  
  bin_dir = /data/graylog/graylog-4.3.7/bin
  data_dir = /data/graylog/graylog-4.3.7/data
  
  mongodb_uri = mongodb://admin:admin@localhost:27017/graylog?authSource=admin&tls=true&tlsInsecure=true
  ```

+ 启动一下

  ``` bash
  bin/graylogctl start
  ```
  
+ 创建SSL证书

  ``` bash
  openssl genrsa -des3 -passout pass:x -out ssl.pass.key 2048
  openssl rsa -passin pass:x -in ssl.pass.key -out ssl.key
  openssl req -new -key ssl.key -out ssl.csr
  openssl x509 -req -days 3650 -in ssl.csr -signkey ssl.key -out ssl.crt
  ```
  
+ 配置https

  ``` bash
  http_enable_tls = true
  http_tls_cert_file = /data/graylog/graylog-4.3.7/etc/certs/ssl.crt
  http_tls_key_file = /data/graylog/graylog-4.3.7/etc/certs/ssl.key
  ```
  
  
