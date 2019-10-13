>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

本文基于 `RxJava 2.x` 版本

---

### 目录
* `RxJava`背压策略简介
* `Observable`背压导致崩溃的原因
* `Flowable` 使用介绍
* `五种`背压策略源码分析
* 小结

---

### RxJava背压策略简介
[官方介绍](https://github.com/ReactiveX/RxJava/wiki/Backpressure-(2.0))
>**Backpressure** is when in an **Flowable** processing pipeline, some asynchronous stages can't process the values fast enough and need a way to tell the upstream producer to slow down.
>**背压**是在**Flowable**处理事件流中，某些异步阶段无法足够快地处理这些值，并且需要一种方法来告诉上游生产商减速。

所以`RxJava`的背压策略(`Backpressure`)是指处理上述上游流速过快现象的一种策略。 类似 [Java中的线程池](https://www.jianshu.com/p/13c82f1a7ad9) 中的饱和策略`RejectedExecutionHandler`。

---

### Observable背压导致崩溃的原因

我们先使用 `Observable`看看是什么情况:
```
Observable
        .create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                for (int i = 0; ; i++) {
                    emitter.onNext(i);
                }
            }
        })
        .subscribeOn(Schedulers.computation())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(integer);
            }
        });
```
![image.png](https://upload-images.jianshu.io/upload_images/1709375-84bcb20abef33e77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输出：
```
I/art: Background partial concurrent mark sweep GC freed 7(224B) AllocSpace objects, 0(0B) LOS objects, 27% free, 43MB/59MB, paused 528us total 106.928ms
I/System.out: 0
I/art: Background partial concurrent mark sweep GC freed 8(256B) AllocSpace objects, 0(0B) LOS objects, 20% free, 62MB/78MB, paused 1.065ms total 327.346ms
I/System.out: 1
I/art: Background partial concurrent mark sweep GC freed 8(256B) AllocSpace objects, 0(0B) LOS objects, 16% free, 82MB/98MB, paused 1.345ms total 299.700ms
I/art: Background partial concurrent mark sweep GC freed 8(256B) AllocSpace objects, 0(0B) LOS objects, 13% free, 103MB/119MB, paused 1.609ms total 377.432ms
I/System.out: 2
I/art: Background sticky concurrent mark sweep GC freed 29(800B) AllocSpace objects, 0(0B) LOS objects, 0% free, 120MB/120MB, paused 1.280ms total 105.749ms
I/art: Background partial concurrent mark sweep GC freed 22(640B) AllocSpace objects, 0(0B) LOS objects, 11% free, 126MB/142MB, paused 1.818ms total 679.398ms
I/System.out: 3
I/art: Background partial concurrent mark sweep GC freed 9(288B) AllocSpace objects, 0(0B) LOS objects, 9% free, 148MB/164MB, paused 1.946ms total 555.619ms
I/System.out: 4
I/art: Background sticky concurrent mark sweep GC freed 29(800B) AllocSpace objects, 0(0B) LOS objects, 0% free, 165MB/165MB, paused 1.253ms total 107.036ms
I/art: Background partial concurrent mark sweep GC freed 8(256B) AllocSpace objects, 0(0B) LOS objects, 8% free, 172MB/188MB, paused 2.355ms total 570.029ms
I/art: Background sticky concurrent mark sweep GC freed 28(768B) AllocSpace objects, 0(0B) LOS objects, 0% free, 188MB/188MB, paused 11.474ms total 82.399ms
I/System.out: 5
I/art: Background partial concurrent mark sweep GC freed 23(672B) AllocSpace objects, 0(0B) LOS objects, 7% free, 197MB/213MB, paused 2.355ms total 631.635ms
I/art: Background partial concurrent mark sweep GC freed 22(640B) AllocSpace objects, 0(0B) LOS objects, 6% free, 226MB/242MB, paused 3.091ms total 908.581ms
I/System.out: 6
I/art: Background sticky concurrent mark sweep GC freed 29(800B) AllocSpace objects, 0(0B) LOS objects, 0% free, 242MB/242MB, paused 1.672ms total 102.676ms
I/art: Waiting for a blocking GC Alloc
I/art: Clamp target GC heap from 267MB to 256MB
I/art: Alloc sticky concurrent mark sweep GC freed 0(0B) AllocSpace objects, 0(0B) LOS objects, 1% free, 252MB/256MB, paused 1.581ms total 10.336ms
I/art: WaitForGcToComplete blocked for 12.447ms for cause Alloc
I/art: Starting a blocking GC Alloc
I/art: Starting a blocking GC Alloc
I/System.out: 9
I/art: Waiting for a blocking GC Alloc
I/art: Waiting for a blocking GC Alloc
I/art: Clamp target GC heap from 268MB to 256MB
I/art: Alloc concurrent mark sweep GC freed 0(0B) AllocSpace objects, 0(0B) LOS objects, 1% free, 252MB/256MB, paused 1.574ms total 818.037ms
I/art: WaitForGcToComplete blocked for 2.539s for cause Alloc
I/art: Starting a blocking GC Alloc
I/art: Waiting for a blocking GC Alloc
W/art: Throwing OutOfMemoryError "Failed to allocate a 12 byte allocation with 4109520 free bytes and 3MB until OOM; failed due to fragmentation (required continguous free 4096 bytes for a new buffer where largest contiguous free 0 bytes)"
```

我们可以从上图中看到，内存在逐步上升，在一定的时间后，到达`256M`之后会触发GC，最后抛出`OutOfMemoryError`。因为上游的事件发送太快而下游的消费者消耗的比较慢。

**那导致内存暴增的源头是什么呢 ？**

---
我们对上面的代码做一点点修改，注释了`observeOn(AndroidSchedulers.mainThread())`，会发现内存显示很正常，不会存在上述问题。
```
    Observable
            .create(new ObservableOnSubscribe<Integer>() {
                @Override
                public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                    for (int i = 0; ; i++) {
                        emitter.onNext(i);
                    }
                }
            })
            .subscribeOn(Schedulers.computation())
//          .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Consumer<Integer>() {
                @Override
                public void accept(Integer integer) throws Exception {
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(integer);
                }
            });
```
![注释了observeOn](https://upload-images.jianshu.io/upload_images/1709375-69a5da4108921003.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**所以内存暴增的源头就在 `observeOn(AndroidSchedulers.mainThread())`.**

我们来看看 `observeOn`的源码，通过  [RxJava subscribeOn和observeOn源码介绍](https://www.jianshu.com/p/757a28295804)，我们知道在 `ObservableObserveOn.ObserveOnObserver`的 `onSubscribe`中构建了一个容量默认为`128`的`SpscLinkedArrayQueue`。
```
queue = new SpscLinkedArrayQueue<T>(bufferSize);
```
上游每发送一个事件都会通过`queue.offer(t)`保存到`SpscLinkedArrayQueue`中。
```
public void onNext(T t) {
    if (done) {
        return;
    }
    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}
```

我们可以写个测试代码来看看，因为生产比消费快的多，相当于一直添加元素，如下：
```
private void test(){
    SpscLinkedArrayQueue<Integer> queue = new SpscLinkedArrayQueue<>(128);
    for (int i = 0; ; i++) {
        queue.offer(i);
    }
}
```
运行会发现内存变化和`Observable`一样迅速暴增。
![测试代码内存变化](https://upload-images.jianshu.io/upload_images/1709375-0d401b973b3cbab7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`SpscLinkedArrayQueue`的详细介绍后面再说。现在可以大致理解为 **一直狂吃，然后最后撑破肚皮，然后裂开**。

---

### Flowable的用法
我们来看看 `Flowable`的用法：
```
Flowable.create(FlowableOnSubscribe<T> source, BackpressureStrategy mode)
```
`BackpressureStrategy` 包含五种模式：`MISSING`、`ERROR`、`BUFFER`、`DROP`、`LATEST`。


下面对这五种`BackpressureStrategy` 分别介绍其用法以及 `发送事件速度 > 接收事件速度` 时的处理方式：
* `BackpressureStrategy.MISSING`
    处理方式：抛出异常`MissingBackpressureException`，并提示 **缓存区满了**
    代码示例：
    ```
    Flowable
            .create(new FlowableOnSubscribe<Object>() {
                @Override
                public void subscribe(FlowableEmitter<Object> emitter) throws Exception {
                    for (int i = 0; i < Flowable.bufferSize() * 2; i++) {
                        emitter.onNext(i);
                    }
                    emitter.onComplete();
                }
            }, BackpressureStrategy.MISSING)
            .subscribeOn(Schedulers.computation())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Subscriber<Object>() {
                @Override
                public void onSubscribe(Subscription s) {
                    s.request(Integer.MAX_VALUE);
                }

                @Override
                public void onNext(Object o) {
                    System.out.println("onNext: " + o);
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                }

                @Override
                public void onComplete() {
                    System.out.println("onComplete");
                }
            });
    ```
    输出结果：
    ```
    System.out: onNext: 0
    System.err: io.reactivex.exceptions.MissingBackpressureException: Queue is full?!
    ```

* `BackpressureStrategy.ERROR`
    处理方式：**直接抛出异常`MissingBackpressureException`**
    修改上述代码的 `BackpressureStrategy.MISSING`为`BackpressureStrategy.ERROR`：
    ```
    Flowable
            .create(new FlowableOnSubscribe<Object>() {
                ...
            }, BackpressureStrategy.ERROR)
            ...
    ```
    输出结果：
    ```
    System.out: onNext: 0
    System.err: io.reactivex.exceptions.MissingBackpressureException: create: could not emit value due to lack of requests
    ```

* `BackpressureStrategy.BUFFER`
    处理方式：类似`Observable`一样**扩充缓存区大小**
    修改上述代码的 `BackpressureStrategy.MISSING`为`BackpressureStrategy.BUFFER`：
    ```
    Flowable
            .create(new FlowableOnSubscribe<Object>() {
                ...
            }, BackpressureStrategy.BUFFER)
            ...
    ```
    输出结果：
    ```
    System.out: onNext: 0
    System.out: onNext: 1
    System.out: onNext: 2
    System.out: onNext: 3
    System.out: onNext: 4
    System.out: onNext: 5
    System.out: onNext: 6
    ...
    System.out: onNext: 247
    System.out: onNext: 248
    System.out: onNext: 249
    System.out: onNext: 250
    System.out: onNext: 251
    System.out: onNext: 252
    System.out: onNext: 253
    System.out: onNext: 254
    System.out: onNext: 255
    System.out: onComplete
    ```
* `BackpressureStrategy.DROP`
    处理方式：**丢弃缓存区满后处理缓冲区数据期间发送过来的事件**
    示例代码：
    ```
      Flowable
            .create(new FlowableOnSubscribe<Object>() {
                @Override
                public void subscribe(FlowableEmitter<Object> emitter) throws Exception {
                    for (int i = 0; ; i++) {
                        emitter.onNext(i);
                    }
                }
            }, BackpressureStrategy.DROP)
            .subscribeOn(Schedulers.computation())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Subscriber<Object>() {
                @Override
                public void onSubscribe(Subscription s) {
                    s.request(Integer.MAX_VALUE);
                }

                @Override
                public void onNext(Object o) {
                    System.out.println("onNext: " + o);
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }

                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                }

                @Override
                public void onComplete() {
                    System.out.println("onComplete");
                }
            });
    ```
    输出结果：
    ```
    System.out: onNext: 0
    System.out: onNext: 1
    System.out: onNext: 2
    System.out: onNext: 3
    ...
    System.out: onNext: 124
    System.out: onNext: 125
    System.out: onNext: 126
    System.out: onNext: 127
    System.out: onNext: 1070801
    System.out: onNext: 1070802
    System.out: onNext: 1070803
    System.out: onNext: 1070804
    System.out: onNext: 1070805
    ...
    ```

* `BackpressureStrategy.LATEST`
    处理方式：**丢弃缓存区满后处理缓冲区数据期间发送过来的非最后一个事件**。下面示例代码输出了 `129` 个事件，下面的源码分析会介绍。
    示例代码：
    ```
    Flowable
            .create(new FlowableOnSubscribe<Object>() {
                @Override
                public void subscribe(FlowableEmitter<Object> emitter) throws Exception {
                    for (int i = 0; i < Flowable.bufferSize() * 2; i++) {
                        emitter.onNext(i);
                    }
                    emitter.onComplete();
                }
            }, BackpressureStrategy.LATEST)
            .subscribeOn(Schedulers.computation())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Subscriber<Object>() {
                @Override
                public void onSubscribe(Subscription s) {
                    s.request(Integer.MAX_VALUE);
                }

                @Override
                public void onNext(Object o) {
                    System.out.println("onNext: " + o);
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }

                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                }

                @Override
                public void onComplete() {
                    System.out.println("onComplete");
                }
            });
    ```
    输出结果：
    ```
    System.out: onNext: 0
    System.out: onNext: 1
    System.out: onNext: 2
    System.out: onNext: 3
    ...
    System.out: onNext: 124
    System.out: onNext: 125
    System.out: onNext: 126
    System.out: onNext: 127
    System.out: onNext: 255
    System.out: onComplete
    ```

---

### 五种背压策略源码分析
通知之前 [RxJava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11) 的介绍。我们知道`Flowable.create(new FlowableOnSubscribe<Object>(){...}, BackpressureStrategy.LATEST)` 返回的是一个`FlowableCreate`对象。

分别对不同的背压策略创建了不同的`Emitter`.
```
public final class FlowableCreate<T> extends Flowable<T> {
    //...
    public FlowableCreate(FlowableOnSubscribe<T> source, BackpressureStrategy backpressure) {
        this.source = source;
        this.backpressure = backpressure;
    }
    public void subscribeActual(Subscriber<? super T> t) {
        BaseEmitter<T> emitter;
        switch (backpressure) {
            case MISSING: {
                emitter = new MissingEmitter<T>(t);
                break;
            }
            case ERROR: {
                emitter = new ErrorAsyncEmitter<T>(t);
                break;
            }
            case DROP: {
                emitter = new DropAsyncEmitter<T>(t);
                break;
            }
            case LATEST: {
                emitter = new LatestAsyncEmitter<T>(t);
                break;
            }
            default: {
                emitter = new BufferAsyncEmitter<T>(t, bufferSize());
                break;
            }
        }
        t.onSubscribe(emitter);
        try {
            source.subscribe(emitter);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            emitter.onError(ex);
        }
    }
    //...
}
```
* `MissingEmitter`
    ```
    static final class MissingEmitter<T> extends BaseEmitter<T> {
        MissingEmitter(Subscriber<? super T> downstream) {
            super(downstream);
        }
        @Override
        public void onNext(T t) {
            if (isCancelled()) {
                return;
            }
            if (t != null) {
                downstream.onNext(t);
            } else {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            for (;;) {
                long r = get();
                if (r == 0L || compareAndSet(r, r - 1)) {
                    return;
                }
            }
        }

    }
    ```
    通过上面的代码我们可以看到`MissingEmitter`基本上没做什么操作，所以`BackpressureStrategy.MISSING`示例中的代码实际上是调用了`ObserveOn`中返回对象的`FlowableObserveOn.ObserveOnSubscriber`的 `onNext`：
    ```
    public final void onNext(T t) {
        if (done) {
            return;
        }
        if (sourceMode == ASYNC) {
            trySchedule();
            return;
        }
        if (!queue.offer(t)) {
            upstream.cancel();
            error = new MissingBackpressureException("Queue is full?!");
            done = true;
        }
        trySchedule();
    }
    ```
    上面代码中我们看到了背压情况下出现的报错信息，出现的前提是`queue.offer(t)`返回`false`。这里的`queue`是`onSubscribe`中构造的容量为`Flowable.bufferSize()`的`SpscArrayQueue`.
    ```
    public void onSubscribe(Subscription s) {
        if (SubscriptionHelper.validate(this.upstream, s)) {
            this.upstream = s;
            //...
            queue = new SpscArrayQueue<T>(prefetch);
            downstream.onSubscribe(this);
            s.request(prefetch);
        }
    }
    ```
    `SpscArrayQueue`的`offer`方法，我们可以看到当`SpscArrayQueue`数据 “满了” 的时候即返回`false`.
    ```
    public boolean offer(E e) {
        //...
        final int mask = this.mask;
        final long index = producerIndex.get();
        final int offset = calcElementOffset(index, mask);
        if (index >= producerLookAhead) {
            int step = lookAheadStep;
            if (null == lvElement(calcElementOffset(index + step, mask))) { // LoadLoad
                producerLookAhead = index + step;
            } else if (null != lvElement(offset)) {
                return false;
            }
        }
        soElement(offset, e); // StoreStore
        soProducerIndex(index + 1); // ordered store -> atomic and ordered for size()
        return true;
    }
    ```
    所以`BackpressureStrategy.MISSING`在缓冲区满了之后再发射事件即会抛出 `message` 为 `"Queue is full?!"` 的 `MissingBackpressureException`.

---

* `ErrorAsyncEmitter`
    ```
    abstract static class BaseEmitter<T>
    extends AtomicLong
    implements FlowableEmitter<T>, Subscription {
        //...
        @Override
        public final void request(long n) {
            if (SubscriptionHelper.validate(n)) {
                BackpressureHelper.add(this, n);
                onRequested();
            }
        }
        //...
    }
    abstract static class NoOverflowBaseAsyncEmitter<T> extends BaseEmitter<T> {
        NoOverflowBaseAsyncEmitter(Subscriber<? super T> downstream) {
            super(downstream);
        }
        @Override
        public final void onNext(T t) {
            //...
            if (get() != 0) {
                downstream.onNext(t);
                BackpressureHelper.produced(this, 1);
            } else {
                onOverflow();
            }
        }
        abstract void onOverflow();
    }
    static final class ErrorAsyncEmitter<T> extends NoOverflowBaseAsyncEmitter<T> {
        ErrorAsyncEmitter(Subscriber<? super T> downstream) {
            super(downstream);
        }
        @Override
        void onOverflow() {
            onError(new MissingBackpressureException("create: could not emit value due to lack of requests"));
        }
    }
    ```
    通过在`onSubscribe`中调用`request(Flowable.bufferSize())`设置当前`AtomicLong`的`value`值。然后 `onNext` 中每传递一个事件就通过` BackpressureHelper.produced(this, 1)`将`value` 减 `1`. 当发送了`Flowable.bufferSize()`个事件，`get() != 0`不成立，调用`onOverflow()`方法抛出 `MissingBackpressureException`异常。

---

* `DropAsyncEmitter`
    ```
    static final class DropAsyncEmitter<T> extends NoOverflowBaseAsyncEmitter<T> {

        private static final long serialVersionUID = 8360058422307496563L;

        DropAsyncEmitter(Subscriber<? super T> downstream) {
            super(downstream);
        }

        @Override
        void onOverflow() {
            // nothing to do
        }

    }
    ```
    和 `ErrorAsyncEmitter` 类似，只不过当发送超过超过`Flowable.bufferSize()`的事件时，啥也没做，即实现丢弃的功能。

---

* `LatestAsyncEmitter`
    ```
    static final class LatestAsyncEmitter<T> extends BaseEmitter<T> {
        final AtomicReference<T> queue;
        //...
        LatestAsyncEmitter(Subscriber<? super T> downstream) {
            super(downstream);
            this.queue = new AtomicReference<T>();
            //...
        }

        @Override
        public void onNext(T t) {
            //...
            queue.set(t);
            drain();
        }
        //...
    }
    ```
    我们可以看到每次调用`onNext`都会更新传过来的值到`queue`中，所以`queue`中保存了最新的值。

    * 接着来看`drain`方法：
    上面我们知道在`onSubscribe`中调用`request()`设置当前`AtomicLong`的`value`值。
        ```
        void drain() {
            //...
            for (;;) {
                long r = get();
                long e = 0L;
                while (e != r) {
                    //...
                    boolean d = done;
                    T o = q.getAndSet(null);
                    boolean empty = o == null;
                    if (d && empty) {
                        //...
                        return;
                    }
                    if (empty) {
                        break;
                    }
                    a.onNext(o);
                    e++;
                }
                if (e == r) {
                    //...
                    boolean d = done;
                    boolean empty = q.get() == null;
                    if (d && empty) {
                        //...
                        return;
                    }
                }
                if (e != 0) {
                    BackpressureHelper.produced(this, e);
                }
                //...
            }
        }
        ```
        * 在`for (;;)`里面通过`get()`获取当前`AtomicLong`的值。然后通过`a.onNext(o);`传递给下游，然后`e++`，在通过`BackpressureHelper.produced(this, e);`减掉`AtomicLong`的值。
        * 当调用`Flowable.bufferSize()`次 `onNext`之后，`get()`返回的值为`0`，所以`e != r`不成立，在`e == r`的判断中，在从`onNext`过来时`empty`为`false`，所以直接跳出 `for`循环。
        * 通过上面我们知道当传递超过`Flowable.bufferSize()`的事件过来，只会更新`queue`中的值为最新的事件，其他啥也没做。**那最后一个事件时怎么发出的呢？**，继续往下看。
    * **最后一个事件时怎么发出的？**
       我们在上面的`drain()`中调用`a.onNext(o)`最终是调用`observeOn`构建对象中的`ObserveOnSubscriber`的`onNext`，即调用`runAsync();`。
        ```
        public final void onNext(T t) {
            //...
            trySchedule();
        }
        final void trySchedule() {
            //...
            worker.schedule(this);
        }
        @Override
        public final void run() {
            if (outputFused) {
                runBackfused();
            } else if (sourceMode == SYNC) {
                runSync();
            } else {
                runAsync();
            }
        }
        ```
    * **`runAsync()`**：
        ```
        void runAsync() {
            //...
            for (;;) {
                long r = requested.get();
                while (e != r) {
                    boolean d = done;
                    T v;
                    try {
                        v = q.poll();
                    } catch (Throwable ex) {
                        //...
                        return;
                    }
                    //...
                    a.onNext(v);
                    e++;
                    if (e == limit) {
                        if (r != Long.MAX_VALUE) {
                            r = requested.addAndGet(-e);
                        }
                        upstream.request(e);
                        e = 0L;
                    }
                }
                //...
            }
        }
        ```
      * 我们可以看到在`for`循环中通过`q.poll()`去获取缓存队列`SpscArrayQueue`中的事件。然后通过`a.onNext(v);`去执行我们示例代码中的耗时操作。
      * 然后当`e == limit`是，回去调用`LatestAsyncEmitter`的`request(e)`，而`limit`是在构造函数中初始化的，值为缓存队列容量`Flowable.bufferSize()`的 `3/4`。**所以当队列中的事件消耗了容量的`3/4`之后，会再去请求上游发送事件。**
        ```
        BaseObserveOnSubscriber(
                Worker worker,
                boolean delayError,
                int prefetch) {
            //...
            this.limit = prefetch - (prefetch >> 2);
        }
        ```

    * `request`方法：
        ```
        @Override
        public final void request(long n) {
            if (SubscriptionHelper.validate(n)) {
                System.out.println("n = " + n);
                BackpressureHelper.add(this, n);
                onRequested();
            }
        }
        @Override
        void onRequested() {
            drain();
        }
        ```
        即继续执行`drain()`方法，因为`queue`中还保存最新的值事件。所以会通过`a.onNext(o)`发送这个最新的事件。

* **如果在执行完等待队列`3/4`的事件之后，上游的事件还没发送结束，下游即会再次缓存上游发送过来的容量的`3/4`个事件。**
    示例代码：
    ```
    Flowable.create(new FlowableOnSubscribe<Object>() {
        @Override
        public void subscribe(FlowableEmitter<Object> emitter) throws Exception {
            for (int i = 0; i < Flowable.bufferSize() * 2; i++) {
                emitter.onNext(i);
            }
            Thread.sleep(10 * Flowable.bufferSize());
            for (int i = 0; i < Flowable.bufferSize() * 2; i++) {
                emitter.onNext(Flowable.bufferSize() * 2 + i);
            }
            emitter.onComplete();
        }
    }, BackpressureStrategy.LATEST)
            .subscribeOn(Schedulers.computation())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Subscriber<Object>() {
                @Override
                public void onSubscribe(Subscription s) {
                    s.request(Integer.MAX_VALUE);
                }
                @Override
                public void onNext(Object o) {
                    System.out.println("onNext: " + o);
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                }
                @Override
                public void onComplete() {
                    System.out.println("onComplete");
                }
            });
    ```
    输出结果：
    ```
    System.out: onNext: 0
    System.out: onNext: 1
    System.out: onNext: 2
    System.out: onNext: 3
    //....
    System.out: onNext: 125
    System.out: onNext: 126
    System.out: onNext: 127
    System.out: onNext: 255
    System.out: onNext: 256
    System.out: onNext: 257
    //...
    System.out: onNext: 349
    System.out: onNext: 350
    System.out: onNext: 511
    System.out: onComplete
    ```
    **可以看到输出结果中`255-350`即为容量`128` 的 `3/4`个元素。**

---

* `BufferAsyncEmitter`
    * 我们可以看到内部有一个`SpscLinkedArrayQueue`的缓存队列，每次调用`onNext`都会先保存到缓存队列，然后通过`drain()`方法一直去遍历当前的缓存队列。然后和`LatestAsyncEmitter`一样，当下游的缓存队列满了之后，即不再放下游发送事件，只是把上游的事件保存在`SpscLinkedArrayQueue`中，等待下游处理了容量的`3/4`的事件之后，上游在发送容量的`3/4`的事件过去。知道上游的事件消耗完，或者异常退出。即和`Observable`的效果类似，只不过缓存队列一个在上游一个在下游。
    ```
    static final class BufferAsyncEmitter<T> extends BaseEmitter<T> {
        final SpscLinkedArrayQueue<T> queue;
        //...
    
        BufferAsyncEmitter(Subscriber<? super T> actual, int capacityHint) {
            super(actual);
            this.queue = new SpscLinkedArrayQueue<T>(capacityHint);
            this.wip = new AtomicInteger();
        }
        @Override
        public void onNext(T t) {
            //...
            queue.offer(t);
            drain();
        }
        void drain() {
            //...
            final SpscLinkedArrayQueue<T> q = queue;
            for (;;) {
                long r = get();
                long e = 0L;
                while (e != r) {
                    //...
                    boolean d = done;
                    T o = q.poll();
                    boolean empty = o == null;
                    if (d && empty) {
                        //...
                        return;
                    }
                    if (empty) {
                        break;
                    }
                    a.onNext(o);
                    e++;
                }
                if (e == r) {
                    //...
                    boolean d = done;
                    boolean empty = q.isEmpty();
                    if (d && empty) {
                        //...
                        return;
                    }
                }
                if (e != 0) {
                    BackpressureHelper.produced(this, e);
                }
                missed = wip.addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }
    }
    ```

---

### 小结
* 我们知道了`Observable`出现背压的原因是上游发送的超多事件缓存在`observeOn`返回对象的缓存队列中，事件的增加导致了内存的增加。
* 我们介绍了`Flowable`的使用和五种背压策略的具体实现。
   * **MISSING**：超过`observeOn`配置的`bufferSize`则抛出异常`MissingBackpressureException`并提示`Queue is full?!`。
   * **ERROR**：超过`observeOn`配置的`bufferSize`则直接抛出异常`MissingBackpressureException`。
   * **BUFFER**：超过`observeOn`配置的`bufferSize`则缓存到上游的缓冲队列，等待下游消耗了容量的`3/4`的事件之后，在继续发送上游缓存的事件给下游。
   * **DROP**：超过`observeOn`配置的`bufferSize`则丢弃。
   * **LATEST**：超过`observeOn`配置的`bufferSize`则丢弃并保存最新的值到`queue`，如果在下游消耗了容量的`3/4`的事件之后，上游还有事件在发送，则继续往下游发送事件，当没有事件的时候，再发送`queue`中保存的最新的那个事件。

---

### 参考文章
* [Backpressure-(2.0)](https://github.com/ReactiveX/RxJava/wiki/Backpressure-(2.0))
* [关于 RxJava 背压](https://juejin.im/entry/58e704cbac502e4957b230eb)
* [Android RxJava ：图文详解 背压策略](https://www.jianshu.com/p/ceb48ed8719d)

---

以上
