# 配置 Java 线程池


## 配置线程池大小

线程池的理想大小取决于被提交任务的类型以及所部署系统的特性。在实际工程中，通常不会固定线程池大小，应该通过某种配置机制来提供，或者根据  `Runtime.availableProcessors` 来动态计算。

在配置线程池大小时，应该避免“过大”和“过小”这两种极端情况。如果线程池过大，那么大量的线程将在相对很少的 CPU 和内存资源上发生竞争，不仅消耗更高的内存，而且还可能耗尽资源。如果线程池过小，将导致许多空闲处理器无法执行工作，从而降低吞吐率。

对于计算密集型的任务，在拥有 $N_{cpu}$ 个处理器系统上，当线程池的大小为 $N_{cpu}+1$ 时，通常能实现最优的利用率。

对于包含 I/O 操作或者其他阻塞操作的任务，由于线程并不会一直执行，因此线程池的规模应该更大。要正确设置线程池大小，你必须估算出任务的等待时间与计算时间的比值。这种估算不需要很精确，并且可以通过一些分析或监控工具来获得。

要使处理器达到期望的使用率，线程池的最优大小等于：

$$N_{threads}=N_{cpu}\times U_{cpu}\times \left( 1+\frac{W}{C} \right)$$

式中，$N_{cpu}$ 代表 CPU 的数量，$U_{cpu}$ 代表 CPU 目标利用率，$U_{cpu}\in [0,1]$，$\frac{W}{C}$ 代表等待时间与计算时间的比值。

可以通过 `Runtime` 来获得 CPU 的数目：

```java
int N_CPUS = Runtime.getRuntime().availableProcessors();
```

CPU 周期并不是唯一影响线程池大小的资源，还包括内存、文件句柄、套接字句柄和数据库连接等。计算这些资源对线程池的约束条件是更容易的：

1. 计算每个任务对该资源的需求量；
2. 用该资源的可用总量除以每个任务的需求量，所得结果就是线程池大小上限。

## 配置 ThreadPoolExecutor

```java
public ThreadPoolExecutor (int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) { ... }
```

### 线程的创建与销毁

以下三个参数主要负责线程的创建与销毁：

* **corePoolSize**，线程池基本大小，即在没有任务执行时线程池的大小，并且只有在工作队列满了的情况下才会创建超出这个数量的线程。
* **maximumPoolSize**，线程池最大大小，表示可同时活动的线程数量的上限。
* **keepAliveTime**，线程池存活时间。如果某个线程的空闲时间超过了存活时间，那么将被标记为可回收的。

线程池的基本大小、最大大小和存储时间等因素共同负责线程的创建与销毁。当线程池的当前大小超过了基本大小时，被标记为可回收的线程将被终止。通过调节基本大小和存活时间，可以帮助线程池回收空闲线程占有的资源，从而使得这些资源可以用于执行其他工作。

### 管理队列任务

ThreadPoolExecutor 允许提供一个 BlockingQueue 来保存等待执行的任务。基本的任务排队方法有 3 种：无界队列、有界队列和同步移交（Synchronous Handoff）。

* **ArrayBlockingQueue**：一个基于数组结构的有界阻塞队列，按照 FIFO 原则对任务进行排序；
* **LinkedBlockingQueue**：一个基于链表结构的无界阻塞队列，按照 FIFO 原则排序任务，吞吐量通常要高于 ArrayBlockingQueue。
* **SynchronousQueue**：一个不存储元素的线程间同步移交，要将一个元素放入 SynchronousQueue 中，必须有另一个线程正在等待接受这个元素。如果没有线程正在等待，并且线程池的当前大小小于最大值，那么 ThreadPoolExecutor 将创建一个新的线程，否则根据饱和策略，这个任务被拒绝。
* **PriorityBlockingQueue**：一个具有优先级的有界阻塞队列，这个队列根据优先级来安排任务，任务的优先级是通过自然顺序或 `Comparator` 来定义的。

### 饱和策略

当有界队列被填满后，饱和策略开始发挥作用。ThreadPoolExecutor 的饱和策略可以通过调用 `setRejectedExecutionHandler` 来修改。JDK 提供了几种不同的实现，每种实现包含不同的饱和策略：

* `ThreadPoolExecutor.AbortPolicy`：中止策略是默认的饱和策略，该策略将在线程池数量等于最大线程数时，抛出未检查的 `RejectedExecutionException`。涉及到的任务将不会执行。调用者可以捕获这个异常，然后根据需求编写自己的处理代码。
* `ThreadPoolExecutor.CallerRunsPolicy`：既不会抛弃任务，也不会抛出异常，而是将某些任务回退到调用者，从而降低新任务的流量。它不会在线程池的某个线程中执行新提交的任务，而是在一个调用了 execute 的线程中执行该任务。
* `ThreadPoolExecutor.DiscardPolicy`：该策略在线程池中数量等于最大线程数时，会悄悄丢弃不能执行的新增任务，不报任何异常；
* `ThreadPoolExecutor.DiscardOldestPolicy`：该策略在线程池中数量等于最大线程数时，会抛弃线程池中工作队列头部的任务（即等待时间最久的任务），并执行当前任务。

### 线程工厂

每当线程池需要创建一个线程时，都是通过线程工厂方法来完成的。默认的线程工厂方法将创建一个新的、非守护的线程，并且不包含特殊的配置信息。通过指定一个线程工厂方法，可以定制线程池的配置信息。

定制线程池配置信息在某些场景下有需求，例如希望为线程池中的线程指定一个 `UncaughtExecptionHandler`，或者实例化一个定制的 Thread 类用于执行调试信息的记录。

下面的例子展示了自定义线程工厂为每个创建的线程池设置更有意义的名字，在 Debug 和定位问题时非常有帮助。

```java
public class MyThreadFactory implements ThreadFactory {
  private final String poolName;
  
  public MyThreadFactory(String poolName) {
    this.poolName = poolName;
  }
  
  public Thread newThread(Runnable runnable) {
    return new MyAppThread(runnable, poolName);
  }
}
```
