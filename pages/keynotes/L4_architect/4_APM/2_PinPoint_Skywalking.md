---
title: PinPoint VS Skywalking
keywords: keynotes, architect, logging, pinpoint_skywalking
permalink: keynotes_L4_architect_4_storage_2_PinPoint_Skywalking.html
sidebar: keynotes_L4_architect_sidebar
typora-copy-images-to: ./pics/2_PinPoint_Skywalking
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 简介

 ![image-20210120142910050](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/image-20210120142910050.png)

在代码侵入性的APM项目中，PinPoint和skywalking是两个非常优秀的工具了。有需要上APM系统的同学可以看一下我对于这两款软件的比较来做出合理的判断。

## 2. 比较项

![image-20210120143026410](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/image-20210120143026410.png)

截止到2021年01月12日，我们的比较项如下

| 比较项          | PinPoint                                   | Skywalking                                                 |
| --------------- | ------------------------------------------ | ---------------------------------------------------------- |
| 版本            | v2.2.0                                     | v8.3.0                                                     |
| 项目发起人      | Woonduk Kang                               | 吴晟                                                       |
| GitHub星数      | 11.1k                                      | 15.8k                                                      |
| 社区            | 非apache                                   | apache                                                     |
| 文档            | http://skywalking.apache.org/docs/（详细） | https://pinpoint-apm.github.io/pinpoint/main.html（详细）  |
| 用户            | 非常多                                     | 非常多                                                     |
| 兼容OpenTracing | 不支持                                     | 支持                                                       |
| 支持语言        | java,php, C/CPP, python                    | Java, LUA, python, node.js(官方),<br />.NET, php, Go(社区) |
| 协议            | thrift                                     | gRPC                                                       |
| 存储            | HBase（hbase+hadoop+zookeeper）+MySQL      | ES, H2, MySQL, TiDB, Sharding-Sphere                       |
| UI丰富度        | 很高                                       | 很高（8.3版本）                                            |
| 实现方式        | 字节码注入                                 | 字节码注入                                                 |
| 代码侵入性      | 无                                         | 无                                                         |
| 扩展性          | 地                                         | 高                                                         |
| TraceId查询     | 不支持                                     | 支持                                                       |
| 告警            | 支持                                       | 支持                                                       |
| JVM监控         | 支持                                       | 支持                                                       |
| 跟踪粒度        | 细                                         | 一般                                                       |
| 过滤跟踪        | filter配置                                 | agent.config + apm-trace-ignore-plugin                     |
| 性能消耗        | 高                                         | 低                                                         |
| 组件            | collector+web+agent+存储                   | OAP+Web+agent+存储+zk                                      |
| 发布包          | war                                        | jar                                                        |

## 3. 社区比较

**skywalking完胜**

skywalking作为近两年的后起之秀的势头已经完全超越了pinpoint，完全归功于强大的社区。我们可以来看一下git的活跃程度就可以看出来了。 

pinpoint

![image-20210120115943969](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/image-20210120115943969.png)

skywalking

![image-20210120120232597](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/image-20210120120232597.png)

## 4. 支持语言

**skywalking支持的语言更多**

pinpoint支持java,php, C/CPP, python，[官方文档](https://github.com/pinpoint-apm/pinpoint-c-agent)

而skywalking除了官方支持的语言Java, LUA, python, node.js外，社区还贡献了.NET, php, Go的agent，[官方文档](https://github.com/apache/skywalking/blob/8.3.0/docs-hotfix/docs/en/setup/README.md#language-agents-in-service)

## 5. 协议

skywalking支持gRPC和http，但是新版本只支持gRPC了。而pinpoint使用的是thrift协议。协议并没有高低

## 6. 存储

Pinpoint只支持Hbase，也就是说，如果想要扩展HBase就需要用到hadoop和zookeeper，代价比较大

而skywalking支持多种的存储，我们最常用的就是es。

而我们用的最多的功能是查，这个时候es的高速查询能力就体现了出来。但是Hbase的海量存储功能是es没办法比的，所以我们需要根据我们的场景来选择合适的工具。

## 7. UI

早期的skywalking（7.0之前）的UI功能很有限，但是daocloud为他定制了一款UI叫rocketbot，功能非常强大，新版本的skywalking就直接吸收了这个项目，使用他作为UI，弥补了UI方面的缺陷。

![image-20210120133723212](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/image-20210120133723212.png)

而pinpoint的UI同样非常出色，二者不相伯仲。

![Pinpoint](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/ss_server-map.png)

## 8. 扩展性

pinpoint设计的时候就没有考虑过扩展性，而skywalking的核心设计之一就是要设计成pluggable。

+ 存储：skywalking如果要自定义实现一套存储，只需要定义一个类实现接口`org.apache.skywalking.oap.server.library.module.ModuleProvider`，然后实现一些DAO即可

+ 探针：由于skywalking数据接口标准很多，并且支持了openTracing，所以agent的集成其实非常简单，.NET, php, Go这些agent都是由社区贡献的

**扩展性方面skywalking更胜一筹**

## 9. 告警

二者都支持自定告警，但是pinpoint告警还需要mysql的支持，虽然二者的告警维度大致相同，但是skywalking支持webhook的方式，也就是说他可以支持和多种方式的报警，比如短信，邮件，微信。

**skywalking在告警方面比pinpoint更好用**

## 10. 支持组件

+ pinpoint: https://github.com/pinpoint-apm/pinpoint#supported-modules

+ skywalking: https://github.com/apache/skywalking/blob/8.3.0/docs-hotfix/docs/en/setup/service-agent/java-agent/Supported-list.md

![image-20210120141057232](/pages/keynotes/L4_architect/4_APM/pics/2_PinPoint_Skywalking/image-20210120141057232.png)

图是我从别的地方截的，时间比较早了，目前二者支持的东西会更多，但是给人的感觉是skywalking是完全面向互联网的，对于开源的软件支持的比较多，而pinpoint支持一些商业的软件，比如weblogic和websphere。

## 结论

如果是从0构建的话，我觉得还是选择skywalking更好，可能也是我比较倾向于使用开源工具，所以对于skywalking有很大的倾向性，这篇文章抛砖引玉，我后面会多多完善。