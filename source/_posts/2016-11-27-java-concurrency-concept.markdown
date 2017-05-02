---
title: Java并发编程系列（三）：回归线程本质
date: 2016/11/27
categories:
- 并发编程
tags:
- Java
- 并发编程
- 线程安全
- 数据同步
---
上一篇文章开篇就讲了用synchronized实现的并发访问的同步方式与原理，这篇探本溯源来看看与线程以及并发相关的最基础的东西。

进程与线程
===
现代操作系统在运行一个程序时，会为其创建一个进程。例如，启动一个Java程序，操作系统就会创建一个Java进程。进程是指正在运行的程序，是一个独立的运行环境，它是系统进行资源分配和调度的一个独立单位，进程被创建时，会获得独立的内存单元，且进程与进程之间用户不能直接访问。现代操作系统调度的最小单元是线程，也叫轻量级进程（Light Weight Process），在一个进程里可以创建多个线程，这些线程都拥有各自的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量，它是进程调度的基本单位。例如：启动一个Java程序时，进入该程序的main方法时，系统就会创建一个相应的线程。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。

线程的实现
===
并发不一定要依赖多线程（ 如PHP中很常见的多进程并发） ， 但是在Java里面谈论并发， 大多数都与线程脱不开关系。那么线程是如何具体实现的呢？
## OS层实现
线程是比进程更轻量级的调度执行单位， 线程的引入， 可以把一个进程的资源分配和执行调度分开， 各个线程既可以共享进程资源（ 内存地址、 文件I/O等） ， 又可以独立调度（ 线程是CPU调度的基本单位） 。在OS层面，实现线程主要有3种方式： 使用内核线程实现、 使用用户线程实现和使用用户线程加轻量级进程混合实现。
<!--more-->
### 使用内核线程实现
内核线程（ Kernel-Level Thread,KLT） 就是直接由操作系统内核（ Kernel， 下称内核） 支持的线程， 这种线程由内核来完成线程切换， 内核通过操纵调度器（ Scheduler） 对线程进行调度， 并负责将线程的任务映射到各个处理器上。 每个内核线程可以视为内核的一个分身， 这样操作系统就有能力同时处理多件事情， 支持多线程的内核就叫做多线程内核（Multi-Threads Kernel） 。<br/>
程序一般不会直接去使用内核线程， 而是去使用内核线程的一种高级接口——轻量级进程（ Light Weight Process,LWP） ， 轻量级进程就是我们通常意义上所讲的线程， 由于每个轻量级进程都由一个内核线程支持， 因此只有先支持内核线程， 才能有轻量级进程。 这种轻量级进程与内核线程之间1:1的关系称为一对一的线程模型， 如图所示。
![](/images/2016-11/java_thread_01.png)
由于内核线程的支持， 每个轻量级进程都成为一个独立的调度单元， 即使有一个轻量级进程在系统调用中阻塞了， 也不会影响整个进程继续工作， 但是轻量级进程具有它的局限性： 首先， 由于是基于内核线程实现的， 所以各种线程操作， 如创建、 析构及同步， 都需要进行系统调用。 而系统调用的代价相对较高， 需要在用户态（ User Mode） 和内核态（ Kernel Mode） 中来回切换。 其次， 每个轻量级进程都需要有一个内核线程的支持， 因此轻量级进程要消耗一定的内核资源（ 如内核线程的栈空间） ， 因此一个系统支持轻量级进程的数量是有限的。
### 使用用户线程实现
从广义上来讲， 一个线程只要不是内核线程， 就可以认为是用户线程（ User Thread,UT） ， 因此， 从这个定义上来讲， 轻量级进程也属于用户线程， 但轻量级进程的实现始终是建立在内核之上的， 许多操作都要进行系统调用， 效率会受到限制。而狭义上的用户线程指的是完全建立在用户空间的线程库上， 系统内核不能感知线程存在的实现。 用户线程的建立、 同步、 销毁和调度完全在用户态中完成， 不需要内核的帮助。 如果程序实现得当， 这种线程不需要切换到内核态， 因此操作可以是非常快速且低消耗的， 也可以支持规模更大的线程数量， 部分高性能数据库中的多线程就是由用户线程实现的。 这种进程与用户线程之间1： N的关系称为一对多的线程模型， 如图所示。
![](/images/2016-11/java_thread_02.png)
使用用户线程的优势在于不需要系统内核支援， 劣势也在于没有系统内核的支援， 所有的线程操作都需要用户程序自己处理。 线程的创建、 切换和调度都是需要考虑的问题， 而且由于操作系统只把处理器资源分配到进程， 那诸如“阻塞如何处理”、“多处理器系统中如何将线程映射到其他处理器上” 这类问题解决起来将会异常困难， 甚至不可能完成。 因而使用用户线程实现的程序一般都比较复杂， 除了以前在不支持多线程的操作系统中（ 如DOS） 的多线程程序与少数有特殊需求的程序外， 现在使用用户线程的程序越来越少了，Java、 Ruby等语言都曾经使用过用户线程， 最终又都放弃使用它。
### 使用用户线程加轻量级进程混合实现
线程除了依赖内核线程实现和完全由用户程序自己实现之外， 还有一种将内核线程与用户线程一起使用的实现方式。 在这种混合实现下， 既存在用户线程， 也存在轻量级进程。 用户线程还是完全建立在用户空间中， 因此用户线程的创建、 切换、 析构等操作依然廉价， 并且可以支持大规模的用户线程并发。 而操作系统提供支持的轻量级进程则作为用户线程和内核线程之间的桥梁， 这样可以使用内核提供的线程调度功能及处理器映射， 并且用户线程的系统调用要通过轻量级线程来完成， 大大降低了整个进程被完全阻塞的风险。 在这种混合模式中， 用户线程与轻量级进程的数量比是不定的， 即为N： M的关系， 如图所示， 这种就是多对多的线程模型。（许多的协程实现就采用了这种方式）
![](/images/2016-11/java_thread_03.png)
## Java线程的实现
Java线程在JDK 1.2之前， 是基于称为“ 绿色线程” （ Green Threads） 的用户线程实现的， 而在JDK 1.2中，线程模型替换为基于操作系统原生线程模型来实现。 因此， 在目前的JDK版本中， 操作系统支持怎样的线程模型，在很大程度上决定了Java虚拟机的线程是怎样映射的， 这点在不同的平台上没有办法达成一致， 虚拟机规范中也并未规定Java线程需要使用哪种线程模型来实现。 线程模型只对线程的并发规模和操作成本产生影响， 对Java程序的编码和运行过程来说， 这些差异都是透明的。对于Sun JDK来说， 它的Windows版与Linux版都是使用一对一的线程模型实现的， 一条Java线程就映射到一条轻量级进程之中， 因此Windows和Linux系统提供的线程模型就是一对一的。
## Java线程的调度
线程调度是指系统为线程分配处理器使用权的过程， 主要调度方式有两种， 分别是协同式线程调度（ Cooperative Threads-Scheduling） 和抢占式线程调度（ Preemptive Threads-Scheduling） 。如果使用协同式调度的多线程系统， 线程的执行时间由线程本身来控制， 线程把自己的工作执行完了之后， 要主动通知系统切换到另外一个线程上。 协同式多线程的最大好处是实现简单， 而且由于线程要把自己的事情干完后才会进行线程切换， 切换操作对线程自己是可知的， 所以没有什么线程同步的问题。  它的坏处也很明显： 线程执行时间不可控制， 甚至如果一个线程编写有问题， 一直不告知系统进行线程切换， 那么程序就会一直阻塞在那里。如果使用抢占式调度的多线程系统， 那么每个线程将由系统来分配执行时间， 线程的切换不由线程本身来决定（ 在Java中， Thread.yield() 可以让出执行时间， 但是要获取执行时间的话， 线程本身是没有什么办法的） 。 在这种实现线程调度的方式下， 线程的执行时间是系统可控的， 也不会有一个线程导致整个进程阻塞的问题， Java使用的线程调度方式就是抢占式调度。 <br/>
虽然Java线程调度是系统自动完成的， 但是我们还是可以“建议”系统给某些线程多分配一点执行时间， 另外的一些线程则可以少分配一点——这项操作可以通过设置线程优先级来完成。 Java语言一共设置了10个级别的线程优先级（ Thread.MIN_PRIORITY至Thread.MAX_PRIORITY） ， 在两个线程同时处于Ready状态时， 优先级越高的线程越容易被系统选择执行。不过， 线程优先级并不是太靠谱， 原因是Java的线程是通过映射到系统的原生线程上来实现的， 所以线程调度最终还是取决于操作系统。
## Java线程状态转换
一个线程在创建直到被销毁前在生命周期内会经历不同的状态，在一个给定的时刻，线程只能处于其中的一个状态，如下所示，显示了线程生命周期内的各个状态：
![](/images/2016-11/java_thread_04.png)
看一个简单的例子：
```java
package edu.sevenvoid.jvm.thread;
import java.util.concurrent.TimeUnit;
/**
 * @author Sevenvoid
 * @since 2016-11-28
 */
public class ThreadState {

public static void main(String[] args) {
	new Thread(new TimeWaiting(), "TimeWaitingThread").start();
	new Thread(new Waiting(), "WaitingThread").start();
	// 使用两个Blocked线程，一个获取锁成功，另一个被阻塞
	new Thread(new Blocked(), "BlockedThread-1").start();
	new Thread(new Blocked(), "BlockedThread-2").start();
}
// 该线程不断地进行睡眠
static class TimeWaiting implements Runnable {
	@Override
	public void run() {
		while (true) {
			try {
				TimeUnit.SECONDS.sleep(100);
			} catch (InterruptedException e) {
			}
		}
	}
}
// 该线程在Waiting.class实例上等待
static class Waiting implements Runnable {
	@Override
	public void run() {
		while (true) {
			synchronized (Waiting.class) {
				try {
					Waiting.class.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}
	}
}
// 该线程在Blocked.class实例上加锁后，不会释放该锁
static class Blocked implements Runnable {
	public void run() {
		synchronized (Blocked.class) {
			while (true) {
				try {
					TimeUnit.SECONDS.sleep(100);
				} catch (InterruptedException e) {
				}
			}
		}
	}
}
}
```
使用jps与jstack命令可以查看到程序执行时此刻的堆栈日志：从日志中可以明显看到各个线程所处于的各自状态。
```java
"BlockedThread-2" #13 prio=5 os_prio=0 tid=0x000000001aeb6800 nid=0x1774 waiting for monitor entry [0x000000001bcaf000]
   java.lang.Thread.State: BLOCKED (on object monitor) 
   //BlockThread-2获取不到锁将阻塞
        at edu.sevenvoid.jvm.thread.ThreadState$Blocked.run(ThreadState.java:54)
        - waiting to lock <0x00000007805b8908> (a java.lang.Class for edu.sevenvoid.jvm.thread.ThreadState$Blocked)
        at java.lang.Thread.run(Thread.java:745)

"BlockedThread-1" #12 prio=5 os_prio=0 tid=0x000000001aeb4800 nid=0x207c waiting on condition [0x000000001bbae000]                  
	//BlockThread-1获取到锁后执行，但会超时等待
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:340)
        at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
        at edu.sevenvoid.jvm.thread.ThreadState$Blocked.run(ThreadState.java:54)
        - locked <0x00000007805b8908> (a java.lang.Class for edu.sevenvoid.jvm.thread.ThreadState$Blocked)
        at java.lang.Thread.run(Thread.java:745)

"WaitingThread" #11 prio=5 os_prio=0 tid=0x000000001aeae800 nid=0x2798 in Object.wait() [0x000000001baaf000]
   java.lang.Thread.State: WAITING (on object monitor)   
   	//WaitingThread线程在实例Waiting.class上等待
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000007805b71b0> (a java.lang.Class for edu.sevenvoid.jvm.thread.ThreadState$Waiting)
        at java.lang.Object.wait(Object.java:502)
        at edu.sevenvoid.jvm.thread.ThreadState$Waiting.run(ThreadState.java:39)
        - locked <0x00000007805b71b0> (a java.lang.Class for edu.sevenvoid.jvm.thread.ThreadState$Waiting)
        at java.lang.Thread.run(Thread.java:745)

"TimeWaitingThread" #10 prio=5 os_prio=0 tid=0x000000001aead800 nid=0x30c4 waiting on condition [0x000000001b9af000]         
	//TimeWaitingThread线程处于超时等待
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:340)
        at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
        at edu.sevenvoid.jvm.thread.ThreadState$TimeWaiting.run(ThreadState.java:25)
        at java.lang.Thread.run(Thread.java:745)

"Service Thread" #9 daemon prio=9 os_prio=0 tid=0x000000001ae4a000 nid=0x3640 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread2" #8 daemon prio=9 os_prio=2 tid=0x000000001adbf000 nid=0x35c8 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
更详细的线程状态转换如图所示：
![](/images/2016-11/java_thread_05.png)
注意：线程使用lock接口实现的加锁解锁会使线程进入等待或超时等待状态，而非阻塞状态。理解了线程的各个状态，以及状态之间的转换，在实际使用时，我们就可以通过线程的堆栈日志分析线上的环境问题。
## Java线程安全
了解了线程是如何实现的以及它的生命周期之后，再来看一个最核心的问题，什么是线程安全？在并发编程中我们需要明白的是，尽管我们可以写出多个线程之间的运行互相可以不受干扰的代码，但线程之间的交互还是不可避免的，这就需要我们确保它们之间的交互是安全的，不会出现错误的数据，脏数据。因此线程安全，就是**数据安全**，本质上也就是管理对状态（数据）的访问，这些状态是共享的、可变的状态。一个对象的状态包含了任何会对它外部可见行为产生影响的数据。所谓共享，是指一个变量可以被多个线程访问；所谓可变，是指变量值在其生命周期内是可以改变的。我们讨论线程安全性好像是关于代码的，但我们真正要做的，是在不可控制的并发访问中**保护数据**。一个被并发访问的对象，线程安全的这个性质，取决于程序中如何使用对象， 而不是对象完成了什么。
### 线程保护对象
有如下方式确保线程安全的访问数据：
1. 不要跨线程共享变量：无同步(ThreadLocal)
2. 使状态变量为不可变的
3. 在任何访问状态变量的时候使用同步：互斥同步(synchronized)、阻塞同步(Lock)
4. 或者确保复合操作是原子性的：如比较改写操作、读改写操作

### 对象保护自己
有如下方式确保一个对象对多线程来说它发布之后就是安全的对象：
1. 使之成为不可变得对象
2. 将共享的域使用volatile或者加锁保护
3. 线程封闭使用ThreadLocal
4. 防止未正确构造的对象被发布造成**对象逃逸**

上面的各种手段只是建议，再具体使用时需要结合业务认真考虑。对象逃逸也是比较常见而且被大多数程序员容易忽视的问题。

## 参考
[深入理解Java虚拟机](https://book.douban.com/subject/6522893/)
[Java并发编程的艺术](https://read.douban.com/ebook/11771712/?dcs=book-search)
Java Concurrent In Practice