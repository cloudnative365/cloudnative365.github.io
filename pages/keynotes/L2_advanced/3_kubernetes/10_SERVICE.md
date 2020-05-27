---
title: service
keywords: keynotes, L2_advanced, kubernetes, CKA, SERVICE
permalink: keynotes_L2_advanced_3_kubernetes_10_SERVICE.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/10_SERVICE
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- 学习service工作原理
- 暴露应用
- 讨论service的类型
- 启动一个本地的proxy
- 使用集群的DNS

## 1. 简介
kubernetes的架构是通过一系列无状态的，解耦的对象组合在一起的。而Service就是把这些pod连接在一起的组件，就好像我们前面说的班主任，他还可以提供对外访问的接口，并且有探活功能，知道后端的pod哪个可能被终止了，或者重建了。这些功能都是使用Label实现的，因此重建后的Pod也可以重新注册到Service之上，后端服务通过Endpoint对象向外提供资源。Google在研究ESP（Extensible Service Proxy），这是一款基于nginx HTTP反向代理的代理服务，他能够提供比Endpoint更加便捷和强大的功能，但是ESP还没有和Google App引擎或者GKE环境完美的结合。

service可以多种类型并存，并且可以根据需要添加更多的类型。每种服务都可以向内或者向外暴露端口。而每个service都可以连接内部或者外部资源，比如可以连接第三方的数据库。

kube-proxy会一直循环监控kube-api，一旦有新的service或者endpoint在节点上创立，就会打开一个随机的端口，并且监听在`ClusterIP:Port`之上，并且把流量随机的导向一个后端的endpoint之上。

service通过查询匹配的标签，会向后端提供自动的负载均衡功能。他还提供了session亲和性功能，可以把流量都指向一个IP地址。同时，还可以配置headless service无头服务，没有固定IP的负载均衡。

获得的IP地址会注册在etcd中，以保证唯一性，Service使用iptables来路由服务，但是很有可能在未来的版本出现新的组件，来提供连接到资源的功能。

## 2. service升级时候的匹配模式

service通过标签来选择谁会做为自己的后端。我们知道，对象的标签是可以动态更新的，这样的话，也可能影响到连接到service的Pods。

默认的升级是匹配滚动更新的，随着新pod的创建，即使存在一个应用的不同版本，由于自动负载均衡，流量依然会被导向之前版本的应用。

如果部署的应用程序存在差异，以致客户端在与不同版本通信时出现问题，则可以考虑为部署设置一个更具体的标签，其中包括版本号。当部署为更新创建新的replication controller时，标签将不匹配。一旦创建了新的pod，并且完全初始化完成，我们再编辑service连接的标签。流量将转移到新的就绪版本，从而最大限度地减少客户机版本混淆。

## 3. 通过service访问某个应用

使用kubectl创建一个service

``` bash
$ kubectl expose deployment/nginx --port=80 --type=NodePort


$ kubectl get svc
NAME        TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)        AGE
kubernetes  ClusterIP  10.0.0.1    <none>       443/TCP        18h
nginx       NodePort   10.0.0.112  <none>       80:31230/TCP   5s

$ kubectl get svc nginx -o yaml
apiVersion: v1
kind: Service
...
spec:
    clusterIP: 10.0.0.112
    ports:
    - nodePort: 31230
...
```

Open browser http://Public-IP:31230.

`kubectl expose`命令为nginx deployment创建了一个服务。这个服务使用80端口，并且在所有的节点上都启动了一个随机的端口。我们也可以手动指定port和targetPort。targetPort默认和port一直，但是我们也可以给他别的值，可以是后端pod上的任何一个端口。每个pod都可以有不同的端口，这不会影响网络流量。但是把流量切换到另外的端口会让我们的客户端需要重新配置端口，比如说更换软件的版本时候。

`kubectl get svc`命令会列出所有的service，我们可以看到nginx服务，他是使用内部的clusterIP模式创建的。

cluster IPs的范围和NodePort的随机地址段都是在API server中配置的（启动选项中）。

service还可以指向在不同名称空间中的另外一个service，甚至是集群外的资源，比如说，不在kubernetes当中运行的传统资源（使用ExternalName）

## 4. Service的类型

+ ClusterIP

这个是Service的默认模式，这种模式下，只能提供向内部访问的接口（除非我们手动创建一个外部的endpoint）。他的地址范围是API server的启动选项中所定义的。

+ NodePort

NodePort方式方便我们找出问题，但是如果我们想要一个静态的IP，比如说在防火墙上打开某个端口。NodePort的范围需要在集群的配置当中定义。

+ LoadBalancer

这种类型是为了能够把请求传递给公有云提供商，比如GKE或者AWS。私有云如果想使用这个类型需要额外的插件，比如CloudStack和OpenStack。即使没有公有云，地址还是公开的，并且可以把请求转发到同一个deployment中的所有pod上。

+ ExternalName

A newer service is ExternalName, which is a bit different. It has no selectors, nor does it define ports or endpoints. It allows the return of an alias to an external service. The redirection happens at the DNS level, not via a proxy or forward. This object can be useful for services not yet brought into the Kubernetes cluster. A simple change of the type in the future would redirect traffic to the internal objects.

这个类型和其他的有一点不同，他没有selector，也不需要定义port或者endpoint。他允许暴露一个DNS的别名给外部的服务。这个是由DNS来做的，不是代理或者转发。这种方式在暴露不由kubernetes集群管理的对象的时候非常有用。只需要改动类型，就可以把请求直接指向内部的对象。

## 5. Service的代理模式

在节点上运行的kube-proxy时刻监控着API server上的service资源。除了**ExternalName**之外的其他资源。这个模式尽管Kubernetes的版本有更新，但是一直在沿用。

在1.0版本，service使用的是基于IP的TCP/UDP 4层的用户空间模式。

![Services overview diagram for userspace proxy](/pages/keynotes/L2_advanced/3_kubernetes/pics/10_SERVICE/services-userspace-overview.svg)

在1.1版本之后，变成了由iptables来代理的模式，在1.2就成为了默认的模式。

![Services overview diagram for iptables proxy](/pages/keynotes/L2_advanced/3_kubernetes/pics/10_SERVICE/services-iptables-overview.svg)

在iptables的模式中，kube-proxy会持续监控着API server中Service的变动和endpoint对象，任何一个对象有创建或者删除的话，他就会更新规则。新模式的一个限制是，如果原始请求失败，将无法连接到Pod，因此它使用就绪性探测来确保连接前所有容器都正常工作。此模式允许最多5000个节点。假设每个节点有多个服务和pod，这会导致内核出现瓶颈。

在1.9版本后，iptables换成了ipvs

![Services overview diagram for IPVS proxy](/pages/keynotes/L2_advanced/3_kubernetes/pics/10_SERVICE/services-ipvs-overview.svg)

这个时候虽然是在beta版中，并且可能会发生变化，但它在内核空间中的工作速度更快，并且可配置负载平衡算法，例如循环、最短延迟、最少连接和其他一些算法。这对大型集群很有帮助，大大超过了以前5000个节点的限制。此模式假定IPV内核模块在kube代理之前安装并运行。

想更改kube-proxy的模式，就需要在初始化的时候添加一个参数`mode=iptables`或者使用ipvs或者userspace，默认使用的是ipvs，当然，要系统支持，如果不支持，有可能会降级为iptables

## 6. 使用本地的代理

在开发应用程序或服务时，检查服务的一个快速方法是使用**kubectl**运行本地代理。它会捕获shell，除非你把它放在后台中运行。运行时，可以在本地主机上调用Kubernetes API，也可以在其API URL上访问ClusterIP服务。proxy侦听的IP和端口可以使用命令参数配置。

启动proxy

``` bash
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

如果我们要使用本地的代理访问**ghost**服务，我们就使用下面的地址，比如**http://localhost:8001/api/v1/namespaces/default/services/ghost**. 我们就可以看到他的端口了。

## 7. DNS

在kubernetes的1.13版本之后，DNS服务默认是由CoreDNS来提供的。使用CoreDNS可以让DNS服务更加方便。一旦有容器启动，他就会为对应的区域启动一个server，随时准备为他服务。然后，每个server都可以加载一个或者多个插件来提供其他的功能。这个server是说配置当中的一个段，可以理解为一个正向和反向解析记录。

目前有30多个插件，他们可以实现大部分的功能，开启功能或者自己开发一个功能也是很容易的。

Common插件可以为prometheus提供metrics，错误日志，健康报告，或者配置TLS证书和gRPC服务。

我们可以去[CoreDNS Plugins web page](https://coredns.io/plugins/)来查找更多的插件

## 8. 验证DNS的注册（考试重点）

为了确保DNS的配置是正确的，并且服务也注册到DNS服务器了，最简单的办法就是在集群中运行一个pod，并且执行DNS lookup

我们先创建一个busybox

``` yaml
apiVersion: v1
kind: Pod
metadata:
    name: busybox
    namespace: default
spec:
    containers:
    - image: busybox
      name: busy
      command:
        - sleep
        - "3600"
```

然后使用**kubectl exec** 来 **nslookup**:

``` bash
$ kubectl exec -ti busybox -- nslookup nginx
Server: 10.0.0.10
Address 1: 10.0.0.10
Name: nginx
Address 1: 10.0.0.112
```

我们可以发现DNS记录`nginx`是被注册成为ClusterIP的。

我们还可以通过查看容器的`/etc/resolv.conf`文件来定位问题。

``` bash
$ kubectl exec busybox cat /etc/resolv.conf
```

最后，查看每个`kube-dns`pod中写着W（warning）或者E（error）或者F（failure）的行。并且确保DNS服务是启动的，并且DNS的后端是被暴露出来的。