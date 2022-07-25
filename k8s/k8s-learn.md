# k8s初认知

## 容器技术介绍

轻量级的容器技术，允许在相同硬件上运行更多数量的组件。容器里运行的进程实际上是运行在宿主机的操作系统上的，只是对于容器内的进程本身而言，它好像是在机器和操作系统上运行的唯一进程，避免了cpu等的额外开销。同时容器技术由于是调用的同一内核，自然会有安全隐患。

## k8s节点

一个 Kubernetes 集群是由一组被称作<font color=#0000FF>节点(node)</font>的机器组成， 这些节点上会运行由 Kubernetes 所管理的容器化应用。 且每个集群至少有一个工作节点。工作节点会托管所谓的 Pods，而 Pod 就是作为应用负载的组件。 控制平面管理集群中的工作节点和 Pods。 为集群提供故障转移和高可用性， 这些控制平面一般跨多主机运行，而集群也会跨多个节点运行。

<figure class=diagram-large><img src=https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg alt="Kubernetes 的组件"><figcaption><p>Kubernetes 集群的组件</p></figcaption>

k8s的节点主要分成两种类型，一种是承载控制面板的主节点，一种是运行用户实际部署应用的工作节点。

### 控制面板

控制平面组件会为集群做出全局决策，比如资源的调度。 以及检测和响应集群事件，例如当不满足部署的 `replicas` 字段时， 要启动新的 pod。
控制平面组件可以在集群中的任何节点上运行。 然而，为了简单起见，设置脚本通常会在同一个计算机上启动所有控制平面组件， 并且不会在此计算机上运行用户容器。

- kube-apiserver负责控制面板其他组件以及工作节点通信: API 服务器是 Kubernetes 控制平面的组件， 该组件负责公开了 Kubernetes API，负责处理接受请求的工作。 API 服务器是 Kubernetes 控制平面的前端。Kubernetes API 服务器的主要实现是 kube-apiserver。 kube-apiserver 设计上考虑了水平扩缩，也就是说，它可通过部署多个实例来进行扩缩。 你可以运行 kube-apiserver 的多个实例，并在这些实例之间平衡流量。
- kube-scheduler: kube-scheduler 是控制平面的组件， 负责监视新创建的、未指定运行节点（node）的 Pods， 并选择节点来让 Pod 在上面运行。
  调度决策考虑的因素包括单个 Pod 及 Pods 集合的资源需求、软硬件及策略约束、 亲和性及反亲和性规范、数据位置、工作负载间的干扰及最后时限。
- kube-controller-manager: kube-controller-manager 是控制平面的组件， 负责运行控制器进程。
  从逻辑上讲， 每个控制器都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在同一个进程中运行。
  这些控制器包括：
  - 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
  - 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
  - 端点控制器（Endpoints Controller）：填充端点（Endpoints）对象（即加入 Service 与 Pod）
  - 服务帐户和令牌控制器（Service Account & Token Controllers）：为新的命名空间创建默认帐户和 API 访问令牌
- etcd 持久化存储集群配置: etcd 是兼顾一致性与高可用性的键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库。你的 Kubernetes 集群的 etcd 数据库通常需要有个备份计划。
- cloud-controller-manager: cloud-controller-manager 是指嵌入特定云的控制逻辑之 控制平面组件。 cloud-controller-manager 允许你将你的集群连接到云提供商的 API 之上， 并将与该云平台交互的组件同与你的集群交互的组件分离开来。
  cloud-controller-manager 仅运行特定于云平台的控制器。 因此如果你在自己的环境中运行 Kubernetes，或者在本地计算机中运行学习环境， 所部署的集群不需要有云控制器管理器。
  与 kube-controller-manager 类似，cloud-controller-manager 将若干逻辑上独立的控制回路组合到同一个可执行文件中， 供你以同一进程的方式运行。 你可以对其执行水平扩容（运行不止一个副本）以提升性能或者增强容错能力。
  下面的控制器都包含对云平台驱动的依赖：
  - 节点控制器（Node Controller）：用于在节点终止响应后检查云提供商以确定节点是否已被删除
  - 路由控制器（Route Controller）：用于在底层云基础架构中设置路由
  - 服务控制器（Service Controller）：用于创建、更新和删除云提供商负载均衡器

### 工作节点

节点组件会在每个节点上运行，负责维护运行的 Pod 并提供 Kubernetes 运行环境。

- docker容器 运行应用的容器: 容器运行环境是负责运行容器的软件。
  Kubernetes 支持许多容器运行环境，例如 Docker、 containerd、 CRI-O 以及 Kubernetes CRI (容器运行环境接口) 的其他任何实现。
- kubelet 和API服务器通信，管理所在节点的容器: kubelet 会在集群中每个节点（node）上运行。 它保证容器（containers）都运行在 Pod 中。
  kubelet 接收一组通过各类机制提供给它的 PodSpecs， 确保这些 PodSpecs 中描述的容器处于运行状态且健康。 kubelet 不会管理不是由 Kubernetes 创建的容器。
- kube-proxy 负责组件之间的负载均衡网络流量: kube-proxy 是集群中每个节点（node）所上运行的网络代理， 实现 Kubernetes 服务（Service） 概念的一部分。
  kube-proxy 维护节点上的一些网络规则， 这些网络规则会允许从集群内部或外部的网络会话与 Pod 进行网络通信。
  如果操作系统提供了可用的数据包过滤层，则 kube-proxy 会通过它来实现网络规则。 否则，kube-proxy 仅做流量转发。

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