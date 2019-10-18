# Java中的线程池 

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
>Java中的线程池是运用场景最多的并发框架，几乎所有需要异步或并发执行任务的程序都可以使用线程池。

在开发过程中，合理地使用线程池能够带来3个好处。
* **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
* **提高响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行。
* **提高线程的可管理性**。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用线程池，必须对其实现原理了如指掌。

---

#### 线程池的实现原理
当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？
本文来看一下线程池的主要处理流程，处理流程图下图所示。
![线程池的主要处理流程](https://upload-images.jianshu.io/upload_images/1709375-3c3aa9672167d9d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。
* **线程池判断核心线程池里的线程是否都在执行任务**。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
* **线程池判断工作队列是否已经满**。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
* **线程池判断线程池的线程是否都处于工作状态**。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。

`ThreadPoolExecutor` 执行 `execute()` 方法的示意图如下：
![ThreadPoolExecutor执行示意图](https://upload-images.jianshu.io/upload_images/1709375-09b6ed198faa6c4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`ThreadPoolExecutor`执行`execute`方法分下面4种情况：
* 如果当前运行的线程少于`corePoolSize`，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）。`上图1`
* 如果运行的线程等于或多于`corePoolSize`，则将任务加入`BlockingQueue`。`上图2`
* 如果无法将任务加入`BlockingQueue`（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）。`上图3`
* 如果创建新线程将使当前运行的线程超出`maximumPoolSize`，任务将被拒绝，并调用`RejectedExecutionHandler.rejectedExecution()`方法。`上图4`

`ThreadPoolExecutor`采取上述步骤的总体设计思路，是为了在执行`execute()`方法时，尽可能地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。在`ThreadPoolExecutor`完成预热之后（`当前运行的线程数大于等于corePoolSize`），几乎所有的`execute()`方法调用都是执行 `上图2` ，而 `上图2` 不需要获取全局锁。

* **源码分析**：上面的流程分析让我们很直观地了解了线程池的工作原理，让我们再通过源代码来看看是如何实现的，线程池执行任务的方法如下：
    ```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            // 如果线程数小于基本线程数，则创建线程并执行当前任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 如线程数大于等于基本线程数或线程创建失败，则将当前任务放到工作队列中。
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (!isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        } else if (!addWorker(command, false))
            // 如果线程池不处于运行中或任务无法放入队列，
            //并且当前线程数量小于最大允许的线程数量,则创建一个线程执行任务.

            // 抛出RejectedExecutionException异常
            reject(command);
    }
    ```

* **工作线程**：线程池创建线程时，会将线程封装成工作线程`Worker`，`Worker`在执行完任务后，还会循环获取工作队列里的任务来执行。我们可以从`Worker`类的`run()`方法里看到这点。
    ```
    public void run() {
        try {
            Runnable task = firstTask;
            firstTask = null;
            while (task != null || (task = getTask()) != null) {
                runTask(task);
                task = null;
            }
        } finally {
            workerDone(this);
        }
    }
    ```

`ThreadPoolExecutor`中线程执行任务的示意图如下：
![ThreadPoolExecutor中线程执行任务的示意图](https://upload-images.jianshu.io/upload_images/1709375-610083ab0b3732fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
线程池中的线程执行任务分两种情况：
* 在`execute()`方法中创建一个线程时，会让这个线程执行当前任务。
* 这个线程执行完`上图中1`的任务后，会反复从`BlockingQueue`获取任务来执行。

---

#### 线程池的使用

##### 线程池的创建
我们可以通过`ThreadPoolExecutor`来创建一个线程池。
```
ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime,
                unit, workQueue, threadFactory, handler);
```
创建一个线程池时需要输入几个参数：
* `corePoolSize`（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。
    如果调用了线程池的`prestartAllCoreThreads()`方法，线程池会提前创建并启动所有基本线程。
* `maximumPoolSize`（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果。
* `keepAliveTime`（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。
* `TimeUnit`（线程活动保持时间的单位），可选的单位有：
    * 天（`DAYS`）
    * 小时（`HOURS`）
    * 分钟（`MINUTES`）
    * 毫秒（`MILLISECONDS`）
    * 微秒（`MICROSECONDS`，千分之一毫秒）
    * 纳秒（`NANOSECONDS`，千分之一微秒）
* `workQueue`（任务队列）：用于保存等待执行的任务的阻塞队列。
    可以选择以下几个阻塞队列：
    * `ArrayBlockingQueue`：是一个基于数组结构的有界阻塞队列，此队列按`FIFO`（先进先出）原则对元素进行排序。
    * `LinkedBlockingQueue`：一个基于链表结构的阻塞队列，此队列按`FIFO`排序元素，吞吐量通常要高于`ArrayBlockingQueue`。静态工厂方法`Executors.newFixedThreadPool()`使用了这个队列。
    * `SynchronousQueue`：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于`LinkedBlockingQueue`，静态工厂方法`Executors.newCachedThreadPool`使用了这个队列。
    * `PriorityBlockingQueue`：一个具有优先级的无限阻塞队列。

* `ThreadFactory`：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架`guava`提供的`ThreadFactoryBuilder`可以快速给线程池里的线程设置有意义的名字，代码如下：
    ```
    new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
    ```
* `RejectedExecutionHandler`（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是`AbortPolicy`，表示无法处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。
    * `AbortPolicy`：直接抛出异常。
    * `CallerRunsPolicy`：只用调用者所在线程来运行任务。
    * `DiscardOldestPolicy`：丢弃队列里最近的一个任务，并执行当前任务。
    * `DiscardPolicy`：不处理，丢弃掉。

  当然，也可以根据应用场景需要来实现`RejectedExecutionHandler`接口自定义策略。如记录日志或持久化存储不能处理的任务。

---

##### 向线程池提交任务
可以使用两个方法向线程池提交任务，分别为`execute()`和`submit()`方法。
 * `execute()`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。通过以下代码可知`execute()`方法输入的任务是一个`Runnable`类的实例。
    ```
    threadsPool.execute(new Runnable() {
        @Override
        public void run() {
            // TODO Auto-generated method stub
        }
    });
    ```
* `submit()`方法用于提交需要返回值的任务。线程池会返回一个`future`类型的对象，通过这个`future`对象可以判断任务是否执行成功，并且可以通过`future`的`get()`方法来获取返回值，`get()`方法会阻塞当前线程直到任务完成，而使用`get(long timeout，TimeUnit unit)`方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。
    ```
    Future<Object> future = executor.submit(harReturnValuetask);
    try {
        Object s = future.get();
    } catch (InterruptedException e) {
        // 处理中断异常
    } catch (ExecutionException e) {
        // 处理无法执行任务异常
    } finally {
        // 关闭线程池
        executor.shutdown();
    }
    ```

---

##### 关闭线程池
可以通过调用线程池的`shutdown`或`shutdownNow`方法来关闭线程池。
它们的原理是遍历线程池中的工作线程，然后逐个调用线程的`interrupt`方法来中断线程，所以无法响应中断的任务可能永远无法终止。
但是它们存在一定的区别：
* `shutdownNow`首先将线程池的状态设置成`STOP`，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，
* `shutdown`只是将线程池的状态设置成`SHUTDOWN`状态，然后中断所有没有正在执行任务的线程。

只要调用了这两个关闭方法中的任意一个，`isShutdown`方法就会返回`true`。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用`isTerminaed`方法会返回`true`。
至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定：
* 通常调用`shutdown`方法来关闭线程池。
* 如果任务不一定要执行完，则可以调用`shutdownNow`方法。

---

##### 合理地配置线程池
要想合理地配置线程池，就必须首先分析任务特性，可以从以下几个角度来分析：
* **任务的性质**：CPU密集型任务、IO密集型任务和混合型任务。
* **任务的优先级**：高、中和低。
* **任务的执行时间**：长、中和短。
* **任务的依赖性**：是否依赖其他系统资源，如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理。
* `CPU密集型任务`应配置尽可能小的线程，如配置`Ncpu+1`个线程的线程池。
* `IO密集型任务`线程并不是一直在执行任务，则应配置尽可能多的线程，如`2*Ncpu`。
* `混合型的任务`，如果可以拆分，将其拆分成一个`CPU密集型任务`和一个`IO密集型任务`：
    * 两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。
    * 两个任务执行时间相差太大，则没必要进行分解。
    可以通过`Runtime.getRuntime().availableProcessors()`方法获得当前设备的`CPU`个数。

优先级不同的任务可以使用优先级队列`PriorityBlockingQueue`来处理。它可以让优先级高的任务先执行。

> 注意：如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。


执行时间不同的任务可以交给不同规模的线程池来处理，或者可以使用优先级队列，让
执行时间短的任务先执行。

**依赖数据库连接池的任务**，因为线程提交SQL后 **需要等待数据库返回结果**，等待的时间越长，则CPU空闲时间就越长，那么线程数应该设置得越大，这样才能更好地利用CPU。

**建议使用有界队列**。有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。
>有一次，我们系统里后台任务线程池的队列和线程池全满了，不断抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞，任务积压在线程池里。如果当时我们设置成无界队列，那么线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然，我们的系统所有的任务是用单独的服务器部署的，我们使用不同规模的线程池完成不同类型的任务，但是出现这样问题时也会影响到其他任务。

---

##### 线程池的监控
如果在系统中 **大量使用线程池**，则有必要 **对线程池进行监控**，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性：
* `taskCount`：线程池需要执行的任务数量。
* `completedTaskCount`：线程池在运行过程中已完成的任务数量，小于或等于`taskCount`。
* `largestPoolSize`：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，则表示线程池曾经满过。
* `getPoolSize`：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。
* `getActiveCount`：获取活动的线程数。

**通过扩展线程池进行监控**。
可以通过继承线程池来自定义线程池，重写线程池的`beforeExecute`、`afterExecute`和`terminated`方法，也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。
例如，监控任务的`平均执行时间`、`最大执行时间` 和 `最小执行时间` 等。这几个方法在线程池里是空方法。
```
protected void beforeExecute(Thread t, Runnable r) { }
```

----

#### 小结
本文我们介绍了：
* **线程池的原理**
* **线程池的创建** 
  `ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime,unit, workQueue, threadFactory, handler);`
* **向线程池提交任务**  `execute()`和`submit()`
* **关闭线程池** `shutdown`或`shutdownNow`
* **线程池的合里配置**
* **线程池的监控**


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
