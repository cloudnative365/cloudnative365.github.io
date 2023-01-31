---
title: MinIO管理
keywords: keynotes, architecture, storage, object_storage, minio_management
permalink: keynotes_L7_architect_storage_1_storage_2_3_minio_management.html
sidebar: keynotes_L7_architect_storage_sidebar
typora-copy-images-to: ./pics/2_3_minio_management
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 命令工具mc

### 1.1. 创建第一个用户

集群刚创建完成的时候，mc并没有连接任何的集群

``` bash
# mc config host list
gcs  
  URL       : https://storage.googleapis.com
  AccessKey : YOUR-ACCESS-KEY-HERE
  SecretKey : YOUR-SECRET-KEY-HERE
  API       : S3v2
  Path      : dns

local
  URL       : http://localhost:9000
  AccessKey : 
  SecretKey : 
  API       : 
  Path      : auto

play 
  URL       : https://play.min.io
  AccessKey : Q3AM3UQ867SPQQA43P2F
  SecretKey : zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG
  API       : S3v4
  Path      : auto

s3   
  URL       : https://s3.amazonaws.com
  AccessKey : YOUR-ACCESS-KEY-HERE
  SecretKey : YOUR-SECRET-KEY-HERE
  API       : S3v4
  Path      : dns
```

我们先连接本地的集群

``` bash
mc config host add local http://localhost:9000 admin Passw0rd
```

再list就会发现已经配置好了

``` bash
# mc config host list
gcs  
  URL       : https://storage.googleapis.com
  AccessKey : YOUR-ACCESS-KEY-HERE
  SecretKey : YOUR-SECRET-KEY-HERE
  API       : S3v2
  Path      : dns

local
  URL       : http://localhost:9000
  AccessKey : admin
  SecretKey : Passw0rd
  API       : s3v4
  Path      : auto

play 
  URL       : https://play.min.io
  AccessKey : Q3AM3UQ867SPQQA43P2F
  SecretKey : zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG
  API       : S3v4
  Path      : auto

s3   
  URL       : https://s3.amazonaws.com
  AccessKey : YOUR-ACCESS-KEY-HERE
  SecretKey : YOUR-SECRET-KEY-HERE
  API       : S3v4
  Path      : dns
```

我们接着来创建一个policy，先来一个所有权限的，all.json

``` bash
{
  "Version": "2012-10-17",         
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
                "s3:ListAllMyBuckets",
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::*"
      ]
    }
  ]
}
```

把这个policy加到集群中，名字就叫all

``` bash
mc admin policy add local all all.json
```

创建用户

``` bash
mc admin user add local user1 password1
```

把all策略给user1

``` bash
mc admin policy set local all user=user1
```

