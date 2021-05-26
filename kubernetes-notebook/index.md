# Kubernetes 核心概念


## 什么是 Kubernetes

Kubernetes 是一个自动化的容器编排平台，负责应用的部署、弹性以及管理，这些都是基于容器的。可以简称为 `k8s`，是将中间 8 个字母 "ubernete" 替换为 "8" 而导致的一个缩写。

## Kubernetes 的核心功能

* 服务发现与负载均衡；
* 容器的自动装箱，也就是<ruby>调度<rt>scheduling</rt></ruby>，把一个容器放到一个集群的某一个机器上，K8s 会自动做存储的编排，让存储的生命周期和容器的生命周期能有一个连接；
* K8s 会做自动化的**容器恢复**。在一个集群中，经常会出现宿主机的问题或者说是 OS 的问题，导致容器本身的不可用，K8s 会自动恢复不可用的容器；
* K8s 会做应用的**自动发布**与应用的**回滚**，以及与应用相关的配置密文的管理；
* 对于 Job 类型任务，K8s 可以做**批量执行**；
* 为了让这个集群、这个应用更富有弹性，K8s 也支撑**水平伸缩**。

**举个例子**

1. 调度

   K8s 可以把用户提交的容器放到 K8s 管理的集群的某一台节点上去。K8s 的**调度器**是执行这项工作的组件，它会观察正在被调度的这个容器的大小、规格。

   根据容器所需要的 CPU 和需要的 memory，在集群中找一台相对比较空闲的机器来放置。

2. 自动修复

   K8s 有一个**节点健康检查**的功能，会监测这个集群中所有的宿主机，当宿主机本身出现故障，或软件出现故障的时候，K8s 会发现这个情况。

   下面，K8s 将运行在失败节点上的容器进行**自动迁移**，迁移到一个正在健康运行的宿主机上，来完成集群内容器的一个自动恢复。

3. 水平伸缩

   K8s 有**业务负载检查**能力，会监测业务上所承担的负载，如果这个业务本身的 CPU 利用率过高，或者响应时间过长，它可以对这个业务进行一次扩容。

   比如，将这个业务分为三份，分布到不同的节点上，以此来提高响应的时间。

## Kubernetes 的架构

K8s 架构是一个比较典型的二层架构和 server-client 架构。Master 作为中央的管控节点，会去与 Node 进行一个连接。

所有 UI 的 clients 和 user 侧的组件，只会和 Master 进行连接，把希望的状态或者想执行的命令下发给 Master，Master 会把这些命令或状态下发给相应的节点，执行最终的执行。

{{< figure src="/images/202105/k8s-notebook/k8s-architecture.png" title="Kubernetes 架构" height="600" width="600">}}

### K8s 的架构：Master

K8s 的 Master 包含四个主要的组件：API Server、Controller、Scheduler 以及 etcd。如下图所示：

{{< figure src="/images/202105/k8s-notebook/master.png" title="Kubernetes Master" height="600" width="600">}}

* **API Server**： 顾名思义是用来处理 API 操作的，K8s 中**所有组件**都会和 API Server 进行连接，组件与组件之间一般不进行独立的连接，都依赖于 API Server 进行消息的传送；
* **Controller**： 控制器，用来完成对集群状态的一些管理；
* **Scheduler**： 调度器，依据用户提交 Container 所需的 CPU 和 memory 请求大小，找到合适的节点进行放置；
* **etcd**： 分布式存储系统，API Server 中所需的原信息放置在 etcd 中，etcd 本身也是一个高可用系统，通过 etcd 保证整个 K8s 的 Master 组件的高可用性。

### K8s 的架构：Node

K8s 的 Node 是真正**运行业务负载**的，每个业务负载会以 Pod 的形式运行。一个 Pod 中运行的一个或者多个容器，真正去运行这些 Pod 的组件的是叫做 kubelet，也就是 Node 上最为关键的组件，它通过 API Server 接收到所需要 Pod 运行的状态，然后提交到 Container Runtime 组件中。

{{< figure src="/images/202105/k8s-notebook/node.png" title="Kubernetes Node" height="600" width="600">}}

在 OS 上去创建容器所需要运行的环境，最终把容器或者 Pod 运行起来，也需要对存储和网络进行管理。**K8s 并不会直接进行网络存储的操作，他们会靠 Storage Plugin 或者是网络的 Plugin 来进行操作**。用户自己或者云厂商都会去写相应的 Storage Plugin 或者 Network Plugin，去完成存储操作或网络操作。

在 K8s 自己的环境中，也会有 K8s 的 Network，它是为了提供 Service network 来进行搭网组网的。**真正完成 service 组网的组件的是 Kube-proxy**，它是利用了 **iptable** 的能力来进行组建 K8s 的 Network，就是 cluster network，以上就是 Node 上面的四个组件。

### 组件之间的通信

{{< figure src="/images/202105/k8s-notebook/comm.png" title="Component Communication" height="600" width="600">}}

1. 用户可以通过 UI 或 CLI 提交一个 Pod 给 K8s 进行部署，这个 Pod 请求首先会通过 CLI 或者 UI 提交给 K8s API Server；
2. API Server 会把这个信息写入到它的存储系统 etcd；
3. 之后 Scheduler 会通过 API Server 的 watch 或者叫做 notification 机制得到这个信息：有一个 Pod 需要被调度；
4. Scheduler 会根据内存状态进行一次调度决策，完成调度后通知 API Server；
5. API Server 收到这次操作后会把结果再次写到 etcd 中；
6. API Server 会通知相应节点进行这次 Pod 真正的执行启动；
7. 相应节点的 kubelet 得到通知，调 Container runtime 去启动容器和环境，调度 Storage Plugin 去配置存储，network Plugin 去配置网络。

## K8s 的核心概念与 API

### K8s 的核心概念：Pod

Pod 是 K8s 的一个**最小调度以及资源单元**。用户可以通过 K8s 的 Pod API 生产一个 Pod，让 K8s 对这个 Pod 进行调度，也就是把它放在某一个 K8s 管理的节点上运行起来。一个 Pod 简单来说是对一组容器的抽象，它里面会包含一个或多个容器。

在 Pod 里面，可以去定义容器所需要运行的方式。比如说运行容器的 Command，以及运行容器的环境变量等等。Pod 这个抽象也给这些容器提供了一个**共享的运行环境**，它们会共享同一个网络环境，这些容器可以用 localhost 来进行直接的连接。而 Pod 与 Pod 之间，是互相有 **isolation 隔离**的。

### K8s 的核心概念：Volume

Volume 就是卷的概念，它是用来管理 K8s **存储**的，是用来声明在 Pod 中的容器可以访问文件目录的，一个卷可以被挂载在 Pod 中一个或者多个容器的指定路径下面。

{{< figure src="/images/202105/k8s-notebook/volume.png" title="Kubernetes Conception: Volume" height="300" width="150">}}

而 Volume 本身是一个抽象的概念，一个 Volume 可以去支持多种的后端的存储。比如说 K8s 的 Volume 就支持了很多存储插件，它可以支持本地的存储，可以支持分布式的存储，比如说像 ceph，GlusterFS ；它也可以支持云存储，比如说阿里云上的云盘、AWS 上的云盘、Google 上的云盘等等。

### K8s 的核心概念：Deployment

Deployment 是在 Pod 这个抽象上更为上层的一个抽象，它可以定义一组 Pod 的副本数目、以及这个 Pod 的版本。一般大家用 Deployment 这个抽象来做应用的真正的管理，而 Pod 是组成 Deployment 最小的单元。

{{< figure src="/images/202105/k8s-notebook/deployment.png" title="Kubernetes Conception: Deployment" height="300" width="600">}}

Kubernetes 是通过 Controller，也就是我们刚才提到的控制器去维护 Deployment 中 Pod 的数目，它也会去帮助 Deployment 自动恢复失败的 Pod。

比如说我可以定义一个 Deployment，这个 Deployment 里面需要两个 Pod，当一个 Pod 失败的时候，控制器就会监测到，它重新把 Deployment 中的 Pod 数目从一个恢复到两个，通过再去新生成一个 Pod。通过控制器，我们也会帮助完成发布的策略。比如说进行滚动升级，进行重新生成的升级，或者进行版本的回滚。

### K8s 的核心概念：Service

Service 提供了一个或者多个 Pod 实例的**稳定访问地址**。

比如在上面的例子中，我们看到：一个 Deployment 可能有两个甚至更多个完全相同的 Pod。对于一个外部的用户来讲，访问哪个 Pod 其实都是一样的，所以它希望做一次负载均衡，在做负载均衡的同时，我只想访问某一个固定的 VIP，也就是 Virtual IP 地址，而不希望得知每一个具体的 Pod 的 IP 地址。

我们刚才提到，这个 pod 本身可能 terminal go（终止），如果一个 Pod 失败了，可能会换成另外一个新的。

对一个外部用户来讲，提供了多个具体的 Pod 地址，这个用户要不停地去更新 Pod 地址，当这个 Pod 再失败重启之后，我们希望有一个抽象，**把所有 Pod 的访问能力抽象成一个第三方的一个 IP 地址**，实现这个的 K8s 的抽象就叫 Service。

实现 Service 有多种方式，K8s 支持 Cluster IP，上面我们讲过的 kuber-proxy 的组网，它也支持 nodePort、 LoadBalancer 等其他的一些访问的能力。

{{< figure src="/images/202105/k8s-notebook/service.png" title="Kubernetes Conception: Service" height="400" width="600">}}

### K8s 的核心概念：Namespace

Namespace 是用来做一个**集群内部的逻辑隔离**的，它包括**鉴权、资源管理**等。K8s 的每个资源，比如刚才讲的 Pod、Deployment、Service 都属于一个 Namespace，同一个 Namespace 中的资源需要命名的唯一性，不同的 Namespace 中的资源可以重名。

{{< figure src="/images/202105/k8s-notebook/namespace.png" title="Kubernetes Conception: Namespace" height="400" width="600">}}

### K8s 的 API

从 high-level 上看，K8s API 是由 HTTP+JSON 组成的：用户访问的方式是 HTTP，访问的 API 中 content 的内容是 JSON 格式的。

K8s 的 kubectl 也就是 command tool，Kubernetes UI，或者有时候用 curl，直接与 K8s 进行沟通，都是使用 HTTP + JSON 这种形式。

{{< figure src="/images/202105/k8s-notebook/api.png" title="Kubernetes API" height="400" width="600">}}

如图所示，对于这个 Pod 类型的资源，它的 HTTP 访问路径，就是 API，然后是 apiVersion: V1，之后是相应的 Namespace，以及 Pods 资源，最终是 Podname。

如果我们去提交一个 Pod，或者 get 一个 Pod 的时候，它的 content 内容都是用 JSON 或者是 YAML 表达的。上图中右侧是 yaml 的例子，在这个 yaml file 中，对 Pod 资源的描述也分为几个部分。

第一个部分，一般来讲会是 API 的 **version**。比如在这个例子中是 V1，它也会描述我在操作哪个资源；比如说我的 **kind** 如果是 pod，在 Metadata 中，就写上这个 Pod 的名字；比如说 nginx，我们也会给它打一些 **label**，我们等下会讲到 label 的概念。在 Metadata 中，有时候也会去写 **annotation**，也就是对资源的额外的一些用户层次的描述。

比较重要的一个部分叫做 **Spec**，Spec 也就是我们希望 Pod 达到的一个预期的状态。比如说它内部需要有哪些 container 被运行；比如说这里面有一个 nginx 的 container，它的 image 是什么？它暴露的 port 是什么？

当我们从 Kubernetes API 中去获取这个资源的时候，一般来讲在 Spec 下面会有一个项目叫 **status**，它表达了这个资源当前的状态；比如说一个 Pod 的状态可能是正在被调度、或者是已经 running、或者是已经被 terminates，就是被执行完毕了。

刚刚在 API 之中，我们讲了一个比较有意思的 metadata 叫做“label”，这个 label 可以是一组 KeyValuePair。

这些 label 是可以被 **selector**，也就是选择器所查询的。这个能力实际上跟我们的 sql 类型的 select 语句是非常相似的。

通过 label，k8s 的 API 层就可以对这些资源进行一个筛选，那这些筛选也是 kubernetes 对资源的集合所表达默认的一种方式。
