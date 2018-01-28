---
title: ThreadPoolExecutor — 基本理论
date: 2018/01/28
categories:
- 并发编程
tags:
- Java
- 并发编程
- 线程池
- ThreadPool
- Executor
---

![](/images/2018-01/1516891652933.png)
**声明：这两篇有关线程池 ThreadPoolExecutor 的分析文章，非原创内容，是自己先阅读了几遍源码并分析后，然后基于网上的分享的梳理总结，请看参考资源。请你自己一定要真正的去阅读源码，才能理解得更深，因为我也做到了**
### 前言
在web开发中，服务器需要接受并处理请求，所以会为一个请求来分配一个线程来进行处理。如果每次请求都新创建一个线程的话实现起来非常简便，但是存在一个问题：

如果并发的请求数量非常多，但每个线程执行的时间很短，这样就会频繁的创建和销毁线程，如此一来会大大降低系统的效率。可能出现服务器在为每个请求创建新线程和销毁线程上花费的时间和消耗的系统资源要比处理实际的用户请求的时间和资源更多。

那么有没有一种办法使执行完一个任务，并不被销毁，而是可以继续执行其他的任务呢？

<!-- more -->
这就是线程池的目的了：
1. 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

线程池为线程生命周期的开销和资源不足问题提供了解决方案。通过对多个任务重用线程，线程创建的开销被分摊到了多个任务上。

### Executor 框架
Executor框架是一个根据一组执行策略调用，调度，执行和控制的异步任务的框架，目的是提供一种将”任务提交”与”任务如何运行”分离开来的机制。

J.U.C中有三个Executor接口：

Executor：一个运行新任务的简单接口；
ExecutorService：扩展了Executor接口。添加了一些用来管理执行器生命周期和任务生命周期的方法；
ScheduledExecutorService：扩展了ExecutorService。支持Future和定期执行任务。

#### Executor接口
```java
public interface Executor {
    void execute(Runnable command);
}
```
Executor接口只有一个execute方法，用来替代通常创建或启动线程的方法。对于不同的Executor实现，execute()方法可能是创建一个新线程并立即启动，也有可能是使用已有的工作线程来运行传入的任务，也可能是根据设置线程池的容量或者阻塞队列的容量来决定是否要将传入的线程放入阻塞队列中或者拒绝接收传入的线程。
#### ExecutorService接口
ExecutorService接口继承自Executor接口，提供了管理终止的方法，以及可为跟踪一个或多个异步任务执行状况而生成 Future 的方法。增加了shutDown()，shutDownNow()，invokeAll()，invokeAny()和submit()等方法。如果需要支持即时关闭，也就是shutDownNow()方法，则任务需要正确处理中断。
#### ScheduledExecutorService接口
ScheduledExecutorService扩展ExecutorService接口并增加了schedule方法。调用schedule方法可以在指定的延时后执行一个Runnable或者Callable任务。ScheduledExecutorService接口还定义了按照指定时间间隔定期执行任务的scheduleAtFixedRate()方法和scheduleWithFixedDelay()方法。
### 基本原理（Javadoc）
线程池解决了两个问题：对于大量异步任务的执行，它通常提供了更好的性能，因为减少了每个任务调用的开销（译注：任务调用的开销与任务运行的开销不同，前者是指安排任务开始运行所需的开销，而后者是指任务运行期间的开销，这是由任务本身的逻辑和具体实现决定的）；并且它提供了一种方法来限制和管理运行一批任务时所消耗的资源。此外，它也维护一些基础的统计信息，如已完成的任务数。
#### core and maximum pool size
ThreadPoolExecutor 会根据 corePoolSize 和 maximumPoolSize 设置的界限自动调整线程池的大小。当在 execute(Runnable) 方法中提交了新任务，且正在运行的线程数量少于 corePoolSize 的时候，就创建新线程来处理刚提交的请求（新任务直接交给线程运行，不会进入队列），即使其他工作线程是空闲的。如果有多于 corePoolSize 且少于 maximumPoolSize 个线程在运行，仅当队列已满时才会创建新线程。通过将 corePoolSize 和 maximumPoolSize 设置为同样的值，就可以创建固定大小（fixed-size）的线程池。通过将 maximumPoolSize 设置为一个本质上无界限的值如 Integer.MAX_VALUE，来允许线程池容纳任意数量的并发任务。
#### 按需创建
默认情况下，连 core threads 也仅在新任务到达时才被创建和启动，但这可以使用 prestartCoreThread() 或 prestartAllCoreThreads() 方法来动态地覆盖。如果你用非空队列来构造线程池，你很可能想要预先启动线程。
#### keep-alive time
如果当前线程池中的线程数多于 corePoolSize，那么当那些过量的线程（excess threads）空闲的时间超过 keepAliveTime 时它们就会被终止。当线程池未被活跃地使用时，这提供了一个降低资源消耗的手段。如果后来线程池变得更加活跃了，新线程将会被创建。将此参数设置为 Long.MAX_VALUE TimeUnit.NANOSECONDS 实际上禁止空闲线程在 Executor 被关闭(shut down)之前被终止。默认情况下，keep-alive 策略仅当线程数多于 corePoolSize 时才应用。但是 allowCoreThreadTimeOut(boolean) 方法可以将这个超时策略也应用于核心线程(core threads)，只要 keepAliveTime 值非零。
#### 任务排队 (阻塞队列)
任何 BlockingQueue 都可用来传递和保存提交的任务。这个队列的使用和线程池大小的设定(pool sizing)相互作用：
1. 若正在运行的线程数少于 corePoolSize，Executor 总是优先添加新线程，而非让任务排队
2. 若 corePoolSize 或更多个线程正在运行，Executor 总是优先让请求进入队列排队，而非添加新线程
3. 如果请求无法入队（如队列已满），就创建新线程。除非这将导致工作线程总数超过 maximumPoolSize，在这种情况下，任务就会被拒绝。

有三个通用的排队策略：
1. 直接手递手传递任务。此时，工作队列的一个不错的默认选择就是 SynchronousQueue，它不保存任务，而是直接将任务交给线程运行。
2. 使用无界队列(Unbounded queue)传递任务。如果使用无界队列（例如有预定义容量的 LinkedBlockingQueue），那么当所有 corePoolSize 个线程都在忙时新任务就在队列中等待。如此，任何时候都不会有多于 corePoolSize 个线程被创建。也就是说，maximumPoolSize 的值不起作用了。
3. 使用有界队列(bounded queue)。将有界队列（例如 ArrayBlockingQueue）和有限的(finite) maximumPoolSizes 值一起使用可防止资源耗尽，但可能会更加难于调整和控制（tune and control），因为队列大小和线程池的最大大小之间要互相协调/平衡。
    + 如果要想降低系统资源的消耗（包括CPU的使用率，操作系统资源的消耗，上下文环境切换的开销等）, 可以设置较大的队列容量和较小的线程池容量, 但这样也会降低线程处理任务的吞吐量。
    + 如果提交的任务经常发生阻塞，那么可以考虑通过调用setMaximumPoolSize() 方法来重新设定线程池的容量。
    + 如果队列的容量设置的较小，通常需要将线程池的容量设置大一点，这样CPU的使用率会相对的高一些。但如果线程池的容量设置的过大，则在提交的任务数量太多的情况下，并发量会增加，那么线程之间的调度就是一个要考虑的问题，因为这样反而有可能降低处理任务的吞吐量。

#### 被拒绝的任务
当 Executor 被关闭(shut down)，或者对最大线程数和工作队列容量都指定了有限的(finite)界限且线程池和工作队列都已饱和的时候，在 execute(Runnable) 方法中提交的新任务将会被拒绝。在每种情况下，execute 方法都会调用 RejectedExecutionHandler.rejectedExecution(Runnable, ThreadPoolExecutor) 方法来处理被拒绝的任务。

### ThreadPoolExecutor 
ThreadPoolExecutor继承自AbstractExecutorService，也是实现了ExecutorService接口，它维护了一个工作线程集合和一个阻塞队列，以达到线程池的目的。核心构造方法如下所示：
```java
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
#### 配置参数
ThreadPoolExecutor 类中字段比较多，但只有一部分是用户可以通过构造方法的参数来控制的。有下面这些字段用户可以控制：
##### corePoolSize
核心线程数量，当有新任务在execute()方法提交时，会执行以下判断：
1. 如果运行的线程少于 corePoolSize，则创建新线程来处理任务，即使线程池中的其他线程是空闲的；
2. 如果线程池中的线程数量大于等于 corePoolSize 且小于 maximumPoolSize，则只有当workQueue满时才创建新的线程去处理任务；
3. 如果设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池的大小是固定的，这时如果有新任务提交，若workQueue未满，则将请求放入workQueue中，等待有空闲的线程去从workQueue中取任务并处理；
4. 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue已经满了，则通过handler所指定的策略来处理任务；
所以，任务提交时，判断的顺序为 corePoolSize --> workQueue --> maximumPoolSize。
##### maximumPoolSize
最大池大小，也就是工作者的最大数量。注意，实际的最大数量在内部还被 CAPACITY 常量限制。
##### keepAliveTime
空闲线程等待工作的超时时间。当线程数多于 corePoolSize 时或若 allowCoreThreadTimeOut 为真，就会使用这个超时时间。否则，他们会永远等待新的工作。
##### allowCoreThreadTimeOut
若为 false（默认值），核心线程(core threads)会一直存活，即使它们空闲着。若为 true，核心线程就使用 keepAliveTime 作为超时时间等待新工作。核心线程(core threads)指线程数不大于 corePoolSize 时仍存活着的线程。
##### workQueue 
用来保存任务并将其交给工作线程的阻塞队列。阻塞队列的具体类型可以自己选择，有这么几种：(队列的选择及大小与工作线程数的权衡可见[ 任务排队](# 任务排队))
1. ArrayBlockingQueue：数组支持的（array-backed）有界（bounded）阻塞队列，FIFO。
2. LinkedBlockingDeque：链表支持的可选有界(optionally-bounded)阻塞队列，FIFO。
3. SynchronousQueue：不存储元素的无界阻塞队列。每一个插入操作必须等待一个对应的由另一个线程执行的移除操作，反之亦然。也就是说，每个插入操作必须等到另一个线程调用移除操作，否则插入操作会一直阻塞。它的吞吐量要高于 LinkedBlockingQueue。
4. PriorityBlockingQueue：基于平衡二叉堆（balanced binary heap）的无界优先级队列。

##### threadFactory
它是ThreadFactory类型的变量，用来创建新线程。默认使用Executors.defaultThreadFactory() 来创建线程。使用默认的ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。
##### handler (RejectedExecutionHandler)
线程池和任务队列都已满时对新来的任务所采取的策略。
1. AbortPolicy：对于被拒绝的任务，直接抛出 RejectedExecutionException
2. CallerRunsPolicy：在调用 execute 方法的线程中直接运行被拒绝的任务，除非当前 executor 被关闭(shut down)，在这种情况下，任务会被丢弃。
3. DiscardPolicy：默默地丢弃掉被拒绝的任务
4. DiscardOldestPolicy：丢弃最老的未处理请求，然后重新调用 execute 方法，除非当前 executor 被关闭(shut down)，在这种情况下，任务会被丢弃。
5. 当然你也可以自定义策略。

### 参考
[ThreadPoolExecutor源码分析](http://konglong.me/post/ThreadPoolExecutor%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%8A%B6%E6%80%81/)
[深入理解 Java 线程池：ThreadPoolExecutor](https://juejin.im/entry/58fada5d570c350058d3aaad)

