---
title: API
keywords: keynotes, advanced, kubernetes, CKA, API
permalink: keynotes_L2_advanced_3_kubernetes_13_APIS_AND_ACCESS.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/13_APIS_AND_ACCESS
typora-root-url: ../../../../../cloudnative365.github.io
---

## 课程目标
- 了解API是基于REST风格的
- 使用annotation
- 使用kubelet进行trouble shooting
- 使用namespace区分集群资源

## API和准入控制

### API

kubernetes拥有一个强大的REST风格的API。整个的架构都是API风格的。了解去哪能才能找到资源的endpoints和了解API是怎样在不同版本见切换的对于我们进行管理工作是很重要的，因为有许多正在进行的API变更还有API的增加都在同步进行。从1.16版本之后废弃的对象讲不再由API server支持。

我们在前面的kubernetes架构一章中学过，不管是内部，还是外部，负责通信的只有APIserver。我们可以使用`curl`去查询当前暴露的API组。组可能有很多的版本，他们逐渐的和其他组分离开来，并且都在一个domian-name之下，有很多的名称。



我们可以使用`kubectl`的子命令`api-versions`来查询当前kubernetes下所有的API，下面的例子是1.17版本的

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

我们看到他们的格式都是group/version

- group：如果不加group，groups就是core。
- version：有alpha，beta还有stable的。而stable是没有标识的，比如v1。如果是alpha版本，不建议使用，如果是beta版本，这个是可以使用的，即使以后会有小的修改，也不会像alpha一样有可能随时被废弃。

### RESTful

kubectl访问API的时候，其实就是把我们的请求转化成了HTTP的请求（GET，POST，DELETE）。我们也可以直接使用curl或者外部的工具去访问，但是在访问的时候需要加上证书和key，可以直接访问，也可以加上json请求

``` bash
$ curl --cert userbob.pem --key userBob-key.pem \  
--cacert /path/to/ca.pem \   
https://k8sServer:6443/api/v1/pods 
```

在配置了RBAC的集群当中，这样我们就可以模拟其他用户或者组去请求API服务器。这样可以帮助我们找出其他用户的认证策略中的漏洞

### 检查准入权限

由于后面还有一节是专门介绍安全的，我们这里就演示一下管理员账户和其他账户的访问权限。

下面的例子是说用户bob在default名称空间和developer名称空间中的不同权限。可以使用`auth can-i`子命令来查询

``` bash
$ kubectl auth can-i create deployments
yes 

$ kubectl auth can-i create deployments --as bob
no 

$ kubectl auth can-i create deployments --as bob --namespace developer
yes 
```

目前有三种API是用来配置谁和什么可以被访问

- **SelfSubjectAccessReview**

  任何用户的准入都需要审查，授权给其他的准入控制

- **LocalSubjectAccessReview**

  审查指定的名称空间

- **SelfSubjectRulesReview**

  审核指定名称空间中是否可以做某些操作

使用**reconcile**可以检查从文件创建对象所需的授权。没有输出指示将允许创建。

### 使用kubectl管理资源

kubernetes通过RESTful风格的API暴露资源，允许所有的资源通过HTTP，JSON或者是XML的格式被访问，最典型的协议就是HTTP。我们可以使用标准的HTTP请求来访问，比如GET,POST,PATCH,DELETE。

**kubectl**有一个verbose mode参数，该参数显示命令从何处获取和更新信息的详细信息。其他输出包括**curl**命令，您可以使用这些命令来获得相同的结果。虽然verbosity接受从零到任意数字的级别，但当前没有大于10的verbosity值。

``` bash
$ kubectl --v=10 get pods firstpod

....
I1215 17:46:47.860958 29909 round_trippers.go:417]
curl -k -v -XGET -H "Accept: application/json"
-H "User-Agent: kubectl/v1.8.5 (linux/amd64) kubernetes/cce11c6"
https://10.128.0.3:6443/api/v1/namespaces/default/pods/firstpod
....
```

如果我们删除pod我们就会看到请求从XGET变成了XDELETE

``` bash
$ kubectl --v=10 delete pods firstpod

....
I1215 17:49:32.166115 30452 round_trippers.go:417]
curl -k -v -XDELETE -H "Accept: application/json, */*"
-H "User-Agent: kubectl/v1.8.5 (linux/amd64) kubernetes/cce11c6"
https://10.128.0.3:6443/api/v1/namespaces/default/pods/firstpod
....
```

### 从集群外访问集群

主要的访问集群的工具是kubectl，但是我们也可以使用curl。

基础的服务器信息包块TLS证书信息都可以通过这个命令查看

``` bash
$ kubectl config view 
```

如果我们仔细查看输出，我们会发现，这些内容实际上都是来自于一个文件，叫**~/.kube/config**

``` bash
I1215 17:35:46.725407 27695 loader.go:357] 
     Config loaded from file /home/student/.kube/config 
```

如果没有此文件中的证书颁发机构、密钥和证书，则只能使用不安全的**curl**命令，也就是使用kubectl proxy在本地启动一个代理

### ~/.kube/config

我们来看一下这个文件的内容

``` yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdF.....
    server: https://10.128.0.3:6443 
    name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTib.....
    client-key-data: LS0tLS1CRUdJTi....
```

感觉很想我们的资源清单文件

+ apiVersion

他是一种对象，来指明kube-apiserver应该去哪里找到他的对象

+ clusters

包含集群的名称，以及发送API调用的位置。传递**certificate-authority-data**来验证curl请求。

+ contexts

允许从一个配置文件轻松访问多个集群（可能是多个用户）。可以设置**命名空间**、**用户**、**集群**。

+ current-context

**kubectl**命令将使用哪个集群和用户。这个设置也可以按命令传递。

+ kind

对象的类型是config

+ preferences

目前没有用，这个是kubectl命令的可选项，可以控制文本颜色

+ users

与客户端凭据关联的昵称，可以是客户端密钥和证书、用户名和密码以及令牌。令牌和用户名/密码是不能同时存在的。这些可以通过**kubectl config set credentials**命令配置。