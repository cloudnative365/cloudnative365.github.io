---
title: Helm入门
keywords: keynotes, advanced, kubernetes, CKA, HELM
permalink: keynotes_L2_advanced_3_kubernetes_19_HELM.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/19_HELM
typora-root-url: ../../../../../cloudnative365.github.io
---

## 学习目标

- 使用Helm简单的部署Kubernetes的deployment
- 了解Chart模板是怎样描述应用应该如何部署的
- 讨论Tiller是怎样技术Chart创建Deployment的
- 在集群中初始化一个Helm

## 1. 简介

我们使用了Kubernetes工具来部署简单的Docker应用程序。从v1.4版本开始，我们的目标是为软件提供一个标准的规范。Helm类似于**yum**或**apt**这样的包管理器，Chart类似于包。而Helm v3与v2有着显著的不同。
典型的容器化应用程序会包含很多清单。Deployment、Service和ConfigMap的清单。我们可能还会创建一些Secret、Ingress和其他对象。每一个都需要一份清单。
有了Helm，你可以打包所有这些清单文件，并使它们作为一个单独的tar包使用。我们可以将tar包放在一个存储库中，搜索该存储库，发现一个应用程序，然后使用一个命令部署并启动整个应用程序。
服务器运行在Kubernetes集群中，而客户机是本地的，甚至是本地笔记本电脑。使用客户机，您可以连接到多个应用程序存储库。
我们还可以从命令行轻松地升级或回滚应用程序。

## 2. Helm的两个版本

### 2.1. Helm v2和Tiller

helm工具使用一系列YAML文件将Kubernetes应用程序打包成一个chart或包。这中方式允许用户之间简单的共享，使用模板方案进行优化，以及来源跟踪等。

![zgxbwiuexxtg-BasicFlow-HelmandTiller](/pages/keynotes/L2_advanced/3_kubernetes/pics/19_HELM/zgxbwiuexxtg-BasicFlow-HelmandTiller.png)

#### 基础的Helm和Tiller的流程

Helm的v2版本是由两个组件构成的

+ 服务器端叫Tiller，运行在kubernetes集群之中
+ 客户端叫Helm，在我们本地的机器上运行

Helm v2会在集群中部署一个Tiller的pod。这样会引发很多安全和集群权限的问题。而Helm v3就不用部署这个pod了.

使用Helm客户端，您可以浏览包存储库（包含已发布的Chart），并在Kubernetes集群上部署这些Chart。kubernetes将下载Chart并将请求传递给Tiller来创建应用。这个应用是由运行在Kubernetes集群中的各种资源组成。

### 2.2. Helm v3

最近Helm彻底的翻修了一遍，流程和命令都有很大的改变。如果我们正在使用Helm的v2版本，那么可能需要花些时间去升级和集成这些改变。

最显著的变化之一就是Tiller的移除。这是一个一直存在的安全问题，因为pod需要提升权限才能部署Chart。在新版本中，这个功能被单独放在了命令行中，不再需要初始化才能使用。

在v2中，对chart和deployment的更新需要使用双向策略合并来完成。这需要将先前的manifest和预期的menifest进行比较，而不是在**helm**命令之外进行编辑。目前的检查是使用另外的方法，检查目前对象的状态。

还有其他的改变，比如软件安装不再自动生成名称。我们必须手动指定名称，要不就传递`--generated name`参数

## 3. Chart

### 3.1. Chart中的内容

**Chart**是组成分布式应用程序的Kubernetes资源清单的存档集，您可以从[Helm 3文档](https://helm.sh/docs/topics/charts)中了解更多信息。我们还可以使用其他的方式轻松地创建，例如由供应商提供软件。Chart类似于独立的yum存储库的使用。

**├── Chart.yaml
├── README.md
├── templates
│  ├── NOTES.txt
│  ├── _helpers.tpl
│  ├── configmap.yaml
│  ├── deployment.yaml
│  ├── pvc.yaml
│  ├── secrets.yaml
│  └── svc.yaml
└── values.yaml**

+ Chart.yaml

**Chart.yaml**文件包含很多Chart的元数据，比如name, version, keywords等等

+ values.yaml

**values.yaml**文件包含了k-v，用来生成集群中的某个release。这些值会被资源清单中的值所替代，他使用的是Go的模板语法。

+ templates

**templates**文件夹包含了创建这个MariaDB应用所需的所有的清单文件

### 3.2. 模板

模板是使用Go模板化语法的资源清单。例如，在**values**文件中定义的变量在创建发布时被注入到模板中。在我们提供的MariaDB示例中，数据库密码存储在Kubernetes secret中，数据库配置存储在Kubernetes ConfigMap中。

我们可以看到，在Secret metadat中使用chart名称、Release名称等定义了一组标签。而实际的值都是来自于**values.yaml**文件。

``` yaml
apiVersion: v1
kind: Secret
metadata:
    name: {{ template "fullname" . }}
    labels:
        app: {{ template "fullname" . }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        release: "{{ .Release.Name }}"
        heritage: "{{ .Release.Service }}"
type: Opaque
data:
    mariadb-root-password: {{ default "" .Values.mariadbRootPassword | b64enc | quote }}
    mariadb-password: {{ default "" .Values.mariadbPassword | b64enc | quote }}
```

## 4. 初始化Helm v2

Helm v3是不需要执行这个操作的

我们可以从源代码中构建Helm，或者下载tar包。我们希望很快能看到稳定版本的Linux包（yum或者apt）。当前部署helm的RBAC安全需求需要创建一个新的**serviceaccount**，并分配权限和角色。有几个可选的设置可以传递给**helm init**命令，通常是针对特定的安全问题、存储选项和试运行选项。

```bash
$ helm init
...
Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!

$ kubectl get deployments --namespace=kube-system
NAMESPACE    NAME           READY  UP-TO-DATE  AVAILABLE  AGE
kube-system  tiller-deploy  1/1    1           1          15s
```

helm v2在初始化的过程当中会在集群当中创建一个叫做**tiller-deploy**的pod。这个会在kube-system的名称空间中创建一个deployment

客户端需要通过pod转发和tiller的pod进行通信。因此，我们不会看到任何的service来暴露tiller

### 4.1. Chart的仓库

初始化helm时包含默认存储库，但添加其他存储库是常见的。存储库目前是简单的HTTP服务器，包含一个索引文件和所有Chart的tar包。

您可以使用**helm repo**命令与存储库交互。

```bash
$ helm repo add testing http://storage.googleapis.com/kubernetes-charts-testing

$ helm repo list
NAME URL
stable http://storage.googleapis.com/kubernetes-charts
local http://localhost:8879/charts
testing http://storage.googleapis.com/kubernetes-charts...
```

一旦仓库是可用的，我们就可以使用关键字来搜索Chart。下面我们来搜索redis的Chart

```bash
$ helm search redis
WARNING: Deprecated index file format. Try 'helm repo update'
NAME                     VERSION DESCRIPTION
testing/redis-cluster    0.0.5   Highly available Redis cluster with multiple se...
testing/redis-standalone 0.0.1   Standalone Redis Master testing/...
```

如果我们在仓库中找到对应的Chart，我们就可以部署了。

### 4.2. Deploying a Chart

要部署Chart，只需使用**helm install**命令。安装成功可能需要几个资源，例如与chart PVC匹配的可用pv。目前，发现哪些资源需要存在的唯一方法是读取每个图表的**READMEs**：

```bash
$ helm install testing/redis-standalone
Fetched testing/redis-standalone to redis-standalone-0.0.1.tgz
amber-eel
Last Deployed: Fri Oct 21 12:24:01 2016
Namespace: default
Status: DEPLOYED

Resources:
==> v1/ReplicationController
NAME             DESIRED CURRENT READY AGE
redis-standalone 1       1       0     1s

==> v1/Service
NAME  CLUSTER-IP EXTERNAL-IP PORT(S)  AGE
redis 10.0.81.67 <none>      6379/TCP 0s

```

我们还可以列出可用版本，删除，或者升级和回滚

```bash
$ helm list
NAME      REVISION UPDATED                  STATUS   CHART
amber-eel 1        Fri Oct 21 12:24:01 2016 DEPLOYED redis-standalone-0.0.1
```

系统将为部署的每个helm实例创建一个独特的、多样的名称。您还可以使用kubectl查看集群中创建的新资源Helm。

应仔细审查部署的结果。它通常包含有关访问内部应用程序的信息。如果集群中没有所需的集群，则输出通常是开始故障排除的第一步。