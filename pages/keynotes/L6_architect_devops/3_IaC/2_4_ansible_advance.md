---
title: ansible进阶
keywords: keynotes, architect, devops, ansible_advance
permalink: keynotes_L6_architect_devops_3_IaC_2_2_ansible_advance.html
sidebar: keynotes_L6_architect_devops
typora-copy-images-to: ./pics/2_2_ansible_basic
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

这一节，我们列出几个比较常见的场景，然后通过一步步的增加功能来实现我们的需求

## 2. 文件场景

## 3. 服务管理场景

## 4. 用户管理场景

### 4.1. 场景一

+ 场景：我们有一台ansible的服务端，原来ansible是通过root用户直接ssh到被ansible管理的机器上的，但是现在，由于安全的要求，root用户被禁止直接登录，我们需要通过ansible服务器上的ansible-server，ssh到目标机器上的普通用户ansible-client来登录，并且给他sudo的权限，让他可以登录到机器上，并且可以sudo到root用户

+ 思考：

  + 对于普通ansible-client用户登录机器，我们需要创建ansible服务器上ansible-server用户和被管控机器ansible-client用户的互信，互信一般都是把ansible-server用户的ssh_key，远程copy到ansible-client所在机器上，ansible-client用户的家目录下的.ssh/authorized_keys文件中来实现的

    ``` bash
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDJqXViSD94eWVRSMkTZnEwdcqFuEPiHMFmhfTacOQySzcLlSC73rDNFW87dVs2Xf9Bc8+6j+409ed+Uw/ZUElUC93tsJFS1Z9wUFOeJUBkUKlo9a/eEgiUQsc/OxZz2P2ldmxwmjUJFOdlKewzVHQMCXJxZflj0PM+IvHvkWwq9apgf+3jZM1/va4vUdsaqT4ZF1itJk8S7pR+CXGsOCD8Pd7Ul1dOvYKh7g+U1ljVpBw5JnjvupXXp8CAYjcKbOrIZyaHuu2pBMdcKMzywMsrfLzPqkxhnOkx+t06q6fFEjifZjuYfXQgOHqnLRHtaFTYQwnnc5hAIKi7NFn1/etL root@ecs-ansible
    ```

    最后的root@ecs-ansible就是可以登录客户端的用户名和机器名

  + 我们需要在ansible服务器上面创建ansible-server用户，并且给他生成一个公钥，把公钥的内容，也就是上面的内容通过文件或者字符串的形式传递给ansible-play，playbook需要先创建用户，然后通过文件模块，生成新的文件

+ 实施：

  + 通过ssh命令生成钥匙对

    ``` bash
    ssh-keygen -t rsa -f ~/.ssh/id_rsa -N "" -C ""
    ```

  + 会生成两个文件

    ``` bash
    tree .ssh
    .ssh
    |-- id_rsa
    `-- id_rsa.pub
    
    0 directories, 2 files
    ```

  + 我们拿到id_rsa.pub的内容
  
    ``` bash
    cat .ssh/id_rsa.pub 
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDEgr6PtFLP+4XQYs1X+g3K0mOs88fqYTFvvvObSRpcWyDS8It7aJrIIV7Xk9V/oJyMb0Ng1GD3ZqSLOa1/KfxfFbrtDFxcDyMsWWlxHvYP7xTosg3ExvK8zwruOtdVGj2h6wlM1fyfpDtodumclNojGyKWMo6kXqOCuOTgdQvokEnn6B/IKFiMckCmvEVzSpcnr9GTTgY+BCDUA6RXNQVrqoV+nhcytAyuuCj+pYQLWDXIGHM6gQvt/p4G1ndd+bykWc9J3HrHqVa5d0FKIM4KwJGA1gyNzQIpF9d3maYqzMgzKmG+QU7F9VF/m5swgtFljd/3sWjO2LiWb0AGhg0N
    ```
  
  + 所以我们的yml文件可以这么写
  
    ``` yaml
    ##
    # 注释：
    ##
     - hosts: all
       pre_tasks:
         - name: Verify Ansible meets the version requirements.
           assert:
             that: "ansible_version.full is version_compare('2.9', '>=')"
             msg: >
               "You must update Ansible to at least version 2.9 to use this role."
    
       vars:
          var_ssh_id_pub: !!str ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDEgr6PtFLP+4XQYs1X+g3K0mOs88fqYTFvvvObSRpcWyDS8It7aJrIIV7Xk9V/oJyMb0Ng1GD3ZqSLOa1/KfxfFbrtDFxcDyMsWWlxHvYP7xTosg3ExvK8zwruOtdVGj2h6wlM1fyfpDtodumclNojGyKWMo6kXqOCuOTgdQvokEnn6B/IKFiMckCmvEVzSpcnr9GTTgY+BCDUA6RXNQVrqoV+nhcytAyuuCj+pYQLWDXIGHM6gQvt/p4G1ndd+bykWc9J3HrHqVa5d0FKIM4KwJGA1gyNzQIpF9d3maYqzMgzKmG+QU7F9VF/m5swgtFljd/3sWjO2LiWb0AGhg0N
          var_ansible_server_user: !!str ansible-server
          var_ansible_client_user: !!str ansible-client
          var_ansible_server_host: !!str ansible-host
          var_sudo_user_group: !!str wheel
    
       tasks:
        - name: user '{{ var_ansible_client_user }}' should be exist
          block:
          
          - name: verify if the user '{{ var_ansible_client_user }}' is exist
            user:
              name: '{{ var_ansible_client_user }}'
              state: present
              comment: 'AnsibleUserForConfigureSystem'
              groups: '{{ var_sudo_user_group }}'
              append: yes # append参数会让groups中的两个组成为用户的附加组
    
          - name: add ssh key for user '{{ var_ansible_client_user }}'
            authorized_key:
              user: '{{ var_ansible_client_user }}'
              state: present
              key: '{{ var_ssh_id_pub }}'
          
          tags:
          - add_user
    
        - name: let '{{ var_sudo_user_group }}' group users have the privilege
          block:
    
          - name: make sure '{{ var_sudo_user_group }}' under /etc/sudoer.d is exist
            file:
              path: /etc/sudoers.d/{{ var_sudo_user_group }}
              state: touch
    
        - name: modify sudoers to allow group "{{ var_sudo_user_group }}" switch to root without password
          copy:
            content: "%{{ var_sudo_user_group }} ALL=(ALL) NOPASSWD: ALL"
            dest: /etc/sudoers.d/{{ var_sudo_user_group }}
            mode: 0440
    
          tags:
          - grant_sudo_root_privilege
    ```
    
    
