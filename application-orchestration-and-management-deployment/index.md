# 应用编排与管理：Deployment


## 需求来源
### 背景问题
现在存在三个应用，分别对应一些 Pod，那么我们可以直接管理集群中所有的 Pod 吗？

如果管理所有的 Pod，那么如何保证集群里可用 Pod 数量？如何为所有 Pod 更新镜像版本？更新的过程中，如何保证服务可用性？更新的过程中，发现问题如何快速回滚？

### Deployment：管理部署发布的控制器
可以通过 Deployment 将三个应用分别规划到不同的 Deployment，每个 Deployment 管理一组相同的应用 Pod，这组 Pod 是相同的一个副本。

{{< figure src="/images/202106/application-orchestration-and-management-deployment/deployment.png" title="Deployment" height="400" width="450">}}

Deployment 可以实现以下功能：

1. Deployment 定义了 Pod 期望数量；
2. 配置 Pod 发布方式，即 controller 会按照用户给定的策略来更新 Pod，而且更新过程中，可以设定不可用 Pod 数量在多少范围内；
3. 更新过程中发现问题可以回滚。

## 架构设计

### 管理模式

{{< figure src="/images/202106/application-orchestration-and-management-deployment/management-model.png" title="Management Model" height="400" width="200">}}

Deployment 只管理不同版本的 ReplicaSet，由 ReplicaSet 来管理具体的 Pod 副本数，每个 ReplicaSet 对应 Deployment template 的一个版本。

### Deployment 控制器

{{< figure src="/images/202106/application-orchestration-and-management-deployment/deployment-controller.png" title="Deployment Controller" height="600" width="650">}}

控制器通过 Informer 中的 Event 做一些 Handler 和 Watch，收到事件后会加入到队列中。而 Deployment controller 从队列中取出来之后，它的逻辑会判断 Check Paused，如果 Paused 设置为 true 的话，就表示这个 Deployment 只会做一个数量上的维持，不会做新的发布，如果为 false 的话，就会做 Rollout。

### ReplicaSet 控制器

{{< figure src="/images/202106/application-orchestration-and-management-deployment/replicaset-controller.png" title="Replicaset Controller" height="600" width="650">}}

当 Deployment 分配 ReplicaSet 之后，ReplicaSet 控制器本身也是从 Informer 中 watch 一些事件，这些事件包含了 ReplicaSet 和 Pod 的事件。从队列中取出之后，ReplicaSet controller 的逻辑很简单，就只管理副本数。

### 发布模拟

{{< figure src="/images/202106/application-orchestration-and-management-deployment/deployment-process.png" title="Deployment Process" height="600" width="650">}}

Deployment 当前初始版本为 template1，底下有三个 Pod：Pod1、Pod2、Pod3。

这时修改 template 中一个容器的 image，Deployment controller 就会新建一个对应 template2 的 ReplicaSet。创建出来之后 ReplicaSet 会逐渐修改两个 ReplicaSet 的数量，比如逐渐增加 ReplicaSet2 中 replicas 的期望数量，逐渐减少 ReplicaSet1 中的 Pod 数量。

### spec 字段解析

* MinReadySeconds：Deployment 会根据 Pod ready 来看 Pod 是否可用，但是如果我们设置了 MinReadySeconds 之后，比如设置为 30 秒，那 Deployment 就一定会等到 Pod ready 超过 30 秒之后才认为 Pod 是 available 的。Pod available 的前提条件是 Pod ready，但是 ready 的 Pod 不一定是 available 的，它一定要超过 MinReadySeconds 之后，才会判断为 available
* revisionHistoryLimit：保留历史 revision，即保留历史 ReplicaSet 的数量，默认值为 10 个。这里可以设置为一个或两个，如果回滚可能性比较大的话，可以设置数量超过 10；
* paused：paused 是标识，Deployment 只做数量维持，不做新的发布，这里在 Debug 场景可能会用到；
* progressDeadlineSeconds：前面提到当 Deployment 处于扩容或者发布状态时，它的 condition 会处于一个 processing 的状态，processing 可以设置一个超时时间。如果超过超时时间还处于 processing，那么 controller 将认为这个 Pod 会进入 failed 的状态。

### 升级策略字段解析

Deployment 在 RollingUpdate 中主要提供了两个策略，一个是 MaxUnavailable，另一个是 MaxSurge。

* MaxUnavailable：滚动过程中最多有多少个 Pod 不可用；
* MaxSurge：滚动过程中最多存在多少个 Pod 超过预期 replicas 数量。

上文提到，ReplicaSet 为 3 的 Deployment 在发布的时候可能存在一种情况：新版本的 ReplicaSet 和旧版本的 ReplicaSet 都可能有两个 replicas，加在一起就是 4 个，超过了我们期望的数量三个。这是因为我们默认的 MaxUnavailable 和 MaxSurge 都是 25%，默认 Deployment 在发布的过程中，可能有 25% 的 replica 是不可用的，也可能超过 replica 数量 25% 是可用的，最高可以达到 125% 的 replica 数量。

这里其实可以根据用户实际场景来做设置。比如当用户的资源足够，且更注重发布过程中的可用性，可设置 MaxUnavailable 较小、MaxSurge 较大。但如果用户的资源比较紧张，可以设置 MaxSurge 较小，甚至设置为 0，**这里要注意的是 MaxSurge 和 MaxUnavailable 不能同时为 0**。

