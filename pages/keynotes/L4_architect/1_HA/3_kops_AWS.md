---
title: 使用kops在aws上创建集群
keywords: keynotes, architect, HA, kops_AWS
permalink: keynotes_L4_architect_1_HA_3_kops_AWS.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/3_kops_AWS
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 简介

Kops是一个用来创建生产级别的kubernetes集群的工具。他被戏称为Kubernetes the Easy Way。我们可以认为他是管理kubernetes集群的kubectl。他是使用命令行的方式，帮助我们创建，销毁，升级并且维护生产级别的，高可用的kubernetes集群。目前AWS是官方支持的，GCE和Openstack是在beta版本，Vmware是在alpha版本。

写这篇文章的时候是2020年3月17日，稳定版本是1.16，最新版是1.17-beta.1。也就是说，截止当前日志，kops依然托管在github的kubernetes账户下，算是kubernetes对AWS的官方支持，但是AWS却在发展自己的产品EKS和fargate。

从kubernetes的角度出发，他是在开发一款支持其他云平台（公有云，私有云）的一键部署工具。但是其他势力也在暗流涌动，从我接触AWS的SA的经验来看，如果有机会，AWS的SA还是会向客户推荐EKS（因为EKS刚进入中国，但是Fargate在中国区还没落地）。

## 2. 功能

下面是github上写的，我给大家解释一下

- Automates the provisioning of Kubernetes clusters in [AWS](https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md), [OpenStack](https://github.com/kubernetes/kops/blob/master/docs/getting_started/openstack.md) and [GCE](https://github.com/kubernetes/kops/blob/master/docs/getting_started/gce.md)

  自动配置，就是说你跑上命令就可以去喝茶了

- Deploys Highly Available (HA) Kubernetes Masters

  可以配置高可用的kubernetes集群，就是可以配置多master节点的集群

- Built on a state-sync model for **dry-runs** and automatic **idempotency**

  可以进行同步状态的试运行（dry-run），自动的idempotency，这个翻译成中文叫幂等性，这样非常容易混淆，其实我们叫他非线性增长更贴切，也就是说，如果你创建一个节点用1分钟，如果创建30个节点，不是30分钟，而是10分钟，这样的增长就叫idempotency。在kubeadm的中也有这么一段[代码](https://github.com/kubernetes/kubernetes/blob/master/cmd/kubeadm/app/util/apiclient/idempotency.go)是讲这个的，有兴趣的可以深入研究一下。

- Ability to generate [Terraform](https://github.com/kubernetes/kops/blob/master/docs/terraform.md)

  可以生成Terraform的代码，Terraform是一个IaC的工具，我们架构师的课程会涉及到用代码的方式实现基础架构，Terraform就是一个非常好用的工具。如果没有这个经验的话，我们可以理解他为Ansible的play-book

- Supports custom Kubernetes [add-ons](https://github.com/kubernetes/kops/blob/master/docs/operations/addons.md)

  支持自定义的kubernetes插件，比如CoreDNS之类的

- Command line [autocompletion](https://github.com/kubernetes/kops/blob/master/docs/cli/kops_completion.md)

  命令行自动补全，和bash的自动补全是一样的

- YAML Manifest Based API [Configuration](https://github.com/kubernetes/kops/blob/master/docs/manifests_and_customizing_via_api.md)

  配置文件是Yaml格式

- [Templating](https://github.com/kubernetes/kops/blob/master/docs/cluster_template.md) and dry-run modes for creating Manifests

  在创建资源清单的时候，支持模板和测试，这个和前面的dry-run不一样，这个是清单文件的测试，上面那个是云资源的dry-run

- Choose from eight different CNI [Networking](https://github.com/kubernetes/kops/blob/master/docs/networking.md) providers out-of-the-box

  可以选择8个不同的CNI网络插件

- Supports upgrading from [kube-up](https://github.com/kubernetes/kops/blob/master/docs/upgrade_from_kubeup.md)

  可以使用kube-up升级集群

- Capability to add containers, as hooks, and files to nodes via a [cluster manifest](https://github.com/kubernetes/kops/blob/master/docs/cluster_spec.md)

  可以在创建集群的时候，添加自定义的容器，关系（谁先谁后之类的）和文件到节点上

## 3 在AWS上安装集群

### 3.1. 配置AWS CLI和kubectl

首先得有个AWS账户，在本机上安装[AWS CLI](https://amazonaws-china.com/cn/cli/)。

还要安装kubectl命令，点[这里](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux)

### 3.2. 安装Kops

我这边使用的是MacOS，所以使用brew安装最方便。

``` bash
$ brew update && brew install kops
```

Linux看这个

``` bash
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

Windows的[这里下载](https://github.com/kubernetes/kops/releases/latest)，点点点就好了

### 3.3. 配置 AWS Route53

+ 注意：中国区是没有Route53的，虽然马上就要有了。但是，kops已经支持gossip-based cluster，只要集群名称是.k8s.local结尾，那么就会跳过DNS检测。如果想用Route53服务的请参考[官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kops/)

### 3.4. 创建S3存储，存放集群信息

``` bash
$ aws s3 mb s3://clusters.prod.k8s.local
make_bucket: clusters.prod.k8s.local
```

设置环境变量，为我们下面的步骤做准备

``` bash
export KOPS_STATE_STORE=s3://clusters.prod.k8s.local
```

### 3.5. 创建集群配置

这一步并不是实际创建集群，也就是前面特性中的dry-run，后面的update才是创建集群

``` bash
kops create cluster \
     --name=clusters.prod.k8s.local \
     --zones=ap-south-1c \
     --master-count=3 \
     --master-size="t3.large" \
     --node-count=2 \
     --node-size="t3.large"  \
     --networking=calico \
     --ssh-public-key="~/.ssh/id_rsa.pub"
```

列出集群

``` bash
kops get cluster
```

编辑集群

``` bash
kops edit cluster cluster.kubernetes.cloudnative.com
```

### 3.6. 在AWS上创建集群

``` bash
kops update cluster clusters.prod.k8s.local --yes
.
.
.
Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.clusters.prod.k8s.local
 * the admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons at: https://github.com/kubernetes/kops/blob/master/docs/operations/addons.md.
```

查看下instance

``` bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIP:PublicIpAddress,Name:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table
---------------------------------------------------------------------------------------
|                                  DescribeInstances                                  |
+-------------------------------------------------------+-----------------+-----------+
|                         Name                          |    PublicIP     |  Status   |
+-------------------------------------------------------+-----------------+-----------+
|  master-ap-south-1c-2.masters.clusters.prod.k8s.local |  13.235.54.117  |  running  |
|  master-ap-south-1c-3.masters.clusters.prod.k8s.local |  13.234.10.56   |  running  |
|  master-ap-south-1c-1.masters.clusters.prod.k8s.local |  13.235.53.61   |  running  |
|  None                                                 |  52.66.233.129  |  running  |
|  nodes.clusters.prod.k8s.local                        |  13.235.53.80   |  running  |
|  nodes.clusters.prod.k8s.local                        |  13.235.214.165 |  running  |
+-------------------------------------------------------+-----------------+-----------+
```

登录到master上看下状态

``` bash
admin@ip-172-20-37-171:~$ sudo -i
root@ip-172-20-37-171:~# kubectl get pods -A
NAMESPACE     NAME                                                                   READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-8b55685cc-5w2rh                                1/1     Running   0          10m
kube-system   calico-node-47pk5                                                      1/1     Running   0          9m3s
kube-system   calico-node-fvxgn                                                      1/1     Running   0          8m51s
kube-system   calico-node-pbp42                                                      1/1     Running   0          8m58s
kube-system   calico-node-vmkbc                                                      1/1     Running   0          9m54s
kube-system   calico-node-w88kr                                                      1/1     Running   0          10m
kube-system   dns-controller-5769c5f8b6-v5bpt                                        1/1     Running   0          10m
kube-system   etcd-manager-events-ip-172-20-37-171.ap-south-1.compute.internal       1/1     Running   0          8m58s
kube-system   etcd-manager-events-ip-172-20-53-146.ap-south-1.compute.internal       1/1     Running   0          7m59s
kube-system   etcd-manager-events-ip-172-20-55-51.ap-south-1.compute.internal        1/1     Running   0          9m18s
kube-system   etcd-manager-main-ip-172-20-37-171.ap-south-1.compute.internal         1/1     Running   0          9m10s
kube-system   etcd-manager-main-ip-172-20-53-146.ap-south-1.compute.internal         1/1     Running   0          8m57s
kube-system   etcd-manager-main-ip-172-20-55-51.ap-south-1.compute.internal          1/1     Running   0          9m21s
kube-system   kops-controller-lffj8                                                  1/1     Running   0          8m43s
kube-system   kops-controller-mq4x7                                                  1/1     Running   0          9m1s
kube-system   kops-controller-nv2g5                                                  1/1     Running   0          9m13s
kube-system   kube-apiserver-ip-172-20-37-171.ap-south-1.compute.internal            1/1     Running   3          8m43s
kube-system   kube-apiserver-ip-172-20-53-146.ap-south-1.compute.internal            1/1     Running   4          7m42s
kube-system   kube-apiserver-ip-172-20-55-51.ap-south-1.compute.internal             1/1     Running   2          9m38s
kube-system   kube-controller-manager-ip-172-20-37-171.ap-south-1.compute.internal   1/1     Running   0          9m7s
kube-system   kube-controller-manager-ip-172-20-53-146.ap-south-1.compute.internal   1/1     Running   0          8m18s
kube-system   kube-controller-manager-ip-172-20-55-51.ap-south-1.compute.internal    1/1     Running   0          9m39s
kube-system   kube-dns-autoscaler-594dcb44b5-6vjwx                                   1/1     Running   0          10m
kube-system   kube-dns-b84c667f4-4jmhx                                               3/3     Running   0          8m24s
kube-system   kube-dns-b84c667f4-jfp7h                                               3/3     Running   0          10m
kube-system   kube-proxy-ip-172-20-35-53.ap-south-1.compute.internal                 1/1     Running   0          8m48s
kube-system   kube-proxy-ip-172-20-37-171.ap-south-1.compute.internal                1/1     Running   0          9m22s
kube-system   kube-proxy-ip-172-20-45-175.ap-south-1.compute.internal                1/1     Running   0          8m10s
kube-system   kube-proxy-ip-172-20-53-146.ap-south-1.compute.internal                1/1     Running   0          7m53s
kube-system   kube-proxy-ip-172-20-55-51.ap-south-1.compute.internal                 1/1     Running   0          8m48s
kube-system   kube-scheduler-ip-172-20-37-171.ap-south-1.compute.internal            1/1     Running   0          9m
kube-system   kube-scheduler-ip-172-20-53-146.ap-south-1.compute.internal            1/1     Running   0          8m34s
kube-system   kube-scheduler-ip-172-20-55-51.ap-south-1.compute.internal             1/1     Running   0          8m44s
```



+ 注意1：集群所使用的AMI，也就是镜像，debian-jessie在中国区并没有，想用的话就要从国际区拉一个过来，我们可以使用CoreOS来代替

+ 注意2：国内区创建集群的时候，会去找grc.io的镜像仓库，这个非常慢，我们有两种思路，一个是做个假的docker仓库，把镜像拉下来，在push上去，然后在dns里面写一个假的解析，吧gcr.io指向伪造的镜像仓库。第二种就是自定义镜像仓库，我们用到的镜像在这里

  ``` bash
  gcr.io/google_containers/etcd
  gcr.io/google_containers/pause-amd64
  gcr.io/google_containers/cluster-proportional-autoscaler-amd64
  gcr.io/google_containers/k8s-dns-kube-dns-amd64
  gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64
  gcr.io/google_containers/k8s-dns-sidecar-amd64
  gcr.io/google_containers/kubedns-amd64
  gcr.io/google_containers/k8s-dns-dnsmasq-amd64
  gcr.io/google_containers/dnsmasq-metrics-amd64
  gcr.io/google_containers/exechealthz-amd64
  ```

  

### 3.7. 删除集群

``` bash
kops delete cluster cluster.kubernetes.cloudnative.com --yes
```

