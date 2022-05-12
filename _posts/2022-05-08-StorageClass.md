---
layout: post
title:  "Kubernetes 存储篇（2）— StorageClass"
date:   2022-05-08 12:00:00
categories: 容器与Kubernetes
tags:  Kubernetes 存储 StorageClass
author: yunmengze
---

* content
{:toc}

### StorageClass 介绍

上篇文章介绍 PV 和 PVC 的时候，提到 PV 这个对象的创建一般是由运维人员完成的。但是，在大规模的生产环境里，这其实是一个非常麻烦的工作。这是因为，一个大规模的 Kubernetes 集群里很可能有成千上万个 PVC，这就意味着运维人员必须得事先创建出成千上万个 PV（PV 与 PVC 是一一对应的）。更麻烦的是，随着新的 PVC 不断被提交，运维人员就不得不继续添加新的、能满足条件的 PV，否则新的 Pod 就会因为 PVC 绑定不到 PV 而失败。在实际操作中，这几乎没办法靠人工做到。所以，Kubernetes 为我们提供了一套可以自动创建 PV 的机制，即：**Dynamic Provisioning**。相比之下，前面人工管理 PV 的方式就叫作 Static Provisioning。









Dynamic Provisioning 机制工作的核心，在于一个名叫 **StorageClass** 的 API 对象。**而 StorageClass 对象的作用，其实就是创建 PV 的模板。**

具体地说，StorageClass 对象会定义如下两个部分内容：

* 第一，PV 的属性。比如，存储类型、Volume 的大小等等。

* 第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。

有了这样两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass 了。然后，Kubernetes 就会调用该 StorageClass 声明的存储插件，创建出需要的 PV。

举个例子，假如我们的 Volume 的类型是 GCE 的 Persistent Disk 的话，运维人员就需要定义一个如下所示的 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

这个 YAML 文件定义了一个名叫 block-service 的 StorageClass。这个 StorageClass 的 provisioner 字段的值是：kubernetes.io/gce-pd，这正是 Kubernetes 内置的 GCE PD 存储插件的名字。而这个 StorageClass 的 parameters 字段，就是 PV 的参数。比如：上面例子里的 type=pd-ssd，指的是这个 PV 的类型是“SSD 格式的 GCE 远程磁盘”。

有了 StorageClass 的 YAML 文件之后，运维人员就可以在 Kubernetes 里创建这个 StorageClass 了：

```yaml
$ kubectl create -f sc.yaml
```

这时候，作为应用开发者，我们只需要在 PVC 里指定要使用的 StorageClass 名字即可，如下所示：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: block-service
  resources:
    requests:
      storage: 30Gi
```

可以看到，在这个 PVC 里添加了一个叫作 storageClassName 的字段，用于指定该 PVC 所要使用的 StorageClass 的名字是：block-service。

以 Google Cloud 为例。当我们通过 kubectl create 创建上述 PVC 对象之后，Kubernetes 就会调用 Google Cloud 的 API，创建出一块 SSD 格式的 Persistent Disk。然后，再使用这个 Persistent Disk 的信息，自动创建出一个对应的 PV 对象。

我们可以一起来实践一下这个过程：

```yaml
$ kubectl create -f pvc.yaml
```

可以看到，创建的 PVC 会绑定一个 Kubernetes 自动创建的 PV，如下所示：

```yaml
$ kubectl describe pvc claim1
Name:           claim1
Namespace:      default
StorageClass:   block-service
Status:         Bound
Volume:         pvc-e5578707-c626-11e6-baf6-08002729a32b
Labels:         <none>
Capacity:       30Gi
Access Modes:   RWO
No Events.
```

而且，通过查看这个**自动创建**的 PV 的属性，可以看到它和 PVC 里声明的存储的属性是一致的，如下所示：

```yaml
$ kubectl describe pv pvc-e5578707-c626-11e6-baf6-08002729a32b
Name:            pvc-e5578707-c626-11e6-baf6-08002729a32b
Labels:          <none>
StorageClass:    block-service
Status:          Bound
Claim:           default/claim1
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        30Gi
...
No events.
```

这个自动创建出来的 PV 的 StorageClass 字段的值，也是 block-service。这是因为，**Kubernetes 只会将 StorageClass 相同的 PVC 和 PV 绑定起来。**

有了 Dynamic Provisioning 机制，运维人员只需要在 Kubernetes 集群里创建出数量有限的 StorageClass 对象就可以了。这就好比，运维人员在 Kubernetes 集群里创建出了各种各样的 PV 模板。这时候，当开发人员提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV。

需要注意的是，StorageClass 并不是专门为了 Dynamic Provisioning 而设计的。比如，在本篇一开始的例子里，PV 和 PVC 里都声明了 storageClassName=manual。而我的集群里，实际上并没有一个名叫 manual 的 StorageClass 对象。这完全没有问题，这个时候 Kubernetes 进行的是 Static Provisioning，但在做绑定决策的时候，它依然会考虑 PV 和 PVC 的 StorageClass 定义。而这么做的好处也很明显：这个 PVC 和 PV 的绑定关系，就完全在用户自己的掌控之中（**PV 和 PVC 指定了相同的 StorageClassName 意味者用户想将两者进行绑定**）。

如果集群开启了名叫 DefaultStorageClass 的 Admission Plugin，它就会为 PVC 和 PV 自动添加一个默认的 StorageClass；**否则，PVC 的 storageClassName 的值就是“”，这也意味着它只能够跟 storageClassName 也是“”的 PV 进行绑定。**

### 总结

PV、PVC 和 StorageClass 的关系可以用如下示意图描述：

![img](https://cdn.jsdelivr.net/gh/IT-YUNMENGZE/ImgDB/blog_img/PVandPVC.png)

从图中我们可以看到，在这个体系中：

* PVC 描述的，是 Pod 想要使用的持久化存储的属性，比如存储的大小、读写权限等。

* PV 描述的，则是一个具体的 Volume 的属性，比如 Volume 的类型、挂载目录、远程存储服务器地址等。

* StorageClass 的作用，则是充当 PV 的模板。并且，只有同属于一个 StorageClass 的 PV 和 PVC 才可以绑定在一起。当然，StorageClass 的另一个重要作用，是指定 PV 的 Provisioner（存储插件）。这时候，如果你的存储插件支持 Dynamic Provisioning 的话，Kubernetes 就可以自动为你创建 PV 了。

因此，容器持久化存储整体流程总结如下：

1. 用户提交请求创建 Pod，Kubernetes 发现这个 Pod 声明使用了 PVC，那就靠 PersistentVolumeController 帮它找一个PV配对。 
2. 没有现成的 PV，就去找对应的 StorageClass，帮它新创建一个PV，然后和PVC完成绑定。 
3. 新创建的PV，还只是一个 API 对象，需要经过“两阶段处理”变成宿主机上的“持久化 Volume”才真正有用： 第一阶段由运行在master上的 AttachDetachController 负责，为这个PV完成 Attach 操作，为宿主机挂载远程磁盘； 第二阶段是运行在每个节点上kubelet组件的内部，把第一步attach的远程磁盘 mount 到宿主机目录。这个控制循环叫VolumeManagerReconciler，运行在独立的Goroutine，不会阻塞kubelet主循环。 
4. 完成这两步，PV 对应的“持久化 Volume”就准备好了，Pod 可以正常启动，将“持久化 Volume”挂载在容器内指定的路径。