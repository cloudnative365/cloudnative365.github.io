---
title: 常见监控指标
keywords: keynotes, architect, monitoring, normal_metrics
permalink: keynotes_L4_architect_2_monitoring_6_common_metrics.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/6_common_metrics
typora-root-url: ../../../../../cloudnative365.github.io

---

## 学习目标

+ 常见监控指标之CPU
+ 常见监控指标之内存
+ 常见监控指标之硬盘
+ 常见监控指标之网络
+ 常见监控指标之进程

了解常见的需要上监控的系统

挑选合适的软件来监控

常见的监控指标

## 1. 简介

我们前面说过了，从运维复杂度的角度考虑系统架构，而模板就是一个我们需要考虑的问题。一些传统的监控软件，它内部提供了非常丰富的模板，让我们的软件安装完成之后，就可以用最少的工作量，或者一个漂亮的图形，比如zabbix，它内置了大量的模板，让我们展示起来非常方便。而比较新的监控软件则采取了不同的思路，比如grafana，他官方提供的模板非常少，大部分都来自于社区，然后他把社区内的一些模板集中起来，做了一些统一的验证和质量控制，做到了grafana.com/grafana/dashborads和pugins中，以扩展的形式来丰富软件的功能。我们后面说grafana的时候会详细再说，我们今天就是要使用

+ 系统命令
+ grafana的dashborad（grafana7+dashboard1860）我用了一个下载了480多万次的模板做参考
+ zabbix的dashborad（zabbix5）
+ node_exporter（1.0.1）

中常用的一些指标来做例子，来说明我们常用的一些指标，比如

+ CPU
+ 内存
+ 硬盘
+ 网络
+ 进程

## 2.  CPU监控指标

CPU是计算机的大脑，他中间涉及到了非常多的指标。首先我们要知道的，是CPU主要由两个空间来使用，一个是用户空间，一个是系统/内核空间。

### 2.1. 系统上查看CPU指标

我们在troubleshooting的时候通常会登录控制台来定位问题，那么久需要用到一些系统工具或者命令来查看CPU指标

+ 通过命令，比如：TOP，vmstat，sar
+ 通过文件，一切皆文件，实际上我们CPU的指标，都是储存在文件中的，命令或者工具不过是把它们的瞬时状态拿出来做了一些出来，用我们比较舒服的方式展示出来。一般是来自于/proc/stat

我们以vmstat为例，他的输出如下：

``` bash
root@node5:~# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 3316576  25828 290740    0    0     9    10   91  111  3  1 96  0  0
```

咱们先说CPU部分，他分为了system和CPU，我们man一下

``` bash
   System
       # 系统时钟周期每秒的中断的次数
       in: The number of interrupts per second, including the clock.
       # 系统上下文每秒切换的次数
       cs: The number of context switches per second.

   CPU
       These are percentages of total CPU time.
       # 运行用户空间代码所消耗的时间
       us: Time spent running non-kernel code.  (user time, including nice time)
       # 运行内核空间代码所消耗的时间
       sy: Time spent running kernel code.  (system time)
       # 没有程序执行的时间
       id: Time spent idle.  Prior to Linux 2.5.41, this includes IO-wait time.
       # 程序花费在等待运行时候的时间
       wa: Time spent waiting for IO.  Prior to Linux 2.5.41, included in idle.
       # 交给虚拟机运行的时候，主要是说hypervisor层使用的时间
       st: Time stolen from a virtual machine.  Prior to Linux 2.6.11, unknown.
```

System主要是衡量CPU中断的指标，我们知道，CPU是通过切换时钟周期来实现多任务并发的，他的切换速度足够快，才让我们感觉系统是在同时运行很多程序。但是，切换可以简单分为两个步骤。

+ 程序知道应该让出CPU使用权而主动放弃，CPU出于某种考虑强制要求切换
+ 切换时，如果下一个时钟周期还是上一个时钟周期内运行的程序，则不需要切换上下文，如果其他程序获得了执行权，CPU会把他对应的指令从内存或者CPU缓存中拿出来进行计算，这个时候就需要切换上下文。

这个指标有两种异常

+ 一种是切换次数非常频繁，那么说明系统非常忙碌。
+ 第二种是in的次数非常高，但是cs次数非常低，说明一个某个程序占据了大量的CPU计算能力

CPU主要是衡量CPU本身的消耗时间，他包括了内核空间和用户空间，这里面的单位是percentage，也就是百分比，也就是说，前三个us，sy和id加起来应该是100%

### 2.2. zabbix上的CPU监控指标

zabbix的CPU监控指标在`templates/Operating Systems/Template OS Linux by Zabbix agent`的item下面，有一个CPU指标，里面一共有17项。

![image-20200721201156652](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200721201156652.png)

这里面把系统的中断细分了一下

+ CPU softirq time（软中断）：The amount of time the CPU has been servicing software interrupts.（这边就是我们说的软件主动放弃的情况）
+ CPU interrupt time（硬中断）：The amount of time the CPU has been servicing hardware interrupts.（由于各种原因，CPU控制器主动要求中断）

我们发现还有个参数叫niced time，这个是说用做nice加权的进程分配的用户态cpu时间比

> The time the CPU has spent running users' processes that have been niced.

然后guest niced time就是用做nice加权的虚拟出来的主机分配的用户态cpu时间比

> Time spent running a niced guest (virtual CPU for guest operating systems under the control of the Linux kernel)

其他的应该都比较好理解了。

### 2.3. grafana上的CPU监控指标

感觉还没有zabbix的指标全面，但是可以非常清晰的看到我们想要的东西

![image-20200721211732297](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200721211732297.png)

都是我们见过的，我就不详细说了，实际上每个dashboard的数据源都是我们下面要说的node_exporter，让我们看看node_eporter的指标是不是更全

#### 2.4. node_exporter上的CPU监控指标

我们可以curl一下目标机器的9100端口，然后找到带CPU的指标

``` bash
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 18064.83
node_cpu_seconds_total{cpu="0",mode="iowait"} 34
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 0
node_cpu_seconds_total{cpu="0",mode="softirq"} 18.34
node_cpu_seconds_total{cpu="0",mode="steal"} 0
node_cpu_seconds_total{cpu="0",mode="system"} 80.34
node_cpu_seconds_total{cpu="0",mode="user"} 277.38
node_cpu_seconds_total{cpu="1",mode="idle"} 18080.73
node_cpu_seconds_total{cpu="1",mode="iowait"} 23.76
node_cpu_seconds_total{cpu="1",mode="irq"} 0
node_cpu_seconds_total{cpu="1",mode="nice"} 0
node_cpu_seconds_total{cpu="1",mode="softirq"} 6.62
node_cpu_seconds_total{cpu="1",mode="steal"} 0
node_cpu_seconds_total{cpu="1",mode="system"} 71.46
node_cpu_seconds_total{cpu="1",mode="user"} 313.63
node_cpu_seconds_total{cpu="2",mode="idle"} 18077
node_cpu_seconds_total{cpu="2",mode="iowait"} 30.14
node_cpu_seconds_total{cpu="2",mode="irq"} 0
node_cpu_seconds_total{cpu="2",mode="nice"} 0
node_cpu_seconds_total{cpu="2",mode="softirq"} 7.69
node_cpu_seconds_total{cpu="2",mode="steal"} 0
node_cpu_seconds_total{cpu="2",mode="system"} 71.42
node_cpu_seconds_total{cpu="2",mode="user"} 307.3
node_cpu_seconds_total{cpu="3",mode="idle"} 18084.29
node_cpu_seconds_total{cpu="3",mode="iowait"} 24.86
node_cpu_seconds_total{cpu="3",mode="irq"} 0
node_cpu_seconds_total{cpu="3",mode="nice"} 0
node_cpu_seconds_total{cpu="3",mode="softirq"} 7.04
node_cpu_seconds_total{cpu="3",mode="steal"} 0
node_cpu_seconds_total{cpu="3",mode="system"} 72.85
node_cpu_seconds_total{cpu="3",mode="user"} 304.74

# TYPE node_cpu_guest_seconds_total counter
node_cpu_guest_seconds_total{cpu="0",mode="nice"} 0
node_cpu_guest_seconds_total{cpu="0",mode="user"} 0
node_cpu_guest_seconds_total{cpu="1",mode="nice"} 0
node_cpu_guest_seconds_total{cpu="1",mode="user"} 0
node_cpu_guest_seconds_total{cpu="2",mode="nice"} 0
node_cpu_guest_seconds_total{cpu="2",mode="user"} 0
node_cpu_guest_seconds_total{cpu="3",mode="nice"} 0
node_cpu_guest_seconds_total{cpu="3",mode="user"} 0
...
```

其实就是我们在grafana的dashboard上看到的，只不过他是用0-3标识了4颗CPU，然后做了一个聚合。具体可以看到的内容和zabbix的差不多。也就是说，dashboard上并没有展示所有的参数，随着node_exporter版本的升级，暴露的指标越来越详细

## 3. 内存监控指标

内存是系统重要的指标之一，而且是必须提前规划好的资源。因为他和CPU不同，如果CPU资源不足，会导致系统变慢。如果内存资源不足，会触发oom，直接杀掉级别较低的程序。当然，我们还有一些swap可以作为临时的内存周转，主要还是为了防止oom，但是硬盘的速度和内存比简直是一天一地，肯定会拖慢系统速度。最后，现在很多容器技术是不可以使用缓存的，一旦内存满了，直接oom了，比如我们的k8s就是这样的例子。

### 3.1. 系统上查看内存指标

同样是两种方式

+ 命令和工具：top，vmstat，free
+ 文件：/proc/meminfo

工具同样是从文件中取得的数据，所以文件中的指标是最全的。

``` bash
MemTotal:        3898628 kB
MemFree:         3508100 kB
MemAvailable:    3700256 kB
Buffers:           21516 kB
Cached:           210792 kB
SwapCached:            0 kB
Active:           113460 kB
Inactive:         145828 kB
Active(anon):      19828 kB
Inactive(anon):     8716 kB
Active(file):      93632 kB
Inactive(file):   137112 kB
Unevictable:          16 kB
Mlocked:              16 kB
SwapTotal:        102396 kB
SwapFree:         102396 kB
Dirty:                24 kB
Writeback:             0 kB
AnonPages:         27000 kB
Mapped:            35060 kB
Shmem:              8956 kB
Slab:              71708 kB
SReclaimable:      36152 kB
SUnreclaim:        35556 kB
KernelStack:        2304 kB
PageTables:         1376 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     2051708 kB
Committed_AS:     138840 kB
VmallocTotal:   263061440 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
Percpu:              960 kB
CmaTotal:         262144 kB
CmaFree:          223260 kB
```

尽管我们通常不会监控这么细，但是还是希望大家可以深入理解一下，因为有些特殊场景可能会用到，比如内核调优或者特殊软件需求（oracle的bigpage之类）。

而监控中，我们通常是从vmstat和free这两个方面来监控

``` bash
root@node2:~# free -m
              total        used        free      shared  buff/cache   available
Mem:           3807         119        3425           8         262        3613
Swap:            99           0          99
root@node2:~# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 3507864  21520 247080    0    0     0     0   26   41  0  0 100  0  0
```

free中的几个选项

+ Mem:
  + total: 总共的内存
  + used: 使用的内存
  + free: 空闲的内存
  + shared：共享的内存（shmem）
  + buff/cache：缓存内存区（buffer 写缓存/cache读缓存）
  + avilable：可用的内存
+ Swap:
  + total：一共的交换分区
  + used：使用的交换分区
  + free：空闲的交换分区

应该只有buff/cache不好理解。

+ A buffer is something that has yet to be "written" to disk.　　---buffer 写缓存，数据存储时，先保存到磁盘缓冲区，然后再写入到永久空间
+ A cache is something that has been "reed" from the disk adn stored for later use.  --cache 读缓存，数据从磁盘读出后，暂留在缓冲区，预备程序接下来的使用，

### 2.2. zabbix上的内存监控指标

和我们使用free看到的指标基本一样

![image-20200722205507381](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200722205507381.png)

### 2.3. grafana上的内存监控指标

比zabbix上的多一些，对于内存的指标监控的更加详细

![image-20200722210034152](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200722210034152.png)

+ Apps：把用户空间占用的内存拿出来单独作为指标
+ PagetTables：物理和虚拟内存的映射表占用了一部分内存
+ SwapCache：从swap分区中拿出来但是没有使用的内存
+ Slab：内核用来缓存数据的

### 2.4. node_exporter上的内存监控指标

这个上面的指标是最全的，几乎涵盖了我们上面说的所有指标

``` bash
# HELP node_memory_Active_anon_bytes Memory information field Active_anon_bytes.
# TYPE node_memory_Active_anon_bytes gauge
node_memory_Active_anon_bytes 2.23305728e+08
# HELP node_memory_Active_bytes Memory information field Active_bytes.
# TYPE node_memory_Active_bytes gauge
node_memory_Active_bytes 3.76311808e+08
# HELP node_memory_Active_file_bytes Memory information field Active_file_bytes.
# TYPE node_memory_Active_file_bytes gauge
node_memory_Active_file_bytes 1.5300608e+08
# HELP node_memory_AnonPages_bytes Memory information field AnonPages_bytes.
# TYPE node_memory_AnonPages_bytes gauge
node_memory_AnonPages_bytes 1.43417344e+08
# HELP node_memory_Bounce_bytes Memory information field Bounce_bytes.
# TYPE node_memory_Bounce_bytes gauge
node_memory_Bounce_bytes 0
# HELP node_memory_Buffers_bytes Memory information field Buffers_bytes.
# TYPE node_memory_Buffers_bytes gauge
node_memory_Buffers_bytes 5.2629504e+07
# HELP node_memory_Cached_bytes Memory information field Cached_bytes.
# TYPE node_memory_Cached_bytes gauge
node_memory_Cached_bytes 3.90656e+08
# HELP node_memory_CmaFree_bytes Memory information field CmaFree_bytes.
# TYPE node_memory_CmaFree_bytes gauge
node_memory_CmaFree_bytes 2.2861824e+08
# HELP node_memory_CmaTotal_bytes Memory information field CmaTotal_bytes.
# TYPE node_memory_CmaTotal_bytes gauge
node_memory_CmaTotal_bytes 2.68435456e+08
# HELP node_memory_CommitLimit_bytes Memory information field CommitLimit_bytes.
# TYPE node_memory_CommitLimit_bytes gauge
node_memory_CommitLimit_bytes 1.996095488e+09
# HELP node_memory_Committed_AS_bytes Memory information field Committed_AS_bytes.
# TYPE node_memory_Committed_AS_bytes gauge
node_memory_Committed_AS_bytes 9.13432576e+08
# HELP node_memory_Dirty_bytes Memory information field Dirty_bytes.
# TYPE node_memory_Dirty_bytes gauge
node_memory_Dirty_bytes 36864
# HELP node_memory_Inactive_anon_bytes Memory information field Inactive_anon_bytes.
# TYPE node_memory_Inactive_anon_bytes gauge
node_memory_Inactive_anon_bytes 2.871296e+07
# HELP node_memory_Inactive_bytes Memory information field Inactive_bytes.
# TYPE node_memory_Inactive_bytes gauge
node_memory_Inactive_bytes 2.10337792e+08
# HELP node_memory_Inactive_file_bytes Memory information field Inactive_file_bytes.
# TYPE node_memory_Inactive_file_bytes gauge
node_memory_Inactive_file_bytes 1.81624832e+08
# HELP node_memory_KernelStack_bytes Memory information field KernelStack_bytes.
# TYPE node_memory_KernelStack_bytes gauge
node_memory_KernelStack_bytes 3.899392e+06
# HELP node_memory_Mapped_bytes Memory information field Mapped_bytes.
# TYPE node_memory_Mapped_bytes gauge
node_memory_Mapped_bytes 1.35671808e+08
# HELP node_memory_MemAvailable_bytes Memory information field MemAvailable_bytes.
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 3.52661504e+09
# HELP node_memory_MemFree_bytes Memory information field MemFree_bytes.
# TYPE node_memory_MemFree_bytes gauge
node_memory_MemFree_bytes 3.224031232e+09
# HELP node_memory_MemTotal_bytes Memory information field MemTotal_bytes.
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 3.992195072e+09
# HELP node_memory_Mlocked_bytes Memory information field Mlocked_bytes.
# TYPE node_memory_Mlocked_bytes gauge
node_memory_Mlocked_bytes 16384
# HELP node_memory_NFS_Unstable_bytes Memory information field NFS_Unstable_bytes.
# TYPE node_memory_NFS_Unstable_bytes gauge
node_memory_NFS_Unstable_bytes 0
# HELP node_memory_PageTables_bytes Memory information field PageTables_bytes.
# TYPE node_memory_PageTables_bytes gauge
node_memory_PageTables_bytes 1.308672e+07
# HELP node_memory_Percpu_bytes Memory information field Percpu_bytes.
# TYPE node_memory_Percpu_bytes gauge
node_memory_Percpu_bytes 1.327104e+06
# HELP node_memory_SReclaimable_bytes Memory information field SReclaimable_bytes.
# TYPE node_memory_SReclaimable_bytes gauge
node_memory_SReclaimable_bytes 4.685824e+07
# HELP node_memory_SUnreclaim_bytes Memory information field SUnreclaim_bytes.
# TYPE node_memory_SUnreclaim_bytes gauge
node_memory_SUnreclaim_bytes 5.6180736e+07
# HELP node_memory_Shmem_bytes Memory information field Shmem_bytes.
# TYPE node_memory_Shmem_bytes gauge
node_memory_Shmem_bytes 1.15044352e+08
# HELP node_memory_Slab_bytes Memory information field Slab_bytes.
# TYPE node_memory_Slab_bytes gauge
node_memory_Slab_bytes 1.03038976e+08
# HELP node_memory_SwapCached_bytes Memory information field SwapCached_bytes.
# TYPE node_memory_SwapCached_bytes gauge
node_memory_SwapCached_bytes 0
# HELP node_memory_SwapFree_bytes Memory information field SwapFree_bytes.
# TYPE node_memory_SwapFree_bytes gauge
node_memory_SwapFree_bytes 0
# HELP node_memory_SwapTotal_bytes Memory information field SwapTotal_bytes.
# TYPE node_memory_SwapTotal_bytes gauge
node_memory_SwapTotal_bytes 0
# HELP node_memory_Unevictable_bytes Memory information field Unevictable_bytes.
# TYPE node_memory_Unevictable_bytes gauge
node_memory_Unevictable_bytes 16384
# HELP node_memory_VmallocChunk_bytes Memory information field VmallocChunk_bytes.
# TYPE node_memory_VmallocChunk_bytes gauge
node_memory_VmallocChunk_bytes 0
# HELP node_memory_VmallocTotal_bytes Memory information field VmallocTotal_bytes.
# TYPE node_memory_VmallocTotal_bytes gauge
node_memory_VmallocTotal_bytes 2.6937491456e+11
# HELP node_memory_VmallocUsed_bytes Memory information field VmallocUsed_bytes.
# TYPE node_memory_VmallocUsed_bytes gauge
node_memory_VmallocUsed_bytes 0
# HELP node_memory_WritebackTmp_bytes Memory information field WritebackTmp_bytes.
# TYPE node_memory_WritebackTmp_bytes gauge
node_memory_WritebackTmp_bytes 0
# HELP node_memory_Writeback_bytes Memory information field Writeback_bytes.
# TYPE node_memory_Writeback_bytes gauge
node_memory_Writeback_bytes 0
# HELP node_sockstat_FRAG6_memory Number of FRAG6 sockets in state memory.
# TYPE node_sockstat_FRAG6_memory gauge
node_sockstat_FRAG6_memory 0
# HELP node_sockstat_FRAG_memory Number of FRAG sockets in state memory.
# TYPE node_sockstat_FRAG_memory gauge
node_sockstat_FRAG_memory 0
# HELP node_udp_queues Number of allocated memory in the kernel for UDP datagrams in bytes.
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 2.0103168e+07
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 8.26662912e+08
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes -1
```

## 4. 磁盘/存储监控指标

一般来说，我们监控存储设备的时候大多数都是在监控文件系统，也就是可以被操作系统直接使用的部分。但是实际的生产中，我们会有其他的监控需求

+ 监控那些没有被格式化成对应的文件系统（XFS，FAT32，EXT4）的磁盘，这些磁盘使用的时候使用一般的df命令是不可见的，比如：oracle的ASM盘，块存储，存储映射过来的lun（FC，iSCSI）。我们需要使用一些特殊手段才能让他们为我们所用。
+ 提供存储的设备，比如：惠普的3PAR，IBM的DS存储，EMC存储这些设备，或者是NFS服务器，Ceph/swift集群这一类提供存储功能的服务器。对于厂商的产品我们最好是去咨询原厂的工程师，关于监控指标的问题，比如是否可以有插件支持某类（zabbix，grafana）监控软件直接采集，还是有API接口，可以供外部程序采集指标。即使都没有的话，还会有snmp这种简单的方式可以让我们监控。但是snmp方式提供的指标数量有限，算是个保底的solution。而对于使用开源软件这类的解决方案，可以我们客户或者领导最想听到的是一些硬性指标，比如：随机读写的速度，顺时读写的速度等等。因为这类指标是衡量我们系统的重要依据之一。

这块我们后面会在讲分布式存储和Ceph的时候再详细说，我们这里只比较一下一些工具内置模板可以监控到的指标。

### 4.1. 系统上查看硬盘指标

同样是两类

+ 通过命令：top、iostat、vmstat、sar这类属于查看瞬时速度的和查看使用率的df类命令。或者使用dd+time命令，可以通过查看读写的结果来测试速度。当然，还有一些三方工具，比如：FIO，hdparm，smartctl

+ 通过文件：一般来说，这些磁盘也是一个文件，他们都有对应的指标，我们可以在/sys/block/sda下面找到对应的信息，sda是设备名字。当然，不是所有的linux/unix系统都是这样的，比如MacOS就找不到/sys/block目录

  ``` bash
  # ls /sys/block/sda
  alignment_offset  discard_alignment  inflight	queue	   slaves
  bdi		  ext_range	     mmcblk0p1	range	   stat
  capability	  force_ro	     mmcblk0p2	removable  subsystem
  dev		  hidden	     mq		ro	   trace
  device		  holders	     power	size	   uevent
  ```

其实系统上能看到的指标是最全面的，而我们常用的vmstat命令提供的指标也非常少

``` bash
   Swap
       si: Amount of memory swapped in from disk (/s).
       so: Amount of memory swapped to disk (/s).

   IO
       bi: Blocks received from a block device (blocks/s).
       bo: Blocks sent to a block device (blocks/s).
```

只有swap分区的读和写，块存储的读和写。我们经常会使用iostat -d来查看硬盘的IO

``` bash
$ iostat
Linux 2.6.32-431.11.15.el6.ucloud.x86_64 (ssdk1)     10/14/2016     _x86_64_    (4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.44    0.00    0.26    0.01    0.01   99.29

Device:            tps   Blk_read/s   Blk_wrtn/s   Blk_read   Blk_wrtn
vda               0.66         0.09         6.75    1404732  105885456
vdb               1.42        12.47        55.86  195619082  876552296
```

这个会显示所有的每块盘的速度

``` bash
tps：该设备每秒的传输次数
Blk_read/s：每秒从设备（drive expressed）读取的数据量；
Blk_wrtn/s：每秒向设备（drive expressed）写入的数据量；
Blk_read：  读取的总数据量；
Blk_wrtn：写入的总数量数据量；
```

然后就是df命令了，他会显示磁盘的使用率，这个是是很重要的指标，因为如果磁盘满了，和CPU一样，某些运行的程序可能会由于无法写数据而意外终止。

``` bash
df -H
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       126G  2.0G  119G   2% /
devtmpfs        1.9G     0  1.9G   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           2.0G  8.8M  2.0G   1% /run
tmpfs           5.3M  4.1k  5.3M   1% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mmcblk0p1  265M   55M  210M  21% /boot
tmpfs           400M     0  400M   0% /run/user/1000
```

### 4.2. zabbix上的存储监控指标

和我们在系统上看到的指标大同小异

![image-20200724235135473](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200724235135473.png)

### 4.3. grafana上的存储监控指标

多了一个inode的监控，其他的基本一样

![image-20200725000108563](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200725000108563.png)

### 4.4. node_exporter上的存储监控指标

这边的监控貌似多了很多

``` bash
# HELP node_disk_discard_time_seconds_total This is the total number of seconds spent by all discards.
# TYPE node_disk_discard_time_seconds_total counter
node_disk_discard_time_seconds_total{device="mmcblk0"} 0
node_disk_discard_time_seconds_total{device="mmcblk0p1"} 0
node_disk_discard_time_seconds_total{device="mmcblk0p2"} 0
# HELP node_disk_discarded_sectors_total The total number of sectors discarded successfully.
# TYPE node_disk_discarded_sectors_total counter
node_disk_discarded_sectors_total{device="mmcblk0"} 0
node_disk_discarded_sectors_total{device="mmcblk0p1"} 0
node_disk_discarded_sectors_total{device="mmcblk0p2"} 0
# HELP node_disk_discards_completed_total The total number of discards completed successfully.
# TYPE node_disk_discards_completed_total counter
node_disk_discards_completed_total{device="mmcblk0"} 0
node_disk_discards_completed_total{device="mmcblk0p1"} 0
node_disk_discards_completed_total{device="mmcblk0p2"} 0
# HELP node_disk_discards_merged_total The total number of discards merged.
# TYPE node_disk_discards_merged_total counter
node_disk_discards_merged_total{device="mmcblk0"} 0
node_disk_discards_merged_total{device="mmcblk0p1"} 0
node_disk_discards_merged_total{device="mmcblk0p2"} 0
# HELP node_disk_io_now The number of I/Os currently in progress.
# TYPE node_disk_io_now gauge
node_disk_io_now{device="mmcblk0"} 0
node_disk_io_now{device="mmcblk0p1"} 0
node_disk_io_now{device="mmcblk0p2"} 0
# HELP node_disk_io_time_seconds_total Total seconds spent doing I/Os.
# TYPE node_disk_io_time_seconds_total counter
node_disk_io_time_seconds_total{device="mmcblk0"} 11.476
node_disk_io_time_seconds_total{device="mmcblk0p1"} 0.44
node_disk_io_time_seconds_total{device="mmcblk0p2"} 11.064
# HELP node_disk_io_time_weighted_seconds_total The weighted # of seconds spent doing I/Os.
# TYPE node_disk_io_time_weighted_seconds_total counter
node_disk_io_time_weighted_seconds_total{device="mmcblk0"} 16.476
node_disk_io_time_weighted_seconds_total{device="mmcblk0p1"} 0.668
node_disk_io_time_weighted_seconds_total{device="mmcblk0p2"} 15.792
# HELP node_disk_read_bytes_total The total number of bytes read successfully.
# TYPE node_disk_read_bytes_total counter
node_disk_read_bytes_total{device="mmcblk0"} 2.32966144e+08
node_disk_read_bytes_total{device="mmcblk0p1"} 1.153536e+07
node_disk_read_bytes_total{device="mmcblk0p2"} 2.20890112e+08
# HELP node_disk_read_time_seconds_total The total number of seconds spent by all reads.
# TYPE node_disk_read_time_seconds_total counter
node_disk_read_time_seconds_total{device="mmcblk0"} 11.972
node_disk_read_time_seconds_total{device="mmcblk0p1"} 0.704
node_disk_read_time_seconds_total{device="mmcblk0p2"} 11.232000000000001
# HELP node_disk_reads_completed_total The total number of reads completed successfully.
# TYPE node_disk_reads_completed_total counter
node_disk_reads_completed_total{device="mmcblk0"} 4883
node_disk_reads_completed_total{device="mmcblk0p1"} 416
node_disk_reads_completed_total{device="mmcblk0p2"} 4447
# HELP node_disk_reads_merged_total The total number of reads merged.
# TYPE node_disk_reads_merged_total counter
node_disk_reads_merged_total{device="mmcblk0"} 6505
node_disk_reads_merged_total{device="mmcblk0p1"} 3795
node_disk_reads_merged_total{device="mmcblk0p2"} 2710
# HELP node_disk_write_time_seconds_total This is the total number of seconds spent by all writes.
# TYPE node_disk_write_time_seconds_total counter
node_disk_write_time_seconds_total{device="mmcblk0"} 26.967000000000002
node_disk_write_time_seconds_total{device="mmcblk0p1"} 0.008
node_disk_write_time_seconds_total{device="mmcblk0p2"} 26.958000000000002
# HELP node_disk_writes_completed_total The total number of writes completed successfully.
# TYPE node_disk_writes_completed_total counter
node_disk_writes_completed_total{device="mmcblk0"} 1456
node_disk_writes_completed_total{device="mmcblk0p1"} 3
node_disk_writes_completed_total{device="mmcblk0p2"} 1453
# HELP node_disk_writes_merged_total The number of writes merged.
# TYPE node_disk_writes_merged_total counter
node_disk_writes_merged_total{device="mmcblk0"} 2529
node_disk_writes_merged_total{device="mmcblk0p1"} 0
node_disk_writes_merged_total{device="mmcblk0p2"} 2529
# HELP node_disk_written_bytes_total The total number of bytes written successfully.
# TYPE node_disk_written_bytes_total counter
node_disk_written_bytes_total{device="mmcblk0"} 6.9829632e+07
node_disk_written_bytes_total{device="mmcblk0p1"} 5120
node_disk_written_bytes_total{device="mmcblk0p2"} 6.9824512e+07
node_scrape_collector_duration_seconds{collector="diskstats"} 0.001754445
node_scrape_collector_success{collector="diskstats"} 1
```

+ merged的，是说合并所有硬盘后的指标
+ discard是说硬盘的丢包率，也就是说如果丢包率过高，有可能是硬盘本身的介质出现问题

## 5. 网络监控指标

使用我们常见的系统级监控工具监控网络的话，我们可能需要监控下面的指标

+ 进栈和出栈流量
+ IP是否可达
+ 端口是否正常工作
+ HTTP正常返回200

这些简单的指标，使用我们今天例子中的工具基本都可以实现，不过实现方式有些不同。但是，如果我们想要监控一些复杂的指标，比如：访问网站的IP来自于地球上的哪个区域，HTTP的body内是否含有指定字段，后端服务器访问数据库时候能否得到指定的数据这类和业务相关的数据，我们要么使用一些三方插件，要么自定义一些监控脚本，或者干脆更换监控，比如使用jaeger去监控业务连续性。

### 5.1. 系统上查看网络监控指标

这种指标自带的系统工具可以做的不多，主要是网络的包的体量实在太大了，系统默认不会留下详细的通讯包，除非我们手动配置工具。我们使用的工具一般的linux发行版也是不会默认安装的，比如：

+ htop：top的升级版，界面很炫酷，但是占用资源也比较大
+ iostat：可以查看瞬时的网络流量
+ 各种工具：iftop，sar，nload这类工具和htop一样，都是需要额外安装的，且占用系统资源较多。
+ tcpdump：linux上的wireshark，非常好用的命令行抓包工具，当然，监控指标里面不会有的，但是我们定位系统问题的时候还是经常会用到的

### 5.2. zabbix上的网络监控指标

zabbix上的网络监控指标主要是针对网络接口的，也就是ethxxx之类的，而不是针对物理口的

![image-20200809222247116](/pages/keynotes/L4_architect/2_monitoring/pics/6_common_metrics/image-20200809222247116.png)

### 5.3. grafana上的网络监控指标

基于node_export，非常全面，但是由于图片太大，不容易放，所以这里就不展示了

### 5.4. node_exporter上的网络监控指标

node_exporter上的监控指标非常全面，每个网卡都涉及到20多项指标，涵盖了`/sys/class/net/`下面所有的内容，篇幅有限，就不展示了





## 6. 进程监控指标

*进程*（Process）是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。我们监控进程主要是从下面几个维度去监控。

+ 进程是否存活，存活或者不存活，一般用1表示存活，0表示进程不在了
+ 进程占用的内存和CPU资源
+ 各种进程的数量，比如：正在运行的，正在sleep的进程，停止运行的进程，僵尸进程

### 6.1. 系统上查看进程监控指标

两种方式：

+ 通过命令：内核自带，并且我们最常用的命令就是ps（process status）。
+ 通过文件：常见的linux发行版都会把进程作为文件保存在/proc下面，文件名就是进程名

### 6.2. zabbix上的进程监控指标

### 6.3. grafana上的进程监控指标

### 6.4. node_exporter上的进程监控指标

``` bash
# HELP node_procs_blocked Number of processes blocked waiting for I/O to complete.

# TYPE node_procs_blocked gauge

node_procs_blocked 0

# HELP node_procs_running Number of processes in runnable state.

# TYPE node_procs_running gauge

node_procs_running 5

# HELP node_schedstat_running_seconds_total Number of seconds CPU spent running a process.

# HELP node_schedstat_waiting_seconds_total Number of seconds spent by processing waiting for this CPU.

100 65952    0 65952    0     0  4029k      0 --:--:-- --:--:-- --:--:-- 4293k

# HELP node_softnet_processed_total Number of processed packets

# TYPE node_softnet_processed_total counter

node_softnet_processed_total{cpu="0"} 1.047734e+06

node_softnet_processed_total{cpu="1"} 1.314373e+06

node_softnet_processed_total{cpu="2"} 1.268531e+06

node_softnet_processed_total{cpu="3"} 1.957371e+06

# HELP node_softnet_times_squeezed_total Number of times processing packets ran out of quota

# HELP node_vmstat_pgfault /proc/vmstat information field pgfault.

# HELP node_vmstat_pgmajfault /proc/vmstat information field pgmajfault.

# HELP node_vmstat_pgpgin /proc/vmstat information field pgpgin.

# HELP node_vmstat_pgpgout /proc/vmstat information field pgpgout.

# HELP node_vmstat_pswpin /proc/vmstat information field pswpin.

# HELP node_vmstat_pswpout /proc/vmstat information field pswpout.

# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.

# TYPE process_cpu_seconds_total counter

process_cpu_seconds_total 0.24

# HELP process_max_fds Maximum number of open file descriptors.

# TYPE process_max_fds gauge

process_max_fds 1024

# HELP process_open_fds Number of open file descriptors.

# TYPE process_open_fds gauge

process_open_fds 9

# HELP process_resident_memory_bytes Resident memory size in bytes.

# TYPE process_resident_memory_bytes gauge

process_resident_memory_bytes 1.3606912e+07

# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.

# TYPE process_start_time_seconds gauge

process_start_time_seconds 1.59698063249e+09

# HELP process_virtual_memory_bytes Virtual memory size in bytes.

# TYPE process_virtual_memory_bytes gauge

process_virtual_memory_bytes 7.35105024e+08

# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.

# TYPE process_virtual_memory_max_bytes gauge

process_virtual_memory_max_bytes -1
```

