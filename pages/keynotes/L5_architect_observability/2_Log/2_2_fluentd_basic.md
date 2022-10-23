---
title: fluentd基础配置
keywords: keynotes, architect, observability, log, fluentd, fluentd_basic
permalink: keynotes_L5_architect_observability_2_log_2_2_fluentd_basic.html
sidebar: keynotes_L5_architect_observability_sidebar
typora-copy-images-to: ./pics/2_2_fluentd_basic
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. fluentd基础配置

fluentd的主体是由C++语言开发的，而扩展功能都是由Ruby语言开发的。我们把这些功能都叫做插件。插件可以扩展fluentd的功能，从数据源到过滤器都可以扩展，比如让fluentd支持某些特定的数据库，让fluentd按照某些特定的方式来格式化数据。

### 1.1. 配置文件组成

我们先来看一下一个基础的配置文件，一般来说，由两个个部分组成，比如

``` bash
<source>
  @type tail
  path /var/log/containerd/log.json
</source>

<match **>
  @type stdout
</match>
```

这个配置文件是说，数据从source进，输出到match部分，@type是数据源的类型，输入数据源放在<source></source>段，输出数据源放在<match></match>段。



在整个数据流中，我们可以在输入和输出结构之间加入<filter></filter>。比如

``` bash
<source>
  @type tail
  path /var/log/containerd/log.json
</source>

<filter **>
  @type grep
  <regexp>
    key message
    pattern /cool/
  </regexp>
</filter>

<match **>
  @type stdout
</match>
```

这个配置文件中的filter是使用grep模块，把message带cool的记录输出出来。



我们对于全局的配置是放在<system></system>当中的，比如

``` bash
<system>
  log_level info
</system>

<source>
  @type tail
  path /var/log/containerd/log.json
</source>

<filter **>
  @type grep
  <regexp>
    key message
    pattern /cool/
  </regexp>
</filter>

<match **>
  @type stdout
</match>
```

这个配置log_level是说日志的级别，一般都是info，我们也可以改成debug来看看到底有什么问题

### 1.2. 代码段

我们看到，整个的配置文件是由多个代码段组成的，代码段由尖括号包起来的部分开始，到尖括号包起来的，带/的关键字结束，比如<source></source>，我们后面为了方便，在讲解中，都使用<关键字>带代表某个代码块。而代码块中也可以夹着代码块，比如

``` bash
<filter **>
  @type grep
  <regexp>
    key message
    pattern /cool/
  </regexp>
</filter>
```

每个功能都会被抽象成一个代码块，对于官方提供的代码块，我们可以在官方文档找到，比如[grep](https://docs.fluentd.org/filter/grep)

![image-20221023170257221](/pages/keynotes/L5_architect_observability/2_Log/pics/2_2_fluentd_basic/image-20221023170257221.png)

文档开始都会说一下这个模块可以用在哪里，grep模块就是用在filter中的，然后下面是一些例子

![image-20221023170426227](/pages/keynotes/L5_architect_observability/2_Log/pics/2_2_fluentd_basic/image-20221023170426227.png)

最后会有对于每个配置的详细说明，比如：是不是必须的

## 2. 插件

官方文档对于插件分类主要有input，output，filter，而在这三种代码块中可以嵌入parser，formater，buffer，storage，service discovery，metrics等。而最主要的就是parser，formater，buffer三种。

### 2.1. 管理插件

管理插件需要登录到fluentd所在的服务器上，使用gem来安装，gem是ruby语言的包管理器，有点类似于yum或者pip。我们一般可以在安装fluentd的过程中找到他。比如，如果使用的是yum安装，我们就可以在安装的文件中找到他。

``` bash
# 在安装完td-agent之后，可以使用下面的命令查询到底安装了什么东西
rpm -ql td-agent
# 就可以找到两个执行文件
/usr/sbin/td-agent
/usr/sbin/td-agent-gem
```

一般来说/usr/sbin是我们默认的$PATH，我们就可以直接使用td-agent-gem了

``` bash
# td-agent-gem
RubyGems is a sophisticated package manager for Ruby.  This is a
basic help message containing pointers to more information.

  Usage:
    gem -h/--help
    gem -v/--version
    gem command [arguments...] [options...]

  Examples:
    gem install rake
    gem list --local
    gem build package.gemspec
    gem help install

  Further help:
    gem help commands            list all 'gem' commands
    gem help examples            show some examples of usage
    gem help gem_dependencies    gem dependencies file guide
    gem help platforms           gem platforms guide
    gem help <COMMAND>           show help on COMMAND
                                   (e.g. 'gem help install')
    gem server                   present a web page at
                                 http://localhost:8808/
                                 with info about installed gems
  Further information:
    https://guides.rubygems.org
```

我们常用的有

+ 安装（由于插件的版本之间有差异，建议大家在安装的时候指定版本，后面我们就会遇到这种情况）

  ``` bash
  $ td-agent-gem install xxx -v 1.15
  ```

+ 列出插件

  ``` bash
  $ td-agent-gem list
  *** LOCAL GEMS ***
  
  addressable (2.8.1)
  async (1.30.3)
  async-http (0.56.6)
  async-io (1.33.0)
  async-pool (0.3.11)
  aws-eventstream (1.2.0)
  aws-partitions (1.609.0)
  aws-sdk-core (3.131.3)
  aws-sdk-kms (1.58.0)
  ```

+ 删除插件

  ``` bash
  td-agent-gem uninstall xxx
  ```

### 2.2. 数据源插件

数据源插件是指input和output插件，这类基本是和我们数据的输入和输出有关，由于fluentd的插件都是ruby开发的，所以对于兼容性上有一个潜规则，那就是需要ruby可以支持这类的数据源。如果我们遇到一些比较古老的数据源，fluentd本身不支持，我们可以使用比较旧的fluentd来试一试。

官方提供的插件只包含了比较比较常用的插件，比如tail，elasticsearch插件

