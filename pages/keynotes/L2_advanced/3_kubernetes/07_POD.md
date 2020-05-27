---
title: POD
keywords: keynotes, L2_advanced, kubernetes, CKA, POD
permalink: keynotes_L2_advanced_3_kubernetes_7_POD.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/7_POD
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- POD基本概念
- POD生命周期
- initContainer
- 容器探针Probe

+ Pod重启的原因

## 1. POD基本概念
我们首先创建一个最基本的pod
``` bash
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
spec:
  containers:
	- name: myapp
	  image: jocatalin/kubernetes-bootcamp:v1
```
Kubernetes编排的主要对象就是编排容器的生命周期。我们不单独和某一个容器去交互，而是使用另外一个最小容器单元Pod进行编排。考虑到Pod内的资源是共享的，我们通常把pod内部设计成只运行一个主要进程。 



### 1.1. POD与微服务

![image-20200420154642789](/pages/keynotes/L2_advanced/3_kubernetes/pics/7_POD/image-20200420154642789.png)



这张图用交通工具描述了传统架构和微服务架构的优缺点

+ 左侧第一张图描述的是我们传统的应用，他的特点是负载量很大，所有的应用都跑在一个非常庞大的操作系统之上。就好像公交车，公交车上载着上班族们去上班。
+ 左侧第二张图描述的是如果这个系统宕机了，我们想要修复他，就需要花费大量的资金去修复整个系统，会有很长的宕机时间，服务的空档期很长，不灵活。如果用公交来比喻他的话，还会引起交通堵塞。
+ 右侧第一张图就是使用的微服务架构来解决这个问题，我们上班使用电动滑板车就可以了。每个人都有自己的电动滑板车。这样就降低了更换的成本，即使出现宕机，也不会影响整个系统的可用性。他的服务对象也非常广泛，任何人都可以使用，也不会引起交通堵塞。
+ 右侧第二张图是说，他的版本可以是不同的版本，满足客户多种需求，即使某一个节点宕机了，也不会影响系统可用性

### 1.2. 多容器的POD

我们一般会在一个pod中部署一个容器，如果有多容器的pod，另外一个容器一般是日志容器。我们把这种辅助容器叫做sidecar。sidecar的作用一般来说都是帮助主容器完成一些任务，而不影响主容器内应用的。那么，官方把这种辅助容器分成了三种

+ sidecar：日志类的辅助容器，当我们需要添加一些主容器中没有的功能的时候，我们会使用sidecar模式，这样既不会改变原有程序的代码，添加一些部署中不需要的东西，我们的容器还可以扩展原有容器的功能，并且保持了解耦和扩展性。比如：filebeats，logstash，fluentd，fluent-bit，exporter
+ ambassador：这有两重含义，一个是指微服务架构上的API gateway模式，比如envoy（Istio的代理），spring cloud gateway或者是更早的Zuul这类微服务治理相关软件所使用的模式，他们不使用kubenetes自身提供的代理功能，而是使用容器内部的代理来转发请求。我们以后讲Istio的时候再说这类微服务治理相关的知识。而另一重含义就是软件 ambassador，他同样是一款微服务治理软件，和Linkerd一样，都是这类软件的后期之秀。
+ adapter：这种容器的作用是修改入栈和出栈的数据来满足我们其他的需求。举例来说，我们有一个非常专业的监控系统，这个监控系统的数据格式要求非常特殊。想获得这种格式化之后的数据，我们通常会借助adapter模式，这是最有效的方式去格式化主容器所产生的监控数据，而不是去修改监控系统本身或者已经容器化的应用。adapter容器用来把多种应用都输出成统一的格式。

### 1.3. 多容器pod的结构

![Pod Network](/pages/keynotes/L2_advanced/3_kubernetes/pics/7_POD/bfd5haac3htu-Kubernetes-Network-Pod-2.png)

多个容器共享同一个共同的**网络名称空间**。共享底层的net，uts和ipc三个网络名称空间，但是，另外三个，互相隔离。user，pid，mnt。这样一来，一个pod内的多个容器，共享主机名之类的名称空间，他更像是一个宿主机上运行了多个虚拟机。一个宿主机上运行了多个虚拟机，这个虚拟机上运行了应用程序，这些虚拟机使用同一个IPC（inter-process communication），或者lo接口，或者共同的文件系统进行通讯。这就是kubernes在组织容器的时候，使用的一个非常精巧的办法，使得我们可以构建较为精细的容器间通讯了。

同一个pod内的容器还共享第二个资源，**存储卷**，假如我们定义了一个存储卷，让第一个容器可以访问，那么，第二个容器也同样可以访问。存储卷不再属于容器，而属于pod。

原生的kubernetes当中，一个pod只有一个IP地址。这个地址由Pause容器来管理的。或者我们也可以理解为所有的pod其实都是多容器pod，只不过有一个默认的容器叫PAUSE。也有组织在研发多个IP地址的pod，目前只有vmware公司在开发具有多个IP地址的POD。不过这种需求不是很多，我们就不详细讨论这个了。

### 1.4. 容器的对外通信

![Container Network](/pages/keynotes/L2_advanced/3_kubernetes/pics/7_POD/w7eqx8n15cor-ContainerNetwork.png)

即使有两个容器，它们共享相同的名称空间和相同的IP地址，这将由kubectl与kube-proxy一起配置。IP地址在容器启动之前分配，会被插入到容器中。容器将有一个类似于**eth0@tun10**的接口。这个IP是根据pod的生命周期而存在的。

endpoint与service同时创建。注意，它使用pod IP地址，但也包括一个端口。该服务使用ipv中的iptables将网络流量从节点高位端口连接到端点。kube-controller管理器处理监视循环，以监视端点和服务以及任何更新或删除的需要。

## 2. POD生命周期

对pod来说，从创建到结束有一个完整的生命周期。在pod创建的时候，有一个初始化的过程，在pod启动之前有一个过程，我们虽然称容器为秒级启动，但是应用程序自身的初始化可能就需要很多时间。

pod内的主容器在运行之前需要做一些环境设定，一般来说，主容器在启动之前的时间，我们可以用来做别的事，比如我们运行另外一个容器，这个容器专门为我们的主容器做环境初始化，这个容器叫initContainer。初始化容器从他的代码开始执行，到执行完毕，他就可以退出了，不会一直存在。而且，初始化容器可以有多个，而多个初始化容器会串行执行。等最后一个初始化容器执行完毕，主容器才会启动。

主容器在执行的时候可能也需要一些初始化，比如entry point，加载配置文件。而主容器在刚刚启动的时候，用户还可以嵌入一些操作，叫post start，这个命令只执行一次，执行完成就退出。在结束之前，也可以做一些操作，我们叫pre stop。无论是启动后，还是结束前的命令，我们称他们为hook，钩子，勾住一些命令来执行。

在整个主容器的运行过程中，我们还有两类操作，health check，他会在post start执行完成之后，会执行liveness probe。我们说过程序运行并不一定代表程序是健康的，也许他早就已经陷入了死循环，不能提供服务，但是他并不会退出。每一个pod内，我们需要对主容器的健康与否和就绪与否做监测，分别叫liveness probe和readiness probe。

![img](/pages/keynotes/L2_advanced/3_kubernetes/pics/7_POD/POD_LIFE.jpg)



### 2.1. POD的运行阶段（phase）

Pod 的 `status` 定义在 `PodStatus` 对象中，其中有一个 `phase` 字段。

Pod 的运行阶段（phase）是 Pod 在其生命周期中的简单宏观概述。该阶段并不是对容器或 Pod 的综合汇总，也不是为了做为综合状态机制。

Pod 相位的数量和含义是严格指定的。除了本文档中列举的内容外，不应该再假定 Pod 有其他的 `phase` 值。

下面是 `phase` 可能的值：

- 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。
- 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。
- 成功（Succeeded）：Pod 中的所有容器都被成功终止，并且不会再重启。
- 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- 未知（Unknown）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

![img](/pages/keynotes/L2_advanced/3_kubernetes/pics/7_POD/u=1753021737,3638443276&fm=26&gp=0.jpg)

### 2.2. POD的存活时间（lifetime）

一般来说，Pod 不会消失，除非人为销毁他们。这可能是一个人或控制器。这个规则的唯一例外是成功或失败的 phase 超过一段时间（由 master 确定）的Pod将过期并被自动销毁。也就是说，如果pod启动失败，控制器会尝试重新启动pod，再失败，再重新启动，而且每次从失败到重新启动的时间间隔，都会比上一次长。最长是5分钟。

有三种可用的控制器：

+ 使用 Job 运行预期会终止的 Pod，例如批量计算。Job 仅适用于重启策略为 OnFailure 或 Never 的 Pod。

+ 对预期不会终止的 Pod 使用 ReplicationController、ReplicaSet 和 Deployment ，例如 Web 服务器。 ReplicationController 仅适用于具有 restartPolicy 为 Always 的 Pod。

+ 提供特定于机器的系统服务，使用 DaemonSet 为每台机器运行一个 Pod 。

所有这三种类型的控制器都包含一个 PodTemplate。建议创建适当的控制器，让它们来创建 Pod，而不是直接自己创建 Pod。这是因为单独的 Pod 在机器故障的情况下没有办法自动复原，而控制器却可以。

如果节点死亡或与集群的其余部分断开连接，则 Kubernetes 将应用一个策略将丢失节点上的所有 Pod 的 phase 设置为 Failed。

## 3. initContainer

https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

Pod 可以包含多个容器，应用运行在这些容器里面，同时 Pod 也可以有一个或多个先于应用容器启动的 Init 容器。Init 容器与普通的容器非常像，除了如下两点：

+ 总是运行到完成，总是顺序执行到结束。
+ 每个都必须在下一个启动之前成功完成。

如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的 restartPolicy 值为 Never，它不会重新启动。

指定容器为 Init 容器，需要在 Pod 的 spec 中添加 initContainers 字段， 该字段內以Container 类型对象数组的形式组织，和应用的 containers 数组同级相邻。 Init 容器的状态在 status.initContainerStatuses 字段中以容器状态数组的格式返回（类似 status.containerStatuses 字段）。

### 3.1. 与普通容器的不同之处

Init 容器支持应用容器的全部字段和特性，包括资源限制、数据卷和安全设置。 但是，Init 容器对资源请求和限制的处理稍有不同。给定Init 容器的执行顺序下，资源使用适用于如下规则：

- 所有 Init 容器上定义的任何特定资源的 limit 或 request 的最大值，作为 Pod *有效初始 request/limit*
- Pod 对资源的有效 limit/request是如下两者的较大者：
  - 所有应用容器对某个资源的 limit/request 之和
  - 对某个资源的有效初始 limit/request
- 基于有效 limit/request 完成调度，这意味着 Init 容器能够为初始化过程预留资源，这些资源在 Pod 生命周期过程中并没有被使用。
- Pod 的 *有效 QoS 层* ，与 Init 容器和应用容器的一样。

配额和限制适用于有效 Pod的 limit/request。 Pod 级别的 cgroups 是基于有效 Pod 的 limit/request，和调度器相同。

同时 Init 容器不支持 Readiness Probe，因为它们必须在 Pod 就绪之前运行完成。

如果为一个 Pod 指定了多个 Init 容器，这些容器会按顺序逐个运行。每个 Init 容器必须运行成功，下一个才能够运行。当所有的 Init 容器运行完成时，Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行。

### 3.2. init容器能做什么？

因为 Init 容器具有与应用容器分离的单独镜像，其启动相关代码具有如下优势：

- Init 容器可以包含一些安装过程中应用容器中不存在的实用工具或个性化代码。例如，没有必要仅为了在安装过程中使用类似 `sed`、 `awk`、 `python` 或 `dig` 这样的工具而去`FROM` 一个镜像来生成一个新的镜像。
- Init 容器可以安全地运行这些工具，避免这些工具导致应用镜像的安全性降低。
- 应用镜像的创建者和部署者可以各自独立工作，而没有必要联合构建一个单独的应用镜像。
- Init 容器能以不同于Pod内应用容器的文件系统视图运行。因此，Init容器可具有访问 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 的权限，而应用容器不能够访问。
- 由于 Init 容器必须在应用容器启动之前运行完成，因此 Init 容器提供了一种机制来阻塞或延迟应用容器的启动，直到满足了一组先决条件。一旦前置条件满足，Pod内的所有的应用容器会并行启动。

### 3.3. 使用 Init 容器

下面的例子定义了一个具有 2 个 Init 容器的简单 Pod。 第一个等待 `myservice` 启动，第二个等待 `mydb` 启动。 一旦这两个 Init容器 都启动完成，Pod 将启动`spec`区域中的应用容器。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

下面的 yaml 文件展示了 `mydb` 和 `myservice` 两个 Service：

``` yaml
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

要启动这个 Pod，可以执行如下命令：

```bash
$ kubectl apply -f myapp.yaml
pod/myapp-pod created
```

要检查其状态：

``` bash
$ kubectl get -f myapp.yaml
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          6m
```

如需更详细的信息：

``` bash
$ kubectl describe -f myapp.yaml
Name:          myapp-pod
Namespace:     default
[...]
Labels:        app=myapp
Status:        Pending
[...]
Init Containers:
  init-myservice:
[...]
    State:         Running
[...]
  init-mydb:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Containers:
  myapp-container:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Events:
  FirstSeen    LastSeen    Count    From                      SubObjectPath                           Type          Reason        Message
  ---------    --------    -----    ----                      -------------                           --------      ------        -------
  16s          16s         1        {default-scheduler }                                              Normal        Scheduled     Successfully assigned myapp-pod to 172.17.4.201
  16s          16s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulling       pulling image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulled        Successfully pulled image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Created       Created container with docker id 5ced34a04634; Security:[seccomp=unconfined]
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Started       Started container with docker id 5ced34a04634
```

如需查看Pod内 Init 容器的日志，请执行:

``` bash
$ kubectl logs myapp-pod -c init-myservice # Inspect the first init container
$ kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

在这一刻，Init 容器将会等待至发现名称为`mydb`和`myservice`的 Service。

创建`mydb`和`myservice`的 service 命令：

``` bash
$ kubectl create -f services.yaml
service "myservice" created
service "mydb" created
```

这样你将能看到这些 Init容器 执行完毕，随后`my-app`的Pod转移进入 Running 状态：

``` bash
$ kubectl get -f myapp.yaml
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          9m
```

一旦我们启动了 `mydb` 和 `myservice` 这两个 Service，我们能够看到 Init 容器完成，并且 `myapp-pod` 被创建

### 3.4. 启动过程

在 Pod 启动过程中，每个Init 容器在网络和数据卷初始化之后会按顺序启动。每个 Init容器 成功退出后才会启动下一个 Init容器。 如果因为运行或退出时失败引发容器启动失败，它会根据 Pod 的 restartPolicy 策略进行重试。 然而，如果 Pod 的 restartPolicy 设置为 Always，Init 容器失败时会使用 restartPolicy 的 OnFailure 策略。

在所有的 Init 容器没有成功之前，Pod 将不会变成 Ready 状态。 Init 容器的端口将不会在 Service 中进行聚集。 正在初始化中的 Pod 处于 Pending 状态，但会将条件 Initializing 设置为 true。

如果 Pod 重启，所有 Init 容器必须重新执行。

对 Init 容器 spec 的修改仅限于容器的 image 字段。 更改 Init 容器的 image 字段，等同于重启该 Pod。

因为 Init 容器可能会被重启、重试或者重新执行，所以 Init 容器的代码应该是幂等的。 特别地，基于 EmptyDirs 写文件的代码，应该对输出文件可能已经存在做好准备。

Init 容器具有应用容器的所有字段。 然而 Kubernetes 禁止使用 readinessProbe，因为 Init 容器不能定义不同于完成（completion）的就绪（readiness）。 这一点会在校验时强制执行。

在 Pod 上使用 activeDeadlineSeconds和在容器上使用 livenessProbe 可以避免 Init 容器一直重复失败。 activeDeadlineSeconds 时间包含了 Init 容器启动的时间。

在 Pod 中的每个应用容器和 Init 容器的名称必须唯一；与任何其它容器共享同一个名称，会在校验时抛出错误。

## 4. 容器探针Probe

### 4.1. 什么是探针

探针Probe是由 kubelet对容器执行的定期诊断。要执行诊断，kubelet 调用由容器实现的 Handler。有三种类型的处理程序：

+ ExecAction：在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
+ TCPSocketAction：对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。
+ HTTPGetAction：对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态码大于等于200 且小于 400，则诊断被认为是成功的。

每次探测都将获得以下三种结果之一：

- 成功：容器通过了诊断。
- 失败：容器未通过诊断。
- 未知：诊断失败，因此不会采取任何行动。

Kubelet 可以选择是否执行在容器上运行的三种探针执行和做出反应：

- `livenessProbe`：指示容器是否正在运行。如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到`重启策略`的影响。如果容器不提供存活探针，则默认状态为 `Success`。
- `readinessProbe`：指示容器是否准备好服务请求。如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 `Failure`。如果容器不提供就绪探针，则默认状态为 `Success`。
- `startupProbe`: 指示容器中的应用是否已经启动。如果提供了启动探测(startup probe)，则禁用所有其他探测，直到它成功为止。如果启动探测失败，kubelet 将杀死容器，容器服从其重启策略进行重启。如果容器没有提供启动探测，则默认状态为成功`Success`。

PodSpec 中有一个 `restartPolicy` 字段，可能的值为 Always、OnFailure 和 Never。默认为 Always。 `restartPolicy` 适用于 Pod 中的所有容器。`restartPolicy` 仅指通过同一节点上的 kubelet 重新启动容器。失败的容器由 kubelet 以五分钟为上限的指数退避延迟（10秒，20秒，40秒…）重新启动，并在成功执行十分钟后重置。Pod一旦绑定到一个节点，他将永远不会重新绑定到另一个节点。

### 4.2. 该什么时候使用探针?

+ liveness probe

  如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活探针; kubelet 将根据 Pod 的`restartPolicy` 自动执行正确的操作。

  如果您希望容器在探测失败时被杀死并重新启动，那么请指定一个存活探针，并指定`restartPolicy` 为 Always 或 OnFailure。

+ readiness probe

  如果要仅在探测成功时才开始向 Pod 发送流量，请指定就绪探针。在这种情况下，就绪探针可能与存活探针相同，但是 spec 中的就绪探针的存在意味着 Pod 将在没有接收到任何流量的情况下启动，并且只有在探针探测成功后才开始接收流量。

  如果希望容器能够自行维护，您可以指定一个就绪探针，该探针检查与存活探针不同的端点。

  需要注意的是，如果只想在 Pod 被删除时能够拒绝连接（drain request），则不一定需要使用就绪探针；在删除 Pod 时，Pod 会自动将自身置于未完成状态，无论就绪探针是否存在。当等待 Pod 中的容器停止时，Pod 仍处于未完成状态。

+ startup probe（1.16 alpha）

### 4.3. 定义探针

+ 定义存活的命令（liveness）

  许多长时间运行的应用程序最终会过渡到断开的状态，除非重新启动，否则无法恢复。Kubernetes 提供了存活探测器来发现并补救这种情况。

  在这篇练习中，会创建一个 Pod，其中运行一个基于 `k8s.gcr.io/busybox` 镜像的容器。下面是这个 Pod 的配置文件。

  ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      test: liveness
    name: liveness-exec
  spec:
    containers:
    - name: liveness
      image: k8s.gcr.io/busybox
      args:
      - /bin/sh
      - -c
      - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
      livenessProbe:
        exec:
          command:
          - cat
          - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
  ```

  在这个配置文件中，可以看到 Pod 中只有一个容器。`periodSeconds` 字段指定了 kubelet 应该每 5 秒执行一次存活探测。`initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 5 秒。kubelet 在容器内执行命令 `cat /tmp/healthy` 来进行探测。如果命令执行成功并且返回值为 0，kubelet 就会认为这个容器是健康存活的。如果这个命令返回非 0 值，kubelet 会杀死这个容器并重新启动它。

  当容器启动时，执行如下的命令：

  ``` bash
  /bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
  ```

  这个容器生命的前 30 秒， `/tmp/healthy` 文件是存在的。所以在这最开始的 30 秒内，执行命令 `cat /tmp/healthy` 会返回成功码。30 秒之后，执行命令 `cat /tmp/healthy` 就会返回失败码。

  创建 Pod：

  ``` bash
  $ kubectl apply -f https://k8s.io/examples/pods/probe/exec-liveness.yaml
  ```

  在 30 秒内，查看 Pod 的事件：

  ``` bash
  $ kubectl describe pod liveness-exec
  ```

  输出结果显示还没有存活探测器失败：

  ``` bash
  FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
  --------- --------    -----   ----            -------------           --------    ------      -------
  24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
  23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
  23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
  23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
  23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
  ```

  35 秒之后，再来看 Pod 的事件：

  ``` bash
  $ kubectl describe pod liveness-exec
  ```

  在输出结果的最下面，有信息显示存活探测器失败了，这个容器被杀死并且被重建了。

  ``` bash
  FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
  --------- --------    -----   ----            -------------           --------    ------      -------
  37s       37s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
  36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
  36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
  36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
  36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
  2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  ```

  再等另外 30 秒，检查看这个容器被重启了：

  ``` bash
  $ kubectl get pod liveness-exec
  ```

  输出结果显示 `RESTARTS` 的值增加了 1。

  ``` bash
  NAME            READY     STATUS    RESTARTS   AGE
  liveness-exec   1/1       Running   1          1m
  ```

+ 定义就绪探针readiness

  有时候，应用程序会暂时性的不能提供通信服务。例如，应用程序在启动时可能需要加载很大的数据或配置文件，或是启动后要依赖等待外部服务。在这种情况下，既不想杀死应用程序，也不想给它发送请求。Kubernetes 提供了就绪探测器来发现并缓解这些情况。容器所在 Pod 上报还未就绪的信息，并且不接受通过 Kubernetes Service 的流量。

  就绪探测器的配置和存活探测器的配置相似。唯一区别就是要使用 `readinessProbe` 字段，而不是 `livenessProbe` 字段。

  ``` yaml
  readinessProbe:
    exec:
      command:
      - cat
      - /tmp/healthy
    initialDelaySeconds: 5
    periodSeconds: 5
  ```

  HTTP 和 TCP 的就绪探测器配置也和存活探测器的配置一样的。

  就绪和存活探测可以在同一个容器上并行使用。两者都用可以确保流量不会发给还没有准备好的容器，并且容器会在它们失败的时候被重新启动。

### 4.4. Probe的配置选项

**探测器**有很多配置字段，可以使用这些字段精确的控制存活和就绪检测的行为：

- `initialDelaySeconds`：容器启动后要等待多少秒后存活和就绪探测器才被初始化，默认是 0 秒，最小值是 0。
- `periodSeconds`：执行探测的时间间隔（单位是秒）。默认是 10 秒。最小值是 1。
- `timeoutSeconds`：探测的超时后等待多少秒。默认值是 1 秒。最小值是 1。
- `successThreshold`：探测器在失败后，被视为成功的最小连续成功数。默认值是 1。存活探测的这个值必须是 1。最小值是 1。
- `failureThreshold`：当 Pod 启动了并且探测到失败，Kubernetes 的重试次数。存活探测情况下的放弃就意味着重新启动容器。就绪探测情况下的放弃 Pod 会被打上未就绪的标签。默认值是 3。最小值是 1。

**HTTP 探测器**可以在 `httpGet` 上配置额外的字段：

- `host`：连接使用的主机名，默认是 Pod 的 IP。也可以在 HTTP 头中设置 “Host” 来代替。
- `scheme` ：用于设置连接主机的方式（HTTP 还是 HTTPS）。默认是 HTTP。
- `path`：访问 HTTP 服务的路径。
- `httpHeaders`：请求中自定义的 HTTP 头。HTTP 头字段允许重复。
- `port`：访问容器的端口号或者端口名。如果数字必须在 1 ～ 65535 之间。

对于 HTTP 探测，kubelet 发送一个 HTTP 请求到指定的路径和端口来执行检测。除非 `httpGet` 中的 `host` 字段设置了，否则 kubelet 默认是给 Pod 的 IP 地址发送探测。如果 `scheme` 字段设置为了 `HTTPS`，kubelet 会跳过证书验证发送 HTTPS 请求。大多数情况下，不需要设置`host` 字段。这里有个需要设置 `host` 字段的场景，假设容器监听 127.0.0.1，并且 Pod 的 `hostNetwork` 字段设置为了 `true`。那么 `httpGet` 中的 `host` 字段应该设置为 127.0.0.1。可能更常见的情况是如果 Pod 依赖虚拟主机，你不应该设置 `host` 字段，而是应该在 `httpHeaders` 中设置 `Host`。

对于一次 TCP 探测，kubelet 在节点上（不是在 Pod 里面）建立探测连接，这意味着你不能在 `host` 参数上配置 service name，因为 kubelet 不能解析 service name。

## 5. Pod readinessgGate（1.14 stable）

可以把外部的回复信息或者信号注入到我们的应用当中，使用Pod readiness。我们需要在Pod的spec文件中定义`readinessGates`，并且在其中定义一些条件，供kubelet来判定pod是否就绪。

readinessGates会根据`status.condition`的状态来判定，如果kubernetes不能够在pod中找到`status.conditons`中的字段，而`condition`的默认状态是`False`

例如：

``` yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # a built in PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # an extra PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

## 6. Pod 重启的原因

Pod重启导致 Init 容器重新执行，主要有如下几个原因：

- 用户更新 Pod 的 Spec 导致 Init 容器镜像发生改变。Init 容器镜像的变更会引起 Pod 重启. 应用容器镜像的变更仅会重启应用容器。
- Pod 的基础设施容器 (译者注：如 pause 容器) 被重启。 这种情况不多见，必须由具备 root 权限访问 Node 的人员来完成。
- 当 `restartPolicy` 设置为 Always，Pod 中所有容器会终止而强制重启，由于垃圾收集导致 Init 容器的完成记录丢失。

你可以在Pod的规格信息中与containers数组同级的位置指定 Init 容器。