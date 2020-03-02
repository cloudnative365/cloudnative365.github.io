---
title: Linux软件包管理
keywords: keynotes, L1_basic, 1_Linux_basic, 2_linux_operation software_management
permalink: keynotes_L1_basic_1_Linux_basic_2_linux_operation_software_management.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/software_management
typora-root-url: ../../../../../../cloudnative365.github.io

---

## 1. 介绍

在Linux上安装软件是一件有趣的事情，但是有的时候确实令人头疼。Linux系统本身是没有提供任何包管理机制的，也就是说，内核中，只提供了必要的功能，这些包管理工具都是在一些Linux发行版中携带的工具。

想象一下曾经安装软件的`configure && make && make install`到`dpkg`，从`dpkg`到`apt-get`，软件安装管理有了非常大的进步，也让我们这些后来的攻城狮和程序猿感到庆幸，有这么方便的工具，真的是让我们的入门轻松许多。

但是，从管理的角度来说，我经常告诉我的部下，安装软件一定要用包管理器来安装，哪怕要手写spec文件，也要用rpm、deb的方式来安装。这样对于软件的后期管理是非常有效的，特别是在那个容器化并不普及的时代，一旦你编译安装后，使用`ldd`链接库文件的时候，很容易造成人为的事故，让接手这个项目的攻城狮一脚踏入泥坑。

从另一个角度来讲，如果每个程序都需要编译安装，那么我们为什么要花钱去买这些Linux发行商的服务呢？他们已经做好了非常多的框架，我们SRE的目标就是为了稳定不是么？

一个包应该包含操作系统的基本组件，共享库，可执行文件，服务，补丁还有文档之类的。

包管理器，除了安装，还可以升级或者删除。而包本身也是可以加上指纹的，方便开发人员和维护人员的管理。

Linux发行商利用包管理器迭代自己的软件，让自己的发行版更好用，方便用户管理系统。像红帽这样的发行商，还提供repo服务，红帽自己来维护软件包，而用户只需要买相应的服务，就可以享受红帽验证过的软件，提供系统的稳定性。

## 2. 包的底层架构

包的底层其实就是一个打包的文件，你可以理解成我们把某些文件和目录zip成了一个文件。还有一种包里的某个位置都会有一个清单文件，文件中写了这些文件是干什么的，怎样安装，怎样升级，怎样删除。这种包通常都是以.src.rpm命名的源码包，而编写清单文件SPEC是一项非常复杂的工作，Linux的发行商通常会有专门的部门负责这项工作。Redhat的RHCA考试中，有一项就是手动编写SPEC文件，打包，并且用kick-start的方式随着系统的启动安装在系统中。我们这里就不具体说了。

我们以rpm包为例

```
# 下载包，只下载不安装
$ yum install --downloadonly zip --downloaddir=.
```

![image-20200222175722350](/pages/keynotes/L1_basic/1_linux_basic/2_linux_operation/pics/software_management/image-20200222175722350.png)

```
# 我们看看包里面是什么
$ rpm2cpio zip-3.0-11.el7.x86_64.rpm | cpio -ivd
```

![image-20200222175817764](/pages/keynotes/L1_basic/1_linux_basic/2_linux_operation/pics/software_management/image-20200222175817764.png)

阿勒，感觉这就是我们的系统目录哇！

## 3. 打包平台和工具

对于不同平台，他们的打包工具，和包的格式也是不一样的

| 操作系统 | 衍生版 | 格式 | 工具 |
| :------: | :----: | :--: | :--: |
|  Debian  |        |      |      |
|  Ubuntu  |        |      |      |
|   RHEL   |        |      |      |
|          |        |      |      |

