---
title: 自定义资源
keywords: keynotes, advanced, kubernetes, CKA, CUSTOM_RESOURCE_DEFINTIONS
permalink: keynotes_L2_advanced_3_kubernetes_18_CUSTOM_RESOURCE_DEFINTIONS.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/18_CUSTOM_RESOURCE_DEFINTIONS
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- 了解更多可用的kubernetes对象
- 部署一个新的自定义资源
- 部署一个新的资源和API
- API聚合

## 1. 自定资源

我们一直在使用内置资源或API endpoint。Kubernetes的灵活性还允许动态添加新资源。添加这些自定义资源后，可以使用标准调用和命令（如**kubectl**）创建和访问对象。新对象的创建将新的结构化数据存储在etcd数据库中，并允许通过kube apiser进行访问。

要使一个新的自定义资源成为声明式API的一部分，需要有一个控制器来连续检索结构化数据并满足和维护声明的状态。此控制器或运算符是创建和管理特定有状态应用程序的一个或多个实例的代理。我们使用了内置控制器，如deployment、daemonset和其他资源。

如果将应用程序部署到Kubernetes之外，在自定义的操作中定义的函数是该都有一个人来操作。

有两种方法可以将自定义资源添加到Kubernetes集群。最简单的方法是向集群添加自定义资源定义（CRD），但灵活性较差。第二种更灵活的方法是使用聚合API（aggregatedapi，a a），它要求编写一个新的API服务器并将其添加到集群中。

将新对象添加到集群的任何一种方式都称为自定义资源，这与内置资源不同。

如果使用RBAC进行授权，则可能需要授予对新CRD资源和控制器的访问权限。如果使用聚合API，则可以使用相同或不同的身份验证过程。

## 2. 定义自定义资源

正如我们已经了解到的，Kubernetes的解耦特性依赖于一些列的守护进程或控制器不停的监控，询问kube apiserver以确定我们制定的配置是否正确。如果当前状态与声明的状态不匹配，控制器将调用API来修改状态，直到它们匹配为止。如果添加新的API对象和控制器，则可以使用现有的kube apiserver来监视和控制该对象。添加自定义资源定义将添加到群集API路径，当前位于**apiextensions.k8s.io/v1**。

虽然这是向集群添加新对象的最简单方法，但它可能不够灵活，无法满足您的需要。只能使用现有的API功能。对象必须响应REST请求，并以与内置对象相同的方式验证和存储其配置状态。它们还需要与内置对象的保护规则一起存在。

CRD允许将资源部署在命名空间中或在整个集群中可用。YAML文件使用**scope:** parameter设置此值，该参数可以设置为**Namespaced**或**Cluster**。

在v1.8之前，有一种资源类型称为ThirdPartyResource（TPR）。这已被弃用，不再可用。所有资源都需要重建为CRD。升级后，需要删除现有的tpr并用crd替换，以便API URL指向功能对象。

### 2.1. 例子

``` yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.linux.com
spec:
  group: stable.linux.com
  version: v1
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    shortNames:
    - bks
    kind: BackUp
```

### 2.2. 解释

+ apiVersion

这个应该匹配到当前级别的稳定版本，就是**apiextensions.k8s.io/v1**

+ kind: CustomResourceDefinition

这个对象类型是由kube-apiserver给定的

+ name: backups.stable.linux.com

名称必须匹配下面spec字段中的内容（group和name中的plural）。语法应该是\<plural name>\.\<group>

+ group: stable.linux.com

组名字会变成REST API的一部分，**/apis/\<group>/\<version>**。比如稳定版:**/apis/stable/v1**

+ scope

定义这个资源是存在某个namespace中还是整个集群

+ plural

定义API URL的最后一部分，比如**apis/stable/v1/backups**.

+ singular and shortNames

singular：定义了显示的名字

shortNames：让命令在使用的时候更简单

+ kind

在定义资源清单的时候，他的kind类型，是一个驼峰式大小写的单数形式

### 2.3. 配置新的对象

``` yaml
apiVersion: "stable.linux.com/v1"
kind: BackUp
metadata:
  name: a-backup-object
spec:
  timeSpec: "* * * * */5"
  image: linux-backup-image
replicas: 5
```

注意**apiVersion**和**kind**与我们在上一步中创建的CRD匹配。**spec**参数取决于控制器。

对象将由控制器去验证。如果语法（如**timeSpec**）与预期值不匹配，则在配置验证时将收到错误信息。如果不进行验证，则只检查变量的存在性，而不检查其详细信息。

### 2.4. 可选的hooks

与内置对象一样，您可以使用一个称为**Finalizer**的异步预删除挂钩。如果API接收到**delete**请求，则对象元数据字段**metadata.deletionTimestamp**会更新。然后，控制器触发已配置的终结器。当终结器完成时，它将从列表中删除。控制器继续完成并删除终结器，直到字符串为空。然后，对象本身被删除。

Finalizer:

```yaml
metadata:
  finalizers:
  - finalizer.stable.linux.com
```

Validation:

```yaml
validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            timeSpec:
              type: string 
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

从v1.9开始的beta特性允许通过OpenAPI v3模式验证自定义对象。这将检查API服务器传递的对象中配置的各种属性。在上面的示例中，**timeSpec**必须是与特定模式匹配的字符串，并且允许的副本数在1到10之间。如果验证不匹配，则返回的错误是验证的失败行。

## 3. API聚合

聚合API的使用允许向集群添加其他Kubernetes类型的API服务器。添加的服务器充当kube apiserver的从属服务器，从v1.7开始，kube apiserver在进程中运行聚合层。注册扩展资源时，聚合层监视传递的URL路径，并将任何请求代理到新注册的API服务。

聚合层很容易启用。编辑kube apiserver启动期间传递的标志以包括**--enable aggregator routing=true**。一些供应商默认启用此功能。

外部的创建可以通过YAML配置文件或api完成。还需要为各种新对象在组件和RBAC规则之间配置TLS授权。[示例API服务器](https://github.com/kubernetes/sample-apiserver网站)在GitHub上可用。目前处于孵化阶段的项目是一个[API服务器生成器](https://github.com/kubernetes-sigs/apiserver-builder-alpha)它应该处理很多安全和连接配置。