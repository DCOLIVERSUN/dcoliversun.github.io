# Spark on MR3——运行 Apache Spark 的新方式

MR3 是一个通用的执行引擎，原生支持 Hadoop 和 Kubernetes。虽然 Hive on MR3 是主要应用，但 MR3 也可以轻松执行 MapReduce/Tez 作业。例如，Hive on MR3 可以不依赖 MapReduce 进行压缩，因为执行压缩的 MapReduce 作业会被直接发送到 MR3。

随着 MR3 1.3 的发布，开发团队引入了另一个主要应用 –  Spark on MR3。简而言之，就是使用 MR3 作为执行后端的 Apache Spark。MR3 上的 Spark 是作为 Spark 的附加组件实现的，它利用 MR3 来实现 Spark 的调度后端。因此，在 MR3 上运行 Spark 不需要对 Spark 进行任何更改，并且用户在 MR3 上安装 Spark 时可以使用 Spark Binary Distribution。

在详细介绍 Spark on MR3 之前，应该回答一个关键问题——**为什么我们需要一个替代的 Spark 执行后端？** 毕竟 Spark 是一个成熟的系统，已经为 Hadoop 和 Kubernetes 提供了原生支持，所以 Spark on MR3 似乎没有带来额外的优势。然而，Spark on MR3 不仅仅是为了展示 MR3 作为执行引擎的能力。相反，它解决了 Apache Spark 的一个重要架构限制。

## 动机

**在 MR3 上开发 Spark 的主要动机是允许多个 Spark 应用程序共享计算资源，例如 Yarn 容器或 Kubernetes Pod。** 对于普通 Spark，不同的 Spark 应用程序必须维护自己的一组 executor，因为 Spark 缺乏在 Spark 应用程序之间回收计算资源的功能。相反，Spark 依靠资源管理器（例如 Yarn 和 Kubernetes）来分配计算资源。请注意，Apache Livy 不是这里的解决方案，因为它只允许多个用户通过 REST API 共享 Spark 应用程序。

对于 Spark on MR3，MR3 DAGAppMaster 的单实例管理 Spark 应用程序之间共享的所有计算资源。下图说明了在 Kubernetes 上运行的 Spark on MR3，其中四个 Spark driver 共享一个公共 DAGAppMaster Pod。当 Spark driver 终止时，其执行程序 Pod 不会立即释放回 Kubernetes。相反，DAGAppMaster 尝试通过将空闲 Pod 重新分配给那些请求更多计算资源的 Spark driver 或将它们保留在备用池中以备将来使用。

{{< figure src="/images/202109/spark-on-mr3/spark.mr3.k8s.client.mode-fs8.png" title="Spark on MR3 K8s Client Mode" height="500" width="700">}}

因此，当多个 Spark 应用程序并发运行时，Spark on MR3 减少了获取和释放计算资源的开销。由于 MR3 使用 Java 虚拟机(JVM)，回收计算资源也转化为减少 JVM 预热开销。此外，Spark on MR3 将自身与资源管理器的调度策略分离。例如，无论资源管理器的调度策略如何，我们都可以在 Spark 应用程序之间强制执行公平调度。

Spark on MR3 在频繁创建和销毁 Spark 应用程序的云环境中特别有用。在云环境中，配置计算资源的开销远非可以忽略不计。例如，在 Amazon EKS 上预置新的 Kubernetes 节点可能需要几分钟时间。通过允许 DAGAppMaster 保留计算资源的备用池，Spark on MR3 可以有效地最小化计算资源的开销。因为 Spark 作业在提交后立即执行，它可以实现云成本的降低以及用户体验的改进。

## 应用

Spark on MR3 是作为 Spark 的附加组件实现的。Spark-MR3 作为外部集群管理器，提供 Spark 调度后端的实现。Spark-MR3 嵌入在 Spark driver中，并在 Spark driver 和 MR3 之间传输消息。在内部，它提供了 Spark TaskScheduler 和 SchedulerBackend 的实现，并创建了 MR3Client 以便与 MR3 进行通信。当 Spark DAGScheduler 想要执行新的 Spark stage 时，Spark-MR3 将 stage 的格式转换为 DAG，然后发送到 MR3 DAGAppMaster。DAG 由运行在 ContainerWorkers 内的 Spark executor 执行，并将结果发送回 Spark。

{{< figure src="/images/202109/spark-on-mr3/spark-mr3-architecture-fs8.png" title="Spark on MR3 Architecture" height="500" width="700">}}

由于 Spark-MR3 将所有来自 Spark 的请求发送到 MR3，因此从 Spark 的角度来看，MR3 是类似于 Yarn 和 Kubernetes 的资源管理器。这是有效的，因为 Spark 与底层集群管理器无关，而 MR3 能够在与真正的资源管理器通信后提供计算资源。由于 Spark 使用 Java ServiceLoader 机制加载外部集群管理器，因此在 MR3 上运行 Spark 不需要对 Spark 进行任何更改。

## 实验

现在开发团队展示实验结果来证明 Spark on MR3 的性能。除了任务调度和资源管理，Spark on MR3 的行为方式与普通 Spark 相同。首先，开发团队展示了 Spark on MR3 在单个 Spark 应用程序上与普通 Spark 表现一致。接下来，开发团队将展示当多个 Spark 应用程序同时运行时，Spark on MR3 的运行时间大幅减少。

在实验中，开发团队使用两个运行 HDP（Hortonworks Data Platform）3.1.4 的集群。每个工作节点运行一个 ContainerWorker，或者说是一个 Spark executor。

* Gold 集群，由 2 个主节点、34 个工作节点（每个节点具有 24 个内核和 96 GB 内存）和十千兆网络组成。
* Grey 集群，1 个主节点，9 个工作节点（每个节点有 12 个内核和 48 GB 内存）和一千兆网络。慢网络使得 Grey 集群适合评估普通 Spark 和 Spark on MR3 中延迟调度的有效性。

开发团队使用 Spark 多用户基准测试作为工作负载。Spark 应用程序中的每个线程都提交相同的 8 个 TPC-DS 基准查询序列（#19、#42、#52、#55、#63、#68、#73、#98）。开发团队使用的比例因子为 100（大约相当于 100GB），它可以保持每个工作节点处于忙碌状态，也可以清楚地突出回收 ContainerWorkers 的效果。开发团队以文本格式存储数据集而不进行压缩。

### 单个 Spark 应用程序

Spark on MR3 提供了两种任务调度策略——FIFO 调度和公平调度，它们规定如何在同一个 Spark 应用程序中分配 Spark 作业。Spark on MR3 也以类似于普通 Spark 的方式实现延迟调度。对于单个 Spark 应用程序的测试，开发团队使用 1 秒延迟进行延迟调度（例如，设置普通 Spark 的参数 `spark.locality.wait` 为 1 秒 ）。

下图显示了在 Gold 集群中使用普通 Spark 和 Spark on MR3 运行单个 Spark 应用程序的结果。Spark driver 创建八个线程，每个线程提交相同的八个 TPC-DS 查询序列。（因此它总共提交了 $8 \times 8 = 64$ 个 Spark 作业。）开发团队观察到，在 FIFO 调度下，Spark on MR3 的运行速度与普通 Spark 一样快，但在公平调度下，速度稍慢。

{{< figure src="/images/202109/spark-on-mr3/gold.compare.spark.mr3-fs8.png" title="Cluster Gold Compare Result" height="1000" width="3000">}}

Grey 集群测试结果完全不同。开发团队观察到，Spark on MR3 比普通 Spark 执行 Spark 作业的完成速度要快得多，FIFO 调度为 3034 s 与 8042 s，公平调度为 3583 s 与 8017 s。

{{< figure src="/images/202109/spark-on-mr3/grey.compare.spark.mr3-fs8.png" title="Cluster Grey Compare Result" height="1000" width="3000">}}

运行时间的巨大差异是由于任务调度的不同，特别是延迟调度在普通 Spark 和 Spark on MR3 中是如何工作的。下图显示了没有匹配机器的 Spark 任务的百分比。例如，一个带有 grey1 和 grey2 位置标签的 Spark 任务，如果在 grey3 上执行，则没有匹配到机器上。开发团队观察到，普通 Spark 的百分比要高得多，FIFO 调度为 1.8% 与 22.7%，公平调度为 7.2% 与 26.8%。结合使用慢速（1 Gb）网络，高百分比会产生许多落后的任务，从而显着增加运行时间。

{{< figure src="/images/202109/spark-on-mr3/grey.percentage.no.match.host-fs8.png" title="Cluster Grey No Match Host Compare Result" height="1000" width="3000">}}

### 多个 Spark 应用程序

独立于任务调度策略，Spark on MR3 提供了两个策略——FIFO 调度和公平调度，指定如何在多个 Spark 应用程序之间分配 ContainerWorkers（即计算资源）。为了测试多个应用程序，开发团队禁用延迟调度。

下图显示了在使用 Yarn 公平调度的 Gold 集群中使用普通 Spark 运行 12 个相同的 Spark 应用程序的结果。竖条显示在相应工作节点上运行的 Spark executor 的进度，其颜色表示拥有 Spark executor 的 Spark 应用程序。可以观察到，尽管使用了公平调度，但 Spark 应用程序在 Yarn 分配的计算资源总量方面存在很大差异。我们还看到竖线之间的许多间隙，它们表示 Spark executor 终止后和另一个 Spark executor 启动之前的空闲时间。

{{< figure src="/images/202109/spark-on-mr3/vanilla-spark-fair-gold-fs8.png" title="Vanilla Spark in Fair Gold" height="1000" width="3000">}}

下图显示了在 Gold 集群中使用公平调度在 Spark on MR3 运行相同的 12 个 Spark 应用程序的结果。 顺便说一句，资源管理器本身的调度策略对实验来说并不重要。可以观察到，Spark on MR3 通过回收 ContainerWorkers 快速实现了 Spark executor 之间工作节点的公平分配。 请注意，Spark on MR3 不会在垂直条之间产生间隙，因为在整个实验过程中，每个工作节点上都会回收一个 ContainerWorker。 在我们的实验中，与普通 Spark 相比，Spark on MR3 将运行时间减少了 25.6%（从 907.7s 到 674.6s）。

{{< figure src="/images/202109/spark-on-mr3/spark-mr3-fair-gold-fs8.png" title="Spark on MR3 in Fair Gold" height="1000" width="3000">}}

下图显示了当  Yarn 和 Spark on MR3 都使用 FIFO 调度时 Gold 集群中的结果。开发团队以 60 秒的间隔提交每个 Spark 应用程序。普通 Spark 和 Spark on MR3 都一次运行一个 Spark 应用程序，并按照提交的顺序完成所有 Spark 应用程序。 请注意，一旦第一个 Spark 应用程序终止，Spark on MR3 就会更快地完成剩余的 Spark 应用程序，因为 JVM 已经预热。 在实验中，与普通 Spark 相比，Spark on MR3 将运行时间减少了 36.3%（从 797.3s 到 1144.5s）。

{{< figure src="/images/202109/spark-on-mr3/spark.mr3.gold.multi.fifo-fs8.png" title="Vanilla Spark vs. Spark on MR3 in FIFO Gold" height="1000" width="3000">}}

## 总结

Spark on MR3 易于使用在 Hadoop 和 Kubernetes 上，因为它与 Apache Spark 的不同之处仅在于任务调度和资源管理。 除了在 Spark 应用程序中回收 ContainerWorkers 之外，用户还可以利用 MR3 的容错、推测执行和自动缩放等特性。

{{< admonition type=tip title="相关链接" open=true >}}
原文: https://www.datamonad.com/post/2021-08-18-spark-mr3/

code: https://github.com/mr3project/spark-mr3
{{< /admonition >}}


