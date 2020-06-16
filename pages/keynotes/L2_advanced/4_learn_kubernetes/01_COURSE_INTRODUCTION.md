---
title: kubernetes运维都干点啥
keywords: keynotes, L2_advanced, kubernetes, CKA, COURSE_INTRODUCTION
permalink: keynotes_L2_advanced_1_kubernetes_1_CKA_1_COURSE_INTRODUCTION.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/1_COURSE_INTRODUCTION
typora-root-url: ../../../../../cloudnative365.github.io
---

## Course Introduction

### Course Learning Objectives

By the end of this course, you will learn the following:

- The history and evolution of Kubernetes.
- Its high-level architecture and components.
- The API, the most important resources that make the API, and how to use them.
- How to deploy and manage an application.
- Some upcoming features that will boost your productivity.

### Course Audience and Requirements

#### Audience

This course is addressed to Linux administrators or software developers starting to work with containers and wondering how to manage them in production. In this course, you will learn the key principles that will put you on the journey to managing containerized applications in production.

#### Knowledge/Skills

To make the most of this course, you will need the following:

- A good understanding of Linux
- Familiarity with the command line
- Familiarity with package managers
- Familiarity with Git and GitHub
- Access to a Linux server or Linux desktop/laptop
- VirtualBox on your machine, or access to a public cloud.

#### Software Environment

The material produced by The Linux Foundation is distribution-flexible. This means that technical explanations, labs and procedures should work on most modern Linux distributions, and we do not promote products sold by any specific vendor (although we may mention them for specific scenarios).

In practice, most of our material is written with the three main Linux distribution families in mind:

- Debian/Ubuntu
- Red Hat/Fedora
- openSUSE/SUSE. 

Distributions used by our students tend to be one of these three alternatives, or a product that is derived from them.

#### Lab Environment

The lab exercises were written and tested using Ubuntu instances running on Google Cloud Platform. They have been written to be vendor-agnostic, so they could run on AWS, local hardware, or inside of virtual machines, to give you the most flexibility and options. 

Each node has 3 vCPUs and 7.5G of memory, running Ubuntu 18.04. Smaller nodes should work, but you should expect a slow response. Other operating system images are also possible, but there may be a slight difference in some command outputs.

Using GCP requires setting up an account, and will incur expenses if using nodes of the size suggested. For more information review [*Quickstart Using a Linux VM*](https://cloud.google.com/compute/docs/quickstart-linux).

Amazon Web Service (AWS) is another provider of cloud-based nodes, and requires an account; you will incur expenses for nodes of the suggested size. You can find videos and information about [how to launch a Linux virtual machine](https://aws.amazon.com/getting-started/tutorials/launch-a-virtual-machine/) on the AWS website. 

Virtual machines such as KVM, VirtualBox, or VMWare can also be used for the lab systems. Putting the VMs on a private network can make troubleshooting easier. As of Kubernetes v1.16.1, the minimum (as in barely
works) size for VirtualBox is 3vCPU/4G memory/5G minimal OS for the master, and 1vCPU/2G memory/5G minimal OS
for the worker node.

Finally, using bare-metal nodes, with access to the Internet, will also work for the lab exercises.

### Course Resources

Resources for this course can be found online. Making updates to this course takes time. Therefore, if there are any changes in between updates, you can always access course updates, as well as the course resources online:

- Go to the Linux Foundation training website to obtain [Course Resources](https://training.linuxfoundation.org/cm/LFS258/)
- The user ID is **LFtraining** and the password is **Penguin2014**.

### Which Distribution to Choose?

![image-20200420114300960](/pages/keynotes/L2_advanced/1_CKA/pics/1_COURSE_INTRODUCTION/image-20200420114300960.png)

![image-20200420114332961](/pages/keynotes/L2_advanced/1_CKA/pics/1_COURSE_INTRODUCTION/image-20200420114332961.png)

![image-20200420114357743](/pages/keynotes/L2_advanced/1_CKA/pics/1_COURSE_INTRODUCTION/image-20200420114357743.png)

![image-20200420114408534](/pages/keynotes/L2_advanced/1_CKA/pics/1_COURSE_INTRODUCTION/image-20200420114408534.png)

![image-20200420114414429](/pages/keynotes/L2_advanced/1_CKA/pics/1_COURSE_INTRODUCTION/image-20200420114414429.png)

![image-20200420114420805](/pages/keynotes/L2_advanced/1_CKA/pics/1_COURSE_INTRODUCTION/image-20200420114420805.png)

### Using AWS to Set Up Labs

## The Linux Foundation

### The Linux Foundation

[The Linux Foundation](https://www.linuxfoundation.org/) is dedicated to building sustainable ecosystems around open source projects to accelerate technology development and industry adoption. Founded in 2000, the Linux Foundation provided unparalleled support for open source communities through financial and intellectual resources, infrastructure, services, events, and training. Working together, the Linux Foundation and its projects form the most ambitious and successful investment in the creation of shared technology.

Linux is the world's largest and most pervasive open source software project in history. The Linux Foundation is home to the Linux creator Linus Torvalds and lead maintainer Greg Kroah-Hartman, and provides a neutral home where Linux kernel development can be protected and accelerated for years to come. The success of Linux has catalyzed growth in the open source community, demonstrating the commercial efficacy of open source and inspiring countless new projects across all industries and levels of the technology stack.

The Linux Foundation is the umbrella for many critical open source projects that power corporations today, spanning all industry sectors:

- Big data and analytics: [ODPi](https://www.odpi.org/), [R Consortium](https://www.r-consortium.org/)
- Networking: [OpenDaylight](https://www.opendaylight.org/), [OPNFV](https://www.opnfv.org/), [ONAP](https://www.onap.org/)
- Embedded: [Dronecode](https://www.dronecode.org/), [Zephyr](https://www.zephyrproject.org/)
- Web tools: [OpenJS Foundation](https://openjsf.org/)
- Cloud computing: [Cloud Foundry](https://www.cloudfoundry.org/), [Cloud Native Computing Foundation](https://www.cncf.io/), [Open Container Initiative](https://www.opencontainers.org/)
- Automotive: [Automotive Grade Linux](https://www.automotivelinux.org/)
- Security: [The Core Infrastructure Initiative](https://www.coreinfrastructure.org/)
- Blockchain: [Hyperledger](https://www.hyperledger.org/)
- And many more.

### Cloud Native Computing Foundation (CNCF)

[Cloud Native Computing Foundation (CNCF)](https://www.cncf.io/) is an open source software foundation under the Linux Foundation umbrella dedicated to making cloud native computing universal and sustainable. Cloud native computing uses an open source software stack to deploy applications as microservices, packaging each part into its own container, and dynamically orchestrating those containers to optimize resource utilization. Cloud native technologies enable software developers to build great products faster.

CNCF serves as a vendor-neutral home for many of the fastest-growing projects on GitHub, including Kubernetes, Prometheus, and Envoy, fostering collaboration between the industry's top developers, end users and vendors.

## The Linux Foundation Events

The Linux Foundation produces technical events around the world. Whether it is to provide an open forum for development of the next kernel release, to bring together developers to solve problems in a real-time environment, to host workgroups and community groups for active discussions, to connect end users and kernel developers in order to grow Linux and open source software use in the enterprise or to encourage collaboration among the entire community, we know that our conferences provide an atmosphere that is unmatched in their ability to further the platform.

The Linux Foundation hosts an increasing number of events each year, including:

- Open Source Summit North America, Europe, Japan, and China
- Embedded Linux Conference + OpenIoT Summit North America and Europe
- Open Source Leadership Summit
- Open Networking Summit North America and Europe
- KubeCon + CloudNativeCon North America, Europe, and China
- Automotive Linux Summit
- KVM Forum
- Linux Storage Filesystem and Memory Management Summit
- Linux Security Summit North America and Europe
- Cloud Foundry Summit
- Hyperledger Global Forum, etc.

You can learn more about the [Linux Foundation events](https://events.linuxfoundation.org/) online.



