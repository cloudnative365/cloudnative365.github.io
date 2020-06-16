---
title: 日志和错误定位
keywords: keynotes, advanced, kubernetes, CKA, LOGGING_AND_TROUBLESHOOTING
permalink: keynotes_L2_advanced_3_kubernetes_17_LOGGING_AND_TROUBLESHOOTING.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/17_LOGGING_AND_TROUBLESHOOTING
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- kubernetes目前并没有集成记录日志的功能
- 知道使用哪个产品可以来收集日志
- 了解最基础的错误定位的流程
- 讨论怎样使用sidecar在pod内记录日志

## 1. 简介

Kubernetes依赖于API调用，所以对网络问题非常敏感。标准的Linux工具和进程是定位集群故障的最佳方法。如果一个shell（如bash）在有问题的Pod中不可用，请考虑使用一个shell部署另一个类似的Pod，如**busybox**。DNS配置文件和诸如**dig**之类的工具是一个很好的定位问题的方式。对于更困难的挑战，您可能需要安装其他工具，如**tcpdump**。

大而多样的workload可能很难跟踪，因此对使用情况的监视是必不可少的。监视是收集关键指标，如CPU、内存和磁盘使用率，以及节点上的网络带宽，以及监视应用程序中的关键指标。这些特性会被Metric服务器采集到Kubernetes中，Metric服务器是现在被弃用的Heapster的精简版本。一旦安装，Metrics服务器就会公开一个标准API，其他agent（比如autoscaler）可以使用它。一旦安装，这个endpoint可以在主服务器上找到：**/apis/metrics/k8s.io/**。

记录所有节点的活动是Kubernetes之外的另一个特性。使用Fluentd可以成为统一日志记录层的有用数据收集器。拥有聚合的日志可以帮助我们将问题可视化，并提供搜索所有日志的能力。当本地网络故障排除没有暴露根本原因时，我们就可以通过日志追溯问题的根源。

CNCF的另一个项目结合了日志记录、监视和警报，被称为prometheus-您可以从[[prometheus网站](https://prometheus.io/)]了解更多。它提供一个时间序列数据库，以及与Grafana的集成，用于可视化和dashboard。

我们将回顾一些基本的**kubectl**命令，我们可以使用这些命令来调试正在发生的事情，我们将引导大家完成基本步骤，以便能够调试您的容器、pending的容器以及Kubernetes中的系统。

## 2. 最基本的定位问题的步骤

故障排除流程应该从最简单的地方入手。如果命令行中有错误，请先进行调查。问题的症状可能会决定下一步的检查手段。从运行在容器中的应用程序入手，然后再去调查集群是个非常普遍的方式。应用程序的镜像内可能有一个可以使用的shell，例如：

```bash
$ kubectl create deploy busybox --image=busybox --command sleep 3600

$ kubectl exec -ti <busybox_pod> -- /bin/sh 
```

Security settings can also be a challenge. RBAC, covered in the security chapter, provides mandatory or discretionary access control in a granular manner. SELinux and AppArmor are also common issues, especially with network-centric applications. 

A newer feature of Kubernetes is the ability to enable auditing for the kube-apiserver, which can allow a view into actions after the API call has been accepted.

The issues found with a decoupled system like Kubernetes are similar to those of a traditional datacenter, plus the added layers of Kubernetes controllers: 

如果Pod正在运行，请使用**kubectl logs Pod name**从容器外部查看日志。如果没有日志，可以考虑在Pod中部署一个sidecar容器来生成和处理日志。下一个要检查的地方是网络，包括DNS、防火墙和标准的连接，这个时候就使用标准的Linux命令和工具。

安全设置可能是一个问题。安全章节中介绍的RBAC以细粒度的方式提供强制或自主访问控制。SELinux和AppArmor也是常见的问题，特别是在以网络为中心的应用程序中。

Kubernetes的一个新特性是能够为kube apiserver启用审计，这允许在接受API调用后查看操作。

像Kubernetes这样的解耦系统发现的问题与传统数据中心的问题类似，但是需要考虑Kubernetes控制器的问题：

- 命令行中报出的错误
- Pod日志和Pod状态
- 使用Shell来定位Pod的DNS和网络问题
- 检查节点的错误日志，确保有足够的资源用于新Pod的分配
- RBAC, SELinux 或者 AppArmor 等安全配置
- 控制器和kube-apiserver的API调用是否正常
- 打开审计功能
- 内部node之间的网络问题，DNS问题和防火墙问题
- Master节点的控制器（kube-controller是不是出于pending或者error状态，检查日志文件中的错误，冲突的资源等等）

## 3. Ephemeral Containers短生命周期容器

1.16版本的一个新特性是能够将容器添加到正在运行的pod中。这将允许一个有各种功能的容器被添加到现有的pod中，而不必终止和重新创建。间歇性和难以确定的问题可能需要一段时间才能重现，或者添加完容器之后问题就消失了。

作为alpha稳定性特征，它可以随时更改或删除。此外，它们不会自动重新启动，而且有很多资源是没办法使用的，比如端口。

这些容器是通过**ephemeralcontainers**处理程序通过API调用添加的，而不是通过**podSpec**添加的。因此，无法使用**kubectl edit**。

您可以使用**kubectl attach**命令加入容器中的现有进程。这有助于替代执行新进程的**kubectl exec**。附加进程的功能完全取决于您附加到的是什么。

```bash
kubectl debug buggypod --image debian --attach
```

## 4. 集群启动的顺序

如果使用**kubeadm**构建集群，则集群启动序列以systemd开头。其他工具可能利用不同的方法。使用**系统控制状态kubelet.服务**查看用于运行kubelet二进制文件的当前状态和配置文件。

- 使用**/etc/systemd/system/kubelet.service.d/10-kubeadm.conf**

  在**config.yaml**文件您将找到二进制文件的几个设置，包括**staticPodPath**它指示kubelet将读取每个yaml文件并启动每个pod的目录。如果您将一个yaml文件放在这个目录中，这是一种排除调度器故障的方法，因为pod是用对调度器的任何请求创建的。

- 使用**/var/lib/kubelet/config.yaml**配置文件

- **staticPodPath**设置为**/etc/kubernetes/manifests/**

四个默认的yaml文件将启动运行集群所需的基本pod：

+ kubelet的目录中：kube apiserver、etcd、kube controller manager、kube scheduler中的*.yaml创建所有pods。

一旦kube controller manager中的监视循环和控制器使用etcd数据运行，将创建其余配置对象。

## 5. 监控监控是从基础设施和应用程序收集度量。

监控是从基础设施和应用程序收集metrics。

从前一直在使用的Heapster已被集成的Metrics-server所取代。一旦安装并配置好，服务器就会暴露一个标准API，其他agent可以使用它来确定集群的使用情况。我们还可以自定义metrics，然后自动扩展控制器也可以使用这些度量来确定是否应执行操作。

## 6. Plugin

我们一直在使用**kubectl**命令。基本命令可以以更复杂的方式一起使用，以扩展可以执行的操作。有超过70个和不断增长的插件可用于与Kubernetes对象和组件交互。

在编写本课程时，插件不能覆盖现有的**kubectl**命令，也不能向现有命令添加子命令。编写新插件应该考虑命令行运行时包和插件作者的Go库。

作为插件，要使用的名称空间或容器等选项的声明必须在命令之后。

```bash
$ kubectl sniff bigpod-abcd-123 -c mainapp -n accounting
```

插件可以通过多种方式来发布。我们可以使用krew（kubectl的插件管理命令）跨平台打包并且帮我们找到有用的插件。

安装krew请参考 [krew's GitHub repository](https://github.com/kubernetes-sigs/krew/).

```bash
$ kubectl krew help
```

安装好以后就可以通过kubectl krew来使用了

```bash
kubectl krew [command]...
```

Usage:

```bash
krew [command]
```

### 6.1. 可用的命令

| COMMAND       | DESCRIPTION                                 |
| :------------ | :------------------------------------------ |
| **help**      | Help about any command                      |
| **info**      | Show information about kubectl plugin       |
| **install**   | Install kubectl plugins                     |
| **list**      | List installed kubectl plugins              |
| **search**    | Discover kubectl plugins                    |
| **uninstall** | Uninstall plugins                           |
| **update**    | Update the local copy of the plugin index   |
| **upgrade**   | Upgrade installed plugins to newer versions |
| **version**   | Show krew version and diagnostics           |

### 6.2. 管理插件

使用help选项可以查看基础的操作。安装完成之后要保证$PATH目录中有插件，krew就可以轻松愉快的使用了

```
$ export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
$ kubectl krew search

NAME              DESCRIPTION                                           INSTALLED
access-matrix     Show an RBAC access matrix for server resources       no
advise-psp        Suggests PodSecurityPolicies for cluster.             no
....

$ kubectl krew install tail

Updated the local copy of plugin index.
Installing plugin: tail
Installed plugin: tail
\
 | Use this plugin:

....

 | | Usage:
 | |
 | | # match all pods
 | | $ kubectl tail
 | |
 | | # match pods in the 'frontend' namespace
 | | $ kubectl tail --ns staging
....
```

列出当前所有的插件

```bash
kubectl plugin list
```

查找新的插件

```bash
kubectl krew search
```

安装新的插件

```bash
kubectl krew install new-plugin
```

安装完成之后的插件还可以随时的更新和卸载

### 6.3. 使用Wireshark嗅探网络

群集网络通信被加密，使得可能的网络问题的故障排除变得更加复杂。使用sniff插件，您可以从内部查看流量。sniff需要Wireshark和导出图形显示的能力。

**sniff**命令将使用第一个找到的容器，除非我们传递**-c**选项来声明pod中要用于流量监视的容器。

```bash
$ kubectl krew install sniff nginx-123456-abcd -c webcont
```

## 7. 日志工具

日志记录和监视一样，是其中一个庞大的主题。它有很多工具可以作为我们工具的一部分。

通常，日志在被搜索引擎接收之前在本地收集和聚合，并可以通过正则表达式显示。其实有许多软件堆栈可用于日志记录，[Elasticsearch、Logstash和Kibana堆栈](https://www.elastic.co/videos/introduction-to-the-elk-stack)（ELK）已经很普遍了。

在Kubernetes中，kubelet将容器日志写入本地文件（通过Docker日志驱动程序）。**kubectl logs**命令允许您检索这些日志。

集群范围内，您可以使用[Fluentd](https://www.fluentd.org/)以聚合日志。查看[群集管理日志记录概念](https://kubernetes.io/docs/concepts/cluster-administration/logging/)详细描述。

Fluentd是云计算本地计算基础的一部分，与Prometheus一起，它们为监控和日志记录提供了一个很好的组合。你可以找到[运行Fluentd on Kubernetes的详细介绍](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/)在Kubernetes文档中。

