---
title: 安装ingress-nginx
keywords: keynotes, senior, kubernetes, install_ingress_nginx
permalink: keynotes_L3_senior_3_ingress_2_install_ingress_nginx.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/2_install_ingress_nginx
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 安装ingress控制器
- 创建Ingress资源
- 认识Ingress类
- Ingress用法
- 更新Ingress

## 1. 安装ingress控制器

[nginx官方文档](https://docs.nginx.com/nginx-ingress-controller/installation/?_ga=2.87051160.1809112633.1593511491-2130300903.1593399596)上提供了三种安装方法，镜像，manifest或者是helm。我们比较常用的是使用manifest或者helm，咱们这里使用manifest安装。

### 1.1. 使用manifest安装

+ 从git上下载

  ``` bash
  $ git clone https://github.com/nginxinc/kubernetes-ingress/
  $ cd kubernetes-ingress/deployments
  $ git checkout v1.7.2
  ```

+ 创建namespace

  ``` bash
  $ kubectl apply -f common/ns-and-sa.yaml
  ```

+ 配置RBAC

  ``` bash
  $ kubectl apply -f rbac/rbac.yaml
  ```

+ 创建TLS证书

  ``` bash
  $ kubectl apply -f common/default-server-secret.yaml
  ```

+ nginx的配置文件

  ``` bash
  $ kubectl apply -f common/nginx-config.yaml
  ```

+ 自定义资源

  ``` bash
  # VirtualServer
  $ kubectl apply -f common/vs-definition.yaml
  # VirtualServerRoute
  $ kubectl apply -f common/vsr-definition.yaml
  # TransportServer
  $ kubectl apply -f common/ts-definition.yaml
  ```

+ （可选）如果想配置四层（TCP，UDP）的负载均衡就运行下面的

  ``` bash
  # 自定义资源
  $ kubectl apply -f common/gc-definition.yaml
  # 全局配置
  $ kubectl apply -f common/global-configuration.yaml
  ```

+ 配置Ingress控制器

  ``` bash
  $ kubectl apply -f deployment/nginx-ingress.yaml
  ```

+ 或者选择DeamonSet的Ingress控制器

  ``` bash
  $ kubectl apply -f daemon-set/nginx-ingress.yaml
  ```

+ 创建nodePort方式的Service

  ``` bash
  $ kubectl create -f service/nodeport.yaml
  ```


### 1.2. 使用helm安装

不建议使用这种方式安装，因为使用helm安装会使用google的镜像源，需要先下载manifest文件进行编辑，然后再安装，而且需要手动指定名称空间，实在不太方便，所以这里并不推荐

## 2. 创建Ingress

我们先来看一个最简单的例子

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          serviceName: test
          servicePort: 80
```

既然是k8s的一个资源类型，那么他就一定是一个对象，在定义它的时候就需要使用`apiversion`,`kind`,`metadata`和`spec`字段

### 2.1. Ingress的规则

每个HTTP规则都包含下面的信息

- 可选主机。在此示例中，未指定主机，因此该规则适用于通过指定 IP 地址的所有入站 HTTP 通信。如果提供了主机（例如 foo.bar.com），则规则适用于该主机。
- 路径列表（例如，`/testpath`）,每个路径都有一个由 `serviceName` 和 `servicePort` 定义的关联后端。在负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。
- 后端是 [Service 文档](https://kubernetes.io/docs/concepts/services-networking/service/)中所述的服务和端口名称的组合。与规则的主机和路径匹配的对 Ingress 的 HTTP（和 HTTPS ）请求将发送到列出的后端。

通常在 Ingress 控制器中配置默认后端，以服务任何不符合规范中路径的请求。

### 2.2. 默认的backend

没有规则的 Ingress 将所有流量发送到单个默认后端。默认后端通常是 [Ingress 控制器](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)的配置选项，并且未在 Ingress 资源中指定。

如果没有主机或路径与 Ingress 对象中的 HTTP 请求匹配，则流量将路由到您的默认后端。

### 2.3. 路径类型

Ingress 中的每个路径都有对应的路径类型。支持三种类型：

- *`ImplementationSpecific`* （默认）：对于这种类型，匹配取决于 IngressClass. 具体实现可以将其作为单独的 `pathType` 处理或者与 `Prefix` 或 `Exact` 类型作相同处理。
- *`Exact`*：精确匹配 URL 路径且对大小写敏感。
- *`Prefix`*：基于以 `/` 分割的 URL 路径前缀匹配。匹配对大小写敏感，并且对路径中的元素逐个完成。路径元素指的是由 `/` 分隔符分割的路径中的标签列表。如果每个 *p* 都是请求路径 *p* 的元素前缀，则请求与路径 *p* 匹配。

### 2.4. 多重匹配

在某些情况下，Ingress 中的多条路径会匹配同一个请求。这种情况下最长的匹配路径优先。如果仍然有两条同等的匹配路径，则精确路径类型优先于前缀路径类型。

## 3. Ingress 类

Ingress 可以由不同的控制器实现，通常使用不同的配置。每个 Ingress 应当指定一个类，一个对 IngressClass 资源的引用，该资源包含额外的配置，其中包括应当实现该类的控制器名称。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com/v1alpha
    kind: IngressParameters
    name: external-lb
```

IngressClass 资源包含一个可选的参数字段。可用于引用该类的额外配置。

### 3.1. 废弃的注解

在 IngressClass 资源和 `ingressClassName` 字段被引入 Kubernetes 1.18 之前，Ingress 类是通过 Ingress 中的一个 `kubernetes.io/ingress.class` 注解来指定的。这个注解从未被正式定义过，但是得到了 Ingress 控制器的广泛支持。

Ingress 中新的 `ingressClassName` 字段是该注解的替代品，但并非完全等价。该注解通常用于引用实现该 Ingress 的控制器的名称， 而这个新的字段则是对一个包含额外 Ingress 配置的 IngressClass 资源的引用，包括 Ingress 控制器的名称。

### 3.2. 默认 Ingress 类

您可以将一个特定的 IngressClass 标记为集群默认项。将一个 IngressClass 资源的 `ingressclass.kubernetes.io/is-default-class` 注解设置为 `true` 将确保新的未指定 `ingressClassName` 字段的 Ingress 能够分配为这个默认的 IngressClass.

> **注意：** 如果集群中有多个 IngressClass 被标记为默认，准入控制器将阻止创建新的未指定 `ingressClassName` 字段的 Ingress 对象。 解决这个问题只需确保集群中最多只能有一个 IngressClass 被标记为默认。

## 4. Ingress用法

### 4.1. 单服务 Ingress

现有的 Kubernetes 概念允许您暴露单个 Service (查看[替代方案](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#alternatives))，您也可以通过指定无规则的 *默认后端* 来对 Ingress 进行此操作。

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```



如果使用 `kubectl apply -f` 创建它，则应该能够查看刚刚添加的 Ingress 的状态：

```shell
kubectl get ingress test-ingress
NAME           HOSTS     ADDRESS           PORTS     AGE
test-ingress   *         203.0.113.123     80        59s
```

其中 `203.0.113.123` 是由 Ingress 控制器分配以满足该 Ingress 的 IP。

> **说明：** 入口控制器和负载平衡器可能需要一两分钟才能分配IP地址。 在此之前，您通常会看到地址字段的值被设定为 `<pending>`。

### 4.2. 简单分列

一个分列配置根据请求的 HTTP URI 将流量从单个 IP 地址路由到多个服务。 Ingress 允许您将负载均衡器的数量降至最低。例如，这样的设置：

```none
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```

将需要一个 Ingress，例如：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```

当您使用 `kubectl apply -f` 创建 Ingress 时：

```shell
kubectl describe ingress simple-fanout-example
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

Ingress 控制器将提供实现特定的负载均衡器来满足 Ingress，只要 Service (`service1`，`service2`) 存在。 当它这样做了，您会在地址栏看到负载均衡器的地址。

> **说明：** 根据您使用的 [Ingress 控制器](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)，您可能需要创建默认 HTTP 后端 [Service](https://kubernetes.io/docs/concepts/services-networking/service/)。

### 4.3. 基于名称的虚拟托管

基于名称的虚拟主机支持将 HTTP 流量路由到同一 IP 地址上的多个主机名。

```none
foo.bar.com --|                 |-> foo.bar.com service1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com service2:80
```

以下 Ingress 让后台负载均衡器基于[主机 header](https://tools.ietf.org/html/rfc7230#section-5.4) 路由请求。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```

如果您创建的 Ingress 资源没有规则中定义的任何主机，则可以匹配到您 Ingress 控制器 IP 地址的任何网络流量，而无需基于名称的虚拟主机。

例如，以下 Ingress 资源会将 `first.bar.com` 请求的流量路由到 `service1`，将 `second.foo.com` 请求的流量路由到 `service2`，而没有在请求中定义主机名的 IP 地址的流量路由（即，不提供请求标头）到 `service3`。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: second.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
  - http:
      paths:
      - backend:
          serviceName: service3
          servicePort: 80
```

### 4.4. TLS

您可以通过指定包含 TLS 私钥和证书的 secret [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 来加密 Ingress。 目前，Ingress 只支持单个 TLS 端口 443，并假定 TLS 终止。

如果 Ingress 中的 TLS 配置部分指定了不同的主机，那么它们将根据通过 SNI TLS 扩展指定的主机名（如果 Ingress 控制器支持 SNI）在同一端口上进行复用。 TLS Secret 必须包含名为 `tls.crt` 和 `tls.key` 的密钥，这些密钥包含用于 TLS 的证书和私钥，例如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

在 Ingress 中引用此 Secret 将会告诉 Ingress 控制器使用 TLS 加密从客户端到负载均衡器的通道。您需要确保创建的 TLS secret 来自包含 `sslexample.foo.com` 的公用名称（CN）的证书，也被称为全限定域名（FQDN）。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - sslexample.foo.com
    secretName: testsecret-tls
  rules:
    - host: sslexample.foo.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service1
            servicePort: 80
```

> **说明：** 各种 Ingress 控制器所支持的 TLS 功能之间存在差异。请参阅有关文件 [nginx](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)、 [GCE](https://git.k8s.io/ingress-gce/README.md#frontend-https) 或者任何其他平台特定的 Ingress 控制器，以了解 TLS 如何在您的环境中工作。

### 4.5. 负载均衡

Ingress 控制器使用一些适用于所有 Ingress 的负载均衡策略设置进行自举，例如负载均衡算法、后端权重方案和其他等。更高级的负载均衡概念（例如，持久会话、动态权重）尚未通过 Ingress 公开。您可以通过用于服务的负载均衡器来获取这些功能。

值得注意的是，即使健康检查不是通过 Ingress 直接暴露的，但是在 Kubernetes 中存在并行概念，比如 [就绪检查](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)，它允许您实现相同的最终结果。 请检查控制器特殊说明文档，以了解他们是怎样处理健康检查的 ( [nginx](https://git.k8s.io/ingress-nginx/README.md)， [GCE](https://git.k8s.io/ingress-gce/README.md#health-checks))。

## 5. 更新 Ingress

要更新现有的 Ingress 以添加新的 Host，可以通过编辑资源来对其进行更新：

```shell
kubectl describe ingress test
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     35s                loadbalancer-controller  default/test
kubectl edit ingress test
```

这将弹出具有 YAML 格式的现有配置的编辑器。 修改它来增加新的主机：

```yaml
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
        path: /foo
  - host: bar.baz.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
        path: /foo
..
```

保存更改后，kubectl 将更新 API 服务器中的资源，该资源将告诉 Ingress 控制器重新配置负载均衡器。

验证：

```shell
kubectl describe ingress test
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
  bar.baz.com
               /foo   service2:80 (10.8.0.91:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     45s                loadbalancer-controller  default/test
```

您可以通过 `kubectl replace -f` 命令调用修改后的 Ingress yaml 文件来获得同样的结果。