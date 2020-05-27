---
title: 使用清单文件管理资源
keywords: keynotes, L2_advanced, kubernetes, CKA, MANAGE_RESOURCES_WITH_MANIFEST
permalink: keynotes_L2_advanced_3_kubernetes_6_MANAGE_RESOURCES_WITH_MANIFEST.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/5_MANAGE_RESOURCES_WITH_MANIFEST
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 课程目标
- 认识最简单的资源清单manifest
- 认识apiVersion
- 认识kind
- 认识metadata
- 认识spec

## 2. 认识最简单的资源清单
我们在创建资源之后，可以使用kubectl的get选项来查看资源的相关信息
``` bash
$ kubectl get pod nginx-6db489d4b7-4hp48 -o yaml
apiVersion: v1
kind: Pod
metadata:
.
.
spec:
.
.
status:
.
.
```
从上面的输出我们可以看出来，在第一级的一共有5个选项，他们分别是apiVersion，kind，metadata，spec和status。除了status是系统来维护的，是不可以手动更改的。其他都是我们在今后定义资源的时候需要常用的选项。

### 2.1. apiVersion
kubernetes的api-server是一个restful风格的web，用户可以通过GET，PUT等HTTP请求来管理资源，而他的http框架使用的是swagger。kubernetes把API切分成了多个组，而每一个组都可以自主和独立的进行迭代和更新。如果某一个组内的资源发生更新的时候，不会影响其他组内的资源。这样一来，我们来引用某一个对象及其内部的资源的时候，必须标明APIversion，以及他所在的群组。他的完整格式是 group/version。
  + group：如果不加group，groups就是core。
  + version：有alpha，beta还有stable的。而stable是没有标识的，比如v1。如果是alpha版本，不建议使用，如果是beta版本，这个是可以使用的，即使以后会有小的修改，也不会像alpha一样有可能随时被废弃。

 所以这个apiVersion: v1实际上就是core/v1。我们如果想要查看所有的api就需要使用`kubectl api-versions`
 ``` bash
 $ kubectl api-versions
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
discovery.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
 ```

### 2.2. kind
用来指明哪一种资源对象。对于我们的资源来讲，尤其是restful风格的资源来讲，内部就是以资源为主来构建事务类别的，而后我们可以把对应的类别中的属性来赋值，而实例化出对象来，因此pod是一中资源类别，而我们需要实例化出来某一种对象。有点像编程中的实例化。
我们可以在[源码](https://github.com/kubernetes/kubernetes/tree/master/pkg/api/v1)中找到v1下面除了pod之外的其他资源
![file](https://graph.baidu.com/resource/222dc50bdcaa3edcf499901588341864.png)

### 2.3. metadata
元数据，这个对于大多数的类型都是相似的。
+ name：资源的名称
+ namespace：在哪个名称空间
+ labels：标签
+ annotations：声明，这个声明一般是给第三方的应用使用的label
+ selflink：每个资源的引用PATH，/api/GROUP/VERSION/namespaces/NAMESPACE（资源所在的名称空间）/TYPE(资源类别)/NAME

### 2.4. spec
specification的缩写，用来定义我们接下来要创建的对象应该有什么样的属性，或者有什么样的规范。他是非常重要的类型，不同的资源，使用的spec各不相同。如果spec的字段被标记为require就是必须的。他是用来定义期望的状态的，disired state。刚开始创建资源的时候，有可以当前状态（下面的status）并不满足期望期望状态，那么控制器就会让当前状态不停的向期望状态靠近。如果副本数不满足，就会增减pod数量，如果版本数不符合，就会更换新的镜像。

spec中的选项非常的多，如果我们想查看某一个选项的话，可以使用`kubectl explain`命令，比如
```bash
kubectl explain pod.spec
```
这是说查看pod这个kind的spec清单中，有什么选项，如果某个选项的类型是`<object>`，我们就可以使用`pod.spec.<object>`的方式继续查看

### 2.5. 通过清单文件创建资源
我们做一个最简单的pod来试试
``` bash
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
spec:
  containers:
	- name: myapp
	  image: kubernetes/myapp:v1
```
文件创建好了最后，就可以使用`kubectl apply/create`命令来让文件生效了。
+ create一般是第一次创建资源
+ 而如果文件有改动, 第二次在让改动生效的时候，就需要使用apply命令。如果是第一次创建资源，也可以使用apply命令。

