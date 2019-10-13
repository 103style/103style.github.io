>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

####  `timer` 操作符

* `timer` 操作符实际上返回的是一个 `ObservableTimer`对象。两个参数的方法默认在 `Schedulers.computation()`中工作。
   ```
    public static Observable<Long> timer(long delay, TimeUnit unit) {
        return timer(delay, unit, Schedulers.computation());
    }
    public static Observable<Long> timer(long delay, TimeUnit unit, Scheduler scheduler) {
        return RxJavaPlugins.onAssembly(new ObservableTimer(Math.max(delay, 0L), unit, scheduler));
    }
  ```
* `ObservableTimer` 源码：
  * 构建了 `TimerObserver` 对象。
  * 执行 观察者 的 `onSubscribe` 方法。
  * 通过`scheduler.scheduleDirect(ios, delay, unit)` 返回一个 `Disposable` 对象。
  * 将返回的 `Disposable` 对象传给 `TimerObserver` 对象的 `setResource` 方法
  ```
  public final class ObservableTimer extends Observable<Long> {
      final Scheduler scheduler;
      final long delay;
      final TimeUnit unit;
      public ObservableTimer(long delay, TimeUnit unit, Scheduler scheduler) {
          this.delay = delay;
          this.unit = unit;
          this.scheduler = scheduler;
      }

      @Override
      public void subscribeActual(Observer<? super Long> observer) {
          TimerObserver ios = new TimerObserver(observer);
          observer.onSubscribe(ios);
          Disposable d = scheduler.scheduleDirect(ios, delay, unit);
          ios.setResource(d);
      }
      ...
  }
  ```
* `TimerObserver `对象源码：
    ```
    static final class TimerObserver extends AtomicReference<Disposable>
    implements Disposable, Runnable {

        final Observer<? super Long> downstream;

        TimerObserver(Observer<? super Long> downstream) {
            this.downstream = downstream;
        }
        ...
        @Override
        public void run() {
            if (!isDisposed()) {
                downstream.onNext(0L);
                lazySet(EmptyDisposable.INSTANCE);
                downstream.onComplete();
            }
        }

        public void setResource(Disposable d) {
            DisposableHelper.trySet(this, d);
        }
    }
    ```
* 首先看 `TimerObserver` 的 `setResource(Disposable d)`方法 里的 `DisposableHelper.trySet(this, d);`：
    ```
    public static boolean trySet(AtomicReference<Disposable> field, Disposable d) {
        if (!field.compareAndSet(null, d)) {
            if (field.get() == DISPOSED) {
                d.dispose();
            }
            return false;
        }
        return true;
    }
    ```
    * `d` 不为 `null`，直接 `return true`；否则判断 是否为 `DISPOSED` 状态，是的话调用传进来的 `Disposable` 对象（也就是之前 `Scheduler` 构建的 `DisposeTask` 对象）的 `dispose` 方法。

*  `scheduler.scheduleDirect(ios, delay, unit)` 方法：
    ```
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        final Worker w = createWorker();
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        DisposeTask task = new DisposeTask(decoratedRun, w);
        w.schedule(task, delay, unit);
        return task;
    }
    ```
    * 首先创建了一个 `Worker`，因为默认是 `Schedulers.computation()`中工作，查看源码可知 实际调用的是  `ComputationScheduler` 的 `createWorker` 方法 。
       `Schedulers`：
        ```
        ...
        static final class ComputationHolder {
            static final Scheduler DEFAULT = new ComputationScheduler();
        }
        ...
        static {
            COMPUTATION = RxJavaPlugins.initComputationScheduler(new ComputationTask());
            ...
        }

        static final class ComputationTask implements Callable<Scheduler> {
            @Override
            public Scheduler call() throws Exception {
                return ComputationHolder.DEFAULT;
            }
        }
        ```
       `RxJavaPlugins`：
        ```
        public static Scheduler initComputationScheduler(@NonNull Callable<Scheduler> defaultScheduler) {
            ObjectHelper.requireNonNull(defaultScheduler, "Scheduler Callable can't be null");
            Function<? super Callable<Scheduler>, ? extends Scheduler> f = onInitComputationHandler;
            if (f == null) {
                return callRequireNonNull(defaultScheduler);
            }
            return applyRequireNonNull(f, defaultScheduler); // JIT will skip this
        }

        static Scheduler callRequireNonNull(@NonNull Callable<Scheduler> s) {
            try {
                return ObjectHelper.requireNonNull(s.call(), "Scheduler Callable result can't be null");
            } catch (Throwable ex) {
                throw ExceptionHelper.wrapOrThrow(ex);
            }
        }
        ```
        **`f` 默认为 `null`，所以返回的是 `callRequireNonNull(defaultScheduler)`，然后实际调用的是 `ComputationTask` 的 `call` 方法。返回的即为 `ComputationScheduler` 对象。**

    *  `ComputationScheduler` 的  `createWorker` 方法 。
        ```
        public ComputationScheduler() {
            this(THREAD_FACTORY);
        }

        public ComputationScheduler(ThreadFactory threadFactory) {
            this.threadFactory = threadFactory;
            this.pool = new AtomicReference<FixedSchedulerPool>(NONE);
            start();
        }

        public Worker createWorker() {
            return new EventLoopWorker(pool.get().getEventLoop());
        }
        ```
        * `pool.get()`通过构造函数我们可知返回的为 `NONE = new FixedSchedulerPool(0, THREAD_FACTORY);` 所以 `pool.get().getEventLoop()` 返回的为  `SHUTDOWN_WORKER = new PoolWorker(new RxThreadFactory("RxComputationShutdown"));`。实际上是创建了一个 `executor` 为 `Executors.newScheduledThreadPool(1, factory)` ，即 `factory` 为 `RxThreadFactory("RxComputationShutdown")` 的 **单线程线程池对象** 的  `PoolWorker`对象
            ```
            FixedSchedulerPool(int maxThreads, ThreadFactory threadFactory) {
                this.cores = maxThreads;
                this.eventLoops = new PoolWorker[maxThreads];
                for (int i = 0; i < maxThreads; i++) {
                    this.eventLoops[i] = new PoolWorker(threadFactory);
                }
            }
            public PoolWorker getEventLoop() {
                int c = cores;
                if (c == 0) {
                    return SHUTDOWN_WORKER;
                }
                return eventLoops[(int)(n++ % c)];
            }
            ```
        * 所以`createWorker` 返回的是：`poolWorker`  是 `factory` 为 `RxThreadFactory("RxComputationShutdown")` 的 **单线程线程池对象** 的  `PoolWorker`对象
            ```
            static final class EventLoopWorker extends Scheduler.Worker {
                private final ListCompositeDisposable serial;
                private final CompositeDisposable timed;
                private final ListCompositeDisposable both;
                private final PoolWorker poolWorker;
    
                volatile boolean disposed;
    
                EventLoopWorker(PoolWorker poolWorker) {
                    this.poolWorker = poolWorker;
                    this.serial = new ListCompositeDisposable();
                    this.timed = new CompositeDisposable();
                    this.both = new ListCompositeDisposable();
                    this.both.add(serial);
                    this.both.add(timed);
                }
            ...
            ```
    * `decoratedRun` 即为  `TimerObserver` 对象。
        ```
        public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
            final Worker w = createWorker();
            final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
            DisposeTask task = new DisposeTask(decoratedRun, w);
            w.schedule(task, delay, unit);
            return task;
        }
        ```
    * 然后构建了一个 `DisposeTask` 对象。
        ```
        static final class DisposeTask implements Disposable, Runnable, SchedulerRunnableIntrospection {
            final Runnable decoratedRun;
            final Worker w;
            Thread runner;
    
            DisposeTask(@NonNull Runnable decoratedRun, @NonNull Worker w) {
                this.decoratedRun = decoratedRun;
                this.w = w;
            }
    
            @Override
            public void run() {
                runner = Thread.currentThread();
                try {
                    decoratedRun.run();
                } finally {
                    dispose();
                    runner = null;
                }
            }
            ...
        }
        ```
    * `createWorker` 返回的 `poolWorker`  是 `factory` 为 `RxThreadFactory("RxComputationShutdown")` 的 **单线程线程池对象** 的  `PoolWorker`对象，并执行 `schedule` 方法。
    实际上是执行了  **单线程线程池对象** `Executors.newScheduledThreadPool(1, factory)`  的 `schedule(task, delayTime, unit)`方法，并将返回值 `Future` 对象 传给`ScheduledRunnable` 的 `setFuture` 方法。
        ```
        public Disposable schedule(@NonNull final Runnable action, long delayTime, @NonNull TimeUnit unit) {
            if (disposed) {
                return EmptyDisposable.INSTANCE;
            }
            return scheduleActual(action, delayTime, unit, null);
        }
    
        public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
            Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
            ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
            ...
            Future<?> f;
            try {
                if (delayTime <= 0) {
                    f = executor.submit((Callable<Object>)sr);
                } else {
                    f = executor.schedule((Callable<Object>)sr, delayTime, unit);
                }
                sr.setFuture(f);
            } catch (RejectedExecutionException ex) {
                ...
                RxJavaPlugins.onError(ex);
            }
            return sr;
        }
        ```
* 线程池的`schedule(task, delayTime, unit)` 方法实际时延时 `delayTime` 执行 `task` 的 `run` 方法。即为 执行 `TimerObserver` 对象的 `run` 方法。
    ```
    public void subscribeActual(Observer<? super Long> observer) {
        TimerObserver ios = new TimerObserver(observer);
        observer.onSubscribe(ios);
        Disposable d = scheduler.scheduleDirect(ios, delay, unit);
        ios.setResource(d);
    }
    ```
* `TimerObserver` 对象的 `run` 方法： 即执行了 **观察者** 的  `onNext(0L)` 和`onComplete()`。
    ```
    public void run() {
        if (!isDisposed()) {
            downstream.onNext(0L);
            lazySet(EmptyDisposable.INSTANCE);
            downstream.onComplete();
        }
    }
    ```

---

#### `interval` 系列操作符
* `interval`系列 包含 `interval`  和 `intervalRange`两个操作符，包含以下 6 个方法：
  * `interval(long period, TimeUnit unit)` 
  * `interval(long initialDelay, long period, TimeUnit unit)` 
  * `interval(long period, TimeUnit unit, Scheduler scheduler)` 
  * `interval(long initialDelay, long period, TimeUnit unit, Scheduler scheduler)`
  * `intervalRange(long start, long count, long initialDelay, long period, TimeUnit unit)` 
  * `intervalRange(long start, long count, long initialDelay, long period, TimeUnit unit, Scheduler scheduler)`

  分别返回的是 `ObservableInterval` 和  `ObservableIntervalRange` 对象，默认的 `Scheduler` 为 `Schedulers.computation()`。
```
public static Observable<Long> interval(long initialDelay, long period, TimeUnit unit) {
    return interval(initialDelay, period, unit, Schedulers.computation());
}
public static Observable<Long> interval(long initialDelay, long period, TimeUnit unit, Scheduler scheduler) {
    return RxJavaPlugins.onAssembly(new ObservableInterval(Math.max(0L, initialDelay), Math.max(0L, period), unit, scheduler));
}
public static Observable<Long> intervalRange(long start, long count, long initialDelay, long period, TimeUnit unit) {
    return intervalRange(start, count, initialDelay, period, unit, Schedulers.computation());
}
public static Observable<Long> intervalRange(long start, long count, long initialDelay, long period, TimeUnit unit, Scheduler scheduler) {
    return RxJavaPlugins.onAssembly(new ObservableIntervalRange(start, end, Math.max(0L, initialDelay), Math.max(0L, period), unit, scheduler));
}
```
* `ObservableInterval` 源码：
    * 构建了 `IntervalObserver` 对象。
    * 因为默认`Schedulers.computation()` 所以 `sch instanceof TrampolineScheduler`不成立，除非我们手动传参 `Scheduler` 为 `Schedulers.trampoline()`。
    * 和前面的 `ObservableTimer`类似， 即为调用  `ObservableInterval` 的 `run` 方法。只是返回的为`PeriodicDirectTask`对象。
    * `setResource` 和  `ObservableTimer`类似，就不再赘述了。
    ```
    public final class ObservableInterval extends Observable<Long> {
        final Scheduler scheduler;
        final long initialDelay;
        final long period;
        final TimeUnit unit;
    
        public ObservableInterval(long initialDelay, long period, TimeUnit unit, Scheduler scheduler) {
            this.initialDelay = initialDelay;
            this.period = period;
            this.unit = unit;
            this.scheduler = scheduler;
        }
    
        @Override
        public void subscribeActual(Observer<? super Long> observer) {
            IntervalObserver is = new IntervalObserver(observer);
            observer.onSubscribe(is);
    
            Scheduler sch = scheduler;
            if (sch instanceof TrampolineScheduler) {
                Worker worker = sch.createWorker();
                is.setResource(worker);
                worker.schedulePeriodically(is, initialDelay, period, unit);
            } else {
                Disposable d = sch.schedulePeriodicallyDirect(is, initialDelay, period, unit);
                is.setResource(d);
            }
        }
        ...
    }
    ```
* `sch.schedulePeriodicallyDirect(is, initialDelay, period, unit)` 实际调用的为 `schedulePeriodically`方法：
    * 将 `interval` 的间隔时间转化为 `Nanoseconds`。
    * 然后设置 第一次的 响应时间为  `当前时间+ 间隔时间` 的 纳秒数。 
    * 里面又将 `PeriodicDirectTask`对象 包装成 `PeriodicTask` 对象。
    ```
    public Disposable schedulePeriodicallyDirect(@NonNull Runnable run, long initialDelay, long period, @NonNull TimeUnit unit) {
        final Worker w = createWorker();
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        PeriodicDirectTask periodicTask = new PeriodicDirectTask(decoratedRun, w);
        Disposable d = w.schedulePeriodically(periodicTask, initialDelay, period, unit);
        if (d == EmptyDisposable.INSTANCE) {
            return d;
        }
        return periodicTask;
    }

    public Disposable schedulePeriodically(@NonNull Runnable run, final long initialDelay, final long period, @NonNull final TimeUnit unit) {
        final SequentialDisposable first = new SequentialDisposable();
        final SequentialDisposable sd = new SequentialDisposable(first);
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        final long periodInNanoseconds = unit.toNanos(period);
        final long firstNowNanoseconds = now(TimeUnit.NANOSECONDS);
        final long firstStartInNanoseconds = firstNowNanoseconds + unit.toNanos(initialDelay);

        Disposable d = schedule(new PeriodicTask(firstStartInNanoseconds, decoratedRun, firstNowNanoseconds, sd,
                periodInNanoseconds), initialDelay, unit);
        if (d == EmptyDisposable.INSTANCE) {
            return d;
        }
        first.replace(d);
        return sd;
    }
    ```
* `PeriodicTask` 对象的 `run` 方法
    * `decoratedRun.run();` 又调用了 `PeriodicDirectTask`对象的 `run` 方法.
    *  `run` 方法的最后 `sd.replace(schedule(this, delay, TimeUnit.NANOSECONDS));` 这里又 **重复执行** 这个任务，直到 `IntervalObserver`对象 `isDisposed()` 为 `true`。
    ```
     final class PeriodicTask implements Runnable, SchedulerRunnableIntrospection {
        final Runnable decoratedRun;
        final SequentialDisposable sd;
        final long periodInNanoseconds;
        long count;
        long lastNowNanoseconds;
        long startInNanoseconds;

        PeriodicTask(long firstStartInNanoseconds, @NonNull Runnable decoratedRun,
                long firstNowNanoseconds, @NonNull SequentialDisposable sd, long periodInNanoseconds) {
            this.decoratedRun = decoratedRun;
            this.sd = sd;
            this.periodInNanoseconds = periodInNanoseconds;
            lastNowNanoseconds = firstNowNanoseconds;
            startInNanoseconds = firstStartInNanoseconds;
        }

        @Override
        public void run() {
            decoratedRun.run();
            if (!sd.isDisposed()) {
                long nextTick;
                long nowNanoseconds = now(TimeUnit.NANOSECONDS);
                // If the clock moved in a direction quite a bit, rebase the repetition period
                if (nowNanoseconds + CLOCK_DRIFT_TOLERANCE_NANOSECONDS < lastNowNanoseconds
                        || nowNanoseconds >= lastNowNanoseconds + periodInNanoseconds + CLOCK_DRIFT_TOLERANCE_NANOSECONDS) {
                    nextTick = nowNanoseconds + periodInNanoseconds;
                    /*
                     * Shift the start point back by the drift as if the whole thing
                     * started count periods ago.
                     */
                    startInNanoseconds = nextTick - (periodInNanoseconds * (++count));
                } else {
                    nextTick = startInNanoseconds + (++count * periodInNanoseconds);
                }
                lastNowNanoseconds = nowNanoseconds;

                long delay = nextTick - nowNanoseconds;
                sd.replace(schedule(this, delay, TimeUnit.NANOSECONDS));
            }
        }
        ...
    }
    ```

* `PeriodicDirectTask` 的 `run` 方法：
    * 实际调用的即为`IntervalObserver` 的 `run()`
    ```
    static final class PeriodicDirectTask
    implements Disposable, Runnable, SchedulerRunnableIntrospection {
        final Runnable run;
        final Worker worker;
        volatile boolean disposed;

        PeriodicDirectTask(@NonNull Runnable run, @NonNull Worker worker) {
            this.run = run;
            this.worker = worker;
        }

        @Override
        public void run() {
            if (!disposed) {
                try {
                    run.run();
                } catch (Throwable ex) {
                    Exceptions.throwIfFatal(ex);
                    worker.dispose();
                    throw ExceptionHelper.wrapOrThrow(ex);
                }
            }
        }
        ...
    }
    ```
* `IntervalObserver` 的 `run()`：
    * 调用 **观察者** 的 `onNext` 方法
    ```
    static final class IntervalObserver
    extends AtomicReference<Disposable>
    implements Disposable, Runnable {
        final Observer<? super Long> downstream;
        long count;

        IntervalObserver(Observer<? super Long> downstream) {
            this.downstream = downstream;
        }

        @Override
        public void run() {
            if (get() != DisposableHelper.DISPOSED) {
                downstream.onNext(count++);
            }
        }
    }
    ```

* 然后直到我们直接调用  `dispose()` 方法结束流程。

以上
