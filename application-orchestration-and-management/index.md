# 应用编排与管理：核心原理


## 资源元信息
### K8s 资源对象
K8s 的资源对象组成：主要包括了 Spec、Status 两部分。其中 Spec 部分用来描述期望的状态，Status 部分用来描述观测到的状态。

其实，K8s 还有另外一部分，即元数据部分。该部分主要包括了用来识别资源的标签：Labels，用来描述资源的注解：Annotations，用来描述多个资源之间相互关系的 OwnerReference。

### labels
第一个元数据，也是最重要的元数据是：资源标签。资源标签是一种具有标识型的 Key：Value 元数据。

标签主要用来筛选资源和组合资源，可以使用类似于 SQL 查询 select，来根据 Label 查询相关的资源。

### Annotations
Annotations 是系统或者工具用来存储资源的非标志性信息，可以用来扩展资源的 spec/status 的描述。

### Ownereference
所谓所有者，一般就是指集合类的资源，比如说 Pod 集合，就有 replicaset、statefulset。

集合类资源的控制器会创建对应的归属资源。比如：replicaset 控制器在操作中会创建 Pod，被创建 Pod 的 Ownereference 就指向了创建 Pod 的 replicaset。Ownereference 使得用户可以方便地查找一个创建资源的对象，另外，还可以用来实现级联删除的效果。

## 控制器模式

### 控制循环

控制型模式最核心的就是控制循环的概念。在控制循环中包括了控制器，被控制的系统，以及能够观测系统的传感器。

{{< figure src="/images/202106/application-orchestration-and-management/control-operation.png" title="Control Operation" height="600" width="650">}}

这些组件都是逻辑的，外界通过修改资源 spec 来控制资源，控制器比较资源 spec 和 status，从而计算一个 diff，diff 最后会用来决定对系统的控制操作，控制操作会使得系统产生新的输出，并被传感器以资源 status 形式上报，控制器的各个组件将都会是独立自主地运行，不断使系统向 spec 表示终态趋近。

### Sensor
循环控制中逻辑的传感器主要由 Reflector、Informer、Indexer 三个组件构成。

{{< figure src="/images/202106/application-orchestration-and-management/sensor.png" title="Sensor" height="600" width="650">}}

Reflector 通过 List 和 Watch K8s server 来获取资源的数据。List 用来在 Controller 重启以及 Watch 中断的情况下，进行系统资源的全量更新；而 Watch 则在多次 List 之间进行增量的资源更新；Reflector 在获取新的资源数据后，会在 Delta 队列中塞入一个包括资源对象信息本身以及资源对象事件类型的 Delta 记录，Delta 队列中可以保证同一个对象在队列中仅有一条记录，从而避免 Reflector 重新 List 和 Watch 的时候产生重复的记录。

Informer 组件不断从 Delta 队列中弹出 delta 记录，然后把资源对象交给 indexer，让 indexer 把资源记录在一个缓存中，缓存在默认设置下是用资源的命名空间来做索引的，并且可以被 Controller Manager 或多个 Controller 所共享。之后，再把这个事件交给事件的回调函数。

控制循环中的控制器组件主要由事件处理函数以及 worker 组成，事件处理函数会相互关注资源的新增、更新、删除的事件，并根据控制器的逻辑去决定是否需要处理。对需要处理的事件，会把事件关联资源的命名空间以及名字塞入一个工作队列中，并且由后续的 worker 池中的一个 worker 来处理，工作队列会对存储的对象进行去重，从而避免多个 worker 处理同一个资源的情况。

worker 在处理资源对象时，一般需要用资源的名字来重新获得最新的资源数据，用来创建或更新资源对象，或者调用其他的外部服务，worker 如果处理失败的时候，一般情况下会把资源的名字重新加入到工作队列中，从而方便之后进行重试。

### 控制循环例子-扩容

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rsA
  namespace: nsA
spec:
  replicas: 3
  selector:
    matchLabels:
      env: prod
  template:
    metadata:
      labels:
        env: prod
      spec:
        containers:
        - images: registry.cn-hangzhou.aliyuncs.com/acs/nginx
        name: nginx
status:
  replicas: 2
```

{{< figure src="/images/202106/application-orchestration-and-management/onUpdate.png" title="更新" height="600" width="650">}}

1. Reflector会 watch 到 ReplicaSet 和 Pod 两种资源的变换；
2. 向 delta 队列中塞入 rsA 对象，类型为更新；
3. Informer 取出记录；
4. 将新的 ReplicaSet 更新到缓存中，并与 Namespace nsA 作为索引；
5. 调用 Update 回调函数，ReplicaSet 控制器发现 ReplicaSet 发生变化后把字符串 nsA/rsA 字符串塞入到工作队列中；
6. worker 从工作队列中取到 nsA/rsA 这个字符串的 key；
7. 从缓存中取到了最新的 ReplicaSet 数据；
8. worker 通过比较 ReplicaSet 中 spec 和 status 里的数值，发现需要进行扩容操作，因此创建一个 Pod，这个 Pod 中的 Ownereference 指向了 ReplicaSet rsA；

{{< figure src="/images/202106/application-orchestration-and-management/onAdd.png" title="扩容" height="600" width="650">}}

9. Reflector Watch 到 Pod 新增事件；
10. Reflector 在 Delta 队列中额外加入 Add 类型的 delta 记录；
11. Informer 取出记录传送给 Indexer，调用 ReplicaSet 控制器的 Add 回调函数；
12. 将记录存储在缓存中；
13. Add 回调函数通过检查 pod ownerReferences 找到了对应的 ReplicaSet，并把包括 ReplicaSet 命名空间和字符串塞入到了工作队列中；
14. ReplicaSet 的 worker 在得到新工作项之后，从缓存中取到了新的 ReplicaSet 记录，并得到了其所有创建的 Pod；
15. ReplicaSet 更新 status 使得 spec 和 status 达成一致。

## 控制器模式总结

### 两种 API 设计方法

K8s 控制器模式依赖声明式 API，另一种常见 API 类型是命令式 API。

- 声明式 API：只说目标决定，具体操作交给 worker
- 命令式 API：告诉 worker 具体操作

### 命令式 API 的问题
1. 错误处理：在大规模分布式系统中，错误是无处不在的。一旦发出的命令没有响应，调用方只能通过反复重试的方式来试图恢复错误，然而盲目的重试可能会带来更大的问题。
2. 命令式 API 后台往往还有一个巡检系统，用来修正命令处理超时、重试等一些场景造成数据不一致的问题。
3. 容易在并发访问时出现问题：假如有多方并发的对一个资源请求进行操作，一旦其中有操作出现错误，就要重试。那么很难确认是哪个操作生效。很多命令式系统往往在操作前会对系统进行加锁，从而保证整个系统最后生效行为的可预见性，但是加锁行为会降低整个系统的操作执行效率。
4. 声明式 API 系统天然记录了系统现在和最终的状态。

### 总结

1. K8s 所采用的控制器模式，是由声明式 API 驱动的。确切来说，是基于对 K8s 资源对象的修改来驱动的；
2. K8s 资源之后，是关注该资源的控制器。这些控制器将异步的控制系统向设置的终态驱近；
3. 控制器是自主运行的，使得系统的自动化和无人值守成为可能；
4. 因为 K8s 的控制器和资源都是可以自定义的，因此可以方便的扩展控制器模式。特别是对于有状态应用，往往通过自定义资源和控制器的方式，来自动化运维操作。
