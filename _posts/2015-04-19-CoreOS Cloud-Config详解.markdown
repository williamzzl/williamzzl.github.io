---
layout: post
title:  "CoreOS Cloud-Config详解"
date:   2015-04-19 12:30:25
categories: CoreOS
tags: CoreOS Docker
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

#如何使用CoreOS Cloud-Config

CoreOS允许自定义各种OS级别的项，如网络、用户管理和systemd units。这篇文档展示了所有可修改的配置，然后`coreos-cloudinit`程序在OS启动后或运行时使用这些文件来完成配置。

每次启动都会执行cloud-config。没有通过校验的cloud-config不会被执行，并且失败结果会被记录在journal中。可以使用[CoreOS validator]或`coreos-cloudinit -validate`来校验cloud-config。

##配置文件

系统启动所使用的这个文件叫cloud-config。它的设计灵感来自于[cloud-init]项目的[cloud-config]文件，”解决分布式云部署初始化问题的文件”([cloud-init文档])。cloud-init项目中的工具不能直接在CoreOS中使用，只有配置文件中相关的一部分在CoreOS的cloud-config文件中被实现。此外，我们增加了一些CoreOS特有的配置，如etcd配置、OEM定义和systemd units。

同一个cloud-config文件可以跨所有平台上使用。

###文件格式

Cloud-config是[YAML]文件格式，使用空格和新行来定义list，数组和值。

Cloud-config文件必须包含`#cloud-config`，后面跟相关的数组，包含一个或多个以下的Key:

   * `coreos`
   * `ssh_authorized_keys`
   * `hostname`
   * `users`
   * `write_files`
   * `manage_etc_hosts`

它们对应的值的定义将在接下来的文章中介绍。

###使用Config-Drive提供Cloud-Config

在每个平台上，CoreOS尝试使用平台的Native方法来管理用户数据。每个平台都不一样，但是这个复杂性已经被CoreOS处理掉了。你可以查看每个平台的指导手册。其中最通用的提供Cloud-Config的方式是[config-drive]，挂载一个包含cloud-config文件的只读设备。

##配置参数

###coreos

####etcd(不再推荐，请参考etcd2)

`coreos.etcd.*`参数作为etcd配置文件，会被转换为部分systemd unit。如果环境变量支持coreos-cloudinit的模板功能，也可以在etcd的配置中使用`$private_ipv4`和`$public_ipv4`。例如，以下的cloud-config...

    #cloud-config
    
    coreos:
        etcd:
        name: node001
        # generate a new token for each unique cluster from https://discovery.etcd.io/new
        discovery: https://discovery.etcd.io/<token>
        # multi-region and multi-cloud deployments need to use $public_ipv4
        addr: $public_ipv4:4001
        peer-addr: $private_ipv4:7001
...会转换为etcd.service的systemd unit：

    [Service]
    Environment="ETCD_NAME=node001"
    Environment="ETCD_DISCOVERY=https://discovery.etcd.io/<token>"
    Environment="ETCD_ADDR=203.0.113.29:4001"
    Environment="ETCD_PEER_ADDR=192.0.2.13:7001"

有关更多可使用的配置参数，请参考[etcd文档]。

>Note: 在这里和其他文档中提到的`$private_ipv4`和`$public_ipv4`变量只在以下平台支持：Amazon EC2,Google Compute Engine, OpenStack, Rackspace, DigitalOcean和Vagrant。

####etcd2

`coreos.etcd2.*`参数作为etcd配置文件，会被转换为部分systemd unit。如果环境变量支持coreos-cloudinit的模板功能，也可以在etcd的配置中使用`$private_ipv4`和`$public_ipv4`。使用[discovery token]时，需要设置size参数，这是因为etcd会使用这个参数来判断是否所有成员都已经加入集群。集群启动后，可以根据这个配置来扩容或缩减。
例如，下面的cloud-config...

    #cloud-config
    
    coreos:
      etcd2:
        # generate a new token for each unique cluster from     https://discovery.etcd.io/new?size=3
        discovery: https://discovery.etcd.io/<token>
        # multi-region and multi-cloud deployments need to use $public_ipv4
        advertise-client-urls: http://$public_ipv4:2379
        initial-advertise-peer-urls: http://$private_ipv4:2380
        # listen on both the official ports and the legacy ports
        # legacy ports can be omitted if your application doesn't depend on them
        listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
        listen-peer-urls: http://$private_ipv4:2380,http://$private_ipv4:7001
...会转换为etcd2.service的systemd unit：

    [Service]
    Environment="ETCD_DISCOVERY=https://discovery.etcd.io/<token>"
    Environment="ETCD_ADVERTISE_CLIENT_URLS=http://203.0.113.29:2379"
    Environment="ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.0.2.13:2380"
    Environment="ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379,http://0.0.0.0:4001"
    Environment="ETCD_LISTEN_PEERS_URLS=http://192.0.2.13:2380,http://192.0.2.13:7001"

有关更多可使用的配置参数，请参考[etcd文档]。

>Note: 在本文和其他文档中提到的`$private_ipv4`和`$public_ipv4`变量只在以下平台支持：Amazon EC2,Google Compute Engine, OpenStack, Rackspace, DigitalOcean和Vagrant。

####fleet

`coreos.fleet.*`参数工作方式和`coreos.etcd2.*`类似，并且允许使用环境变量。例如，下面的cloud-config...

    #cloud-config
    
    coreos:
      fleet:
          public-ip: $public_ipv4
          metadata: region=us-west
...会转换为systemd unit；

    [Service]
    Environment="FLEET_PUBLIC_IP=203.0.113.29"
    Environment="FLEET_METADATA=region=us-west"

有关更多fleet配置的信息，请参考[fleet文档]。

####flannel

`coreos.flannel.*`参数工作方式同样和`coreos.etcd2.*`，`coreos.fleet.*`类似，并且允许使用环境变量。例如，下面的cloud-config...

    #cloud-config
    
    coreos:
      flannel:
          etcd_prefix: /coreos.com/network2
...会转换为systemd unit；

    [Service]
    Environment="FLANNELD_ETCD_PREFIX=/coreos.com/network2"

flannel配置的参数列表：

* **etcd_endpoints**: 逗号分隔的etcd节点
* **etcd_cafile**: etcd用来TLS通信的CA文件路径
* **etcd_certfile**: etcd用来TLS通信的证书文件路径
* **etcd_keyfile**:  etcd用来TLS通信的private key路径
* **etcd_prefix**: flannel keys的etcd路径前缀
* **ip_masq**: 用于flannel子网向外发包的IP masquerade规则
* **sub_net_file**: flannel子网写入文件的路径
* **interface**：用于内部主机间通信的接口（名字或IP）

####locksmith

`coreos.locksmith.*`参数可以用来为locksmith设置环境变量。例如，下面的cloud-config...

    #cloud-config
    
    coreos:
      locksmith:
          endpoint: example.com:4001
…会转换为systemd unit：

    [Service]
    Environment="LOCKSMITHD_ENDPOINT=example.com:4001"

有关完整的locksmith配置参数列表，请参考[locksmith文档]。

####upate

`coreos.upate.*`参数用来决定如何升级CoreOS系统。

这些值将会写入并替换`/etc/coreos/update.conf`文件。如果只提供一个参数，它将只替换文件中对应的项。`reboot-strategy`参数同样影响[locksmith]的行为。

* **reboot-strategy**: “reboot”, “etcd-lock”, “best-effor”或”off”其一，决定升级完成后什么时候执行重启。
 * **reboot**: 升级之后立即重启。
 * **etcd-lock**: etcd第一次获得分布式锁后重启。保证了同时只有一台机器重启，这样在升级中集群中就一定有其他机器来提供服务。
 * **best-effort**: 如果etcd在运行，使用’etcd-lock’策略，否则直接使用’reboot’策略。
 * **off**: 升级后禁止重启（不推荐使用）。
* **server**: 执行升级查询的[CoreUpdate]服务器地址。也被称作[omaha]服务器节点。
* **group**: 标识用来自动升级的通道。默认是镜像文件开始下载时的值。（”master”, ‘alpha’,’beta’, ’stable’其一）

>Note: cloudinit只会修改locksmith在ststemd运行时的目录下的unit文件(`/run/systemd/system/locksmithd.service`)。如果手工修改覆盖了unit配置文件(如`/etc/systemd/system/locksmithd.service`)，cloudinit将不能再控制locksmith服务。

例如

    #cloud-config
    coreos:
      update:
        reboot-strategy: etcd-lock

####units

`coreos.units.*`参数定义了一系列系统启动后运行的systemd units。这个功能意在帮助你在加入CoreOS集群前启动必要的服务来挂载存储，配置网络。不应该作为Chef/Puppet的替代品。

每一项都含有以下参数：

* **name**: String，unit名字，必须。
* **runtime**: Boolean，是否重启仍然生效。类似`systemctl * enable`的`--runtime`参数。默认false。
* **enable**:  Boolean，是否处理unit的[Install]部分。类似`systemctl enable <name>`。默认false。
* **content**: Plaintext string，unit文件部分。为空表示假设unit已经退出。
* **command**: unit执行的命令: start, stop, reload, restart, try-restart, reload-or-restart, reload-or-try-restart. 默认不执行任何命令。
* **mask**: 是否符号链接unit文件到`/dev/null`来作mask（类似`systemctl mask <name>`）。需要注意的是，不同于`systemctl mask`，这个参数会**删除一个已存在**的`etc/systemd/system<unit>`文件来保证mask成功。默认false。
* **drop-ins**: unit的drop-ins列表，包含这些部分:
 * **name**: String, unit名字，必须。
 * **content**: Plaintext String, unit文件部分，必须。

>Note: Command参数在所有network, netdev和link unit中会被忽略。systemd-networkd.service会自己重启。

例如：
写一个unit到磁盘上，并且自动启动。

    #cloud-config
    
    coreos:
      units:
        - name: docker-redis.service
          command: start
          content: |
            [Unit]
            Description=Redis container
            Author=Me
            After=docker.service
    
            [Service]
            Restart=always
            ExecStart=/usr/bin/docker start -a redis_server
            ExecStop=/usr/bin/docker stop -t 2 redis_server
增加DOCKER_OPT环境变量到docker.service。

    #cloud-config
    
    coreos:
      units:
        - name: docker.service
          drop-ins:
            - name: 50-insecure-registry.conf
              content: |
                [Service]
                Environment=DOCKER_OPTS='--insecure-registry="10.0.1.0/24"'
启动内置的`etcd2`和`fleet`服务：

    #cloud-config
    
    coreos:
      units:
        - name: etcd2.service
          command: start
        - name: fleet.service
          command: start

###ssh_authorized_keys

`ssh_authorized_keys`参数添加`core`用户公共ssh keys来执行鉴权。

默认key命名为”cores-cloudinit”。在执行`coreos-cloudinit`时，使用`--ssh-key-name`来覆盖这个值。

    #cloud-config
    
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...

###hostname

`hostname`参数用来定义系统的名称。这个是完成的domain name的local部分(如,`foo.example.com`中的`foo`)。

    #cloud-config
    
    hostname: coreos1

###users

`users`参数添加或修改指定用户列表。每一个用户都包含这些参数。除非特别说明，每一个参数都是可选并且是string类型。除了`passwd`和`ssh-authorized-keys`，其他参数如果已存在的话会被忽略。

* **name**，必须。登陆用户名
* **gecos**:  用户详细信息
* **passwd**: 用户密码hash
* **homedir**: 用户主目录。默认/home/<name>
* **no-create-home**: Boolean。不创建用户主目录。
* **primary-group**: 用户默认组。默认创建同名的用户组。
* **groups**: 添加用户到其他组。
* **no-user-group**: Boolean. 不创建用户组。
* **ssh-authorized-keys**: 用户公共ssh key列表。
* **coreos-ssh-import-github**: Github用户ssh keys.
* **coreos-ssh-import-github-users**: 一组github用户的ssh keys.
* **coreos-ssh-import-url**: 从url导入的ssh keys.
* **system**: 创建系统用户。不会创建用户主目录。
* **no-log-init**: Boolean。不初始化lastlog和faillog。
* **shell**: 用户登陆shell。

以下参数还没有实现：

* **inactive**: 创建后禁掉用户。
* **lock-passwd**: Boolean。禁止用户使用密码登陆。
* **sudo**: 将用户加入/etc/sudoers。默认不加入sudo权限。
* **selinux-user**: 对应SELinux用户。
* **ssh-import-id**: 通过ID从Launchpad导入ssh keys.

例如：
  
    #cloud-config
    
    users:
      - name: elroy
        passwd: $6$5s2u6/jR$un0AvWnqilcgaNB3Mkxd5yYv6mTlWfOoCYHZmfi3LDKVltj.E8XNKEcwWm...
        groups:
          - sudo
          - docker
        ssh-authorized-keys:
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...

####生成密码hash

如果选择使用密码而不是SSH key，生成hash对系统安全就显得非常重要。简单地md5 hash在最新的GPU上很容易被攻克。这里列出了几种生成安全hash的方式：

    # On Debian/Ubuntu (via the package "whois")
    mkpasswd --method=SHA-512 --rounds=4096
    
    # OpenSSL (note: this will only make md5crypt.  While better than plantext it should not be considered fully secure)
    openssl passwd -1
    
    # Python (change password and salt values)
    python -c "import crypt, getpass, pwd; print crypt.crypt('password', '\$6\$SALT\$')"
    
    # Perl (change password and salt values)
    perl -e 'print crypt("password","\$6\$SALT\$") . "\n"'
使用更高位数可以生成更安全的密码，但有更多时间，这样的密码hash也会被解密。大多数以发布的rpm使用一个叫做mkpasswd的工具，在`expect`包中，但不能处理位数或更高级的hash算法。

####获取SSH鉴权Keys

#####从Github
使用`coreos-ssh-import-github`参数，可以从github用户导入公共SSH keys来执行服务器鉴权。

    #cloud-config
    
    users:
      - name: elroy
        coreos-ssh-import-github: elroy

#####从HTTP节点
同样可以从HTTP节点拉取公共SSH keys(遵循[GitHub’ API response格式])。例如，如果安装了GitHub企业版，可以提供包含鉴权token的完整URL:

    #cloud-config
    
    users:
      - name: elroy
        coreos-ssh-import-url: https://github-enterprise.example.com/api/v3/users/elroy/keys?access_token=<TOKEN>
    
可以同样指定能够回复JSON格式的URL来提供公共SSH key:

    #cloud-config
    
    users:
      - name: elroy
        coreos-ssh-import-url: https://example.com/public-keys

###write_files

`write_files`参数定义要在本地文件系统创建的一系列文件。其中的每一项有以下的参数：

* **path**: 磁盘上文件将被写入的绝对路径
* **content**: 写入到路径中的内容
* **permissions**: 体现为访问权限的数字，典型的是8进制标记（如0644）
* **owner**: 文件所属用户和组。等于`chown <user>:<group> <path>`中的`<user>:<group>`
* **encoding**: 可选。文件内容编码。如果没有指定，使用yaml的编码（通常为utf-8）。支持的编码类型为：
* **b64, base64**: Base64编码
* **gz, gzip**: gzip压缩，配合!!binary标签使用
* **gz+b64, gz+base64, gzip+b64, gzip+base64**: base64编码的压缩内容

例如：

    #cloud-config
	
    write_files:
      - path: /etc/resolv.conf
        permissions: 0644
        owner: root
        content: |
          nameserver 8.8.8.8
      - path: /etc/motd
        permissions: 0644
        owner: root
        content: |
          Good news, everyone!
      - path: /tmp/like_this
        permissions: 0644
        owner: root
        encoding: gzip
        content: !!binary |
          H4sIAKgdh1QAAwtITM5WyK1USMqvUCjPLMlQSMssS1VIya9KzVPIySwszS9SyCpNLwYARQFQ5CcAAAA=
      - path: /tmp/or_like_this
        permissions: 0644
        owner: root
        encoding: gzip+base64
        content: |
          H4sIAKgdh1QAAwtITM5WyK1USMqvUCjPLMlQSMssS1VIya9KzVPIySwszS9SyCpNLwYARQFQ5CcAAAA=
      - path: /tmp/todolist
        permissions: 0644
        owner: root
        encoding: base64
        content: |
          UGFjayBteSBib3ggd2l0aCBmaXZlIGRvemVuIGxpcXVvciBqdWdz

###manage_etc_hosts

`manage_etc_hosts`参数配置`/etc/hosts`文件，用来作域名解析。目前只支持“localhost”，这样就可以将机器名解析为”127.0.0.1”。这样，当没有DNS来解析主机名是就很方便，比如在使用vagrant时。

    #cloud-config
    
    manage_etc_hosts: localhost
    
[CoreOS validator]:[https://coreos.com/validate]
[cloud-init]:[https://launchpad.net/cloud-init]
[cloud-config]:[http://cloudinit.readthedocs.org/en/latest/topics/format.html#cloud-config-data]
[cloud-init文档]:[http://cloudinit.readthedocs.org/en/latest/index.html]
[YAML]:[https://en.wikipedia.org/wiki/YAML]
[config-drive]:[]https://github.com/coreos/coreos-cloudinit/blob/master/Documentation/config-drive.md
[etcd文档]:[https://github.com/coreos/etcd/blob/86e616c6e974828fc9119c1eb0f6439577a9ce0b/Documentation/configuration.md]
[discovery token]:[https://discovery.etcd.io/new?size=3]
[fleet文档]:[https://github.com/coreos/fleet/blob/master/Documentation/deployment-and-configuration.md#configuration]
[locksmith文档]:[https://github.com/coreos/locksmith/blob/master/README.md]
[locksmith]:[https://github.com/coreos/locksmith]
[CoreUpdate]:[https://coreos.com/products/coreupdate]
[omaha]:[https://coreos.com/docs/coreupdate/custom-apps/coreupdate-protocol/]
[GitHub’ API response格式]:[https://developer.github.com/v3/users/keys/#list-public-keys-for-a-user]