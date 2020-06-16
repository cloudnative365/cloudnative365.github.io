---
title: 调度
keywords: keynotes, advanced, kubernetes, CKA, SCHEDULING
permalink: keynotes_L2_advanced_3_kubernetes_16_SCHEDULING.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/16_SCHEDULING
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- 学习kube-cheduler如何调度Pod
- 使用标签来管理Pod的调度
- 配置taints和tolerations
- 使用 **podAffinity** 和 **podAntiAffinity**.
- 了解怎样去运行多个调度器

## 1. 调度

### 1.1. kube-scheduler

Kubernetes部署的应用数量越多、越多样化，调度的管理就越是重要。kube-scheduler使用topology-aware算法确定哪些节点将运行Pod。

用户可以设置pod的优先级，这将允许强行驱逐较低优先级的pod。逐出较低优先级的pod之后将安排较高优先级的pod。

调度器跟踪集群中的节点的状态，根据Predicates（预选）过滤它们，然后使用优先级函数来确定每个Pod应该在哪个节点上调度。作为请求的一部分，Pod的spec被发送到节点上的kubelet进行创建。

默认的调度决策可以通过在节点或pod上使用label来决定。podAffinity、tolerations和pod绑定的label允许从pod或节点的角度进行配置。有些，像tolerations，允许Pod在特定节点上运行，即使节点有taints，也不会阻止Pod被调度。

不是所有的标签都是绝对的。Affinity可能会鼓励在节点上部署Pod，但如果节点不可用，则会将Pod部署到其他位置。有时，文档中可能使用术语*require*，但实践表明，设置他更像是一个请求。作为beta特性，预计小地方会有所改变。如果所需条件不再为true，某些设置将从节点中逐出播客，例如**requiredDuringScheduling**、**RequiredDuringExecution**。

其他选项，如自定义调度程序，需要编程并部署到Kubernetes集群中。

### 1.2. Predicates（预选）

调度器通过一组过滤器或predicates来查找可用的节点，然后使用priority（优选）对每个节点进行排序。选择等级最高的节点来运行Pod。

[源代码](https://github.com/kubernetes/kubernetes/blob/323f34858de18b862d43c40b2cced65ad8e24052/pkg/scheduler/framework/plugins/legacy_registry.go)

``` go
func PredicateOrdering() []string {
	return []string{CheckNodeUnschedulablePred,
		GeneralPred, HostNamePred, PodFitsHostPortsPred,
		MatchNodeSelectorPred, PodFitsResourcesPred, NoDiskConflictPred,
		PodToleratesNodeTaintsPred, CheckNodeLabelPresencePred,
		CheckServiceAffinityPred, MaxEBSVolumeCountPred, MaxGCEPDVolumeCountPred, MaxCSIVolumeCountPred,
		MaxAzureDiskVolumeCountPred, MaxCinderVolumeCountPred, CheckVolumeBindingPred, NoVolumeZoneConflictPred,
		EvenPodsSpreadPred, MatchInterPodAffinityPred}
}
```

预选predicates（如**PodFitsHost**或**NoDiskConflict**）按特定顺序计算，然后决定顺序。这样，一个节点对新的Pod部署的检查数量降到最小，如果该节点的状态不正确，就可以从不必要的检查中排除该节点。

例如，有一个名为**HostNamePred**的过滤器，也称为**HostName**，它过滤掉与pod spec中指定的节点名不匹配的节点。另一个预选是**PodFitsResources**，以确保可用的CPU和内存能够满足Pod所需的资源。

调度程序可以通过传递一个**kind:Policy**配置来更新，该配置可以对预选进行排序，为优先级赋予特殊的权重，甚至**hardpodaffinitysymmeryweight**，即使我们将Pod a设置为与Pod B一起运行，那么Pod B应该自动与Pod a一起运行。

### 1.3. Priorities（优选）

**优先级**是用来衡量资源的函数。除非配置了Pod和Node的Affinity被配置成SelectorSpreadPriority设置（该设置基于现有运行Pod的数量对节点进行排序），否则它们将选择Pod数量最少的节点。这是一种在集群中分配Pod的基本方法。

其他优先级可以用于特定的集群需求。**ImageLocalityPriorityMap**支持已下载容器image的节点。镜像大小的总和与具有最高优先级的最大值进行比较，但不检查即将使用的镜像。

目前，包含的优选规则超过10个，范围从检查标签的存在到选择请求CPU和内存使用率最高的节点。您可以在**master/pkg/scheduler/algorithm/priorities**查看优先级列表。（1.18版本的不在这了，应该在[源代码](https://github.com/kubernetes/kubernetes/blob/323f34858de18b862d43c40b2cced65ad8e24052/pkg/scheduler/framework/plugins/imagelocality/image_locality.go)）

从v1.14开始的稳定功能允许设置**PriorityClass**并通过使用**PriorityClassName**设置来调度pod。这允许用户抢占或逐出较低优先级的pod，以便可以安排其较高优先级的pod。如果一个或多个现有pod被逐出，kube调度程序将确定挂起状态的pod可以运行的节点。如果找到一个节点，低优先级pod将被逐出，而高优先级pod将被调度。使用Pod中断预算（Pod Disruption Budget简称PDB）可以限制Pod抢占收回的数量，以确保有足够的Pod保持运行。如果没有其他可用的选项，即使违反了PDB，调度器也将删除pods。

### 1.4. 调度的策略

默认的调度策略包含了一堆的预选和优选规则。但是这些可以使用调度的策略文件去改变他。

下面是一个例子

``` json
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
        {"name" : "MatchNodeSelector", "order": 6},
        {"name" : "PodFitsHostPorts", "order": 2},
        {"name" : "PodFitsResources", "order": 3},
        {"name" : "NoDiskConflict", "order": 4},
        {"name" : "PodToleratesNodeTaints", "order": 5},
        {"name" : "PodFitsHost", "order": 1}
        ],
"priorities" : [
        {"name" : "LeastRequestedPriority", "weight" : 1},
        {"name" : "BalancedResourceAllocation", "weight" : 1},       
        {"name" : "ServiceSpreadingPriority", "weight" : 2},
        {"name" : "EqualPriority", "weight" : 1}   
        ],
"hardPodAffinitySymmetricWeight" : 10
}
```

一般来说，我们通过使用**--policy-config-file**参数来应用这个Policy配置，并使用**--scheduler-name**参数定义此调度程序的名称。然后，我们将有两个调度程序运行，并将能够指定在pod的spec中使用哪个调度程序。

对于多个调度程序，Pod分配可能会发生冲突。每个Pod都应该声明应该使用哪个调度程序。但是，如果单独的调度程序确定某个节点由于可用资源而符合条件，并且两个调度程序都试图部署，从而导致该资源不再可用，则会发生冲突。当前的解决方案是让本地kubelet将Pods返回调度程序进行重新分配。最终，一个Pod将成功部署，另一个将安排在别处。

## 2. 在Pod的Spec中配置调度

大部分的调度策略可以作为Pod的Spec文件中的一部分来定义。一个Pod的Spec文件包含了相面几个字段可以影响调度的：

- **nodeName**
- **nodeSelector**
- **affinity**
- **schedulerName**
- **tolerations**

### 2.1. 配置的解释：

+ nodeName and nodeSelector

这个选项允许Pod被分配到特定node或者是一组拥有特定标签的node

+ affinity and anti-affinity

亲和性和反亲和性是说pod被调度的时候需要或者喜欢被分配到某个节点。如果使用了偏好，首先就需要在已经预选出来的节点中匹配，但是如果不存在匹配的节点，那么就会任意选择一个节点。

+ taints and tolerations

污点的使用允许对node进行标记，这样Pods就不会由于某些原因而被调度，比如初始化后的主节点。配置容忍度会让Pod忽略污点，并在满足其他要求的情况下进行调度。

+ schedulerName

如果上面的选项都不能满足集群的需要，那么还可以部署自定义调度程序。然后，每个Pod可以包含一个**schedulerName**来选择要使用的计划。

### 2.2. 节点选择器

pod规范中的**nodeSelector**字段提供了使用一个或多个键值对以node或一组node为调度目标的简单方法。

``` yaml
spec:
  containers:
  - name: redis
    image: redis
  nodeSelector:
    net: fast
```

配置**nodeSelector**会告诉调度器将pod放置在与标签匹配的节点上。必须满足所有列出的选择器，但节点可能有多个标签。在上面的例子中，任何键为**net**的node的配置为**fast**都是调度的候选节点。请记住，标签是管理员创建的标签，与实际资源没有关联。即使节点的网络可能很慢。

在找到具有匹配标签的节点之前，pod将保持挂起状态。

使用affinity/anti-affinity和nodeSelector的表达方式是一样的。

### 2.3. Pod Affinity Rules

如果位于同一个node，可以进行大量通信或共享数据的Pod可能运行得最好，这就是一种亲和力。为了获得更大的容错性，您可能希望Pods尽可能地分开，这就是反亲和力的。这些配置由调度器根据已经运行的pod的标签来决定。因此，调度器必须询问每个节点并跟踪正在运行的pod的标签。大于几百个节点的集群可能会出现显著的性能损失。Pod关联规则使用**In**、**NotIn**、**Exists**和**DoesNotExist**运算符。

#### Pod Affinity规则

+ requiredDuringSchedulingIgnoredDuringExecution

表示除非下面operator的值为true，否则Pod就不会运行在这里。即使operator将来变成了false，pod也会继续在这个节点上运行。我们可以认为这是一个硬性规定

+ preferredDuringSchedulingIgnoredDuringExecution

和上面相同，调度器会选择一个最好有下面配置的节点。如果没有节点拥有匹配的标签，pod也会被调度到没有配置的节点。这是一个相对不那么强烈的要求，所以说前缀是preferred，而不是require。

+ podAffinity

使用Pod的亲和性，调度器会把Pod调度到一起

+ podAntiAffinity

这个会让调度器去保证pod运行在不同的节点

+ topologyKey

  这个规则允许对Pod在部署时候进行一般分组。亲和性（或反亲和性）将尝试在具有已声明topologyKey的节点上运行，并在具有特定标签的pod上运行。**topologyKey**可以是任何合法密钥，但需要考虑一些重要因素。

  + 如果使用了 **requiredDuringScheduling** 和**LimitPodHardAntiAffinityTopology** 配置,  **topologyKey** 必须被配置为**kubernetes.io/hostname**. 
  + 如果使用**PreferredDuringScheduling**,  **topologyKey** 的值留空就可以了, 或者是下面配置的组合**kubernetes.io/hostname**, **topology.kubernetes.io/zone** and **topology.kubernetes.io/region**.

#### podAffinity的例子

亲和性和pod亲和性的配置，我们看下面的例子。这个同样需要在pod启动的时候给出一个特殊的标签，但是这个标签即使在后面被删除了也没有关系。

``` yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
```

在声明的topology区域，Pod可以被调度到标签名字为security并且值包含S1的节点上。如果这个需求没有被满足，pod就会处于pending状态。

#### podAntiAffinity的例子

使用podAntiAffinity，我们可以防止pod被调度到拥有摸个特定标签的节点上。从这个角度来说，调度器会倾向于避开label的名字为security并且值包含S2的节点。

``` yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: security
          operator: In
          values:
          - S2
    topologyKey: kubernetes.io/hostname 
```

在一个大而多变的环境中，可能有多种情况需要避免。作为首选配置，这个设置会尝试避开某些标签，但仍会在某些节点上调度Pod。由于Pod始终处于运行状态，我们可以为特定规则提供权重的选项。权重可以声明为1到100之间的值。然后调度程序尝试选择或避免具有最大组合值的node。

#### 2.4. Node Affinity规则

Where Pod affinity/anti-affinity has to do with other Pods, the use of **nodeAffinity** allows Pod scheduling based on node labels. This is similar and will some day replace the use of the **nodeSelector** setting. The scheduler will not look at other Pods on the system, but the labels of the nodes. This should have much less performance impact on the cluster, even with a large number of nodes.

- Uses **In**, **NotIn**, **Exists**, **DoesNotExist** operators
- **requiredDuringSchedulingIgnoredDuringExecution**
- **preferredDuringSchedulingIgnoredDuringExecution**
- Planned for future: **requiredDuringSchedulingRequiredDuringExecution**.

Until **nodeSelector** has been fully deprecated, both the selector and required labels must be met for a Pod to be scheduled.

### 2.5. Node Affinity例子

``` yaml
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:  
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/colo-tx-name
            operator: In
            values:
            - tx-aus
            - tx-dal
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk-speed
            operator: In
            values:
            - fast
            - quick 
```

第一个**nodeAffinity**规则需求节点的key是**kubernetes.io/colo-tx-name**并且值是**tx-aus** 或者 **tx-dal**. 

第二个规则给出了额外的节点的权重，并且**disk-speed**的值是**fast** 或者 **quick**. Pod会被调度到相同的节点之上。但是为了防止意外，这个配置只是倾向于一个拥有特定标签的节点。

### 3. Taints和Toleration

### 3.1. Taints

一个有特殊污点Taints的节点会排斥不会容忍这种污点的Pod。污点表示为**key=value:effect**. key和value是由管理员创建。

使用的键和值可以是任何合法的字符串，这允许根据任何需要灵活地防止pod在节点上运行。如果Pod当前没有容忍度，调度器将不会考虑有污点的节点。

### 3.2. 防止Pod被调度的方式

在node上配置下面的选项

- NoSchedule

  不会把Pod调度到这个节点上，除非pod有toleration。已经存在的Pod会持续的运行，和容忍度无关。

- PreferNoSchedule

  调度器会尽量的避免把pod调度到这个节点上，除非有没有污点的节点可以匹配到Pod的容忍度。已经存在的Pod不会受到影响。

- NoExecute

  这一污点将导致现有的Pod被驱逐，以后也不会调度pod。如果现有的Pod有容忍度，它将继续运行。如果设置了Pod的**tolerationSeconds**，则它们将保留数秒，然后被逐出。而节点故障将导致kubelet增加300秒的容忍度，以避免不必要的逐出。

如果节点有多个污点，调度器会忽而略这些匹配到的容忍度。其他的没有被忽略的污点就会生效。

**TaintBasedEvictions**目前还处在alpha阶段。当节点出现问题时，kubelet使用污点来控制需要驱逐的pod的比例

### 3.3. Tolerations

在节点上设置容忍度tolerations用于在有污点Taints的节点上调度pod。这提供了一种避免pod被调度到某个节点的简单方法。只有那些有特殊的容忍度的节点才会被调度pod上去。

Pod Spec中可以包含运算符，如果未明确说明，则默认为**等于**。使用运算符**Equal**需要一个值来匹配。不应指定**Exists**运算符。如果一个空的key使用**Exists**运算符，它将容忍每一个污点。如果没有生效，但声明了键和运算符，则所有效果都与声明的键匹配。

``` yaml
tolerations:
- key: "server"
  operator: "Equal"
  value: "ap-east"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

上面的例子中，在这个节点被打上污点**NoExecute**之后，Pod会停留在由键为server和他的值为ap-east的节点3600秒。时间结束之后，pod会被驱逐。

## 4. 自定义调度器

如果默认的调度机制（affinity, taints, policies）还不能满足我们的需求，我们可以开发自己的调度器。自定义调度程序的编程不在本课程的范围内，但如果希望从现有的调度程序代码开始，可以在[GitHub](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler)中找到这些代码。

如果Pod Spec没有声明要使用哪个调度程序，则默认使用标准调度程序。如果Pod声明了一个调度程序，而该容器没有运行，那么Pod将永远处于**挂起**状态。

调度过程的最终结果是让pod绑定到应该运行整个pod的节点之上。binding是**API/v1**组中的Kubernetes API之一。从技术上讲，在没有任何调度程序运行的情况下，您仍然可以通过指定pod的绑定来调度节点上的pod。

您还可以同时运行多个调度程序。

您可以使用以下命令查看计划程序和其他信息：

``` bash
$ kubectl get events
```

