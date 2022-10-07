---
title: terraform
keywords: keynotes, architect, devops, terraform
permalink: keynotes_L6_architect_devops_3_IaC_1_terraform.html
sidebar: keynotes_L6_architect_devops
typora-copy-images-to: ./pics/1_terraform
typora-root-url: ../../../../../cloudnative365.github.io


---

## 1. terraform简介

``` 
Terraform is an open-source infrastructure as code software tool that provides a consistent CLI workflow to manage hundreds of cloud services. Terraform codifies cloud APIs into declarative configuration files.
```

terraform是一款开源的IaC工具，他提供了一个连续的命令行工作流来管理成百上千的云资源。Terraform把云的接口代码化为配置文件。

terraform是hashcorp公司的一款IaC工具。他目前提供了三种版本

![image-20220306150941657](/pages/keynotes/L6_architect_devops/3_IaC/pics/1_terraform/image-20220306150941657.png)

免费版就是开源版，开源版的代码可以在github上找到。https://github.com/hashicorp/terraform。这也是我们最常用的版本。

Team & Governance是SaaS服务，也是最近比较流行的一种方式。用SaaS服务的方式向客户开放窗口，客户可以通过直接使用界面的方式来使用产品。

Business是商业版，也是很多公司的选择之一。他们会向Hashcorp公司购买商业授权，然后获得相应的技术支持。

这里要说的是，商业版本相对于免费版本不仅仅是技术上的支持，并且还提供了UI界面，还具备了很多开源版本不具备的功能。对用户更加的友好，如果我们的系统体量非常庞大，并且需要一个纯IaC的环境的话，建议使用商业版本的Terraform。



## 2. 架构

![Terraform deployment workflow](/pages/keynotes/L6_architect_devops/3_IaC/pics/1_terraform/terraform-iac.png)

上面的架构图更像是一张流程图，因为社区版本与其说是一个软件，更像是一个命令，他和其他的云原生软件一样，基本都是使用子命令来区别功能的，比如：

``` bash
terraform plan
terraform apply
```

## 3. 安装

需要安装的版本是开源版，如果我们使用二进制方式的话其实非常简单。https://www.terraform.io/downloads

![image-20220307152335856](/pages/keynotes/L6_architect_devops/3_IaC/pics/1_terraform/image-20220307152335856.png)

安装完成之后就可以使用命令了

``` bash
% terraform version
Terraform v1.1.2
on darwin_arm64

Your version of Terraform is out of date! The latest version
is 1.1.7. You can update by downloading from https://www.terraform.io/downloads.html
```



## 4. Provider

我们可以将provider理解为terraform的插件，每种provider对应一种资源的供应商，比如AWS provider，Vmware provider。他们主要去和他们的API做对接，详细的介绍在这里https://www.terraform.io/language/providers。

prodiver提供了一系列的资源类型或者是数据源，来让Terraform管理。换句话说，如果资源本身并没有提供接口可供调用，理论上terraform是不能管理的，terraform并不能管理所有的资源，提供接口或者任何方式给provider去调用的资源才可以被terraform管理。

我们本次以AWS为例

+ 官方文档：https://registry.terraform.io/providers/hashicorp/aws/latest/docs
+ github：https://github.com/hashicorp/terraform-provider-aws

## 5. demo



