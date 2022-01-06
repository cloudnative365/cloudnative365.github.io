---
title: Ingress简介
keywords: keynotes, senior, kubernetes, ingress
permalink: keynotes_L3_senior_3_ingress_1_ingress.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/1_ingress
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- ingress概念
- ingress控制器
- ingress原理
- ingress与service

## 1. ingress概念

+ ingress的中文意思是进入，也就是流量的入口，词缀`in`代表的是进，那么对应的，我们还有`e`，也就是`egress`，翻译为`出口流量`。我们可以把这个简单理解为由集群外部进入应用的请求和从应用发出到集群外部的请求，都需要经过`ingress`和`egress`进入和离开。

+ 在kubernetes中，ingress和egress实际上都是一种资源类型。我们会在资源清单中定义好相应的规则，让进入集群的流量按照我们的规则来转发请求。

  感觉这好像我们的负载均衡啊！其实ingress就是借助了一些带有负载均衡的软件来实现的，我们会单独拿出两个课时来详细介绍nginx和envoy。

  可以将 Ingress 配置为提供服务外部可访问的 URL、负载均衡流量、终止 SSL / TLS，以及提供基于名称的虚拟主机。Ingress 控制器 通常负责通过负载均衡器来实现 Ingress，尽管它也可以配置边缘路由器或其他前端来帮助处理流量。

+ 但是，不管是ingress还是egress，他们本身并不是kubernetes的核心资源，我们需要额外安装Ingress控制器，然后在控制器的资源指定要使用的ingress类型

  ``` yaml
  kubernetes.io/ingress.class: "nginx"
  ```

  

+ 官方对于ingress的定义如下

  > Ingress 公开了从集群外部到集群内 services 的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

  然后他给了一张丑丑的图

  ``` bash
      internet
          |
     [ Ingress ]
     --|-----|--
     [ Services ]
  ```

  意思是说，外部的流量是经过ingress到达Service的，也就是说Ingress和Service是互相帮助的地位，而Ingress是不可以取代Service的，这个我们后面也会有章节说。

+ 最后，我们今天要讨论的是ingress，应用最广泛的也是ingress，而egress在service mesh中才会用到，我们后面讲service mesh的时候再详细说。

## 2. ingress控制器

ingress作为集群中的一个API，是有一个控制器来管理ingress的。他与`kube-controller-manager`管理下的controllers是不同的，这个控制器不会随controller-manager的启动而自动启动。我们有[很多的控制器](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)可以选择：

+ AKS Application Gateway Ingress Controller is an ingress controller that enables ingress to AKS clusters using the Azure Application Gateway.
+ Ambassador API Gateway is an Envoy based ingress controller with community or commercial support from Datawire.
+ AppsCode Inc. offers support and maintenance for the most widely used HAProxy based ingress controller Voyager.
+ AWS ALB Ingress Controller enables ingress using the AWS Application Load Balancer.
+ Contour is an Envoy based ingress controller provided and supported by VMware.
+ Citrix provides an Ingress Controller for its hardware (MPX), virtualized (VPX) and free containerized (CPX) ADC for baremetal and cloud deployments.
+ F5 Networks provides support and maintenance for the F5 BIG-IP Controller for Kubernetes.
+ Gloo is an open-source ingress controller based on Envoy which offers API Gateway functionality with enterprise support from solo.io.
+ HAProxy Ingress is a highly customizable community-driven ingress controller for HAProxy.
+ HAProxy Technologies offers support and maintenance for the HAProxy Ingress Controller for Kubernetes. See the official documentation.
+ Istio based ingress controller Control Ingress Traffic.
+ Kong offers community or commercial support and maintenance for the Kong Ingress Controller for Kubernetes.
+ NGINX, Inc. offers support and maintenance for the NGINX Ingress Controller for Kubernetes.
+ Skipper HTTP router and reverse proxy for service composition, including use cases like Kubernetes Ingress, designed as a library to build your custom proxy
+ Traefik is a fully featured ingress controller (Let's Encrypt, secrets, http2, websocket), and it also comes with commercial support by Containous.

## 3. Ingress原理

ingress的底层是借助于带有虚拟主机，基于URL重定向，TLS/SSL，负载均衡等功能的软件来实现从集群外部请求导入集群内部。我们以nginx为例

+ 当我们部署好ingress控制器之后，会生成一个或者若干个nginx pod，但是里面没有任何规则。
+ 然后，我们就可以创建ingress资源了，创建ingress资源的时候就像写nginx的配置文件一样可以配置虚拟主机，负载均衡等等。
+ 当我们使用kubectl把清单文件提交给kubernetes之后，ingress控制器会负责把我们的配置文件转换成nginx的配置文件，丢给nginx作为他的配置文件使用。

![img](/pages/keynotes/L3_senior/3_ingress/pics/1_ingress/NGINX-Plus-Ingress-Controller-1-7-0_ecosystem-1024x535.png)

+ 同时，ingress-controller也是通过service方式暴露出来的，ingress的service默认会暴露80和443端口，如果没有添加任何规则的时候，我们通过ingress请求后端服务的时候是会显示404的。

## 4. ingress与service

刚才说了，ingress和service是相辅相成的，只不过请求不是由service转发的，因为service的ClusterIP会被设置为NONE，也就是我们常说的headless，无头服务。在这里，Service把请求的转发权让给了ingress，但是其他的功能，比如endpoint的管理还是由service通过nodeselector来管理的，只不过service不会去转发请求了，因为规则是NONE。

也正是因为ingress是基于7层的转发，所以我们可以做很多的规则限制，从而实现虚拟主机，基于URL重定向，TLS/SSL，负载均衡等功能，而service是基于4层的，可以做的事情很少，所以真正的生产中，我们就需要借助于ingress来做入口流量控制。

最后，我们在工作中，通常会把ingress的流量叫做入栈流量，egress的叫出栈流量。这样叫起来是不是有一点点专业的味道了呢。