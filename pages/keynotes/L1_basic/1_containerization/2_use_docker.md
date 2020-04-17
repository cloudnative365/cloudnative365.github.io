---
title: 容器基础用法
keywords: keynotes, basic, containerization, use_docker
permalink: keynotes_L1_basic_1_containerization_2_use_docker.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/2_use_docker
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. Docker

### 1.1.  Docker基础架构

+ docker daemon
  + docker守护进程dockerd监听docker API的请求，并且管理docker的对象，比如镜像，容器，网络和卷组
+ docker client
  + docker客户端docker是大部分用户用来和docker交互的工具。
  + docker命令是请求的docker的API
+ docker registries
  + docker仓库是用来储存docker镜像的
  + docker hub和docker cloud是公有仓库，docker默认使用docker hub作为镜像仓库
  + 我们还可以使用我们的私有仓库



![file](/pages/keynotes/L1_basic/1_containerization/pics/2_use_docker/222e7bccfacb32658115301585389619.png)



Docker是C/S架构的软件，实际上，不管是Server端还是Client端，都是由`docker`这一个命令来实现的，他其中有很多子程序。其中有一个子命令叫daemon，他就表示运行为守护进程，也就是docker的服务器程序，他是监听在某个套接字（socket）只上。为了保证安全，他只提供本地的Unix Sock文件套接字。

由containers和images两个部分组成，他们是docker服务端两个非常重要的组成部分。而镜像部分，就来自于registry，我们通常叫他为镜像仓库，默认是dockerhub。而本地开始是没有任何镜像的，需要去仓库下载，下载到本地之后，可以共享。启动容器的时候，就是基于镜像，在镜像的基础之上，为一个容器创建一个专用的可写仓库，从而启动容器。所以说，docker镜像也需要在本地存储。但是我们在本地运行哪些容器，事先是没办法评估的。因此这有个专门的仓库，但是在仓库中有几十万个镜像，实在是太庞大了，我们只能用到哪个就下载哪个。而这个仓库是使用http或者https（默认）协议，如果想要使用http，必须明确定义他是不安全的，使用insecure-registries。同样的，client端和server端也使用http或者https协议。

但是有个问题，镜像服务器在国外，我们如果想要下载镜像到本地，速度是非常慢的。而为了提高访问速度，docker在中国有一个镜像服务器，叫dockercn，但是他们的镜像速度也不是非常好，我们建议使用阿里云的镜像，那个加速是非常快的。其实，163和中科大都有镜像加速，大家可以自行的百度。但是，如果使用dockercn的镜像，仓库是公开的，而使用阿里云的话，一般是私有的，专用的，需要你拥有阿里云的账号。

 

### 1.2. Docker registry
启动容器时，docker daemon会试图从本地获取相关的镜像；本地镜像不存在时，将从registry中下载该镜像并保存到本地；镜像应当是一个无状态的，可扩展的服务器端应用，可以用来存储并且分发docker镜像。

![image-20200330143239277](/pages/keynotes/L1_basic/1_containerization/pics/2_use_docker/image-20200330143239277.png)



首先我们有一个Docker的server端，也就是docker host，上面有一个Docker daemon。接受客户端请求是http或者https协议交互，可以是本地客户端，也可以是其他客户端，Docker在接受请求之后就会在本地启动一个或者多个容器，而容器的启动是需要镜像的，如果本地没有，docker daemon会自动连接到docker的registry(s)，一般来说是docker hub，如果需要从其他地址上下载，需要在下载路径中指明地址。先存储在本地一个能够专门存储这种镜像的存储空间，这个存储空间需要是一个专用的文件系统。比如：1.18使用的是overlay2。镜像是只读的，镜像在镜像仓库的名字一般就是程序名字，仓库内放多个镜像，仓库内存放的一般是镜像的不同版本。



registry用户保存docker镜像，包括镜像的层次结构和元数据，用户可以自建Tegistry，也可以使用官方的Docker hub。registry的分类如下

+ Sponsor Registry：第三方的registry，供客户和Docker社区使用
+ Mirror Registry：第三方的registry，只让客户使用，docker-cn
+ Vendor Registry：由发布Docker镜像的供应商提供的registry，红帽的
+ Private Registry：通过舍友防火墙和额外的安全层的私有实体提供的registry



而registry由两部分组成，Repository和index

+ Repository：由特定的docker镜像的所有迭代版本组成的镜像仓库，一个Registry中可以存在多个Repository，repository可分为“顶级仓库”和“用户仓库”，用户仓库名称格式为“用户名/仓库名”，每个仓库可以包含多个Tag（标签），每个标签对应一个镜像
+ Index：维护用户账户、镜像的校验以及公共明明空间的信息，相当于为Registry提供了一个完成用户认证等功能的检索接口。



Docker registry中的镜像通常由开发人员制作，而后推送至“公共”或“私有”registry上保存，供其他人员使用



### 1.3. registry和repository的区别

一个registry上拥有两重功能

+ 提供镜像存储的仓库，当然仓库中所有镜像的索引
+ 还提供用户访问镜像时候的认证功能。当然，这个额外功能他是一个应用程序。



一个docker registry上，一个docker镜像的仓库，他有仓库的名称，叫repository。一般来说，一个仓库只用来存放一个应用程序的镜像。就是说，某一个应用程序的不同版本会放在一个仓库。而一个仓库的名字，就是应用程序的名字。那么，怎么去标识每一个镜像呢，我们会给每一个镜像添加一个tag，叫标签。

比如：

+ `nginx:1.14`，这样就能唯一标识这一个镜像了。如果给了镜像名，但是没给标签，就默认使用最新的镜像`latest` 。一个镜像可以有多个标签，比如：`nginx: 1.15`，`nginx:latest`，如果升级到了1.16，那么latest就是1.16版本。当然，有的程序还会添加另外的标签叫`stable`，就是指稳定版，所以默认的版本会有三个标签`latest`，`stable`，`1.xxx`。
+ 还有一种是操作系统，比如centos，centos的版本会有6，7，8，如果你使用centos6，最新版本就是6.9，如果使用7，最新版就是7.6。

这些都是根据自己的程序自己的规则来添加标签的。

### 1.4. Docker的对象

创建docker的时候，我们可以创建并且使用镜像，容器，网络，存储卷，插件，或者其他的对象。任何的对象都可以通过http请求来增删改查。

+ 镜像
  + 镜像是只读的模板，包含创建docker容器的说明
  + 通常来说，镜像都是以另外一个镜像为基础的，然后在上面添加了一些自定义的信息
  + 我们可以使用自己镜像，或者可以使用其他人发布在仓库中的镜像
+ 容器
  + 容器就是镜像运行起来的实例
  + 我们可以使用docker的API或者命令创建，运行，停止，移动或者删除容器
  + 我们可以把容器连接到一个或者多个网络中去，可以添加存储，甚至基于他现在的状态创建一个新的镜像



## 2. Docker的安装

### 2.1. 依赖的基础环境
+ 64 bit CPU
+ Linux Kernel 3.10+
+ Linux Kernel cgroups and namespaces

### 2.2. 不同操作系统上安装需要注意的地方
+ CentOS7 “Extras” repository（rhel6系列也支持，但是都是红帽自己打进去的补丁，不建议使用）
+ Unbutu 16之后的版本都带docker，可以直接使用
+ 如果操作系统是在公有云上创建的，一般都会提供epel源，比如aws，可以使用`amazon-linux-extras install epel`
+ 官网提供了一键安装脚本
``` bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
```
+ 我们也可以选择在Mac或者Windows上通过下载安装包的方式在本地安装，参考[这个](https://www.docker.com/products/docker-desktop)

注意：不要使用国内的所有引擎去搜索关键字`docker 安装`之类的，出现的一定是那些自媒体作者放出的大量的半真半假的信息。由于docker版本更新较快，所以安装的时候建议参考[官方文档](https://docs.docker.com/install/linux/docker-ce/centos/)

### 2.3. 安装
一键安装的方法我就不说了，毕竟太简单了。需要注意的是，如果使用的是docker提供的源，那么安装docker的时候使用
``` bash
yum install docker-ce
```
如果是ubuntu上默认的源，安装的时候使用
``` bash
apt-get update
apt install docker.io
```
如果是centos的源，安装的时候使用
``` bash
yum install docker
```

### 2.4. 配置
每个环境的配置文件不一样，我们以centos举例，但是基本来说不会相差太多
+ 环境配置文件 /etc/docker/daemon.json
这个目录开始是不存在的，需要我们手动创建，或者启动docker守护进程，他会自动创建，我们在这个里面主要是配置镜像加速
``` bash
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
# "registry-mirrors": ["https://gvfjy25r.mirror.aliyuncs.com"]
	"registry-mirrors": ["https://registry.docker-cn.com"]
}
EOF
```
+ 完成之后重新加载
``` bash
systemctl daemon-reload
systemctl restart docker
```

## 3. Docker命令
### 3.1. 信息相关
+ Docker安装完成之后，可以使用docker version查看一下信息

![file](/pages/keynotes/L1_basic/1_containerization/pics/2_use_docker/222f43c49025ed20fde6d01585453007.png)

+ 更详细的信息我们使用docker info

![file](/pages/keynotes/L1_basic/1_containerization/pics/2_use_docker/2228adb3d8a094c4d2b4401585453167.png)

+ 命令上，我们发现有两类，一类是`Management Commands:` ，这些就是分组归类之后的命令，是比较新的，而下面还有一类，叫`Commands:` 这些是原来使用的命令。

![file](/pages/keynotes/L1_basic/1_containerization/pics/2_use_docker/222d55f002115d2ff8d9301585452905.png)

### 3.2. 镜像相关
+ docker search：在Docker Hub搜索镜像
+ docker pull：从仓库拉取镜像，可以指定仓库
+ docker images：列出所有本地镜像
+ docker rmi：删除镜像
+ docker image ls：列出已经有的镜像

一般来说，镜像是基于ubuntu或者其他完整操作系统镜像来构建的，其实我们还有一种非常小的镜像，叫做alpine，我们可以使用这个。其实alpine是一个专门用来构建镜像的非常微小的发行版，他能够给我们的应用程序提供运行环境，但是体积非常的小。体积非常小就有另外一个问题了，他没有调试工具，所以生产系统上不建议使用alpine的镜像。我们为了演示就方便以后就使用最小镜像
``` bash
docker image pull nginx:alpine
```

### 3.3. 容器相关
+ docker create：创建一个新的容器
+ docker start：开始一个或者多个停止的容器
+ docker run：在一个新的容器中运行一个命令，如果容器不存在，就创建一个
+ docker pause：暂停容器
+ docker attach：进入到一个正在运行的容器内
+ docker ps（docker container ls）：列出容器，（运行的和不在运行的）
+ docker logs：取出指定容器的日志
+ docker restart：重启一个容器
+ docker stop：停止一个或者多个容器
+ docker kill：干掉一个或者多个容器
+ docker rm：删除一个或者多个容器
+ docker inspect：查看容器的元数据
+ docker exec

### 3.4. docker run
咱们详细说说docker run命令
``` bash
Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```
IMAGE是必须的，COMMAND ARG是可以被传递的，不传递，就使用默认的。OPTION很多，我们说几个常用的
+ -t：分配一个终端
+ -i：交互式访问
+ --name：可以给容器起一个名字
+ --network：指定网络，不指定默认是bridge，也就是docker0
+ --rm：容器一停就删除
+ -d：运行为守护进程

实际上我们可以参考这个图把所有命令都看一下

![img](/pages/keynotes/L1_basic/1_containerization/pics/2_use_docker/1191498-5af2004ca2d1becb.PNG)

``` bash
docker run --name b1 -it busybox:latest
```
这样就进入busybox了，可以看一下/bin下面的命令。如果我们推出了bash，整个容器就直接停了。而神奇的是里面还有httpd程序
``` bash
mkdir -p /data/html
echo "this is busybox" > /data/html/index.html
httpd -f -h /data/html
```
我们就可以通过本机访问了
``` bash
curl 172.17.0.2
```