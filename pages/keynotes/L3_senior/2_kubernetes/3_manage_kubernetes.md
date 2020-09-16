---
title: 管理kubernetes集群
keywords: keynotes, senior, kubernetes, manage_kubernetes
permalink: keynotes_L3_senior_2_kubernetes_3_manage_kubernetes.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/3_manage_kubernetes
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 使用kubeadm增删master节点
- 使用kubeadm增删node节点
- 使用二进制安装的集群增删master节点
- 使用二进制安装的集群增删master节点

## 1. 使用kubeadm增加master/node节点

### 1.1. 查看token

下面那个23h的是我们的，发现还没过期。

``` bash
$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
424h2b.bmvx996zclz8uu5k   1h          2020-06-30T08:24:55Z   <none>                   Proxy for managing TTL for the kubeadm-certs secret        <none>
nzjpz8.vkfaw9phnwh32jol   23h         2020-07-01T06:24:55Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
[ec2-user@ip-10-0-12-135 ~]$
```

### 1.2. 合成加入集群的命令

+ 直接获取证书的hash值

``` bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb
```

+ 重新生成certificate key

``` bash
$ kubeadm init phase upload-certs --upload-certs
W0630 07:14:53.905530   28330 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
6e8d24cb72dc0e9096394d602f95b027907bcf3d16750e72cccde33c00177f74
```

+ 所以加入的命令为

``` bash
# master节点加入
  kubeadm join 10.0.1.94:6443 --token nzjpz8.vkfaw9phnwh32jol \
    --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb \
    --control-plane --certificate-key 6e8d24cb72dc0e9096394d602f95b027907bcf3d16750e72cccde33c00177f74

# node节点加入

kubeadm join 10.0.1.94:6443 --token nzjpz8.vkfaw9phnwh32jol \
    --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb
```

+ node节点加入也可以直接使用这个命令（证书过期会生成新的）

``` bash
$ kubeadm token create --print-join-command
W0630 07:13:25.055002   26926 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
kubeadm join 10.0.1.94:6443 --token 2pvbmf.24m09oruy70t6bzj     --discovery-token-ca-cert-hash sha256:89022963a3104da98a595443b6be361c7920700bd3f43fd29491eb0d4c18e0eb
```

## 2. 使用kubeadm删除master/node节点

+ 在master上执行

``` bash
kubectl drain k8s-node2 --delete-local-data --force --ignore-daemonsets
kubectl delete node k8s-node2
```

+ node上执行

``` bash
kubeadm reset
```

## 3. 使用二进制方式安装的集群增加master节点