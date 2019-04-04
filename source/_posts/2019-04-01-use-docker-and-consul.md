---
title: 使用Docker和Consul(译)
date: 2019-04-01 14:34:52
tags: 
- Docker
- Consul
- 翻译
---
[原文链接](https://medium.com/zendesk-engineering/making-docker-and-consul-get-along-5fceda1d52b9) 可能需要科学上网

&emsp;&emsp;如果你正在管理一个一定规模的互联网技术栈，你很可能听说过[Consul](https://www.consul.io/)。Consul是一个非常棒的解决方案，它能给你的网络提供强大、可靠的服务发现能力。你想要使用它一点也不会让人感到意外。

&emsp;&emsp;让我们假设你已经决定在你的生产环境中使用[Docker](https://www.docker.com/)容器。让我们再假设你打算把容器中的服务发布到Consul的服务注册中心。怎样可靠、轻松的实现这一需求呢？

## 总结

- 直接在你的宿主机上安装Consul和[dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html)，或者把它们安装在使用主机网络(--net=host)的容器里。
- 在宿主机上创建一个[虚拟网络接口(dummy network interface)](http://www.tldp.org/LDP/nag/node72.html)，并给它分配一个本地IP(例如：169.254.1.1)。
- 配置Consul，绑定它的HTTP和客户端RPC服务到上述虚拟网络接口的IP地址上。
- 配置dnsmasq监听虚拟IP地址。
- 配置你的容器，使用虚拟IP地址作为它们的DNS服务器和Consul服务器。
- 使用程序如：[Registrator](https://github.com/gliderlabs/registrator)发布你的容器服务。
<!--more-->

## Consul和容器化应用

&emsp;&emsp;假设你已经决定在你的Docker主机上使用Consul，并且你有以下需求。

- 容器化应用必须能够准确的确定其他应用的地址和端口--不论其他应用是否在同一台主机或者不同的主机。
- 容器化应用必须能够读写Consul的key-value数据库，并且可靠的执行锁操作
- 外部主机上的应用必须能够连接这个容器化应用。
- 健康检查失败的应用必须能够准确的被报告到Consul服务注册中心。
- 如果一个Docker主机变为不可达，其上的所有应用会被标记为down，并且/或者在Consul服务注册中心取消发布。

## 在你的Docker主机上安装Consul

&emsp;&emsp;在你的网络中的所有主机包括Docker主机上安装、运行Consul Agent 被被认为是一个最佳实践。这有一些很重要的好处：
&emsp;&emsp;首先，这使得配置服务变得非常简单。运行于宿主机之上的服务(即非容器化的)可以简单的将包含健康检查的服务定义放置到/etc/consul.d/<service_name>.json，Consul Agent 会在启动时或者信号通知时加载它们。然后Consul Agent会将这些服务发不到注册中心并按照你指定的频率执行你设计的健康检查。

&emsp;&emsp;其次，这能够提供可靠的失败监测。如果你的主机因为关机会其他任何原因变得不可达，运行于其他主机上的Consul Agent网络马上会注意到;并且任何注册在这台主机上的服务都会自动被标记为不可用。

&emsp;&emsp;最后，它提供了一个本地节点来接收Consul DNS查询和HTTP API 请求。这些请求可以不必经过网络，这可以简化网络安全策略和减少网络通讯。

&emsp;&emsp;最具争议的问题是：你应该在宿主机还是一个容器里安装Consul Agent?

&emsp;&emsp;答案是：这无所谓-但有所谓的是`网络配置`。Consul本身是一个很小的、自包含的Linux二进制文件；它没有运行时依赖。如果你愿意你当然可以在容器环境中运行它，但是运行环境隔离带来的吸引力由于Consul根本不需要隔离而变得很小。我个人喜欢在宿主机上与其他系统基础服务Docker engine和sshd等服务一样以一等服务来运行Consul。

&emsp;&emsp;当然你也可以选择在容器中运行Consul。Hashicorp 在Docker Hub发布了[官方镜像](https://hub.docker.com/_/consul/)。重要的部分是当你运行容器时，你必须使用 --net=host 选项。

    $ sudo docker run -d --net=host consul:latest

## Consul和回环接口

&emsp;&emsp;当你运行Consul agent时，它监听6个端口来提供不同的功能。以下三个端口是我们重点讨论的：

- HTTP API (默认：8500)：处理来自客户端的HTTP API 请求
- CLI RPC（默认：8400）：处理来自命令行的请求
- DNS（默认：8600）：回答DNS查询

&emsp;&emsp;默认情况下，Consul只允许来自回环接口（127.0.0.1）的连接。出于安全考虑，这是一个合理的默认选项，而且在非容器环境下没什么问题。但是对于容器应用来说存在一个难题：容器里边的回环接口和宿主机回环接口是分开的。这是由于在Docker里每个容器都在私有的网络命名空间中运行。所以当一个容器化应用尝试通过地址http://127.0.0.1:8500连接Consul时，它一定会失败。

## 我们考虑过但拒绝的想法

- 配置Consul使之绑定所有接口。这将会使HTTP和CLI RPC 端口项外网开放除非我们配置iptables规则来阻止外部主机的访问。而且我们必须确保服务容器知道它们的宿主机IP地址以便和它的Consul agent通讯。
- 配置Consul使之绑定Docker网桥IP地址。这个选择能够正常工作但是：(a) 一般网桥接口是Docker动态分配的；(b) 可能存在多个网桥接口；(c) 容器必须清除选择的网桥接口；(d) Consul agent和dnsmasq(下面将描述)在Docker engin启动之前将不会启动。我们不想创建任何不必要的依赖
- 给每一个容器安装一个Consul agent。Consul的架构体系期待每个主机IP地址一个agent; 并且在大多数环境里，一个Docker主机有一个可访问IP地址。每个容器运行一个Consul agent会造成过个agent加入Consul网络并且声明负责这台主机，引起集群不稳定。
- 和应用容器分享Consul agent容器的网络。一个容器有且仅有一个网络命名空间。所以如果你的应用程序和Consul agent容器分享网络命名空间，它们之间也将分享网络命名空间。这将剥夺我们使用容器带来的主要好处-网络隔离。

## 虚拟(dummy)接口解决方案

&emsp;&emsp;Linux提供了一个叫做“虚拟接口（dummy interface）”的鲜为人知的网络接口类型。它很像一个回环接口，但是你可以给他分配任何IP，并且你可以创建任意多的虚拟接口（我们只需要一个）。以下是一个例子：

    $ sudo ip link add dummy0 type dummy
    $ sudo ip link set dev dummy0 up
    $ ip link show type dummy
    25: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
        link/ether 2a:bb:3e:f6:50:1c brd ff:ff:ff:ff:ff:ff

&emsp;&emsp;我们该为接口分配什么IP?169.254.1.1是一个不错的选择。169.254.0.0/16网段内的地址是本地连接保留地址，这意味着无论在你的本地网络或者互联网上它们都是不可路由的，并且它们对于分配者来说完全是私有的。（一个例外：亚马逊 EC2使用了一个169.254.169.254地址来获取示例元数据，但是我们的操作不会影响这一功能）

    $ sudo ip addr add 169.254.1.1/32 dev dummy0 
    $ sudo ip link set dev dummy0 up
    $ ip addr show dev dummy0
    25: dummy0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN qlen 1000
        link/ether 2a:bb:3e:f6:50:1c brd ff:ff:ff:ff:ff:ff
        inet 169.254.1.1/32 scope global dummy0
        valid_lft forever preferred_lft forever
        inet6 fe80::28bb:3eff:fef6:501c/64 scope link 
        valid_lft forever preferred_lft forever
    
&emsp;&emsp;每个主机可以使用相同的虚拟接口地址169.254.1.1。这样会大大简化配置，因为你不必为应用程序写脚本来为它确定、提供它需要得IP地址。

## 配置接口

&emsp;&emsp;如果你的linux发行版使用systemd,可以很方便的通过创建两个文件来达到启动时配置虚拟接口。（你可能需要通过你的发行版包管理器来安装systemd-networkd，启用并启动它）

&emsp;&emsp;把下面的内容写到文件`/etc/systemd/network/dummy0.netdev`里:

    [NetDev]
    Name=dummy0
    Kind=dummy

&emsp;&emsp;然后把下面的内容写到文件`/etc/systemd/network/dummy0.network`里
    
    [Match]
    Name=dummy0

    [Network]
    Address=169.254.1.1/32

&emsp;&emsp;运行命令`sudo systemctl restart systemd-networkd`后，dummy0接口应该生成了。

&emsp;&emsp;如果你没有使用`systemd`，查看你的linux发行版文档来学习如何在你的主机创建一个虚拟接口。

## 配置Consul来使用上面的虚拟接口

&emsp;&emsp;接下来让我们配置Consul agent，使它绑定它的HTTP、CLI RPC，和DNS接口到地址169.254.1.1.

&emsp;&emsp;假设angent使用`-config-dir=/etc/consul.d`选项启动。我们可以简单的创建一个文件`/etc/consul.d/interfaces.json`，内容如下，用你的主机IP地址替换`HOST_IP_ADDRESS`变量。

```
{
  "client_addr": "169.254.1.1",
  "bind_addr": "HOST_IP_ADDRESS"
}
```

&emsp;&emsp;做完之后你需要重启Consul agent。

## 配置dnsmasq使用虚拟接口

&emsp;&emsp;dnsmasq是一个非常棒的软件。它可以在你的主机上扮演本地DNS缓存。它极其的可靠并且可以很容易和Consul的DNS服务集成。我们将在我们的服务器上安装它；绑定它到我们的回环接口和虚拟接口；使他传递请求到Consul agent的 `.consul`；在主机和容器上配置`/etc/resolv.conf`来分发DNS请求到它。

&emsp;&emsp;首先，使用你的系统包管理工具（`yum`， `apt-get`等）来安装dnsmasq

&emsp;&emsp;接下来，配置dnsmasq绑定到回环接口和虚拟接口，并且向前传递Consul查询到agent.创建一个文件 `/etc/dnsmasq.d/consul.conf`，内容如下

```
server=/consul/169.254.1.1#8600
listen-address=127.0.0.1
listen-address=169.254.1.1
```
然后重启dnsmasq.

## 组合起来：容器、Consul、DNS

&emsp;&emsp;现在让一切正常运行的关键是确保这些容器和容器内运行的的代码在解析DNS查询的时候指向正确的地址或连接到Consul的HTTP API

&emsp;&emsp;当启动你的Docker容器时，配置它以dnsmasq作为他的解析器

    docker run --dns 169.254.1.1

&emsp;&emsp;由于dnsmasq将传递dns查询到Consul agent，所以容器化应用将能够查询 `.consul`结尾的地址

&emsp;&emsp;Consul API 访问呢？关键是设置两个标准的环境变量： CONSUL_HTTP_ADDR 和 CONSUL_RPC_ADDR。几乎所有标准COnsul客户端库都是用这些值来决定向哪里发送查询。请确认你的代码也使用这些变量--永远不要在你的程序中硬编码Consul地址！

```
sudo docker run --dns 169.254.1.1 \
            -e CONSUL_HTTP_ADDR=169.254.1.1:8500 \
            -e CONSUL_RPC_ADDR=169.254.1.1:8400 ...
```

现在让我们实践一下！

&emsp;&emsp;假设我们有一个叫做`myapp`的已经注册到Consul的服务。我们能够在容器中找到他吗？当然：

```
$ sudo docker run --dns 169.254.1.1 \
                -e CONSUL_HTTP_ADDR=169.254.1.1 \
                -e CONSUL_RPC_ADDR=169.254.1.1 \
                -it \
                myImage:latest /bin/sh

 # curl http://$CONSUL_HTTP_ADDR/v1/catalog/service/myapp?pretty
[
   {
      "ID": "6c542e7f-a68d-4de0-bcc0-7eb6b80b68e3",
      "Node": "vessel",
      "Address": "10.0.0.2",
      "ServiceID": "myapp",
      "ServiceName": "myapp",
      "ServiceTags": [],
      "ServiceAddress": "",
      "ServicePort": 80,
      "ServiceEnableTagOverride": false,
      "CreateIndex": 60,
      "ModifyIndex": 60
    }
]
# dig +short myapp.service.consul
10.0.0.2               
```

&emsp;&emsp;将CONSUL_HTTP_ADDR 和CONSUL_RPC_ADDR设为所有用户shell的默认环境变量是个好主意。你可以简单地通过编辑主机上的 `/etc/environment` 文件，内容如下：

```
# /etc/environment
CONSUL_HTTP_ADDR=169.254.1.1:8500
CONSUL_RPC_ADDR=169.254.1.1:8400
```

## 注册容器

&emsp;&emsp;现在我们已经演示了容器能够访问Consul agent，你可能想要发布他们的服务到Consul注册中心。

&emsp;&emsp;有很多工具可以实现这个需求。我最喜欢的开源工具是[Registrator](https://github.com/gliderlabs/registrator)，可以在[Docker hub](https://hub.docker.com/r/gliderlabs/registrator/)获取。

&emsp;&emsp;让我们安装Registrator并且使用它发布一个容器。首先：

```
sudo docker run -d --name=registrator --net=host \
            --volume=/var/run/docker.sock:/tmp/docker.sock \
            gliderlabes/registrator:latest consul://$CONSUL_HTTP_ADDR
```

&emsp;&emsp;现在，让我们启动一个简单的运行Nginx的容器:

```
sudo docker run -d --name=webservice -e CONSUL_HTTP_ADDR=$CONSUL_HTTP_ADDR \
                                     -e SERVICE_NAME=webservice \
                                     --dns 169.254.1.1 -P nginx:laterst
```

&emsp;&emsp;Registrator将会检测到服务并发布到Consul。（由于 `nginx` 镜像暴露两个端口，Registrator在注册服务到注册中心时将追加 `-80`和`-443`到服务名 `webservice`，你可以改变这一行为，如果你愿意设置[其他环境变量](http://gliderlabs.com/registrator/latest/user/services/)）

```
$ sudo docker logs registrator
2017/02/17 22:50:52 added: cd09c82f01ba vessel:webservice:443
2017/02/17 22:50:52 added: cd09c82f01ba vessel:webservice:80

$ curl http://$CONSUL_HTTP_ADDR/v1/catalog/service/webservice-80?pretty
[
    {
        "ID": "6c542e7f-a68d-4de0-bcc0-7eb6b80b68e3",
        "Node": "vessel",
        "Address": "10.0.0.2",
        "ServiceID": "vessel:webservice:80",
        "ServiceName": "webservice-80",
        "ServiceTags": [],
        "ServiceAddress": "",
        "ServicePort": 32772,
        "ServiceEnableTagOverride": false,
        "CreateIndex": 496,
        "ModifyIndex": 496
    }
]
```

&emsp;&emsp;当容器停止时，Registrator会自动从Consul注册中心移除它。

## 结论

&emsp;&emsp;使用虚拟借口，我们可以避免复杂的配置和困难让Docker主机建立Consul agent。

&emsp;&emsp;使用Registrator，我们可以简单的发布Docker容器到Consul。