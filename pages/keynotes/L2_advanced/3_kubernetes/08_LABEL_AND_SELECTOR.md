---
title: 标签和标签选择器
keywords: keynotes, L2_advanced, kubernetes, CKA, LABEL_AND_SELECTOR
permalink: keynotes_L2_advanced_1_kubernetes_3_kubernetes_8_LABEL_AND_SELECTOR.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/8_LABEL_AND_SELECTOR
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- 标签Label
- 标签选择器Selector

## 1. 标签
在我们使用pod的时候，我们都会使用标签来标记我们的pod，因为我们的pod会越来越多，将来我们分类的时候，将pod分成许多较小的分组，都能够显著的提高管理效率，更何况，我们的控制器，servier资源也需要使用识别他们所管控或者关联到的资源。当我们给任何资源设定标签以后，还可以使用标签来查看和删除等管理操作。 标签是k8s上极具特色的功能之一，所谓的标签就是附加在对象上面的键值对。一个资源之上可存在多个标签，每个标签都是一个键值对。而且每一个标签都可以被标签选择器做匹配度检查，从而完成资源挑选。通常情况下，一个资源对象可使用多个标签，而一个标签可以被贴在多个资源对象上，他们是多对多的关系。另外，标签既可以在资源创建时指定，也可以在资源创建之后，添加，删除或者修改。

``` json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```



### 1.1. 用途

生产系统上，我们通常会对资源添加多个维度的标签，从而实现多个维度的管理。比如：

- `"release" : "stable"`, `"release" : "canary"`
- `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
- `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
- `"partition" : "customerA"`, `"partition" : "customerB"`
- `"track" : "daily"`, `"track" : "weekly"`

下面是一个best practice，指导我们如何在生产系统上打标签

![file](/pages/keynotes/L2_advanced/3_kubernetes/pics/8_LABEL_AND_SELECTOR/2220a321c709574285f1301587484008.png)

资源标签的名称的key和value必须小于或者等于63个字符，字母 数字 点号 下划线 连接线，且必须以字母或者数字开头。我们使用键名的时候也可以使用键前缀，dns的域名当前缀，但是加前缀不能超过253个字符。标签中的值和key一样，但是可以为空，而且只能以字母和数字开头和结尾。

### 1.2. 操作标签

我们在查看pod或者任何资源时，也可以直接指定标签选择器，来选择能够选择那些标签。

 ``` bash
 #表示对象资源拥有标签app
 kubectl get pods -l app
 #还可以同时指定多个标签
 kubectl get pods -l app,run
 #显示指定资源的对象时，对每一类的资源的标签，显示其标签值
 kubectl get pods -L app, run
 #给资源对象打标
 kubectl label pods pod-demo release=alpha
 #如果给已经有的标签强行修改
 kubectl label pods pod-demo release=stable --overwrite
 ```

标签选择器在选择的时候支持两类，一个是等值关系的标签选择器，一个是基于集合的标签选择器。另外我们在使用标签选择器的时候，可以指定多个条件。如果使用多个条件，内部生成的隐含的关系叫逻辑与，两个选择器都要满足。如果我们不使用标签选择器，就表示选择空的标签选择器，就表示我们使用所有的对象。

+ 基于等值的：=，或者==（等于）!=（不等于）

 ``` bash
 kubectl get pods -l app=myapp
 kubectl get pods -l app=myapp, release=stable
 ```

+ 基于集合关系的：key in （val1， val2）或者 key notin（val1，val2）

 ``` bash
  kubectl get pods -l "release in (canary, beta, alpha)"
	kubectl get pods -l "release notin (canary, beta, alpha)"
 ```



## 2. 标签选择器

有很多资源都需要使用标签来关联相关资源的，比如控制器和service。pod控制器和service这种资源来关联pod的时候会使用另外两个字段来嵌套，一个叫matchLables，一个叫matchExpressions。deployment，service等资源通常支持使用matchLables来定义怎样使用标签选择器。

```yaml
+ matchLables：表示直接给定键值
+ matchExpressions：基于给定的表达式来定义标签选择器。{key: "KEY", operator: "POERATOR", values:[VAL1, VAL2, ...]}，这个表示把key和values做operator的比较，这种一般使用逻辑关系操作符，in，notin，exist，not exist
  + in, notin：values字段的值必须为非空列表
  + exist，not exist：values字段的值必须为空列表
```
### 2.1. nodeSelector

其实不止pod，其他所有资源都都可以打标签，比如node，我们也可以使用get node --show-labels来显示标签，也可以使用label来打标签，随后，就可以让我们的资源有倾向性。

我们在定义pod的时候，有一个内建字段叫nodeSelector，他能够让我们自由的选择pod运行在哪个节点上。比如，我们需要pod运行在拥有标签disk=ssd的节点上，我们就需要使用nodeSelector。他可以影响调度算法，让pod永远运行在拥有这个标签的机器上，同时，我们还有nodeName，直接指定到某个节点上
``` yaml
apiVersion: v1
kind: Pod
metadata:
  namespace:default
  name: pod-demo
	lables:
	  app: myapp
		tire: frontend
spec:
  containers:
	- name: myapp
	  image: kubernetes/myapp:v1
  - name: busybox
    image: busybox
		command:
		- "/bin/sh"
		- "-c"
		- "sleep 3600"
  nodeSelector:
	  disk: ssd
```

### 2.2. annotations

同时我们还可以使用注解：annotations，他和labels很像，但是区别是annotations不能用于挑选资源对象，仅用于为对象提供”元数据“，一般说来，他会作为外部程序挑选资源的标准，也可以认为是外部的label。而且，他的字符长度没有要求。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
```

查看annotation没有专门的命令，必须使用inspect。