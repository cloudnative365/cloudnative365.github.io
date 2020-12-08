---
title: 实验环境
keywords: keynotes, basic, labs, docker
permalink: keynotes_L2_basic_4_labs_3_docker.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/03_docker
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 课程目标

使用docker搭建实验环境

## 2. 简介

如果我们只有一台机器，但是想基于centos或者ubuntu或者任何linux发行版来做多主机的实验的话，其实也是可以的。但是，这种方式有局限性，那就是不能做虚拟化的实验。在容器中跑程序是没问题的，但是容器中再run一个docker或者kvm基本是不太可能的。

这种方式非常轻量，适合在mac笔记本上来做虚拟机的实验。

## 3. Docker desktop

Docker在mac/windows上叫做Docker desktop，MAC点击这里[下载](https://desktop.docker.com/mac/stable/Docker.dmg)，Windows点击这里[下载](https://desktop.docker.com/win/stable/Docker%20Desktop%20Installer.exe)。其实就是Docker的有UI的版本，可以通过点点点的方式来配置docker。但是，作为专业人员还是选择命令行来管理比较合适。

### 3.1. Docker

system

## 4. 借助docker做一台虚拟机

### 4.1. 启动基础镜像

在安装好docker之后就可以使用命令行了

``` bash
docker run -d -p 10122:22 --name node1 --privileged=true centos /sbin/init
```

这里简单介绍一下几个选项

-d：使用守护进程的方式运行在后台

-p: 端口映射，把本机的10122端口映射到容器的22端口，从而实现ssh登录

--name: 容器的名字

--privileged: 特权模式，一定要加，要不没办法作为root来执行命令

centos: 镜像的名字，默认是最新版，也就是centos8.2

/sbin/init: 进入系统后启动的命令，我们的systemd都是借助这个命令来运行的

### 4.2. 配置基础镜像

基础镜像启动之后，可以使用docker ps来查看镜像的信息

``` bash
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                   NAMES
eed6caa15378        centos              "/sbin/init"        7 days ago          Up 2 seconds        0.0.0.0:10122->22/tcp   node1
```

然后进入镜像进行配置

``` bash
docker exec -it eed6caa15378 sh
```

安装必要的包，主要是ssh服务

``` bash
yum install openssh-server passwd net-tools
```

配置服务

``` bash
systemctl start sshd
systemctl enable sshd
# 给root账户一个密码
passwd root
# 看一下IP地址
ifconfig eth0
# 退出
exit
```

把刚才的镜像保存一下

``` bash
docker commit eed6caa15378 node-dev
```

### 4.3. 启动新的容器（服务器）

先看一下我们刚才的做好的镜像

``` bash
docker image ls node-dev
```

使用下面的命令启动新服务器

``` bash
docker run -d -p 10122:22 --name node3 --ip 172.17.0.13 --privileged=true node-dev /sbin/init
```

-p xxx:22的映射记得改一下，--name和--ip也要换一下，不要重复

### 4.4. 通过ssh连接容器（服务器）

我们可以像连接其他服务器一样连接我们的容器，但是需要加上-p选项指定端口，而且IP地址要指定本地

``` bash
ssh -p 10122 root@localhost
```

