---
title: 课程介绍
keywords: keynotes, advanced, kubernetes, kubernetes_2023, COURSE_INTRODUCTION
permalink: keynotes_L2_advanced_4_kubernetes_2023_1_course_introduction.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/1_course_introduction
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 课程简介

### 1.1. 运维工程师的发展方向

最早的容器技术是LXC，最广泛的容器技术是Docker，最流行的容器编排技术则是Kubernetes。也是目前运维职位的天花板SRE工程师的技术基础。我们目前的运维体系基本都是围绕IaaS，PaaS和SaaS展开的。

![img](/pages/keynotes/L2_advanced/4_kubernetes_2023/pics/1_course_introduction/1620.jpeg)

虽然目前很多的企业纷纷把业务转向了公有云，但是依然有很多大型企业由于各种原因，依旧把业务放在私有云上。这些原因有业务方面的，比如业务比较复杂，或者对于公有云支持不好等，也有法律合规安全方面的，比如安全考虑，等保要求之类的。这就导致了我们的架构更加倾向于混合云架构，也就是传统的DC环境，通过专线直连，或者VPN专线的方式，和公有云上的环境打通，形成混合云的架构。对于IaaS层本身，我们可以通过打通网络的方式。而对于PaaS而言，混合云PaaS也是目前非常火热的一个话题。各大厂商纷纷开始基于混合云PaaS开发自己的产品，有开源的也有商用的。开源的比如红帽的OpenShift，红帽，阿里合作的产品KubeVula，青云的kubesphere，老牌PaaS产品Rancher，都是在原有PaaS平台基础上添加了多云的融合功能。但是由于网络的复杂性，并不是每个产品都能适配所有的公有云的，这就需要架构师有足够的经验才能判断是否适合自己的架构。

对于我们传统的运维工程师，懂操作系统，懂虚拟化，懂网络就足够胜任了，这类工程师我们通常叫PE（production engineer），随着容器化时代的到来，PE工程师需要面对更多更复杂的环境，以网络来说，不仅要面对主机网络，还要应对服务网络和容器网络，工作量乘几何倍数增长，就需要更多的时间来应对容器化的环境，我们把维护这种环境的工程师叫做SRE（Site Reliability Engineering），为了说明这个新的概念，google专门出了一门书，中文叫《Google SRE运维解密》，其实这个书是有[电子在线阅读版](https://sre.google/sre-book/table-of-contents/)的，只不过需要一些上网的技巧。

![img](https://pic3.zhimg.com/v2-598d1a8c0a891593504500ae14f1fbae_r.jpg)

因此，从我们的成长历程来说，成为一名合格的SRE工程师，我们要掌握的知识可真是不少。而如果想成为一名高级的运维工程师，比如我们在看招聘网站的时候那些云计算工程师，SRE工程师，DevOps工程师，容器工程师，k8s都是比必不可少的技能，而且算是其他技能的基础。根本原因还是因为新技术都在向容器化发展，老技术也在向容器化迁移，而容器化编排工具必不可少，kubernetes是目前唯一的选择。

### 1.2. 从Docker到Kubernetes

Docker作为最广泛的容器工具，目前势头依然不减，虽然kubernetes官方已经在新的1.23版本中摒弃了Docker作为容器引擎，但是并不是封杀Docker，而是换一种方式而已。我们后面介绍kubernetes架构的时候还会说到。

Docker的优势很明显，一键部署，跨平台，但是缺点也很明显，那就是没办法通过引擎本身进行容器的生命周期管理，比如多个容器环境下，谁先启动，谁后启动。Docker针对这个问题推出了自己的解决方案，叫做Docker swarm。Docker swarm虽然解决了容器的生命周期管理问题，但是没办法在多台宿主机上进行编排。而此时另外一家公司已经把容器编排技术在企业内部实现了，这家公司就是Google。Google用C语言开发了自己的容器编排系统Borg，并且已经经过了很长时间的生产系统的验证，为了抢占先机，Google这步棋可谓是一箭多雕。他用Golang语言重写了Borg系统，并且命名为kubernetes，并且和Linux基金会一起成立了CNCF，最后用云原生改变了IT行业的格局。我们可以看到这步棋一共攻占了多个市场，kubernetes用golang编写，一下子提高了Golang语言的知名度，CNCF的成立，为kubernetes报价护航，可以说，里面吸收的开源项目都是为kubernetes服务的，而会员的也必须多少和kubernetes搭边，要不扩展kubernetes，要么基于kubernetes，云原生的概念一下被推到了风口浪尖。

### 1.3. kubernetes培训

官方为了普及kubernetes，培训和认证也是不可或缺的，官方针对Kubernetes推出了4个kubernetes认证和1个prometheus认证

- [CKA](https://www.cncf.io/certification/cks/)(Certified Kubernetes Administrator)：这个是我们的主要目标，里面的知识涵盖了我们所有的基础知识，比较倾向于管理
- CKAD(Certified Kubernetes Application Developer)：这个是针对开发者推出的培训，主要倾向于使用，没有CKA的内容深刻
- CKS(Certified Kubernetes Security Specialist)：这个是CKA的进阶，需要有CKA认证才能考取CKS认证，是针对安全的认证
- KCNA(Kubernetes and Cloud Native Associate)：算是一个特别基础的课程，适用于非技术人员，比如销售人员
- PCA(Prometheus Certified Associate)：这是2022年新推出的课程，针对prometheus的培训

CKA同时也是获得CNCF认可的方式之一，比如我们想加入CNCF成为赞助商，或者提供kubernetes的咨询或者培训，都需要有一定数量的CKA证书。

## 2. 课程内容

### 2.1. 官方课程

官方的课程是网页形式的，不是视频讲座，中间会穿插一下图片和互动。然后提供一个实验手册，考试的内容全部都是从实验手册中来的。

| DOMAIN                                    | WEIGHT |
| ----------------------------------------- | ------ |
| Cluster Setup                             | 10%    |
| Cluster Hardening                         | 15%    |
| System Hardening                          | 15%    |
| Minimize Microservice Vulnerabilities     | 20%    |
| Supply Chain Security                     | 20%    |
| Monitoring, Logging, and Runtime Security | 20%    |

### 2.2. 课程计划

咱们的课程会以考试为目标，实用为重点，扩展内容随时穿插，给大家最系统的知识梳理，同时授人以渔，让大家具备自学的能力。因为kubernetes是一门快速发展的技术，每年都会以4个小版本的速度进行更新，且每个版本都会有一些变化。就好像网游一样，不停的推出新的任务和系统，让大家保持热情。kubernetes也一样，目前还是处在非常高速的版本迭代期。这就要求我们老师和同学都保持对游戏一样的热情，首先了解现有版本的玩法，再以此为基础，跟上社区的发展速度，走到技术的前端。

+ 基础部分：着重kubernetes入门，让没有kubernetes经验的同学快速上手kubernetes
+ CKA考试部分：针对考试涉及到的内容进行重点说明，方便大家有针对性的学习
+ 实用部分：针对平时工作中会用到的部分进行讲解，让大家学完这套课程之后就可以直接上手工作
+ 案例部分：针对官方文档中涉及案例进行讲解，授人以渔，给大家自己学习的能力

## 3. 学习kubernetes途径和方法

刚才说到，kubernetes的更新速度非常的快，想跟上他的发展速度，最有效的方式就是跟社区，也就是[github](https://github.com/kubernetes/kubernetes)。但是这对于大部分同学都是不太现实的。除非我们是kubernetes的开发者，否则不可能把所有精力投入到代码中去。特别是对于初学者来说更是不可能的事，我把整个学习过程分为三个部分

+ 打好基础：我们可以选择在实际工作中找前辈带一带，同时跟着文档做一做。但是工作不是学习，别人不可能把时间都放在带新人上，况且工作上的知识以实用为主，并不一定是系统的，好理解的知识，我们就需要在业余时间多查资料，夯实基础。
+ 熟练掌握：当我们已经有了一定基础的时候，想要继续深入，我们最常使用的方式就是查阅文档和社区提问，文档指的是[官方文档](https://kubernetes.io/docs/home/)，社区就是github或者stackoverflow等。官方文档是最权威的解释，github的提问最新最快。
+ 深入研究：最直接的方式就是读源码！当然，这对于大部分的同学来说还是有些难度的，这要求我们有代码经验和一定的英文阅读能力。我们也可以关注一些k8s贡献者的动态，看看他们是怎么提交bug，更新版本的。这是一个终身学习的阶段，也就表示我们要把全部精力都投入到k8s当中了

对于一般开发者来说，我们只需要做到打好基础就可以。对于Admin来说，我们需要做到熟练掌握。如果想真正在这个领域所有建树，就需要做深入研究。