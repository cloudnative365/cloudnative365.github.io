---
title: 认识kubernetes
keywords: keynotes, L2_advanced, kubernetes, CKA, BASICS_OF_KUBERNETES
permalink: keynotes_L2_advanced_1_kubernetes_3_kubernetes_2_BASICS_OF_KUBERNETES.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/2_BASICS_OF_KUBERNETES
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 课程目标

- 容器编排
- kubernetes的特性

## 2. 容器编排

### 2.1. 传统编排工具

+ 手动部署：比如我们部署tomcat，我们就需要下载war包-->停止tomcat-->备份原来的解压出来的文件-->把war包放在指定位置-->解压war包-->修改war包的位置文件，把链接测试数据库的用户名和密码改成生产的-->启动应用-->监控日志，访问应用看看是不是正常-->反馈给开发

+ shell编程：编写脚本，在机器上执行

+ 自动化框架（ansible，puppet，saltstack，saltstack）：比如ansible，通过我们定义的playbook，还有一些参数的定义，逻辑的处理，完成对多种应用程序的组合部署

但是有了Docker之后，我们的应用程序可以容器化，各种应用程序都被封装在了容器中执行。想象一下，我们最早在使用Ansible编排应用程序的过程当中，我们经常会发现对象在不停的变化，这个时候，我们的ansible或者puppet这种传统的应用程序或者工具已经不再适合这种场景了。因为，容器化所提供的接口，与我们早期传统形式上的应用程序的访问控制和管理是有所不同的。所以，在docker时代就开始呼唤新式的，面向容器化应用的编排工具。

### 2.2. 早期容器编排工具

+ 第一款是Docker自身提供的，叫docker compose，

  ![image-20200420125836228](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/image-20200420125836228-7909048.png)这个工具更适合于单机编排。他只能面向一个docker hosts来进行编排操作。或者是更加适用于这种情景。Docker就不得不开始提供一款工具能够将多主机的Docker host，也就是很多的docker主机，能有一定程度上的调度效果的，至少是面向集群的编排效果。那于是Docker后来提供了另外一款工具，叫docker swarm，而这个Docker swarm就是一个能够将多个docker整合成为由一个统一的管理工具管理之下的集群的工具，我们将多个host所提供的计算资源整合在一起，整合成为一个资源池，随后docker compose在编排时，只需要针对Docker swarm整合出来的资源池进行编排就可以了，而不用关心底层是什么样的主机，或者有几个docker主机。当然，docker swarm也是一个应用程序，甚至可以理解成为Docker自身的应用程序，那么，这里面的主机，怎样成为docker主机的呢？一个主机，要想加入docker swarm资源池，成为docker swarm的成员，他自己首先是一个docker 主机，那么docker 如何安装上去，他怎样加入docker swarm集群，成为docker swarm资源池的一员呢？我们就需要另外的工具，叫docker machine，能够将一个主机迅速初始化，满足加入docker swarm的集群的先决条件，从而能够成为docker swarm集群的一份子，这样一个预置处理工具，这就是人们当年口中的docker编排三剑客。我们知道，此前docker compose 面向单个的docker host，照样可以工作，当时的人们就是这么编排工具。

  Docker三剑客：docker compose，docker hosts，docker swarm

![image-20200307231554527](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/image-20200307231554527.png)

我们可以看到，人们不再是面向单个主机上的应用程序去编排，就好像ansible，他本来就可以面向多台主机上的应用程序去编排，那么，容器化的编排工具也需要面向多级执行编排，而且更重要的是，我们将来要运行容器时，在不同主机上编排时候，可以会用到同一个应用程序，这同一个应用程序，如果编排在不同主机上不相冲突等各种任务，都需要有Docker编排工具，或者容器编排工具来实现。

+ Mesos：那么，除了docker自身所提供的工具，我们还有第二类工具，mesos。

  ![image-20200420125854597](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/image-20200420125854597-7909094.png)

  目前是apache旗下的产品。他原来是由Twitter受到google borg系统的启发，想开发一款资源管理的软件。整个时候，他们发现加州大学博克利分校AMPLab正在开发一款叫做mesos的软件，后来他们就把负责开发这款软件的人挖到了Twitter，然后开始开发和部署mesos。但是mesos是一个面向IDC的OS，IDC的操作系统能够把IDC当中所有的硬件所提供的计算资源统一的调度和分配，但是他所面向的上层接口，提供的不是容器运行的接口，而只是资源分配工具，并非能够运行和托管容器的，所以在此之上，他必须还要提供一个能够允许容器编排的框架，叫marathon。

  ![img](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/mp59172955_1455676809045_11.png)

  

尽管mesos+marthon的解决方案最终并没有成为容器编排的主流，但是毕竟术业有专攻，mesos的用户之地不在于编排，而在于资源整合，所以，mesos并没有像docker三剑客一样逐渐走向没落，而是在另外的领域牢牢占领一席之地，这就是大数据领域，比较知名的就是spark+mesos的解决方案。这里就不多说了，有兴趣的同学可以深入学习一下

![img](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/mesos.jpg)

### 2.3. 高端编排工具

而这场容器编排之争的最终获胜者，就是我们要学习的kubernetes了，Kubernetes在这个领域中据说已经牢牢占据了90%以上的份额。

k8s出现的时间并不是很长，只有区区几年的时间，kubernetes中文的意思叫做舵手，而他的标志就是一个舵，或者叫飞行员。他主要是由Google的几位工程师创立的，大概是在2014年对外首次宣布，一直到今天为止，也不过是短短6年的时间，kubernetes的开发，深受google内部的borg系统的影响。

![The Kubernetes Lineage](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/cfhkkbh42zcw-Kuberneteslineage.jpg)

Borg系统是google内部的，已经工作了10几年的，非常稳定的容器编排工具。Borg是google内部使用的大规模的集群管理系统，久负盛名，人们对他觊觎已久，但是没有渠道获得的话，就开始开发自己的产品。当谷歌的攻城狮发现这块市场要被占领的时候，就用go语言重构了borg系统，沿用思路而不沿用代码的方式来进行构建，而且google公司使用非常聪明的做法，先把各大厂商集合在一起，特别是Linux基金会，成立了CNCF，把kubernetes贡献给了开源组织，然后用互相制约的方式来管理kubernetes。同时获得了大量开发者，特别是开源爱好者的好评。短短几年时间，kubernetes系统发展非常的迅速，尤其是国内对于kubernetes这种新兴的技术倾注了大量的心血和精力，在相关项目的参与度非常高。也正是kubernetes是站在borg系统的肩膀上，他从一出世开始就吸引了太多的目光和关注，到现在为止，他也确实没有辜负人们的期望。



kubernetes的1.0版本在2015年7月份才发布，到今天为止，它的版本已经到了1.18版本，kubenetes的代码托管在[github](www.github.com/kubernetes/kubernetes)之上，差不多是每年4个版本的速度在迭代。而2017年在容器的发展史上是非常具有里程碑意义的一年，因为在这一年当中AWS，还有微软的Azure，还有国内的著名的阿里云，这些著名的云计算公司都宣布在他们的平台上原生支持k8s，他允许用户点击几个按钮就可以快速部署k8s应用，这种就是面向kuberntes的云原生，还有一些平台甚至可以吧自己的k8s直接提供给用户，让客户可以直接把应用部署在上面，也就是容器即服务的环境，国内比如灵雀云就是一个PaaS的平台。也正是因为这些大型云厂商的支持，也使得K8s在业内收到了广泛的认可和支持，大概在2017年10月份，docker宣布他们的docker swarm上也支持k8s和docker compose，我们知道docker swarm本来是docker公司用来占领容器编排市场的最有力武器，但是docker在他的企业版本的发行过程当中，同时支持swarm和kuberntes，本来是竞争对手的，但是后来不得不把他整合进自己的产品当中去，才能获得竞争优势，同时rkt也是另外一款容器工具，rkt的容器工具当中，也有自己的编排工具叫fleet，但是他也放弃了这个项目，而原生使用kubernetes，rkt是coreos旗下的产品，而coreos目前已经被红帽收购了。红帽在他的PaaS产品openshift的核心就是kubernetes，但是kubernetes只是一个容器编排工具，没有到达PaaS这种平台即服务的标准。而openshift却解决了这些问题，成为了一款真正意义上的PaaS产品，我们也可以认为openshift是kuberntes的发行版。我们知道，kubernetes做的非常底层，真正离终端用户要自己使用kubernetes，你需要在kubernetes上面部署很多工具，以满足devops的需要，或者满足自己PaaS的需要，那么openshifit就是一个解决方法，他的里面已经集成了一切关于devops和PaaS所需要的一起工具。红帽又被IBM收购了，那么openstack这款PaaS就属于IBM了，我们可以看到这些大厂商已经花了大价钱在容器编排市场上

## 3. kubernetes特性

下面是[官方](https://kubernetes.io/)列出的kubernetes的特性

![image-20200426145026116](/pages/keynotes/L2_advanced/3_kubernetes/pics/2_BASICS_OF_KUBERNETES/image-20200426145026116.png)

咱们分三类来说：

### 3.1. 还在测试阶段的特性

+ Service Topology：基于集群的拓扑来路由服务的流量，有点像service mesh。1.17新特性，还在alpha测试。

+ EndpointSlices：可以伸缩的去追踪kubernetes集群中的网络节点。1.17新特性，beta测试

这两个是新特性，还在测试当中，我们目前不关注。

### 3.2. 一般的特性

+ Self-healing：一旦容器崩溃，由于容器易于启动，我们没有必要修复他，直接kill掉重新启动一个新的。有了k8s这样的平台之后，我们更多的是关注群体，而不再是个体。如果个体坏了就直接干掉，换一个新的。
+ Automatic bin packing：“根据资源需求和其他约束自动放置容器，同时不会牺牲可用性，将任务关键工作负载和尽力服务工作负载进行混合放置，以提高资源利用率并节省更多资源。”官方的描述很暧昧，其实就是对资源的限制，我们会对正在运行的容器进行资源使用的限制，这样，就实现了一定程度上的隔离，让其他程序在运行的时候不会受到伤害，就好像把应用程序放进bin中打包了一样。
+ IPv4/IPv6 dual-stack：自动的给pod和service分配IPv4或者IPv6的地址
+ Batch execution：作为服务的扩展，kubernetes还可以管理批量操作，CI的工作，替换失败的容器
+ Horizontal scaling：水平扩展，通过一条简单的命令或者图形界面，或者基于CPU的使用率自动的扩展应用。一个不够就在启动一个，再不够就在启动一个，只要你物理平台的支撑是足够的。但是我们的物理平台往往是不够的，所以我们会选择上公有云，而公有云的特性，按需付费，弹性伸缩就和容器的水平扩展功能无缝结合，实现真正意义上的自动扩展。

### 3.3. 考试会考的特性

+ Service discovery and load balancing：服务发现和负载均衡，当我们在k8s上有很多应用程序的时候，程序和程序之间如果存在依赖，他就可以通过服务发现的方式，找到依赖的服务，如果每个服务，我们启动了多个容器，他可以实现自动的负载均衡。

+ Storage orchestration：把存储卷实现动态编辑，也就意味着某一个容器需要用到存储卷时，根据容器自身的需要，创建能够满足他需要的存储卷，实现存储编排

+ Automated rollouts and rollbacks：Kubernetes支持一步一步的推出对应用程序或其配置的更改，同时监视应用程序运行状况，以确保它不会同时杀死所有实例。一旦出现问题，Kubernetes会为您回滚更改。从而实 现了灰度或者金丝雀发布

+ Secret and configuration management：部署和更新机密和应用程序配置，而不重建映像，也不公开堆栈配置中的机密。