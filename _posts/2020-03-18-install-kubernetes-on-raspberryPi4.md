---
title:  在树莓派4上安装kubernetes集群
categories: 树莓派
permalink: install_kubernetes_on_reapberryPi4.html
tags: [news]
typora-copy-images-to: ./pics/2020-03-18-install-kubernetes-on-raspberryPi4
typora-root-url: ../cloudnative365.github.io
---

## 1. 简介

使用树莓派创建kubernetes集群的灵感来自于CNCF大会上一些极客们在展示自己公司的产品时候，把产品搭建在装有kubernetes集群的树莓派上，然后带到会场去做现场演示。

![IMG_3885](/../_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/IMG_3885.png)

上图中使用的是最新的树莓派4，我也模仿这个做了一个（请忽略凌乱的布线）

![img](/_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/IMG_4212.png)

## 2. 架构图

这个就是一个标准的3master-2node+外部LB的架构，前面带屏幕的机器是负载均衡器和跳板机。

![ha-master-gce](/../_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/ha-master-gce.png)

## 3. 硬件清单

+ 服务器（树莓派）

| 机器型号  | IP地址                                                 | 主机名    | 内存 | 组件                    |
| --------- | ------------------------------------------------------ | --------- | ---- | ----------------------- |
| 树莓派4B  | 10.1.1.11                                              | master1   | 4G   | kube-panel              |
| 树莓派4B  | 10.1.1.12                                              | master2   | 4G   | kube-master/etcd        |
| 树莓派4B  | 10.1.1.13                                              | master3   | 4G   | kube-master/etcd        |
| 树莓派4B  | 10.1.1.14                                              | worker1   | 4G   | kube-worker/etcd        |
| 树莓派4B  | 10.1.1.15                                              | master1   | 4G   | kube-worker/etcd        |
| 树莓派3B+ | 10.1.1.10<br />192.168.18.17（连接无线网DHCP到的地址） | mgtserver | 4G   | dhcp，loadbalancer，dns |

![IMG_4218](/../_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/IMG_4218.JPG)

![20190626094948762](/../_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/20190626094948762.png)

+ 路由器：TP-LINK8口路由器

  ![IMG_4220](/../_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/IMG_4220.JPG)

+ 电源适配器：小米6口充电器，官方要求树莓派4B+的电流是3A，但是目前大部分USB充电都是最大2.4A，而且多口同时供电会产生电流不到2A的情况，我实了很多种方法，最后还是小米的这个最稳定，虽然到不了3A，但是机器可以正常运转，我还没有压力测试，所以不知道满负荷的情况下会不会断电，但是这个已经是家庭级别最稳定的方式了。

  ![IMG_4219](/../_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/IMG_4219.JPG)

+ 其他：6类千兆线若干，USB转typeA线1根（树莓派3B+），USB转typeC线若干（树莓派4B），一般来说typeC的线都可以达到5A，只要不是质量太差的。一个tf卡读卡器，其他转换器（我用mac系统就需要有typeC扩展USB的转换器）

## 4. 烧录镜像

### 4.1. 操作系统

开始我在树莓派4B上使用的是树莓派系统，但是鉴于他和kubernetes的兼容性，我选择的是ubuntu18.04，在我写这篇文章的时候，ubuntu已经有了19.01，但是19不是LTS版，可以算是测试版，我还是选择的1804，下载地址[点这里](https://ubuntu.com/download/raspberry-pi)

树莓派3B+我安装了一个屏幕，但是需要安装淘宝店铺官方提供的镜像才能驱动这块屏幕，我看了下，官方的镜像用的是树莓派的镜像，所以这个系统没有选择，只有树莓派

### 4.2. 烧录系统到SD卡

+ 官方教程在这里，windows[点这里](https://ubuntu.com/tutorials/create-an-ubuntu-image-for-a-raspberry-pi-on-windows#1-overview)，ubuntu[点这里](https://ubuntu.com/tutorials/create-an-ubuntu-image-for-a-raspberry-pi-on-ubuntu#1-overview)，MacOS[点这里](https://ubuntu.com/tutorials/create-an-ubuntu-image-for-a-raspberry-pi-on-macos#1-overview)

+ 我的是MacOS系统，把tf卡插进转换器，再插到机器上

  ![](/_posts/pics/2020-03-18-install-kubernetes-on-raspberryPi4/IMG_4216.png)

+ 查看磁盘

``` bash
diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                 Apple_APFS Container disk1         250.7 GB   disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +250.7 GB   disk1
                                 Physical Store disk0s2
   1:                APFS Volume Macintosh HD - 数据     197.3 GB   disk1s1
   2:                APFS Volume Preboot                 82.8 MB    disk1s2
   3:                APFS Volume Recovery                526.6 MB   disk1s3
   4:                APFS Volume VM                      1.1 GB     disk1s4
   5:                APFS Volume Macintosh HD            11.0 GB    disk1s5

/dev/disk2 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *127.9 GB   disk2
   1:             Windows_FAT_32 boot                    268.4 MB   disk2s1
   2:                      Linux                         127.6 GB   disk2s2
```

/dev/disk2是我们刚才插进去的tf卡

+ 取消挂载

  ``` bash
  diskutil unmountDisk /dev/disk2
  Unmount of all volumes on disk2 was successful
  ```

+ 把镜像写入tf卡

  ``` bash
  sudo sh -c 'gunzip -c ~/Downloads/ubuntu-18.04.4-preinstalled-server-arm64+raspi3.img.xz | sudo dd of=/dev/disk2 bs=32m'
  ```

## 5. 在管理机上配置服务

在管理机上配置各种服务，比如dhcp，dns和负载均衡，模拟真实环境中的dhcp服务器，名称解析服务器和F5防火墙。

### 5.1. 连接mgtserver

我用的是微型屏幕，插上鼠标键盘就有图形界面了。但是不是每个人都买了屏幕，我来说一下一般的方法。

+ 获取IP地址

  把tf卡插进卡槽，找一根网线，把机器接入路由器（一般的家庭路由器都有dhcp功能）。同时，把笔记本也接入同一个路由器。

  使用ifconfig命令查看自己机器获取到的IP地址

  ``` bash
  en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
  	options=400<CHANNEL_IO>
  	ether a4:83:e7:89:fb:62
  	inet6 fe80::75:39db:87f:6452%en0 prefixlen 64 secured scopeid 0x6
  	inet 192.168.18.12 netmask 0xffffff00 broadcast 192.168.18.255
  	inet6 2408:8210:2425:5ee0:1414:8ebc:bf37:2eb5 prefixlen 64 autoconf secured
  	inet6 2408:8210:2425:5ee0:a40c:3b21:5f39:2ab prefixlen 64 autoconf temporary
  	nd6 options=201<PERFORMNUD,DAD>
  	media: autoselect
  	status: active
  ```

  

  使用nmap命令查看在同一个网段中其他机器的地址(没有nmap请brew install)，找到地址是mgtserver (192.168.18.17)

  ``` bash
  $ nmap -sP 192.168.18.0/24
  Nmap scan report for 192.168.18.1 (192.168.18.1)
  Host is up (0.062s latency).
  Nmap scan report for miwifi-r1cm (192.168.18.2)
  Host is up (0.066s latency).
  Nmap scan report for zhimi-humidifier-v1_miio94054944 (192.168.18.3)
  Host is up (0.15s latency).
  Nmap scan report for zhimi-airpurifier-v3_miio437238 (192.168.18.4)
  Host is up (0.15s latency).
  Nmap scan report for lumi-gateway-v3_miio45176213 (192.168.18.5)
  Host is up (0.16s latency).
  Nmap scan report for katsutekiiphone (192.168.18.7)
  Host is up (0.064s latency).
  Nmap scan report for jormunsmbp2019 (192.168.18.12)
  Host is up (0.0023s latency).
  Nmap scan report for mgtserver (192.168.18.17)
  Host is up (0.20s latency).
  Nmap done: 256 IP addresses (8 hosts up) scanned in 14.79 second
  ```

+ ssh上去就好了，默认用户名/密码是pi/raspberry，我这个是已经改完的，可能和新机器略有区别

  ``` bash
  ssh pi@192.168.18.17
  pi@192.168.18.17's password:
  Linux mgtserver 4.19.97-v7+ #1294 SMP Thu Jan 30 13:15:58 GMT 2020 armv7l
  
  The programs included with the Debian GNU/Linux system are free software;
  the exact distribution terms for each program are described in the
  individual files in /usr/share/doc/*/copyright.
  
  Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
  permitted by applicable law.
  Last login: Thu Mar 19 10:48:27 2020 from 192.168.18.12
  
  SSH is enabled and the default password for the 'pi' user has not been changed.
  This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.
  
  pi@mgtserver:~ $
  ```

### 5.2. 配置无线网络

树莓派是自带无线模块的，所以外网的链接，我们使用无线链接，修改`/etc/wpa_supplicant/wpa_supplicant.conf`文件(可以配置多个无线网)

``` bash
update_config=1
country=CN

network={
	ssid="raspberrypi"
	psk="xxxxxxxx"
	key_mgmt=WPA-PSK
}

network={
	ssid="CU_hehe"
	psk="xxxxxxxx"
	key_mgmt=WPA-PSK
}
```

改完了重启一下，然后拔掉网线，测试一下是否可以自动获取地址，依然使用nmap查看新获取到的地址

### 5.3. 配置私有网络

成功之后，我们就可以把树莓派的网口连接到我们的8口路由器上了。然后修改以太网口的地址为静态地址`/etc/dhcpcd.conf`

``` bash
interface enxb827eb835a18
static ip_address=10.1.1.10/24
static router=10.1.1.1
```

### 5.4. 配置树莓派为dhcp服务器

+ 安装dhcp服务器的包

  ``` bash
  apt-get install isc-dhcp-server
  ```

+ 编辑配置文件`/etc/default/isc-dhcp-server`，选择需要开启dhcp服务器的网卡，我们选择刚才的以太网口enxb827eb835a18

  ```bash
  INTERFACES="enxb827eb835a18"
  ```

+ 配置dhcp服务，`/etc/dhcp/dhcpd.conf`

  ``` bash
  subnet 10.1.1.0 netmask 255.255.255.0 {
  option routers 10.1.1.1;
  option subnet-mask 255.255.255.0;
  range dynamic-bootp 10.1.1.100 10.1.1.200;
  }
  ```

+ 启动服务

  ```  bash
  systemctl start isc-dhcp-server
  ```

+ 注意：不管是实验还是生产环境，dhcp服务都不建议开机启动，因为和网络相关的服务器，系统默认的timeout时间都比较长，也就是说，如果机器意外重启，那么他在启动的时候，如果遇到问题，会不停的重试，导致启动时间非常的长，如果有数据不一致的情况，系统为了保护数据，会让自己进入安全模式，这样的话，我们是无法通过ssh连上去的，必须要到机房才可以

## 6. 启动树莓派集群

把其他的树莓派集群都通过以太网口连接到8口路由器上，同样的方式获取和配置IP地址，这里就不赘述了。



## 7. 安装kubernetes集群

参考我以前的[文章](https://cloudnative365.github.io/keynotes_L4_architect_1_solutions_design_1_HA_2_k8s_cluster_kubeadm_apt.html)

{% include links.html %}
