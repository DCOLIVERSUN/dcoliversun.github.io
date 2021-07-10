# 应用编排与管理：DaemonSet


## 需求来源

如果我们没有 DaemonSet 会怎么样？下面有几个需求：

* 首先如果希望每个节点都运行同样一个 pod 怎么办？
* 如果新节点加入集群的时候，想要立刻感知到它，然后去部署一个 pod，帮助我们初始化一些东西，这个需求如何做？
* 如果有节点退出的时候，希望对应的 pod 会被删除掉，应该怎么操作？
* 如果 pod 状态异常的时候，我们需要及时地监控这个节点异常，然后做一些监控或者汇报的一些动作，那么这些东西运用什么控制器来做？

## DaemonSet：守护进程控制器

DaemonSet 也是 Kubernetes 提供的一个 default controller，它实际是做一个守护进程的控制器，它能帮我们做到以下几件事情：

* 首先能保证集群内的每一个节点都运行一组相同的 pod；
* 同时还能根据节点的状态保证新加入的节点自动创建对应的 pod；
* 在移除节点的时候，能删除对应的 pod；
* 而且它会跟踪每个 pod 的状态，当这个 pod 出现异常、Crash 掉了，会及时地去 recovery 这个状态。

## 用例解读

### DaemonSet 语法

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  lables:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: fluent/fluentd:v1.4-1
```

首先是 kind:DaemonSet。它会有 matchLabel，通过 matchLabel 去管理对应所属的 pod，这个 pod.label 也要和这个 DaemonSet.controller.label 想匹配，它才能去根据 label.selector 去找到对应的管理 Pod。下面 spec.container 里面的东西都是一致的。

DaemonSet 适用场景：

1. 集群存储进程： GlusterFS 或者 Ceph 之类的东西，需要每台节点上都运行一个类似于 Agent 的东西，DaemonSet 就能很好地满足这个诉求；
2. 日志收集进程：logstash 或者 fluentd，这些都是同样的需求，需要每台节点都运行一个 Agent，这样的话，我们可以很容易搜集到它的状态，把各个节点里面的信息及时地汇报到上面；
3. 需要在每个节点运行的监控收集器，例如 Promethues。

### 查看 DaemonSet 状态

创建完 DaemonSet 之后，我们可以使用 kubectl get DaemonSet（DaemonSet 缩写为 ds）。

```
kubectl get ds
NAME					DESIRED	CURRENT	 READY	UP-TO-DATE	AVAILABLE NODE SELECTOR	AGE
fluentd-elasticsearch	4		4		 4 		4			4		  <none>		4s
```

状态描述：

* DESIRED：需要的 pod 个数
* CURRENT：当前已存在的 pod 个数
* READY：就绪的个数
* UP-TO-DATE：最新创建的个数
* AVAILABLE：可用 pod 个数
* NODE SELECTOR：节点选择标签，可用于选择部分节点去运行 pod，而不是全部节点运行 pod

### 更新 DaemonSet

其实 DaemonSet 和 deployment 特别像，它也有两种更新策略：一个是 RollingUpdate，另一个是 OnDelete。

* RollingUpdate 其实比较好理解，就是会一个一个的更新。先更新第一个 pod，然后老的 pod 被移除，通过健康检查之后再去见第二个 pod，这样对于业务上来说会比较平滑地升级，不会中断；
* OnDelete 其实也是一个很好的更新策略，就是模板更新之后，pod 不会有任何变化，需要我们手动控制。我们去删除某一个节点对应的 pod，它就会重建，不删除的话它就不会重建，这样的话对于一些我们需要手动控制的特殊需求也会有特别好的作用。

## 架构设计

### DaemonSet 管理模式

{{< figure src="/images/202107/application-orchestration-and-management-daemonset/architecture.png" title="DaemonSet Architecture" height="400" width="250">}}

1. DaemonSet Controller 负责根据配置创建 Pod
2. DaemonSet Controller 跟踪 Job 状态，根据配置及时重试 Pod 或者继续创建
3. DaemonSet Controller 会自动添加 affinity&label 来跟踪对应的 pod，并根据配置在每个节点或者适应的部分节点创建 Pod

### DaemonSet 控制器

{{< figure src="/images/202107/application-orchestration-and-management-daemonset/controller.png" title="DaemonSet Controller" height="600" width="650">}}

DaemonSet 其实和 Job controller 做的差不多：两者都需要根据 watch 这个 API Server 的状态。现在 DaemonSet 和 Job controller 唯一的不同点在于，DaemonsetSet Controller需要去 watch node 的状态，但其实这个 node 的状态还是通过 API Server 传递到 ETCD 上。

当有 node 状态节点发生变化时，它会通过一个内存消息队列发进来，然后DaemonSet controller 会去 watch 这个状态，看一下各个节点上是都有对应的 Pod，如果没有的话就去创建。当然它会去做一个对比，如果有的话，它会比较一下版本，然后加上刚才提到的是否去做 RollingUpdate？如果没有的话就会重新创建，Ondelete 删除 pod 的时候也会去做 check 它做一遍检查，是否去更新，或者去创建对应的 pod。

当然最后的时候，如果全部更新完了之后，它会把整个 DaemonSet 的状态去更新到 API Server 上，完成最后全部的更新。

## 本节总结

DaemonSet 基础操作与概念解析：通过类比 Deployment 控制器，我们理解了一下DaemonSet 控制器的工作流程与方式，并且通过对 DaemonSet 的更新了解了滚动更新的概念和相对应的操作方式。


