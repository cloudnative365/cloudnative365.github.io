---
title: 使用kubeadm安装etcd
keywords: keynotes, senior, etcd, install_etcd_with_kubeadm
permalink: keynotes_L3_senior_1_etcd_3_install_etcd_with_kubeadm.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/3_install_etcd_with_kubeadm
typora-root-url: ../../../../../cloudnative365.github.io

---

## 课程目标

- 使用kubeadm安装etcd集群
- 使用kubeadm创建etcd的证书

## 1. 使用kubeadm安装etcd集群

### 1.1. 环境

+ 三个可以通过 2379 和 2380 端口相互通信的主机。本文档使用这些作为默认端口。不过，它们可以通过 kubeadm 的配置文件进行自定义。
+ 一些可以用来在主机间复制文件的基础设施。例如 `ssh` 和 `scp` 就可以满足需求。

### 1.2. 安装所需的软件

+ docker

  + 私有云[点这里](https://developer.aliyun.com/mirror/docker-ce?spm=a2c6h.13651102.0.0.3e221b11OiZivH)

  + AWS

    ``` bash
    yum -y install docker
    ```

  注意：一定要把docker的cgroups的方式和kubelet的cgroup方式修改成一致的，否则会报错

  ``` bash
  # Setup daemon.
  cat > /etc/docker/daemon.json <<EOF
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "registry-mirrors": ["https://gvfjy25r.mirror.aliyuncs.com"]
  }
  EOF
  
  mkdir -p /etc/systemd/system/docker.service.d
  
  # Restart docker.
  systemctl daemon-reload
  systemctl restart docker
  ```

  

+ kubeadm，kubelet

  + 正常方式[点这里](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
  + 国内环境[点这里](https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.3e221b11OiZivH)

### 1.3. 建立集群

+ 将 kubelet 配置为 etcd 的服务管理器。由于 etcd 是首先创建的，因此必须通过创建具有更高优先级的新文件来覆盖 kubeadm 提供的 kubelet 单元文件。

  注意：不同的操作系统的systemd是略有区别的，主要配置文件的位置，可以使用

  ``` bash
  cat << EOF > /usr/lib/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
  [Service]
  ExecStart=
  #  Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
  ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
  Restart=always
  EOF
  systemctl daemon-reload
  systemctl restart kubelet
  ```

  注意：不同的操作系统的systemd是略有区别的，主要配置文件的位置，可以使用`systemctl status kubelet`来查看systemd文件的位置

+ 为 kubeadm 创建配置文件，使用以下脚本为每个将要运行 etcd 成员的主机生成一个 kubeadm 配置文件

  ``` bash
  # 使用 IP 或可解析的主机名替换 HOST0、HOST1 和 HOST2
  export HOST0=172.31.33.72
  export HOST1=172.31.32.70
  export HOST2=172.31.43.7
  
  # 创建临时目录来存储将被分发到其它主机上的文件
  $ mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/
  
  ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
  NAMES=("infra0" "infra1" "infra2")
  
  for i in "${!ETCDHOSTS[@]}"; do
  HOST=${ETCDHOSTS[$i]}
  NAME=${NAMES[$i]}
  cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
  apiVersion: "kubeadm.k8s.io/v1beta2"
  kind: ClusterConfiguration
  etcd:
      local:
          serverCertSANs:
          - "${HOST}"
          peerCertSANs:
          - "${HOST}"
          extraArgs:
              initial-cluster: infra0=https://${ETCDHOSTS[0]}:2380,infra1=https://${ETCDHOSTS[1]}:2380,infra2=https://${ETCDHOSTS[2]}:2380
              initial-cluster-state: new
              name: ${NAME}
              listen-peer-urls: https://${HOST}:2380
              listen-client-urls: https://${HOST}:2379
              advertise-client-urls: https://${HOST}:2379
              initial-advertise-peer-urls: https://${HOST}:2380
  EOF
  done
  ```

+ 生成证书颁发机构CA

  + 如果已经拥有 CA，那么唯一的操作是复制 CA 的 `crt` 和 `key` 文件到 `/etc/kubernetes/pki/etcd/ca.crt` 和 `/etc/kubernetes/pki/etcd/ca.key`。复制完这些文件后继续下一步，“为每个成员创建证书”。

  + 如果还没有 CA，则在 `$HOST0`（您为 kubeadm 生成配置文件的位置）上运行此命令。

    ``` bash
    kubeadm init phase certs etcd-ca
    ```

    创建了如下两个文件

    - `/etc/kubernetes/pki/etcd/ca.crt`
    - `/etc/kubernetes/pki/etcd/ca.key`

+ 为每个成员创建证书

  ``` bash
  kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
  kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
  kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
  kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
  cp -R /etc/kubernetes/pki /tmp/${HOST2}/
  # 清理不可重复使用的证书
  find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete
  
  kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
  kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
  kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
  kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
  cp -R /etc/kubernetes/pki /tmp/${HOST1}/
  find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete
  
  kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
  kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
  kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
  kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
  # 不需要移动 certs 因为它们是给 HOST0 使用的
  
  # 清理不应从此主机复制的证书
  find /tmp/${HOST2} -name ca.key -type f -delete
  find /tmp/${HOST1} -name ca.key -type f -delete
  ```

+ 复制证书和 kubeadm 配置，证书已生成，现在必须将它们移动到对应的主机。

  ``` bash
  USER=ec2-user
  HOST=${HOST1}
  scp -r /tmp/${HOST}/* ${USER}@${HOST}:
  ssh ${USER}@${HOST}
  USER@HOST # sudo -Es
  root@HOST # chown -R root:root pki
  root@HOST # mv pki /etc/kubernetes/
  ```

+ 确保已经所有预期的文件都存在

  `$HOST0` 所需文件的完整列表如下：

  ``` bash
  /tmp/${HOST0}
  └── kubeadmcfg.yaml
  ---
    /etc/kubernetes/pki
  ├── apiserver-etcd-client.crt
  ├── apiserver-etcd-client.key
  └── etcd
      ├── ca.crt
      ├── ca.key
      ├── healthcheck-client.crt
      ├── healthcheck-client.key
      ├── peer.crt
      ├── peer.key
      ├── server.crt
      └── server.key
  ```

  在 `$HOST1`:

  ``` bash
  $HOME
  └── kubeadmcfg.yaml
  ---
  /etc/kubernetes/pki
  ├── apiserver-etcd-client.crt
  ├── apiserver-etcd-client.key
  └── etcd
      ├── ca.crt
      ├── healthcheck-client.crt
      ├── healthcheck-client.key
      ├── peer.crt
      ├── peer.key
      ├── server.crt
      └── server.key
  ```

  在 `$HOST2`

  ``` bash
  $HOME
  └── kubeadmcfg.yaml
  ---
  /etc/kubernetes/pki
  ├── apiserver-etcd-client.crt
  ├── apiserver-etcd-client.key
  └── etcd
      ├── ca.crt
      ├── healthcheck-client.crt
      ├── healthcheck-client.key
      ├── peer.crt
      ├── peer.key
      ├── server.crt
      └── server.key
  ```

+ 创建静态 Pod 清单，既然证书和配置已经就绪，是时候去创建清单了。在每台主机上运行 `kubeadm` 命令来生成 etcd 使用的静态清单。

  ``` bash
  root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
  root@HOST1 $ kubeadm init phase etcd local --config=/home/ec2-user/kubeadmcfg.yaml
  root@HOST2 $ kubeadm init phase etcd local --config=/home/ec2-user/kubeadmcfg.yaml
  ```

+ 注意：修改`/etc/kubernetes/manifests/etcd.yaml`镜像地址

  ``` bash
    image: registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
  ```

+ 注意：手动拉取pause镜像并修改tag

  ``` bash
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
  ```

+ 重新启动kubelet

  ``` bash
  systemctl restart kubelet
  ```

+ 检查群集运行状况

  ``` bash
  ETCD_TAG=3.4.3-0
  docker run --rm -it \
  --net host \
  -v /etc/kubernetes:/etc/kubernetes registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:${ETCD_TAG} etcdctl \
  --cert /etc/kubernetes/pki/etcd/peer.crt \
  --key /etc/kubernetes/pki/etcd/peer.key \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --endpoints https://${HOST0}:2379 endpoint health --cluster
  ...
  https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
  https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
  https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
  ```

  + 将 `${ETCD_TAG}` 设置为你的 etcd 镜像的版本标签，例如 `3.4.3-0`。要查看 kubeadm 使用的 etcd 镜像和标签，请执行 `kubeadm config images list --kubernetes-version ${K8S_VERSION}`，其中 `${K8S_VERSION}` 是 `v1.17.0` 作为例子。

  - 将 `${HOST0}` 设置为要测试的主机的 IP 地址

  