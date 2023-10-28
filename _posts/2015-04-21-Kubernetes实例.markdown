---
layout: post
title:  "Kubernetes实例"
date:   2015-04-21 14:34:25
categories: kubernetes
tags: kubernetes docker
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

## GuestBook 例子

这个例子展示了如何使用Kubernetes和Docker来创建一个简单、多层的web应用。

这个例子包含两部分:

* 一个web前端
* 一个redis master（用作存储）和一组复用的redis slaves

Web前端通过redis的javascript API与redis master交互。

>还不了解Kubernetes，没有关系，[Kubernetes概述](/kubernetes/2015/04/11/Kubernetes%E6%A6%82%E8%BF%B0.html)可以帮助你快速入门。

### 第0步: 前提条件

这个例子需要一个Kubernetes集群环境。如何开始请参考[Getting Started guides](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/getting-started-guides)。

如果从source文件运行的，需要将下面命令的`kubectl`替换为`cluster/kubectl.sh`。

### 第一步: 启动redis master

注: 这个redis-master*不是*highly available。变为highly available将非常有意思，但也比较复杂，因为到目前为止，redis还不支持multi-master部署，所以实现high availability有些棘手，而且会引发一些磁盘写乱序之类的问题.


使用（或创建）一个单pod文件`examples/guestbook/redis-master-controller.json`来在容器中运行一个redis key-value server。

>需要注意，尽管这个redis server只运行一份拷贝，我们仍然使用replication controller来强制只有一个pod在运行（例如，当node停掉后，replication controller会确保redis master重启至正常状态）。这可能导致数据丢失。

**这些json文件对应v1beta1。更新后的版本请参考[v1beta3/](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/v1beta3/)目录**

	{
	  "id": "redis-master-controller",
	  "kind": "ReplicationController",
	  "apiVersion": "v1beta1",
	  "desiredState": {
	    "replicas": 1,
	    "replicaSelector": {"name": "redis-master"},
	    "podTemplate": {
	      "desiredState": {
	        "manifest": {
	          "version": "v1beta1",
	          "id": "redis-master",
	          "containers": [{
	            "name": "redis-master",
	            "image": "redis",
	            "ports": [{
	              "containerPort": 6379,   # containerPort: Where traffic to redis ultimately is routed to.
	            }]
	          }]
	        }
	      },
	      "labels": {
	        "name": "redis-master",
	        "app": "redis"
	      }
	    }
	  },
	  "labels": {
	    "name": "redis-master" # This label needed for when we start our redis-master service.
	  }
	}


现在，通过运行以下命令来在你的Kubernetes集群中创建redis pod：

	$ kubectl create -f examples/guestbook/redis-master-controller.json
    
	$ cluster/kubectl.sh get rc
	CONTROLLER                             CONTAINER(S)            IMAGE(S)                                 SELECTOR                     REPLICAS
	redis-master-controller                redis-master            redis                                    name=redis-master            1

一旦启动，你可以列出集群中所有pods，来验证master正在运行：

    $ kubectl get pods

你将看到所有kubernetes模块，其中最重要的是redis master pod。一旦部署好，它同样会显示出这个pod被部署的那台机器（可以最多需要30秒）：

	POD                                          IP                  CONTAINER(S)            IMAGE(S)                                 HOST                                                              LABELS                                                     STATUS
	redis-master-controller-gb50a                10.244.3.7          redis-master            redis                                    kubernetes-minion-7agi.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-master                                Running

如果希望ssh登陆这台机器，运行`docker ps`来查看具体的pod:

	me@workstation$ gcloud compute ssh kubernetes-minion-7agi
    
	me@kubernetes-minion-7agi:~$ sudo docker ps
	CONTAINER ID        IMAGE                                  COMMAND                CREATED              STATUS              PORTS                    NAMES
	0ffef9649265        redis:latest                           "redis-server /etc/r   About a minute ago   Up About a minute                            k8s_redis-master.767aef46_redis-master-controller-gb50a.default.api_4530d7b3-ae5d-11e4-bf77-42010af0d719_579ee964

>注，取决于网络条件，初始执行`docker pull`可能会花几分钟。Image文件下载时，可以通过运行`journalctl -f -u docker`来查看状态。当然，同时也可以运行`journalctl -f -u kuberlet`来查看kubelet的状态。

### 第二步：启动master服务

一个Kubernetes中的load balancer叫'service'，用来分发请求到*一个或多个*容器中。这用到了我们在前面定义redis-master时使用的*labels*元数据。之前说过，redis只有一个master，但我们仍然希望将它创建为一个service。为什么？因为这将使我们可以使用一个可变的IP地址指向这个单独的master服务。

Kubernetes集群中得其他容器可以通过环境变量来定位到这个services。

请求基于pod的labels来寻找的对应的容器。

第一步创建的pod含有标签`name=redis-master`。service的选择器决定了*哪一个pod将会收到发送至这个服务的请求*，并且端口和容器端口（v1beta3和更新的APIs中的targetPort）信息定义了service将会运行在哪个端口上。

使用文件`examples/guestbook/redis-master-service.json`来定义service:

	{
	  "id": "redis-master",
	  "kind": "Service",
	  "apiVersion": "v1beta1",
	  "port": 6379,
	  "containerPort": 6379,
	  "selector": {
	    "name": "redis-master"
	  },
	  "labels": {
	    "name": "redis-master"
	  }
	}

运行以下命令启动service:

	$ kubectl create -f examples/guestbook/redis-master-service.json
	redis-master
    
	$ kubectl get services
	NAME                    LABELS                                    SELECTOR                     IP                  PORT
	redis-master            name=redis-master                         name=redis-master            10.0.246.242        6379


这将使所有的pod看到redis master运行在<ip>:6379。从slaves到masters的请求分以下两个步骤：

- 一个*redis slave*将会连接到*redis master service*的"端口"。
- 请求将会从service的"端口"（在service node上）转发至这个service所监听的pod里面的*容器端口*（v1beta3和更新的APIs中的targetPort)。 

这样，一旦创建好，每个子节点上的service就配置好了对应具体端口（本例中得6379）的映射。

### 第三步：启动多个slave pods

虽然redis master是单个的pod，redis slaves是一组’复用‘的pod。Kuernetes中，replication controller负责管理复用的pod的多个实例。如果具体数目和配置不一样，replicationController会自动拉起新的Pods（做一个简单有趣的试验，随便kill掉几个你的docker进程，看看它们稍后怎么在一个新node上恢复起来）。

使用文件`examples/guestbook/redis-slave-controller.json`创建一个replication controller:

	{
	  "id": "redis-slave-controller",
	  "kind": "ReplicationController",
	  "apiVersion": "v1beta1",
	  "desiredState": {
	    "replicas": 2,
	    "replicaSelector": {"name": "redis-slave"},
	    "podTemplate": {
	      "desiredState": {
	         "manifest": {
	           "version": "v1beta1",
	           "id": "redis-slave",
	           "containers": [{
	             "name": "redis-slave",
	             "image": "kubernetes/redis-slave:v2",
	             "ports": [{"containerPort": 6379}]
	           }]
	         }
	      },
	      "labels": {
	        "name": "redis-slave",
	        "uses": "redis-master",
	        "app": "redis"
	      }
	    }
	  },
	  "labels": {"name": "redis-slave"}
	}

运行以下命令来运行这个replication controller:

	$ kubectl create -f examples/guestbook/redis-slave-controller.json
	redis-slave-controller
    
	$ kubectl get rc
	CONTROLLER                             CONTAINER(S)            IMAGE(S)                                 SELECTOR                     REPLICAS
	redis-master-controller                redis-master            redis                                    name=redis-master            1
	redis-slave-controller                 redis-slave             kubernetes/redis-slave:v2                name=redis-slave             2

通过以下命令启动redis slave：

    redis-server --slaveof redis-master 6379

启动完毕后，列出集群内的pods，确认master和slave都在运行：

	$ kubectl get pods
	POD                                          IP                  CONTAINER(S)            IMAGE(S)                                 HOST                                                              LABELS                                                     STATUS
	redis-master-controller-gb50a                10.244.3.7          redis-master            redis                                    kubernetes-minion-7agi.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-master                                Running
	redis-slave-controller-182tv                 10.244.3.6          redis-slave             kubernetes/redis-slave:v2                kubernetes-minion-7agi.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-slave,uses=redis-master               Running
	redis-slave-controller-zwk1b                 10.244.2.8          redis-slave             kubernetes/redis-slave:v2                kubernetes-minion-3vxa.c.hazel-mote-834.internal/104.154.54.6     app=redis,name=redis-slave,uses=redis-master               Running

你会看到一个redis master pod和两个redis slave pods。

### 第四步：创建redis slave service

类似master，我们需要一个映射读请求到slaves的service。这里，除了service discovery，这个slave的service为web app客户端提供了透明的load balance。

slave的service定位文件`examples/guestbook/redis-slave-service.json`:

	{
	  "id": "redis-slave",
	  "kind": "Service",
	  "apiVersion": "v1beta1",
	  "port": 6379,
	  "containerPort": 6379,
	  "labels": {
	    "name": "redis-slave"
	  },
	  "selector": {
	    "name": "redis-slave"
	  }
	}

这次，这个service的选择器是`name=redis-slave`，这个用来识别运行redis slave的pod。为了更方便些，可以在service上添加自定义的labels，然后就可以通过命令`cluster/kubectl.sh get services -l "label=value"`来找到service。

现在已经创建好了service，运行以下命令在集群中运行：

	$ kubectl create -f examples/guestbook/redis-slave-service.json
	redis-slave
    
	$ kubectl get services
	NAME                    LABELS                                    SELECTOR                     IP                  PORT
	redis-master            name=redis-master                         name=redis-master            10.0.246.242        6379
	redis-slave             name=redis-slave                          name=redis-slave             10.0.72.62          6379

### 第五步：创建frontend pod

这是一个简单地PHP server，根据是读还是写来分别向slave或master发送请求。它开发一个简单地AJAX接口，提供基于angular的UX。像redis read slave一样，它也是一个被replication controller初始化的复用的service。

它可以将对负载均衡的redis-slaves的请求和写请求分别开，于是达到了高可复用性。

pod的定义文件`examples/guestbook/frontend-controller.json`:

	{
	  "id": "frontend-controller",
	  "kind": "ReplicationController",
	  "apiVersion": "v1beta1",
	  "desiredState": {
	    "replicas": 3,
	    "replicaSelector": {"name": "frontend"},
	    "podTemplate": {
	      "desiredState": {
	         "manifest": {
	           "version": "v1beta1",
	           "id": "frontend",
	           "containers": [{
	             "name": "php-redis",
	             "image": "kubernetes/example-guestbook-php-redis:v2",
	             "ports": [{"name": "http-server", "containerPort": 80}]
	           }]
	         }
	       },
	       "labels": {
	         "name": "frontend",
	         "uses": "redis-slave-or-redis-master",
	         "app": "frontend"
	       }
	      }},
	  "labels": {"name": "frontend"}
	}

通过以下命令启用frontend：

	$ kubectl create -f examples/guestbook/frontend-controller.json
	frontend-controller
    
	$ kubectl get rc
	CONTROLLER                             CONTAINER(S)            IMAGE(S)                                   SELECTOR                     REPLICAS
	frontend-controller                    php-redis               kubernetes/example-guestbook-php-redis:v2  name=frontend                3
	redis-master-controller                redis-master            redis                                      name=redis-master            1
	redis-slave-controller                 redis-slave             kubernetes/redis-slave:v2                  name=redis-slave             2

一旦启动（可能10-30秒来创建pod)，可以通过以下命令列出集群中的pods，查看master，slaves和frontend：

	$ kubectl get pods
	POD                                          IP                  CONTAINER(S)            IMAGE(S)                                   HOST                                                              LABELS                                                     STATUS
	frontend-controller-5m1zc                    10.244.1.131        php-redis               kubernetes/example-guestbook-php-redis:v2  kubernetes-minion-3vxa.c.hazel-mote-834.internal/146.148.71.71    app=frontend,name=frontend,uses=redis-slave,redis-master   Running
	frontend-controller-ckn42                    10.244.2.134        php-redis               kubernetes/example-guestbook-php-redis:v2  kubernetes-minion-by92.c.hazel-mote-834.internal/104.154.54.6     app=frontend,name=frontend,uses=redis-slave,redis-master   Running
	frontend-controller-v5drx                    10.244.0.128        php-redis               kubernetes/example-guestbook-php-redis:v2  kubernetes-minion-wilb.c.hazel-mote-834.internal/23.236.61.63     app=frontend,name=frontend,uses=redis-slave,redis-master   Running
	redis-master-controller-gb50a                10.244.3.7          redis-master            redis                                      kubernetes-minion-7agi.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-master                                Running
	redis-slave-controller-182tv                 10.244.3.6          redis-slave             kubernetes/redis-slave:v2                  kubernetes-minion-7agi.c.hazel-mote-834.internal/104.154.54.203   app=redis,name=redis-slave,uses=redis-master               Running
	redis-slave-controller-zwk1b                 10.244.2.8          redis-slave             kubernetes/redis-slave:v2                  kubernetes-minion-3vxa.c.hazel-mote-834.internal/104.154.54.6     app=redis,name=redis-slave,uses=redis-master               Running

可以看到单个redis master pod，两个redis slaves和三个frontend pods。

PHP service的代码：

	<?
    
	set_include_path('.:/usr/share/php:/usr/share/pear:/vendor/predis');
    
	error_reporting(E_ALL);
	ini_set('display_errors', 1);
    
	require 'predis/autoload.php';
    
	if (isset($_GET['cmd']) === true) {
	  header('Content-Type: application/json');
	  if ($_GET['cmd'] == 'set') {
	    $client = new Predis\Client([
	      'scheme' => 'tcp',
	      'host'   => 'redis-master',
	      'port'   => 6379,
	    ]);
    
	    $client->set($_GET['key'], $_GET['value']);
	    print('{"message": "Updated"}');
	  } else {
	    $client = new Predis\Client([
	      'scheme' => 'tcp',
	      'host'   => 'redis-slave',
	      'port'   => 6379,
	    ]);
    
	    $value = $client->get($_GET['key']);
	    print('{"data": "' . $value . '"}');
	  }
	} else {
	  phpinfo();
	} ?>

### 第六步：创建guestbook service.

和其他一样，需要创建一个指向frontend pods的service。

Service的定义文件`examples/guestbook/frontend-service.json`:

>**NOTE** 这个json文件有过更新，增加的publicIPs字段会在下面章节中提到。

	{
	  "id": "frontend",
	  "kind": "Service",
	  "apiVersion": "v1beta1",
	  "port": 8000,
	  "containerPort": "http-server",
	  "publicIPs":["10.11.22.33"],
	  "selector": {
	    "name": "frontend"
	  },
	  "labels": {
	    "name": "frontend"
	  },
	  "createExternalLoadBalancer": true
	}

如果运行在单node，或单个虚拟机上，就不需要`createExternalLoadBalancer`，也不需要`publicIPs`。

阅读下面的*从外部访问Guestbook*章节，对应的设置10.11.22.33（目前，你可以删除这些参数来启动，全部保留也没有关系）。


	$ kubectl create -f examples/guestbook/frontend-service.json
	frontend
    
	$ kubectl get services
	NAME                    LABELS                                    SELECTOR                     IP                  PORT
	frontend                name=frontend                             name=frontend                10.0.93.211         8000
	redis-master            name=redis-master                         name=redis-master            10.0.246.242        6379
	redis-slave             name=redis-slave                          name=redis-slave             10.0.72.62          6379

### 将service用在Google Container Engine上需要做的一些说明

GCE中，`cluster/kubectl.sh`自动为包含有`createExternalLoadBalancer`的services创建转发规则。

	$ gcloud compute forwarding-rules list
	NAME                  REGION      IP_ADDRESS     IP_PROTOCOL TARGET
	frontend              us-central1 130.211.188.51 TCP         us-central1/targetPools/frontend

你可以通过这个rule关联的load balancer获取对应的外部IP，然后访问`http://130.211.188.51:8000`。

GCE中，可能需要使用[console][cloud-console]或`gcloud`命令来开启8080的防火墙。以下命令会允许来自任何地址的标签为`kubernetes-minion`的请求：

    $ gcloud compute firewall-rules create --allow=tcp:8000 --target-tags=kubernetes-minion kubernetes-minion-8000

需要在GCE中限定请求的指定源地址，参考[GCE firewall documentation][gce-firewall-docs]。

[cloud-console]: https://console.developer.google.com
[gce-firewall-docs]: https://cloud.google.com/compute/docs/networking#firewalls

### 从外部访问Guestbook

现在，从frontend service可以访问到我们设置好的pod了，你可以会注意到从kubernetes之外访问不了10.0.93.211(frontend service的IP)。

当然，如果你在本地运行kubernetes minions的话，这很简单 - 端口绑定允许你通过localhost:8080来访问guestbook... 但是最熟悉的**localhost**方式明显在现实生活中不适用。

如果你不了解`createExternalLoadBalancer`功能（取决于不同的云提供商），你会希望**在一个minion上设置公网IP**，这样就可以从外部访问了。其实不用这样麻烦，看一下kubelet IP列表，更新service文件来包含一个映射到任意一个已存在的kubeletes的`publicIPs`字符串，这样就可以允许从外面访问kubeletes了（解释：这也可以允许你使用kubelet IP从浏览器上访问guestbook）。

如果你在运维方面有更高的要求，你可以通过`kubectl get pods, services`的输出来获取service IP地址，然后使用你熟悉的标准工具或命令来修改防火墙。

当然，最终，本地运行Kubernetes的话，访问http://localhost:8000即可。

### 第七步：清理

如果你是从source文件启动Kubernetes集群的话，使用以下命令停止Kubernetes集群：

    $ cluster/kube-down.sh

如果kubernetes集群还在运行，可以使用这些命令kill pods（运行前确保你了解命令的含义，这会自动停止所有pods）。

	### First, kill services and controllers.
	kubectl stop -f examples/guestbook/redis-master-controller.json
	kubectl stop -f examples/guestbook/redis-slave-controller.json
	kubectl stop -f examples/guestbook/frontend-controller.json
	kubectl delete -f examples/guestbook/redis-master-service.json
	kubectl delete -f examples/guestbook/redis-slave-service.json
	kubectl delete -f examples/guestbook/frontend-service.json

### Troubleshooting

Guestbook例子可能由于不同原因失败，从测试的角度也是个好消息。为了简单起见，我们使用*curl*命令来测试。

开始之前，列出一些容易犯的错误从而导致app运行失败：

- 获取的是最新版Kubernetes而不是稳定版，这样有可能导致kubernetes内部模块交互有问题。
- 在启用安全策略的环境中运行kubernetes，而阻止容器真正做事。
- 测试前，启动了kubernetes但没有等足够的时间来让所有的service和pod执行online.

发布一个message（注意这将会*覆盖*原有message），这样只会重置一个对象。

	curl "localhost:8000/index.php?cmd=set&key=messages&value=jay_sais_hi"

之后再获取messages...

	curl "localhost:8000/index.php?cmd=get&key=messages"

1) 如果*页面一直没有内容*:

当访问localhost:8000，可能都看不到任何页面。使用curl测试一下...

    ==> default: curl: (56) Recv failure: Connection reset by peer

这意味着前端还没有完全启动。具体的说，"reset by peer"指访问了*正确的端口*，但是那个端口*没有任何内容*。等一会儿，取决你的配置，可能2分钟左右。同样，运行`docker ps`来查看容器是否不停的启停或是一直没有启动。

	$> watch -n 1 docker ps

如果访问的的确是frontend所在的node，最终会看到frontend容器启动好。这时，就不会再有刚才的错误了。

2) *在等待app启动时，偶尔的* , 你可能看到:

	==> default: <br />
	==> default: <b>Fatal error</b>:  Uncaught exception 'Predis\Connection\ConnectionException' with message 'Error while reading line from the server [tcp://10.254.168.69:6379]' in /vendor/predis/predis/lib/Predis/Connection/AbstractConnection.php:141

不要急，很可能稍后service就启动好了。如果没有，确保它在运行，并且redis master/slave有类似这样的日志：

	$> docker logs 26af6bd5ac12
	...
	[9] 20 Feb 23:47:51.015 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
	[9] 20 Feb 23:47:51.015 * The server is now ready to accept connections on port 6379
	[9] 20 Feb 23:47:52.005 * Connecting to MASTER 10.254.168.69:6379
	[9] 20 Feb 23:47:52.005 * MASTER <-> SLAVE sync started

3) *如果是安全策略原因导致redis写失败* 需要运行在redis容器上运行*docker logs*:


	==> default: <b>Fatal error</b>:  Uncaught exception 'Predis\ServerException' with message 'MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.' in /vendor/predis/predis/lib/Predis/Client.php:282" 

解决方法是调整SElinux(不是简单关掉)。记得，你可以使用dockerfile来从头或重新搭建这个app。