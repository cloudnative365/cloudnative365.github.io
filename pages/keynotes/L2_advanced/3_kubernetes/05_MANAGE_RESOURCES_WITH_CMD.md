---
title: 使用命令行管理资源
keywords: keynotes, L2_advanced, kubernetes, CKA, MANAGE_RESOURCES_WITH_CMD
permalink: keynotes_L2_advanced_3_kubernetes_5_MANAGE_RESOURCES_WITH_CMD.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/5_MANAGE_RESOURCES_WITH_CMD
typora-root-url: ../../../../../cloudnative365.github.io
---

## 学习目标

+ CKA中考试会涉及到的资源类型
+ 使用命令行工具对pod、deployment和service进行增删改查

## 1. kubernetes常用的资源
kubernetes的常用资源也就是我们常说的kubernetes概念，我们昨天用比较通俗的比喻讲解了一下，不知道大家有没有从感性上认识这些概念。其实我们的概念都来自于官方网站的“concepts”，链接在[这里](https://kubernetes.io/docs/concepts/)

![file](/pages/keynotes/L2_advanced/3_kubernetes/pics/5_MANAGE_RESOURCES_WITH_CMD/222759a2aa562dd54ecaf01588255575.png)

我们创建资源的方式一般有两种，一种是通过命令行创建，另外一种方式是通过manifest文件来创建。通过命令创建的资源选项并不是非常全面，想要对每一个参数都进行配置的话，就需要使用yaml文件来定义我们的资源。

我们这套课程在所有资源中挑选常用的资源并分成了三部分来讲解：
1. CKA考试会考到的资源概念
2. CKA考试不会考到，但是常用的资源
3. kubernetes自身并不提供此类功能，而是通过插件方式，接入第三方软件才能实现的资源

我们这里先来说第一类，CKA考试会考到的资源
1.  Namespaces
2.  Labels和Selectors
3.  Field Selectors
4.  Nodes
5.  Pods
6.  Deployment
7.  DaemonSet
8.  Service
9.  Volumes
10.  Persistent Volumes
11.  ConfigMap
12.  Secrets

我们的上半部分课程会着重这些概念进行讲解

## 2. 使用命令管理资源
我们通常说的对于一个已经运行的系统的管理操作不外乎增删改查，如果要使用命令进行增删改查的话就需要用到下面的命令
+ 增：kubectl run/expose/create
+ 删：kubectl delete
+ 改：kubectl edit
+ 查：kubectl get

### 2.1. 查看资源
查看资源，我们使用kubectl命令
``` bash
kubectl get
[(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...]
(TYPE[.VERSION][.GROUP] [NAME | -l label] | TYPE[.VERSION][.GROUP]/NAME ...) [flags] [options]
```
格式我们目前先简单理解为
``` bash
kubectl get TYPE/NAME
```
比如，我们查看所有pod，就使用
``` bash
kubectl get pods
```
而查看特定pod，就使用
``` bash
kubectl get pod/pod-name
```
意思就是查看名字为nginx的pod，比如
``` bash
$ kubectl get pod/nginx-6db489d4b7-bs72b
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6db489d4b7-bs72b   1/1     Running   0          2m21s
```
通常我们在查看特定的pod的所有信息时候，都会加上-o yaml选项来查看他的清单文件
``` bash
$ kubectl get pod/nginx-6db489d4b7-bs72b -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-04-30T15:19:58Z"
  generateName: nginx-6db489d4b7-
  labels:
    pod-template-hash: 6db489d4b7
    run: nginx
  name: nginx-6db489d4b7-bs72b
  namespace: default
  ownerReferences:
	.
	.
	.
```

### 2.2. 删除资源
和查看差不多
``` bash
kubectl delete ([-f FILENAME] | [-k DIRECTORY] | TYPE [(NAME | -l label | --all)]) [options]
```
我们目前就使用最简单的
``` bash
kubectl delete TYPE/NAME
```
比如，删除刚才的pod
``` bash
$ kubectl delete pod/nginx-6db489d4b7-bs72b
pod "nginx-6db489d4b7-bs72b" deleted
```

### 2.3. 修改资源
这里说的修改，是说在资源创建以后，对正在运行状态的资源进行修改，但是，并不是每个选项都可以修改。
比如：我们可以修改deployment的副本数，使用
``` bash
kubectl edit TYPE/NAME
```
也就是
``` bash
kubectl edit deployment/nginx
```
kubernetes会调用默认的编辑器对资源进行修改，修改完成之后保存退出就可以了，比如，我们默认的编辑器是vim，我们把spec下面的replicas从1改到3
![file](/pages/keynotes/L2_advanced/3_kubernetes/pics/5_MANAGE_RESOURCES_WITH_CMD/222222f0b03c69fc5c78701588261425.png)
保存退出之后就会看到deployment变成了三份

``` bash
$ kubectl edit deployment/nginx
deployment.apps/nginx edited
$ kubectl get deployment/nginx
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/3     3            1           23m
```

### 2.4. 创建资源
kubectl run命令专门用于创建pod资源（1.17及其以下的版本，run可以创建deployment）
kubectl expose命令用于创建service
kubectl create可以创建下面的资源
![file](/pages/keynotes/L2_advanced/3_kubernetes/pics/5_MANAGE_RESOURCES_WITH_CMD/2221de9314f65fc62893401588261615.png)

我们这里就创建一个deployment，让他暴露给service，在1.18版本中创建deployment的时候指定副本数也会有警告，也就是--replicas也要被废弃了
``` bash
$ kubectl create deployment nginx-deploy --image=nginx
deployment.apps/nginx-deploy created
$ kubectl expose deployment nginx-deploy --port=80
service/nginx-deploy exposed
$ kubectl get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP   52m
nginx-deploy   ClusterIP   10.105.31.197   <none>        80/TCP    31s
```

这个时候，如果我们在master的机器上执行`curl 10.105.31.197`就能访问服务了，但是如果我们在外部的机器就不行了，这是由service的暴露方式决定的，我们讲到service时候再详细说

