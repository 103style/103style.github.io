>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`Base on RxJava 2.X`

#### 简介
首先我们来看`subscribeOn`和`observeOn`这两个方法的实现：
* `subscribeOn`
    ```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }
    ```
* `observeOn`
    ```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> observeOn(Scheduler scheduler) {
        return observeOn(scheduler, false, bufferSize());
    }
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError) {
        return observeOn(scheduler, delayError, bufferSize());
    }

    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }
    ```
我们可以看到分别返回了`ObservableSubscribeOn`和`ObservableObserveOn`对象，下面对这两个类分别介绍。

---

####  ObservableSubscribeOn 源码解析
```
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
        observer.onSubscribe(parent);
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
    ....
}
```
* 通过之前的 [Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11) 的介绍，我们知道`subscribe(observer)`实际上是调用前一步返回对象的`subscribeActual(observer);`方法。
* 这里首先构造了一个 `SubscribeOnObserver`对象，然后执行 **观察者** 的  `onSubscribe` 方法。
* 然后将在传入的`Scheduler`中执行任务完成返回的结果传入 `SubscribeOnObserver`的 `setDisposable`方法。
* `scheduler.scheduleDirect(new SubscribeTask(parent))`，这里通过之前 [RxJava之Schedulers源码介绍](https://www.jianshu.com/p/e3fbcd79b037) 我们知道，实际时候执行了 `SubscribeTask(parent)`的 `run`方法。通过下面的源代码` source.subscribe(parent)`，我们知道 实际上 `run` 方法 就是 调用了`subscribeOn`前一步操作符返回对象的 `subscribeActual(observer);`方法。
    ```
    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
    ```

`SubscribeOnObserver`源码：
```
static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {
    private static final long serialVersionUID = 8094547886072529208L;
    final Observer<? super T> downstream;
    final AtomicReference<Disposable> upstream;

    SubscribeOnObserver(Observer<? super T> downstream) {
        this.downstream = downstream;
        this.upstream = new AtomicReference<Disposable>();
    }

    @Override
    public void onSubscribe(Disposable d) {
        DisposableHelper.setOnce(this.upstream, d);
    }

    @Override
    public void onNext(T t) {
        downstream.onNext(t);
    }

    @Override
    public void onError(Throwable t) {
        downstream.onError(t);
    }

    @Override
    public void onComplete() {
        downstream.onComplete();
    }
    ...
    void setDisposable(Disposable d) {
        DisposableHelper.setOnce(this, d);
    }
}
```
* 我们可以看到 `onNext`、`onError`、`onComplete` 实际上还是调用了 观察者的 对应方法。
* `DisposableHelper.setOnce(this, d);` 即为设置`SubscribeOnObserver`的`value`值为线程池执行的任务结果。
    ```
    public static boolean setOnce(AtomicReference<Disposable> field, Disposable d) {
        ObjectHelper.requireNonNull(d, "d is null");
        if (!field.compareAndSet(null, d)) {
            d.dispose();
            if (field.get() != DISPOSED) {
                reportDisposableSet();
            }
            return false;
        }
        return true;
    }
    ```
我们来个示例介绍下：
```
Observable
        .create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                System.out.println("subscribe = " + Thread.currentThread().getName());
                for (int i = 0; i < 3; i++) {
                    emitter.onNext(String.valueOf(i));
                }
                emitter.onComplete();
            }
        })
        .subscribeOn(Schedulers.single())
        .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                System.out.println("onSubscribe thread name = " + Thread.currentThread().getName());
            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext s = " + s + " thread name = " + Thread.currentThread().getName());
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("onError thread name = " + Thread.currentThread().getName());
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete thread name = " + Thread.currentThread().getName());
            }
        });
```
输出结果：
```
onSubscribe thread name = main
subscribe = RxSingleScheduler-1
onNext s = 0 thread name = RxSingleScheduler-1
onNext s = 1 thread name = RxSingleScheduler-1
onNext s = 2 thread name = RxSingleScheduler-1
onComplete thread name = RxSingleScheduler-1
```
通过输出结果我们可以看到 任务处理都是在 `Schedulers.single()`构建的线程池中执行的。
现在来一步一步介绍，顺便复习一下：
流程图大致如下：
![SubscribeOnObserver示例流程图](https://upload-images.jianshu.io/upload_images/1709375-1d9f602865e67afa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `(1.0)` `create` 操作符 返回的是 `ObservableCreate`对象。
* `(2.0)` 然后 `ObservableCreate.subscribeOn(Schedulers.single())`返回 `source` 为`ObservableCreate`，`scheduler` 为 `SingleScheduler` 的 `ObservableSubscribeOn`对象。
* `(3.0)` 然后 `ObservableSubscribeOn.subscribe(new Observer<T>(){})`，即调用 `ObservableSubscribeOn` 的 `subscribeActual(observer)`。
* `(4.0)` 然后执行 `observer.onSubscribe(parent);`，即执行观察者的 `onSubscribe(...)`方法。
* `(5.0)` 接着在`SingleScheduler`构建的线程池中执行 `SubscribeTask` 的 `run`方法(`source.subscribe(parent)`)。 
  即执行 `ObservableCreate.subscribe(new SubscribeOnObserver<T>(observer))`。
  即为 `ObservableCreate.subscribeActual(new SubscribeOnObserver<T>(observer))`。
* `(6.0)` 然后执行 `SubscribeOnObserver` 的 `onSubscribe(...)` 。
* `(7.0)` 然后执行`create`操作符传进来的`ObservableOnSubscribe`的 `subscribe(ObservableEmitter<String> emitter) `方法。
* `(8.0)` 接着我们在`subscribe(...)`中依次执行了 三次`onNext`和 一次`onComplete`。
  即调用 `new SubscribeOnObserver<T>(observer)`的三次`onNext`和 一次`onComplete`。
  即为`subscribe`传入的`observer`的三次`onNext`和 一次`onComplete`。

---

#### ObservableObserveOn 源码解析
* `observeOn`函数中的`bufferSize`，在`2.X`中默认为 **128**.
```
public static int bufferSize() {
    return Flowable.bufferSize();
}
public static int bufferSize() {
    return BUFFER_SIZE;
}
static final int BUFFER_SIZE;
static {
    BUFFER_SIZE = Math.max(1, Integer.getInteger("rx2.buffer-size", 128));
}
```

* `observeOn` 方法：
    ```
    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }
    ```
* `ObservableObserveOn`主要的方法: 
    ```
    public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
        final Scheduler scheduler;
        final boolean delayError;
        final int bufferSize;
        public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
            super(source);
            this.scheduler = scheduler;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        @Override
        protected void subscribeActual(Observer<? super T> observer) {
            if (scheduler instanceof TrampolineScheduler) {
                source.subscribe(observer);
            } else {
                Scheduler.Worker w = scheduler.createWorker();
                source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
            }
        }
        ...
    }
    ```
    * 赋值`source`为链式调用上一步返回的对象。
    * 保存传进来的 `Scheduler`、`delayError`、`bufferSize`的值。
    * 然后在 `subscribe` 的时候 调用 `subscribeActual`, 先判断 `scheduler`是否是 `TrampolineScheduler`的子类：
        * 是的话直接把 `observer` 传给 链式调用上一步返回的对象的 `subscribeActual`方法。
        * 不是的话 就把`observer` 包装成一个`ObserveOnObserver` 对象传给 链式调用上一步返回的对象的 `subscribeActual`方法。
    * 通过上面 `subscribeOn` 的介绍， 我们知道接下来就是调用 观察者的 `onSubscribe` 方法，以及后续的调用逻辑 `onNext`、`onComplete`以及`onError`，即`ObserveOnObserver` 对象对应的方法。

* 接下来我们看看 `ObserveOnObserver` 的源码：
    ```
    static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
        final Observer<? super T> downstream;
        final Scheduler.Worker worker;
        final boolean delayError;
        final int bufferSize;
        ...
        ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
            this.downstream = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.validate(this.upstream, d)) {
                ...
                queue = new SpscLinkedArrayQueue<T>(bufferSize);
                downstream.onSubscribe(this);
            }
        }
        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }
            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }

        @Override
        public void onError(Throwable t) {
            if (done) {
                RxJavaPlugins.onError(t);
                return;
            }
            error = t;
            done = true;
            schedule();
        }

        @Override
        public void onComplete() {
            if (done) {
                return;
            }
            done = true;
            schedule();
        }
        ...
    }
    ```
    * 重写的`onSubscribe` 即调用观察者的 `onSubscribe`。
    * `onNext`、`onError`、`onComplete`都是调用 `schedule()`。
    * 我们来看看`schedule()`的实现：即在传进来的 `Scheduler` 对象构建的线程池里执行当前类的 `run()`。
        ```
        void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }
        ```
    *  `run()`的代码实现：
        ```
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }
        ```
    * `outputFused` 默认是 `false`，我们看看 `drainNormal()`的代码实现：
        当`outputFused`为 `true`是，则下面调用的`onNext` 改成 `onComplete`。
        ```
        void drainNormal() {
            int missed = 1;
            final SimpleQueue<T> q = queue; //1.0
            final Observer<? super T> a = downstream;
            for (;;) {
                if (checkTerminated(done, q.isEmpty(), a)) {
                    return;
                }
                for (;;) {
                    boolean d = done;
                    T v;
                    try {
                        v = q.poll();//2.0
                    } catch (Throwable ex) {
                        ...
                        a.onError(ex);
                        worker.dispose();
                        return;
                    }
                    boolean empty = v == null;
                    if (checkTerminated(d, empty, a)) {//2.1
                        return;
                    }
                    if (empty) {//2.2
                        break;
                    }
                    a.onNext(v);//2.3
                }
                missed = addAndGet(-missed);//3.0
                if (missed == 0) {//3.1
                    break;
                }
            }
        }
        ```
        * `(1.0):` 我们在上面的 `onNext()` 中看到，每次调用都会把传入的对象存入`queue`中。
        * `(2.0):` 在循环中依次获取存入的对象，`(2.1)`如果 已经是`done`状态 或者 `disposed`则直接结束。`(2.2)`如果 队列中没有对象了，即终止循环。`(2.3)`否则调用 **观察者** 的 `onNext` 方法。
        * `(3.0):` `addAndGet(-missed);`即通过原子操作把·missed·的值置为`0 `。`(3.1)`然后结束`onNext`。

来我们继续举个例子：给`subscribeOn`例子加上`observeOn` 方法：
```
Observable
        .create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                System.out.println("subscribe = " + Thread.currentThread().getName());
                for (int i = 0; i < 5; i++) {
                    emitter.onNext(String.valueOf(i));
                }
                emitter.onComplete();
            }
        })
        .subscribeOn(Schedulers.single())
        .observeOn(Schedulers.io())
        .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                System.out.println("d.classname = " + d.getClass().getSimpleName());
                System.out.println("onSubscribe thread name = " + Thread.currentThread().getName());
            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext s = " + s + " thread name = " + Thread.currentThread().getName());
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("onError thread name = " + Thread.currentThread().getName());
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete thread name = " + Thread.currentThread().getName());
            }
        });
```
输出结果：
```
System.out: d.classname = ObserveOnObserver
System.out: onSubscribe thread name = main
System.out: subscribe = RxSingleScheduler-1
System.out: onNext s = 0 thread name = RxCachedThreadScheduler-1
System.out: onNext s = 1 thread name = RxCachedThreadScheduler-1
System.out: onNext s = 2 thread name = RxCachedThreadScheduler-1
System.out: onComplete thread name = RxCachedThreadScheduler-1
```
通过输出结果我们可以看到 : 
* `create`操作符 传入的`ObservableOnSubscribe`  的 `subscribe`方法是在`Schedulers.single()`构建的线程池中执行的。
* `onNext` 和`onComplete` 则是在`Schedulers.io()`构建的线程池中执行的 。

继续来看下`subscribeOn`流程图：
![SubscribeOnObserver示例流程图](https://upload-images.jianshu.io/upload_images/1709375-1d9f602865e67afa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 上述示例相对于 `subscribeOn`来说只是 把 `subscribe(observer)` 里得参数改成了 `ObserveOnObserver`对象。
* `(4.0:)` 执行`ObserveOnObserver` 的 `onSubscribe`方法。即`observer.onSubscribe(ObserveOnObserver)` 即下面方法的 `Disposable`对象为`ObserveOnObserver`对象。
    ```
    new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
        }
         ...
    });
    ```
* `(5.0:)` 在`SingleScheduler`构建的线程池中执行`source.subscribe(parent);`，即运行如下代码：
    ```
    ObservableCreate.subscribeActual(
            new ObserveOnObserver<T>(
                    observer,
                    new EventLoopWorker(new CachedWorkerPool(KEEP_ALIVE_TIME, KEEP_ALIVE_UNIT, threadFactory)),
                    delayError,
                    bufferSize)
    );
    ```
* 我们再来回顾下`ObservableCreate.subscribeActual(observer)`：
    ```
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);
        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

    static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
        ...
        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException(...));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }
        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }
        ...
    }
    ```
* `(8.0:)` 所以调用 `onNext(T t)`和`onComplete()`即调用 `ObserveOnObserver`对象的 `onNext(T t)`和`onComplete()`。 即切换到`Schedulers.io()`构建的线程池执行`onNext(T t)`和`onComplete()`。

----

#### 小结 
`subscribeOn`返回得即`ObservableSubscribeOn`对象。
`ObservableSubscribeOn`的`subscribeActual`即为在 传入的 `XXXScheduler`中 执行 上一步返回对象的 `subscribeActual`方法。

`observeOn`返回得即`ObservableObserveOn`对象。
`ObservableObserveOn`的`subscribeActual`即为把 传入的 `XXXScheduler` 和 `observer`包装成一个 `Observer` 传给上一步返回对象的 `subscribeActual`方法，让 `onNext`、`onComplete`、`onNext`都在传入的 `XXXScheduler` 构建的线程池中执行。

**所以，你知道RxJava是如何完成线程切换的了吗？**

以上 
