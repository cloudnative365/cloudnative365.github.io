---
title: StatefulSets
keywords: keynotes, L2_advanced, kubernetes, CKA, STATEFULSETS
permalink: keynotes_L2_advanced_3_kubernetes_14_STATEFULSETS.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/14_STATEFULSETS
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. STATEFULSETS简介
StatefulSet 是用来管理有状态应用的workload API 对象。

StatefulSet 用来管理 Deployment 和扩展一组 Pod，并且能为这些 Pod 提供*顺序和唯一性保证*。

和 Deployment 相同的是，StatefulSet 管理了基于相同容器定义的一组 Pod。但和 Deployment 不同的是，StatefulSet 为它们的每个 Pod 维护了一个固定的 ID。这些 Pod 是基于相同的声明来创建的，但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。

StatefulSet 和其他控制器使用相同的工作模式。在 StatefulSet *对象* 中定义我们所期望的状态，然后 StatefulSet 的 *控制器* 就会通过各种更新来达到那种你想要的状态。

## 2. 什么时候应该使用StatefulSet

如果我们的应用有以下一个或者多个需求，我们就应该使用StatefulSet

- Stable, unique network identifiers. 稳定的、唯一的网络标识符
- Stable, persistent storage. 稳定的、持久的存储。
- Ordered, graceful deployment and scaling.有序的、优雅的部署和缩放。
- Ordered, automated rolling updates.有序的、自动的滚动更新。

综上所述，稳定意味着 Pod 调度或重调度的整个过程是有持久性的。如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 Deployment或者 ReplicaSet 可能更适用于我们的无状态应用部署需要。

## 3. 使用StatefulSet的限制

+ 给定 Pod 的存储必须由 PersistentVolume 驱动 基于所请求的 storage class 来提供，或者由管理员预先提供。
+ 删除或者收缩 StatefulSet 并不会删除它关联的存储卷。这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值。
+ StatefulSet 当前需要 headless 服务 来负责 Pod 的网络标识。您需要负责创建此服务。
+ 当删除 StatefulSets 时，StatefulSet 不提供任何终止 Pod 的保证。为了实现 StatefulSet 中的 Pod 可以有序和优雅的终止，可以在删除之前将 StatefulSet 缩放为 0。
+ 在默认 Pod 管理策略(OrderedReady) 时使用 滚动更新，可能进入需要 人工干预 才能修复的损坏状态。

## 4. StatefulSet的示例

下面的示例包含了所有的StatefulSet组件

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

+ 名为 nginx 的 Headless Service 用来控制网络域名。
+ 名为 web 的 StatefulSet 有一个 Spec，它表明将在独立的 3 个 Pod 副本中启动 nginx容器
+ volumeClaimTemplates 将通过 PersistentVolumes 驱动提供的 PersistentVolumes 来提供稳定的存储。

### 4.1. Pod选择器

必须设置标签选择器，使之匹配其在 `.spec.template.metadata.labels` 中设置的标签。在 Kubernetes 1.8 版本之前，被忽略 `.spec.selector` 字段会获得默认设置值。在 1.8 和以后的版本中，未指定匹配的 Pod 选择器将在创建 StatefulSet 期间导致验证错误。

### 4.2. Pod标识

StatefulSet Pod 具有唯一的标识，该标识包括**顺序标识**、**稳定的网络标识**和**稳定的存储**。该标识和 Pod 是绑定的，不管它被调度在哪个节点上。

+ 有序索引

对于具有 N 个副本的 StatefulSet，StatefulSet 中的每个 Pod 将被分配一个整数序号，从 0 到 N-1，该序号在 StatefulSet 上是唯一的。

+ 稳定的网络 ID

StatefulSet 中的每个 Pod 根据 StatefulSet 的名称和 Pod 的序号派生出它的主机名。组合主机名的格式为`$(StatefulSet 名称)-$(序号)`。上例将会创建三个名称分别为 `web-0、web-1、web-2` 的 Pod。 StatefulSet 可以使用 [headless 服务](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 控制它的 Pod 的网络域。管理域的这个服务的格式为： `$(服务名称).$(命名空间).svc.cluster.local`，其中 `cluster.local` 是集群域。 一旦每个 Pod 创建成功，就会得到一个匹配的 DNS 子域，格式为：`$(pod 名称).$(所属服务的 DNS 域名)`，其中所属服务由 StatefulSet 的 `serviceName` 域来设定。

下面给出一些选择集群域、服务名、StatefulSet 名、及其怎样影响 StatefulSet 的 Pod 上的 DNS 名称的示例：

| Cluster Domain | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain              | Pod DNS                                      | Pod Hostname |
| :------------- | :---------------- | :-------------------- | :------------------------------ | :------------------------------------------- | :----------- |
| cluster.local  | default/nginx     | default/web           | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
| cluster.local  | foo/nginx         | foo/web               | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local     | web-{0..N-1} |
| kube.local     | foo/nginx         | foo/web               | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local        | web-{0..N-1} |

> **注意：** 集群域会被设置为 `cluster.local`，除非有[其他配置](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)。

+ 稳定的存储

Kubernetes 为每个 VolumeClaimTemplate 创建一个 PersistentVolume。在上面的 nginx 示例中，每个 Pod 将会得到基于 StorageClass `my-storage-class` 提供的 1 Gib 的 PersistentVolume。如果没有声明 StorageClass，就会使用默认的 StorageClass。当一个 Pod 被调度（重新调度）到节点上时，它的 `volumeMounts` 会挂载与其 PersistentVolumeClaims 相关联的 PersistentVolume。请注意，当 Pod 或者 StatefulSet 被删除时，与 PersistentVolumeClaims 相关联的 PersistentVolume 并不会被删除。要删除它必须通过手动方式来完成。

+ Pod 名称标签

当 StatefulSet 控制器 创建 Pod 时，它会添加一个标签 `statefulset.kubernetes.io/pod-name`，该标签设置为 Pod 名称。这个标签允许您给 StatefulSet 中的特定 Pod 绑定一个 Service。

## 5. 保证部署和扩展

- 对于包含 N 个 副本的 StatefulSet，当部署 Pod 时，它们是依次创建的，顺序为 `0..N-1`。
- 当删除 Pod 时，它们是逆序终止的，顺序为 `N-1..0`。
- 在将缩放操作应用到 Pod 之前，它前面的所有 Pod 必须是 Running 和 Ready 状态。
- 在 Pod 终止之前，所有的继任者必须完全关闭。

StatefulSet 不应将 `pod.Spec.TerminationGracePeriodSeconds` 设置为 0。这种做法是不安全的，要强烈阻止。更多的解释请参考 [强制删除 StatefulSet Pod](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/)。

在上面的 nginx 示例被创建后，会按照 web-0、web-1、web-2 的顺序部署三个 Pod。在 web-0 进入 Running 和 Ready 状态前不会部署 web-1。在 web-1 进入 Running 和 Ready 状态前不会部署 web-2。如果 web-1 已经处于 Running 和 Ready 状态，而 web-2 尚未部署，在此期间发生了 web-0 运行失败，那么 web-2 将不会被部署，要等到 web-0 部署完成并进入 Running 和 Ready 状态后，才会部署 web-2。

如果用户想将示例中的 StatefulSet 收缩为 `replicas=1`，首先被终止的是 web-2。在 web-2 没有被完全停止和删除前，web-1 不会被终止。当 web-2 已被终止和删除、web-1 尚未被终止，如果在此期间发生 web-0 运行失败，那么就不会终止 web-1，必须等到 web-0 进入 Running 和 Ready 状态后才会终止 web-1。

### 5.1. Pod 管理策略

在 Kubernetes 1.7 及以后的版本中，StatefulSet 允许您不要求其排序保证，同时通过它的 `.spec.podManagementPolicy` 域保持其唯一性和身份保证。 在 Kubernetes 1.7 及以后的版本中，StatefulSet 允许您放宽其排序保证，同时通过它的 `.spec.podManagementPolicy` 域保持其唯一性和身份保证。

### 5.2. OrderedReady Pod 管理

`OrderedReady` 是 StatefulSet 管理Pod的默认设置。它实现了上面所有的功能。

### 5.3. Parallel Pod 管理

`Parallel` 是 StatefulSet 控制器管理Pod并行的启动或终止所有的 Pod，启动或者终止其他 Pod 前，无需等待 Pod 进入 Running 和 ready 或者完全停止状态。

## 6. 更新策略

在 Kubernetes 1.7 及以后的版本中，StatefulSet 的 `.spec.updateStrategy` 字段让您可以配置和禁用掉自动滚动更新 Pod 的容器、标签、资源请求或限制、以及注解。

### 6.1. 关于删除策略

`OnDelete` 更新策略保留了 1.6 及以前版本的历史遗留行为。当 StatefulSet 的 `.spec.updateStrategy.type` 设置为 `OnDelete` 时，它的控制器将不会自动更新 StatefulSet 中的 Pod。用户必须手动删除 Pod 以便让控制器创建新的 Pod，以此来对 StatefulSet 的 `.spec.template` 的变动作出反应。

### 6.2. 滚动更新策略

`RollingUpdate` 更新策略对 StatefulSet 中的 Pod 执行自动的滚动更新。在没有声明 `.spec.updateStrategy` 时，`RollingUpdate` 是默认配置。 当 StatefulSet 的 `.spec.updateStrategy.type` 被设置为 `RollingUpdate` 时，StatefulSet 控制器会删除和重建 StatefulSet 中的每个 Pod。 它将按照与 Pod 终止相同的顺序（从最大序号到最小序号）进行，每次更新一个 Pod。它会等到被更新的 Pod 进入 Running 和 Ready 状态，然后再更新其前身。

### 6.3. 分区更新

通过声明 `.spec.updateStrategy.rollingUpdate.partition` 的方式，`RollingUpdate` 更新策略可以实现分区。如果声明了一个分区，当 StatefulSet 的 `.spec.template` 被更新时，所有序号大于等于该分区序号的 Pod 都会被更新。所有序号小于该分区序号的 Pod 都不会被更新，并且，即使他们被删除也会依据之前的版本进行重建。如果 StatefulSet 的 `.spec.updateStrategy.rollingUpdate.partition` 大于它的 `.spec.replicas`，对它的 `.spec.template` 的更新将不会传递到它的 Pod。 在大多数情况下，您不需要使用分区，但如果您希望进行阶段更新、执行金丝雀或执行分阶段展开，则这些分区会非常有用。

### 6.4. 强制回滚

在默认 Pod 管理策略(`OrderedReady`) 时使用 滚动更新 ，可能进入需要人工干预才能修复的损坏状态。

如果更新后 Pod 模板配置进入无法运行或就绪的状态（例如，由于错误的二进制文件或应用程序级配置错误），StatefulSet 将停止回滚并等待。

在这种状态下，仅将 Pod 模板还原为正确的配置是不够的。由于已知问题，StatefulSet 将继续等待损坏状态的 Pod 准备就绪（永远不会发生），然后再尝试将其恢复为正常工作配置。

恢复模板后，还必须删除 StatefulSet 尝试使用错误的配置来运行的 Pod。这样，StatefulSet 才会开始使用被还原的模板来重新创建 Pod。