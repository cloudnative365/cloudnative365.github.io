---
title: Docker file
keywords: keynotes, basic, containerization, docker_image
permalink: keynotes_L1_basic_1_containerization_7_docker_file.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/3_docker_image
typora-root-url: ../../../../../cloudnative365.github.io
---

## 引言
一般来说，我们运行某一个应用程序，比如：nginx，他都不是运行在默认配置下的，我们需要手动定义模块化的配置文件然后启动服务。不同的生产场景用到的参数各不相同，因此我们只能在满足大多数场景，或者是满足最小要求这么一个前提来启动镜像。如果我们从docker hub中拖出来一个nginx镜像来启动容器的时候，他的默认配置不一定会满足我们的需要，我们必须要去修改他的配置。
+ 我们目前使用的方式是使用docker exec进入到容器内部，然后使用vim命令，然后在reload。
+ 另外，还有一种方式，我们可以把他配置文件对应的路径作为存储卷。也就是在容器启动之前，把配置文件编辑好，然后容器启动的时候，把应用程序启动的时候，默认加载配置文件的那个路径与宿主机上的目录建立关联关系，这样也可以加载到我们定制的配置文件。但是我们做的编辑同样不能立即生效，依然需要reload
+ 最后一种方式就是我们今天要说的，自己制作镜像。

## 1. 制作镜像的方式
我们前面说过一种基于容器制作镜像的方式，等容器启动之后，基于交互式连入容器，然后做修改，修改完成以后是保存在最上层的可写层的，我们就可以把可写层保存成为一个新镜像。然后我们再新建镜像的时候，根据我们所创建的镜像来启动。但是这样也有不足的地方，那就是我们制作的镜像也是把配置文件直接写死在了镜像中的，如果我们想要修改还是没办法实现，对于日常的变更非常不友好。而且，这种方式最不方便的地方在于，比如：我们现在有三个环境，开发测试，测试环境，生产环境，他们各不相同，因为他们的规则和服务器配置都不一样，当然，设置也不一样。为此，我们要做至少三个镜像。如果我们现在的nginx是1.12版，后来想升级到1.14版本，三个镜像都需要重新制作。而且，我们的nginx服务器的使用场景也不一样，有的地方做反向代理，有的是图片服务器，有的是静态内容服务器。那么三个环境，三种场景，我们至少要做9个镜像。这样就让我们的运维工作变得繁琐而复杂。显然，这种方式也不是最妥当的。

很多时候，我们需要部署在生产环境的应用程序不是公开的，是自己开发的。在互联网上根本不可能存在，我们就必须要自制镜像，这样就不仅仅是配置文件的问题，很多时候我们都迫切需要一种更友好的方式来解决这个问题。我们来看看docker是怎么解决这个问题的。

我们以nginx为例，假如我们这个nginx镜像启动为容器之后只做一件事，配置一个虚拟主机，配置一个虚拟server。尽管这个主机的主机名，监听的端口和root document的目录是不一样的，但是配置文件的格式是一样的。我们就把配置文件server.conf放在/etc/nginx/conf.d目录下，做成类似模板一样的东西。而刚才说到的内容全部用变量来表示，比如
``` bash
{
	server_name $NGX_SERVER_NAME;
	listener $NGX_IP:$NGINX_PORT;
	root $DOC_ROOT;
}
```

为了让我们的配置更加简单，我们在内部都使用变量替换的方式来实现。当镜像要启动为容器的时候，容器内部的主进程在启动之前需要启动一个别的程序，这个程序根据镜像中的文件，以及用户启动为容器时向容器传递的环境变量都是默认值，而后这个程序就把我们用户传递进来的值替换在配置文件内，然后保存在/etc/nginx/conf.d/server.conf，保存完成之后由这个程序再启动主进程，然后这个进程就退出了。由一个进程启动另外一个进程，并替换当前进程，我们使用exec。我们启动一个子进程，但是子进程，是把当前进程直接替换掉的，而且顶了当前进程的ID号。而我们这个进程就是帮容器预设环境的，完成以后，由主进程把这个进程一覆盖，就完成了。这就是我们通过环境变量能配置服务的原因。这就是说，我们将来基于一个镜像启动多个容器，让多个容器拥有不同配置的做法，其实就是向容器传递变量，而且容器在启动的时候，可以获得变量，并且替换为主进程的配置信息。而默认情况下，我们的nginx是不支持的，我们做镜像的时候，就需要把这个框架，或者模板准备好，或者说这个处理程序要设计好，处理完成后由他启动nginx，就能做到去替换配置信息，从而实现自定义接受参数，以配置变量来传递配置信息的，适用于多种不同环境的镜像。这就是为什么说，对于容器来讲，通过环境变量来配置是至关重要的。而我们的cloudnative的应用程序，他们通常都是设置为类似的方式，他们的配置文件天生就是这种格式的，他们可以直接接受变量，直接替换，这个程序在启动的时候可以做到完全不用读取配置文件而是去加载系统上的环境变量就可以启动程序。但是我们说过，把程序运行在容器当中，从某种意义上说，其实并不是更简单了，而是更复杂了。在容器化时代，我们颠覆了传统的Linux的哲学理念，不是使用一个配置文件来配置程序，而是使用环境变量。

## 2. 基于Dockerfile制作镜像

![img](/pages/keynotes/L1_basic/1_containerization/pics/3_docker_image/874963-20200209160957298-684710888.png)
我们前面讲过镜像的生成方式，一种是基于容器制作，一种是基于Dockerfile制作。Dockerfile是创建Docker镜像的源代码。

+ Docker可以通过读取Dockerfile中的内容，而自动构建镜像
+ Dockerfile是文本形式的，包含了用户在命令行可以调用的，集成在镜像中的所有指令。
+ 使用Docker build，用户可以创建一个自动构建，执行多个在命令行连续运行的指令。

这些指令一共不过20个左右，我们只要了解他们的逻辑和语法格式，就能把他们堆起来形成dockerfile。

## 3. Dockerfile
### 3.1. 格式
+ # Comment（注释）
+ INSTRUCTION arguments（指令+参数）

为了区分指令和参数，我们通常会把指令大写，但是指令本身不是大小写敏感的。dockerfile是顺序执行的，但是dockerfile的第一个非注释行必须是`FROM`，他用来指定我们的镜像是基于哪个基础镜像来构建。所以我们所有的镜像都必须基于某个镜像来做，如果没有，dockerfile这个方式就不行了，我们需要使用其他的方式手动去拼凑。
### 3.2. Dockerfile的工作逻辑
+ 我们在操作系统上工作的时候，都是在某个目录下工作的，我们叫工作目录，也叫当前目录，我们通常把这个目录叫做WORKDIR。
+ 我们做docker镜像的时候，不一定在当前目录，但是必须在某一个目录下进行，要做docker镜像时，需要找一个专门目录，在这个目录放进dockerfile，而且dockerfile文件的文件名的首字母必须大写，就是`Dockerfile`。
+ 如果在docker镜像内打包进很多文件，我们必须把文件做好之后放在工作目录当中，或者在工作目录的子目录。
+ 我们如果把很多文件都包含进去，但是有些文件只是一些说明，在打包的时候不想包含进去，我们可以使用一个隐藏文件叫`.dockerignore`，用这个文件可以写进来一些文件的路径，一行一个，或者使用通配符来匹配多个文件。他表示里面的文件或者路径，在打包的时候都不包含进来。
+ 接下来，我们就可以通过读取dockerfile中的命令来制作docker镜像了，而制作docker镜像的命令叫docker  build
+ 完成以后，打好标签，推送到镜像仓库里面就可以用了

这样，我们就不需要启动容器做修改在保存而形成新的镜像。但是docker build这个命令只是代替我们完成这个过程。我们执行的命令必须是镜像中包含的，可执行的命令，而不是宿主机的命令。

### 3.3. 环境变量替换
这个环境变量不是我们容器启动时候的环境变量，这个是我们做docker build时候可用的环境变量，毕竟做镜像就需要基于镜像先启动容器。
+ 环境变量使用ENV来声明，他在特定的指令中替换变量，然后被dockerfile解释
+ 环境变量在Dockerfile中使用$VAR或者${ VAR}表示，和我们在shell中用的非常相似
+ ${VAR}还是支持一些标准的bash修饰符，比如下面这两种变量替换的特殊格式：
  + ${VAR:-word}表示如果变量存在，就使用这个变量，如果变量不存在，就使用word作为变量的值
  + ${VAR:+word}表示如果变量存在，就设置为word，其他条件下，变量值为空字符串

## 4. Dorkerfile的指令
### 4.1. From
+ FROM指令是最重要的一个且必须为Dockerfile文件开篇的第一个非注释行，用于为镜像文件构建过程指定基准镜像，后续的指令运行于此基准镜像所提供的运行环境
+ 实践中，基准镜像可以是任何可用镜像文件，默认情况下，docker build会在docker主机上查找指定的镜像文件，在其不存在时，则会从Docker Hub Registry上拉取所需的镜像文件，如果找不到指定的镜像文件，docker build会返回一个错误信息
+ 语法：
``` bash
FROM <repository>[:<tag>]
```
或者
``` bash
FROM <repository>@<digest>
```
digest就是哈希码的意思

### 4.2. MAINTAINER（depreacted）
从括号可以看出这个已经被废弃了，目前使用的是LABLE，然后标签中有一个键值是`MAINTAINER：作者信息`。虽然已经废弃了，但是在很多旧的镜像中还是可以看到
+ 用于让Dockerfile制作者提供本人的详细信息
+ Dockerfile并不限制MAINTAINER指令可出现的位置，但推荐将其放置于FROM指令之后
+ 语法：
``` bash
MAINTAINER <author's detail>
```
`<author's detail>`可以是任何文本信息，但一般都使用作者名称及邮件地址
比如
``` bash
MAINTAINER “Jormun <29371962@qq.com>”
```

### 4.3. LABEL
+ LABEL指令是给镜像添加元数据的
+ LABEL是键值对儿，key-value
+ 语法：
``` bash
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```
+ 在LABEL中包含空格的话，需要使用引用或者反斜线
+ 一个镜像可以包含不止一个标签
+ 我们可以在一行中指定多个标签

### 4.4. COPY
+ 用于从Docker主机复制文件至创建的新映像文件，相当于从宿主机当前的工作目录中把一个文件或者一些文件，复制到目标系统的镜像文件当中去
+ 语法
``` bash
COPY <src> ... <dest>
```
或者多个源，到一个目标
``` bash
COPY ["<src>" ,... "<dest>"]
```
+ `<src>`源文件路径，表示要复制的源文件或者目录，一般使用相对路径支持使用通配符，`<dest>`表示目标路径，即正在创建的image的文件系统路径；建议为`<dest>`使用绝对路径，如果要使用相对路径，COPY指定则以WORKDIR为其起始路径；
+ 在路径中有空白字符时，通常使用第二种格式
+ 文件复制的准则
  + `<src>`必须是build上下文中的路径，不能是其父目录中的文件
  + 如果`<src>`是目录，则其内部文件或子目录会被递归复制，不像shell命令的copy，需要使用`-R`选项，但`<src>`目录自身不会被复制
  + 如果制定了多个`<src>`，或在`<src>`中使用了通配符，则`<dest>`必须是一个目录，且必须以`/`结尾
  + 如果`<dest>`事先不存在，它将会被自动创建，这包括其父目录路径

### 4.5. ADD
+ 类似于COPY，ADD支持使用TAR文件和URL路径，如果宿主机在执行build的过程中，能访问到网络上的服务器，就可以基于URL的形式下载，他可以把文件引用下载到本地，并且打包进镜像中
+ 语法
``` bash
ADD <src> ... <dest>
```
或者多个源，到一个目标
``` bash
ADD ["<src>" ,... "<dest>"]
```
+ 操作准则
  + 同COPY指令
  + 如果`<src>`为URL且`<dest>`不以`/`结尾，则`<src>`指定的文件将被下载并直接创建为`<dest>`；如果`<dest>`以`/`结尾，则文件名URL指定的文件将被直接下载并保存为`<dest>/<filename>`
  + 如果`<src>`是一个**本地**系统上的压缩格式的tar文件，它将被展开为一个目录，其行为类似于"tar -x"；然而，通过URL获取到的tar文件将不会自动展开；
  + 如果`<src>`有多个，或其间接或直接使用了通配符，则`<dest>`必须是一个以`/`结尾的目录路径；如果`<dest>`不以`/`结尾，则其被视为一个普通文件，`<src>`的内容将被直接写入到`<dest>`

### 4.6. WORKDIR
+ 用于为Dockerfile中所有的RUN、CMD、ENTRYPOINT、COPY和ADD指定设定工作目录
+ WORKDIR可以指定多次，每次都会对语句之后的指令生效，其路径也可以为相对路径，不过，他是相对此前一个WORKDIR指令指定的路径
+ WORKDIR也可以调用由ENV指定定义的变量
+ 语法：
``` bash
WORKDIR <dirpath>
```

### 4.7. VOLUME
+ 用于在image总创建一个挂载点目录，以挂载Docker host上的卷或其他容器上的卷。
+ 我们学过docker在启动的时候可以指定卷，有两种方式，可以参考前面的文章，但是dockerfile只有一种，就是docker自己管理的卷
+ 语法：
``` bash
VOLUME <mountpoint>
```
或者
``` bash
VOLUME ["<mountpoint>"]
```
+ 如果挂载点目录路径下此前文件存在，docker run命令会在挂载完成后讲此前的所有文件复制到新挂载的卷中

### 4.8. EXPOSE
+ 用于为容器打开指定要监听的端口以实现与外部通信
+ 复习一下docker的-p选项暴露端口，相当于打开在net桥上监听的，暴露给主机或者宿主机之外的其他客户端访问的，相当于自动生成了一个DNAT规则
+ 但是，dockerfile中的expose是说可以暴露，而不是一定要暴露，也就是说，如果想要暴露给其他客户端访问，要使用docker的-p指定暴露端口或者-P选项才能真正暴露，-P选项的说明就是暴露所有需要暴露的端口，而需要暴露的端口就是说dockerfile的expose选项指定的端口
+ 语法
``` bash
EXPOSE <port>[/<protocol>] [<port>[/<protocol>]...]
```
+ `<protocol>`用于指定传输层协议，可为tcp或udp二者之一，默认为TCP协议
+ EXPOSE指令可以一个指定多个端口，例如
``` bash
EXPOSE 11211/udp 11211/tcp
```
+ 注意，这个暴露端口不是说


### 4.9. ENV
+ 用于为镜像定义所需的环境变量，并可被Dockerfile文件中位于气候的其他命令所调用
+ 调用格式为$variable_name或${variable_name}
+ 语法
``` bash
ENV <key> <value>
ENV <key>=<value>...
```
+ 第一种格式中，`<key>`之后的所有内容均会被视作其`<value>`的组成部分，因此，一次只能设置一个变量
+ 第二种格式可以一次设置多个变量，每个变量为一个`<key>=<value>`的键值，如果`<value>`中包含空格，可以以反斜线（\）进行转义，也可通过对`<value>`加引号进行表示；另外，反斜线也可用于续行；
+ 定义多个变量时，建议使用第二种方式，以便在同一层中完成所有功能
+ 如果我们在docker run的时候，使用-e来传递变量会不会影响dockerfile中的变量，而使得镜像出现问题呢。其实这是两个不同的过程，run是启动，build是镜像构建。我们在dockerfile中定义的环境变量，在docker容器启动之后可以直接使用，所以我们即使给同一个环境变量传值，他是不会影响build的过程的

### 4.10. RUN
+ 用于指定docker build过程中运行的程序，其可以是任何命令
+ 语法
``` bash
RUN <command>
RUN ["<executable>", "<param1>", "<param2>"]
```
+ 第一种格式中，`<command>`通常是一个shell命令，且以"/bin/sh -c"来运行他，这意味着此进程在容器中的PID不为1，不能接受Unix信号，因此，当使用`docker stop <container>`，命令停止容器时，此进程接收不到SIGTER M信号；
+ 第二种语法格式中的参数是一个JSON格式的数组，其中`<executable>`为要运行的命令，后面的`<paramN>`为传递给命令的选项或者参数；然而，此种格式指定的命令不会以"/bin/sh -c"来发起，因此常见的sheel操作，比如变量替换以及通配符`(? *)`替换讲不会进行；不过，如果要运行的命令依赖于此shell特性的话，可以将其替换为类似下面的格式
``` bash
RUN ["/bin/bash", "-c", "<executable>", "<param1>"]
```

### 4.11. CMD
+ 类似于RUN命令，CMD指令也可用于运行任何命令或程序，不过，二者的运行时间点不同
  + RUN指令运行于映像文件构建过程中，而CMD指令运行于基于Dockerfile构建出的新影响文件启动一个容器时
  + CMD指令的首先目的在于为启动的容器指定默认要运行的程序，而且如果运行结束，容器也将终止；不过，CMD指定的命令可以被docker run命令行选项所覆盖
  + 在Dockerfile中可以存在多个CMD命令，但仅最后一个会生效
+ 语法
``` bash
CMD <command> # 这种方式是没法接受docker stop命令的，因为他的进程id号不为1，没法接受docker stop命令
CMD ["<executable>", "<param1>", "<param2>"]
CMD ["<param1>", "<param2>"]
```
+ 前两种语法格式的意义同RUN
+ 第三种则用于为ENTRYPOINT指令提供默认参数



### 补充shell的一点知识
如果我们运行一个会运行为守护进程的程序，比如nginx，他在启动之后是作为shell的子进程存在的，用户创建并启动进程的接口是shell，如果我们打开命令行提示符，就证明我们正在运行一个shell进程，而后我们在命令行创建的任何进程都应该是这个shell的子进程，有些进程会占据当前shell进程的终端设备，也就是我们平时说的前台运行。如果我们想在启动的时候就把他送到后台去运行，需要在进程序结尾加上一个`&`。但是这样也不能够脱离和shell的关系，他的父进程依然是当前启动的shell。任何启动中的命令在
终止的时候，会先把自己的子进程先停止再把自己停止，也就是说，如果我们退出shell，那么所有当前shell启动的程序也都会被停止。如果想在启动的时候让进程的父进程变成系统进程，就需要使用nohub。这样就会让启动的进程剥离与当前shell的关系，直接安排给init。

我们在容器当中要启动一个进程，这个是由内核直接启动还是托管给shell进程就很重要了。一般来说都是委托给内核，也就是id号为1的进程，而id号为1的进程是init。我们平时使用的通配符，输入输出重定向，管道这些，比如`ls /var/*`都是shell的功能，而不是内核功能。但是如果程序基于shell进程启动，那么他们的父进程就不是id进程为1的程序了。这就是我们前面说过的EXEC了，使用exec就可以基于shell来启动，让shell的进程id为1。也就是`exec COMMAND`这样一来exec就顶替init，成为id为1的进程。


### 4.12. ENTRYPOINT
+ 类似CMD指令的功能，用于为容器指定默认运行程序，从而使得容器像是一个单独的可执行程序
+ 与CMD不同的是，由ENTRYPOINT启动的程序不会被docker run命令指定的参数所覆盖，而且，这些命令行参数会被当做参数传递给ENTRYPOINT指定的程序，但是docker run命令的--entrypoint选项的参数可覆盖ENTRYPOINT指令指定的程序
+ 语法
``` bash
ENTRYPOINT <cmomand>
```
或者
``` bash
ENTRYPOINT ["<executable>", "<param1>", "<param2>"]
```
+ docker run命令传入的命令参数会覆盖CMD指令的内容并且附加到ENTRYPOINT命令最后作为其参数使用
+ Dockerfile文件中也可以存在多个ENTRYPOINT指令，但仅有随后一个会生效
+ 如果CMD和ENTRYPOINT同时存在，那么CMD会被作为参数传递给ENTRYPOINT

### 灵活接受参数的Dockerfile
如果我们想要灵活的接受配置应该这样设置
``` bash
FROM nginx:1.14-alpine
LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”

ENV NGX_DOC_ROOT=‘/data/web/html’

ADD index.html ${NGX_DOC_ROOT}
ADD entrypoint.sh /bin/

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
ENTRYPOINT ["/bin/entrypoint.sh"]
```
entrypoint.sh
``` bash
#!/bin/sh
#
cat > /etc/nginx/conf.d/www.conf << EOF
server {
	servver_name $HOSTNAME;
	listen ${IP:-0.0.0.0}:${PORT:-80};
	root ${NGX_DOC_ROOT:-/usr/share/nginx/html};
}
EOF

exec "$@" 
```
打包镜像
``` bash
docker build ./ -t myweb:v0.3-1
```
运行容器
``` bash
docker run --name myweb:v0.3-1
```
然后我们还可以传递变量
``` bash
docker run --name --rm -P -e "PORT=8080" myweb:v0.3-1
```
然后我们可以通过这种方式就可以做一个镜像，其他的作为参数传递给镜像。

### 4.13. USER
+ 用于指定运行image时或者运行dockerfile中任何RUN，CMD或ENTRYPOINT指令指定的程序的用户名或UID
+ 默认情况下，容器的运行身份为root用户
+ 语法：
``` bash
USER <UID>|<UserName>
```
需要注意的是`<UID>`可以为任意数字，但实践中，他必须为/etc/passwd中某用户的有效UID，否则，docker run命令将运行失败。

### 4.14. HEALTHCHECK
docker容器在主进程退出的时候，容器就会终止。如果我们有一个nginx容器，但是我们的DOC_ROOT指定的时候指定错了，但是DOC_ROOT指定的那个目录也存在，那么这个时候，nginx进程启动是不会报错的，但是我们访问的时候会报错，所以docker引擎来判断里面的程序健康与否不是去检查主进程能不能正常提供服务，而仅仅是看他是不是运行的。因此，他的判断机制并不健全，我们就需要其他的工具来帮助我们。比如，我们可以在本机使用一个curl命令对这个服务进行访问，如果返回的结果是我们想要的，就认为他是健康的，而不仅仅是查看进程是否存在来判断进程健康与否。那么我们就有了HEALTHCHECK指令

+ 这个指令告诉Docker怎样测试容器，来保证他一直处于运行状态
+ 这个可以用来叹词web服务器是否陷入死循环而无法响应新的链接
+ 语法
``` bash
# 在容器内使用命令来检查容器健康与否
HEALTHCHECK [OPTIONS] CMD command
# 禁用所有的healthcheck
HEALTHCHECK NONE
```
+ HEALTHCHECK命令可以通过`OPTION`选项来定义一些周期性的检测机制
  + --interval=<间隔>：两次健康检查的间隔，默认为 30 秒；
  + --timeout=<时长>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
  + --start-period=<时长>：等待多长时间开始做检测，默认为0
  + --retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
+ 在 HEALTHCHECK [选项] CMD 后面的命令，格式和 ENTRYPOINT 一样，分为 shell 格式，和 exec 格式。命令的返回值决定了该次健康检查的成功与否：0：成功；1：失败；2：保留，不要使用这个值。
+ 例子
``` bash
HEATHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost/ || exit 1 
HEATHCHECK --start-period=3s --interval=5m --timeout=3s \
  CMD wget -O - -q http://${IP:-0.0.0.0}:${PORT:80}/
```

### 4.15. SHELL
+ 这个指令允许指定程序默认运行的shell
+ 默认的Linux shell是["/bin/sh", "-c"]，而Windows是["cmd", "/S", "/C"]
+ shell指令在Dockerfile中必须以JSON的形式存在，也就是`SHELL ["executable", "parameters"]`
+ SHELL指令可以出现多次
+ 每个SHELL指令都会覆盖前面的SHELL指令，然后对新SHELL指令下面的指令生效

### 4.16. STOPSIGNAL
+ 这个指令是设置系统调用中，让容器停止的信号的信号的，我们默认让容器停止是-15
+ 这个信号可以是内核中任何的有效的，没有被使用的数字，也可以使用9，那么如果我们使用docker stop，就是强行杀死这个进程
+ 语法：`STOPSIGNAL` signal

### 4.17. ARG
+ 构建参数和 ENV 的效果一样，都是设置环境变量。所不同的是，ARG 所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。但是不要因此就使用 ARG 保存密码之类的信息，因为 docker history 还是可以看到所有值的。
+ Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg <参数名>=<值> 来覆盖。
+ 在 1.13 之前的版本，要求 --build-arg 中的参数名，必须在 Dockerfile 中用 ARG 定义过了，换句话说，就是 --build-arg 指定的参数，必须在 Dockerfile 中使用了。如果对应参数没有被使用，则会报错退出构建。从 1.13 开始，这种严格的限制被放开，不再报错退出，而是显示警告信息，并继续构建。这对于使用 CI 系统，用同样的构建流程构建不同的 Dockerfile 的时候比较有帮助，避免构建命令必须根据每个 Dockerfile 的内容修改。
+ 格式：`ARG <参数名>[=<默认值>]`
比如：
``` bash
ARG author=“CLOUDNATIVE365 <29371962@qq.com>”
LABEL maintainer="${author}"
```
如果我们build的时候想build1.15的alpine，那么我传值的时候就需要这样build
``` bash
docker build --build-arg author="JORMUN <29371962@qq.com>" -t myweb:v0.3-9 ./
```
+ 其实我们可以理解ARG是给build传ENV的，因为build时候是不允许传递ENV值的，ENV只能在docker run的时候传值

### 4.19. ONBUILD
+ 用于在Dockerfile中定义一个触发器
+ Dockerfile用于build映像文件，此映像文件也可以作为base image被另一个Dockerfile用作FROM指令的参数，并用他来构建新的影响文件
+ 在后面的这个Dockerfile中的FROM指令在build过程中被执行时，将会”触发“创建其base image的Dockerfile文件中的ONBUILD指令定义的触发器
+ 语法：`ONBUILD <INSTRUCTION>`
+ 尽管任何指令都可注册成为触发器指令，但ONBUILD不能自我嵌套，且不会触发FROM和MAINTAINER指令
+ 使用包含ONBUILD指令的Dockerfile构建的镜像应该使用特殊的标签，例如ruby:2.0-onbuild
+ 在ONBUILD指令中使用ADD或COPY指令应该格外小心，因为新构建过程的上下文在缺少指定的源文件时会失败

## 例子
### EX1
我们来试一下，首先创建工作目录并进入
``` bash
mkdir img1
cd img1
```
创建Dockerfile
``` bash
cat << EOF > Dockerfile
# Description: test image1
# 其实在生产中建议使用busybox@hashcode的格式，这样更安全
FROM busybox:latest
MAINTAINER “CLOUDNATIVE365 <29371962@qq.com>”
# LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”
# copy文件
COPY index.html /data/web/html/
# copy目录
COPY yum.repos.d /etc/yum.repos.d/
# ADD的url下载
ADD http://nginx.org/download/nginx-1.17.9.tar.gz /usr/local/src
# ADD的本地文件
ADD nginx-1.17.9.tar.gz /usr/local/nginx
# workdir指定之后可以使用它作为相对路径
WORKDIR /usr/local/src
ADD http://nginx.org/download/nginx-1.17.9.tar.gz ./
# 指定多个workdir的时候，指令会向上找到第一个带有WORKDIR指令的行，让那个值作为WORKDIR
WORKDIR /usr/local/
ADD nginx-1.17.9.tar.gz ./
# 指定卷，使用inspect也可以查看
VOLUME /data/mysql/
# 指定ENV
ENV DOC_ROOT /data/web/html
COPY index.html $DOC_ROOT
# 如果DOC_ROOT没有值，就需要指定默认值
COPY index.html ${DOC_ROOT:-/data/web/html}
# ENV多行
ENV DOC_ROOT /data/web/html \
         WEB_SERVER_PACKAGE="nginx-1.17.9"
ADD $WEB_SERVER_PACKAGE.tar.gz ./src/
# RUN
RUN cd /usr/local/src/ && \
         tar xf nginx-1.17.9.tar.gz && \
				 mv nginx-1.17.9 nginx
EOF
# 暴露端口
EXPOSE 80/tcp
```
然后在WORKDIR下添加一个index.html文件
``` bash
cat << EOF > index.html
<h1>busybox index</h1>
EOF
```
这时就可以打包了
``` bash
docker build ./ -t tinyhttpd:v0.1
```
验证一下
``` bash
docker run --name tinyweb1 --rm tnyhttpd:v0.1-1 cat /data/web/html/index.html
docker run --name tinyweb1 --rm tnyhttpd:v0.1-2 ls /etc/yum.repos.d/
docker run --name tinyweb1 --rm tnyhttpd:v0.1-3 ls /usr/local/src/
docker run --name tinyweb1 --rm tnyhttpd:v0.1-4 ls /usr/local/
docker run --name tinyweb1 --rm tnyhttpd:v0.1-5 mount
docker run --name tinyweb1 --rm tnyhttpd:v0.1-6 /bin/httpd -f -h /data/web/html
```

### EX2
为了说明CMD和RUN的区别，我们做一个新的镜像img2
``` bash
mkdir img2
cd img2
```
定义一个dockerfile
``` bash
cat << EOF > Dockerfile
FROM busybox
LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”

ENV WEB_DOC_ROOT="/data/web/html/"

RUN mkdir -p WEB_DOC_ROOT && \
         echo "<h1>Hello Busybox server.</h1>" > ${WEB_DOC_ROOT}/index.html
				 
CMD /bin/httpd -f -h ${WEB_DOC_ROOT}
EOF
```
打包成为镜像
``` bash
docker build -t tinyhttpd:v0.2-1 ./
```
但是当我们运行命令去启动容器的时候
``` bash
docker run --name tinyweb2 -it --rm -P tinyhttpd:v0.2-1
```
终端会卡住，即使我们加了交互式选项`-it`也不行，因为我们目前是在httpd的进程下，httpd是没有交互式接口的，但是我们可以使用exec命令来进入交互式接口
``` bash
docker exec -it tinyweb2 /bin/sh
```
就可以进入shell交互式接口了，但是进入容器后使用`ps`命令，会发现PID为1的进程是httpd命令，这样就保证了这个容器可以接受到docker stop的信号。这个地方 1 号进程是 httpd，是因为 httpd 镜像中的 sh -c 处理有点特殊，不是 Docker 自动做 exec COMMAND。
如果我们把dockerfile改成下面的样子
``` bash
cat << EOF > Dockerfile
FROM busybox
LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”

ENV WEB_DOC_ROOT="/data/web/html/"

RUN mkdir -p WEB_DOC_ROOT && \
         echo "<h1>Hello Busybox server.</h1>" > ${WEB_DOC_ROOT}/index.html
				 
CMD ["/bin/httpd", "-f", "-h ${WEB_DOC_ROOT}"]
EOF
```
打包成为镜像
``` bash
docker build -t tinyhttpd:v0.2-2 ./
```
但是当我们运行命令去启动容器的时候
``` bash
docker run --name tinyweb2 -it --rm -P tinyhttpd:v0.2-2
```
就会报错，说没有${WEB_DOC_ROOT}，我们就需要修改一下，成为
``` bash
cat << EOF > Dockerfile
FROM busybox
LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”

ENV WEB_DOC_ROOT="/data/web/html/"

RUN mkdir -p WEB_DOC_ROOT && \
         echo "<h1>Hello Busybox server.</h1>" > ${WEB_DOC_ROOT}/index.html
				 
CMD [“/bin/sh”, "-c", "/bin/httpd", "-f", "-h ${WEB_DOC_ROOT}"]
EOF
```

### EX3
ENTRYPOINT
定义一个dockerfile
``` bash
cat << EOF > Dockerfile
FROM busybox
LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”

ENV WEB_DOC_ROOT="/data/web/html/"

RUN mkdir -p WEB_DOC_ROOT && \
         echo "<h1>Hello Busybox server.</h1>" > ${WEB_DOC_ROOT}/index.html
				 
ENTRYPOINT /bin/httpd -f -h ${WEB_DOC_ROOT}
EOF
```
打包成为镜像
``` bash
docker build -t tinyhttpd:v0.2-5 ./
```
我们运行命令去启动容器的时候没有问题
``` bash
docker run --name tinyweb2 -it --rm -P tinyhttpd:v0.2-5
```
但是我们运行命令去启动容器的时候加上其他的命令，作为entrypoint的时候，比如ls命令，不会有返回，他是不允许覆盖的
``` bash
docker run --name tinyweb2 -it --rm -P tinyhttpd:v0.2-5 ls /data/web/html/
```

### EX4
ENTRYPOINT和CMD同时存在
定义一个dockerfile
``` bash
cat << EOF > Dockerfile
FROM busybox
LABEL maintainer=“CLOUDNATIVE365 <29371962@qq.com>”

ENV WEB_DOC_ROOT="/data/web/html/"

RUN mkdir -p WEB_DOC_ROOT && \
         echo "<h1>Hello Busybox server.</h1>" > ${WEB_DOC_ROOT}/index.html
				 
CMD ["/bin/httpd", "-f", "-h ${WEB_DOC_ROOT}"]
ENTRYPOINT ["/bin/sh", "-c"]
EOF
```
打包成为镜像
``` bash
docker build -t tinyhttpd:v0.2-6 ./
```
我们在运行ls命令就没问题了，因为后面的命令作为参数传递给了ENTRYPOINT，而CMD中定义的是默认参数，默认没有参数才会使用CMD，有参数的话就使用我们传递的参数
``` bash
docker run --name tinyweb2 -it --rm -P tinyhttpd:v0.2-5 ls /data/web/html/
```