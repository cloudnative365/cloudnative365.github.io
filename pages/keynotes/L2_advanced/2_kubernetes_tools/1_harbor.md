---
title: 从docker镜像到高可用仓库
keywords: keynotes, advanced, kubernetes_tools, harbor
permalink: keynotes_L2_advanced_2_kubernetes_tools_1_harbor.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/1_harbor
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 镜像基础知识

我们在使用Docker部署应用的时候，需要加载Docker镜像，我们可以简单的认为镜像就是我们以前主机虚拟化中的镜像，比如vmware默认的VMDK文件，或者是KVM的QCOW2文件。Docker镜像含有启动容器所需要的文件系统及其内容，一个容器启动之后，如果没有做任何更改，那么我们看到的，就是镜像所提供的内容。因此，其作用在于创建并启动docker容器。

### 1.1. 镜像采用分层构建机制

最底层是bootfs，上面是rootfs

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/pics/1_harbor/20200420234212601.png)

- bootfs：里面是lxc以及aufs，btrfs或者overlay2这样的文件系统，来确保能够引导并且启动一个用户空间，这个内核所实现用于系统引导的文件系统，包括bootloader和kernel，容器启动完成后会被卸载以节省内存资源，实际上我们是看不见的
- rootfs：位于bootfs之上，表现为docker容器的根文件系统。传统模式中，系统启动的时候，内核挂载rootfs时会首先将其挂载为只读模式，完整性自检完成后，将其重新挂载为读写模式。docker中，rootfs由内核挂载为“只读”模式，而后通过“联合挂载”技术额外挂载一个“可写”层；

### 1.2. Docker镜像层

- 位于下层的镜像称为父镜像（parent image），最底层的称为基础镜像（base image）
- 最上层为“可读写”层，其下的均为“只读”层

![img](/pages/keynotes/L2_advanced/2_kubernetes_tools/pics/1_harbor/2F143098875606357619.JPG)

如果我们想要创建一个Apache的镜像，在底层我们就要在Debian操作系统的基础上创建一个编辑器emacs，也可以是vim，在此基础上再添加一个httpd，每一个功能都是一个独立的层，而最下面的层，在容器启动之后，一旦引导rootfs成功，就被卸载和移除，是说从内存中移除。Base image一般是用来供给基础操作系统，比如/bin，/sbin这样的目录。当然都是最小化的，其他的就需要额外安装。而后，如果我们要安装vim，vim是独立的层，再安装apache，apache也是独立的层，然后，这三层都是只读的层，在只读层上面加一个可写的层。如果我们删除这个容器，这个容器的可写层也会被删除。

### 1.3. Aufs

这种分层的架构是需要特殊的文件系统来支持的，那么最早的就是Aufs。Aufs全称叫advanced multi-layered unification filesystem，高级多层统一文件系统。用于为Linux文件系统实现“联合挂载”。aufs是之前的UnionFS的重新实现，2006年由Junjiro Okajima开发。Docker最初使用aufs作为容器文件系统层，他目前仍作为存储后端之一来支持。aufs的竞争产品是overlayfs，后者自从3.18版本开始被合并到Linux内核。docker在分层镜像，除了aufs，docker还支持btrfs，devicemapper和vfs等。在ubuntu系统下，docker默认ubuntu的aufs，在centos7上使用的是devicemapper。

```据说UnionFS代码很烂，然后被重写成AUFS之后依然很烂，想ext4这种系统代码只有5000行，而aufs据说有30000行，这些代码想整合进内核实在是太大了。AUFS想要合并进内核代码的时候，Linus认为这是烂代码。被拒绝了多次，直到最后放弃。所以我们想要使用AUFS的话，需要额外安装。而像红帽这种系统，他是追求稳定的，这种系统他是不会使用的。但是ubuntu稍微有点激进，他是最早一批把AUFS打包进内核的。红帽使用的dm，他的多路径也是用的dm。这都是早期，那么现在新版的docker已经默认是overlayfs了。overlayfs是一种抽象的二级文件系统，他后端是构建在其他的文件系统之上的，比如xfs```

## 2. 镜像仓库

### 2.1. Docker hub

Docker hub有下面几个主要的功能

- 镜像仓库：可以搜索和拉取社区和官方仓库，如果有注册账号，就可以管理，推送，从私有仓库拉取镜像
- 自动构建：可以从源码自动构建新的镜像
- Webhooks：自动构建的一个功能，允许我们成功推送之后触发构建动作
- 组织：创建工作组来管理访问镜像仓库的权限
- 可以和github和bitbucket集成：允许把hub和docker镜像构建成工作流

### 2.2. 工作流

我们可以把dockerfile存放在github仓库中，这个仓库可以与dockerhub的仓库建立关系，而dockerhub的仓库可以持续监控github的仓库，我们一旦发现dockerfile或者发现dockerfile有改动，我们就把他拖过来，然后根据这个构建docker image放在仓库当中。

### 2.3. 从远程的docker仓库拉取镜像

我们使用docker pull

``` bash
docker pull <registry>[:<port>]/[<namespace>/]<name>:<tag>
```

- registry：就是提供docker镜像服务的，监听在tcp端口上的地址（默认是：5000），比如quay.io
- namespace和name定义了一个由namespace控制的registry

比如我们使用

``` bash
docker pull quay.io/coreos/flannel:v0.10.0-amd64
```

