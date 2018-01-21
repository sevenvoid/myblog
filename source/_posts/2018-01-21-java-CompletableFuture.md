---
title: Java 8 特性：CompletableFuture
date: 2018/01/21
categories:
- 函数式编程
tags:
- Java 8
- 函数式编程
- stream
- CompletableFuture
---

![](/images/2018-01/1516518138439.png)

在之前的文章中，介绍了Java 8 的新特性之一流，这篇文章介绍另一个新的特性 [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)。
### 什么是 CompletableFuture
CompletbaleFuture 主要用在 Java 的异步编程中，异步编程是指在一个独立的线程而非主线程中通过运行一个任务来执行非阻塞的代码，并告知主线程它的运行过程，完成或失败情况。通过这种方式，主线程就不需要阻塞或者等待这个任务的完成，它可以并行的执行其他的任务，从而提高程序的性能。

CompletableFuture 接口是 [Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html) 接口的扩展，Future 接口代表了一个异步执行任务产生的结果引用，可以通过 isDone() 方法来判断任务是否执行完成，通过get() 等方法来阻塞获取完成结果。但是 Future 还有一些局限性：
<!-- more --> 
1. 它不能手动完成：也就是说如果异步任务一直阻塞无法返回，则获取结果的线程也会一直阻塞。
2. 不能对未来的结果做进一步的处理而不会阻塞：Future 没有提供回调的能力，任务完成后不会得到通知，只有 get() 方法一直阻塞到获取到结果时，才能对结果做进一步处理。
3. 多个 Future 不能链接起来形成 Future Chains：通过 Future 无法创建一个异步处理的工作流水线。
4. 不能联合多个 Future 结果：不能实现等到多个 Future 都完成时，再执行某个动作。
5. 没有异常处理机制： Future 没有提供任何的异常处理机制。

而使用 CompletableFuture 均可以实现以上的功能。CompletableFuture 同时实现了Future 和 [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) 接口。CompletionStage 表示是一个异步计算的阶段，当另一个CompletionStage完成时，它可以被触发执行某个动作，或者计算一个值。一个CompletionStage 一旦完成它的计算就结束了，但可能会触发其他依赖它的 CompletionStage 的执行。

CompletableFuture 也正是通过实现CompletionStage接口而实现了强大的异步处理能力。

### CompletableFuture API 
[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 类中提供了有五六十个方法，每个方法签名都比较长，但是还是有很多的类似性，可以通过前缀或后缀，或者参数来区别它们。

#### 开启异步操作
通过 Runnable 接口使用 runAsync方法，返回`CompletableFuture<Void>` 类型。
通过 Supplier 接口使用 supplyAsync 方法，返回`CompletableFuture<T>` 类型。
![](/images/2018-01/1516522002025.png)

#### 链式异步操作
通过 Runnable 接口使用 thenRun()方法，返回`CompletableFuture<Void>`类型。
通过 Consumer 接口使用 thenAccept() 方法，返回`CompletableFuture<Void>` 接口。
通过 Function 接口使用 thenApply() 方法，返回`CompletableFuture<R>` 类型。
![](/images/2018-01/1516522561934.png)

链式的 CompletableFuture 的效果等价于在一个 CompletableFuture 执行完成事件之后添加一个回调，如果将这种模式应用于所有的计算，那么最终会得到一个**完全异步的（或者是响应式 Reactive 的）**应用程序，这个应用程序可以非常强大且可扩展。
#### 联合异步操作
CompletableFuture类 也提供了一些方法来实现一个异步操作完成后，执行另一个操作从而产生一个新的CompletableFuture，列如如下所示：
```java
<U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
```
thenCombine 方法接受另一个 CompletableFuture，并在这两个future 完成后会将这两个结果传递给 fn 函数，从而产生一个新的CompletableFuture。

#### 异步和非异步
正如 [Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) 所提到的，我们可以完全控制这些方法如何异步的执行：

> Actions supplied for dependent completions of non-async methods may be performed by the thread that completes the current CompletableFuture, or by any other caller of a completion method.
> 
> All async methods without an explicit Executor argument are performed using the ForkJoinPool.commonPool() (unless it does not support a parallelism level of at least two, in which case, a new Thread is created to run each task)

这里完整的列出了类中所有的方法，分为异步或非异步执行，异步执行可以显示的传递一个 Executor 执行器：
![](/images/2018-01/1516524534228.png)

#### thenCompose 方法
CompletableFuture 也提供了一个类似于flatMap 函数的 thenCompose方法，它的方法签名如下所示，这个方法接受一个函数式接口 fn，fn函数将会接受一个 T 类型的参数，并返回一个 `CompletionStage<U>`的返回值，不同于 thenApply方法，thenApply 方法会将 fn 函数的返回值包装成一个 CompletionStage 返回，但该方法返回不会再次包装 fn 函数的返回值，而是直接将 fn 函数返回的 CompletionStage 直接返回。
```java
<U> CompletionStage<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)
``` 
#### 异常处理
CompletableFuture 提供了whenComplete 和 handle 方法来处理异常。

### 参考
[Java CompletableFuture 详解](http://colobu.com/2016/02/29/Java-CompletableFuture/)
[Introduction to CompletionStage and CompletableFuture](http://millross-consultants.com/completion-stage-future-introduction.html)
[Error propagation in CompletableFuture/CompletionStage pipelines](http://millross-consultants.com/completable-future-error-propagation.html)
[Java 8 Concurrency - CompletableFuture in practice](http://www.marccarre.com/2016/05/08/java-8-concurrency-completablefuture-in-practice.html)
