# RxJava之Schedulers源码介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`Base on RxJava 2.X`
#### 简介
RxJava 的 `Schedulers` 提供了以下五种 `Scheduler`（调度器）：
* [`SINGLE`]() 
* [`COMPUTATION`](#)
* [`IO`](#) 
* [`NEW_THREAD`](#)
* [`TRAMPOLINE`](#)
```
static {
    SINGLE = RxJavaPlugins.initSingleScheduler(new SingleTask());
    COMPUTATION = RxJavaPlugins.initComputationScheduler(new ComputationTask());
    IO = RxJavaPlugins.initIoScheduler(new IOTask());
    NEW_THREAD = RxJavaPlugins.initNewThreadScheduler(new NewThreadTask());
    TRAMPOLINE = TrampolineScheduler.instance();
}
```

---

#### 以 `Schedulers.single()` 为例介绍
如果我们没有调用 `setInitXXSchedulerHandler` 或者 `setXXSchedulerHandler` 自己实现调度器的话(`XX` 代表上面除了 `TRAMPOLINE` 的四种调度器的名字)，我们开发中用到的 `Schedulers.io();` `Schedulers.computation();` `Schedulers.newThread();` `Schedulers.single();` 实际上就是对应的 `XXTask` 的 `call()`方法返回的 `Scheduler` 对象，即对应的 `XXScheduler` 对象。
```
public static void setInitSingleSchedulerHandler(@Nullable Function<? super Callable<Scheduler>, ? extends Scheduler> handler) {
    if (lockdown) {
        throw new IllegalStateException("Plugins can't be changed anymore");
    }
    onInitSingleHandler = handler;
}

public static void setSingleSchedulerHandler(@Nullable Function<? super Scheduler, ? extends Scheduler> handler) {
    if (lockdown) {
        throw new IllegalStateException("Plugins can't be changed anymore");
    }
    onSingleHandler = handler;
}
```

**以下是 `Schedulers.single()`  的源码介绍：**
 * `onSingleScheduler()` 中的   `onSingleHandler` 是通过 `setSingleSchedulerHandler()` 设置的，默认为 `Null` ，所以即返回 `SINGLE`。 
    ```
    public static Scheduler single() {
        return RxJavaPlugins.onSingleScheduler(SINGLE);
    }
  
    public static Scheduler onSingleScheduler(@NonNull Scheduler defaultScheduler) {
      Function<? super Scheduler, ? extends Scheduler> f = onSingleHandler;
        if (f == null) {
            return defaultScheduler;
        }
        return apply(f, defaultScheduler);
    }
    ```
* `SINGLE` 是静态常量，通过 `RxJavaPlugins.initSingleScheduler(new SingleTask());` 初始化。
    ```
    static final Scheduler SINGLE;

    static {
        SINGLE = RxJavaPlugins.initSingleScheduler(new SingleTask());
        ...
    }
    ```
* `initSingleScheduler()` 中的 `onInitSingleHandler` 是通过 `setInitSingleSchedulerHandler()` 设置的，默认为 `Null` ，所以即调用 `callRequireNonNull(new SingleTask())`。 
    ```
    public static Scheduler initSingleScheduler(@NonNull Callable<Scheduler> defaultScheduler) {
        ObjectHelper.requireNonNull(defaultScheduler, "Scheduler Callable can't be null");
        Function<? super Callable<Scheduler>, ? extends Scheduler> f = onInitSingleHandler;
        if (f == null) {
            return callRequireNonNull(defaultScheduler);
        }
        return applyRequireNonNull(f, defaultScheduler);
    }
    ```
* `callRequireNonNull(new SingleTask())` 即返回 `SingleTask`对象的 `call` 方法。
    ```
    static Scheduler callRequireNonNull(@NonNull Callable<Scheduler> s) {
        try {
            return ObjectHelper.requireNonNull(s.call(), "Scheduler Callable result can't be null");
        } catch (Throwable ex) {
            throw ExceptionHelper.wrapOrThrow(ex);
        }
    }

    static final class SingleTask implements Callable<Scheduler> {
        @Override
        public Scheduler call() throws Exception {
            return SingleHolder.DEFAULT;
        }
    }
    ```
* 即创建了一个 `SingleScheduler` 对象。
    ```
    static final class SingleHolder {
        static final Scheduler DEFAULT = new SingleScheduler();
    }
    ```
**所以 `Schedulers.single()`  实际返回的是 `SingleScheduler` 对象**.
同样的：
**`Schedulers.io();` 实际返回的是 `IoScheduler` 对象**
**`Schedulers.computation();` 实际返回的是 `ComputationScheduler` 对象**
**`Schedulers.newThread();` 实际返回的是 `NewThreadScheduler` 对象**

---

####  `SingleScheduler` 源码介绍
```
public final class SingleScheduler extends Scheduler {
    final ThreadFactory threadFactory;
    final AtomicReference<ScheduledExecutorService> executor = new AtomicReference<ScheduledExecutorService>();

    private static final String KEY_SINGLE_PRIORITY = "rx2.single-priority";
    private static final String THREAD_NAME_PREFIX = "RxSingleScheduler";

    static final RxThreadFactory SINGLE_THREAD_FACTORY;

    static final ScheduledExecutorService SHUTDOWN;
    static {
        SHUTDOWN = Executors.newScheduledThreadPool(0);
        SHUTDOWN.shutdown();

        int priority = Math.max(Thread.MIN_PRIORITY, Math.min(Thread.MAX_PRIORITY,
                Integer.getInteger(KEY_SINGLE_PRIORITY, Thread.NORM_PRIORITY)));

        SINGLE_THREAD_FACTORY = new RxThreadFactory(THREAD_NAME_PREFIX, priority, true);//1.1
    }

    public SingleScheduler() { //1.0
        this(SINGLE_THREAD_FACTORY);
    }

    public SingleScheduler(ThreadFactory threadFactory) {//2.0
        this.threadFactory = threadFactory;
        executor.lazySet(createExecutor(threadFactory));//4.0
    }

    static ScheduledExecutorService createExecutor(ThreadFactory threadFactory) {//3.0
        return SchedulerPoolFactory.create(threadFactory);
    }
    ...
}
```
* `(1.0)`默认的构造方法，传入了一个 `SINGLE_THREAD_FACTORY`的静态常量。`(1.1)`我们可以看到它是在初始化为 `new RxThreadFactory("RxSingleScheduler",  5 , true);` 即为 **线程名称前缀** 为 `RxSingleScheduler`，**优先级为5 不阻塞** 的 `RxThreadFactory` 对象。
* `(2.0)`然后设置当前的  `threadFactory` 为此 `RxThreadFactory` 对象。
* `(3.0)`然后通过`SchedulerPoolFactory.create(threadFactory)`创建了一个执行者。
  `(3.1)`即通过 `Executors.newScheduledThreadPool(1, factory)`创建了一个核心线程数为 `1` 的 `ScheduledExecutorService`(调度线程池)。
  `(3.2)`并将`ScheduledExecutorService` 放进 `SchedulerPoolFactory`的  `key` 为 `ScheduledThreadPoolExecutor`的 `Map` 集合 `POOLS`中。
    ```
    // Upcast to the Map interface here to avoid 8.x compatibility issues.
    // See http://stackoverflow.com/a/32955708/61158
    //这个用map接口是为了解决java8的一个bug，具体可以点击上面的链接查看
    static final Map<ScheduledThreadPoolExecutor, Object> POOLS =
            new ConcurrentHashMap<ScheduledThreadPoolExecutor, Object>();

    public static ScheduledExecutorService create(ThreadFactory factory) {
        final ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, factory);//3.1
        tryPutIntoPool(PURGE_ENABLED, exec);
        return exec;
    }

    static void tryPutIntoPool(boolean purgeEnabled, ScheduledExecutorService exec) {
        if (purgeEnabled && exec instanceof ScheduledThreadPoolExecutor) {
            ScheduledThreadPoolExecutor e = (ScheduledThreadPoolExecutor) exec;
            POOLS.put(e, exec);//3.2
        }
    }
    ```
* `(4.0)`然后 将  `AtomicReference<ScheduledExecutorService>` 对象 `executor` 的 `value` 设置为上面创建的 `ScheduledExecutorService`。

我们之前在 [Rxjava之timer和interval操作符源码解析](https://www.jianshu.com/p/316966e3841a) 介绍过 `timer`操作符在订阅的时候会执行`ObservableTimer`的 `subscribeActual` 方法，
```
public void subscribeActual(Observer<? super Long> observer) {
    TimerObserver ios = new TimerObserver(observer);
    observer.onSubscribe(ios);
    Disposable d = scheduler.scheduleDirect(ios, delay, unit);
    ios.setResource(d);
}
```
其中的 `scheduler.scheduleDirect(ios, delay, unit)`中 会通过`createWorker()`创建一个 `Worker`。
```
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker(); //
    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
    DisposeTask task = new DisposeTask(decoratedRun, w);
    w.schedule(task, delay, unit);
    return task;
}
```
* 我们来看看 `SingleScheduler`的 `createWorker()`：
  ```
  public Worker createWorker() {
      return new ScheduledWorker(executor.get()); 
  }

  static final class ScheduledWorker extends Scheduler.Worker {
      final ScheduledExecutorService executor;
      final CompositeDisposable tasks;
      ScheduledWorker(ScheduledExecutorService executor) {
          this.executor = executor;
          this.tasks = new CompositeDisposable();
      }
      ...
  }
  ```
  * 通过`executor.get()`获取 `AtomicReference` 的`value`值，通过上面的`SingleScheduler` 源码`(4.0)`的介绍，即获取到的是核心线程数为`1`的 `ScheduledExecutorService`。
  * 然后将其赋值给 `ScheduledWorker`的 `executor`。

* 然后我们看看`w.schedule(task, delay, unit)`：
    ```
    public Disposable schedule(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        if (disposed) {
            return EmptyDisposable.INSTANCE;
        }
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, tasks);
        tasks.add(sr);
        try {
            Future<?> f;
            if (delay <= 0L) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delay, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            dispose();
            RxJavaPlugins.onError(ex);
            return EmptyDisposable.INSTANCE;
        }
        return sr;
    }
    ```
    * 首先校验 `disposed` 的状态，`true`就直接返回`EmptyDisposable.INSTANCE`。[Rxjava之timer和interval操作符源码解析](https://www.jianshu.com/p/316966e3841a) 中介绍的`interval`操作符里`schedulePeriodicallyDirect`中会校验这个返回值。
    * 然后构建了也给`ScheduledRunnable`对象(继承自`AtomicReferenceArray`)。
      将传递进来的`Runnable`对象赋值给`actual`。
      将 `tasks`赋值给`AtomicReferenceArray`的长度为`3`的`array`的第一个索引位置。
      ```
      public ScheduledRunnable(Runnable actual, DisposableContainer parent) {
          super(3);
          this.actual = actual;
          this.lazySet(0, parent);
      }
      ```
    * `tasks.add(sr)`即把`ScheduledRunnable`添加到`OpenHashSet<Disposable>`的`resources`集合中，在调用`dispose()`的时候去清空这个集合。
        ```
        public void dispose() {
            if (!disposed) {
                disposed = true;
                tasks.dispose();
            }
        }
        ```
    * 然后把这个任务丢给线程池去执行：以`timer`操作符为例，线程池执行任务即为执行 `ObservableTimer`中`TimerObserver` 的 `run` 方法。
        ```
        //ObservableTimer
        public void subscribeActual(Observer<? super Long> observer) {
            TimerObserver ios = new TimerObserver(observer);
            observer.onSubscribe(ios);
            Disposable d = scheduler.scheduleDirect(ios, delay, unit);
            ios.setResource(d);
        }
        //ScheduledRunnable
        public void run() {
            lazySet(THREAD_INDEX, Thread.currentThread());//1.0
            try {
                try {
                    actual.run();
                } catch (Throwable e) {
                    // Exceptions.throwIfFatal(e); nowhere to go
                    RxJavaPlugins.onError(e);
                }
            } finally {
                lazySet(THREAD_INDEX, null);//1.1
                Object o = get(PARENT_INDEX);//2.0
                if (o != PARENT_DISPOSED && compareAndSet(PARENT_INDEX, o, DONE) && o != null) {
                    ((DisposableContainer)o).delete(this);//2.1
                }
                for (;;) {
                    o = get(FUTURE_INDEX);//3.0
                    if (o == SYNC_DISPOSED || o == ASYNC_DISPOSED || compareAndSet(FUTURE_INDEX, o, DONE)) {
                        break;
                    }
                }
            }
        }
        ```
        * `(1.0)`保存当前执行任务的线程，`(1.1)`置空当前执行任务的线程。
        * `(2.0)` 获取上面设置的`CompositeDisposable`对象。`(2.2)` 去删除`OpenHashSet<Disposable> resources`中执行完成的任务。
        * `(3.0)`直到任务执行完成或者被取消才结束。
     * 返回的 `Future`对象，被赋值给 `ScheduledRunnable`中 `array`的第二个位置。
        ```
        static final int PARENT_INDEX = 0;
        static final int FUTURE_INDEX = 1;
        static final int THREAD_INDEX = 2;
        public void setFuture(Future<?> f) {
            for (;;) {
                Object o = get(FUTURE_INDEX);
                if (o == DONE) {
                    return;
                }
                if (o == SYNC_DISPOSED) {
                    f.cancel(false);
                    return;
                }
                if (o == ASYNC_DISPOSED) {
                    f.cancel(true);
                    return;
                }
                if (compareAndSet(FUTURE_INDEX, o, f)) {
                    return;
                }
            }
        }
        ```

---

#### NewThreadScheduler 源码介绍
和`SingleScheduler`类似`NewThreadScheduler`也是构建了一个核心线程数为`1`的`ScheduledExecutorService`。
区别就是 `NewThreadScheduler`不需要去记录之前运行的任务，每个任务之前不会有什么关联。
以下代码是 `NewThreadWorker`的 `scheduleDirect`方法：
```
public Disposable scheduleDirect(final Runnable run, long delayTime, TimeUnit unit) {
    ScheduledDirectTask task = new ScheduledDirectTask(RxJavaPlugins.onSchedule(run));
    try {
        Future<?> f;
        if (delayTime <= 0L) {
            f = executor.submit(task);
        } else {
            f = executor.schedule(task, delayTime, unit);
        }
        task.setFuture(f);
        return task;
    } catch (RejectedExecutionException ex) {
        RxJavaPlugins.onError(ex);
        return EmptyDisposable.INSTANCE;
    }
}
```
`ScheduledDirectTask `：执行任务的返回值为 `null`。
```
public final class ScheduledDirectTask extends AbstractDirectTask implements Callable<Void> {
    private static final long serialVersionUID = 1811839108042568751L;
    public ScheduledDirectTask(Runnable runnable) {
        super(runnable);
    }
    @Override
    public Void call() throws Exception {
        runner = Thread.currentThread();
        try {
            runnable.run();
        } finally {
            lazySet(FINISHED);
            runner = null;
        }
        return null;
    }
}
```
---

#### ComputationScheduler 源码介绍
`ComputationScheduler` 在[Rxjava之timer和interval操作符源码解析](https://www.jianshu.com/p/316966e3841a) 中已经介绍过，就不再赘述了。

---

#### IoScheduler 源码介绍
* 首先我们看看构造函数做了些什么：
    ```
    private static final TimeUnit KEEP_ALIVE_UNIT = TimeUnit.SECONDS;
    static {
        KEEP_ALIVE_TIME = Long.getLong(KEY_KEEP_ALIVE_TIME, KEEP_ALIVE_TIME_DEFAULT);
        SHUTDOWN_THREAD_WORKER = new ThreadWorker(new RxThreadFactory("RxCachedThreadSchedulerShutdown"));
        SHUTDOWN_THREAD_WORKER.dispose();
        int priority = Math.max(Thread.MIN_PRIORITY, Math.min(Thread.MAX_PRIORITY,
            Integer.getInteger(KEY_IO_PRIORITY, Thread.NORM_PRIORITY)));
        WORKER_THREAD_FACTORY = new RxThreadFactory(WORKER_THREAD_NAME_PREFIX, priority);
        EVICTOR_THREAD_FACTORY = new RxThreadFactory(EVICTOR_THREAD_NAME_PREFIX, priority);
        NONE = new CachedWorkerPool(0, null, WORKER_THREAD_FACTORY);//1.0
        NONE.shutdown();
    }

    static final class CachedWorkerPool implements Runnable {
        private final long keepAliveTime;
        private final ConcurrentLinkedQueue<ThreadWorker> expiringWorkerQueue;
        final CompositeDisposable allWorkers;
        private final ScheduledExecutorService evictorService;
        private final Future<?> evictorTask;
        private final ThreadFactory threadFactory;

        CachedWorkerPool(long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory) {
            this.keepAliveTime = unit != null ? unit.toNanos(keepAliveTime) : 0L;
            this.expiringWorkerQueue = new ConcurrentLinkedQueue<ThreadWorker>();
            this.allWorkers = new CompositeDisposable();
            this.threadFactory = threadFactory;

            ScheduledExecutorService evictor = null;
            Future<?> task = null;
            if (unit != null) {
                evictor = Executors.newScheduledThreadPool(1, EVICTOR_THREAD_FACTORY);
                task = evictor.scheduleWithFixedDelay(this, this.keepAliveTime, this.keepAliveTime, TimeUnit.NANOSECONDS);
            }
            evictorService = evictor;
            evictorTask = task;
        }
        ...
        void shutdown() {
            allWorkers.dispose();
            if (evictorTask != null) {
                evictorTask.cancel(true);
            }
            if (evictorService != null) {
                evictorService.shutdownNow();
            }
        }
    }

    public IoScheduler() {
        this(WORKER_THREAD_FACTORY);
    }

    public IoScheduler(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
        this.pool = new AtomicReference<CachedWorkerPool>(NONE);
        start();
    }

    @Override
    public void start() {
        CachedWorkerPool update = new CachedWorkerPool(KEEP_ALIVE_TIME, KEEP_ALIVE_UNIT, threadFactory);
        if (!pool.compareAndSet(NONE, update)) {
            update.shutdown();
        }
    }
    ```
    * `(1.0)`首先构造了一个`CachedWorkerPool`。
    * `(2.0)`将构造的`CachedWorkerPool`设置为`AtomicReference`的`value`的值。
    * `(3.0)`构造了一个`CachedWorkerPool(60, TimeUnit.SECONDS, new RxThreadFactory(WORKER_THREAD_NAME_PREFIX, priority))`，`(3.1)`即创建了`evictorService`为核心线程数为`1`的`ScheduledExecutorService`的`CachedWorkerPool`对象。 
    * `(4.0)`更新`AtomicReference`的`value`的值为`(3.0)`构造的`CachedWorkerPool`，`!pool.compareAndSet(NONE, update)`不成立。


* 接下来我们看看`createWorker()`：
    ```
    public Worker createWorker() {
        return new EventLoopWorker(pool.get());//1.0
    }

    static final class EventLoopWorker extends Scheduler.Worker {
        private final CompositeDisposable tasks;
        private final CachedWorkerPool pool;
        private final ThreadWorker threadWorker;

        final AtomicBoolean once = new AtomicBoolean();

        EventLoopWorker(CachedWorkerPool pool) {
            this.pool = pool;
            this.tasks = new CompositeDisposable();
            this.threadWorker = pool.get();//2.0
        }
        ....
    }
    ```
    * `(1.0)` `pool.get()`返回的即是 `evictorService`为核心线程数为`1`的`ScheduledExecutorService`的`CachedWorkerPool`对象。 
    * `(2.0)` 调用`CachedWorkerPool`对象的`get()`获取`ThreadWorker`。
      `(2.1)`  `expiringWorkerQueue`初始化为空，所以不成立。
      `(2.2)`  所以`get()`返回的是一个`new ThreadWorker(new RxThreadFactory("RxCachedThreadScheduler", 5))`。
        ```
        ThreadWorker get() {
            if (allWorkers.isDisposed()) {
                return SHUTDOWN_THREAD_WORKER;
            }
            while (!expiringWorkerQueue.isEmpty()) { //2.1
                ThreadWorker threadWorker = expiringWorkerQueue.poll();
                if (threadWorker != null) {
                    return threadWorker;
                }
            }
            // No cached worker found, so create a new one.
            ThreadWorker w = new ThreadWorker(threadFactory);//2.2
            allWorkers.add(w);
            return w;
        }
        ```

* 接下来我们看看`EventLoopWorker`的`schedule()`：
   ```  
    public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
        if (tasks.isDisposed()) {
            // don't schedule, we are unsubscribed
            return EmptyDisposable.INSTANCE;
        }
        return threadWorker.scheduleActual(action, delayTime, unit, tasks);
    }
   ```
   * 在`createWorker()`中我们知到`threadWorker`即为 `new ThreadWorker(new RxThreadFactory("RxCachedThreadScheduler", 5))`。

* 接下来我们看看`ThreadWorker`继承自`NewThreadWorker`的`scheduleActual(...)`：
    ```
    public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
        if (parent != null) {
            if (!parent.add(sr)) {
                return sr;
            }
        }
        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            if (parent != null) {
                parent.remove(sr);
            }
            RxJavaPlugins.onError(ex);
        }
        return sr;
    }
    ```
    * 可以看到基本和 `SingleScheduler`类似，就不再赘述了。

---

#### 小结
`Schedulers.single()`  实际返回的是 `SingleScheduler`。
`Schedulers.io()` 实际返回的是 `IoScheduler`。
`Schedulers.computation()` 实际返回的是 `ComputationScheduler`。
`Schedulers.newThread()` 实际返回的是 `NewThreadScheduler`。

`createWorker()` 返回的值分别为：
`Schedulers.single()`  ： `ScheduledWorker`。
`Schedulers.io()` ：`ThreadWorker`。
`Schedulers.computation()` ：`PoolWorker`。
`Schedulers.newThread()` ：`NewThreadWorker`。

 `SingleScheduler`、`Schedulers.io()`、`NewThreadScheduler` 、`Schedulers.computation()` 最终都是通过 `Executors.newScheduledThreadPool(1, factory);`构建的核心线程数为`1`的线程池。

以上
