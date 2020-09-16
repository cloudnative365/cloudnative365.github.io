---
title: INGRESS和INGRESS控制器
keywords: keynotes, advanced, kubernetes, CKA, INGRESS
permalink: keynotes_L2_advanced_3_kubernetes_15_INGRESS.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/15_INGRESS
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. INGRESS简介
在前面的章节中，我们学习了如何使用service在集群外暴露容器化应用程序。我们使用Ingress控制器和Rule来执行相同的功能。区别在于效率。您可以根据请求主机或路径路由流量，而不是使用Service（如LoadBalancer）。这允许将许多服务从一个地址来访问后端。从而实现了7层负载均衡。

INGRESS控制器不同于大多数控制器，因为它不作为kube-controller-manager二进制文件的一部分运行。我们可以部署多个控制器，每个控制器具有唯一的配置。控制器使用Ingress规则来处理进出集群外部的流量。

有许多入口控制器，如GKE，nginx，Traefik，Contour和Envoy等。任何能够反向代理的工具都应该可以正常工作。这些代理使用规则并侦听特定的流量。Ingress是一个API资源，可以使用**kubectl**创建。创建该资源时，它会重新编排并重新配置Ingress控制器，以允许流量从外部流向内部服务。我们可以将Service配置为ClusterIP类型，它的值为none，并使用Ingress规则定义如何将流量路由到该内部服务。

Ingress 暴露了从集群外部到集群内 services 的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

可以将 Ingress 配置为提供服务外部可访问的 URL、负载均衡流量、终止 SSL / TLS，以及提供基于名称的虚拟主机。Ingress 控制器 通常负责通过负载均衡器来实现 Ingress，尽管它也可以配置边缘路由器或其他前端来帮助处理流量。

Ingress 不会公开任意端口或协议。 将 HTTP 和 HTTPS 以外的服务公开到 Internet 时，通常使用 Service.Type=NodePort 或者 Service.Type=LoadBalancer 类型的服务。

## 2. Ingress Controller



Ingress Controller是一个运行在Pod中的守护进程，它监视API服务器上的**入口**端点，新对象创建时，该资源位于**networking.k8s.io/v1beta1**组下。当创建新的endpoint时，守护进程使用配置的规则集来允许到服务的入站连接，通常是HTTP通信。这使得无论Pod部署在哪里，都可以通过边缘路由器轻松访问Pod服务。

可以部署多个入口控制器。流量应该使用annotations来选择正确的控制器。缺少匹配的注释将导致每个控制器尝试满足入口流量。

![5ln4zg2183da-TheIngress](/pages/keynotes/L2_advanced/3_kubernetes/pics/15_INGRESS/5ln4zg2183da-TheIngress.png)

## 3. 使用nginx的Ingress

通过使用提供的YAML文件，部署nginx控制器变得很容易，可以在 [ingress-nginx/deploy GitHub repository](https://github.com/kubernetes/ingress-nginx/tree/master/deploy)找到相关的配置

这个页面有配置文件，可以在多个平台上配置nginx，比如AWS、GKE、Azure和bare metal等等。

对于任何Ingress控制器，都有一些正确部署的配置要求。可以通过ConfigMap、Annotations或自定义模板进行自定义：

- Easy integration with RBAC （与RBAC轻易集成）
- Uses the annotation **kubernetes.io/ingress.class: "nginx"**
- L7 traffic requires the **proxy-real-ip-cidr** setting（七层流量需要配置proxy-real-ip-cidr配置）
- Bypasses kube-proxy to allow session affinity（可以通过kube-proxy实现session的粘性）
- Does not use **conntrack** entries for iptables DNAT（不需要使用iptables DNAT的conntrack入口）
- TLS requires the host field to be defined.（TLS的配置需要在host的地方去定义）

## 4. Ingress API资源

Ingress对象属于 networking.k8s.io这个API，但是仍然是一个beta版本的对象。典型的Ingress对象我们可以使用下面的清单文件来创建

``` yaml
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress 
metadata:
  name: ghost
spec:
  rules:
    - host: ghost.192.168.99.100.nip.io
http:
paths:
    - backend:
            serviceName: ghost
            servicePort: 2368 
```

我们可以像管理其他资源一样管理ingress

``` bash
$ kubectl get ingress

$ kubectl delete ingress <ingress_name>

$ kubectl edit ingress <ingress_name>
```

## 5. 部署Ingress Controller

部署Ingress Controller非常简单，我们可以直接使用kubectl来部署。部署的样例可以在[GitHub](https://github.com/kubernetes/ingress-nginx/tree/master/deploy)

``` bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/baremetal/deploy.yaml
```

部署完成之后会生产一些由RC管理的pod还有一些service。默认的HTTP后端会返回404页面

``` bash
$ kubectl get pods,rc,svc
NAME                               READY  STATUS   RESTARTS  AGE
po/default-http-backend-xvep8      1/1    Running  0         4m
po/nginx-ingress-controller-fkshm  1/1    Running  0         4m

NAME                     DESIRED  CURRENT  READY  AGE
rc/default-http-backend  1        1        0      4m

NAME                      CLUSTER-IP  EXTERNAL-IP  PORT(S)  AGE
svc/default-http-backend  10.0.0.212  <none>       80/TCP   4m
svc/kubernetes            10.0.0.1    <none>       443/TCP  77d
```

## 6. 创建Ingress规则

我们在刚才的环境中创建一个规则

``` bash
$ kubectl run ghost --image=ghost

$ kubectl expose deployments ghost --port=2368
```

deployment暴露和Ingress规则创建之后，我们就可以从外部访问应用了

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
    name: ghost
spec:
    rules:
    - host: ghost.192.168.99.100.nip.io
      http:
      paths:
      - backend:
            serviceName: ghost
            servicePort: 2368
```

定义多个规则

在刚才的例子中，我们定义了一个规则。如果我们还有更多的服务，我们可以在一个Ingress中定义多个规则

``` yaml
rules:
- host: ghost.192.168.99.100.nip.io
  http:
    paths:
    - backend:
        serviceName: ghost
        servicePort: 2368
- host: nginx.192.168.99.100.nip.io
  http:
    paths:
    - backend:
        serviceName: nginx
        servicePort: 80
```

