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
- MongoDB 4.4：文档上写着4.0，但是我一直在用4.4，也没啥问题

### 3.2. 下载地址

+ OpenSearch: [下载页](https://opensearch.org/downloads.html)，[下载地址](https://artifacts.opensearch.org/releases/bundle/opensearch/1.3.5/opensearch-1.3.5-linux-x64.tar.gz)
+ MongoDB: [下载页](https://www.mongodb.com/try/download/community?tck=docs_server)，[Server](https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.16.tgz)，[Shell](https://fastdl.mongodb.org/linux/mongodb-shell-linux-x86_64-rhel70-4.4.16.tgz)

+ Graylog：[下载页](https://www.graylog.org/downloads-2)，[下载地址](https://downloads.graylog.org/releases/graylog/graylog-4.3.7.tgz)

### 3.3. 安装与配置

#### 3.3.1. 准备

+ 创建graylog用户，opensearch用户，mongo用户来分别运行他们的程序，同时制定他们的日志目录和数据目录，如果有条件，可以为三个应用挂载独立的分区

  ``` bash
  mkdir /data # 在根目录创建独立的文件夹
  mkdir /data/src # 用来存放二进制文件
  mkdir -p /data/opensearch/{data,logs,run}
  mkdir -p /data/mongodb/{data,logs,run}
  mkdir -p /data/graylog/{data,logs,run}
  ```
  
+ 下载二进制包

  ``` bash
  cd /data/src
  wget https://artifacts.opensearch.org/releases/bundle/opensearch/1.3.5/opensearch-1.3.5-linux-x64.tar.gz
  wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.16.tgz
  wget https://fastdl.mongodb.org/linux/mongodb-shell-linux-x86_64-rhel70-4.4.16.tgz
  wget https://downloads.graylog.org/releases/graylog/graylog-4.3.7.tgz
  ```

+ 创建用户

  ``` bash
  useradd opensearch
  useradd mongod
  useradd graylog
  ```

+ 解压

  ``` bash
  tar xf /data/src/opensearch-1.3.5-linux-x64.tar.gz -C /data/opensearch/
  tar xf /data/src/mongodb-linux-x86_64-rhel70-4.4.16.tgz -C /data/mongodb/
  tar xf /data/src/graylog-4.3.7.tgz -C /data/graylog/
  
  # mongo的客户端命令可以直接丢进$PATH
  cd /data/src
  tar xf mongodb-shell-linux-x86_64-rhel70-4.4.16.tgz
  mv mongodb-linux-x86_64-rhel70-4.4.16/bin/mongo /usr/local/sbin
  ```

+ 给与对应目录权限

  ``` bash
  chown -R opensearch:opensearch /data/opensearch
  chown -R mongod:mongod /data/mongodb
  chown -R graylog:graylog /data/graylog
  ```

  

+ 配置JAVA

  ``` bash
  cat >/etc/profile.d/java.sh << EOF
  export JAVA_HOME=/data/opensearch/opensearch-1.3.5/jdk
  export CLASSPATH=$JAVA_HOME/lib:$CLASSPATH
  export PATH=$JAVA_HOME/bin:$PATH
  EOF
  ```

+ 读取一下环境变量`source /etc/profile`

  ``` bash
  $ java -version
  openjdk version "11.0.16" 2022-07-19
  OpenJDK Runtime Environment Temurin-11.0.16+8 (build 11.0.16+8)
  OpenJDK 64-Bit Server VM Temurin-11.0.16+8 (build 11.0.16+8, mixed mode)
  ```

#### 3.3.2. OpenSearch

+ 修改配置文件/data/opensearch/opensearch-1.3.5/config/opensearch.yml，只改这两个，其他默认就好

  ``` bash
  path.data: /data/opensearch/data
  path.logs: /data/opensearch/logs
  ```

+ 启动一下看看，如果没有报错再继续

  ``` bash
  su - opensearch -c "/data/opensearch/opensearch-1.3.5/opensearch-tar-install.sh"
  ```

+ 配置systemd：/etc/systemd/system/opensearch.service

  ``` bash
  [Unit]
  Description=OpenSearch
  Documentation=https://www.elastic.co
  Wants=network-online.target
  After=network-online.target
  
  [Service]
  Environment=OPENSEARCH_HOME=/data/opensearch/opensearch-1.3.5
  Environment=PID_DIR=/data/opensearch/opensearch-1.3.5/run
  WorkingDirectory=/data/opensearch/opensearch-1.3.5
  ExecStartPre=/usr/bin/mkdir -p /data/opensearch/run
  ExecStartPre=/usr/bin/chown -R opensearch:opensearch /data/opensearch/
  ExecStartPre=/usr/bin/chmod 0755 /data/opensearch/run
  User=opensearch
  Group=opensearch
  ExecStart=/data/opensearch/opensearch-1.3.5/bin/opensearch -p ${PID_DIR}/opensearch.pid --quiet
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
    "name" : "ecs-graylog",
    "cluster_name" : "opensearch",
    "cluster_uuid" : "jr9zyDw7RjKKgSCdD0Ym4g",
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

+ 根据[官方文档](https://docs.graylog.org/docs/securing-opensearch)的说法，然后我们需要先关闭安全插件，让Graylog先正确读取一次内容，后面再打开安全配置，修改opensearch.yml

  ``` bash
  plugins.security.disabled: true
  ```

+ 这个时候用户名和密码还是需要的

  ``` bash
  curl -u admin:admin http://localhost:9200
  {
    "name" : "ecs-graylog",
    "cluster_name" : "opensearch",
    "cluster_uuid" : "jr9zyDw7RjKKgSCdD0Ym4g",
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

+ 配置文件

  ``` bash
  wget https://raw.githubusercontent.com/mongodb/mongo/master/rpm/mongod.conf -P /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/
  ```

+ 鉴于国内环境复杂，我们也可以直接使用下面的

  ``` bash
  cat > /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/mongod.conf <<EOF
  systemLog:
    destination: file
    logAppend: true
    path: /data/mongodb/logs/mongod.log
    
  storage:
    dbPath: /data/mongodb/data
    journal:
      enabled: true
  
  processManagement:
    fork: true  # fork and run in background
    pidFilePath: /data/mongodb/run/mongod.pid
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
  cat > /etc/systemd/system/mongod.service <<'EOF'
  [Unit]
  Description=MongoDB Database Server
  Documentation=https://docs.mongodb.org/manual
  After=network-online.target
  Wants=network-online.target
  
  [Service]
  User=mongod
  Group=mongod
  Environment="OPTIONS=-f /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/mongod.conf"
  EnvironmentFile=-/etc/sysconfig/mongod
  ExecStart=/data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/bin/mongod $OPTIONS
  ExecStartPre=/usr/bin/mkdir -p /data/mongodb/run
  ExecStartPre=/usr/bin/chown mongod:mongod /data/mongodb/
  ExecStartPre=/usr/bin/chmod 0755 /data/mongodb/run
  PermissionsStartOnly=true
  PIDFile=/data/mongodb/run/mongod.pid
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
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

+ 可以启动了`systemctl start mongod`

+ 创建账号，mongo安装完成之后默认是不需要认证的，我们需要手动创建。直接运行mongo命令进入mongo-shell

  ``` bash
  > show dbs
  admin   0.000GB
  config  0.000GB
  local   0.000GB
  > db.createUser({user:"admin",pwd:"admin",roles:["root"]}) # 创建一个管理账户
  Successfully added user: { "user" : "admin", "roles" : [ "root" ] }
  > use graylog
  switched to db use graylog
  > db.createUser({user:"graylog",pwd:"graylog",roles:[{role:"dbOwner",db:"graylog"}]})
  Successfully added user: {
  	"user" : "graylog",
  	"roles" : [
  		{
  			"role" : "userAdminAnyDatabase",
  			"db" : "admin"
  		}
  	]
  }
  > exit
  bye
  
  $ mongo -u graylog -p graylog --authenticationDatabase graylog
  > show dbs;
  graylog  0.002GB
  ```

+ 修改启动命令，增加认证选项--auth，修改/etc/systemd/system/mongod.service

  ``` bash
  Environment="OPTIONS=-f /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/mongod.conf --auth"
  ```

+ 再启动

  ``` bash
  systemctl daemon-reload
  systemctl restart mongod
  ```

+ 为mongodb创建TLS证书，首先创建目录

  ``` bash
  mkdir /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/certs
  cd /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/certs
  ```

+ 生成根证书

  ``` bash
  $ openssl genrsa -out ca.key 2048
  $ openssl req -new -key ca.key -out ca.csr # 只填写State or Province Name和Common Name
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [XX]:
  State or Province Name (full name) []:beijing
  Locality Name (eg, city) [Default City]:
  Organization Name (eg, company) [Default Company Ltd]:
  Organizational Unit Name (eg, section) []:
  Common Name (eg, your name or your server's hostname) []:graylog
  Email Address []:
  
  Please enter the following 'extra' attributes
  to be sent with your certificate request
  A challenge password []:
  An optional company name []:
  $ openssl x509 -req -days 36500 -in ca.csr -signkey ca.key -out ca.pem
  ```

  

+ 生成服务器证书

  ``` bash
  $ openssl genrsa -out server.key 2048
  $ openssl req -new -key server.key -out server.csr # 只填写State or Province Name和Common Name
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [XX]:
  State or Province Name (full name) []:beijing
  Locality Name (eg, city) [Default City]:
  Organization Name (eg, company) [Default Company Ltd]:
  Organizational Unit Name (eg, section) []:
  Common Name (eg, your name or your server's hostname) []:graylog
  Email Address []:
  
  Please enter the following 'extra' attributes
  to be sent with your certificate request
  A challenge password []:
  An optional company name []:
  $ openssl ca -days 36500 -in server.csr -out server.crt -cert ca.pem -keyfile ca.key
  # 如果报错，就直接创建一下文件
  touch /etc/pki/CA/index.txt
  touch /etc/pki/CA/serial && echo 01 > /etc/pki/CA/serial
  # 如果出现线面的结果就是正确的，记得都选y
  Using configuration from /etc/pki/tls/openssl.cnf
  Check that the request matches the signature
  Signature ok
  Certificate Details:
          Serial Number: 1 (0x1)
          Validity
              Not Before: Oct  1 12:44:59 2022 GMT
              Not After : Sep  7 12:44:59 2122 GMT
          Subject:
              countryName               = XX
              stateOrProvinceName       = beijing
              organizationName          = Default Company Ltd
              commonName                = graylog
          X509v3 extensions:
              X509v3 Basic Constraints:
                  CA:FALSE
              Netscape Comment:
                  OpenSSL Generated Certificate
              X509v3 Subject Key Identifier:
                  69:FD:83:9E:94:49:75:99:60:05:0F:C2:71:24:EB:0F:88:37:5E:53
              X509v3 Authority Key Identifier:
                  DirName:/C=XX/ST=beijing/L=Default City/O=Default Company Ltd/CN=graylog
                  serial:89:58:2F:A0:5F:FA:9D:21
  
  Certificate is to be certified until Sep  7 12:44:59 2122 GMT (36500 days)
  Sign the certificate? [y/n]:y
  
  
  1 out of 1 certificate requests certified, commit? [y/n]y
  Write out database with 1 new entries
  Data Base Updated
  ```

+ 删掉server.crt中的certificate信息，只留下BEGIN到END之间的内容

  ``` bash
  -----BEGIN CERTIFICATE-----
  .
  .
  .
  -----END CERTIFICATE-----
  ```

+ 合并证书和私钥成PEM文件 ，构建命令如下

  ``` bash
  cat server.key server.crt > server.pem
  ```

+ 添加证书信任，这个是为了让graylog服务器信任mongodb，所以如果graylog的server在其他机器，就需要在其他机器做这个操作

  ``` bash
  cp ca.pem /etc/pki/ca-trust/source/anchors/
  update-ca-trust
  ```

+ 修改配置文件mongod.conf，加上配置文件的信息

  ``` bash
  net:
    port: 27017
    bindIp: 127.0.0.1
    tls:
      mode: requireTLS
      certificateKeyFile: /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/certs/server.pem
      CAFile: /data/mongodb/mongodb-linux-x86_64-rhel70-4.4.16/certs/ca.pem
      allowInvalidHostnames: true
      allowInvalidCertificates: true
      allowConnectionsWithoutCertificates: true
  ```

+ 重启mongodb

  ``` bash
  systemctl restart mongod
  ```

#### 3.3.4. Graylog

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
  cd /data/graylog/graylog-4.3.7
  mkdir etc
  cp graylog.conf.example etc/server.conf
  ```

+ 修改server.conf

  ``` bash
  http_bind_address = 0.0.0.0:9000
  
  password_secret = tWmmJmUpdefwLAO1xFmVTdeRM8IeDX29gwmIOktctkr3s8vkPtA2PTB1fPoZzzL2XxEk0rZRPRXi4bUYLpFFlaVrC8FFZdH8
  root_password_sha2 = 8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
  
  elasticsearch_hosts = http://admin:admin@127.0.0.1:9200
  
  node_id_file = /data/graylog/run/node-id
  bin_dir = /data/graylog/graylog-4.3.7/bin
  data_dir = /data/graylog/graylog-4.3.7/data
  
  mongodb_uri = mongodb://admin:admin@localhost:27017/graylog?authSource=admin&tls=true&tlsInsecure=true
  ```

+ 修改启动脚本，/data/graylog/graylog-4.3.7/bin/graylogctl

  ``` bash
  GRAYLOG_CONF=${GRAYLOG_CONF:=/data/graylog/graylog-4.3.7/etc/server.conf}
  GRAYLOG_PID=${GRAYLOG_PID:=/data/graylog/run/graylog.pid}
  LOG_FILE=${LOG_FILE:=/data/graylog/logs/graylog-server.log}
  ```

+ 启动一下

  ``` bash
  /data/graylog/graylog-4.3.7/bin start
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

+ 重启

  ``` bash
  /data/graylog/graylog-4.3.7/bin restart
  ```

#### 3.3.5. 配置Opensearch

+ 启用安全插件

  ``` bash
  plugins.security.disabled: true
  ```

+ 修改graylog使用https连接opensearch，修改/data/graylog/graylog-4.3.7/etc/server.conf

  ``` bash
  elasticsearch_hosts = https://admin:admin@127.0.0.1:9200
  ```

+ 重启Graylog

  ``` bash
  /data/graylog/graylog-4.3.7/bin restart
  ```

  

