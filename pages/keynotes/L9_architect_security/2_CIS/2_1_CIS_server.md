---
title: 主机CIS
keywords: keynotes, architect, security, CIS, CIS_server
permalink: keynotes_L9_architect_security_2_CIS_2_1_CIS_server.html
sidebar: keynotes_L9_architect_security_sidebar
typora-copy-images-to: ./pics/2_1_CIS_server
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 主机CIS

在github上有一个开源的项目https://github.com/ansible-lockdown，这个项目中包含了大部分的linux发行版的CIS标准化的ansible脚本，比如我们常见的RHEL,CENTOS和Ubuntu，甚至连windows也包括了。

我们可以通过git clone的方式把脚本下载到本地

由于是redhat和cis合作的项目，redhat也把这个功能集中在了yum中，我们可以通过yum直接安装ansible和对应的脚本包来实现CIS标准化。

## 2. Ansible-CIS

### 2.1. 说明

### 2.2. 安装和配置

+ 安装ansible和CIS插件

  ``` bash
  yum install -y ansible scap-security-guide
  ```

+ 其他插件：由于ansible是依赖插件的，在安装过程中可能会报错，我们就使用ansible-galaxy命令来安装

  ``` bash
  ansible-galaxy collection install community.general
  ansible-galaxy collection install ansible.posix
  ```
  
+ 配置ssh免密登录

  ``` bash
  [root@ecs-ansible ~]# ssh-keygen
  Generating public/private rsa key pair.
  Enter file in which to save the key (/root/.ssh/id_rsa):
  Enter passphrase (empty for no passphrase):
  Enter same passphrase again:
  Your identification has been saved in /root/.ssh/id_rsa.
  Your public key has been saved in /root/.ssh/id_rsa.pub.
  The key fingerprint is:
  SHA256:Zp3rLoLUj3gpupD5ofIR1wQfKEMUY/HAfkeH4kJMs8U root@ecs-ansible
  The key's randomart image is:
  +---[RSA 2048]----+
  | *@+o...         |
  | .*BEoo..        |
  | o.+.oo.         |
  |  o oo.  . .     |
  |  .o.o. S o      |
  | o o. .o   .     |
  |+ o. o +  .      |
  |.+ o+ = o.       |
  |o.=o o . oo      |
  +----[SHA256]-----+
  [root@ecs-ansible ~]# ssh-copy-id 127.0.0.1
  /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
  The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
  ECDSA key fingerprint is SHA256:g2zBRWwZayAmncnBn2QsfWK5bZto7jRvFMX+fv+CYDU.
  ECDSA key fingerprint is MD5:75:58:a4:87:7b:56:0a:35:2e:0d:cb:b1:b1:c7:76:cd.
  Are you sure you want to continue connecting (yes/no)? yes
  /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
  /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
  root@127.0.0.1's password:
  
  Number of key(s) added: 1
  
  Now try logging into the machine, with:   "ssh '127.0.0.1'"
  and check to make sure that only the key(s) you wanted were added.
  
  [root@ecs-ansible ~]#
  ```

+ 添加host

  ``` bash
  echo 127.0.0.1 >> /etc/ansible/hosts
  ```

+ playbook位置

  ``` bash
  [root@ecs-ansible ansible]# ls -al /usr/share/scap-security-guide/ansible/rhel7-playbook-cis*
  -rw-r--r-- 1 root root 293839 9月  21 21:24 /usr/share/scap-security-guide/ansible/rhel7-playbook-cis_server_l1.yml
  -rw-r--r-- 1 root root 288475 9月  21 21:24 /usr/share/scap-security-guide/ansible/rhel7-playbook-cis_workstation_l1.yml
  -rw-r--r-- 1 root root 820023 9月  21 21:24 /usr/share/scap-security-guide/ansible/rhel7-playbook-cis_workstation_l2.yml
  -rw-r--r-- 1 root root 820820 9月  21 21:24 /usr/share/scap-security-guide/ansible/rhel7-playbook-cis.yml
  ```

### 2.3. 运行基线playbook

+ 在运行之前，我们先来看一下配置文件原来的样子，以ssh登录为例

  ``` bash
  [root@ecs-ansible ansible]# grep -v "^#" /etc/ssh/sshd_config |grep -v "^$"
  HostKey /etc/ssh/ssh_host_rsa_key
  HostKey /etc/ssh/ssh_host_ecdsa_key
  HostKey /etc/ssh/ssh_host_ed25519_key
  SyslogFacility AUTHPRIV
  AuthorizedKeysFile	.ssh/authorized_keys
  ChallengeResponseAuthentication no
  GSSAPIAuthentication yes
  GSSAPICleanupCredentials no
  UsePAM yes
  X11Forwarding yes
  AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES
  AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT
  AcceptEnv LC_IDENTIFICATION LC_ALL LANGUAGE
  AcceptEnv XMODIFIERS
  Subsystem	sftp	/usr/libexec/openssh/sftp-server
  PermitRootLogin yes
  PasswordAuthentication yes
  UseDNS no
  ```

+ 运行playbook

  ``` bash
  cd /usr/share/scap-security-guide/ansible
  ansible-playbook rhel7-playbook-cis_server_l1.yml
  # 最后出现结果
  PLAY RECAP ****************************************************************************************************************************************************************************************************
  127.0.0.1                  : ok=386  changed=92   unreachable=0    failed=0    skipped=155  rescued=0    ignored=4
  ```

+ 这个时候我们再看/etc/ssh/sshd_config文件，就会发现多了很多的东西

  ``` bash
  [root@ecs-ansible ansible]# grep -v "^#" /etc/ssh/sshd_config |grep -v "^$"
  HostKey /etc/ssh/ssh_host_rsa_key
  HostKey /etc/ssh/ssh_host_ecdsa_key
  HostKey /etc/ssh/ssh_host_ed25519_key
  SyslogFacility AUTHPRIV
  AuthorizedKeysFile	.ssh/authorized_keys
  ChallengeResponseAuthentication no
  GSSAPIAuthentication yes
  GSSAPICleanupCredentials no
  UsePAM yes
  X11Forwarding yes
  AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES
  AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT
  AcceptEnv LC_IDENTIFICATION LC_ALL LANGUAGE
  AcceptEnv XMODIFIERS
  Subsystem	sftp	/usr/libexec/openssh/sftp-server
  ClientAliveCountMax 0
  ClientAliveInterval 900
  HostbasedAuthentication no
  PermitEmptyPasswords no
  IgnoreRhosts yes
  PermitUserEnvironment no
  LoginGraceTime 60
  LogLevel VERBOSE
  MaxAuthTries 4
  MaxSessions 10
  MaxStartups 10:30:60
  Ciphers chacha20-poly1305@openssh.com,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com,aes128-cbc,aes192-cbc,aes256-cbc,blowfish-cbc,cast128-cbc,3des-cbc
  MACs umac-64-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-64@openssh.com,umac-128@openssh.com,hmac-sha2-256,hmac-sha2-512,hmac-sha1,hmac-sha1-etm@openssh.com
  PermitRootLogin no
  PasswordAuthentication yes
  UseDNS no
  [root@ecs-ansible ansible]#
  ```

+ 这个时候，我们发现`PermitRootLogin no`了，也就是如果重启sshd，root就登录不进来了，我们就创建一个普通账户，然后给他sudo到root的权限

  ``` bash
  useradd jormun
  password jormun
  # 如果我们输入少于13位的密码就会报错
  # 无效的密码： 密码少于 13 个字符
  # 修改sudoer文件，让jormun可以切换到root
  jormun  ALL=(ALL)       NOPASSWD: ALL
  ```

+ 同时，记得把防火墙和selinux配置改一下，要不重启后会登录不上来

  ``` bash
  systemctl disable firewalld
  ```

+ 我们也可以直接使用下面的命令

  ``` bash
  ansible-playbook -l local --skip-tags package_firewalld,service_firewalld_enabled,package_libselinux_installed,grub2_enable_selinux,selinux_policytype,selinux_state rhel7-playbook-cis_server_l1.yml
  ```
  
  
  
+ 重启机器，之后用jormun用户登录

  ``` bash
  jormun@zhangkedeMacBook-Pro ~ % ssh jormun@114.116.204.62
  jormun@114.116.204.62's password:
  Last login: Sun Oct  2 14:15:50 2022 from 218.68.227.141
  Authorized uses only. All activity may be monitored and reported.[jormun@ecs-ansible ~]$
  ```

## 3. 总结

我们发现，这个其实有很多不完善的地方，比如防火墙和selinux，但是框架都摆在眼前了，我们只需要改就可以了。而我们发现在安装了scap包之后，在`/usr/share/scap-security-guide`目录下面还有集中方式，比如bash和kickstart

![image-20221002142343702](/pages/keynotes/L9_architect_security/2_CIS/pics/2_1_CIS_server/image-20221002142343702.png)

我们也可以通过机器安装完成后，手动去运行脚本或者使用kickstart，在机器初始化的时候就做这些
