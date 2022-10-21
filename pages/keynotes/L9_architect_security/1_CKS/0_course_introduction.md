---
title: CKS考试心得
keywords: keynotes, advanced, kubernetes, CKS, CKS_CERTIFICATE_GUIID
permalink: keynotes_L9_architect_security_1_kubernetes_1_CKS_1_CKS_CERTIFICATE_GUIID.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/1_CKS_CERTIFICATE_GUIID
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. CKS考试介绍



## 2. 报名

## 2.1. 折扣

目前考CKS只有去LF官方去考，中国的LF大学是没有这个考试的。建议大家如果不是非常着急去考试的话，可以在每年的黑五前后来买培训和考试，这个时候会有一个**Bundle**，原价$499的培训加认证只需要$199。

+ 有的时候，LF官方并不会叫黑五打折活动，会叫syber monday，在黑五之后的一周半都会持续打折，大家没必要等到活动刚出来就抢购，时间是足够的。另外，打折是按照美国时间来的，由于美国时间比咱们的时候错后一些，所以千万不要等到12点去抢购。比如，2020年的cybr monday是美国时间11月30号，单是核算到北京时间是12月1号的下午4点开始。而且LF没有限时抢购一说，也不会在12月1号下午4点准时更改页面，这样就更没有必要去等着刚出来就去抢购了

  ![image-20201206100314232](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206100314232.png)

+ 如果是一种认证加培训，比如CKS和CKS培训，是$199，也就是4折（60%off），而如果同时购买两种认证加培训，比如CKS+CKA+CKS培训+CKA培训是$349，也就是3.5折（65%off）。如果是在校生，时间充裕，可以多选两门课程，比如linux认证，nodejs认证之类，都是含金量很高的。CKAD的课程其实并不是很受欢迎，认可度也不高，也不会限制大家考CKS，所以个人不是很建议大家先考这个。

  ![image-20201206100817228](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206100817228.png)

  ![image-20201206100847890](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206100847890.png)

+ 即使你只想考一门或者认证一门，也是有折扣的，是6折，40%off

  ![image-20201206101017647](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206101017647.png)

+ 然后，在付款的界面，记得输入打折码`CYBERSAVEXX`，比如我购买的CKS和CKS培训就是**CYBERSAVE60**，点一下apply，右边的价格才会从499到199

  ![image-20201206101546660](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206101546660.png)

+ 最后就是信用卡信息了，支付完成再进入portal

  ![image-20201206101757891](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206101757891.png)

感觉我买的太早了，课程还没上线。。。

### 2.2. 准备工作

点开考试的页面，会出现下面的准备界面

![image-20201206102009529](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201206102009529.png)

大部分的checklist项目直接点开后面的说明就好，有两个地方需要注意

+ verify name，一定要和护照上的名字一样，如果使用的是身份证，需要去公安局公证一下，换成英文名字，名字要写成汉语拼音

+ Check System Requirements，考试需要使用Chrome浏览器，并且安装一个插件，插件地址在检查的时候会有连接，如果上面图中的连接没有跳转到选择考试的页面，我们可以用下面的[连接](https://www.examslocal.com/ScheduleExam/Home/CompatibilityCheck)直接跳

  ![image-20201207103943731](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201207103943731.png)

  上面是[下载](https://chrome.google.com/webstore/detail/innovative-exams-screensh/dkbjhjljfaagngbdhomnlcheiiangfle)地址，地址会连接到google商店，需要科学上网

+ 下面的红叉子表示我的网速不够，因为我用的是科学上网，所以上传下载带宽不够。大家如果使用的是联通网络，可以不用科学上网，如果不是联通。。。建议换联通，或者用一个非常高速的vpn。电信和移动在考试的时候基本是GG的，根本不行。

+ 最后需要允许第三方的cookie，chrome://settings/content

  ![image-20201207104613836](/pages/keynotes/L2_advanced/4_CKS/pics/1_CKS_CERTIFICATE_GUIID/image-20201207104613836.png)

## 3. 备考

### 3.1. 实验环境

官方给出了三种方案，AWS，GCE或者自建虚拟机。考虑到国内的环境问题，我说两套环境的方案。