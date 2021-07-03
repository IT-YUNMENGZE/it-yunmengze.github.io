---
layout: post
title:  "Kubernetes的介绍"
date:   2020-06-10 14:36:54
categories: 容器与Kubernetes
tags:  Kubernetes 云原生 容器
author: yunmengze
---

* content
{:toc}

### 1. 云原生的概念

* 云原生（Cloud native）的愿景是**应用生在云上，长在云上**，希望能够将云的能力发挥到极致。
* 云原生为用户指定了一条低心智负担的、敏捷的、能够以可扩展、可复制的方式最大化地利用云的能力、发挥云的价值的最佳路径。
* 云原生基金会 —— CNCF，CNCF 有一张云原生全景图（https://github.com/cncf/landscape）





### 2. 云原生的两个基础理论

* **第一个理论基础是：不可变基础设施**。这一点目前是通过容器镜像来实现的，其含义就是应用的基础设施应该是不可变的，是一个自包含、自描述可以完全在不同环境中迁移的东西；
* **第二个理论基础就是：云应用编排理论。**当前的实现方式就是 Google 所提出来的“容器设计模式”，这也是 Kubernetes 部分的核心内容。



### 3. 容器的基本概念

#### 容器

* 在Linux中，可以通过 ps 等命令看到各式各样的进程，这些进程包括系统自带的服务和用户的应用进程。那么，这些进程都有什么样的特点？
  - 第一，这些进程可以相互看到、相互通信；
  - 第二，它们使用的是同一个文件系统，可以对同一个文件进行读写操作；
  - 第三，这些进程会使用相同的系统资源。
* 这样的三个特点会带来什么问题呢？
  - 因为这些进程能够相互看到并且进行通信，高级权限的进程可以攻击其他进程；
  - 因为它们使用的是同一个文件系统，因此会带来两个问题：这些进程可以对于已有的数据进行增删改查，具有高级权限的进程可能会将其他进程的数据删除掉，破坏掉其他进程的正常运行；此外，进程与进程之间的依赖可能会存在冲突，如此一来就会给运维带来很大的压力；
  - 因为这些进程使用的是同一个宿主机的资源，应用之间可能会存在资源抢占的问题，当一个应用需要消耗大量 CPU 和内存资源的时候，就可能会破坏其他应用的运行，导致其他应用无法正常地提供服务。
* 针对上述的三个问题，如何为进程提供一个独立的运行环境呢？
  * 针对不同进程使用同一个文件系统所造成的问题而言，Linux 和 Unix 操作系统可以通过 **chroot 系统调用将子目录变成根目录，达到视图级别的隔离**；**进程在 chroot 的帮助下可以具有独立的文件系统，对于这样的文件系统进行增删改查不会影响到其他进程；**
  * **因为进程之间相互可见并且可以相互通信，使用 Namespace 技术来实现进程在资源的视图上进行隔离。在 chroot 和 Namespace 的帮助下，进程就能够运行在一个独立的环境下了；**
  * 但在独立的环境下，进程所使用的还是同一个操作系统的资源，一些进程可能会侵蚀掉整个系统的资源。**为了减少进程彼此之间的影响，可以通过 Cgroup 来限制其资源使用率**，设置其能够使用的 CPU 以及内存量。
* 综上，我们将实现以上条件的进程集合定义为容器。
  * 其实，**容器就是一个视图隔离、资源可限制、独立文件系统的进程集合**。
  * 所谓“视图隔离”就是能够看到部分进程以及具有独立的主机名等；控制资源使用率则是可以对于内存大小以及 CPU 使用个数等进行限制。容器就是一个进程集合，它将系统的其他资源隔离开来，具有自己独立的资源视图。
  * 容器具有一个独立的文件系统，因为使用的是系统的资源，**所以在独立的文件系统内不需要具备内核相关的代码或者工具，我们只需要提供容器所需的二进制文件、配置文件以及依赖即可。**只要容器运行时所需的文件集合都能够具备，那么这个容器就能够运行起来。

#### 镜像

* 综上所述，**我们将这些容器运行时所需要的所有的文件集合称之为容器镜像。**
* 通常情况下，我们会采用 Dockerfile 来构建镜像，这是因为 Dockerfile 提供了非常便利的语法糖，能够帮助我们很好地描述构建的每个步骤。当然，每个构建步骤都会对已有的文件系统进行操作，这样就会带来文件系统内容的变化，我们将这些变化称之为 changeset。**当我们把构建步骤所产生的变化依次作用到一个空文件夹上，就能够得到一个完整的镜像。**
* 容器就是和系统其它部分隔离开来的进程集合，这里的其他部分包括进程、网络资源以及文件系统等。而镜像就是容器所需要的所有文件集合，其具备“一次构建、到处运行”的特点。

#### 容器的生命周期

* 容器是一组具有隔离特性的进程集合，在使用 docker run 的时候会选择一个镜像来提供独立的文件系统并指定相应的运行程序。这里指定的运行程序称之为 initial 进程**，这个 initial 进程启动的时候，容器也会随之启动，当 initial 进程退出的时候，容器也会随之退出。**
* 因此，可以认为容器的生命周期和 initial 进程的生命周期是一致的。当然，因为容器内不只有这样的一个 initial 进程，**initial 进程本身也可以产生其他的子进程或者通过 docker exec 产生出来的运维操作，也属于 initial 进程管理的范围内。当 initial 进程退出的时候，所有的子进程也会随之退出**，这样也是为了防止资源的泄漏。
* 应用里面的程序往往是有状态的，其可能会产生一些重要的数据，**当一个容器退出被删除之后，数据也就会丢失了，**这对于应用方而言是不能接受的，所以需要将容器所产生出来的重要数据持久化下来。容器能够直接将数据持久化到指定的目录上，这个目录就称之为数据卷。数据卷有一些特点，其中非常明显的就是**数据卷的生命周期是独立于容器的生命周期的**，也就是说容器的创建、运行、停止、删除等操作都和数据卷没有任何关系，因为它是一个特殊的目录，是用于帮助容器进行持久化的。简单而言，我们会将数据卷挂载到容器内，这样一来容器就能够将数据写入到相应的目录里面了，而且容器的退出并不会导致数据的丢失。
* 通常情况下，数据卷管理主要有两种方式：
  * 第一种是通过 bind 的方式，直接将宿主机的目录直接挂载到容器内；这种方式比较简单，但是会带来运维成本，因为其依赖于宿主机的目录，需要对于所有的宿主机进行统一管理。
  * 第二种是将目录管理交给运行引擎。

#### 容器 vs 传统虚拟机VM

![image-20210703144108559](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-1.png)

* VM 利用 Hypervisor 虚拟化技术来模拟 CPU、内存等硬件资源，这样就可以在宿主机上建立一个 Guest OS，这是常说的安装一个虚拟机。

* 每一个 Guest OS 都有一个独立的内核，比如 Ubuntu、CentOS 甚至是 Windows 等，在这样的 Guest OS 之下，每个应用都是相互独立的，VM 可以提供一个更好的隔离效果。但这样的隔离效果需要付出一定的代价，因为需要把一部分的计算资源交给虚拟化，这样就很难充分利用现有的计算资源，并且每个 Guest OS 都需要占用大量的磁盘空间，比如 Windows 操作系统的安装需要 10~30G 的磁盘空间，Ubuntu 也需要 5~6G，同时这样的方式启动很慢。正是因为虚拟机技术的缺点，催生出了容器技术。

* 容器是针对于进程而言的，因此无需 Guest OS，只需要一个独立的文件系统提供其所需要文件集合即可。所有的文件隔离都是进程级别的，因此启动时间快于 VM，并且所需的磁盘空间也小于 VM。当然了，进程级别的隔离并没有想象中的那么好，隔离效果相比 VM 要差很多。

  总体而言，容器和 VM 相比，各有优劣，因此容器技术也在向着强隔离方向发展。



###  4. Kubernetes的核心概念

Kubernetes 是一个自动化的容器编排平台，它负责应用的部署、应用的弹性以及应用的管理，这些都是基于容器的。

#### k8s的核心功能

* 服务发现与负载均衡；
* 容器的自动装箱，我们也会把它叫做 scheduling，就是“调度”，即把一个容器放到一个集群的某一个机器上；
* 自动化的容器的恢复，在一个集群中，经常会出现宿主机的问题或者说是 OS 的问题，导致容器本身的不可用，Kubernetes 会自动地对这些不可用的容器进行恢复；
* 应用的自动发布与应用的回滚，以及与应用相关的配置密文的管理；
* 对于 job 类型任务，Kubernetes 可以去做批量的执行；
* 为了让这个集群、这个应用更富有弹性，Kubernetes 支持水平的伸缩。

##### 调度

Kubernetes 可以把用户提交的容器放到 Kubernetes 管理的集群的某一台节点上去。Kubernetes 的调度器是执行这项能力的组件，它会观察正在被调度的这个容器的大小、规格。比如说它所需要的 CPU以及它所需要的 memory，然后在集群中找一台相对比较空闲的机器来进行一次 placement，也就是一次放置的操作。

##### 自动修复

Kubernetes 有一个节点健康检查的功能，它会监测这个集群中所有的宿主机，当宿主机本身出现故障，或者软件出现故障的时候，这个节点健康检查会自动对它进行发现。随后 Kubernetes 会把运行在这些失败节点上的容器进行自动迁移，迁移到一个正在健康运行的宿主机上，来完成集群内容器的一个自动恢复。

##### 水平伸缩

Kubernetes 有业务负载检查的能力，它会监测业务上所承担的负载，如果这个业务本身的 CPU 利用率过高，或者响应时间过长，它可以对这个业务进行一次扩容。

### 5. Kubernetes 的架构

Kubernetes 架构是一个比较典型的二层架构和 server-client 架构。Master 作为中央的管控节点，会去与 Node 进行一个连接。所有 UI、CLI等这些 user 侧的组件，只会和 Master 进行连接，把希望的状态或者想执行的命令下发给 Master，Master 会把这些命令或者状态下发给相应的节点，进行最终的执行。

![image-20210703150008413](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-2.png)

Kubernetes 的 Master 包含四个主要的组件：API Server、Controller、Scheduler 以及 etcd。如下图所示：

![image-20210703150053011](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-3.png)

* **API Server**：顾名思义是用来处理 API 操作的，**Kubernetes 中所有的组件都会和 API Server 进行连接，组件与组件之间一般不进行独立的连接，都依赖于 API Server 进行消息的传送；**
* **Controller**：是控制器，它用来完成对集群状态的一些管理。自动对容器进行修复和自动进行水平扩张，都是由 Kubernetes 中的 Controller 来进行完成的；
* **Scheduler**：是调度器，顾名思义就是完成调度的操作，就是我们刚才介绍的第一个例子中，把一个用户提交的 Container，依据它对 CPU、对 memory 请求的大小，找一台合适的节点，进行放置；
* **etcd**：是一个分布式的存储系统，用于持久化存储K8s集群的配置和状态。API Server 中所需要的这些元信息都被放置在 etcd 中，etcd 本身是一个高可用系统，通过 etcd 保证整个 Kubernetes 的 Master 组件的高可用性。

补充： API Server 本身在结构上是一个可以水平扩展的一个部署组件；Controller 是一个可以进行热备的部署组件，但只有一个 active；同样，Scheduler 也只有一个 active，也是可以进行热备。

#### Node

Kubernetes 的 Node 是真正运行业务负载的，每个业务负载会以 Pod 的形式运行，Node可以是真实的一台物理主机或是虚拟机，Node可分为 Master Node 和 Worker Node 。

![img](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-4.png)

一个 Pod 中运行的一个或者多个容器，真正去运行这些 Pod 的组件的是叫做 **kubelet**，也就是 Node 上最为关键的组件，它通过 API Server 接收到所需要 Pod 运行的状态，然后提交到我们下面画的这个 Container Runtime 组件中。

在 OS 上去创建容器所需要运行的环境，最终把容器或者 Pod 运行起来，也需要对存储跟网络进行管理。Kubernetes 并不会直接进行网络存储的操作，他们会靠 Storage Plugin 或者是Network Plugin 来进行操作。用户自己或者云厂商都会去写相应的 Storage Plugin 或者 Network Plugin，去完成存储操作或网络操作。

在 Kubernetes 自己的环境中，也会有 Kubernetes 的 Network，它是为了提供 Service network 来进行搭网组网的。真正完成 service 组网的组件的是 **Kube-proxy**，它是利用了 iptables 的能力来进行组建 Kubernetes 的 Network，就是 cluster network（集群网络），以上就是 Node 上面的四个组件。

Kubernetes 的 Node 并不会直接和 user 进行 interaction，它的 interaction 只会通过 Master。而 User 是通过 Master 向节点下发这些信息的。Kubernetes 每个 Node 上，都会运行我们刚才提到的这几个组件。

#### Pod

Pod 是 Kubernetes 的一个**最小调度以及资源单元**。用户可以通过 Kubernetes 的 Pod API 生产一个 Pod，让 Kubernetes 对这个 Pod 进行调度，也就是把它放在某一个 Kubernetes 管理的节点上运行起来。一个 Pod 简单来说是对一组容器的抽象，它里面会包含一个或多个容器。

在 Pod 里面，我们也可以去定义容器所需要运行的方式。比如说运行容器的 Command，以及运行容器的环境变量等等。Pod 这个抽象也给这些容器提供了一个共享的运行环境，它们会共享同一个网络环境，这些容器可以用 localhost 来进行直接的连接。而 Pod 与 Pod 之间，是互相有 isolation 相隔离的。

![img](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-5.png)

用户可以通过 UI 或者 CLI 提交一个 Pod 给 Kubernetes 进行部署，这个 Pod 请求首先会通过 CLI 或者 UI 提交给 Kubernetes API Server，下一步 API Server 会把这个信息写入到它的存储系统 etcd，之后 Scheduler 会通过 API Server 的 watch 或者叫做 notification 机制得到这个信息：有一个 Pod 需要被调度。

这个时候 Scheduler 会根据它的内存状态进行一次调度决策，在完成这次调度之后，它会向 API Server report 说：“OK！这个 Pod 需要被调度到某一个节点上。”

这个时候 API Server 接收到这次操作之后，会把这次的结果再次写到 etcd 中，然后 API Server 会通知相应的节点进行这次 Pod 真正的执行启动。相应节点的 kubelet 会得到这个通知，kubelet 就会去调 Container runtime 来真正去启动配置这个容器和这个容器的运行环境，去调度 Storage Plugin 来去配置存储，Network Plugin 去配置网络。

这个例子我们可以看到：这些组件之间是如何相互沟通相互通信，协调来完成一次Pod的调度执行操作的。

#### Volume

Volume 就是卷的概念，它是用来管理 Kubernetes 存储的，是用来声明在 Pod 中的容器可以访问文件目录的，一个卷可以被挂载在 Pod 中一个或者多个容器的指定路径下面。

而 Volume 本身是一个抽象的概念，一个 Volume 可以去支持多种的后端的存储。比如说 Kubernetes 的 Volume 就支持了很多存储插件，它可以支持本地的存储，可以支持分布式的存储，比如说像 ceph，GlusterFS ；它也可以支持云存储，比如说阿里云上的云盘、AWS 上的云盘、Google 上的云盘等等。

![image-20210703150844251](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-6.png)

#### Deployment

Deployment 是在 Pod 这个抽象上更为上层的一个抽象，它可以定义一组 Pod 的副本数目、以及这个 Pod 的版本。一般大家用 Deployment 这个抽象来做应用的真正的管理，而 Pod 是组成 Deployment 最小的单元。

Kubernetes 是通过 Controller，也就是我们刚才提到的控制器去维护 Deployment 中 Pod 的数目，它也会去帮助 Deployment 自动恢复失败的 Pod。

比如说我可以定义一个 Deployment，这个 Deployment 里面需要两个 Pod，当一个 Pod 失败的时候，控制器就会监测到，它重新把 Deployment 中的 Pod 数目从一个恢复到两个，通过再去新生成一个 Pod。通过控制器，我们也会帮助完成发布的策略。比如说进行滚动升级，进行重新生成的升级，或者进行版本的回滚。

![image-20210703150935230](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-7.png)

#### Service

Service 提供了一个或者多个 Pod 实例的稳定访问地址。

比如在上面的例子中，我们看到：一个 Deployment 可能有两个甚至更多个完全相同的 Pod。对于一个外部的用户来讲，访问哪个 Pod 其实都是一样的，所以它希望做一次负载均衡，在做负载均衡的同时，我只想访问某一个固定的 VIP，也就是 Virtual IP 地址，而不希望得知每一个具体的 Pod 的 IP 地址。

我们刚才提到，这个 Pod 本身可能 terminal go（终止），如果一个 Pod 失败了，可能会换成另外一个新的。

对一个外部用户来讲，提供了多个具体的 Pod 地址，这个用户要不停地去更新 Pod 地址，当这个 Pod 再失败重启之后，我们希望有一个抽象，把所有 Pod 的访问能力抽象成一个第三方的一个 IP 地址，实现这个的 Kubernetes 的抽象就叫 Service。

实现 Service 有多种方式，Kubernetes 支持 Cluster IP，上面我们讲过的 kuber-proxy 的组网，它也支持 nodePort、 LoadBalancer 等其他的一些访问的能力。

![image-20210703151012514](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-8.png)

#### Namespace

Namespace 是用来做一个集群内部的逻辑隔离的，它包括鉴权、资源管理等。Kubernetes 的每个资源，比如刚才讲的 Pod、Deployment、Service 都属于一个 Namespace，同一个 Namespace 中的资源需要命名的唯一性，不同的 Namespace 中的资源可以重名。

#### Kubernetes的 API

下面我们介绍一下 Kubernetes 的 API 的基础知识。从 high-level 上看，Kubernetes API 是由 **HTTP+JSON** 组成的：**用户访问的方式是 HTTP，访问的 API 中 content 的内容是 JSON 格式的。**

Kubernetes 的 kubectl 也就是 command tool，Kubernetes UI或者有时候用 curl，直接与 Kubernetes 进行沟通时，都是使用 HTTP + JSON 这种形式。

下面有个例子：比如说，对于这个 Pod 类型的资源，它的 HTTP 访问的路径，就是 API，然后是 apiVesion: V1, 之后是相应的 Namespaces，以及 Pods 资源，最终是 Podname，也就是 Pod 的名字。

![img](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-9.png)

如果我们去提交一个 Pod，或者 get 一个 Pod 的时候，它的 content 内容都是用 JSON 或者是 YAML 表达的。上图中有个 yaml 的例子，在这个 yaml file 中，对 Pod 资源的描述也分为几个部分。

第一个部分，一般来讲会是 API 的 version。比如在这个例子中是 V1，它也会描述我在操作哪个资源；比如说我的 kind 如果是 pod，在 Metadata 中，就写上这个 Pod 的名字；比如说 nginx，我们也会给它打一些 label，我们等下会讲到 label 的概念。在 Metadata 中，有时候也会去写 annotation，也就是对资源的额外的一些用户层次的描述。

比较重要的一个部分叫做 Spec，Spec 也就是我们希望 Pod 达到的一个预期的状态。比如说它内部需要有哪些 container 被运行；比如说这里面有一个 nginx 的 container，它的 image 是什么？它暴露的 port 是什么？

当我们从 Kubernetes API 中去获取这个资源的时候，一般来讲在 Spec 下面会有一个项目叫 status，它表达了这个资源当前的状态；比如说一个 Pod 的状态可能是正在被调度、或者是已经 running、或者是已经被 terminates，就是被执行完毕了。

刚刚在 API 之中，我们讲了一个比较有意思的 metadata 叫做“label”，这个 label 可以是一组 Key-Value Pair。

比如下图的第一个 pod 中，label 就可能是一个 color 等于 red，即它的颜色是红颜色。当然你也可以加其他 label，比如说 size: big 就是大小，定义为大的，它可以是一组 label。

**这些 label 是可以被 selector，也就是选择器所查询的。**这个能力实际上跟我们的 SQL类型的 select 语句是非常相似的，比如下图中的三个 Pod 资源中，我们就可以进行 select。name color 等于 red，就是它的颜色是红色的，我们也可以看到，只有两个被选中了，因为只有他们的 label 是红色的，另外一个 label 中写的 color 等于 yellow，也就是它的颜色是黄色，是不会被选中的。

![img](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/8-10.png)

通过 label，kubernetes 的 API 层就可以对这些资源进行一个筛选，那这些筛选也是 kubernetes 对资源的集合所表达默认的一种方式。

例如说，我们刚刚介绍的 Deployment，它可能是代表一组的 Pod，它是一组 Pod 的抽象，一组 Pod 就是通过 label selector 来表达的。当然我们刚才讲到说 service 对应的一组 Pod，就是一个 service 要对应一个或者多个的 Pod，来对它们进行统一的访问，这个描述也是通过 label selector 来进行 select 选取的一组 Pod。

所以可以看到 label 是一个非常核心的 kubernetes API 的概念，我们在接下来的课程中也会着重地去讲解和介绍 label 这个概念，以及如何更好地去使用它。

