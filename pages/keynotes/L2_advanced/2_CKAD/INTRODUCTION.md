---
title: INTRODUCTION
keywords: keynotes, L2_advanced, kubernetes, CKAD, INTRODUCTION
permalink: keynotes_L2_advanced_1_kubernetes_2_CKAD_1_INTRODUCTION.html
sidebar: keynotes_L2_advanced_sidebar
typora-copy-images-to: ./pics/1_INTRODUCTION
typora-root-url: ../../../../../cloudnative365.github.io

---

## Course Information

### Course Learning Objectives

By the end of this course, you should be able to:

- Containerize and deploy a new Python script.
- Configure the deployment with ConfigMaps, Secrets, and SecurityContexts.
- Understand multi-container Pod design.
- Configure probes for Pod health.
- Update and roll back an application.
- Implement services and NetworkPolicies.
- Use PersistentVolumeClaims for state persistence.

### Course Audience and Requirements

+ Course Audience

  This course is for developers looking to learn how to deploy, configure, and test their containerized applications on a multi-node Kubernetes cluster.

+ Knowledge/Skills Requirements

  For a successful learning experience, basic Linux command line and file editing skills are required. Familiarity with using a programming language (such as Python, Node.js, Go) and Cloud Native application concepts and architectures is helpful.

  Our free [LFS158x - Introduction to Kubernetes](https://www.edx.org/course/introduction-to-kubernetes) MOOC on edX is a useful preparation for this course.

+ Software Environment

  The material produced by The Linux Foundation is distribution-flexible. This means that technical explanations, labs and procedures should work on most modern distributions, and we do not promote products sold by any specific vendor (although we may mention them for specific scenarios).

  In practice, most of our material is written with the three main Linux distribution families in mind: 

  \- Debian/Ubuntu

  \- Red Hat/Fedora

  \- openSUSE/SUSE. 

  Distributions used by our students tend to be one of these three alternatives, or a product that is derived from them.

+ Lab Environment

  The lab exercises were written using Google Compute Engine (GCE) nodes. They have been written to be vendor-agnostic, so they could run on AWS, local hardware, or inside of virtual machines, to give you the most flexibility and options.

  Each node has 2 vCPUs and 8G of memory, running Ubuntu 18.04. Smaller nodes should work, but you should expect a slow response. Other operating system images are also possible, but there may be a slight difference in some command outputs.

  Using GCE requires setting up an account, and will incur expenses if using nodes of the size suggested. For more information review ["Quickstart Using a Linux VM"](https://cloud.google.com/compute/docs/quickstart-linux).

  Amazon Web Service (AWS) is another provider of cloud-based nodes, and requires an account; you will incur expenses for nodes of the suggested size. You can find videos and information about [how to launch a Linux virtual machine](https://aws.amazon.com/getting-started/tutorials/launch-a-virtual-machine/) on the AWS website. 

  Virtual machines such as KVM, VirtualBox, or VMWare can also be used for the lab systems. Putting the VMs on a private network can make troubleshooting easier.

  Finally, using bare-metal nodes, with access to the Internet, will also work for the lab exercises.

### Course Resources

Resources for this course can be found online. Making updates to this course takes time. Therefore, if there are any changes in between updates, you can always access course updates, as well as the course resources online:

- Go to the Linux Foundation training website to obtain [Course Resources](https://training.linuxfoundation.org/cm/LFD259/)
- The user ID is **LFtraining** and the password is **Penguin2014**. 

### Which Distribution to Choose?

![image-20200425110009299](/pages/keynotes/L2_advanced/2_CKAD/pics/1_INTRODUCTION/image-20200425110009299.png)

![image-20200425110035118](/pages/keynotes/L2_advanced/2_CKAD/pics/1_INTRODUCTION/image-20200425110035118.png)

![image-20200425110119137](/pages/keynotes/L2_advanced/2_CKAD/pics/1_INTRODUCTION/image-20200425110119137.png)

![image-20200425110136998](/pages/keynotes/L2_advanced/2_CKAD/pics/1_INTRODUCTION/image-20200425110136998.png)

![image-20200425110152962](/pages/keynotes/L2_advanced/2_CKAD/pics/1_INTRODUCTION/image-20200425110152962.png)

![image-20200425110209810](/pages/keynotes/L2_advanced/2_CKAD/pics/1_INTRODUCTION/image-20200425110209810.png)

### IMPORTANT: Using AWS to Set Up Labs

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