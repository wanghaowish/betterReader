# k8s初认知

## 容器技术介绍

轻量级的容器技术，允许在相同硬件上运行更多数量的组件。容器里运行的进程实际上是运行在宿主机的操作系统上的，只是对于容器内的进程本身而言，它好像是在机器和操作系统上运行的唯一进程，避免了cpu等的额外开销。同时容器技术由于是调用的同一内核，自然会有安全隐患。

## k8s节点

k8s的节点主要分成两种类型，一种是承载控制面板的主节点，一种是运行用户实际部署应用的工作节点。

### 控制面板

- K8sAPI服务器 负责控制面板其他组件以及工作节点通信
- Scheculer 调度应用
- Controller Manager 执行集群级别的功能
- etcd 持久化存储集群配置

### 工作节点

- docker容器 运行应用的容器
- kubelet 和API服务器通信，管理所在节点的容器
- kube-proxy 负责组件之间的负载均衡网络流量

### pod的概念

pod是一组紧密相关的容器，拥有自己的IP主机名、进程等。它包含一个或多个容器，每个容器都运行一个应用进程，所有容器都运行在同一个逻辑机器上。一个node可以有多个pod。

### 服务的概念

pod的IP地址是多变的，服务就是为了解决不断变化的podIP地址的问题。服务的地址是静态的，客户端通过固定IP连接到服务，而不是直连pod。当应用有多个pod在运行时，请求会随机切换到不同的pod，服务作为负载均衡挡在多个pod前面。

### k8s的优势

- 简化应用程序的部署
- 更好的应用硬件 算法保障最优组合
- 健康检查和自我修复 可迁移应用程序
- 自动扩容 k8s监控应用的资源并根据需要自动调整

## pod：运行在k8s中的容器

pod是k8s中的基本构建模块，是一组并置的容器。同一个pod的容器共享相同的IP地址和端口空间，所以同一个pod的容器运行时需要注意端口冲突。容器不该包含多个进程，pod也不该包含多个并不需要运行在同一主机上的容器。

k8s资源三大重要部分：

- metadata包括名称、命名控件、标签及关于该容器的其他信息
- spec包含pod内容的实际说明
- status包含运行中的pod的当前信息

kubectl explain可用来发现可能的API对象字段。可查看每个API对象支持哪些属性，例如：

```bash
$ kubectl explain pod.spec
KIND:     Pod
VERSION:  v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the pod. More info:
     <https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status>

     PodSpec is a description of a pod.

FIELDS:
   ...
   hostPID      <boolean>
     Use the host's pid namespace. Optional: Default to false.
   ...
   volumes      <[]Object>
     List of volumes that can be mounted by containers belonging to the pod.
     More info: <https://kubernetes.io/docs/concepts/storage/volumes>

```

### 标签

`kubectl lable`

标签是一种简单却功能强大的k8s特性。标签是可以附加到资源的任意键值对，用以选择具有该确切标签的资源。更改现有标签需要使用--overwrite选项。

标签选择器 `-l`

### 注解

`kubectl annotate`

注解本质和标签非常相似，也是键值对，但是注解不是为了保存标识信息而存在的，它可以容纳更多的信息，主要用于工具使用。