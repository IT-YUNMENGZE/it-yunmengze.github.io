---
layout: post
title:  "Kubernetes的CNI网络插件"
date:   2022-04-26 10:00:00
categories: 容器与Kubernetes
tags:  Kubernetes 容器网络 CNI
author: yunmengze
---

* content
{:toc}

### CNI0

上一篇文章介绍了 Flannel 实现容器跨主机网络通信的两种实现方法—UDP和VXLAN ，它们都有着一个共性，那就是用户的容器都连接在 docker0 网桥上。而网络插件则在宿主机上创建了一个特殊的设备（UDP 模式创建的是 TUN 设备，VXLAN 模式创建的则是 VTEP 设备），docker0 与这个设备之间，通过 IP 转发（路由表）进行协作。然后，网络插件通过某种方法，把不同宿主机上的特殊设备连通，从而达到容器跨主机通信的目的。

实际上，上面这个流程，也正是 Kubernetes 对容器网络的主要处理方法。只不过，Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。这个网桥的名字就叫作：**CNI 网桥**，它在宿主机上的设备名称默认是：**cni0**。







以 Flannel 的 VXLAN 模式为例，在 Kubernetes 环境里，它的工作方式跟上篇文章中没有任何不同。只不过，docker0 网桥被替换成了 CNI 网桥而已，如下所示：

![img](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/cni.jpg)

Kubernetes 之所以要设置这样一个与 docker0 网桥功能几乎一样的 CNI 网桥，主要原因包括两个方面：

* 一方面，Kubernetes 项目并没有使用 Docker 的网络模型（CNM），所以它并不希望、也不具备配置 docker0 网桥的能力；
* 另一方面，这与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相关。

我们知道，Kubernetes 创建一个 Pod 的第一步，就是创建并启动一个 Infra 容器，用来“hold”住这个 Pod 的 Network Namespace。

所以，CNI 的设计思想，就是：**Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。**

### CNI的实现

要实现一个给Kubernetes使用的容器网络方案，需要完成两部分工作，以Flannel项目为例：

1. **首先需要实现这个网络方案本身。**这一部分需要编写的是 flanneld 进程里的主要逻辑。比如，创建和配置 flannel.1 设备、配置宿主机路由、配置 ARP 和 FDB 表里的信息等等；

2. **然后，实现该网络方案对应的 CNI 插件。**这一部分主要需要做的，就是配置 Infra 容器里面的网络栈，并把它连接在 CNI 网桥上。

由于 Flannel 项目对应的 CNI 插件已经被内置了，所以它无需再单独安装。而对于 Weave、Calico 等其他项目来说，我们就必须在安装插件的时候，把对应的 CNI 插件的可执行文件放在 /opt/cni/bin/ 目录下。

接下来，需要在宿主机上安装 flanneld（网络方案本身）。而在这个过程中，flanneld 启动后会在每台宿主机上生成它对应的 CNI 配置文件（它其实是一个 ConfigMap），从而告诉 Kubernetes，这个集群要使用 Flannel 作为容器网络方案。

这时候，dockershim 会把这个 CNI 配置文件加载起来，并且把列表里的第一个插件、也就是 flannel 插件，设置为默认插件。

> 在 Kubernetes 中，处理容器网络相关的逻辑并不会在 kubelet 主干代码里执行，而是会在具体的 CRI（Container Runtime Interface，容器运行时接口）实现里完成。对于 Docker 项目来说，它的 CRI 实现叫作 dockershim。

我们以kubelet创建Pod为例，来讲解CNI插件的工作流程：

1. kubelet创建Pod，第一个创建的容器是Infra容器，在这一步，dockershim 就会先调用 Docker API 创建并启动 Infra 容器；

2. dockershim 执行 SetUpPod() 方法，该方法作用是为 CNI 准备参数，然后调用 CNI 插件（/opt/cni/bin/flannel) 为Infra配置网络；其中参数来源于：

   ​	1）dockershim设置的一组CNI环境变量；

   ​	2）dockershim 从 CNI 配置文件里（有flanneld启动后生成，类型为configmap）加载到的、默			认插件的配置信息（network configuration)。dockershim 会把 Network Configuration 以 			JSON 数据的格式，通过标准输入（stdin）的方式传递给 Flannel CNI 插件。

3. 参数准备好后，Flannel CNI 插件就会调用 CNI bridge 插件，也就是执行：/opt/cni/bin/bridge 二进制文件；

4. CNI bridge 插件就可以“代表”Flannel，进行“将容器加入到 CNI 网络里”这一步操作。首先，CNI bridge 插件会在宿主机上检查 CNI 网桥是否存在。如果没有的话，那就创建它。接下来，CNI bridge 插件会通过 Infra 容器的 Network Namespace 文件，进入到这个 Network Namespace 里面，然后创建一对 Veth Pair 设备。紧接着，它会把这个 Veth Pair 的其中一端“移动”到宿主机上。随后，CNI bridge 插件把宿主机上的veth设备连接在 CNI 网桥上。

5. CNI bridge 插件会调用 CNI ipam 插件，从 ipam.subnet 字段规定的网段里为容器分配一个可用的 IP 地址。然后，CNI bridge 插件就会把这个 IP 地址添加在容器的 eth0 网卡上，同时为容器设置默认路由；

6. 在执行完上述操作之后，CNI 插件会把容器的 IP 地址等信息返回给 dockershim，然后被 kubelet 添加到 Pod 的 Status 字段。

### 总结
本篇文章讲解了 Kubernetes 中 CNI 网络的实现原理。根据这个原理，其实能容易理解所谓的“Kubernetes 网络模型”了：

1. 所有容器都可以直接使用 IP 地址与其他容器通信，而无需使用 NAT。
2. 所有宿主机都可以直接使用 IP 地址与所有容器通信，而无需使用 NAT。反之亦然。
3. 容器自己“看到”的自己的 IP 地址，和别人（宿主机或者容器）看到的地址是完全一样的。

可以看到，这个网络模型，其实可以用一个字总结，那就是“通”。**容器与容器之间要“通”，容器与宿主机之间也要“通”。并且，Kubernetes 要求这个“通”，还必须是直接基于容器和宿主机的 IP 地址来进行的。**

