---
layout: post
title:  "Service 与 Ingress"
date:   2022-05-02 14:00:00
categories: 容器与Kubernetes
tags:  Kubernetes Service Ingress
author: yunmengze
---

* content
{:toc}

### Ingress介绍

上篇文章讲解了将 Service 暴露给外界的三种方法。其中有一个叫作 LoadBalancer 类型的 Service，它会为你在 Cloud Provider（比如：Google Cloud 或者 OpenStack）里创建一个与该 Service 对应的负载均衡服务。但是，由于每个 Service 都要有一个负载均衡服务，所以这个做法实际上既浪费成本又高。

用户其实更希望看到 Kubernetes 为他们内置一个全局的负载均衡器。然后通过用户访问的 URL，把请求转发给不同的后端 Service。**这种全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的 Ingress 服务**。所以，Ingress 的功能其实很容易理解：**所谓 Ingress，就是 Service 的“Service”。**

举个例子，假如现在有这样一个站点：https://cafe.example.com。其中，https://cafe.example.com/coffee，对应的是“咖啡点餐系统”。https://cafe.example.com/tea 对应的则是“茶水点餐系统”。这两个系统，分别由名叫 coffee 和 tea 这样两个 Deployment 来提供服务。那么就可以使用 Kubernetes 的 Ingress 来创建一个统一的负载均衡器，从而实现当用户访问不同的域名时，能够访问到不同的 Deployment 。

上述功能，用 Ingress 对象来描述，如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

在上面这个名叫 cafe-ingress.yaml 文件中，最值得关注的是 rules 字段。在 Kubernetes 里，这个字段叫作：**IngressRule**。

IngressRule 的 Key 就叫做：host。它必须是一个标准的域名格式（Fully Qualified Domain Name）的字符串，而不能是 IP 地址。

> 备注：Fully Qualified Domain Name 的具体格式，可以参考[RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986)标准。

而 host 字段定义的值，就是这个 Ingress 的入口。这也就意味着，**当用户访问 cafe.example.com 的时候，实际上访问到的是这个 Ingress 对象**。这样，Kubernetes 就能使用 IngressRule 来对你的请求进行下一步转发。

而接下来 IngressRule 规则的定义，则依赖于 path 字段。可以简单地理解为，这里的每一个 path 都对应一个后端 Service。所以在上面的例子中定义了两个 path，它们分别对应 coffee 和 tea 这两个 Deployment 的 Service（即：coffee-svc 和 tea-svc）。

**不难看到，所谓 Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象。**

一个 Ingress 对象的主要内容，实际上就是一个“反向代理”服务（比如：Nginx）的配置文件的描述。而这个代理服务对应的转发规则，就是 IngressRule。这就是为什么在每条 IngressRule 里，需要有一个 host 字段来作为这条 IngressRule 的入口，然后还需要有一系列 path 字段来声明具体的转发策略。这其实跟 Nginx、HAproxy 等项目的配置文件的写法是一致的。

而有了 Ingress 这样一个统一的抽象，Kubernetes 的用户就无需关心 Ingress 的具体细节了。在实际的使用中，你只需要从社区里选择一个具体的 Ingress Controller，把它部署在 Kubernetes 集群里即可。然后，这个 Ingress Controller 会根据你定义的 Ingress 对象，提供对应的代理能力。目前，业界常用的各种反向代理项目，比如 Nginx、HAProxy、Envoy、Traefik 等，都已经为 Kubernetes 专门维护了对应的 Ingress Controller。

### 总结
Ingress 实际上就是 Kubernetes 对“反向代理”的抽象。目前，Ingress 只能工作在七层，而 Service 只能工作在四层。所以当你想要在 Kubernetes 里为应用进行 TLS 配置等 HTTP 相关的操作时，都必须通过 Ingress 来进行。

当然，正如同很多负载均衡项目可以同时提供七层和四层代理一样，将来 Ingress 的进化中，也会加入四层代理的能力。这样，一个比较完善的“反向代理”机制就比较成熟了。而 Kubernetes 提出 Ingress 概念的原因其实也非常容易理解，有了 Ingress 这个抽象，用户就可以根据自己的需求来自由选择 Ingress Controller。比如，如果你的应用对代理服务的中断非常敏感，那么你就应该考虑选择类似于 Traefik 这样支持“热加载”的 Ingress Controller 实现。更重要的是，一旦你对社区里现有的 Ingress 方案感到不满意，或者你已经有了自己的负载均衡方案时，你只需要做很少的编程工作，就可以实现一个自己的 Ingress Controller。在实际的生产环境中，Ingress 带来的灵活度和自由度，对于使用容器的用户来说，其实是非常有意义的。