---
title: 最困难的方式安装k8s集群
keywords: keynotes, senior, kubernetes, hardest_way
permalink: keynotes_L3_senior_2_kubernetes_2_hardest_way.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/2_hardest_way
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. 简介

我们都知道在kubernetes的业界有一个标准，叫the easy way of kubernetes和the hard way of kubernetes。他们分别是指kops安装kubernetes和二进制方式安装kubernetes。二进制方式安装的文档在[github](https://github.com/kelseyhightower/kubernetes-the-hard-way)上。在写这篇文档的时候，我参考了一下git上文档，他已经更新到了1.15这个版本，而我目前在使用的是1.18.3版本。所以我打算在增加点难度。



## 2. 加大难度

在看git的文档的时候，我发现还可以加大难度。

+ 整篇文章只用了一组CA签署了一组证书，但是如果使用kubeadm安装的时候是两组CA（etcd一组ca和kubenetes的control-panel一组ca）创建了八组证书(etcd三组和kubernetes五组)
+ 整篇文章没有考虑权限和硬盘规划问题，我觉得还可以在权限和硬盘规划上再增加难度，做成生成级别的系统



## 3. 架构图

![External etcd topology](/pages/keynotes/L3_senior/2_kubernetes/pics/2_hardest_way/kubeadm-ha-topology-external-etcd.svg)来自[网站](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)



## 4. 软硬件清单

### 4.1. 硬件环境

平台环境：AWS

机器列表

| hostname   | 型号      | 功能                                | 子网 | IP地址 |
| ---------- | --------- | ----------------------------------- | ---- | ------ |
| master1    | t3.medium | control-plane                       |      |        |
| master2    | t3.medium | control-plane                       |      |        |
| master3    | t3.medium | control-plane                       |      |        |
| master1    | t3.medium | etcd                                |      |        |
| master2    | t3.medium | etcd                                |      |        |
| master3    | t3.medium | etcd                                |      |        |
| jumpserver | t3.medium | load balancer for internal traffic  |      |        |
| jumpserver | t3.medium | load balancer for external traffic  |      |        |
| jumpserver | t3.medium | jumpserver for managing the cluster |      |        |
| node1      | t3.medium | worker                              |      |        |
| node2      | t3.medium | worker                              |      |        |



### 4.2. 软件环境

操作系统：ubuntu：16.04.6

docker：19.03.4

containerd：1.2.10

kubernetes：1.18.3

CoreDNS：

Calico：



## 5. 准备环境

### 5.1. 在公有云环境中创建硬件清单中的机器

### 5.2. 配置LB机器为我们的跳板机，配置他免密登录其他的机器



## 6. 安装工具

### 6.1. 下载相应的工具

### 6.2. 下载相应的包



## 补充：使用kubeadm完成下面的7到9步

参考[官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)



## 7. 安装etcd



## 8. 创建证书（etcd的三组）



## 9. 配置带ssl认证的etcd



## 10. 安装负载均衡



## 补充：使用kubeadm完成下面的11到14步

参考：[官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)



## 11. 创建kubernetes证书（kubernetes的五组）



## 12. 安装api-server



## 13. 安装api-scheduler



## 14. 安装api-controller-manager



## 15. 安装kubelet



## 16. 安装CoreDNS



## 17. 安装Calico



## 18. 安装Ingress，dashboard，helm，prometheus



所有的key

``` bash
root@ip-10-0-1-80:/etc/kubernetes/pki# find .
.
./ca.crt
./apiserver-kubelet-client.crt
./ca.key
./front-proxy-client.key
./sa.pub
./sa.key
./front-proxy-ca.crt
./front-proxy-ca.key
./apiserver.key
./apiserver.crt
./apiserver-etcd-client.crt
./etcd
./etcd/ca.crt
./etcd/server.crt
./etcd/healthcheck-client.crt
./etcd/ca.key
./etcd/peer.crt
./etcd/server.key
./etcd/peer.key
./etcd/healthcheck-client.key
./front-proxy-client.crt
./apiserver-kubelet-client.key
./apiserver-etcd-client.key
```

etcd

``` bash
root@ip-10-0-1-80:/etc/kubernetes/manifests# cat etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://10.0.1.80:2379
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://10.0.1.80:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://10.0.1.80:2380
    - --initial-cluster=ip-10-0-1-80=https://10.0.1.80:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.0.1.80:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://10.0.1.80:2380
    - --name=ip-10-0-1-80
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: k8s.gcr.io/etcd:3.4.3-0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /health
        port: 2381
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: etcd
    resources: {}
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```

api-server

``` bash
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.0.1.80:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.0.1.80
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver:v1.18.3
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 10.0.1.80
        path: /healthz
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-apiserver
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
status: {}
```

controller

``` bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --node-cidr-mask-size=24
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --use-service-account-credentials=true
    image: k8s.gcr.io/kube-controller-manager:v1.18.3
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10257
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-controller-manager
    resources:
      requests:
        cpu: 200m
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /usr/libexec/kubernetes/kubelet-plugins/volume/exec
      name: flexvolume-dir
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /etc/kubernetes/controller-manager.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec
      type: DirectoryOrCreate
    name: flexvolume-dir
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /etc/kubernetes/controller-manager.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
status: {}
```

scheduler

``` bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    image: k8s.gcr.io/kube-scheduler:v1.18.3
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
status: {}
```



https://github.com/kubernetes-sigs/kubespray)