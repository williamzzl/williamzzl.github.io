---
layout: post
title:  "Kubernetes + Dashboard"
date:   2015-05-17 08:42:25
categories: kubernetes
tags: kubernetes dashboard docker
---

# Kubernetes Dashboard (cockpit)

>Cockpit是当前对Kubernetes原生支持的少有的一个Dashboard。本文略过Fedora 22 的安装步骤


### 在Fedora 22(Ansible)搭建Kubernetes
[Kubernetes原文链接](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/getting-started-guides/fedora/fedora_ansible_config.md)

1. 这里只使用一台机器同时部署cluster master和minions， 

        # 安装依赖包，下载kubernetes ansible code
        [root@103-59 ~]# yum install -y ansible git yum-utils
        [root@103-59 ~]# git clone https://github.com/eparis/kubernetes-ansible.git
        [root@103-59 ~]# cd kubernetes-ansible

2. 修改部署信息，其中minions后面的kube_ip_addr需要是一个新的网段，kubernetes使用这个独有的IP用来管理pods.

        [root@103-59 ~]# more inventory
        [masters]
        10.0.2.15
        
        [etcd]
        10.0.2.15
        
        [minions]
        10.0.2.15 kube_ip_addr=[10.254.0.1]

3. ansible需要与目的机器配置ssh互信：

        [root@103-59 ~]# ssh-keygen -t rsa
        [root@103-59 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub 10.0.2.15

4. 开始安装Kubernetes:

        [root@103-59 ~]# ansible-playbook -i inventory setup.yml

### 安装cockpit dashboard

[Cockpit原文链接](https://github.com/cockpit-project/cockpit/blob/master/HACKING.md)

>Cockpit是一个轻量级的Linux管理界面，提供了一些简单的方式来管理磁盘、查看journal，和停起services。

* 编译安装cockpit

        [root@103-59 ~]# yum install -y nodejs npm
        [root@103-59 ~]# git clone https://github.com/cockpit-project/cockpit.git
        [root@103-59 ~]# cd cockpit/
        #编译
        [root@103-59 ~]# yum-builddep tools/cockpit.spec
        [root@103-59 ~]# mkdir build
        [root@103-59 ~]# cd build/
        [root@103-59 ~]# ../autogen.sh --prefix=/usr --enable-maintainer-mode --enable-debug
        [root@103-59 ~]# make
        [root@103-59 ~]# make install
        [root@103-59 ~]# cp ../src/bridge/cockpit.pam.insecure /etc/pam.d/cockpit
        [root@103-59 ~]# cat ../src/bridge/sshd-reauthorize.pam >> /etc/pam.d/sshd
        #加入启动项
        [root@103-59 ~]# systemctl start cockpit.socket
        [root@103-59 ~]# systemctl daemon-reload

### 运行Guestbook

**参考[Kubernetes实例][1]搭建一个多层redis应用**

然后登陆ccockpit，查看管理刚才建好的pod, service和replication controller:

![cockpit image](/images/cockpit_kubernetes.PNG)

[1]: http://williamzzl.github.io/kubernetes/2015/04/21/Kubernetes%E5%AE%9E%E4%BE%8B.html