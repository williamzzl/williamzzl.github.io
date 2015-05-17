---
layout: post
title:  "Kubernetes概述"
date:   2015-04-11 12:34:25
categories: kubernetes
tags: kubernetes docker
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

#Kubernetes 概述

- [概述](#概述)
- [核心概念](#核心概念)
  - [Pods](#Pods)
  - [Lables](#Lables)
- [Kubernetes节点](#Kubernetes节点)
  - [Kubelet](#Kubelet)
  - [Kubernetes代理](#Kubernetes代理)
- [Kubernetes控制层](#Kubernetes控制层)
  - [etcd](#etcd)
  - [Kubernetes API Server](#Kubernetes API Server)
  - [Kubernetes Controller Manager Server](#Kubernetes Controller Manager Server)

## 概述
Kubernetes是一个用来管理跨主机的容器应用的系统，提供一个基础的框架来对应用做部署、维护和扩容。它的API展示了它可以被用做开放工具集，自动化系统和更高层API封装的基础。

Kubernetes使用[Docker](http://www.docker.io)来打包、实例化和运行容器应用。

那么Kubernetes就是一个Docker调度系统吗？是，也不是。

Kubernetes帮助用户实现他们所期望的健壮运维系统。我们将用户这种最基本的需求视作Kubernetes最核心的价值。自恢复机制，如自动重启、自动调节和容器复制。这需要主动控制，而不仅仅是一个被动调度。

Kubernetes主要作用于多容器组成的应用，如易扩展、分布式的子应用。它也被设计来简化非容器应用到Kubernetes的迁移工作。所以它同时包括了对低耦合和高耦合分组的抽象，并提供一种方式允许容器间按照它们之前的方式来交互。

Kubernetes允许用户在集群中运行一组容器，并且系统自动选择机器来运行这些容器。虽然Kubernetes的Scheduler现在还非常简单，但我们会预见到随着时间的增长，它将慢慢复杂起来。Schedule是一个关注多策略性、拓扑关系、业务流相关的功能，这将极大影响可用性、性能和可扩展性。Scheduler需要关注到单独的和共用的资源管理需求、QOS需求、软硬件限制、资源绑定和非绑定情况、数据本地化、负载均衡、超时管理，等等。负载相关的需求将会有相关的API来解决。

除了本地安装，Kubernetes还可以在很多其他平台上安装。

单个的Kubernetes集群不应该用来处理多个可用的分区。相反，我们推荐创建另一个更高层抽象来复制整个分区，在不同的分区上建立整套HA的系统。

Kubernetes目前还不支持多用户场景。

最后，Kubernetes立志成为一个可扩展、可插拔、构建化的OSS平台和工具集。所以，架构上，我们希望Kubernetes成为一系列可插拔的模块和分层的集合，可以使用不同的Scheduler、Controller、存储系统和分布式机制，我们正在将代码向这个方向演进。此外，我们希望其他人能一起来在不更改Kubernetes核心代码的基础上，扩展它的功能，例如增加更高层抽象的PAAS功能、多集群分层。所以，它的API不应该(或仅仅)只限于面向用户，而是成为开发者的工具集。所以，这里其实没有所谓的"内部"模块间API。所有的API都是可见并可使用的，包括Scheduler使用的、node controller和replication controller manager使用的和Kubelete API使用的，等等。没有任何界限需要打破，为了解决更复杂的用户场景，你可以用直接透明地组合访问访问底层的API。

### 集群架构

一个运行中的Kubernetes集群包括一个分布式存储上的一个或多个node agent和master组件(API，Scheduler等)。这个图显示了我们希望的最佳模型，其中有些部分还在设计之中，例如使kubelete本身(实际上应该是全部组件)运行在容器当中，并且使scheduler100%可被替换。

![architechture](/images/architecture.png)

## 核心概念

对比Docker管理单独的容器，Kubernetes提供了更高一层的管理框架来支持集群的使用方式，并且目前主要关注应用服务。今后也会扩展到批处理和定制化测试。


### Pods

一个_pod_(可大可小)是同一台机器上相对紧耦合一组容器。它模拟了容器化环境中一个面向应用的"虚拟主机"。Pod作为基础单元，使用在调度、部署、水平扩容和复制、共享资源，如共享存储、共享IP地址。

### Labels

松耦合的Pod之间使用key/value的_lable_来管理。

单独的label用来区分元数据，在语意上表达容器所属pod的目的和角色。Pod的labeld的key典型的例子有，`service`, `environment`(例如，以`dev`,`qa`,`production`为value)，`tier`(以`frontend`,`backend`为value)，`track`(以`daily`和`weekly`为value)。当然你也可以有自己的定义。

通过_lable selector_，用户可以定位到一组pod。Label selector是Kubernetes最核心的分组功能。它可以用来识别pod复制或分享，工作池对象，或分布式应用的节点。Kubernetes目前支持两种对象来使用label selector跟踪它们的成员，`service`和`replicationController`。

- `Service`，一个service是proxy的配置单元，这些proxy运行在每一个worker node上。它命名并指定到一个或多个pod。
- `ReplicationController`，一个replicationController使用一个模板并保证任何时刻，这个模板的指定数量的"复制品"都在运行中。如果太多了，杀掉一些，如果太少了，再启动一些。

`Service`指向的这组pod是由label selector定义的。同样的，`replicationController`监控下的pod的复制也是由label selector定义的。

出于便利性和一致性的管理考虑，`service`和`replicationController`自己也可以定义label，并且一般也将这些label使用在它们对应的pod上。

## Kubernetes节点

回顾系统架构，我们可以把它分为运行在worker节点上的service和组成集群control panel的service。

Kubernetes node被master管理，包含有运行Docker容器所必须的service。

Kubernetes node设计被视为[Container-optimized Google Compute Engine image](https://developers.google.com/compute/docs/containers/container_vms)的演进。长久以来，它用来在不同的方式上管理node/image。它被master管理，并包含必要的service来运行Docker容器。

当然了，每个node上都运行Docker。Docker负责具体的镜像下载和容器运行。

### Kubelet

这个node上的第二个组件叫Kubelet。逻辑上，Kubelet是Goole compute engine中[Container Agent](https://github.com/GoogleCloudPlatform/container-agent)的实现(使用Go语言重写)。

Kubelet的运行与容器的manifest相关。一个容器的manifest文件就是用来描述一个pod的yaml文件(定义在 [这里](https://developers.google.com/compute/docs/containers/container_vms#container_manifest))。Kubelet使用来自不同方式的一组manifest，并保证其中定义的容器启动和运行。

这里有四种方式把容器manifest提供给Kubelet

* **文件**，用命令行参数传递文件路径。每20秒检查一次文件（参数可配）。
* **HTTP节点**，用命令行参数传递HTTP地址。每20秒检查一次地址（参数可配）。
* **etcd server**，Kubelet将访问并监听[etcd](https://github.com/coreos/etcd)。被监控的etcd路径为``/registry/hosts/${hostname -f)``。由于是实时监控，更改的通知和回调会被立刻触发。
* **HTTP server**，Kubelet同样可以监听HTTP，并响应简单的API(正在开发中)来提交新的manifest。

### Kubernetes代理

每个node也会运行一个简单地网络代理。这一点体现为Kubernetes API在每个node中定义的services，它可以转发简单地TCP和UDP到一组backend上。

目前，service节点可以通过环境变量来定义（[Docker-links-compatible](https://docs.docker.com/userguide/dockerlinks/)和Kubernetes的环境变量{FOO}_SERVICE_HOST、{FOO}_SERVICE_PORT都支持）。这些环境变量作为service代理所管理的端口。

### Kubernetes 控制层

Kubernetes控制层包括一组组件，全部运行在单个master节点上。它们协同工作提供一个统一的集群管理。

### etcd

所有master的持久状态保存在一个etcd实例中，可靠地储存了配置数据。伴随着watch功能，状态变化可以被迅速通知到组件上。

### Kubenetes API server

这个server提供绝大多数的[Kubernetes API](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/api)服务。

它校验并配置3种类型的对象：`pod`，`service`和`replicationController`。

除了支持REST操作、验证并且储存对象在etcd中，API Server还做另外两件事情：

* 分发pod到workder节点。目前分发功能还非常简单。
* 使用service配置来同步pod信息(pod在哪儿，都暴露了哪些端口)。

### Kubernetes Controller Manager Server

严格的说，要使用Kubernetes时，上面提到的replicationController类型不是必须的。它事实上是简单的pod API上一层的service。为了强化这种分层概念，replicationController的逻辑实际上分解到了另外的Server上。它监听etcd中replicationController的改变通知，并使用公共的API来实现复制的逻辑。