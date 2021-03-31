# Executor与线程池


## Executor框架

我们将执行的逻辑工作单元抽象为任务，那么线程就是使任务异步执行的容器。如果把所有任务放在单个线程执行，将对应用的响应性和吞吐量造成灾难性影响。

为每个任务分配一个线程似乎是不错的解决方案，不过这对资源管理有着较高的要求。线程池缓解了这一压力，它承担了管理线程工作。`java.util.concurrent` 提供了一种灵活的线程池作为Executor框架的一部分。

在 Java 类库中，任务执行的主要抽象不是 Thread，而是 Executor。

```java
public interface Executor {
  void execute (Runnable command);
}
```

Executor 为灵活且强大的异步任务执行框架提供了基础，为任务的提交与执行提供了标准的方法，将两个过程解耦开来。此外，Executor 实现了对生命周期的支持，以及监控管理等机制。

Executor 基于生产者-消费者模式，提交任务的操作相当于生产者，执行任务的操作类似于消费者。反之，如果想实现简单的生产消费模型，可以采用 Executor 实现。

## 执行策略

Executor 框架需要设计一套任务执行策略，以解决多任务执行混乱的问题。执行策略包括以下内容：

* 在什么线程中执行任务？
* 任务按照什么顺序执行（FIFO、LIFO、优先级）？
* 有多少个任务能并发执行？
* 在队列中有多少个任务在等待执行？
* 如果系统需要拒绝一个任务，应该选择哪个任务？如何通知应用有任务被拒绝？
* 在执行一个任务前后，应该进行哪些动作？

## 线程池

线程池是管理诸多线程的资源池，依靠任务保持所有等待执行的任务。[工作者线程]^(Work Thread)的任务很简单：从工作队列获取一个任务，执行任务，然后返回线程池并等待下一个任务。

为每个任务分配一个线程可能引入线程新建、销毁的开销，不如让多个任务在线程池中执行。通过重用现有的线程可以分摊多个线程的开销。此外，任务不会因等待线程创建而延迟执行，提升整体响应性。用户可以通过配置线程池大小，创建足够多的线程使处理器保持忙碌状态，还可防止多线程相互竞争资源而使应用程序耗尽内存或失败。

### newFixedThreadPool

newFixedThreadPool 是拥有固定数量线程的线程池，每当提交一个任务就创建一个线程，直到达到最大数量。如果某个线程因异常结束，那么线程池会补充一个新线程。

### newCachedThreadPool

newCachedThreadPool 是可缓存的线程池，如果线程池当前规模超过处理需求，将回收空闲线程，而当需求增加时，可以添加新线程。线程池规模不存在任何限制。

### newSingleThreadExecutor

newSingleThreadExecutor 是单线程的 Executor，如果线程因异常结束，newSingleThreadExecutor 会创建另一个线程来替代。任务按照队列顺序而执行。

### newScheduledThreadPool

newScheduledThreadPool 的线程数量也是固定的，但可以以延迟或定时方式执行任务。

## Executor 的生命周期

Executor 通常会创建线程来执行任务，只有当所有线程全部终止后才会退出，如果无法正确关闭 Executor，JVM 将无法结束。

因为 Executor 以异步方式执行任务，可能引发任务状态不同步的问题。同一时刻，不同任务可能处于执行、完成、等待执行三个状态。为了解决执行任务的生命周期问题，Executor 扩展了 ExecutorService 接口，添加了一些用于生命周期管理的方法。

```java
public interface ExecutorService extends Executor {
  void shutdown();
  List<Runnable> shutdownNow();
  boolean isShutdown();
  boolean isTerminated();
  boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException;
  // ...
}
```

ExecutorService 的生命周期有三种状态：运行、关闭和已终止。shutdown 方法将平缓地关闭 ExecutorService：不再接受新任务，同时等待已经提交的任务执行完成——包括还未开始执行的任务。shutdownNow 方法将粗暴地关闭 ExecutorService：直接取消所有执行中的任务，并且不再启动队列中尚未开始执行的任务。

awaitTermination 方式可以等待 ExecutorService 到达终止状态，或者调用 isTerminated 来轮询 ExecutorService 是否已经终止。通常在调用 awaitTermination 方法后立即调用 shutdown，从而产生同步关闭的效果。

## 延迟任务与周期任务

Timer 类负责管理延迟任务以及周期任务。但 Timer 类支持的是基于绝对时间的调度机制，对系统时钟变化的容忍度很低，存在天然缺陷。ScheduledThreadPool 只支持基于相对时间的调度，可以通过 ScheduledThreadPool 的构造函数或 newScheduledThreadPool 工厂方法来创建该类的对象。

如果构建调度服务，可以使用 DelayQueue，它实现了 BlockingQueue，并为 ScheduledThreadPoolExecutor 提供调度功能。DelayQueue 管理着一组 Delayed 对象。每个 Delayed 对象都有一个相应的延迟时间。

## 总结

本文一开始介绍了 Executor 框架，框架采用了任务提交、执行的解耦方案。为了让该方案适配不同场景，需要将多种因素考虑进执行策略中。不同的执行策略也衍生出不同的线程池，我们在使用前需要分析真实环境去选择适当的线程池。线程池异步执行多个任务，导致任务可能处于不同的状态。为了管理整个线程池的生命周期，ExecutorService 提供了多种方法，一般采取 awaitTermination、shutdown 组合使用的方式，达到同步关闭的效果。最后，本文介绍了 ScheduledThreadPool 在延迟任务、周期任务的优越性，如果构建调度服务，可以采用 DelayQueue。
