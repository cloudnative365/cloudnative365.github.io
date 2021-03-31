---
title: 安全与容器安全
keywords: keynotes, architecture, security, CIS
permalink: keynotes_L4_architect_8_security_1_security_overview.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/1_security_overview
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标

+ 课程介绍
+ 认识到安全的重要性
+ 安全的一般工作定义
+ 4C安全

## 1. 课程介绍

CKS是cncf在2020年底推出的一门新的课程，CKS的全称叫Certified Kubernetes Security Specialist，他配了一门考试，同样是linuxfoundation的training，我们这套课程是从官方的E-LEARNING课程LFS260演变而来。

这套课程主要讲述了关于在kubernetes平台上，安全的构建容器时，应该遵守的一些最佳实现。所以说，建议没有基础的同学还是先从CKA开始，把kubernetes的知识掌握牢固再来看这套课程，并且CKS的证书需要我们首先获得CKA的证书。

这套课程从kubernetes的基于动态的，多项目的环境入手，解释了生产级的云环境的一些问题。提供了一系列的解决方案，很多软件和工具都是比较新的，解决了我们在部署，环境运行中，或者敏捷部署的环境中的一些问题。

## 2. 安全

这是一个非常有意思的话题，安全本身在大部分时候并不能发挥作用，因为我们在大部分的时间是**安全的**，当我们出现问题的时候，可能才会想到安全部门，而在IT部分更是这样，我们大部分的时候更多考虑的是高并发，易管理这种需求，总是把安全的需求放在最后。很多项目都是由于不得不过ISO9001或者等保三级四级之类才勉强开始把安全问题提上议程。我觉得造成这种局面的原因大致有两个

+ 越安全的系统限制越多，让负责开发和管理的工程师在做事情的时候捉襟见肘，总是得不到发挥，非常影响工程师的工作效率和心情
+ 安全部门本身没有足够的经验去把控项目，不明白什么时候，以什么样的方式，以什么样的尺度来介入项目，让项目在不耽误进度的情况下严格把控系统的安全等级

项目做到最后，安全工程师和运维工程师/开发工程师就站在了对立的立场之上，一旦有机会，就会互相倾轧，让两方势同水火。所以我觉得我们应该都来静下心考虑一下到底安全是什么，应该做什么，怎样科学的加固我们的IT系统。

### 2.1. 什么是安全

计算机安全的主要任务是保证我们的计算机或者计算机系统上所有的资产的安全。资产的类型有很多种，比如硬件，软件，数据，进程，人员，或者他们任意结合而成的资产。我们把安全的角色和职责总结成下面的闭环

![Security is a process](/pages/keynotes/L4_architect/8_security/pics/1_security_overview/b94yvrlfm5wo-LFS260_CourseGraphics_7.png)

+ 在资产assets的生命周期管理中，安全从获得资产就应该开始做了。从物理设备的监管到机器启动都是应该在安全的监管之下进行的。
+ 在软件部署的生命周期管理中，软件部署的生命周期包括了设计，开发，测试和部署等步骤。而安全需要参与到项目的每一个环节中。
+ 建立规章制度和流程来减少意外带来的风险
+ 定义系统异常时候的角色和职责

### 2.2. 安全的行为准则

我们有很多的方式去看待这个问题，但是大致上可以归纳为下面几个

![Basic security principles](/pages/keynotes/L4_architect/8_security/pics/1_security_overview/xvj8bc3xfy4o-LFS260_CourseGraphics_5.png)

+ assetment（评估）：安全部门需要针对资产进行评估，计算出安全加固的成本。以为资源有限，所以100%的安全是不可能的，我们需要根据紧急程度来对资产进行分类。那些可以在部署完成后在做加固的项，可以先记录下来，比如单点登录，删除不必要的成员，把那些有单点故障和容易遭受攻击的系统先记录下来，等到上线前再进行优化。
+ prevention（预防）：预防是对于那些显而易见的，或者花费很少的钱就可以搞定的东西提前进行防护。我们通常把这个叫做control，而control有三种：1. 技术的control，控制软件和硬件。2. 流程的control，制定流程来减少风险。3. 物理上的control，比如设备，人员，工卡，锁之类的。
+ Detection（探测）：我们需要针对系统的多种指标进行探测，比如远程登录，系统的统计信息和性能信息。而探测的动作可能是最贵的，并且是最花费精力的。Intrusion Detection and Prevention Systems (IDPS)是用来定义可能存在的风险，创建日志审计并且把状况报告给管理员的一个系统，比如splunk，就是基于可能存在的风险进行评估，最终展示出来。
+ Reaction（反馈）：很多人不太重视这一个动作，大部分的安全部门只通过一些工具发现了问题，但是如果这些问题不修复，势必会对系统造成长期的损害，所以发现了漏洞就要及时的修复，而不仅仅是把一些pdf展示出来。

### .3. 攻击者的分类

+ White Hat：这类攻击者并没有什么恶意，他们可以帮助我们来**测试**系统的漏洞
+ Black Hat：这类攻击者会攻击系统，毁坏数据
+ Script Kiddie：这些人主要使用别人已经开发好的脚本，对已知漏洞进行攻击
+ Hacktivist：这种攻击者攻入系统主要是为了发表一些社会言论
+ Nation State：国家级别的攻防战，主要是针对军事系统的
+ Organized Crime：有组织的犯罪，强行搞垮对手
+ Bots：机器人，随机的去访问，发现漏洞就攻入系统

### .4. 攻击源

攻击可能来自外部，也可能来自内部。外部的攻击基本都是一些未授权的人员利用漏洞登录系统来进行控制，而内部的攻击则指授权用户进行非法操作，比如删库跑路。。。

### 2.5. 攻击类型

+ 主动攻击：利用工具进行主动的扫描，然后侵入系统，比如使用snmp工具
+ 被动攻击：我们可能会通过截取并且分析网络流量的方式来获取到我们一些敏感信息，比如使用tcpdump抓包，那些没有经过ssl加密的数据就可以被我们获取到

## 3. 云安全

云安全有四个层次

![img](/pages/keynotes/L4_architect/8_security/pics/1_security_overview/4ys2xh00zqpn-LFS260_CourseGraphics_8.png)

我们这套课程就是要针对上面三个层次code、container、cluster进行安全加固

### 3.1. NIST Cybersecurity Framework

这是一套理论的框架，我们这边不做深入讲解了。Cybersecurity Framework (CSF) 遵循Federal Information Processing Standard (FIPS)，有兴趣的同学可以看一下这个[Computer Security Resource Center Publications web page](https://csrc.nist.gov/publications)。

就想中国的信息安全部一样，美国也会定期发布一些漏洞信息，叫做[National Vulnerability Database (NVD)](https://nvd.nist.gov/) 。我们可以通过这个网站来搜索特定的漏洞[National Checklist Program Repository page](https://nvd.nist.gov/ncp/repository)。

![img](/pages/keynotes/L4_architect/8_security/pics/1_security_overview/ep1k0dvdnc7o-NistChecklist.png)

### 3.2. CIS Benchmarks

CIS全称[Center for Internet Security](https://www.cisecurity.org/)，CIS Benchmark更多关注的是合规和最佳实现等方向的问题，关于kubernetes的CIS可以看[这个](https://www.cisecurity.org/cis-benchmarks/#kubernetes1.5.0)

![img](/pages/keynotes/L4_architect/8_security/pics/1_security_overview/wdx34n79l333-CISBenchmark.png)

CIS提供了一个工具用来跑benchmark的，我们下个文章来实验一下。

### 3.4. kube-bench

如果不能使用CIS，我们还可以考虑使用开源工具kube-bench，我们下个章节再详细介绍这个吧。
