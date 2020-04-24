---
title: Docker网络管理
keywords: keynotes, basic, containerization, docker_network
permalink: keynotes_L1_basic_1_containerization_4_docker_network.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/4_docker_network
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. 简介
一般来说，一个网卡只能分配给一个名称空间。但是如果名称空间数量超过了实体网卡的数量，我们就需要模拟网卡。而linux内核中支持两种级别的模拟网卡，一种是二层设备，一种是三层设备。我们的物理网卡本来也就是二层设备，他就是一个工作在链路层，能封装物理报文的，实现在各设备网络之间实现报文转发的设备。而这个功能完全能够通过在Linux内核的功能，在二层之上虚拟设备的支持，创建虚拟网卡接口的，而且这种虚拟网卡很独特，他是成对儿出现的，可以模拟为一根网线的两头。一头插在一个主机之上，一头插在交换机之上。那就相当于让一个主机插到了交换机之上，而linux内核原生支持二层虚拟网桥设备，就是用软件来构建一个交换机，我们可以brctl来实现。那么，一个软件交换机，一个软件实现的虚拟机，也就是名称空间。内核自己创建一个网卡，一头分配给这个名称空间，一头分配给交换机，就相当于模拟了一个主机连接到交换机。同样的，如果你有两个名称空间，这两个名称空间都这么干。各自创建一个虚拟网卡，一个创建在交换机上，一个创建在名称空间上，实现了接入的机制。那么我们就能实现了网络连接功能，他们就好像连接到一个交换机上的两个主机。显然，如果他们两台机器配置的网络地址在同一个网段，就可以直接通信了。这就是所谓的虚拟化网络，从网络设备通信的物理设备到网卡都是用纯软件的方式实现，我们在一台主机上通过纯软件的方式来实现，所以我们把他叫做网络虚拟化技术当中的一种简单的实现。

有一个著名的应用程序，叫ovs（open vswitch），用纯软件的方式实现交换机，他还能模拟非常高级的三层网络设备才有的功能，比如：vlan，vxlan技术，gre技术，甚至是流控技术，就是SDN。完全用软件实现，功能非常强大。只不过他不属于linux内核模块本身的功能，我们需要额外安装这个软件。这个软件是由CISCO等众多专业的网络设备生产公司联合研发的软件，而目前在云计算的大潮之下，我们要构建一个云计算中心时，构建网络是非常复杂的工作，因为网络之上承载着主机，他们需要通信。而这个网络虚拟化实现的功能，需要软硬件结合起来，把传统意义上的网络平面，控制平面，传输平面剥离开来，实现将控制平面集中到一个专业的设备之上实现全局调度。也就是实现了SDN的机制。所以我们以后如果构建云计算中心的时候，不仅需要软件，还需要在硬件层面支持，在每一个主机之上构建出非常复杂的网络环境来，毕竟，在同一个网络之上，我们要运行多个主机或者多个容器，这每一个容器都需要用到网络。

## 2. 容器间通信

如果是在同一个物理机的两个容器，两个名称空间想通信，就在这同一个主机之上建立一个虚拟交换机，我们让两个容器或者名称空间各自用纯软件的方式建一对儿虚拟网卡，那么一半在容器上，一半在虚拟交换机上。如果我们有多个交换机怎么办，我们有两个软件交换机，交换机S1上的C1要和交换机上的C3通信怎么办

![file](https://graph.baidu.com/resource/222842583a8fcc4671c1001585582593.png)

刚才说过，我们可以在主机之上再做一对儿网卡，一个在S1上一个在S2上。但是，如果我们希望他们之间通过路由转发的话，我们就需要做一个路由。

![file](https://graph.baidu.com/resource/2226f88064c41a28c07d001585582769.png)

其实linux自己就可以当一个路由来使用，使用iptables规则，或者直接打开转发就可以了。路由器是三层设备，linux在内核级别，使用一个单独的名称空间就能支持。我们可以在做一个容器，容器里面只有路由一个功能。但是需要模拟出网卡，让他们建立关联关系。假如C1和C5想通信怎么办。

我们可以使用桥接，就是把物理网卡当做交换机来用，所有请求都到达物理网卡，然后根据桥接上来的机器的MAC地址，来判断请求应该转发给哪个主机。也就是说桥接上的机器的网卡必须有自己的MAC地址，而且不能和主机一样。这种通信代价很大，首先，你的所有容器都是桥接的，如果大家都在同一个网络平面，那么就很容易产生风暴，因此，在隔离上也是极其不容易的。在大规模的虚拟机或者虚拟机的场景中基本不可行，除非我们可以使用大二层的技术把他们隔离开来。

![file](https://graph.baidu.com/resource/222b60ecf8e945f6028c801585582895.png)

如果不桥接，而且能够实现跨主机通信，我们应该使用的是NAT技术。比如，C3想和C6通信，C3的网络地址和物理网卡的地址不在同一个网段，那么C3把网关指向S2，我们可以把S2当做网关来使用，给S2配置一个IP地址，跟C3在一个网段。然后在物理机上打开核心转发功能，所以C3跟C6通信时，先把请求送给S2，到达物理内核，物理机判定不是自己的地址，然后查路由经由物理网卡把请求送出去。但是报文回不来，因为C3是私有地址，我们最好在C3报文送走到S1之前要把源IP改成，H1主机的物理网卡的IP地址，这样C6回包的时候，就把请求回给物理机就好了，然后物理机拿到回报之后，发现不是自己的，查找网卡查路由表之后发现是C3访问的，然后把回包送给C3。就是SNAT。但是这里有个问题在于，H2的通信也是需要经由NAT来实现，就是两级NAT，C6也是私有地址，C6怎么能被C3看见呢？想要暴露自己，就必须要DNAT。在主机上的某一个端口提供服务，然后由这个端口映射给C6。也就是说，想要访问C6就必须访问H2的物理网卡地址，请求发给C6实际上是H2做的DNAT，把请求发送过去的。也就是说，请求出去要做SNAT，到达目标要做DNAT。C4和C6实际上是隔着两层来通信的，效率非常的低。 

![file](https://graph.baidu.com/resource/222c27a8b4a07f9a1322301585582967.png)

而解决这种问题需要用到叠加网络，也是我们后面kubernetes中讲到的技术，overlay network。

简单来说，我们有物理机上面运行了虚拟机，物理机的物理网卡，同时在虚拟机上做一个虚拟的桥，让虚拟机的网卡都连接到这个桥上来，而接上来的网卡通过物理网卡的功能实现隧道转发，从而实现C1直接看到C5。那么，本来他们可以通信，而C1和物理网络不在同一地址段内，而和C5在同一地址段内，C1想要访问C5，但是物理网卡知道C5并不在本地服务器上，使得请求从C1出来之后，到达C5之前是这么转发报文的，要做隧道转发，就是C1的报文的源地址是C1，目标地址是C5，然后在报文上在封装一层，源地址是C1网卡的地址，目标地址是C5网卡的地址，所以C5的网卡拿到报文拆包之后发现，目标地址是C5，就直接把报文送给本地交换机，由软网桥转发。本来他应该是三层报文，但是通过叠加的方式，本来应该封装二层了，但是没有封装二层，而是重新封装一个三层四层报文，实现两级三层封装。 

![file](https://graph.baidu.com/resource/222831d29c4b7f1b2e80501585625050.png)

## 3. Docker的网络
Docker安装完成后提供三种网络，bridge，host和none，默认是bridge，但是这个bridge不是物理桥，是NAT桥。他会创建一个纯粹的软交换机叫docker0，也可以当网卡使用，不给地址就是交换机，给地址既能当交换机，又能当网卡。
``` bash
yum -y install bridge-utils

brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242800e4768	no

ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:58:a7:45:b5:b6 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:80:0e:47:68 brd ff:ff:ff:ff:ff:ff
```

随后我们创建容器的时候，会创建一对儿网卡，一个放在虚拟机上，一个放在交换机上。我们看到的veth就是一对网卡的一半。我们可以使用brctl或者ip命令
``` bash
docker run --name t1 -it -d busybox

ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:58:a7:45:b5:b6 brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:80:0e:47:68 brd ff:ff:ff:ff:ff:ff
5: veth893bb3f@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether da:c2:8f:62:da:2e brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

veth893bb3f@if4就是新创建的网卡，我们登陆到容器中，就可以看到另外一个网卡，使用ifconfig，而且可以ping通docker0。docker0是一个nat桥，每启动一个桥，他会自动生成一个iptables规则。可以使用iptables -t nat -vnL查看

![file](https://graph.baidu.com/resource/222b23725b4026e421d8b01585664881.png)

如果我们的容器内跑的是nginx，而nginx想被其他客户端访问，我们应该有下面几种来源

``` bash
docker run --name n1 -d nginx

docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
667558b5bcf2        nginx               "nginx -g 'daemon of…"   About a minute ago   Up About a minute   80/tcp              n1
9cdad181c82a        busybox             "sh"                     11 minutes ago       Up 11 minutes                           t1

```

+ 如果是同一个主机上的同一个容器，另外一个容器也使用了桥接网络，他们可以直接通信

``` bash
docker inspect 667558b5bcf2 |grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.3",
                    "IPAddress": "172.17.0.3",
```

在busybox上，使用`wget -O - -q http://172.17.0.3`，可以访问到nginx

+ 我们还可以从物理机上访问，使用“curl 172.17.0.2”也可以访问

+ 如果我们想从其他机器访问怎么办呢，我们就需要发布，做DNAT，在物理端口上启动一个端口来提供端口，而端口使用DNAT的方式转发到我们的容器中。也就是说我们创建任何的容器时，都会默认使用桥接的网络，如果想被访问，就需要添加一个DNAT规则，便于他被外部其他客户端访问。那么如果有两个容器，都需要使用80端口，我们就需要使用两个端口，而且一个必须是非80端口，但是这个是无解的。必须启用多个端口

+ 而如果容器需要访问容器，我们就可以通过另一种方式。我们容器内有6个名称空间，他们是互相独立的。

![file](https://graph.baidu.com/resource/222189c7d9fa222b9702e01585627185.png)

为了让他们能从内部访问，我们可以这样设计。把User，Mount和Pid独立，UTS，Net和IPC共享，就形成了下面的形式。

![file](https://graph.baidu.com/resource/22278fc03fe0f8537efd401585627175.png)

+ 那么，物理机上是不是也有一个名称空间。我们是不是也可以使用物理机的名称空间呢？必须可以。我们对于第三种情况，还有一个这种的办法，就是把一个容器暴露出来，而把另外一个使用物理的名称空间。这种就叫host方式。

+ 最后一种方式，就是none，就是没有网卡，没有网络。比如：我们有可能只需要加载一些数据，处理之后就销毁，这样根本不需要网络。

其实就是下面这4中网络模型

![file](https://graph.baidu.com/resource/222a753d504ba2b0f5ffd01585627544.png)

这四种类型，第一个是，只创建一个lo接口，而不用和其他的容器通信，第二个创建两个接口，一半在容器上，一半在docker0桥上。可以通过brctl来实现，第三种是一个容器加入了一个容器，先创建一个容器A，然后再创建容器B去共享A的一部分空间，叫联盟容器。最后一个，容器共享的是主机的名称空间。

+ 最后，我们在创建的时候，需要指定网络类型，就使用
``` bash
docker container run --network XXXX
```
查看网络类型就使用
``` bash
docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
08676b91ca7c        bridge              bridge              local
782b9a1ab54d        host                host                local
86960aa7fb1d        none                null                local
```

查看网桥的具体信息可以使用下面的命令，这些信息我们是可以修改的，我们后面继续讲
``` bash
docker network inspect bridge
```

## 4 手动添加网络名称空间
我们可以使用ip netns来模拟docker管理网络名称空间，当我们使用IP来创建网络名称空间的时候，只有网络名称空间是隔离的，其他都是共享的。
``` bash
ip netns add r1
ip netns add r2
ip netns list
r2
r1
```

如果我们没有单独给他们指定网卡，他们应该只有一个网卡lo
``` bash
ip netns exec r1 ifconfig -a
lo: flags=8<LOOPBACK>  mtu 65536
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

我们可以使用ip命令创建虚拟网卡对，然后人工分配到网络名称空间中
``` bash
ip link add name veth1.1 type veth peer name veth1.2
```

查看网卡，会看到两个网卡
``` bash
ip link show
.
.
.
10: veth1.2@veth1.1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether d2:2b:e6:bf:b5:73 brd ff:ff:ff:ff:ff:ff
11: veth1.1@veth1.2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 9a:24:a4:50:be:21 brd ff:ff:ff:ff:ff:ff
```

但是这两个网卡都没有被激活，ifconfig看不见，我们需要把其中一个挪到创建的网络名称空间当中去
``` bash
ip link set dev veth1.2 netns r1
```
再看，会少一个网卡
``` bash
ip link show
.
.
.
11: veth1.1@if10: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 9a:24:a4:50:be:21 brd ff:ff:ff:ff:ff:ff link-netnsid 2
```

然后到r1中去看
``` bash
ip netns exec r1 ifconfig -a
lo: flags=8<LOOPBACK>  mtu 65536
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth1.2: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether d2:2b:e6:bf:b5:73  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

我们还可以改名
``` bash
ip netns exec r1 ip link set dev veth1.2 name eth0
```

发现名称变了
``` bash
ip netns exec r1 ifconfig -a
eth0: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether d2:2b:e6:bf:b5:73  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=8<LOOPBACK>  mtu 65536
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

然后激活他们就可以通信了，先激活宿主机上的
``` bash
ifconfig veth1.1 10.1.0.1/24 up
```

再把r1中的激活
``` bash
ip netns exec r1 ifconfig eth0 10.1.0.2/24 up
```

查看连接
``` bash
ping 10.1.0.2
```

我们也可以把宿主机上的那个送给r2，就好像r2和r1在通信
``` bash
ip link set dev veth1.1 netns r2
```

在看主机上的网卡，veth1.1就消失了
``` bash
ifconfig
```

跑到了r2上
``` bash
ip netns exec r2 ifconfig -a
lo: flags=8<LOOPBACK>  mtu 65536
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth1.1: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether 9a:24:a4:50:be:21  txqueuelen 1000  (Ethernet)
        RX packets 11  bytes 866 (866.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11  bytes 866 (866.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

默认是没有激活的，我们再手动激活
``` bash
ip netns exec r2 ifconfig veth1.1 10.1.0.3/24 up
```

发现IP地址已经有了
``` bash
ip netns exec r2 ifconfig
veth1.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.1.0.3  netmask 255.255.255.0  broadcast 10.1.0.255
        inet6 fe80::9824:a4ff:fe50:be21  prefixlen 64  scopeid 0x20<link>
        ether 9a:24:a4:50:be:21  txqueuelen 1000  (Ethernet)
        RX packets 13  bytes 1046 (1.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17  bytes 1382 (1.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

我们在r2中ping r1
``` bash
ip netns exec r2 ping 10.1.0.2
```

也是没问题的

## 5 创建容器的时候指定一些网络选项

创建的容器使用bridge网络
``` bash
docker run --name t1 -it --rm busybox:latest
```
或者
``` bash
docker run --name t1 -it --network bridge --rm busybox:latest
```

创建none网络的容器
``` bash
docker run --name t1 -it --network none --rm busybox:latest
```

默认创建的容器，主机名就是容器的ID号，我们可以改，但是我们一般会使用--h选项，在创建的时候就注入主机名
``` bash
docker run --name t1 -it --network bridge -h t1.jormun.com --rm busybox:latest
```

如果我们还需要使用主机名作为dns名称而被访问，那么有两种方式，第一是主机名可以被解析，通过DNS或者本地hosts文件，hosts文件中是没问题的，但是resolve文件会把nameserver指向我们的物理机上使用的解析服务器

使用nslookup -type=A www.baidu.com

如果我们的宿主机没有dns服务器怎么办呢？我们可以在创建容器的时候指定dns服务器，--dns和--dns-search
``` bash
docker run --name t1 -it --network bridge -h t1.jormun.com  --dns 114.114.114.114 --dns-search jormun.com --rm busybox:latest
```

我们还可以修改hosts的记录，通过--add-hosts来外部注入
``` bash
docker run --name t1 -it --network bridge -h t1.jormun.com  --dns 114.114.114.114 --dns-search jormun.com --add-hosts www.jormun.com:1.1.1.1 --rm busybox:latest
```

## 6 开放入栈的通信

我们使用nginx的时候，nginx是隐藏在docker0之后的，一般来说，从外部访问是不可达的。我们需要主动发布，或者暴露到对外通信的网络，叫expose，使用-p选项，他的方式有4种

+ -p containerPort：将指定的容器端口映射至主机所有地址的一个动态端口，也就是一个30000以上的随机端口，假如我们在一个机器上运行多个nginx，使用这种方式不会产生冲突
``` bash
docker run --name web -it --network bridge -p 80 --rm nginx
```

我们可以使用iptables -t nat -vnL生成的规则也可以使用docker port web来查看，我们发现，这个是监听在所有地址上的。

+ -p hostPort:containerPort：将容器端口containerPort映射至指定的主机端口hostPort
``` bash
docker run --name web -it --network bridge -p 80:80 --rm nginx
```

使用docker port web来查看，我们发现，这个是监听在任意地址的80端口上的

+ -p ip::containerPort：将指定的容器端口containerPort映射至主机指定ip的动态端口
``` bash
docker run --name web -it --network bridge -p 宿主机的IP::80 --rm nginx
```

使用docker port web来查看，我们发现，这个是监听在宿主机的IP上的

+ -p ip:hostPort:containerPort：将指定的容器端口containerPort映射至主机指定ip的端口hostPort
``` bash
docker run --name web -it --network bridge -p 宿主机的IP:80:80 --rm nginx
```

使用docker port web来查看，我们发现，这个是监听在宿主机的IP上的80端口的

+ 而我们也可以多次使用-p 选项来暴露多个端口
``` bash
docker run --name web -it --network bridge -p 宿主机的IP:80:80 --rm nginx
```

## 7. 联盟式容器

我们使用交互式启动一个容器
``` bash
docker run --name b1 -it --rm busybox
```

然后启动第二个容器的时候使用 --network选项共享b1的网络名称空间
``` bash
docker run --name b2 --network container:b1 -it --rm busybox
```

但是他们的目录不是共享的，我们可以通过创建一个文件来验证一下，而他们的网络是共享的

在一个运行
``` bash
echo "hello" > /tmp/index.html
httpd -h /tmp/
netstat -tnl
```
在另一个上
``` bash
wget -O - -q 127.0.0.1
```

## 7. host容器

docker run --name b2 --network host -it --rm busybox

我们用ifconfig就会发现自己使用的是主机的网络

此时，如果我们启动一个http服务器，会发现他监听的端口就是本机的端口
``` bash
echo "hello" > /tmp/index.html
httpd -h /tmp/
netstat -tnl
```

## 8. 自定义docker网桥
### 8.1. 修改docker0桥的网络属性信息

需要修改/etc/docker/daemon.json
``` bash
{
“bip”: "192.168.1.5./24",
"fixed-cidr":"10.20.0.0/16",
"fixed-cird-v6":"2001:db8::/64",
"mtu":1500,
"default-gateway":"10.20.1.1",
"default-gateway-v6":"2001:db8:abcd::89",
"dns":["10.20.1.2","10.20.1.3"]
}
```

### 8.2. 修改客户端和服务器通讯的地址
默认是/var/run/docker.sock，我们可以使用-H选项来指定docker去连接哪个docker服务器，而我们的服务器也需要暴露端口，需要修改/etc/docker/daemon.json
``` bash
“hosts”:["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"]
```
或者在dockerd启动的时候传递 -H选项

### 8.3. 创建一个自定的桥
``` bash
docker network create -d bridge --subnet "172.26.0.0/16" --gateway "172.26.0.1" mybr0
f0aa13b02d9874cfb4656a5362094387f5a4c29e74a89a6e27363e4519e34e28

docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
08676b91ca7c        bridge              bridge              local
782b9a1ab54d        host                host                local
f0aa13b02d98        mybr0               bridge              local
86960aa7fb1d        none                null                local
```

ifconfig可以看到一个新的接口
``` bash
ifconfig
br-f0aa13b02d98: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.26.0.1  netmask 255.255.0.0  broadcast 172.26.255.255
        ether 02:42:fd:25:e4:4a  txqueuelen 0  (Ethernet)
        RX packets 7  bytes 1272 (1.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25  bytes 1844 (1.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

我们在创建容器的时候就可以让他加入mybr0
``` bash
docker run --name t3 -it --net mybr0 busybox
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:1A:00:02
          inet addr:172.26.0.2  Bcast:172.26.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1032 (1.0 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```