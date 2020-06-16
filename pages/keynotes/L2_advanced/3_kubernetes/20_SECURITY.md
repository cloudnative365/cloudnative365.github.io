---
title: 安全
keywords: keynotes, advanced, kubernetes, CKA, SECURITY
permalink: keynotes_L2_advanced_3_kubernetes_20_SECURITY.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/20_SECURITY
typora-root-url: ../../../../../cloudnative365.github.io
---

## 学习目标

- API请求的流程
- 配置认证的规则
- 配置认证的策略
- 使用网络策略限制网络流量

## 1. 简介

安全性是一个庞大而复杂的课题，特别是在像Kubernetes这样的分布式系统中。因此，我们将讨论一些在Kubernetes上下文中处理安全性的概念。

然后，我们将重点关注API服务器的身份验证方面，我们将深入研究授权，看看ABAC和RBAC之类的东西，这现在是使用**kubeadm**引导Kubernetes集群时的默认配置。

我们将查看**admission control**（准入系统）系统，它允许您查看并可能修改传入的请求，并对这些请求执行最终拒绝或接受操作。

接下来，我们将讨论一些其他概念，包括如何使用安全上下文和pod安全策略更严密地保护pod，这是Kubernetes中成熟的API对象。

最后，我们将研究网络策略。默认情况下，我们倾向于不启用网络策略，它允许任何流量在所有不同的名称空间中流过所有pod。使用网络策略，我们实际上可以定义入口规则，以便限制不同命名空间之间的入口流量。正在使用的网络工具（如Flannel或Calico）将确定是否可以实现网络策略。随着Kubernetes的成熟，这将成为一个强烈建议的配置。

## 2. 访问API

要在kubernetes集群中做操作，我们需要访问API并且经过下面三个步骤。

- Authentication: 认证
- Authorization (ABAC or RBAC): 授权
- Admission Control. 准入控制

在官方文档中有更加详细的描述 [controlling access to the Kubernetes API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/) 

我们来看下面的图片

![1u7mjwewyqon-Accessing_the_API](/pages/keynotes/L2_advanced/3_kubernetes/pics/20_SECURITY/1u7mjwewyqon-Accessing_the_API.png)

### 2.1. Accessing the API

一旦请求安全地到达API服务器，它将首先通过已配置的任何身份验证模块。如果身份验证失败或请求获得身份验证并传递到授权步骤，则可以拒绝该请求。

在授权步骤中，将对照现有策略检查请求。如果用户有权执行请求的操作，则将对其进行授权。然后，请求将进入最后一步。一般来说，准入控制器会检查正在创建的对象的实际内容，并在允许请求之前验证它们。

除了这些步骤之外，通过网络到达API服务器的请求还使用TLS加密。这需要使用SSL证书正确配置。如果您使用**kubeadm**，这个配置是为您完成的；否则，请遵循[*Kubernetes the Hard Way*](https://github.com/kelseyhightower/kubernetes-the-hard-way网站)或者查看[API服务器配置选项](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

## 3. 认证Authentication

关于kubernetes的认证，有下面三点需要牢记

- 最直接了当的方式，认证是通过证书，token或者基础认证（用户名和密码）来实现的
- 用户不是由API创建的，但是应该有外部系统去管理
- 系统账户是请求API的必要流程（详见 [*Configure Service Accounts for Pods*](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)）

还有两种更高级的身份验证机制。webhook可用于验证承载令牌，以及与外部OpenID提供程序的连接。

使用的身份验证类型在kube apiserver启动选项中定义。下面是配置选项子集的四个示例，需要根据您选择的身份验证机制进行设置：

**--basic-auth-file**

**--oidc-issuer-url**

**--token-auth-file**

**--authorization-webhook-config-file**

必须使用下面一个或者多个认证模块

- x509 Client Certs; 
- static token, bearer or bootstrap token; 
- static password file; 
- service account;
- OpenID connect tokens. 

每一种都会一直尝试直到成功，顺序并不是定义好的。也可以开启匿名访问，否则的话我们会得到一个401的回应。用户不是API创建的，但是应该有外部的系统去管理 [Kubernetes Documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).

## 4. 授权Authorization

一旦请求经过身份验证，就需要授权它能够通过Kubernetes系统并执行其预期操作。

有三种主要的授权模式和两个全局的允许/拒绝请求，三个主要的授权模式是：

- ABAC
- RBAC
- Webhook.

他们可以在API server启动的时候通过参数指定

**--authorization-mode=ABAC**

**--authorization-mode=RBAC**

**--authorization-mode=Webhook**

**--authorization-mode=AlwaysDeny**

**--authorization-mode=AlwaysAllow**

授权模块借助策略来允许请求。请求的属性是通过策略来检查的（比如，user，group，namespace，verb）

#### 4.1. ABAC, RBAC and Webhook Modes

+ ABAC

  [ABAC](https://kubernetes.io/docs/reference/access-authn-authz/abac/) 是基于属性的访问控制（Attribute Based Access Control）的缩写。这是Kubernetes中第一个允许管理员实现正确策略的授权模型。如今，RBAC正成为默认的授权模式。

  策略在JSON文件中定义，并由kube apiserver启动选项引用：

  **--authorization-policy-file=my_policy.json**

  下面的例子中，policy文件显示下面的认证用户Bob可以读取名称空间foobar中的pod

  ```json
  {
      "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
      "kind": "Policy",
      "spec": {
          "user": "bob",
          "namespace": "foobar",
          "resource": "pods",
          "readonly": true     
      }
  }
  ```

  更多的策略请参考官方文档 [policy examples](https://kubernetes.io/docs/reference/access-authn-authz/abac/#examples)

+ RBAC

  [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 是Role Based Access Control的缩写

  所有资源都是Kubernetes中建模的API对象，从Pods到名称空间。它们也属于API组，例如**core**和**apps**，这些资源允许创建、读取、更新和删除（CRUD）等操作，我们一直在使用这些操作。在YAML文件中，操作称为**verbs**。添加到这些基本组件中，我们将添加更多的API元素，然后可以通过RBAC进行管理。

  规则是可以作用于API组的操作。角色Role是一组影响单个命名空间或作用域的规则，而**ClusterRoles**具有整个群集的作用域。

  在使用kubectl时，每个操作都可以作用于三个主题中的一个：不作为API对象存在的**User Accounts**、**Service Accounts**和**Groups**，这三个主题称为**clusterrolebinding**。

  然后RBAC编写规则，允许或拒绝用户、角色或组对资源的操作。

  虽然RBAC可能很复杂，但基本流程是为用户创建证书。由于用户不是Kubernetes的API对象，我们需要外部身份验证，比如OpenSSL证书。根据群集证书颁发机构生成证书后，我们可以使用上下文为用户设置该凭据。

  然后，可以使用角色来配置**apiGroups**、**resources**和它们所允许的**verbs**的关联。然后可以将用户绑定到一个角色，限制他们在集群中的工作内容和工作位置。

  

  以下是RBAC流程：

  + 确定或创建命名空间

  + 为用户创建证书凭据

  + 使用上下文将用户的凭据设置为命名空间

  + 为预期的任务集创建角色

  + 将用户绑定到角色

  + 验证用户的访问权限是否有限。

+ Webhook

  A Webhook is an HTTP callback, an HTTP POST that occurs when something happens; a simple event-notification via HTTP POST. A web application implementing Webhooks will POST a message to a URL when certain things happen.

  Webhook是一个HTTP的回调，一旦集群中有情况，HTTP的POST请求就会生成，一个通过HTTP生成的POST的事件通知请求。当特定事件发生的时候，网络应用产生的Webhooks会POST一个消息给特定的URL。 参考： [*Webhook Mode*](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)

## 5. 准入控制器Admission Controller

允许API请求进入Kubernetes的最后一步是许可控制。

准入控制器是可以访问由请求创建的对象内容的程序。他们可以修改或验证内容，并可能拒绝请求。

某些功能正常工作需要准入控制器。随着Kubernetes的成熟，controller也加入了。从1.13.1版本的**kube apiserver**开始，准入控制器现在被编译成二进制文件，而不是在执行期间传递的列表。要启用或禁用，可以传递以下选项，更改要启用或禁用的插件：

**--enable-admission-plugins=Initializers,NamespaceLifecycle,LimitRanger
--disable-admission-plugins=PodNodeSelector**

第一个控制器是**Initializers**，它允许动态修改API请求，提供了很大的灵活性。文档中解释了每个许可控制器的功能。例如，**ResourceQuota**控制器将确保创建的对象不违反任何现有配额。

## 6. 安全上下文Security Contexts

pod和pod中的容器可以被赋予特定的安全约束，以限制容器中运行的进程可以做什么。例如，进程的UID、Linux功能和文件系统组可以受到限制。

这种安全限制称为安全上下文。它可以为整个pod或每个容器定义，并在资源清单中表示为附加部分。显著的区别在于Linux的功能是在容器级别设置的。

例如，如果要强制执行容器不能以根用户身份运行其进程的策略，可以添加pod安全上下文，如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - image: nginx
```

当我们创建这个Pod的时候，我们可以看到一个报警，说容器正常试图使用root权限运行，这个是不允许的，因此Pod永远不会运行起来

```bash
$ kubectl get pods

NAME   READY  STATUS                                                 RESTARTS  AGE
nginx  0/1    container has runAsNonRoot and image will run as root  0         10s
```

参考 [*Configure a Security Context for a Pod or Container*](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) 

## 7. Pod Security Policies

要自动执行安全上下文，可以定义[PodSecurityPolicies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)（PSP）。PSP是通过遵循PSP API模式的标准Kubernetes清单定义的。下面是一个例子。

这些策略是集群级别的规则，用于控制pod可以做什么、可以访问什么、以什么用户身份运行等等。

例如，如果不希望集群中的任何容器作为根用户运行，则可以为此定义PSP。您还可以防止容器被授予特权或使用主机网络命名空间或主机PID命名空间。

你可以在下面看到一个PSP的例子：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
```

要启用Pod安全策略，需要将控制器管理器的许可控制器配置为包含**PodSecurityPolicy**。当与集群中的RBAC配置结合使用时，这些策略更有意义。这将允许您微调哪些用户可以运行，以及他们的容器将具有哪些功能和低级权限。

See the [PSP RBAC example](https://github.com/kubernetes/examples/tree/master/staging/podsecuritypolicy/rbac/) on GitHub for more details.

## 8. Network Security Policies

By default, all pods can reach each other; all ingress and egress traffic is allowed. This has been a high-level networking requirement in Kubernetes. However, network isolation can be configured and traffic to pods can be blocked. In newer versions of Kubernetes, egress traffic can also be blocked. This is done by configuring a **NetworkPolicy**. As all traffic is allowed, you may want to implement a policy that drops all traffic, then, other policies which allow desired ingress and egress traffic.

The **spec** of the policy can narrow down the effect to a particular namespace, which can be handy. Further settings include a **podSelector**, or label, to narrow down which Pods are affected. Further ingress and egress settings declare traffic to and from IP addresses and ports.

Not all network providers support the **NetworkPolicies** kind. A non-exhaustive list of providers with support includes Calico, Romana, Cilium, Kube-router, and WeaveNet.

In previous versions of Kubernetes, there was a requirement to annotate a namespace as part of network isolation, specifically the **net.beta.kubernetes.io/network-policy= value**. Some network plugins may still require this setting.

On the next page, you can find an example of a **NetworkPolicy** recipe. More [network policy recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes) can be found on GitHub.

默认情况下，所有Pod都可以相互访问；允许所有进出流量。这是Kubernetes的高级网络需求。但是，可以配置网络隔离，并阻止到Pod的流量。在更新版本的Kubernetes中，出口流量也可以被阻塞。这是通过配置**NetworkPolicy**来完成的。由于允许所有流量，您可能希望实现一个禁止所有流量的策略，然后其他策略允许必须的进出流量的策略。

策略的**spec**可以将效果缩小到特定的命名空间，这非常方便。进一步的设置包括**podSelector**或label，以缩小到哪些Pods受到影响。进一步的入口和出口设置声明进出IP地址和端口的流量。

并非所有网络提供商都支持**NetworkPolicies**类型。提供支持的非详尽的提供者列表包括Calico、Romana、Cilium、Kube router和WeaveNet。

在以前的Kubernetes版本中，需要作为网络隔离的一部分对名称空间进行注释，特别是**net.beta.kubernetes.io/network-policy= value**。某些网络插件可能仍然需要此设置。



## 9. Network Security Policy Example

从**v1 apiVersion**开始，网络策略就是个稳定版本的API了。下面的例子中限制了对于defualt名称空间中的进出流量

拥有标签 **role: db**的pod会被这个策略所限制，并且策略限制进出的流量

**ingress**配置包含了172.17的网络，而他的范围内的网络**172.17.1.0**是不包含在这个控制范围中的

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-egress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
  - namespaceSelector:
      matchLabels:
        project: myproject
  - podSelector:
      matchLabels:
        role: frontend
  ports:
  - protocol: TCP
    port: 6379
egress:
- to:
  - ipBlock:
      cidr: 10.0.0.0/24
  ports:
  - protocol: TCP
    port: 5978
```

这些规则将以下设置的命名空间更改为标记**project:myproject**。受影响的Pod还需要匹配标签**role: frontend**。最后，pod中端口6379上的TCP通信将被允许。

出口规则具有**to**设置，在本例中**10.0.0/24**将TCP流量范围设置为端口5978。

使用空的入口或出口规则会拒绝所包含的pod的所有类型的流量，尽管不建议这样做。请改用另一个专用的**NetworkPolicy**。



### 10. Default Policy Example

空大括号将匹配其他**网络策略**未选择的所有pod，并且不允许进入流量。出口流量不受此策略的影响。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

由于可能存在复杂的进出规则，因此创建包含简单隔离规则和使用易于理解的名称和标签的多个对象可能会有帮助。

一些网络插件，比如WeaveNet，可能需要对名称空间进行注释。下面显示了**myns**命名空间的**DefaultDeny**设置：

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: myns
  annotations:
    net.beta.kubernetes.io/network-policy: |
     {
        "ingress": {
          "isolation": "DefaultDeny"
        }
     }
```

