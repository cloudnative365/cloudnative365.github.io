---
title: Kubernetes架构
keywords: keynotes, L2_advanced, 1_kubernetes, 3_CKAD, 2_Kubernetes_Architecture
permalink: keynotes_L2_advanced_1_kubernetes_3_CKAD_2_Kubernetes_Architecture.html
sidebar: keynotes_L2_advenced_sidebar
typora-copy-images-to: ./pics/2_Kubernetes_Architecture
typora-root-url: ../../../../../../cloudnative365.github.io
---

# Kubernetes架构

## 1. 课程目标

- 了解Kubernetes术语
- Kubernetes历史
- 了解master节点上的组件
- 了解worker节点上的组件
- 了解CNI（Container Network Interface）配置和网络插件

## 2. kubernetes架构

### 2.1. 什么是Kubernetes

![image-20200203171941290](/pages/keynotes/L2_advanced/1_kubernetes/2_CKAD/pics/2_Kubernetes_Architecture/image-20200203171941290.png)

在笔记本上独立运行一个容器是非常简单的。在多个主机上部署和衔接多个容器，扩展容器，无缝部署应用和全局服务发现是非常复杂的。

kubernetes从一开始就从这些阻碍着手，使用了一些**原生的**，**开放的**和**可扩展的**API。允许自定义添加新的对象（objects）和控制器（controllers），满足生产环境的需要。同时，他允许多个调度器和多个API服务器同时支持自定义的模式。

官方对于Kubernetes的定义是

```
"an open-source system for automating deployment, scaling, and management of containerized applications".
```

很重要的一点是，kubernetes是起源于谷歌15年来一直在项目中应用的系统Borg

Google的基础设施很早就实现了自动扩展，比虚拟机在数据中心的流行时代还要早，而容器对于打包整个集群提供了一个细粒度的解决方案。怎样更有效的使用集群和管理分布式应用一直是Google的核心挑战。

```
In Greek, κνβερνητης means the Helmsman, or pilot of the ship. Keeping with the maritime theme of Docker containers, Kubernetes is the pilot of a ship of containers. Due to the difficulty in pronouncing the name, many will use a nickname, K8s, as Kubernetes has eight letters between K and S. The nickname is said like Kate's.
```

Kubernetes是CI（Continuous Integration）/CD（Continuous Delivery）的不可或缺的部分，因为他提供了很多必要的组件

+ CI：用一致的方法去打包和测试软件。每天都使用代码的方式部署写完的代码，或者每小时都部署一次，不要一个季度（quarterly）一部署。像Helm和Jenkins经常在Kubernetes内部使用
+ CD：用自动化的方式在多个环境中去测试和部署软件。Kubernetes可以处理容器的全生命周期，通过更改与基础设施资源的链接，在多个deloyment中，来实现滚动升级和回滚。

### 2.2. Kubernetes的组件

部署容器和使用kubernetes可能需要在开发环境中做变更，然后系统管理员来部署应用。在传统的环境里，一个应用（比如一个网站服务器）是一个巨大的应用程序，他需要部署在一台专有的服务器上。随着网络流量的增加，应用需要调整，比如不停地迁移到更大的硬件平台。很多年后，为了满足现有的需求，就有了非常的个性化配置（这个时候，如果有人离职了，并且没有吧一些细微的改动记录或者传承下来，接了这份工作的人通常称这个为“坑”）。

如果不使用更大的服务器，kubernetes用另外的方法来解决这个问题，比如很多小的网站服务器，或者微服务。服务器和客户端都期望有更多的agent来响应。更重要的是，客户端希望无法响应的服务器进程马上结束，并且启动一个新的服务器来替代他。这样会临时部署一个新的服务器。不使用一个大型的Apache网站服务器，用很多httpd的守护进程来响应页面请求，而使用很多nginx服务器，每个服务器都去响应请求。

小规模服务器的这种临时性也同时实现了解耦。每一个这种传统的引用都使用一个替代的，但是临时的，微服务或者一个代理。把这些代理，或者他们的替身整合在一起，我们叫他service和API调用。当一套服务的流量流向另一套服务（例如，一个前端服务器到一个后端服务器），他们的IP或者任何其他的信息，如果有其中一项不可用了，应该立即有新的服务去替代它的位置。

组件之中的调用，不管是内部还是外部的调用，都是由API来调用的，这样做是为了稀松架构。配置信息是用JSON格式存储的，但是我们写的时候都是用YAML格式。Kubernetes的agents负责把YAML格式转换成JSON，并且持久化存储在数据库中。

### 2.3. Kubernetes所解决的问题

容器技术在过去的三年中被广泛应用。他为我们提供了一个非常好的方式去打包，提交和运行应用。这就是Docker的宗旨。

由于容器化技术的兴起，开发者的经验被大大增强了。容器化，特别是Docker，让开发者更方便的去打包容器镜像，使用Docker仓库去分享镜像，还使得用户方便的管理容器

但是，容器的扩展和分布式架构下的微服务，使用Docker是非常不便的。

你需要一个持续集成的流水线去构建你的容器镜像，测试，最终确认。然后，你需要一整个集群的机器去部署你运行你的容器。随后要监控他们，如果有失败的要重启应用，甚至要重启机器。你必须能够实现滚动升级和回滚，如果不需要，还得把他关掉。

所有的这些，需要松耦合的，可扩展的，容易管理的网络和存储。因为容器可能运行在任意的网络节点，容器之间的网络必须共享，但是还需要保证安全性。我们同时还需要一个持续存储，可以随时回收不需要的资源。

### 2.4. 从Borg到Kubernetes

Kubernetes与其他系统相比的优势还是他的历史非常悠久。Kubernetes是从Borg系统演变而来的，Borg系统是Google用来管理应用的系统（Gmail，Apps，GCE）

随着谷歌倾注了15年来大量的开发和运维Borg的经验才产生了Kubernetes，这让Kubernetes成为了一个非常安全，用来管理容器的方案。随着大量工具的产生，目前的kubernetes非常易于管理，大量的重复性工作在Google的数据中心已经消失。

![8h5c5pp0snna-Kuberneteslineage](/pages/keynotes/L2_advanced/1_kubernetes/2_CKAD/pics/2_Kubernetes_Architecture/8h5c5pp0snna-Kuberneteslineage.jpg)

+ Kubernetes的族谱

Chip Childers, Cloud Foundry Foundation, 参考 ["The Platform for Forking Cloud Native Applications"](https://www.slideshare.net/chipchilders/cloud-foundry-the-platform-for-forging-cloud-native-applications).

想学习更多的Kubernetes的思想，可以参考 ["Large-Scale Cluster Management at Google with Borg"](https://ai.google/research/pubs/pub43438).

Borg促进了当前的数据中心系统的演变，激发了容器运行时技术的潜力。Google在2007年吧cgroups技术贡献给了Linux内核；cgroups是用来限制资源的使用。cgroups和Linux namespaces都是目前容器技术的核心，也是Docker的核心。

Mesos就是在和Google接触之后被激发了灵感，Meso当时还不知道Borg的存在。诚然，Mesos创造了一个多级的调度机制，它的目的也是为了更好的使用数据中心的集群。

Cloud Foundry Foundation严格遵守着12条铁律[twelve-factor application principles](https://12factor.net/). 遵守这些铁律，可以让大家在创建web应用时可以轻松的扩展，可以在云上部署，自动化部署。Borg和Kubernetes都谨记这些规则。

### 2.5. Kubernetes架构

为了更好的理解Kubernetes，让我们来看一看架构图，这是一个全局的架构图，包含了系统的组件。

![himie4w8ap52-KubernetesArchitecture](/pages/keynotes/L2_advanced/1_kubernetes/2_CKAD/pics/2_Kubernetes_Architecture/himie4w8ap52-KubernetesArchitecture.png)

在这个最简单的模型里，Kubernetes有一个中心控制器，和一些工作节点。这个管理器上运行着一个API server，一个scheduler，多种的控制器和一个存储系统，还有容器和网络配置。

kubernetes通过API server暴露API：你可以通过本地客户端kubectl或者开发一个你自己的客户端。kube-scheduler监控着向API申请容器的请求，然后找一个合适的节点来运行。集群中每个节点都有两个进程：kubelet和kube-proxy。kubelet接受运行容器的请求，管理所有需要的资源并且在本地监控他们。kube-proxy创建并且管理容器暴露在网络的上的规则。

Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

### 2.6 专业术语

我们知道，kubernetes是一个编排系统，来用部署和管理容器。容器不是单独被管理的，他们是作为一个大的对象中的一员，他叫**pod**。一个pod包含一个或者多个容器，他们共享一个IP地址，有共同的存储和名称空间。我们一般用一个pod运行一个容器作为一个应用，而其他的容器辅助这个主要的应用。

编排就是被不停的监视，我们所说的是operators和controllers。每个controller都会不停的访问kube-apiserver关于特定对象的状态，修改对象一直到预期的状态。默认的，最新的和满足预期状态的容器控制器叫做Deployment。一个Deployment确保资源是可用的，比如IP地址和存储，然后就部署一个ReplicaSet。ReplicaSet是用来部署，重启容器或者Docker的，直到满足需要运行的容器数量为止。在以前的版本中，这个功能是由ReplicationController来负责的。我们还有Jobs和CronJobs来处理单次的，或者循环的请求。

我们还有很多其他的API对象可以用来部署pods。DaemonSet是用来保证每个节点上都运行一个相同的pod。这个经常通用来记日志或者是跑metrics的pod。StatefulSet是用来按顺序部署pod，例如第二个pod需要在第一个pod完成之后再开始运行。

想象一个成千上万的Pods跑在几百台机器上是多么难以管理。为了让管理更简单，我们可以使用labels，并且赋值的方式，让他们成为对象元数据的一部分。这样我们在查找和更改对象状态的时候，就不用去知道他们自己的名字或者UID了。可以给节点打taints来阻止Pod被分配到这个节点上，除非在它的元数据中加入了toleration。

元数据中还可以加annotations，但是kubernetes命令不会使用他们，但是第三方的工具有可能会用到。

### 2.7 Master节点

Kubernetes的Master运行着多个服务，并且管理着集群的进程。Master节点上运行着kube-apiserver，kube-scheduler和etcd数据库。随着软件的逐渐成熟，有很多专门为了满足特定需求的组件应运而生，比如cloud-controller-manager，他的作用是，当有其他的交互型工具和kube-controller-manager交互的时候，他就会启动，就比如Rancher和DigitalOcean等第三方集群管理工具。

在Master节点上同时还有很多插件，已经成为了生产系统的标配，比如DNS服务器。还有很多其他的第三方工具，但是Kubernetes还没把他们变成自己的组件，就比如集群级别的日志系统和监控系统。

### 2.8. Master节点的组件

+ kube-apiserver
  kube-apiserver是kubernetes集群的中心控制节点。
  所有的调用，不管是内部的还是外部的，都是有它来负责处理的。他负责接收和验证所有的动作，而且他是唯一一个和数据库连接的组件。所以他运行为整个集群的主进程，作为集群的前端来共享状态。每个API调用都需要三个步骤，认证，授权然后发给对应的控制器。
  
+ kube-scheduler
  kube-scheduler使用某种算法去决定运行容器的Pod在哪个节点上。scheduler会衡量所有的可用资源，然后去部署并且在失败后会重试，直到成功，当然如果容器本身有问题，会有规则去规定他重试的次数，每隔几分钟去重试。
  算法规则有非常多的种类，或者你可以使用自己开发的。还可以把Pod绑定在某一个特殊的节点，但是如果不满足其他条件的话，可能会让Pod变成pending状态。
  
  Pod是否能够部署在某个节点的第一个因素就是当前节点的资源限制。然后是taints（污点）和tolerations（容忍度），然后是Node节点上的标签是否满足Pod的要求。
  
  想看深入学习的话，请看源代码，[details about the scheduler on GitHub](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/scheduler.go).
  
+ etcd Database
  etcd负责存储集群，网络，和其他需要持久存储的信息，确切的说，他是一个b+树的K-V存储。etcd不是查找或者是改变一个entry，数值一般是在结尾追加。而前一个数据就被标记成为要被移除。我们可以使用curl或者其他的http方式去查看内部的数据。
  向集群发送的更新状态的并发请求都是要经过kube-apiserver的，然后他们被一起送给etcd。如果第一个请求会更新数据库，第二个请求就不会有相同的版本号，否则kube-apiserver就会给发起者一个409的错误代码。在服务器端不会去处理请求的逻辑问题，这就是说，客户端需要等待服务器端的拒绝请求，然后再根据请求做出响应。
  集群中有一个主节点，其他可用的节点都作为从节点。这些节点和其他所有节点进行通讯，然后选出master节点，然后其他所有选举失败的都作为从节点。在使用集群的过程中，可能会有闪断，比如集群的整体升级。从1.15.1版本开始，kubeadm可以轻松实现部署多个etcd主节点的集群或者其他数据库的集群。
  
+ 其他的组件（agents）
  kube-controller-manager是一个守护进程，他不停的和kube-apiserver进行交互，来查询集群的状态。如果状态不能达到预期，集群就会通知相应的控制器，让控制器来控制应用达到预期的状态。一直在使用的控制器种类很多，比如endpoints，namespaces和replication。这些控制器的扩展让kubernetes日趋成熟。
  在v1.16beta版本中，cloud-controller-manager和外部的组件（agents）进行交互。他们负责处理从kube-controller-manager分发来的请求。这样的话，会让集群快速变化而不用改变kubernetes核心的控制进程。但是在kubelet编译的过程当中，必须使用--cloud-provider-external选项。

### 2.9. 工作节点（Minion/Worker nodes）

所有的工作节点都会运行kubelet和kube-proxy，当然还有容器的引擎，比如Docker或者cri-o。其他用来管理的守护进程都是用来监控这些进程（agents）或者给kubernetes提供自身没有的服务。

我们也可以去运行一些Docker引擎同类的产品，比如CoreOS的rkt。想要知道怎么用的话，请自己去查询文档。：）在未来的版本当中，kubernetes很有可能集成容器运行的引擎。

