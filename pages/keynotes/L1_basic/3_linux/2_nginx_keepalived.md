---
title: nginx和keepalived实现软负载均衡
keywords: keynotes, advanced, kubernetes_tools, nginx_keepalived
permalink: keynotes_L1_basic_3_linux_2_nginx_keepalived.html
sidebar: keynotes_L1_basic_sidebar
typora-copy-images-to: ./pics/2_nginx_keepalived
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 架构

### 1.1. 系统环境

| IP          | VIP         | 操作系统  | 版本                                                      |
| ----------- | ----------- | --------- | --------------------------------------------------------- |
| 10.114.2.72 | 10.114.2.67 | centos7.9 | keepalived-1.3.5-19.el7.x86_64，nginx-1.16.1-2.el7.x86_64 |
| 10.114.2.73 | 10.114.2.74 | centos7.9 | keepalived-1.3.5-19.el7.x86_64，nginx-1.16.1-2.el7.x86_64 |

## 2.安装

### 2.1. 准备工作

+ 配置epel源

  ``` bash
  cd /etc/yum.repo.d/ && wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
  ```

+ 安装

  ```bash
  yum install -y nginx keepalived
  ```

### 2.2. 配置keepalived

+ 在10.114.2.72上修改`/etc/keepalived/keepalived.conf`

  ``` bash
  global_defs {
     notification_email {
       acassen@firewall.loc
       failover@firewall.loc
       sysadmin@firewall.loc
     }
     notification_email_from Alexandre.Cassen@firewall.loc
     smtp_server 192.168.200.1
     smtp_connect_timeout 30
     router_id LVS_DEVEL
     vrrp_skip_check_adv_addr
     vrrp_strict
     vrrp_garp_interval 0
     vrrp_gna_interval 0
  }
  
  vrrp_script check_nginx {
  script "killall -0 nginx"
          interval 1
          weight 21
  }
  
  vrrp_script chk_mantaince_down {
     script "[[ -f /etc/keepalived/down ]] && exit 1 || exit 0"
     interval 1
     weight 2
  }
  
  vrrp_instance VI_1 {
      state MASTER
      interface eth0
      virtual_router_id 1
      garp_master_delay 1
      mcast_src_ip 10.114.2.72
      lvs_sync_daemon_interface eth0
      priority 110
      advert_int 2
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      track_interface {
          eth0
      }
      virtual_ipaddress {
          10.114.2.67/26 dev eth0 label eth0:1
      }
      track_script {
      check_nginx
      chk_mantaince_down
      }
  }
  
  vrrp_instance VI_2 {
      state BACKUP
      interface eth0
      virtual_router_id 2
      garp_master_delay 1
      mcast_src_ip 10.114.2.72
      lvs_sync_daemon_interface eth0
      priority 100
      advert_int 2
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      track_interface {
          eth0
      }
      virtual_ipaddress {
          10.114.2.74/26 dev eth0 label eth0:2
      }
      track_script {
      check_nginx
      chk_mantaince_down
      }
  }
  ```

+ 在10.114.2.73上修改`/etc/keepalived/keepalived.conf`

  ``` bash
  global_defs {
     notification_email {
       acassen@firewall.loc
       failover@firewall.loc
       sysadmin@firewall.loc
     }
     notification_email_from Alexandre.Cassen@firewall.loc
     smtp_server 192.168.200.1
     smtp_connect_timeout 30
     router_id LVS_DEVEL
     vrrp_skip_check_adv_addr
     vrrp_strict
     vrrp_garp_interval 0
     vrrp_gna_interval 0
  }
  
  vrrp_script check_nginx {
  script "killall -0 nginx"
          interval 1
          weight 21
  }
  
  vrrp_script chk_mantaince_down {
     script "[[ -f /etc/keepalived/down ]] && exit 1 || exit 0"
     interval 1
     weight 2
  }
  
  vrrp_instance VI_1 {
      state BACKUP
      interface eth0
      virtual_router_id 1
      garp_master_delay 1
      mcast_src_ip 10.114.2.73
      lvs_sync_daemon_interface eth0
      priority 100
      advert_int 2
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      track_interface {
          eth0
      }
      virtual_ipaddress {
          10.114.2.67/26 dev eth0 label eth0:1
      }
      track_script {
      check_nginx
      chk_mantaince_down
      }
  }
  
  vrrp_instance VI_2 {
      state MASTER
      interface eth0
      virtual_router_id 2
      garp_master_delay 1
      mcast_src_ip 10.114.2.73
      lvs_sync_daemon_interface eth0
      priority 110
      advert_int 2
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      track_interface {
          eth0
      }
      virtual_ipaddress {
          10.114.2.74/26 dev eth0 label eth0:2
      }
      track_script {
      check_nginx
      chk_mantaince_down
      }
  }
  ```

### 2.3. 配置nginx

假设我们要代理tcp的3000端口

+ 我们需要在nginx.conf配置文件中添加stream段，加配置如下

  ``` bash
  stream {
      log_format main '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
      access_log /var/log/nginx/tcp-access.log main;
      include /etc/nginx/conf.d/tcp.d/*.conf;
  }
  ```

+ 然后在/etc/nginx/conf.d/tcp.d/*.conf中写我们的真正配置

  ``` bash
  upstream grafana-server {
          server 172.16.0.1:3000;
          server 172.16.0.2:3000;
  }
  server {
          listen 0.0.0.0:3000;
          proxy_pass grafana-server;
  }
  ```

  

