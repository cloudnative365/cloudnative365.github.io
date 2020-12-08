---
title: systemd
keywords: keynotes, basic, linux, SSL
permalink: keynotes_L1_basic_3_linux_3_SSL.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/3_SSL
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

SSL全称是

## 2. SSL的原理



## 3. 签署SSL证书

### 3.1. 通过openssl签署

+ 安装openssl工具

  ``` bash
  yum -y install openssl 
  ```

+ 自建CA

  ``` bash
  # 生成CA私钥与CA证书
  openssl genrsa -out ca.pem 2048
  openssl req -new -x509 -sha256  -key ca.pem -out cacert.pem -days 3650 -subj /C=CN/ST=Beijing/L=Beijing/O=Jormun/OU=Jormun/CN=helloworld.jormun.com
  # -subj这个可以省略，但是会出现输入提示，一路留空即可，或者像下面这样
  openssl req -new -x509 -sha256  -key ca.pem -out cacert.pem -days 3650 -subj /C=/ST=/L=/O=/OU=/CN=helloworld.jormun.com
  # 我们这里只为了加密数据，所以全部留空
  openssl req -new -x509 -sha256  -key ca.pem -out cacert.pem -days 3650 -subj /C=/ST=/L=/O=/OU=/CN=/
  ```

+ 签署服务器证书

  ``` bash
  # 生成私钥
  openssl genrsa -out server.key 2048
  # 使用私钥生成证书，后面丢给CA签署，可以全部留空，但是如果是为某个域名签署证书，CN一定是域名
  openssl req -new -sha256 -key server.key -out server.csr -days 3650 -subj /C=/ST=/L=/O=/OU=/CN=helloworld.jormun.com
  # 我们这里同样留空
  openssl req -new -sha256 -key server.key -out server.csr -days 3650 -subj /C=/ST=/L=/O=/OU=/CN=/
  ```

+ CA为服务端证书签名

  ``` bash
  openssl x509 -req -sha256 -in server.csr -CA cacert.pem -CAkey ca.pem -CAcreateserial -out server.crt
  ```

+ 查看证书

  ``` bash
  openssl x509 -noout -text -in server.crt
  ```

### 3.2. 通过cfssl工具签署

- 下载cfssl工具

  ```
  curl -s -L -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  curl -s -L -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  chmod +x /usr/local/bin/{cfssl,cfssljson}
  ```

- 测试一下

  ```
  $ cfssl
  No command is given.
  Usage:
  Available commands:
  	serve
  	version
  	genkey
  	gencrl
  	ocsprefresh
  	selfsign
  	scan
  	print-defaults
  	revoke
  	bundle
  	sign
  	gencert
  	ocspdump
  	ocspserve
  	info
  	certinfo
  	ocspsign
  Top-level flags:
    -allow_verification_with_non_compliant_keys
      	Allow a SignatureVerifier to use keys which are technically non-compliant with RFC6962.
    -loglevel int
      	Log level (0 = DEBUG, 5 = FATAL) (default 1)
  ```

- 创建工作目录

  ```
  $ mkdir -p /etc/kubernetes/pki/etcd
  ```

- 创建CA配置文件（默认创建）

  ```
  cd /etc/kubernetes/pki/etcd
  cfssl print-defaults config > ca-config.json
  cfssl print-defaults csr > ca-csr.json
  ```

- 修改`ca-config.json`为

  ```
  {
      "signing": {
          "default": {
              "expiry": "43800h"
          },
          "profiles": {
              "server": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth"
                  ]
              },
              "client": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "client auth"
                  ]
              },
              "peer": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  ```

- 修改`ca-csr.json`为

  ```
  {
      "CN": "My own CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "US",
              "L": "CA",
              "O": "My Company Name",
              "ST": "San Francisco",
              "OU": "Org Unit 1",
              "OU": "Org Unit 2"
          }
      ]
  }
  ```

- 创建CA的证书

  ```
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
  ```

- 会得到三个文件

  ```
  ca-key.pem
  ca.csr
  ca.pem
  ```

- 生成服务器证书配置文件

  ```
  cfssl print-defaults csr > server.json
  ```

- 修改server.json中CN和host的部分

  ```
  {
      "CN": "etcd",
      "hosts": [
          "127.0.0.1",
          "10.0.1.204",
          "10.0.1.67",
          "10.0.1.236"
      ],
      "key": {
          "algo": "ecdsa",
          "size": 256
      },
      "names": [
          {
              "C": "US",
              "L": "CA",
              "ST": "San Francisco"
          }
      ]
  }
  ```

- 生成证书

  ```
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
  ```

- 同样是三个文件（这个是服务器启动时候的证书）

  ```
  server-key.pem
  server.csr
  server.pem
  ```

