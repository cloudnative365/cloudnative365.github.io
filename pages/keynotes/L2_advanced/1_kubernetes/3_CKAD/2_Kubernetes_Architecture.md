---
title: Kubernetes架构
keywords: keynotes, L2_advanced, 1_kubernetes, 3_CKAD, 2_Kubernetes_Architecture
permalink: keynotes_L2_advanced_1_kubernetes_3_CKAD_2_Kubernetes_Architecture.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/2_Kubernetes_Architecture
typora-root-url: ../../../../../../cloudnative365.github.io
---

## 1. 课程目标

- 了解Kubernetes术语
- Kubernetes历史
- 了解master节点上的组件
- 了解worker节点上的组件
- 了解CNI（Container Network Interface）配置和网络插件

## 2. kubernetes架构

### 2.1. 什么是Kubernetes

![image-20200203171941290](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/2_Kubernetes_Architecture/image-20200203171941290.png)

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

![8h5c5pp0snna-Kuberneteslineage](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/2_Kubernetes_Architecture/8h5c5pp0snna-Kuberneteslineage.jpg)

+ Kubernetes的族谱

Chip Childers, Cloud Foundry Foundation, 参考 ["The Platform for Forking Cloud Native Applications"](https://www.slideshare.net/chipchilders/cloud-foundry-the-platform-for-forging-cloud-native-applications).

想学习更多的Kubernetes的思想，可以参考 ["Large-Scale Cluster Management at Google with Borg"](https://ai.google/research/pubs/pub43438).

Borg促进了当前的数据中心系统的演变，激发了容器运行时技术的潜力。Google在2007年吧cgroups技术贡献给了Linux内核；cgroups是用来限制资源的使用。cgroups和Linux namespaces都是目前容器技术的核心，也是Docker的核心。

Mesos就是在和Google接触之后被激发了灵感，Meso当时还不知道Borg的存在。诚然，Mesos创造了一个多级的调度机制，它的目的也是为了更好的使用数据中心的集群。

Cloud Foundry Foundation严格遵守着12条铁律[twelve-factor application principles](https://12factor.net/). 遵守这些铁律，可以让大家在创建web应用时可以轻松的扩展，可以在云上部署，自动化部署。Borg和Kubernetes都谨记这些规则。

### 2.5. Kubernetes架构

为了更好的理解Kubernetes，让我们来看一看架构图，这是一个全局的架构图，包含了系统的组件。

![himie4w8ap52-KubernetesArchitecture](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/2_Kubernetes_Architecture/himie4w8ap52-KubernetesArchitecture.png)

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

Supervisord是一个轻量级的进程监控工具，用来监控和通知传统Linux环境上的其他进程。集群中，这个进程监控kubelet和docker进程。如果他们出现意外，这个进程就会重启他们，并且把日志记录下来。

kubernetes目前没有能够覆盖整个集群的日志系统。但是，我们使用另外一个CNCF项目叫Fluentd。他为集群提供了一个统一的日志管理解决方案，包括过滤，缓存还有路由消息。

### 2.10. 工作节点的组件

+ kubelet
  kubelet和底层的Docer引擎进行交互，确保需要运行的容器处于正常运行的状态。
  worker节点承担了大部分的变更和配置工作。Pod的spec（spacifications）文件通过API调用传递给worker节点。worker节点就开始操作本地的节点，直到满足spce文件的所有需求。
  如果Pod需要其他资源，比如storage，secrets或者configMaps，kubelet会确保Pod可以访问他们，如果没有，就创建。然后把最终结果返回给kube-apiserver。
+ kube-proxy
  kube-proxy负责管理容器之间的网络连接。他是通过iptables来管理网络的。他通过用户用户空间（userspace）的方式，通过一个很高数字的随机端口，给Services和Endpoint转发流量
+ Container Runtime
  容器的运行环境就是指那些运行容器的软件。容器不等于Docker，Docker是软件，容器是技术，Docker实现了容器技术。Kubernetes支持多种容器运行环境：Docker，containerd，cri-o，rktlet和很多种的实现方式。[Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)。

### 2.11.  Pods

Kubernetes的关键就是要编排容器的整个生命周期。我们不会针对某一个特殊的容器进行交互。我们通常把最小的一个单元叫做[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)。有些人会把他想象成一群鲸鱼的集合（Docker的形象就是鲸鱼）或者是一个豌豆荚。基于资源共享的设计，一个Pod一般会遵循一个容器跑一个进程的架构。

Pod中的容器，默认是同时启动的。这样会导致没法判断Pod中哪个容器是第一个启动的。我们可以使用initContainers来确保某些容器先于其他在同一个Pod中的容器启动。为了支撑容器中的某个进程，我们可以需要记日志logging，代理proxy，或者其他的适配器adapter。这些任务通常由同一个Pod中的其他容器来负责。

大部分的网络模型中，一个pod只有一个IP地址。（HPE实验室发明了一个插件，可以让一个Pod拥有不止一个IP地址）这样的话，如果一个Pod中有不止一个容器，他们就必须共享IP。想要和其他容器沟通，就必须使用IPC（Inter-Process Communication），loopback接口或者是共享文件系统。

一般来说一个容器中只有一个应用，通常有多个容器的情况中，都会有一个容器是用来记日志的。尽管主要的应用也会有这些功能，但是我们通常会使用sidecar来辅助，例如记日志，或者响应请求。

Pods和其他的对象可以使用多种方式去创建。他们通常使用生成器（generator）去创建。但是生成器的版本会随着新版本的发布而改变。

```bash
$ kubectl run newpod --image=nginx --generator=run-pod/v1
```

或者我们也可以使用应用文件的方式来创建和删除（这也是我最推荐的方式）

```bash
$ kubectl create -f newpod.yaml

$ kubectl delete -f newpod.yaml
```

其他资源则使用**operators/watch-loops**来保证spec文件和当前的状态保持一致。

### 2.12. Services

随着对象object（pod）和代理agent（kube-proxy）的解耦，我们需要一个灵活的，可扩展的代理在Pod死掉或者被替换之后，把资源整合在一起，并且支持重新连接。每个服务都是一个微服务，负责处理一部分的流量，就好像一个NodePort或者一个LoadBalancer把入栈流量分发给多个Pods。

Service也负责处理入栈的规则，比如资源的控制和安全控制。

Service和kubectl命令一样，都使用selector来判断应该和哪个对象去链接。目前版本支持两个selector：

+ Equality-based
  通过标签的键和值来过滤。我们可以使用三种符号，**=**，**==**，**!=**。如果有很多的键值，必须满足所有的条件。
+ Set-based
  通过集合来过滤。我们可以使用**in**，**notin**和**exists**。比如`status notin (dev, test, maint)`会选择键**status**的值是**dev**，**test**，或者**maint**。

### 2.13. Controllers/Operators

在编排中一个很重要的概念就是使用控制器controllers。他们负责监视和具体的操作。他们查询当前的状态，然后和spec中的状态做比较，然后去填补他们不同的那部分。kubernetes当中有非常多的控制器，也可以开发自己的控制器。我们可以吧控制器简单的认为是一个代理，一个通知器，还有一个下游的存储。他们使用DeltaFIFO队列，用来比较源头和下游的状态。一个循环的进程从FIFO的队列中接受一个对象，对象中含有一个数组，包含了delta（不同的部分）。只要delta不是Deleted（删除）类型的，逻辑控制器就开始创建或者修改相应的对象，直到他满足spec中的定义。

通知器从API server接受请求，并且查询某个对象的状态。为了减少API server的负载，数据都是被缓存的。当一个对象被很多其他的对象复用时，kubernetes会使用SharedInformer。他可以创建一个共享的缓存，用来存储多个请求的状态。

每个工作队列都是独立的，用来处理多个工作节点的任务。标准的Go工作队列workqueues都是有使用限制的，并且有延时，而时间队列就是最典型的应用。

**endpoint**，**namespace**和**serviceaccounts**控制器分别控制与他们同名的资源，也就是说，有endpoint资源，也有endpoint控制器。

### 2.14. Pod的独立IP

一个Pod代表着一组互相协助的容器，他们共享着数据卷。Pod中所有的容器共享一个相同的网络名称空间。

下面的图表显示了一个Pod带着两个容器，A和B，和两个数据卷，1和2。容器AB与第三个容器共享一个网络名称空间，第三个容器叫做Pause容器，他是用来获取IP地址的，然后所有Pod内的容器都去使用它的网络名称空间。他们和卷组1，2组成了一个完整的Pod。

![hl3vukvzmyl3-APod](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/2_Kubernetes_Architecture/hl3vukvzmyl3-APod.png)

为了互相通信，容器可以使用loopback接口，把文件写在文件系统上，或者通过inter-process communication（IPC）的方式。如果把协作的应用都放在一个pod中会产生问题。目前只有一个网络插件可以让pod拥有多个IP地址，他是由HPE实验室开发的项目。

![image-20200212144428013](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/2_Kubernetes_Architecture/image-20200212144428013.png)

### 2.15. 网络是怎么工作的

我们前面说到的组件对于习惯于配置管理的系统管理员来说可能就是一些普通的任务。但是，想让kubernetes集群获得所有的功能，还需要合理的配置网络。

如果有在IaaS平台部署过虚拟机，其实这个非常的相似。唯一不同的是，在kubernetes中，最小的计算单元不是容器，而是Pod。

Pod是一组协作的容器，他们公用一个IP地址。从网络的角度出发，pod可以被看作是一个物理机上的虚拟机。网络需要把IP分配给Pod，然后在所有节点上提供网络流量的路由。

在容器编排系统中，我们需要解决的问题有：

+ 容器和容器之间的耦合（这个已经用pod解决了）
+ pod和pod之间的通信
+ 外部和pod之间的通信

kubernetes期望网络可以实现pod和pod之间的通信，但是他自己是没办法实现的。pod在容器启动的时候就会获得IP。对于内部网络通信，Service通过ClusterIP的方式，对于外部的通信，通过NodePort的方式，并且可以通过LoadBalancer去配置负载均衡。

详细的网络模型解释，可以参考官方文档，[Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)。

下面我们来研究一下Service的工作流。

![rs7sj920kkap-NetworkingSetup-3](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/2_Kubernetes_Architecture/rs7sj920kkap-NetworkingSetup-3.png)

下面是整个的集群网络的工作流。

![rs7sj920kkap-NetworkingSetup-4](/pages/keynotes/L2_advanced/1_kubernetes/2_CKAD/pics/2_Kubernetes_Architecture/rs7sj920kkap-NetworkingSetup-4.png)

带领大家开发Kubernetes的领导之一Tim Hockin，写了一个非常实用的slide来帮助我们了解kubernetes的网络，[An illustrated Guide to Kubernetes Networking](https://speakerdeck.com/thockin/illustrated-guide-to-kubernetes-networking).

### 2.16. CNI网络配置文件

为了给容器提供网络，kubernetes需要用到[Container Network Interface (CNI)](https://github.com/containernetworking/cni)配置文件。从v1.6.0开始，kubeadm实用CNI作为默认的网络接口机制。

CNI是一个非常重要的spec文件，因为他内部含有大量的库，这些库是用来支持管理容器的插件所准备的，这些插件可以在容器被删除后重新配置或者删除已经分配的资源。它的目的是在不同的网络解决方案（calico，flannel）和不同的容器运行环境（docker，containerd，rkt）之间提供一个通用接口。因为CNI的规则和他的插件的开发语言相关，所以有很多插件，比如Amazon ECS，SR-IOV，到Cloud Foundry等等。

使用CNI，我们可以使用下面的网络配置文件

```JSON
{
   "cniVersion": "0.2.0",
   "name": "mynet",
   "type": "bridge",
   "bridge": "cni0",
   "isGateway": true,
   "ipMasq": true,
   "ipam": {
       "type": "host-local",
       "subnet": "10.22.0.0/16",
       "routes": [
           { "dst": "0.0.0.0/0" }
            ]
   }
}
```

这个配置定义了一个标准的Linux bridge，叫cni0，他可以在子网10.22.0.0/16网段下分配IP地址。这个桥接插件会在合适的名称空间中，为容器定义适当的网络和网络接口。详细信息我们可以参考[CNI GitHub repository](https://github.com/containernetworking/cni)的README文件

### 2.17. pod间通信

尽管CNI插件可以用来配置pod的网络，并且为pod提供一个独立的IP，但是他并不能帮助你实现跨node的pod间通信。

早期的kubernetes需求有下面这几个：

+ 所有的pod可以和其他任何节点上的node通信
+ 所有的node可以和任意的pod通信
+ 没有NAT（Network Address Translation）网络地址映射

一般来说，所有的IP地址，包括node和pod的，都是通过路由表，而不是NAT。当我们访问他们的时候，就好像是在访问物理的网络架构。否则，要不然，我们就需要用一些软件定义的overlay解决方案，比如：

+ [Weave](https://www.weave.works/oss/net/)
+ [Flannel](https://coreos.com/flannel/docs/latest/)
+ [Calico](https://www.projectcalico.org/)
+ [Romana](https://romana.io/)

大多数的网络插件目前都支持使用网络策略（Network Policies），他就好像内部防火墙，控制入栈和出栈的流量。

更多的信息请参考[Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) 和 [networking add-ons](https://kubernetes.io/docs/concepts/cluster-administration/addons/).

### 2.18. 扩展阅读

报告：["Large-Scale Cluster Management at Google with Borg"](https://ai.google/research/pubs/pub43438)

听一下：[John Wilkes talking about Borg and Kubernetes](https://www.gcppodcast.com/post/episode-46-borg-and-k8s-with-john-wilkes/)

参加交流会：[community hangout](https://github.com/kubernetes/community) 

加入Slack社区：[Slack](http://slack.kubernetes.io/)的**#kubernetes-users**的频道

Stack Overflow社区：[Stack Overflow community](https://stackoverflow.com/search?q=kubernetes)

