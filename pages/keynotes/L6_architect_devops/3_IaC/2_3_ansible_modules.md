---
title: ansible常用模块
keywords: keynotes, architect, devops, ansible_basic
permalink: keynotes_L6_architect_devops_3_IaC_2_3_ansible_modules.html
sidebar: keynotes_L6_architect_devops
typora-copy-images-to: ./pics/2_3_ansible_modules
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

ansible最常用的模块

## 2. 文件操作

## 3. 服务管理

## 4. 用户管理

### 4.1. 说明

用户模块 User，[官方文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html#ansible-collections-ansible-builtin-user-module)，用户模块用来管理操作系统上的用户

### 4.2. 例子

我们来看官方的例子

``` yaml
- name: Add the user 'johnd' with a specific uid and a primary group of 'admin'
  ansible.builtin.user:
    name: johnd
    comment: John Doe
    uid: 1040
    group: admin

- name: Add the user 'james' with a bash shell, appending the group 'admins' and 'developers' to the user's groups
  ansible.builtin.user:
    name: james
    shell: /bin/bash
    groups: admins,developers
    append: yes

- name: Remove the user 'johnd'
  ansible.builtin.user:
    name: johnd
    state: absent
    remove: yes

- name: Create a 2048-bit SSH key for user jsmith in ~jsmith/.ssh/id_rsa
  ansible.builtin.user:
    name: jsmith
    generate_ssh_key: yes
    ssh_key_bits: 2048
    ssh_key_file: .ssh/id_rsa

- name: Added a consultant whose account you want to expire
  ansible.builtin.user:
    name: james18
    shell: /bin/zsh
    groups: developers
    expires: 1422403387

- name: Starting at Ansible 2.6, modify user, remove expiry time
  ansible.builtin.user:
    name: james18
    expires: -1

- name: Set maximum expiration date for password
  ansible.builtin.user:
    name: ram19
    password_expire_max: 10

- name: Set minimum expiration date for password
  ansible.builtin.user:
    name: pushkar15
    password_expire_min: 5
```

### 4.3. 选项

常用
