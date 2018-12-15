---
title: ThreadPoolExecutor —— 源码分析
date: 2018/01/28
comments: true
categories:
- 并发编程
tags:
- Java
- 并发编程
- 线程池
- ThreadPool
- Executor
---

**声明：这两篇有关线程池 ThreadPoolExecutor 的分析文章，非原创内容，是自己先阅读了几遍源码并分析后，然后基于网上的分享的梳理总结，请看参考资源。请你自己一定要真正的去阅读源码，才能理解得更深，因为我也做到了**

上一篇文章详细介绍了Executor框架的原理，以及创建线程池时的核心参数。有了理论知识做铺垫，那我们再来看看具体的源码实现。
### 线程池状态
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
<!-- more -->
线程池的控制状态（control state），简称为 ctl，是一个 atomic integer，它打包了两个概念性的字段：
+ workerCount, 表示工作线程的有效数量
+ runState, 表示线程池运行的状态

总共有 5 个状态：
+ RUNNING：可接受新的任务，也可处理阻塞队列里的任务
+ SHUTDOWN：不再接受新的任务，但可以处理阻塞队列里的任务，在线程池处于 RUNNING 状态时，调用 shutdown()方法会使线程池进入到该状态。（finalize() 方法在执行过程中也会调用shutdown()方法进入该状态）
+ STOP：不再接受新的任务，也不会处理阻塞队列里的任务，且会中断正在处理的任务，在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
+ TIDYING：过渡状态。所有的任务都终止了（可能是执行完了，也可能是被中断了），有效线程数 workerCount 也为 0 了，此时线程池的状态就转变为 TIDYING，并且将要调用钩子方法 terminated()。
+ TERMINATED：终止状态。terminated() 方法调用完成后的状态

这些值按照数值大小是有序的，这点很重要，可以进行次序比较。runState 随着时间单调递增，但不需要走过每一个状态。请看状态变迁图：
![](/images/2018-01/1516894611427.png)

#### 相关源码分析
ctl 的低 29 位表示 workerCount，高 3 位表示 runState。这样我们就将 workerCount 限制为最多 (2^29)-1 个线程(大约 5 亿个)。

workerCount 就是被允许启动(start)，但未被允许停止(stop)的线程的数量。

现在来看看具体如何表示这两个概念的。整型中 32 位的前 3 位用来表示状态，后 29 位表示有效线程数，也被称为线程池容量。
```java
// Integer.SIZE 就是二进制补码形式的 int 值使用的二进制位数，即 32
// 线程池容量所占用的位数
private static final int COUNT_BITS = Integer.SIZE - 3;
```
为了方便描述和理解，使用符号 {<0|1>:n 位} 表示 0 或 1 有多少位。
线程池容量大小为 1 << 29 - 1，1 << 29 结果是 001{0:29位}，再减 1 就是 000{1:29位}，这个值等于 (2^29)-1。有人可能会问，(2^29) 为啥要再减 1 呢？因为 (2^29) 是从 0 开始能表示多少个数，减去 1 才是能表示的最大的数。代码如下：
```java
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```
**负数在计算机中使用二进制补码来表示。如何计算补码？先将十进制数表示成二进制，所有位取反，再加 1。**

**RUNNING 状态**，-1 使用 1 的补码表示就是 {1:32位}，-1 << 29 = {1:32位} << 29 = 111{0:29位}，前 3 位为 111：
```java
private static final int RUNNING    = -1 << COUNT_BITS;
```
**SHUTDOWN 状态**，0 << 29 = {0:32位} << 29 = {0:32位}，前 3 位为 000：
```java
private static final int SHUTDOWN   =  0 << COUNT_BITS;
```
**STOP 状态**，1 << 29 = {0:31位}1 << 29 = 001{0:29位}，前 3 位为 001：
```java
private static final int STOP       =  1 << COUNT_BITS;
```
**TIDYING 状态**，2 << 29 = {0:30位}10 << 29 = 010{0:29位}，前 3 位为 010：
```java
private static final int TIDYING    =  2 << COUNT_BITS;
```
**TERMINATED 状态**，3 << 29 = {0:30位}11 << 29 = 011{0:29位}，前 3 位为 011：
```java
private static final int TERMINATED =  3 << COUNT_BITS;
```
这些状态值使用了高 3 位，低 29 位全部是 0。搞清楚状态位后，先来看看状态和有效线程数的初始化：
```java
// 这个字段就是线程池的 control state
// 状态初始化为 RUNNING，有效线程数为 0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
接着看看获取、设置状态和有效线程数的方法：
```java
// 注意，对于下面这些方法，传给形参 c 的实参就是 ctl 字段的 int 值

// CAPACITY 为 000{1:29位}，按位求反(~ 操作符)后的结果是 111{0:29位}
// 按位或的结果就是 {ctl的高三位}{0:29位}
   private static int runStateOf(int c)     { return c & ~CAPACITY; }

// 返回结果为 000{ctl的低29位}
   private static int workerCountOf(int c)  { return c & CAPACITY; }

// 将 runState 和 workerCount 打包为一个 int 值，用来设置 ctl 字段
// 取 rs 的高 3 位和 wc 的低 29 位，组成一个 32 位的整数值
   private static int ctlOf(int rs, int wc) { return rs | wc; }
```
对状态进行比较的方法：
```java
// 对于下面这些方法, 传给 c 的是 ctl 的 int value，s 是状态常量
// 将 ctl 表示的当前状态跟给定的状态常量进行比较
// 能直接比较的原因就在于本身高3位的 runState 的值是有大小的，可以忽略低29位的值
   private static boolean runStateLessThan(int c, int s) {
       return c < s;
   }

   private static boolean runStateAtLeast(int c, int s) {
       return c >= s;
   }

   private static boolean isRunning(int c) {
       return c < SHUTDOWN;
   }
```
### 工作线程集合和工作队列
```java
  private final HashSet<Worker> workers = new HashSet<Worker>();
  private final ReentrantLock mainLock = new ReentrantLock();
  private final Condition termination = mainLock.newCondition();
  private final BlockingQueue<Runnable> workQueue;
```
**workers** 表示线程池中所有的有效工作线程集合，这是一个非线程安全地集合，因此，访问该集合时需要获取到 mainLock 锁。

**mainLock** 锁，用于给访问workers 集合和处理相关的簿记工作时加锁。这里的workers 没有使用某种类型的并发集合的主要原因是序列化的 interruptIdleWorkers，可以避免不必要的中断风暴，特别是在SHOTDOWN期间，否则，退出线程将同时中断那些尚未中断的线程。它还简化一些相关的统计簿记工作，比如 largestPoolSize 等。在SHOTDOWN和SHOTDOWNNOW期间也需要获取该 mainLock 锁，为了确保 workers 集合的稳定。具体可以看稍后的源码分析。

**workQueue**工作队列，用来存放提交的待处理的任务。

### 重要抽象 Worker
线程池中的线程是通过Worker 类来表示的，Worker 可被看作工作线程，因为它包含了工作线程中最重要的 run-loop。该类主要为运行任务的线程维护中断控制状态(interrupt control state)，以及簿记工作（如工作线程完成的任务数）。代码如下：
```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** 直接委托给 ThreadPoolExecutor.runWorker(Worker) 方法 */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        // 值 -1 表示阻止被中断，直到 runWorker 方法被调用
        // 值 0 代表已解锁状态（unlocked state）
        // 值 1 代表已锁定状态（locked state）

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            // 状态为 -1 就不能中断当前工作线程
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
Worker 还讨巧地通过继承 AbstractQueuedSynchronizer 来简化获取和释放包围运行任务的代码块的锁。换句话说，一个 Worker 实例就是一个锁，要锁住的目标就是执行任务的代码块。这样就能防止 意欲唤醒正在等待任务的线程的中断 反而中断了正在运行的任务。

Worker继承了AQS，使用AQS来实现独占锁的功能。为什么不使用ReentrantLock来实现呢？可以看到tryAcquire方法，它是不允许重入的，而ReentrantLock是允许重入的：
1. lock方法一旦获取了独占锁，表示当前线程正在执行任务中；
2. 如果正在执行任务，则不应该中断线程；
3. 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断；
4. 线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；
5. 之所以设置为不可重入，是因为我们不希望任务在调用像setCorePoolSize这样的线程池控制方法时重新获取锁。如果使用ReentrantLock，它是可重入的，这样如果在任务中调用了如setCorePoolSize这类线程池控制的方法，会中断正在运行的线程。

所以，Worker继承自AQS，用于判断线程是否空闲以及是否可以被中断。

此外，在构造方法中执行了setState(-1);，把state变量设置为-1，为什么这么做呢？是因为AQS中默认的state是0，如果刚创建了一个Worker对象，还没有执行任务时，这时就不应该被中断，看一下tryAquire方法：
```java
protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
```
tryAcquire方法是根据state是否是0来判断的，所以，setState(-1)，将state设置为-1是为了禁止在启动线程前进行中断。正因为如此，开始执行任务时再调用 Worker.unlock() 方法将state设置为0，清除锁状态，这个时候就有个机会可以去中断工作线程的执行，否则，就不允许中断一个正在执行任务的线程。具体细节可以查看 runWoker 方法分析。
### 核心逻辑
#### execute 方法
execute()方法用来提交任务，代码如下：
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
            
        int c = ctl.get();
        // 如果当前活动线程数小于corePoolSize，则新建一个线程放入线程池中；
        // 并把任务添加到该线程中。
        // addWorker 中会继续判断线程池状态，可防止shotdown()方法在这之后调用出现继续创建线程情况
        if (workerCountOf(c) < corePoolSize) {
	        /*
	         * addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断；
	         * 如果为true，根据corePoolSize来判断；
	         * 如果为false，则根据maximumPoolSize来判断
	         */
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
         /*
		  * 如果当前线程池是运行状态并且任务添加到队列成功
	      */
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了，
	        // 这时需要移除该command
	        // 执行过后通过handler使用拒绝策略对该任务进行处理，整个方法返回
            if (! isRunning(recheck) && remove(command))
                reject(command);
            /*
	         * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
	         * 这里传入的参数表示：
	         * 1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动；
	         * 2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
	         * 如果判断workerCount大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行。
	         */
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        /*
	     * 如果执行到这里，有两种情况：
	     * 1. 线程池已经不是RUNNING状态；
	     * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
	     * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize；
	     * 如果失败则拒绝该任务
	     * addWorker 中会继续判断线程池状态
	     */
        else if (!addWorker(command, false))
            reject(command);
    }
    
	public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate(); // In case SHUTDOWN and now empty
        return removed;
    }
	final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
```
简单来说，在执行execute()方法时如果状态一直是RUNNING时，的执行过程如下：
1. 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务；
2. 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中；
3. 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务；
4. 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

这里要注意一下addWorker(null, false);，也就是创建一个线程，但并没有传入任务，因为任务已经被添加到workQueue中了，所以worker在执行的时候，会直接从workQueue中获取任务。所以，在workerCountOf(recheck) == 0时执行addWorker(null, false)，也是为了保证线程池在RUNNING状态下必须要有一个线程来执行任务。

remove 方法从内部的任务队列中移除给定的任务。每次从任务队列移除任务后都要调用 tryTerminate 方法尝试终止线程池。

#### addWorker 方法
addWorker方法的主要工作是在线程池中创建一个新的线程并执行，firstTask参数 用于指定新增的线程执行的第一个任务，core参数为true表示在新增线程时会判断当前活动线程数是否少于corePoolSize，false表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize。

这里重点梳理下该方法的返回值。返回 true 表示 worker 添加成功（包括线程的创建和启动）；返回 false 表示要么不应当添加 worker，要么线程创建失败。如果线程创建失败，会执行回滚操作。哪些情况下该方法返回 false：
+ 线程池已停止，即 (rs > SHUTDOWN)
+ 线程池已关闭，但 firstTask 不为 null，即 (rs == SHUTDOWN && firstTask != null)。因为此时线程池不再接收新任务了
+ 线程池已关闭，firstTask 为 null，任务队列也为空，即 (rs == SHUTDOWN && firstTask == null && workQueue.isEmpty())。此时已没有任何任务需要执行了
+ worker 数已达上限
+ 线程创建失败。ThreadFactory 返回 null 或者发生异常（通常都是 Thread.start() 抛出 OutOfMemoryError）
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 外层循环（主要控制状态变化）
    for (;;) {
        int c = ctl.get();
        // 获取运行状态
        int rs = runStateOf(c);

        /*
         * 这个if判断
         * 如果rs >= SHUTDOWN，则表示此时不再接收新任务；
         * 接着判断以下3个条件，只要有1个不满足，则返回false：
         * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务，如果不满足就表示线程池状态为 >= STOP 状态，这时候任何线程都不允许创建，返回false
         * 2. firstTask为空
         * 3. 阻塞队列不为空,
         * 
         * 首先考虑rs == SHUTDOWN的情况
         * 这种情况下不会接受新提交的任务，所以在firstTask不为空的时候会返回false；
         * 然后，如果firstTask为空，并且workQueue也为空，则返回false，
         * 因为队列中已经没有任务了，不需要再添加线程了
         * 否则，是队列中还有任务，但需要新添加工作线程，此时就可以继续走下面的流程
         */
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
            
		// 内层循环（主要控制 workCount 计数）
        for (;;) {
            // 获取线程数
            int wc = workerCountOf(c);
            // 如果wc超过CAPACITY，也就是ctl的低29位的最大值（二进制是29个1），返回false；
            // 这里的core是addWorker方法的第二个参数，如果为true表示根据corePoolSize来比较，
            // 如果为false则根据maximumPoolSize来比较。
            // 
            if (wc >= CAPACITY ||
	            // 再次检查是否应当添加核心/非核心线程，因为自 execute 方法中的
            	// 检查之后池的其他用户也可能提交了任务进而添加了新 worker
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
                
            // 这里使用基于 CAS 的乐观并发策略实现非阻塞同步，CAS 是先检测是否有冲突，若无再操作。
            // 若有冲突，调用方需要进行补偿，所以需要把这句代码放入循环中。
            // 如果将 workerCount 加 1 成功，就中止外层循环。
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 如果增加workerCount失败，则重新获取ctl的值
            c = ctl.get();  // Re-read ctl
            // 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
	
	// workCount 增加成功后，就执行新建和启动线程
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 根据firstTask来创建Worker对象
        w = new Worker(firstTask);
        // 每一个Worker对象都会创建一个线程
        final Thread t = w.thread;
        if (t != null) {
            // 前面已经分析过，访问 workers 集合时需要获取 mainLock 锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                // rs < SHUTDOWN表示是RUNNING状态；
                // 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                // 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // workers是一个HashSet
                    workers.add(w);
                    int s = workers.size();
                    // largestPoolSize记录着线程池中出现过的最大线程数量
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 启动线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
	    // 如果线程创建失败或启动失败，或发生了异常
        if (! workerStarted)
	        // 执行必需的回滚逻辑
            addWorkerFailed(w);
    }
    return workerStarted;
}

```
#### addWorkerFailed 方法
addWorker 方法中线程创建失败时，将会调用 addWorkerFailed 方法进行必要的回滚逻辑：
1. 从workers 集合中删除当前 worker，如果存在
2. workCount 工作线程数减一
3. 调用 tryTerminate 重新检查线程池终止状态，可能由于当前的 worker 存在阻止了线程池的终止
```java
 private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```
#### tryTerminate 方法
如果（SHUTDOWN、池和队列为空）或（STOP、池为空），则转换到TERMINATED状态。 如果有资格终止，但workerCount不为零，将中断一个空闲的工作，以确保 shutdown 信号传播。 此方法必须在任何可能会终止线程池的操作之后调用 ，包括在 shutdown 期间减少 workCount 数量或从队列中删除任务。
```java
final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            /*
	         * 当前线程池的状态为以下几种情况时，直接返回：
	         * 1. RUNNING，因为还在运行中，不能停止；
	         * 2. TIDYING或TERMINATED，因为线程池中已经没有正在运行的线程了；
	         * 3. SHUTDOWN并且等待队列非空，这时要执行完workQueue中的task；
	         */
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 如果线程数量不为0，则中断一个空闲的工作线程，并返回
            if (workerCountOf(c) != 0) { // Eligible(资格) to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
	            // 这里尝试设置状态为TIDYING，如果设置成功，则调用terminated方法
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
	                    // terminated方法默认什么都不做，留给子类实现
                        terminated();
                    } finally {
	                    // 设置状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```
#### interruptIdleWorkers 方法
此方法会中断一个正在等待任务的空闲线程，但是不会中断一个正在执行的 command，也即正在执行用户逻辑的线程。前面我们介绍 Worker 时就提到 Worker 巧妙的继承了 AQS 接口，并在构造 Worker 时会设置锁状态值为 -1 `setState(-1)`，因此除非 unlock 方法被调用，否则下面方法中的 tryLock() 方法始终获取不到锁，也就无法中断 w 线程。而 unlock() 方法的首次被调用是在 runWorker 方法中。
```java
private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
interruptIdleWorkers遍历workers中所有的工作线程，若线程没有被中断且 tryLock 成功，就中断该线程。

#### worker 如何执行任务（runWorker 方法）
ThreadPoolExecutor.runWorker 方法才是实际的线程体方法。通过 run-loop，不断地从任务队列获取任务，执行它们。在每次循环中，妥善地处理下面几件事儿：
1. 若给定的 worker 带有初始任务，就将它作为第一次循环要执行的任务。否则，就从队列获取任务。如果 getTask 方法返回 null，就表示当前工作线程必须退出(exit)，然后 run-loop 就会中止，当前线程就退出了。在退出前，会为临终的 worker 执行清理和簿记工作，就是 processWorkerExit 方法。如果是因为 runWorker 方法内发生了异常导致退出，那么标示 worker 异常结束的布尔变量 completedAbruptly 就为真，processWorkerExit 内就会使用新 worker 替代当前 worker。
2. 在运行任何任务之前，先获取锁以防止正在执行任务时发生其他池中断(pool interrupts)，然后我们确保除非池正在停止，否则此线程的中断状态不会被设置。
3. 在执行任务之前先调用钩子方法 beforeExecute，它可能会抛出异常，在这种情况下会导致当前工作线程还未运行任务就要死亡（中止循环且 completedAbruptly 为 true）了。
4. 假设 beforeExecute 方法正常完成，就运行任务，收集任何抛出的异常，并将其发送给钩子方法 afterExecute 进行处理。这里会捕获 RuntimeException, Error 和任意 Throwable。因为不能在 Runnable.run() 中重新抛出 Throwable，所以在去往线程的 UncaughtExceptionHandler 的路上将其包装为 Error。对于任何抛出的异常，这里都会保守地/谨慎地让当前线程死亡。
5. task.run() 完成后，就调用 afterExecute 方法，它也可能抛出异常，这也将导致线程死亡。根据 JLS Sec 14.20（Java语言规范14.20章节），这个异常将会生效，即使 task.run 也抛出了异常。

这个异常机制的实际作用就是给 afterExecute 和线程的 UncaughtExceptionHandler 提供关于用户代码碰到的任何问题的尽可能准确的信息。
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 获取第一个任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 允许中断
    w.unlock(); // allow interrupts
    // 是否因为异常退出循环
    boolean completedAbruptly = true;
    try {
        // 如果task为空，则通过getTask来获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 如果线程池正在停止，确保线程被中断；如果不是，再检查当前线程状态
            // Thread.interrupted()返回 false，表示线程没有中断过则跳出 if 判断
            // Thread.interrupted()返回 true，表示线程中断过，此时会清除中断状态(interrupted status)，并继续判断
            // 对于第二种情况，这需要重新检查状态以处理在清除中断状态的时候与 shutdownNow 之间的竞争。
            // 注：shutdownNow 方法会将状态推进至 STOP。
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
	            // 将 task 置为 null 很重要，
                // 否则 firstTask 不为空时永远也不会从队列获取任务
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
	    // 不管当前线程是正常还是异常退出，都要进行清理和簿记工作
        processWorkerExit(w, completedAbruptly);
    }
}
```
这里说明一下第一个if判断，目的是：
1. 如果线程池正在停止，那么要保证当前线程是中断状态
2. 如果不是的话，则要保证当前线程不是中断状态

这里又要考虑在执行该if语句期间可能也执行了shutdownNow方法，shutdownNow方法会把状态设置为STOP，回顾一下STOP状态：
> 不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态。

STOP状态要中断线程池中的所有线程，而这里使用Thread.interrupted()来判断是否中断是为了确保在RUNNING或者SHUTDOWN状态时线程是非中断状态的，因为Thread.interrupted()方法会复位中断的状态。
#### shutdownNow 方法
该方法做了下面这几件事儿：
1. 将池状态变迁为 STOP
2. 通过 Thread.interrupt 中断所有正在活跃地执行着的任务
3. 停止处理等待执行的任务，将它们从队列中排空（从队列删除）至一个列表，返回此列表。
```java
   public List<Runnable> shutdownNow() {
       List<Runnable> tasks;
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           checkShutdownAccess();
           advanceRunState(STOP);
           // 中断所有工作线程，无论是否空闲
           interruptWorkers();
           // 取出队列中没有被执行的任务
           tasks = drainQueue();
       } finally {
           mainLock.unlock();
       }
       tryTerminate();
       return tasks;
   }
```
**注意，shutdownNow 方法不会等待正在执行的任务正常结束，而且它通过 interruptWorkers 方法使用Thread.interrupt 设置中断标记位来撤销任务的执行，因此任何未能响应中断的任务仍会继续执行下去，甚至可能永远不会终止。**
#### interruptWorkers 方法
只会设置中断状态位，在这种情况下，线程有可能会继续运行。
```java
private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
// 该方法位于 Worker 中
void interruptIfStarted() {
            Thread t;
            // 仅仅设置中断状态，可能会导致线程仍然继续运行，甚至永远不会停止
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
```
#### getTask 方法
getTask() 方法很有料，它返回 null 就表示当前线程必须要退出。线程空闲时间的记录和空闲超时处理也在其中。
该方法执行阻塞(blocking)或定时(timed)等待任务，具体取决于当前的配置设置。如果因为以下任何一种情形当前 worker 必须退出就返回 null：
1. worker 数多于 maxPoolSize（由于调用 setMaximumPoolSize 方法重新设置了）
2. 线程池已停止
3. 线程池已关闭，且任务队列为空
4. 当前 worker 等待任务超时，且超时的 worker 需要终止（根据配置或线程总数），也就是说，条件 [allowCoreThreadTimeOut || workerCount > corePoolSize] 为真。

多于的线程会销毁掉，什么时候会销毁？当然是runWorker方法执行完之后，也就是Worker中的run方法执行完，由JVM自动回收。
```java
private Runnable getTask() {
    // timeOut变量的值表示上次从阻塞队列中取任务时是否超时
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        /*
         * 如果线程池状态rs >= SHUTDOWN，也就是非RUNNING状态，再进行以下判断：
         * 1. rs >= STOP，线程池是否正在stop；
         * 2. 阻塞队列是否为空。
         * 如果以上条件满足，则将workerCount减1并返回null。
         * 因为如果当前线程池状态的值是SHUTDOWN或以上时，不允许再向阻塞队列中添加任务。
         */
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 是否需要剔除超时的 worker（根据配置或线程总数）
        // timed变量用于判断是否需要进行超时控制
        // allowCoreThreadTimeOut默认是false，也就是核心线程不允许进行超时；
        // wc > corePoolSize，表示当前线程池中的线程数量大于核心线程数量；
        // 对于超过核心线程数量的这些线程，需要进行超时控制
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		
		/*
         * wc > maximumPoolSize的情况是因为可能在此方法执行阶段同时执行了setMaximumPoolSize方法；
         * timed && timedOut 如果为true，表示当前操作需要进行超时控制，并且上次从阻塞队列中获取任务发生了超时
         * 接下来判断，如果有效线程数量大于1，或者阻塞队列是空的，那么尝试将workerCount减1；
         * 如果减1失败，则返回重试。
         * 如果wc == 1时，也就说明当前线程是线程池中唯一的一个线程了。
         */
        if ( (wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty()) ) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 阻塞或定时获取任务
            Runnable r = timed ?
            	// 定时等待
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                // 阻塞等待
                workQueue.take();
            if (r != null)
                return r;

            // 此时虽然 r 为 null，但可不能直接返回 null，
            // 因为还需要根据用户的配置或线程总数来决定
            timedOut = true;
        } catch (InterruptedException retry) {
            // 走到这儿，肯定是定时获取任务超时。
            // 如果获取任务时当前线程发生了中断，则设置timedOut为false并返回循环重试
            timedOut = false;
	    }
   }
}
```
getTask方法返回null时，在runWorker方法中会跳出while循环，然后会执行processWorkerExit方法。

#### processWorkerExit 方法
processWorkerExit 方法为正在死亡的 worker 执行清理和簿记工作。对于正常退出的 worker，该方法假定 workerCount 在其他地方已被调整。
在下面几种情况下，会添加新 worker 来替代正在退出的 worker：
1. 因用户任务异常而退出
2. worker 数小于 corePoolSize
3. 任务队列非空，但没有 worker 了
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果completedAbruptly值为true，则说明线程执行时出现了异常，需要将workerCount减1；
    // 如果线程执行时没有出现异常，说明在getTask()方法中已经已经对workerCount进行了减1操作，这里就不必再减了。  
    if (completedAbruptly) 
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
    	// 累计已完成的任务数
        completedTaskCount += w.completedTasks;
        // 从workers中移除，也就表示着从线程池中移除了一个工作线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
	// 根据线程池状态进行判断是否结束线程池
    tryTerminate(); 

    int c = ctl.get();
    /*
     * 当线程池是RUNNING或SHUTDOWN状态时，如果worker是异常结束，那么会直接addWorker；
     * 如果allowCoreThreadTimeOut=true，并且等待队列有任务，至少保留一个worker；
     * 如果allowCoreThreadTimeOut=false，workerCount不少于corePoolSize。
     */
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) { // 如果是正常退出
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;

            // 如果 worker 数大于计算得出的合理最小值，
            // 就无需添加新worker来替代正在死亡的worker
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```
至此，processWorkerExit执行完之后，工作线程被销毁，以上就是整个工作线程的生命周期，从execute方法开始，Worker使用ThreadFactory创建新的工作线程，runWorker通过getTask获取任务，然后执行任务，如果getTask返回null，进入processWorkerExit方法，整个线程结束，如图所示：
![](/images/2018-01/1517151658843.png)

#### shutdown方法
shutdown方法要将线程池切换到SHUTDOWN状态，并调用interruptIdleWorkers方法请求中断所有空闲的worker，最后调用tryTerminate尝试结束线程池。
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 安全策略判断
        checkShutdownAccess();
        // 切换状态为SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 中断空闲线程
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试结束线程池
    tryTerminate();
}
```
这里思考一个问题：在runWorker方法中，执行任务时对Worker对象w进行了lock操作，为什么要在执行任务的时候对每个工作线程都加锁呢？

下面仔细分析一下：
+ 在getTask方法中，如果这时线程池的状态是SHUTDOWN并且workQueue为空，那么就应该返回null来结束这个工作线程，而使线程池进入SHUTDOWN状态需要调用shutdown方法；
+ shutdown方法会调用interruptIdleWorkers来中断空闲的线程，interruptIdleWorkers持有mainLock，会遍历workers来逐个判断工作线程是否空闲。但getTask方法中没有mainLock；
+ 在getTask中，如果判断当前线程池状态是RUNNING，并且阻塞队列为空，那么会调用workQueue.take()进行阻塞；
+ 如果在判断当前线程池状态是RUNNING后，这时调用了shutdown方法把状态改为了SHUTDOWN，这时如果不进行中断，那么当前的工作线程在调用了workQueue.take()后会一直阻塞而不会被销毁，因为在SHUTDOWN状态下不允许再有新的任务添加到workQueue中，这样一来线程池永远都关闭不了了；
+ 由上可知，shutdown方法与getTask方法（从队列中获取任务时）存在竞态条件；
+ 解决这一问题就需要用到线程的中断，也就是为什么要用interruptIdleWorkers方法。在调用workQueue.take()时，如果发现当前线程在执行之前或者执行期间是中断状态，则会抛出InterruptedException，解除阻塞的状态；
+ 但是要中断工作线程，还要判断工作线程是否是空闲的，如果工作线程正在处理任务，就不应该发生中断；
+ 所以Worker继承自AQS，在工作线程处理任务时会进行lock，interruptIdleWorkers在进行中断时会使用tryLock来判断该工作线程是否正在处理任务，如果tryLock返回true，说明该工作线程当前未执行任务，这时才可以被中断。

### 线程池的监控
通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用

+ getTaskCount：线程池已经执行的和未执行的任务总数；
+ getCompletedTaskCount：线程池已完成的任务数量，该值小于等于taskCount；
+ getLargestPoolSize：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize；
+ getPoolSize：线程池当前的线程数量；
+ getActiveCount：当前线程池中正在执行任务的线程数量。

通过这些方法，可以对线程池进行监控，在ThreadPoolExecutor类中提供了几个空方法，如beforeExecute方法，afterExecute方法和terminated方法，可以扩展这些方法在执行前或执行后增加一些新的操作，例如统计线程池的执行任务的时间等，可以继承自ThreadPoolExecutor来进行扩展。

### 总结
本文比较详细的分析了线程池的工作流程，总体来说有如下几个内容：

+ 分析了线程的创建，任务的提交，状态的转换以及线程池的关闭；
+ 这里通过execute方法来展开线程池的工作流程，execute方法通过corePoolSize，maximumPoolSize以及阻塞队列的大小来判断决定传入的任务应该被立即执行，还是应该添加到阻塞队列中，还是应该拒绝任务。
+ 介绍了线程池关闭时的过程，也分析了shutdown方法与getTask方法存在竞态条件；
+ 在获取任务时，要通过线程池的状态来判断应该结束工作线程还是阻塞线程等待新的任务，也解释了为什么关闭线程池时要中断工作线程以及为什么每一个worker都需要lock。

在向线程池提交任务时，除了execute方法，还有一个submit方法，submit方法会返回一个Future对象用于获取返回值。

### 参考
[ThreadPoolExecutor源码分析](http://konglong.me/post/ThreadPoolExecutor%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%8A%B6%E6%80%81/)
[深入理解 Java 线程池：ThreadPoolExecutor](https://juejin.im/entry/58fada5d570c350058d3aaad)