---
title: 管理ES
keywords: keynotes, senior, logging, manage_es
permalink: keynotes_L3_senior_6_logging_5_manage_es.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/5_manage_es
typora-root-url: ../../../../../cloudnative365.github.io
---

## 概述

对于管理员和用户来说，kibana是一个非常方便的管理工具，他涵盖了基本上所有的管理和使用功能。但是，作为一个专业的管理人员来说，还是使用命令行来管理系统更加专业。在最新的版本中，es提供了一个叫做elasticsearch-cli的东东，不过这个东西还在测试阶段，有兴趣的同学可以持续关注。我们一般是使用http请求的方式来操作elasticsearch的，也就是使用curl命令。

查询一般是这样的

``` bash
# 没有安全措施的
curl http://localhost:9200/_cluster/health
# 带了很多安全措施的
# -k 信任自签证书
# -u 服务器的用户名和密码
# --cert <certificate[:password]> ssl证书和密码
# --key 私有证书的位置
# --cacert ca证书
curl -k -u admin:admin --cert my.cert --key my.key --cacert my.pem https://localhost:9200/_cluster/health
```

前面我们学过了很多种的方式来安装es，这里我们使用mac的[brew](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/brew.html)来做一下，各位也可以使用其他的方式来做

``` bash
brew tap elastic/tap
brew install elastic/tap/elasticsearch-full
```

## 1. 集群级别的操作

``` bash
# 查询集群健康状态
GET _cluster/health

# 查询所有节点
GET _cat/nodes

# 查询索引及分片的分布
GET _cat/shards

# 查询指定索引分片的分布
GET _cat/shards/.kibana

# 查询所有插件
GET _cat/plugins
```

## 2. 索引级别的操作

### 2.1. 查询

``` bash
# 查询所有索引及容量
GET _cat/indices

# 查询索引映射结构
GET my_index/_mapping

# 查询所有索引映射结构    
GET _all

# 查询所有的相同前缀索引
GET my-*/_search

# 查询所有索引模板   
GET _template

# 查询具体索引模板
GET _template/my_template
```

### 2.2. 创建

``` bash
# 创建模板
PUT _template/test_hot_cold_template
{
    "index_patterns": "test_*",
    "settings": {
        "number_of_shards" : 3,
        "index.number_of_replicas": 1
     },
    "mappings": {
      "order": {
          "dynamic": false, 
          "properties": {
              "id": {"type": "long"},
              "name": {"type": "keyword"}
          }
      }    
    },
    "aliases": {
      "test": {}
    }      
}

# 根据模板创建索引并写入数据
POST test_hot_cold-2019-12-01/order
{
  "id":1,
  "name":"cwx"
}

# 直接创建索引
PUT my_index
{
  "mappings": {
    "doc": {
      "properties": {
        "name": {
          "type": "text"
        },
        "blob": {
          "type": "binary"
        }
      }
    }
  }
}
```

### 2.3. 删除

``` bash
DELETE my-index
```

### 2.4. 修改

``` bash
# 修改配置
PUT /test_hot_cold-2019-12-01/_settings 
{ 
  "settings": { 
    "index.routing.allocation.require.hotwarm_type": "cold"
  } 
}
```

## 3. DSL查询

``` bash
--1.查询所有
GET _search
{
  "query": {
    "match_all": {}
  }
}

--2.查询单个索引 的 固定属性
--精确匹配
GET _search
{
  "query": {
    "term": { "name" : "you" }
  }
}

--模糊匹配
GET _search
{
  "query": {
    "match": { "name" : "you" }
  }
}

--范围查找
GET _search
{
  "query": {
    "range": {
        "age":{ "gte" : 15 , "lte" : 25 }
    }
  }
}
　GET indexName/_search
　{
　　　"query": {
　　　　　"wildcard":{"relateId":"*672499460503*"}
　　　}
　}


--3.功能性查询
--过滤
GET my_index/_search
{
  "query": {
    "bool": {
      "filter": {
        "term":{"age":1095}
      }
    }
  }
}

--或 or
GET my - test / _search 
{
　　"query": {
　　　　"bool": {
　　　　　　"should": [{
　　　　　　　　"term": {
　　　　　　　　　　"name": "you"
　　　　　　　　}
　　　　　　　　}, {
　　　　　　　　"match": {
　　　　　　　　　　"age": 20
　　　　　　　　}
　　　　　　}]
　　　　}
　　}
}

--与 AND
GET my-test/_search
{
  "query": {
    "bool": {
      "must" : [{
        "match" : {
          "name" : "you"
        }
      },{
        "range":{
        "age":{
          "from" : 10 , "to" : 20
        } 
        }
      }]
    }
  }
}

--必须 =
GET my_index/_search
{
  "query": {
    "bool": {
      "must" : {
        "range" : {
          "age" : { "from" : 10, "to" : 20 }
        }
      }
    }
  }
}

--必须不 not
GET my_index/_search
{
  "query": {
    "bool": {
      "must_not" : {
        "term" : {
          "name" : "you"
        }
      }
    }
  }
}

--复合查找
GET my_index/_search 
{
"query": {
"bool": {
"should": [{
"match": {
"age": 40
}
}, 
{
"match": {
"age": 20
}
}],
"filter": {
  "match":{
    "name":"you"
  }
}
}
}
}

--4.索引迁移
--场景 从A索引 复制到B索引
POST _reindex
{
  "source": {
    "index": "my_index"
  },
  "dest": {
    "index": "new_my_index"
  }
}


--5.基于查询的删除
POST test-index/_delete_by_query
{
  "query":{
        "term": {
         "cameraId":"00000000002"
        }
  }

}

--查询
GET test-index/_search
{
  "query":{
        "term": {
         "cameraId":"00000000002"
        }
  }
}
```

时间范围查找

``` bash
GET order_stpprdinf_2019-12/_search
{
  "size":10,
  "query":{
    "range":{
      "order_time":{
        "gte":"2019-12-11T00:00:00+08:00",
        "lte":"2019-12-11T23:59:59+08:00"
      } 
    }
  }
}
```

按照条件删除

``` bash
GET order_stpprdinf_2019-12/_delete_by_query?conflicts=proceed
{
  "query":{
    "range":{
      "order_time":{
        "gte":"2019-12-11T00:00:00+08:00",
        "lte":"2019-12-11T23:59:59+08:00"
      } 
    }
  }
}
```

## 4. 概念

### 4.1. 集群Cluster

### 4.2. 节点Node

### 4.3. 索引Index

数据库

### 4.4. 类型type

表

### 4.5. 文档Document

行

### 4.6. 字段Field

