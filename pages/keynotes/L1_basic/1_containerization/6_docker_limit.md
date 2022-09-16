---
title: Docker镜像管理基础
keywords: keynotes, basic, containerization, docker_image
permalink: keynotes_L1_basic_1_containerization_6_docker_limit.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/6_docker_limit
typora-root-url: ../../../../../cloudnative365.github.io
---

## 限制容器的资源
容器默认是没有资源限制的，可以使用宿主机内核调度给他的所有的资源。Docker提供了一种方式，可以控制容器使用多少内存，CPU和Block IO，需要在使用docker run的时候提供运行时的提供参数。
我们前面说过，CPU是可压缩资源，而Memory是不可压缩资源。

### Memory
+ 在linux主机上，进程启动的时候，如果内核检测到内存不足以运行了，就会抛出一个OOME，叫做Out Of Memory Exception，然后就开始杀死其他进程。在杀进程的时候会挑选内存hogs，找优先级比较低的进程杀，如果优先级一样，那么就会随机杀进程。一旦发生OOME，任何进程都有可能被杀死，包括docker daemon在内。所以，Docker特意调整了docker daemon的OOM优先级，以避免被杀死，但是容器的优先级是没有被调整过的。那么，那些进程是应该被杀死的呢，其实每个进程都有一个OOM_SCORE，然后排序，从上到下先把得分最高的杀死，如果不够，继续杀。而在OOM_SCORE的标准当中有一个OOM_ADJ，就是OOM的权重，也可以简单理解为优先级，这个级别越低，越不容易被杀死，而docker daemon就是调整了这个参数。
+ 下面的选项就描述了怎样限制内存

![file](https://graph.baidu.com/resource/2227ed44727c8fbda3c2001586501742.png)

+ 在内存级别，我们有两个维度去限制，一个是物理内存，一个是虚拟内存。实际上的量级，我们可以使用k，m，g来限制，而例子中是4m。
+ 如果我们要使用--memory-swap选项，就必须设置-m选项，而memory swap也有一些选项，比如正数，0，没设置和-1

![file](https://graph.baidu.com/resource/222b092ad5932e79db12501586503855.png)

而他们的关系可以看下面的表

![file](https://graph.baidu.com/resource/22275c550106331084f4701586509017.png)

+ 最后，我们防止程序被oom kill的时候需要使用--oom-kill-disable选项或者--oom-score-adj选项

### CPU
+ 而CPU默认也是没有限制的
+ 我们还是需要通过参数来配置CPU的周期，下面的参数就是怎样限制内存

![file](https://graph.baidu.com/resource/222ef562a80737dd745ac01586501936.png)

+ 大多数系统都是使用CFS来调度的，CFS，一般来说，每个进程运行都需要一个CPU，但是如果进程的数量大于CPU的数量，我们进程运行的时候的优先级就需要进程调度器来决定这个就是CFS（完全公平调度算法），他决定的是进程在【100-139】之间的数值，数值越低级别越高。那么我们在对于应用分类的时候，会把应用分成两类，一个是CPU密集型的，一个是IO密集型的。对于CPU密集型的，我们要尽量调低他的优先级，因为只要被调度到CPU上，他就尽可能的消耗CPU的时间。我们内核上有一些算法，会对那些对于CPU占用时间较长的程序做一些惩罚措施，动态调底他的优先级。而对于那些starve的程序，由于很长时间没有被调度到CPU，就会适当调高他的优先级。

+ 在Docker1.13或者更高的版本，我们还可以配置实时调度策略【0-99】，这个都是内核级别的调度器，数值越高，级别越高

+ 一般限制有三种方式
  + --cpus是指可以sh使用多少个核，比如1.5个核，如果我们系统上有4核CPU，那么只要满足使用1.5个核就可以了，不一定会被调度到那个核心上，
  + 如果想指定核心，就需要使用--cpuset-cpus选项
  + 最后一种方式是指定压缩比例的方式 --cpu-shares

### 压测工具stress
我们可以使用dockerhub上一个专门测试容器的镜像叫stress
``` bash
docker run --name stress -it --rm -m 256m lorel/docker-stress-ng:latest stress --vm 2
docker run --name stress -it --rm --cpus 2 orel/docker-stress-ng:latest stress --cpu 8
docker run --name stress -it --rm lorel/docker-stress-ng:latest stress --cpu 8
docker run --name stress -it --rm --cpuset-cpus 0,2 lorel/docker-stress-ng:latest stress --cpu 8
docker run --name stress -it --rm --cpu-shares 1024 lorel/docker-stress-ng:latest stress --cpu 8
```
然后我们可以使用`docker top`和`docker stat`命令来追踪系统资源的使用