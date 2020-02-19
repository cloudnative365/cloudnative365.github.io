---
title: 创建Kubernetes资源
keywords: keynotes, L2_advanced, 1_kubernetes, 3_CKAD, 3_Build
permalink: keynotes_L2_advanced_1_kubernetes_3_CKAD_3_Build.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/3_Build
typora-root-url: ../../../../../../cloudnative365.github.io

---

## 1. 课程目标

- 学习有关运行环境和容器的选项
- 应用的容器化
- 管理本地的仓库
- 部署多容器的pod
- 配置readinessProbes（readiness探针）
- 配置livenessProbes（liveness探针）

## 2. 创建资源

### 2.1. 容器的选择

目前有很多的组织都在竞相开发容器。随着各大社区的开放和互相兼容，作为一个编排工具，kubernetes的开发方向也向着兼容大部分的容器而努力。最早的和最健壮的当属Docker。随着Docker的逐渐完善，kubernetes也逐渐通过支持其他厂商的软件的功能来完善其自身容器的创建和部署，这使得新项目和功能的开发变得更加流行。而在其他容器引擎变得成熟的过程中，kubernetes也变得更加开放和独立。

容器的运行环境是为容器化的应用程序提供请求解析的。Docker是kubernetes的默认引擎，同时，CRI-O, rkt和其他的引擎都是由社区支持的。

容器化之后的镜像可以从Docker移植到别的地方，只要镜像不是绑定了非常高级别的工具，这让他在操作系统和环境中更加方便的移植。OCI（Open Container Initiative）这个组织就是为了这个目的成立的。Docker贡献了他们的容器库libcontainer工程组成了一个新的代码库，叫[runC](https://github.com/opencontainers/runc)。

Docker曾经是开发着的不二选择，但是目前趋势向着开放的规格和更加宽松的规范，这说明使用第三方的功能是个明智的选择。

### 2.2. Docker

Docker公司成立于2013年，目前Docker已经等同于运行容器化应用的代名词。Docker可以轻松容器化，部署和使用应用。所以，他成了为生产环境中的默认选项。在复杂的架构中，使用开放的镜像仓库Docker Hub，我们可以一键安装，下载和部署第三方，或者定制化的镜像。他的易用性让他成为了大部分开发着的首选。而kubernetes也默认使用Docker作为他的容器引擎。

在过去几年中，Docker持续的改进，并且增加了新功能。这其中就包含了编排功能，叫做Swarm。这些增加的功能在生产中增加了服务器的开销，同样也增加了生产系统对于某一个供应商的依赖。这就激发了一些开源软件的出现，比如CRI-O。尽管很多聪明的开发者希望寻找容器和kubernetes开源工具，但是在生产中，Docker依然是首选。

### 2.3. CRI（Container Runtime Interface）

CRI主要用来把各种容器运行环境和kubelet集成。通过为API，明细和库提供protobuf方法，新的运行环境可以不用深入理解kubelet内部结构就轻松的和kubelet进行集成。

项目目前还处于alpha测试阶段，有很多的开发工作正在进行。目前Docker-CRI的集成已经完成了，新的运行环境可以轻松的被添加和移除。现在，CRI-O，rktlet和frakit正在开发当中。

### 2.4. rkt

rkt运行环境，发音叫rokcet，他为运行容器提供了一个命令行。在2014年由CoreOS发布的，目前是CNCF家族项目当中的一员。从早期的Docker中总结经验，更加专注于安全，开放和便于操作。很多的功能都随着Docker改进。尽管他还不能轻易的取代Docker，但是已经取得了很好的成效。rkt使用appc的清单文件，可以运行Docker，appc和OCI的镜像。他还可以用来部署pods。

目前有很多人在关注这个项目，并且期望他可以取代Docker，直到CRI-O成为Kubernetes官方的孵化项目改变了这一切。

### 2.5. CRI-O

这个项目目前是Kubernetes的孵化项目之一。他使用的是Kubernetes Contianer Runtime Interface和OCI（Open Container Initialtive）集成，因此名字叫CRI-O。目前已经支持runC（默认支持）和Clear Containers（Intel的技术），但是项目的阶段性目标是兼容任何支持OCI的运行环境。

这个项目比Docker或者rkt都要新，他的灵活性和兼容性还是得到了不少的关注。

### 2.6. containerd

这个项目的不是为了创造一个面向用户的工具，相反的，他专注于交付一个高度解耦，非常底层和复古的环境：

+ 根据OCI的规则，默认使用runC去运行容器
+ 为了可以嵌入更大的系统当中去
+ 最少的命令行，专注于分析系统漏洞debug和开发

由于专注于底层的，钻研容器后端的运行，这个项目更加容器被集成，让运维团队可以在生产上定制化容器，而不是创建一个标准的容器去打包，分发和运行。

### 2.7. 容器化一个应用

想要容器化应用，我们需要从创建应用开始。不是所有的应用都可以容器化。越是无状态的，临时性的应用越好容器化。同时，删除任何与环境相关的配置，我们用其他的工具去提供配置，比如ConfigMaps和secrets。开发程序直到完成一个功能，这个功能可以部署到多个环境中去而不用做任何更改，我们只需要解耦配置文件。这个时候，传统的应用就变成了一些列的对象和功能，围绕在多个容器之间。

Docker的使用已经成为了工业标准。大型的厂商，比如RedHat正在转移向其他的开源工具。目前最新，最有希望成为新的标准的两款工具。

+ [buildah](https://github.com/containers/buildah)：

  ```
  is a tool that focuses on creating Open Container Initiative (OCI) images. It allows for creating images with and without a Dockerfile. It also does not require superuser privilege. A growing golang-based API allows for easy integration with other tools. 
  ```

  

![image-20200217171419942](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/3_Build/image-20200217171419942.png)

+ [podman](https://github.com/containers/libpod)

  ```
  a "pod manager" tool, allows for the life cycle management of a container, including creating, starting, stopping and updating. You could consider this a replacement for docker run.
  ```

  ![image-20200217171536695](/pages/keynotes/L2_advanced/1_kubernetes/3_CKAD/pics/3_Build/image-20200217171536695.png)

### 2.8. 创建Dockerfile

创建一个文件夹储存配置文件。`docker build`进程载创建镜像的时候会把所有东西都放在这个文件夹下。把所有的相关脚本和文件都放在这个文件夹下。

同时，在文件夹里，创建一个Dockerfile。名字很重要，他必须叫`Dockerfile`，首字母要大写。最新的版本已经允许使用不同的名字了，但是在创建的时候必须要使用 `-f <filename>`。这个文件告诉`docker build`命令怎样创建这个镜像。

每个指令被Docker守护进程以迭代的形式放到队列当中去。指令是大小写不敏感的，但是我们约定俗成的把他大写，这样可以轻松的把他和参数区分开来。文件必须以FROM指令开头。这个指令声明了我们的容器是从那个基础镜像制作的。接下来，我们可以使用大量的`ADD`,`RUN`和`CMD`指令去添加资源和运行命令。我们可以参考Docker的官方文档[Dockerfile reference](https://docs.docker.com/engine/reference/builder/)。

我们通过list命令可以查看我们的镜像是否成功创建，并且使用`docker run <app-name>`来执行他。如果我们发现镜像没有问题，我们就可以吧镜像push到仓库当中去。

- Create Dockerfile
- Build the container
  **sudo docker build -t simpleapp**
- Verify the image
  **sudo docker images
  sudo docker run simpleapp**
- Push to the repository
  **sudo docker push**

### 2.9. 管理镜像仓库

我们可以很轻松的把镜像上传到自己的hub里面，但是这些镜像是向世界公开的，所有人都可以访问。有的场景下，我们需要创建一个本地的仓库，把镜像推送到本地仓库。尽管这样会增加管理成本，但是会节省下载带宽，我们还可以让镜像变成私有镜像，且更加安全。

一旦配置了本地的仓库，我们可以使用`docker tag`来打标记，然后使用`push把本地镜像推送到仓库当中去。建议搭建一个没有安全机制的镜像，然后测试，都没有问题得时候，再配置TLS来让仓库更加安全。

### 2.10. 创建一个Deployment

我们使用docker命令推送push和拉取pull镜像，成功之后就可以使用Kubernetes来启动deployment了。在创建deployment的时候使用参数`--image`来指定仓库，然后是应用的名称，然后是版本。

+ 使用`kubectl create`命令来测试镜像

  ```BASH
  $ kubectl create <Deploy-Name> --image=<repo>/<app-name>:<version>
  $ kubectl create time-date --image=10.110.186.162:5000/simpleapp:v2.2
  ```

+ 使用下面的命令来查看Pod的运行状态和所有正在内部运行的容器的状态

  ```BASH
  $ kubectl get pods
  ```

测试这个应用是否能够按照我们的要求运行，可以运行压力测试和质量测试。终止这个Pod看看是否可以很快的恢复

### 2.11. 在容器中运行命令

在容器中运行命令也是测试的一部分。可以运行什么样的命令是由镜像创建的时候所生成的环境所决定的。出于解耦和只安装最小依赖的设计，容器内很有可能没有shell，或者是sh（Bourne shell）来取代bash。容器启动之后，我们可能需要再次访问这个pod并且为生产或者测试添加一些必要的资源。

我们使用`-it`选项来启动一个交互式的shell，这样不用交互式认证。

我们在启动的时候，可能会遇到不止一个容器的pod，这个时候我们需要声明我们要访问的容器名称

```BASH
$ kubectl exec -it <Pod-Name> -- /bin/bash
```

### 2.12. 多容器的Pod

如果想重新创建一个镜像，仅仅为了添加一个shell或者是日志的功能是不太现实的。但是，我们可以在pod里面添加一个容器来提供相关的工具。

pod中的每个容器都应该是临时性的，解耦的。如果添加另外的容器限制了他的扩展性和临时性，我们需要考虑重新设计他了。

pod中的每个容器共享一个IP地址和名称空间。每个容器都有访问分配给Pod的存储的权限。Kubernetes不提供任何的锁机制，所以我们的配置或者应用应该注意不要有写冲突。

当一个容器在写的时候，另外的容器只读。但是我们可以让容器在卷组的不同文件夹来写，或者容器本身有锁机制。没有这些保护机制，我们没法让容器在存储上写文件。

多容器的pod目前有三种使用方式：`ambassador`，`adapter`，`sidecar`。看名字来判断的话，我们可能会以为他们需要额外的配置，实际上并没有。每一种方式都是一个表达式，表达式会告诉另外的pod要做什么。他们都是多容器pod的一种。

+ ambassador
  这种辅助容器是用来和外部资源沟通的，一般来说是外部的集群。他会用到代理，比如Envoy或者其他的代理，我们可以使用这种嵌入式的代理，而不用集群提供给我们的代理。如果你不清楚集群的配置，那么这种方式就非常好用
+ adapter
  这种辅助容器是用来处理主容器生成的数据的。比如说微软版本的ASCII码对于每种用户来说都会生成不用的语言，我们就需要根据用户来生成不同的数据流。
+ sidecar
  就好像摩托车旁边的那个座位一样，这种容器不提供主要的动力，他只是用来多承载一名人员。这种辅助容器提供主要容器中不提供的功能。比如日志的容器就是一个sidecar的典型应用。

### 2.13. readiness和liveness probes（探针）

+ readinessProbe
  通常情况下，我们的容器需要在能够接受请求之前做一下初始化配置的动作，这些动作要在容器ready之前来做。我们在扩展我们的应用的时候，我们的容器在创建之前会处于很多种的状态。我们不是使用客户端来请求应用去判断应用是否ready，我们使用readinessProbe。容器在探针返回健康的状态之前是不会接受任何请求的。
  探针使用`exec`的返回值来判断状态，直到命令返回值是0之前，都不认为容器是ready状态。只要返回值不是0，容器就不会被认为是ready状态，而探针就会一直重试。
  另外的一种探针就是使用HTTP请求的GET方法（httpGet）。使用http header请求的方式来访问路径和端口，如果返回值是200-399之间的数值，那么我们就认为web服务器是ready的。如果返回了其他的值，探针就会重试。
  还有一种TCP Socket套接字探针（tcpSocket）会在预先定义好的端口上打开一个socket，然后过一段时间尝试一次。一旦端口被打开，容器就被认为是ready的。
+ livenessProbe
  就像我们需要等待容器准备好接受请求一样，我们同样想让他一直保持健康状态。有些应用没有内建的检查机制，所以我们就使用livenessPorbes来持续检查容器的健康状态。如果容器在检查的过程中出现了失败的状态，他就会被终止掉。如果有控制器在控制这个pod的话，马上回启动一个新的pod

### 2.14. 验证

由于松耦合，临时性和极大的灵活性，deployment有很多种的组合。每种deployment都有他自己的验证方法。不过使用哪种方法，最终都是实现用户期望的状态。为每一个新创建的deployment创建一个测试用例可以加快开发的进程，减少Kubernetes在集成时候所产生的问题。

为了进一步控制权限，特别是在这种临时性的环境中，确保分布式应用的功能的正确性，开发一些工具是非常好的做法。

虽然客户自己开发的工具可是更容易验证deployment，但是也有一些`kubectl`内建的参数来帮助我们验证。第一个就是`describe`，还有`logs`

我们可以使用`describe`查看资源的详细信息，状态，卷组和时间。下面的`Events`在验证的时候非常有帮助，他可以按照时间顺序显示集群中node的状态和一些日志。我们可以看看`nginx` pod的输出。

```bash
$ kubectl describe pod test1
```

`logs`可以用来查看容器内部的输出。但是不是所有的应用都会生成log，所以单纯看logs没法知道他是应用没有输出还是因为配置错误而没有输出。

```BASH
$ kubectl logs test1
```

## 3 实验

