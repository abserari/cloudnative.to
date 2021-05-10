---
title: "Kubernetes Controller Endpoint Controller 架构设计源码阅读"
date: 2021-05-10T13:04:05+08:00
draft: false
image: "/images/blog/kubernetes-controller.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 Endpoint Controller 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "Controller"]
categories: ["kubernetes"]
keywords: ["Kubernetes","Controller"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---

大家好，我是杨鼎睿，这一次给大家带来的是 Endpoint Controller 部分的源码阅读。

<a name="y7LVx"></a>
## ResourceEventHandler for Service
<a name="Oa9c8"></a>
### Selector Cache

![endpoint-controller-service-cache.svg](9.png)

EndpointController 在收到 v1.Service 资源变更时，将服务 Spec 中指定的 Selector 对象，保存在一个 map 中。该 map 的 key 通过 DeletionHandlingMetaNamespaceKeyFunc 方法生成，如下所示：

![image.png](10.png)

使用的 MetaNamespaceKeyFunc 如下所示：

![image.png](11.png)

最后，将生成的 key 放入 EndpointController 的工作队列，待后续处理。附上 Kubernetes 官方的 Service 对象配置示例。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
<a name="nn98h"></a>
### Synchronize Service

![endpoint-controller-sync-service.svg](12.png)

上图为 syncService 完整的流程。第一步，先根据通过 Service 获取的 Key 值，还原 Namespace 与 Name，这两个值是后续向 API Server 查询的核心依据。



- 服务删除，那么通知 API Server 删除该服务相关的 Endpoint
- 如果是创建、更新服务
   - 从 API Server 根据 (Namespace, Name) 拉取与现有与 Service.Spec.Selector 相关的 Pod 列表
   - 根据 Pod 列表获取子网地址列表
   - 获取当前与 Service 关联的 Endpoints
   - DeepCopy 返回的 Endpoints 对象，并更新 Copy 的 Endpoints 的子网地址列表
   - 使用新 Endpoints，通知 API Server 创建或更新 Endpoints 资源
   - 创建/更新过程发生错误时，记录事件



<a name="0oJNy"></a>
#### Subnets
从 API Server 获取的全部 Pod 都会做如下处理来生成 EndpointAddress，以下两种情况下，Pod 被跳过，继续处理下个 Pod：

- Pod.PodStatus.IP 长度为 0
- 没有设置 tolerateUnreadyEndpoints 且 Pod 将被删除



如果设置了 IPv6DualStack，则根据 Service.Spec.ClusterIP 的类型（IPv4 或 IPv6），从 Pod.PodStatus.PodIPs 中存在同类型的地址，找到即刻返回。如果同类型地址不存在，则报错。

![endpoint-controller-subnet.svg](13.png)



<a name="jCp7r"></a>
#### Set Hostname
获取到 IP 地址的 EndpointAddress 结构，会根据下图条件，设置 EndpointAddress 结构的 Hostname 域。

![endpoint-controller-set-hostname.svg](14.png)



<a name="iyIXW"></a>
#### EndpointSubset
生成 EndpointAddress 后，根据 Service.ServiceSpec.Ports 配置，生成 EndpointSubset 列表，并存入全局的列表中。

- 如果设置了 tolerateUnreadyEndpoints 或当前遍历的 Pod 处于 Ready 状态，Ready 计数 +1
- 如果不满足上述情况，且当前遍历的 Pod 应该归属于 Endpoint，那么 Unready 计数 +1

![endpoint-controller-subnet-port.svg](15.png)



<a name="Rdava"></a>
## ResourceEventHandler for Pod
<a name="7FtyV"></a>
### On Add Pod
![endpoint-controller-on-add-pod.svg](16.png)

从 API Server 获取 Pod.Namespace 下所有 Service。遍历 Service，如果缓存中没有该 Service 存在，更新缓存。从缓存中获取 Service 使用的 Selector，并与 Pod 的 Labels 比对，如果一致，说明该服务受到 Pod 影响，添加进队列等待处理。



<a name="SNwIP"></a>
### On Update Pod
![endpoint-controller-on-update-pod.svg](17.png)



<a name="4zbuZ"></a>
### On Delete Pod
删除的 Pod 无法从 API Server 上获取，那么从传入的 obj 中获取即可。找到被删除节点后，处理方式就和添加 Pod 一样了。

![image.png](18.png)

获取 Pod 对象的方法如下

![image.png](19.png)

[6] Controllers
- [Controllers Queue](/blog/kubernetes-controller-queue/)
- [Controllers](/blog/kubernetes-controller-controllers/)
- [Endporint Controller](/blog/kubernetes-controller-endpoint-controller/)
- [Namespace Controller](/blog/kubernetes-controller-namespace-controller/)
- [Node Controllers](/blog/kubernetes-controller-node-controllers/)
