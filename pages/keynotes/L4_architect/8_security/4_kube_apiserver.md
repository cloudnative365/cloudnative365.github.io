---
title: CIS
keywords: keynotes, architecture, security, CIS
permalink: keynotes_L4_architect_8_security_3_CIS.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/3_CIS
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ rancher bench
+ CIS benchmark
+ kube-bench

## 1. 集群安装

这里就不赘述了，前面我们介绍了很多的安装方式，还没有熟练掌握快速创建集群的同学请看我前面的文章。我这里使用rancher安装的方式，主要是为了比较一下Rancher中的benchmark功能。我们要准备三台机器，一台跑rancher，一台跑master，一台跑node。我们先在rancher机器上拉起来rancher的实例

``` bash
docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

然后通过在已知节点上运行命令的方式来部署集群

在master上

``` bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.5.7 --server https://10.114.6.168 --token htjgc5d9cmzhh8m6tzt6cl896pm7mwdcc4m5x2j6pm8xrxzkpkz7kq --ca-checksum ad921d6183bf76bd855f9e715660550ac92a96ed674667cd3b0dcc4ad09358ac --etcd --controlplane
```

在node上

``` bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.5.7 --server https://10.114.6.168 --token htjgc5d9cmzhh8m6tzt6cl896pm7mwdcc4m5x2j6pm8xrxzkpkz7kq --ca-checksum ad921d6183bf76bd855f9e715660550ac92a96ed674667cd3b0dcc4ad09358ac --worker
```

然后去喝一杯茶，回来就可以在界面上看到了

## 2. Rancher上的CIS

### 2.1. 使用集群管理上的CIS

我们可以在被管理的集群上找到这个功能

![image-20210331161548053](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331161548053.png)

打开扫描界面，创建一个job

![image-20210331164136369](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331164136369.png)

然后等一会就可以看到结果了

![image-20210331164442954](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331164442954.png)

### 2.2. 使用CIS的app来跑benchmark

我们的集群中还有一个叫local的集群，这是rancher默认创建的，如果我们相对他进行cis评测是不可以的，因为这个功能只适用于使用RKE安装的集群，但是我们可以使用应用商店中提供的CIS工具来扫描

![image-20210331161722878](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331161722878.png)

安装完成后就可以切换到benchmark的app中进行扫描了

![image-20210331163323808](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331163323808.png)

结果如下

![image-20210331163928647](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331163928647.png)

## 3. CIS benchmark工具

### 3.1. pdf文档

打开CIS网站，https://www.cisecurity.org/cis-benchmarks/，我们可以找到kubernetes的benchmark

![image-20210331165011469](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331165011469.png)

在这里下载

![image-20210331165139368](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331165139368.png)

下载后是一个pdf文件，可以看到每一项的详细说明

### 3.2. CIS评估的工具

在主界面找到tools

![image-20210331165316205](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331165316205.png)

进入CIS-CAT

![image-20210331165355145](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331165355145.png)

邮箱一定要填写正确

![image-20210331170949887](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331170949887.png)



在服务器上安装必要工具

``` bash
yum -y install wget java unzip
```

使用wget进行下载

``` bash
student@single: ̃$ wget https://learn.cisecurity.org/e/799323/l-799323-2019-11-15-3v7x/2mnnf/\79038343?h=xWidc0ywLqO6rH0WMcM1VXE9q1_WfdjCoVQ-tL2jXks # 下载
student@single: ̃$ mv 79038343\?h\=xWidc0ywLqO6rH0WMcM1VXE9q1_WfdjCoVQ-tL2jXks CIS-Cat.zip # 重命名
student@single: ̃$ unzip CIS-Cat.zip # 解压
```

执行命令

``` bash
student@single: ̃$ Assessor-CLI/Assessor-CLI.sh
```

出现帮助界面

![image-20210331171020856](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331171020856.png)

使用命令行来进行操作

``` bash
student@single: ̃/Assessor-CLI$ sudo bash Assessor-CLI.sh -i
```

选择5，官方只支持Ubuntu，但是rhel也没问题

![image-20210331171225560](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331171225560.png)

选择1

![image-20210331171308640](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331171308640.png)

最后会在report目录下生成html文件`./reports/single-CIS_Ubuntu_Linux_18.04_LTS_Benchmark-20201107T041401Z.html`

找个浏览器就可以打开了

![image-20210331171426238](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331171426238.png)

## 4. kube-bench

这个是社区提供的一个开源工具，支持大部分的linux发行版

``` bash
wget https://github.com/aquasecurity/kube-bench/releases/download/v0.5.0/kube-bench_0.5.0_linux_amd64.rpm
yum install kube-bench_0.5.0_linux_amd64.rpm
```

安装完成后会在/usr/local/bin/kube-bench下面找到，然后就可以使用`kube-bench master`来评测了

![image-20210331172743064](/pages/keynotes/L4_architect/8_security/pics/3_CIS/image-20210331172743064.png)