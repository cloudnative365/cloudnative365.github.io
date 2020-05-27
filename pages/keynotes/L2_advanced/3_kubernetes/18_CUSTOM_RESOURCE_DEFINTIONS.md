---
title: 自定义资源
keywords: keynotes, L2_advanced, kubernetes, CKA, CUSTOM_RESOURCE_DEFINTIONS
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

这个对象类型是由kube-apiserver插入的

+ name: backups.stable.linux.com

The name must match the **spec** field declared later. The syntax must be \<plural name>\.\<group>

+ group: stable.linux.com

The group name will become part of the REST API under **/apis/\<group>/\<version>** or **/apis/stable/v1** in this case with the version set to v1.

+ scope

Determines if the object exists in a single namespace or is cluster-wide.

+ plural

Defines the last part of the API URL, such as **apis/stable/v1/backups**.

+ singular and shortNames

They represent the name displayed and make CLI usage easier.

+ kind

A CamelCased singular type used in resource manifests.

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

Note that the **apiVersion** and **kind** match the CRD we created in a previous step. The **spec** parameters depend on the controller.

The object will be evaluated by the controller. If the syntax, such as **timeSpec**, does not match the expected value, you will receive and error, should validation be configured. Without validation, only the existence of the variable is checked, not its details.

### 2.4. 可选的hooks

Just as with built-in objects, you can use an asynchronous pre-delete hook known as a **Finalizer**. If an API **delete** request is received, the object metadata field **metadata.deletionTimestamp** is updated. The controller then triggers whichever finalizer has been configured. When the finalizer completes, it is removed from the list. The controller continues to complete and remove finalizers until the string is empty. Then, the object itself is deleted.

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

A feature in beta starting with v1.9 allows for validation of custom objects via the OpenAPI v3 schema. This will check various properties of the object configuration being passed by the API server. In the example above, the **timeSpec** must be a string matching a particular pattern and the number of allowed replicas is between 1 and 10. If the validation does not match, the error returned is the failed line of validation.

## 3. API聚合

The use of Aggregated APIs allows adding additional Kubernetes-type API servers to the cluster. The added server acts as a subordinate to kube-apiserver, which, as of v1.7, runs the aggregation layer in-process. When an extension resource is registered, the aggregation layer watches a passed URL path and proxies any requests to the newly registered API service. 

The aggregation layer is easy to enable. Edit the flags passed during startup of the kube-apiserver to include **--enable-aggregator-routing=true**. Some vendors enable this feature by default. 

The creation of the exterior can be done via YAML configuration files or APIs. Configuring TLS authorization between components and RBAC rules for various new objects is also required. A [sample API server](https://github.com/kubernetes/sample-apiserver) is available on GitHub. A project currently in the incubation stage is an [API server builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha) which should handle much of the security and connection configuration.