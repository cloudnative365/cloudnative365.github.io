---
title: 准备集群
keywords: keynotes, architecture, security, prepare_cluster
permalink: keynotes_L4_architect_8_security_2_prepare_cluster.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/2_prepare_cluster
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ 了解最新的工具
+ 从安全的角度来考虑整个集群的架构

## 1. 工具

社区的力量真的非常强大，每年都有非常多的开源工具问世，让我们目不暇接，这套课程里面在第二章（prepare to install）中对于一些热点的工具进行了介绍。

### 1.1.  镜像工具

+ [Trow](https://trow.io/)：开源镜像仓库，提供了镜像管理工具，还有很多正在plan的新功能
+ [Prisma Cloud](https://www.paloaltonetworks.com/prisma/cloud) ：适用于所有的环境，适合在混合云的环境部署
+ [NeuVector](https://neuvector.com/)：基于云与原生的，自带镜像安全和漏洞扫描的镜像平台
+ [Clair](https://github.com/quay/clair) ：开源的项目，可以在不启动容器的情况下进行漏洞扫描
+ [Aqua](https://www.aquasec.com/) ：是一个非常专业的，提供镜像全生命周期管理的解决方案，而扫描工具叫做[Trivy](https://github.com/aquasecurity/trivy).
+ [Notary](https://github.com/theupdateframework/notary) ：目前是CNCF的项目之一，通过类似于TLS的方式来对传输的数据进行校验，保证数据未被篡改过，他是基于[The Update Framework (TUF)](https://theupdateframework.io/)的
+ [Harbor](https://goharbor.io/)：大名鼎鼎的毕业项目，vmware的两个华裔工程师开发的，后来被人们传为第一个进入cncf毕业状态的中国项目

### 1.2. 管理工具

+ [Grafeas](https://grafeas.io/)：Grafeas 用于收集和汇总特定的元数据，为用户提供了一个标准化的方式，在即使由微服务器和容器带来的“软件供应链”缩短的情况下，仍能审核和管理他们的软件供应链

  ![image-20210331103839044](/pages/keynotes/L4_architect/8_security/pics/2_prepare/image-20210331103839044.png)

+ [gVisor](https://github.com/google/gvisor)：谷歌在2018年5月发布了一款新型的沙箱容器运行时 gVisor，号称能够为容器提供更安全的隔离，同时比 VM 更轻量。容器基于共享内核

  ![image-20210331104157974](/pages/keynotes/L4_architect/8_security/pics/2_prepare/image-20210331104157974.png)

+ [Kata](https://katacontainers.io/)：container runtime，kata containers是由OpenStack基金会管理，但独立于OpenStack项目之外的容器项目。kata containers整合了Intel的 Clear Containers 和 Hyper.sh 的 runV，能够支持不同平台的硬件 （x86-64，arm等），并符合OCI(Open Container Initiative)规范，同时还可以兼容k8s的 CRI（Container Runtime Interface）接口规范。项目包含几个配套组件，即Runtime，Agent， Proxy，Shim等。项目已于6月份release了1.0版本。

  ![image-20210331104404472](/pages/keynotes/L4_architect/8_security/pics/2_prepare/image-20210331104404472.png)

- 其他的容器运行时，比如：[PouchContainer](http://pouchcontainer.io/#/)，[Firecracker](https://github.com/firecracker-microvm/firecracker-containerd)，[UniK](https://github.com/solo-io/unik)

+ [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) ：为策略决策需求提供了一个统一的框架与服务。它将策略决策从软件业务逻辑中解耦剥离，将策略定义、决策过程抽象为通用模型，实现为一个通用策略引擎，可适用于广泛的业务场景。

  ![img](/pages/keynotes/L4_architect/8_security/pics/2_prepare/logo-white.png)

## 2. 准备安装

教程中在第三章（INSTALLING THE CLUSTER）中使用大量的文字从安全的角度出发，去构建我们的集群环境，而试验部分是使用脚本来部署我们的集群，我们这里把理论部分先给大家放出来

### 2.1. 选择集群的版本

在生产系统中，我们通常会选择某个软件供应商提供的，包装好的kubernetes发行版（除非是具有填坑能力的互联网公司），所以在这里，选择是从两个方向说的

+ 一个是发行版，比如：私有云上的，rancher，openshift，公有云上的，EKS，AKS。我们需要在用户的需求和成本（人力成本，财务成本）之间做一个平衡，并且在人员的职责方面做好分工，让供应商为我们更好的服务。
+ 另一个是发行版的版本，由于早期的k8s版本迭代非常快，版本间的差异非常明显，这就会导致我们构建在容器编排平台上的CI/CD流水线变的非常不稳定，即使小小的升级，也会让整个发布流程瘫痪。

### 2.2. 保护内核

内核是整个集群稳定的基础，内核的稳定性应该从三个方向考虑

+ 一是Linux的发行版，我们通常会在稳定且收费的Redhat系列，和开放且激进的Ubuntu系列做一个抉择。当然，也可以选择更加稳定的Unix平台，比如BSD系列。

+ 最小化配置，功能和模块的最小化配置和权限的最小化配置。

+ 及时修复漏洞，不要让我们的操作系统处在亚健康状态。我们可以选择收费的工具，他会有一些非常漂亮的报告，让领导赏心悦目，或者我们可以选择一些开源社区的工具，比如`git clone https://github.com/jondonas/linux-exploit-suggester-2.git`,然后`./linux-exploit-suggester-2.pl`

  ``` bash
    #############################
      Linux Exploit Suggester 2
    #############################
  
    Local Kernel: 3.10.0
    Searching 72 exploits...
  
    Possible Exploits
    [1] dirty_cow
        CVE-2016-5195
        Source: http://www.exploit-db.com/exploits/40616
    [2] exploit_x
        CVE-2018-14665
        Source: http://www.exploit-db.com/exploits/45697
    [3] pp_key
        CVE-2016-0728
        Source: http://www.exploit-db.com/exploits/39277
    [4] timeoutpwn
        CVE-2014-0038
        Source: http://www.exploit-db.com/exploits/31346
  ```

是不是有一种花了很多钱都喂狗了的感觉，你们花钱买的产品不过是把文字变成了好看的表格。这直接导致前端开发的人工资猛增，因为我们都是外貌协会。

### 2.3. 利用IGMP进行DoS

在内核3.2.1版本之前，我们有这个漏洞[CVE-2012-0207](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-0207)，这是一个非常严重的漏洞，一定要补上，因为在内网上IGMP基本是不会关闭的，很有可能由于一些网络故障，导致大量的IGMP泛滥，然后搞挂我们的服务器

### 2.4. 减少内核产生漏洞的可能性

+ 禁止模块动态加载：有一些需要运行起来需要运行为root的工具，比如：ps，top，我们需要禁止动态加载拥有root权限的模块，防止root权限泄露。可以通过echo 1给**kernel.modules_disabled**的方式来临时禁止，但是最好还是修改配置文件，否则，机器重启后，配置就会消失
+ 防止缓冲区溢出：学名叫Stack Layout Randomization，通过配置**kernel.randomize_va_space**选项来防止溢出，他有三个选项，我们应该基于禁止随机分配内存，也就是echo 0
  + 0: Disabled
  + 1: Randomized stack, VDSO (Virtual Dynamic Shared Object), shared memory addresses
  + 2: Randomized stack, VDSO, shared memory **and data** addresses.

### 2.5. 硬件安全功能

内核还支持一些硬件安全功能，如果硬件上的这些功能是开启的，内核也会感知到，比如

+ NX (No eXecute)：几乎所有的新CPU都支持这个，我们可以通过`grep nx /proc/cpuinfo`来查看这个功能，但是需要在BIOS里面开启/禁用这个功能，他主要是防止内存中存在可执行文件。同时，我们需要通过`echo 0 > /proc/sys/kernel/exec-shield`来开启，主要是防止内存中的数据被恶意篡改。

+ VT-d (virtualization)：对于虚拟化的支持，通过`grep vmx|svm /proc/cpuinfo`来查看。

+ TPM (Trusted Platform Module)：数据的安全，通过存储的校验机制来保证数据安全。

+ TXT (Trusted Execution Technology)：这个功能是用来对客户机的内存进行隔离的，这是一个硬件上的隔离，虽然软件层面也能做到，但是经常会发生过量分配的情况。这个功能和TPM是绑在一起的

+ SMAF (Secure Memory Allocation Feature)：
+ IMA(Integrity Measurement Architecture)：中文名字叫完整性度量架构。2004年，IBM在13th USENIXSecurity Symposium上发表文章《Design and Implementation of a TCG-based Integrity MeasurementArchitecture》，第一次提出了IMA架构。该架构通过在内核中进行patch，实现当应用程序运行、动态链接库加载、内核模块加载时，将用到的代码和关键数据（如配置文件和结构化数据）做一次度量，将度量结果扩展到PCR10，并创建与维护一个度量列表ML。当挑战者发起挑战时，将度量列表与TPM签名的PCR度量值发送给挑战者，以此来判断平台是否可信。这个实现的基础也是TPM，我们可以在/boot/config*中找到**CONFIG_IMA**配置，内核启动的时候需要添加参数**ima_tcb**和**ima=on**就能启用了，还有一个配置叫**CONFIG_DM_VERITY**这个是存储级别的度量，主要是针对存储块的。
+ (LSM)Linux Security Modules：这个应该不陌生了，经常被我们禁用的Selinux就是这个级别的安全防护，还有AppArmor，Smack，TOMOYO
+  (seccomp)Secure computing mode：这个是针对系统调用的安全防护，如果级别调整为最高的1级，只允许**read()**, **write()**, **exit()**, and **sigreturn()**这四个系统调用来运行。



