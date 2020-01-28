---
title: keynotes_L1_basic_gitpage_blog
tags: [formatting]
keywords: gitpage
last_updated: July 3, 2016
sidebar: keynotes_L1_basic_sidebar
permalink: keynotes_L1_basic_gitpage_blog.html
typora-root-url: ../../../../cloudnative365.github.io
---

# 使用GitPages和Jekyll搭建免费的个人博客



## 1. 简介

### 1.1. gitpages

+ gitpages是github的一项服务，可以让github用户使用自己的github名字作为域名前缀，创建自己的个人博客。

  比如，我的Github名字是github.com/zhangke0516，那么，我的个人主页就是zhangke0516.github.io。

+ 完全免费，如果只是为了上传静态内容，或者作为个人主页/博客使用，gitpages完全可以满足需求，而且可以自己编写HTML，JS，CSS，实现完全定制

详细介绍请看官网网站：https://pages.github.com/

### 1.2. Jekyll

+ 如果各位不是有特殊需求，使用Jekyll生成网站框架，并且利用现有的模板修修改改，足够美观且实用。如果各位有前端基础的话，可以加入自己的创意。
+ Jekyll是一个框架，好像django一样，需要从网络上下载别人的模板，然后使用Jekyll来读取模板中的内容来启动网页服务。
+ Jekyll是由Ruby写成的，所以在安装的时候，最重要的三个组件就是ruby，gem和bundle，ruby是编程语言，gem是ruby的包管理器，bundle是可以通过读取Lockfile来自动加载包的工具。所以三者是依赖关系，bundle依赖gem，gem依赖ruby。如果有兴趣的同学可以自行搜索深入学习，碰到其他ruby框架的程序或者碰到ruby框架的问题的时候会更加快速的解决问题

详细介绍看这个：http://jekyllcn.com/

## 2. 申请gitpages

### 2.1. 申请github

#### 2.1.1. 打开[github](https://github.com/)网站，点击sign up

![image-20200120171618791](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120171618791.png)

#### 2.1.2. 填写必要信息，注意密码规则

![image-20200120171848891](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120171848891.png)

#### 2.1.3. 点击验证

![image-20200120172033018](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120172033018.png)

#### 2.1.4. 验证你不是机器人

![image-20200120172302885](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120172302885.png)

#### 2.1.5. 点击`Next`

![image-20200120172553917](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120172553917.png)

#### 2.1.6. `free`的就够了

![image-20200120172709299](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120172709299.png)

#### 2.1.7. 随便填就好了，点击`Compete setup`

![image-20200120172851663](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120172851663.png)

#### 2.1.8. 需要邮箱验证

![image-20200120211833758](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120211833758.png)

#### 2.1.9. 打开邮箱，点击`Verify`

![image-20200120173423338](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120173423338.png)

#### 2.1.10. 第一次登陆有可能会要验证码

![image-20200120190042050](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120190042050.png)

#### 2.1.11. 这个是验证码

![image-20200120190134028](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120190134028.png)

### 2.2. 申请Gitpages

#### 2.2.1. 点击主页右上角，创建一个新的repository

![image-20200120190459549](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120190459549.png)

#### 2.2.2. repo的名字一定要是自己的账户.github.io，比如CloudNative365.github.io

![image-20200120191130998](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120191130998.png)

#### 2.2.3. 随便在仓库里面写点东西（没有东西的话，后面的选项是看不见的），可以按照官方指导来做，或者直接点击`creating a new file`编辑一个文件，这里我就不赘述了

![image-20200120191720808](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120191720808.png)

#### 2.2.4. 点击仓库，打开setting选项卡

![image-20200120193708446](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120193708446.png)

#### 2.2.5. 这个时候发现系统已经为我们配置好了（这个Theme Chooser不用管，点开也是提供几个链接而已，他提供的模板实在太少了，我们后面会介绍其他的模板）

![image-20200120193927284](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120193927284.png)

#### 2.2.6. 访问一下试试，虽然是空的（囧）

 ![image-20200120194024145](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120194024145.png)



## 3. jekyll安装

### 3.1. 在本地安装调试环境

#### 3.1.1. Windows环境下

+ 到[RubyInstaller](https://rubyinstaller.org/)网站下载Ruby

  ![image-20200120201859184](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120201859184.png)

+ 选他推荐的就好，一般来说目前系统都是64位，32位的同学注意下选32位就好

  ![image-20200120202300309](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120202300309.png)

  + 默认安装就好，但是要记得安装目录

    ![image-20200120202433865](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120202433865.png)

  + 目前的安装程序安装完成后都会默认配置，所以不用手动配置，打开CMD测试一下gem是否好用

    ![image-20200120203515111](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120203515111.png)

  + 可以输出帮助信息就算是成功了

    ![image-20200120203603725](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120203603725.png)

  + 执行`gem install jekyll bundler`

    ![image-20200120203700749](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120203700749.png)

  + 没有报错就是成功了

    ![image-20200120203743087](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120203743087.png)

  + 创建一个测试的站点，`jekyll new sitedemo`

    ![image-20200120204116774](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120204116774.png)

  + 启动站点`jekyll server`

    ![image-20200120204205835](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120204205835.png)

  + 在浏览器中输入http://127.0.0.1:4000 测试一下

    ![image-20200120204351726](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120204351726.png)

#### 3.1.2. MacOS环境下

+ 安装brew

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

+ 安装ruby，gem，jekyll和bundler

```bash
$ brew install ruby
$ brew install gem
$ gem install jekyll bundler
```

+ 创建项目和windows相同，不再赘述

### 3.2. jekyll-theme

### 3.2.1. 从[这里](http://jekyllthemes.org/)或者[这里](http://themes.jekyllrc.org/)找一个模板（有兴趣可以自己google，网上的模板多得是，这里推荐的不是官方的网站），我选择的是这个[模板](https://idratherbewriting.com/documentation-theme-jekyll/)，效果是这样的。

![image-20200120200139956](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120200139956.png)

#### 3.2.2. 下载这个[git仓库](https://github.com/tomjoht/documentation-theme-jekyll)的zip包

![image-20200120200602088](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120200602088.png)

#### 3.2.3. 解压文件夹到桌面

![image-20200120205526757](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120205526757.png)

#### 3.2.4. 下载刚才创建的gitpage仓库

![image-20200120205415527](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120205415527.png)

#### 3.2.5. 复制`documentation-theme-jekyll-gh-pages`的内容到本地的`CloudNative365.github.io`文件夹

![image-20200120210319364](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120210319364.png)

#### 3.2.6. 使用git命令吧改动推送到仓库

```
git add .
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git commit -m "add blog"
git push
```

#### 3.2.7. 可以看到页面了

![image-20200120211040779](/pages/keynotes/L1_basic/pics/keynotes_L1_basic_gitpage_blog/image-20200120211040779.png)

## 4. 认识这个模板的常用功能

### 4.1. 删除没有用的文件

+ `docker-compose.yml` 这是用来创建docker-compose的
+

## 5. 对模板进行个性化定制

### 4.1.
