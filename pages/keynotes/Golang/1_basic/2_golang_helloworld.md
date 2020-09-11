---
title: golang环境配置
keywords: keynotes, golang, golang_basic, golang_hellowworld
permalink: keynotes_golang_1_golang_basic_1_golang_hellowworld.html
sidebar: keynotes_golang_sidebar
typora-copy-images-to: ./pics/1_golang_hellowworld
typora-root-url: ../../../../../cloudnative365.github.io

---

## 1. Golang安装与配置

### 1.1. Golang下载

国内是没办法直接下载golang的，但是有爱好者成了专门的通道让我们可以[下载](https://studygolang.com/dl)，

#### 1.1.1. MacOS环境

+ 下载pkg包，下一步下一步安装即可

+ 打开terminal输入go命令进行测试

+ go被安装在/usr/local/go/bin/go下面

+ 配置环境变量：

  + PATH：安装的时候已经默认配置好了

  + GOROOT:

    ``` bash
    export GOROOT=/usr/local/go # 这是go编译环境的路径
    ```

  + GOPATH:

    ``` bash
    export GOPATH=/usr/local/go # 这是go工程项目的位置
    ```

    

#### 1.1.2. Linux环境

#### 1.1.3. Windows环境

### 1.2. 环境变量

## 2. IDE

常用的IDE有Jet Brain公司的Goland（收费的），微软的产品vscode（免费的），或者大神级别的vim。我们这里使用的是免费的IDE，vscode

### 2.1. 下载与安装

下载[地址](https://code.visualstudio.com/)，同样有三个版本，windows，mac和linux，使用对应方式安装即可，Linux建议使用ubuntu，debian或者Centos这种支持比较好的发行版，使用其他版本有可能会有图形界面的兼容性问题

### 2.2. 安装插件

在左边找到插件按钮，搜索相关插件

+ chinese：中文插件
+ go：go插件