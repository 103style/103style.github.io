# Executor框架 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

### Java并发编程的艺术笔记
* [并发编程的挑战](https://www.jianshu.com/p/a2117af9950d)
* [Java并发机制的底层实现原理](https://www.jianshu.com/p/9d8147ba3092)
* [Java内存模型](https://www.jianshu.com/p/235c777d862c)
* [Java并发编程基础](https://www.jianshu.com/p/d50cfcaf8102)
* [Java中的锁的使用和实现介绍](https://www.jianshu.com/p/f3bb5313fce4)
* [Java并发容器和框架](https://www.jianshu.com/p/50e732981232)
* [Java中的12个原子操作类介绍](https://www.jianshu.com/p/cb64f79058c2)
* [Java中的并发工具类](https://www.jianshu.com/p/79d9baaa396e)
* [Java中的线程池](https://www.jianshu.com/p/13c82f1a7ad9)
* [Executor框架](https://www.jianshu.com/p/8933aa93ee74)

---


#### 前言
>在Java中，使用线程来异步执行任务。
Java线程的创建与销毁需要一定的开销，如果我们为每一个任务创建一个新线程来执行，这些线程的创建与销毁将消耗大量的计算资源。
同时，为每一个任务创建一个新线程来执行，这种策略可能会使处于高负荷状态的应用最终崩溃。

Java的线程既是工作单元，也是执行机制。

从 **JDK 5** 开始，把 **工作单元** 与 **执行机制** 分离开来。
* **工作单元** : 
    * `Runnable`
    * `Callable`
* **执行机制**：
    * `Executor框架`

---

#### Executor框架简介

##### Executor框架的两级调度模型
在`HotSpot VM`的线程模型中，Java线程（`java.lang.Thread`）被 **一对一映射为本地操作系统线程**。Java线程启动时会创建一个本地操作系统线程；当该Java线程终止时，这个操作系统线程也会被回收。
操作系统会调度所有线程并将它们分配给可用的CPU。
在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器（`Executor框架`）将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到硬件处理器上。这种两级调度模型的示意图下面有介绍。
从下图中可以看出，应用程序通过`Executor框架`控制上层的调度；而下层的调度由操作系统内核控制，下层的调度不受应用程序的控制。

##### Executor框架的结构与成员
![Executor框架的两级调度模型](https://upload-images.jianshu.io/upload_images/1709375-110e230e9a24e9aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### Executor框架的结构
* **任务**。包括被执行任务需要实现的接口：`Runnable接口` 或 `Callable接口`。
* **任务的执行**。包括任务执行机制的核心接口`Executor`，以及继承自`Executor`的`ExecutorService`接口。`Executor`框架有两个关键类实现了`ExecutorService`接口（`ThreadPoolExecutor` 和 `ScheduledThreadPoolExecutor`）。
* **异步计算的结果**。包括接口`Future`和实现`Future`接口的`FutureTask`类。


 * `Executor`框架包含的主要的类与接口如下图所示：
    ![Executor框架的类与接口](https://upload-images.jianshu.io/upload_images/1709375-194345ebb166c4b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    下面是这些类和接口的简介：
    * `Executor`是一个接口，它是`Executor`框架的基础，它将任务的提交与任务的执行分离开来。
    * `ThreadPoolExecutor` 是线程池的核心实现类，用来执行被提交的任务。
    * `ScheduledThreadPoolExecutor` 是一个实现类，可以在给定的延迟后运行命令，或者定期执行命令。`ScheduledThreadPoolExecutor`比`Timer`更灵活，功能更强大。
    * `Future`接口和实现`Future`接口的`FutureTask`类，代表异步计算的结果。
    * `Runnable`接口和`Callable`接口的实现类，都可以被`ThreadPoolExecutor` 或`ScheduledThreadPoolExecutor`执行。

##### Executor`框架的使用示意图如下：
![Executor框架的使用示意图](https://upload-images.jianshu.io/upload_images/1709375-1bde020b9a162154.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主线程首先要创建实现`Runnable`或者`Callable`接口的任务对象。
工具类`Executors`可以通过以下两个方法把一个`Runnable`对象封装为一个`Callable`对象：
* `Executors.callable(Runnable task)` 
* `Executors.callable(Runnable task, Object resule)`。

然后可以把`Runnable`对象直接交给`ExecutorService`执行`ExecutorService.execute(Runnable command)`；或者也可以把`Runnable`对象或`Callable`对象提交给`ExecutorService` 执行  `ExecutorService.submit(Runnable task)` 或 `ExecutorService.submit(Callable<T>task)`。

如果执行`ExecutorService.submit(…)`，`ExecutorService` 将返回一个实现 `Future` 接口的对象（`到目前为止的JDK中，返回的是FutureTask对象`）。由于`FutureTask`实现了`Runnable`，程序员也可以创建`FutureTask`，然后直接交给`ExecutorService`执行。

最后，主线程可以执行 `FutureTask.get()` 方法来等待任务执行完成。主线程也可以执行
`FutureTask.cancel(boolean mayInterruptIfRunning)`来取消此任务的执行。

##### Executor框架的成员
这里将介绍`Executor`框架的主要成员：
* `ThreadPoolExecutor`
* `ScheduledThreadPoolExecutor`
* `Future`接口
* `Runnable`接口
* `Callable`接口
* `Executors`

#####  ThreadPoolExecutor
通常使用工厂类`Executors`来创建。`Executors`可以创建`3`种类型的`ThreadPoolExecutor`：
* `SingleThreadExecutor`：适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景。
下面是`Executors`提供的，创建使用单个线程的`SingleThreadExecuto`r的API。
    ```
    public static ExecutorService newSingleThreadExecutor() {
        return new Executors.FinalizableDelegatedExecutorService(
                new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue())
        );
    }
    public static ExecutorService newSingleThreadExecutor(ThreadFactory var0) {
        return new Executors.FinalizableDelegatedExecutorService(
                new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue(), var0)
        );
    }
    ```

* `FixedThreadPool`：适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。
下面是`Executors`提供的，创建使用固定线程数的`FixedThreadPool`的API：
    ```
    public static ExecutorService newFixedThreadPool(int var0) {
        return new ThreadPoolExecutor(var0, var0, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue());
    }
    public static ExecutorService newFixedThreadPool(int var0, ThreadFactory var1) {
        return new ThreadPoolExecutor(var0, var0, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue(), var1);
    }
    ```

* `CachedThreadPool`：是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器。
下面是`Executors`提供的，创建一个会根据需要创建新线程的`CachedThreadPool`的API。
    ```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, 2147483647, 60L, TimeUnit.SECONDS, new SynchronousQueue());
    }
    public static ExecutorService newCachedThreadPool(ThreadFactory var0) {
        return new ThreadPoolExecutor(0, 2147483647, 60L, TimeUnit.SECONDS, new SynchronousQueue(), var0);
    }
    ```

* `ScheduledThreadPoolExecutor`
`ScheduledThreadPoolExecutor`通常使用工厂类`Executors`来创建。
`Executors`可以创建`2`种类型的`ScheduledThreadPoolExecutor`，如下:
    * `ScheduledThreadPoolExecutor`。包含若干个线程的`ScheduledThreadPoolExecutor`。
    适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程的数量的应用场景。
    下面是工厂类`Executors`提供的，创建 **固定个数** 线程的`ScheduledThreadPoolExecutor`的API。
        ```
        public static ScheduledExecutorService newScheduledThreadPool(int var0) {
            return new ScheduledThreadPoolExecutor(var0);
        }
        public static ScheduledExecutorService newScheduledThreadPool(int var0, ThreadFactory var1) {
            return new ScheduledThreadPoolExecutor(var0, var1);
        }
        ```
    * `SingleThreadScheduledExecutor`。只包含一个线程的`ScheduledThreadPoolExecutor`。
        适用于需要单个后台线程执行周期任务，同时需要保证顺序地执行各个任务的应用场景。
        下面是`Executors`提供的，创建单个线程的`SingleThreadScheduledExecutor`的API。
        ```
        public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
            return new Executors.DelegatedScheduledExecutorService(new ScheduledThreadPoolExecutor(1));
        }
        public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory var0) {
            return new Executors.DelegatedScheduledExecutorService(new ScheduledThreadPoolExecutor(1, var0));
        }
        ```

##### Future接口
`Future`接口和实现`Future`接口的`FutureTask`类用来表示异步计算的结果。
当我们把`Runnable`接口或`Callable`接口的实现类提交（`submit`）给`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`时，`ThreadPoolExecutor`或 `ScheduledThreadPoolExecutor`会向我们返回一个`FutureTask`对象。
下面是对应的API：
```
<T> Future<T> submit(Callable<T> task)
<T> Future<T> submit(Runnable task, T result)
Future<> submit(Runnable task)
```

有一点需要读者注意，到目前最新的`JDK 8`为止，Java通过上述API返回的是一个`FutureTask`对象。但从API可以看到，Java仅仅保证返回的是一个实现了Future接口的对象。`在将来的JDK实现中，返回的可能不一定是FutureTask`。

##### Runnable接口和 Callable接口
`Runnable`接口和`Callable`接口的实现类，都可以被`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`执行。
它们之间的区别是`Runnable`不会返回结果，而`Callable`可以返回结果。
除了可以自己创建实现`Callable`接口的对象外，还可以使用工厂类`Executors`来把一个`Runnable`包装成一个`Callable`。

下面是`Executors`提供的，把一个`Runnable`包装成一个`Callable`的API。
 ```
public static Callable<Object> callable(Runnable task)  // 假设返回对象Callable1
```
下面是`Executors`提供的，把一个`Runnable`和一个`待返回的结果`包装成一个`Callable`的API。
```
public static <T> Callable<T> callable(Runnable task, T result) // 假设返回对象Callable2
```

前面讲过，当我们把一个`Callable`对象（比如上面的`Callable1`或`Callable2`）提交给`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`执行时，`submit(…)`会向我们返回一个`FutureTask`对象。我们可以执行`FutureTask.get()`方法来 **等待任务执行完成**。当任务成功完成后`FutureTask.get()`将返回该任务的结果。

例如，如果提交的是对象`Callable1`，`FutureTask.get()`方法将返回`null`；如果提交的是对象`Callable2`，`FutureTask.get()`方法将返回`result`对象。


#### ThreadPoolExecutor详解
`Executor`框架最核心的类是`ThreadPoolExecutor`，它是线程池的实现类，主要由下列4个组件构成。
* `corePool`：核心线程池的大小。
* `maximumPool`：最大线程池的大小。
* `BlockingQueue`：用来暂时保存任务的工作队列。
* `RejectedExecutionHandler`：当`ThreadPoolExecutor`已经关闭或`ThreadPoolExecutor`已经饱和时（达到了最大线程池大小且工作队列已满），`execute()`方法将要调用的`Handler`。

通过`Executor`框架的工具类`Executors`，可以创建3种类型的`ThreadPoolExecutor`。
* `FixedThreadPool`
* `SingleThreadExecutor`
* `CachedThreadPool`

##### FixedThreadPool详解
`FixedThreadPool`被称为可重用固定线程数的线程池。下面是`FixedThreadPool`的源代码实现。
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
```
`FixedThreadPool`的`corePoolSize`和`maximumPoolSize`都被设置为创建`FixedThreadPool`时指定的参数`nThreads`。

当线程池中的线程数大于`corePoolSize`时，`keepAliveTime`为多余的空闲线程等待新任务的最长时间，超过这个时间后多余的线程将被终止。这里把`keepAliveTime`设置为`0L`，意味着多余的空闲线程会被立即终止。

`FixedThreadPool`的`execute()`方法的运行示意图如下图所示。
![FixedThreadPool的execute()的运行示意图](https://upload-images.jianshu.io/upload_images/1709375-08b33a5234812c7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `图中1`：如果当前运行的线程数少于`corePoolSize`，则创建新线程来执行任务。
* `图中2`：在线程池完成预热之后（当前运行的线程数等于`corePoolSize`），将任务加入`LinkedBlockingQueue`。
* `图中3`：线程执行完`1`中的任务后，会在循环中反复从`LinkedBlockingQueue`获取任务来执行。

`FixedThreadPool`使用无界队列`LinkedBlockingQueue`作为线程池的工作队列（队列的容量为`Integer.MAX_VALUE`）。
使用无界队列作为工作队列会对线程池带来如下影响：
*  当线程池中的线程数达到`corePoolSize`后，新任务将在无界队列中等待，因此线程池中的线程数不会超过`corePoolSize`。
*  由于上一点，使用无界队列时`maximumPoolSize`将是一个无效参数。
* 由于前面两点，使用无界队列时`keepAliveTime`将是一个无效参数。
* 由于使用无界队列，运行中的`FixedThreadPool`（未执行方法`shutdown()`或`shutdownNow()`）不会拒绝任务（不会调用`RejectedExecutionHandler.rejectedExecution`方法）。

#### SingleThreadExecutor详解
`SingleThreadExecutor`是使用单个`worker`线程的`Executor`。
下面是`SingleThreadExecutor`的源代码实现：
```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService(
            new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>())
    );
}
```
`SingleThreadExecutor`的`corePoolSize`和`maximumPoolSize`被设置为1。其他参数与`FixedThreadPool`相同。`SingleThreadExecutor`使用无界队列`LinkedBlockingQueue`作为线程池的工作队列（队列的容量为`Integer.MAX_VALUE`）。`SingleThreadExecutor`使用无界队列作为工作队列对线程池带来的影响与`FixedThreadPool`相同，这里就不赘述了。

`SingleThreadExecutor`的运行示意图如下图所示：
![SingleThreadExecutor的execute()的运行示意图](https://upload-images.jianshu.io/upload_images/1709375-6f819c9c09de6cd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `上图1:`如果当前运行的线程数少于`corePoolSize`（即线程池中无运行的线程），则创建一个新线程来执行任务。
* `上图2:`在线程池完成预热之后（当前线程池中有一个运行的线程），将任务加入`LinkedBlockingQueue`。
* `上图3:`线程执行完`上图1`中的任务后，会在一个无限循环中反复从`LinkedBlockingQueue`获取任务来执行。


#### CachedThreadPool详解
`CachedThreadPool`是一个会根据需要创建新线程的线程池。
下面是创建`CachedThreadPool`的源代码。
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>());
}
```
`CachedThreadPool`的`corePoolSize`被设置为0，即`corePool`为空；`maximumPoolSize`被设置为`Integer.MAX_VALUE`，即`maximumPool`是无界的。这里把`keepAliveTime`设置为`60L`，意味着`CachedThreadPool`中的空闲线程等待新任务的最长时间为`60秒`，空闲线程超过`60秒`后将会被终止。

`FixedThreadPool`和`SingleThreadExecutor`使用无界队列`LinkedBlockingQueue`作为线程池的工作队列。
`CachedThreadPool`使用没有容量的`SynchronousQueue`作为线程池的工作队列，但`CachedThreadPool`的`maximumPool`是无界的。
这意味着，如果主线程提交任务的速度高于`maximumPool`中线程处理任务的速度时，`CachedThreadPool`会不断创建新线程。
极端情况下，`CachedThreadPool`会因为创建过多线程而耗尽CPU和内存资源。

`CachedThreadPool`的`execute()`方法的执行示意图如下图所示：
![CachedThreadPool的execute()的运行示意图](https://upload-images.jianshu.io/upload_images/1709375-d6565c728133ef71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `上图1：` 首先执行`SynchronousQueue.offer(Runnable task)`。如果当前`maximumPool`中有空闲线程正在执行`SynchronousQueue.poll(keepAliveTime，TimeUnit.NANOSECONDS)`，那么主线程执行`offer`操作与空闲线程执行的`poll`操作配对成功，主线程把任务交给空闲线程执行，`execute()`方法执行完成；否则执行下面的`步骤2`。
* `上图2：`当初始`maximumPool`为空，或者`maximumPool`中当前没有空闲线程时，将没有线程执行`SynchronousQueue.poll(keepAliveTime，TimeUnit.NANOSECONDS)`。这种情况下，`步骤1`将失败。此时`CachedThreadPool`会创建一个新线程执行任务，`execute()`方法执行完成。
* `上图3：`.在`步骤2`中新创建的线程将任务执行完后，会执行`SynchronousQueue.poll(keepAliveTime，TimeUnit.NANOSECONDS)`。这个`poll`操作会让空闲线程最多在`SynchronousQueue`中等待`60秒`钟。如果`60秒`钟内主线程提交了一个新任务（主线程执行`步骤1`），那么这个空闲线程将执行主线程提交的新任务；否则，这个空闲线程将终止。由于空闲`60秒`的空闲线程会被终止，因此长时间保持空闲的`CachedThreadPool`不会使用任何资源。

前面提到过，`SynchronousQueue`是一个没有容量的阻塞队列。每个插入操作必须等待另一个线程的对应移除操作，反之亦然。`CachedThreadPool`使用`SynchronousQueue`，把主线程提交的任务传递给空闲线程执行。
`CachedThreadPool`中任务传递的示意图如下图所示：
![CachedThreadPool的任务传递示意图](https://upload-images.jianshu.io/upload_images/1709375-745c812615e44165.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  ScheduledThreadPoolExecutor详解
`ScheduledThreadPoolExecutor`继承自`ThreadPoolExecutor`。它主要用来在给定的延迟之后运行任务，或者定期执行任务。`ScheduledThreadPoolExecutor`的功能与`Timer`类似，但`ScheduledThreadPoolExecutor`功能更强大、更灵活。`Timer`对应的是单个后台线程，而`ScheduledThreadPoolExecutor`可以在构造函数中指定多个对应的后台线程数。

* `ScheduledThreadPoolExecutor`的运行机制
  `ScheduledThreadPoolExecutor`的执行示意图（基于JDK 6）如下图所示：
  ![ScheduledThreadPoolExecutor的任务传递示意图](https://upload-images.jianshu.io/upload_images/1709375-777ce5d51df1a8fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


  `DelayQueue`是一个无界队列，所以`ThreadPoolExecutor`的`maximumPoolSize`在`ScheduledThreadPoolExecutor`中没有什么意义（设置`maximumPoolSize`的大小没有什么效果）。

  
`ScheduledThreadPoolExecutor`的执行主要分为两大部分。
* 当调用`ScheduledThreadPoolExecutor`的`scheduleAtFixedRate()`方法或者`scheduleWithFixedDelay()`方法时，会向ScheduledThreadPoolExecutor的`DelayQueue`添加一个实现了`RunnableScheduledFutur`接口的`ScheduledFutureTask`。
* 线程池中的线程从`DelayQueue`中获取`ScheduledFutureTask`，然后执行任务。

`ScheduledThreadPoolExecutor`为了实现周期性的执行任务，对`ThreadPoolExecutor`做了如下的修改。
  * 使用`DelayQueue`作为任务队列。
  * 获取任务的方式不同（后文会说明）。
  * 执行周期任务后，增加了额外的处理（后文会说明）。

#### ScheduledThreadPoolExecutor的实现
前面我们提到过，`ScheduledThreadPoolExecutor`会把待调度的任务（`ScheduledFutureTask`）放到一个`DelayQueue`中。

`ScheduledFutureTask`主要包含3个成员变量，如下。
* `long`型成员变量`time`，表示这个任务将要被执行的具体时间。
* `long`型成员变量`sequenceNumber`，表示这个任务被添加到`ScheduledThreadPoolExecutor`中的序号。
* `long`型成员变量`period`，表示任务执行的间隔周期。

`DelayQueue`封装了一个`PriorityQueue`，这个`PriorityQueue`会对队列中的`ScheduledFutureTask`进行排序。排序时，`time`小的排在前面（时间早的任务将被先执行）。如果两个`ScheduledFutureTask`的`time`相同，就比较`sequenceNumber`，`sequenceNumber`小的排在前面（也就是说，如果两个任务的执行时间相同，那么先提交的任务将被先执行）。

首先，让我们看看`ScheduledThreadPoolExecutor`中的线程执行周期任务的过程。下图是`ScheduledThreadPoolExecutor`中的线程`1`执行某个周期任务的4个步骤。
![ScheduledThreadPoolExecutor的任务执行步骤](https://upload-images.jianshu.io/upload_images/1709375-f18aa834f0e19dd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


下面是对这4个步骤的说明：
* `上图1：` 线程·1从`DelayQueue`中获取已到期的`ScheduledFutureTask` 的 `DelayQueue.take()`。到期任务是指`ScheduledFutureTask`的`time大于等于当前时间`。
* `上图2：` 线程1执行这个`ScheduledFutureTask`。
* `上图3：` 线程1修改`ScheduledFutureTask`的`time`变量为下次将要被执行的时间。
* `上图4：` 线程1把这个修改`time`之后的`ScheduledFutureTask`放回`DelayQueue`中`DelayQueue.add()`。

接下来，让我们看看上面的`步骤1`获取任务的过程。下面是`DelayQueue.take()`方法的源代码实现：
```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 1
    try {
        for (; ; ) {
            E first = q.peek();
            if (first == null) {
                available.await(); // 2.1
            } else {
                long delay = first.getDelay(TimeUnit.NANOSECONDS);
                if (delay > 0) {
                    long tl = available.awaitNanos(delay); // 2.2
                } else {
                    E x = q.poll(); // 2.3.1
                    assert x != null;
                    if (q.size() != 0)
                        available.signalAll(); // 2.3.2
                    return x;
                }
            }
        }
    } finally {
        lock.unlock(); // 3
    }
}
```
下图是`DelayQueue.take()`的执行示意图：
![ScheduledThreadPoolExecutor获取任务的过程](https://upload-images.jianshu.io/upload_images/1709375-6f62e538ebfece73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如图所示，获取任务分为3大步骤。
* 获取`Lock`。
* 获取周期任务。
    * 如果`PriorityQueue`为空，当前线程到`Condition`中等待；否则执行下面的`2.2`。
    * 如果`PriorityQueue`的头元素的time时间比当前时间大，到`Condition`中等待到`time`时间；否则执行下面的`2.3`。
  * 获取`PriorityQueue`的头元素（`2.3.1`）；如果`PriorityQueue`不为空，则唤醒在`Condition`中等待的所有线程（`2.3.2`）。
* 释放`Lock`。

`ScheduledThreadPoolExecutor`在一个循环中执行`步骤2`，直到线程从`PriorityQueue`获取到一个元素之后（执行`2.3.1`之后），才会退出无限循环（结束`步骤2`）。

最后，让我们看看`ScheduledThreadPoolExecutor`中的线程执行任务的步骤4，把`ScheduledFutureTask`放入`DelayQueue`中的过程。

下面是`DelayQueue.add()`的源代码实现：
```
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 1
    try {
        E first = q.peek();
        q.offer(e); // 2.1
        if (first == null || e.compareTo(first) < 0) {
            available.signalAll(); // 2.2
        }
        return true;
    } finally {
        lock.unlock(); // 3
    }
}
```

下图是`DelayQueue.add()`的执行示意图：
  ![ScheduledThreadPoolExecutor添加任务的过程](https://upload-images.jianshu.io/upload_images/1709375-b1c1426b9a6bfa7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如图所示，添加任务分为3大步骤。
* 获取`Lock`。
* 添加任务。
    * 向`PriorityQueue`添加任务。
    * 如果在上面`2.1`中添加的任务是`PriorityQueue`的头元素，唤醒在`Condition`中等待的所有线程。
* 释放`Lock`。

---

#### FutureTask详解
`Future`接口和实现`Future`接口的`FutureTask`类，代表 **异步计算的结果**。

##### FutureTask简介
`FutureTask`除了实现`Future`接口外，还实现了`Runnable`接口。因此，`FutureTask`可以交给`Executor`执行，也可以由调用线程直接执行`FutureTask.run()`。根据`FutureTask.run()`方法被执行的时机，`FutureTask`可以处于下面3种状态。
* **未启动**。`FutureTask.run()`方法还没有被执行之前，`FutureTask`处于未启动状态。当创建一个`FutureTask`，且没有执行`FutureTask.run()`方法之前，这个`FutureTask`处于未启动状态。
* **已启动**。`FutureTask.run()`方法被执行的过程中，`FutureTask`处于已启动状态。
* **已完成**。`FutureTask.run()`方法执行完后正常结束，或被取消`FutureTask.cancel(…)`，或执行`FutureTask.run()`方法时抛出异常而异常结束，`FutureTask`处于已完成状态。


下图是`FutureTask`的 **状态迁移** 的示意图。
![FutureTask的状态迁移示意图](https://upload-images.jianshu.io/upload_images/1709375-7ede8ad85801554a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当`FutureTask`处于未启动或已启动状态时，执行`FutureTask.get()`方法将导致调用线程阻塞；当`FutureTask`处于已完成状态时，执行`FutureTask.get()`方法将导致调用线程立即返回结果或抛出异常。

当`FutureTask`处于未启动状态时，执行`FutureTask.cancel()`方法将导致此任务永远不会被执行；当`FutureTask`处于已启动状态时，执行`FutureTask.cancel(true)`方法将以中断执行此任务线程的方式来试图停止任务；当`FutureTask`处于已启动状态时，执行`FutureTask.cancel(false)`方法将不会对正在执行此任务的线程产生影响（让正在执行的任务运行完成）；当`FutureTask`处于已完成状态时，执行`FutureTask.cancel(…)`方法将返回`false`。

下图是get方法和cancel方法的执行示意图：
![get方法和cancel方法的执行示意图](https://upload-images.jianshu.io/upload_images/1709375-89815492b8c01836.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### FutureTask的使用
可以把`FutureTask`交给`Executor`执行；也可以通过`ExecutorService.submit(...)`方法返回一个`FutureTask`，然后执行`FutureTask.get()`方法或`FutureTask.cancel(...)`方法。除此以外，还可以单独使用`FutureTask`。

当一个线程需要等待另一个线程把某个任务执行完后它才能继续执行，此时可以使用`FutureTask`。假设有多个线程执行若干任务，每个任务最多只能被执行一次。当多个线程试图同时执行同一个任务时，只允许一个线程执行任务，其他线程需要等待这个任务执行完后才能继续执行。

下面是对应的示例代码：
```
private final ConcurrentMap<Object, Future<String>> taskCache = new ConcurrentHashMap<>();

private String executionTask(final String taskName)
        throws ExecutionException, InterruptedException {
    while (true) {
        Future<String> future = taskCache.get(taskName); // 1.1, 2.1
        if (future == null) {
            Callable<String> task = new Callable<String>() {
                @Override
                public String call() throws InterruptedException {
                    return taskName;
                }
            };
            FutureTask<String> futureTask = new FutureTask<String>(task);
            future = taskCache.putIfAbsent(taskName, futureTask); // 1.3
            if (future == null) {
                future = futureTask;
                futureTask.run(); // 1.4执行任务
            }
        }
        try {
            return future.get(); // 1.5, 2.2
        } catch (CancellationException e) {
            taskCache.remove(taskName, future);
        }
    }
}
```
上述代码的执行示意图如下图所示：
![FutureTask的使用示例代码的执行示意图](https://upload-images.jianshu.io/upload_images/1709375-94db59d351411712.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当两个线程试图同时执行同一个任务时，如果`线程1`执行`1.3`后`线程2`执行`2.1`，那么接下来`线程2`将在`2.2`等待，直到`线程1`执行完`1.4`后`线程2`才能2.2（`FutureTask.get()`）返回。

#### FutureTask的实现
`FutureTask`的实现基于`AbstractQueuedSynchronizer`（以下简称为 **AQS**）。`java.util.concurrent中`的很多可阻塞类（比如`ReentrantLock`）都是基于`AQS`来实现的。`AQS`是一个同步框架，它提供通用机制来原子性管理同步状态、阻塞和唤醒线程，以及维护被阻塞线程的队列。
**JDK 6** 中`AQS`被广泛使用，基于`AQS`实现的同步器包括：
* `ReentrantLock`
* `Semaphore`
* `ReentrantReadWriteLock`
* `CountDownLatch`
* `FutureTask`。

每一个基于`AQS`实现的同步器都会包含两种类型的操作，如下:
* 至少一个`acquire`操作。这个操作阻塞调用线程，除非/直到AQS的状态允许这个线程继续执行。`FutureTask`的`acquire`操作为`get() / get(long timeout，TimeUnit unit)`方法调用。
* 至少一个`release`操作。这个操作改变`AQS`的状态，改变后的状态可允许一个或多个阻塞线程被解除阻塞。`FutureTask`的`release`操作包括`run()`方法和`cancel(...)`方法。

基于“ **复合优先于继承** ”的原则，`FutureTask`声明了一个内部私有的继承于`AQS`的子类`Sync`，对`FutureTask`所有公有方法的调用都会委托给这个内部子类。

`AQS`被作为“模板方法模式”的基础类提供给`FutureTask`的内部子类`Sync`，这个内部子类只需要实现状态检查和状态更新的方法即可，这些方法将控制`FutureTask`的获取和释放操作。具体来说，`Sync`实现了`AQS`的`tryAcquireShared(int)`方法和`tryReleaseShared(int)`方法，`Sync`通过这两个方法来检查和更新同步状态。


`FutureTask`的设计示意图如下图所示：
![FutureTask的设计示意图](https://upload-images.jianshu.io/upload_images/1709375-b475f5b36aaf4160.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如图所示，`Sync`是`FutureTask`的内部私有类，它继承自`AQS`。创建`FutureTask`时会创建内部私有的成员对象`Sync`，`FutureTask`所有的的公有方法都直接委托给了内部私有的`Sync`。

`FutureTask.get()`方法会调用`AQS.acquireSharedInterruptibly(int arg)`方法，这个方法的执行过程如下：
* 调用`AQS.acquireSharedInterruptibly(int arg)`方法，这个方法首先会回调在子类`Sync`中实现的`tryAcquireShared()`方法来判断`acquire`操作是否可以成功。`acquire`操作可以成功的条件为：`state`为执行`完成状态RAN`或`已取消状态CANCELLED`，且`runner`不为`null`。
* 如果成功则`get()`方法立即返回。如果失败则到线程等待队列中去等待其他线程执行
`release`操作。
* 当其他线程执行`release`操作（比如`FutureTask.run()`或`FutureTask.cancel(…)`）唤醒当前线程后，当前线程再次执行`tryAcquireShared()`将返回正值1，当前线程将离开线程等待队列并唤醒它的后继线程（这里会产生级联唤醒的效果，后面会介绍）。
* 最后返回计算的结果或抛出异常。

`FutureTask.run()`的执行过程如下:
* 执行在构造函数中指定的任务（`Callable.call()`）。
* 以原子方式来更新同步状态（调用`AQS.compareAndSetState(int expect，int update)`，设置`state`为执行`完成状态RAN`）。如果这个原子操作成功，就设置代表计算结果的变量`result`的值为`Callable.call()`的返回值，然后调用`AQS.releaseShared(int arg)`。
* `AQS.releaseShared(int arg)`首先会回调在子类`Sync`中实现的`tryReleaseShared(arg)`来执行`release`操作（设置运行任务的线程`runner`为`null`，然会返回`true`）；`AQS.releaseShared(int arg)`，然后唤醒线程等待队列中的第一个线程。
* 调用`FutureTask.done()`。

当执行`FutureTask.get()`方法时，如果`FutureTask`不是处于执行`完成状态RAN`或`已取消状态CANCELLED`，当前执行线程将到`AQS`的线程等待队列中等待（见下图的线程A、B、C和D）。当某个线程执行`FutureTask.run()`方法或`FutureTask.cancel(...)`方法时，会唤醒线程等待队列的第一个线程（见下图所示的线程E唤醒线程A）。
![FutureTask的级联唤醒示意图](https://upload-images.jianshu.io/upload_images/1709375-99b164b9ebf8fd3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

假设开始时`FutureTask`处于未启动状态或已启动状态，等待队列中已经有3个线程（`A`、`B`和`C`）在等待。此时，线程`D`执行`get()`方法将导致线程`D`也到等待队列中去等待。

当线程`E`执行`run()`方法时，会唤醒队列中的第一个线程`A`。线程`A`被唤醒后，首先把自己从队列中删除，然后唤醒它的后继线程`B`，最后线程`A`从`get()`方法返回。线程`B`、`C`和`D`重复`A`线程的处理流程。最终，在队列中等待的所有线程都被级联唤醒并从`get()`方法返回。


#### 小结
本文介绍了`Executor`框架的整体结构和成员组件。
希望读者阅读本章之后，能够对`Executor`框架有一个比较深入的理解，同时也希望本章内容有助于读者更熟练地使用`Executor`框架。


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
