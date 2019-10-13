>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

#### `from` 系列操作符
包括以下操作符：
  * `fromArray(T... items)` :  参数 **数组长度** 为 `0` 是执行 `empty`操作符，长度为 `1` 时，是执行 `just`操作符。
    ```
    public static <T> Observable<T> fromArray(T... items) {
        if (items.length == 0) {
            return empty();
        } else
        if (items.length == 1) {
            return just(items[0]);
        }
        return RxJavaPlugins.onAssembly(new ObservableFromArray<T>(items));
    }
    ```
  * `fromIterable(Iterable<? extends T> source)`
  * `fromCallable(Callable<? extends T> supplier)`
  * `fromFuture(Future<? extends T> future)`
  * `fromPublisher(Publisher<? extends T> publisher)`

**`from` 操作符实际上返回的是一个 `ObservableFromXXX`对象。**（ `XXX` 代表 `Array`,`Iterable`,`Callable`,`Future`,`Publisher`）

### `ObservableFromArray`源码：
* 首先构建 `FromArrayDisposable` 对象
* 然后调用观察者的 `onSubscribe(Disposable d)` 方法。
* `FromArrayDisposable` 的 `fusionMode` 默认为 `false`， 所以继续执行 `FromArrayDisposable`  的 `run` 方法。
* `FromArrayDisposable`  的 `run` 方法：依次传入参数数组中的值到 **观察者** 的 `onNext` 方法， 如果某个值为 `null`， 直接 `onError` 结束，否则遍历完之后，执行 `onComplete()`；
```
public final class ObservableFromArray<T> extends Observable<T> {
    final T[] array;
    public ObservableFromArray(T[] array) {
        this.array = array;
    }

    @Override
    public void subscribeActual(Observer<? super T> observer) {
        FromArrayDisposable<T> d = new FromArrayDisposable<T>(observer, array);
        observer.onSubscribe(d);
        if (d.fusionMode) {
            return;
        }
        d.run();
    }

    static final class FromArrayDisposable<T> extends BasicQueueDisposable<T> {
        final Observer<? super T> downstream;
        final T[] array;
        boolean fusionMode;
            ...
        FromArrayDisposable(Observer<? super T> actual, T[] array) {
            this.downstream = actual;
            this.array = array;
        }
            ...
        void run() {
            T[] a = array;
            int n = a.length;

            for (int i = 0; i < n && !isDisposed(); i++) {
                T value = a[i];
                if (value == null) {
                    downstream.onError(new NullPointerException("The element at index " + i + " is null"));
                    return;
                }
                downstream.onNext(value);
            }
            if (!isDisposed()) {
                downstream.onComplete();
            }
        }
    }
}
``` 

---

#### `ObservableFromIterable` 和 `ObservableFromArray`类似：
*  **长度为0** 直接调用 **观察者** 的 `onSubscribe(Disposable d)` 和 `onComplete()` 方法结束。
* 然后依次传入`Iterator` 中的值到 **观察者** 的 `onNext` 方法， 如果某个值为 `null`， 直接 `onError` 结束，否则遍历完之后，执行 `onComplete()`；
```
public void subscribeActual(Observer<? super T> observer) {
    Iterator<? extends T> it;
    try {
        it = source.iterator();
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        EmptyDisposable.error(e, observer);
        return;
    }
    boolean hasNext;
    try {
        hasNext = it.hasNext();
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        EmptyDisposable.error(e, observer);
        return;
    }
    if (!hasNext) {
        EmptyDisposable.complete(observer);
        return;
    }

    FromIterableDisposable<T> d = new FromIterableDisposable<T>(observer, it);
    observer.onSubscribe(d);

    if (!d.fusionMode) {
        d.run();
    }
}
```
```
void run() {
    boolean hasNext;
    do {
        ...
        T v;
        try {
            v = ObjectHelper.requireNonNull(it.next(), "The iterator returned a null value");
        } catch (Throwable e) {
            downstream.onError(e);
            return;
        }
        downstream.onNext(v);
        ...
        try {
            hasNext = it.hasNext();
        } catch (Throwable e) {
            downstream.onError(e);
            return;
        }
    } while (hasNext);

    if (!isDisposed()) {
        downstream.onComplete();
    }
}
```

---

####  `ObservableFromCallable` 相关源码：
* 首先构建 `DeferredScalarDisposable` 对象。
* 调用 **观察者** 的 `onSubscribe(Disposable d)` 方法。
* 获取 `Callable.call()` 的返回值，传给 `DeferredScalarDisposable` 对象的 `complete(T value)` 方法。
```
    @Override
    public void subscribeActual(Observer<? super T> observer) {
        DeferredScalarDisposable<T> d = new DeferredScalarDisposable<T>(observer);
        observer.onSubscribe(d);
        if (d.isDisposed()) {
            return;
        }
        T value;
        try {
            value = ObjectHelper.requireNonNull(callable.call(), "Callable returned null");
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            if (!d.isDisposed()) {
                observer.onError(e);
            } else {
                RxJavaPlugins.onError(e);
            }
            return;
        }
        d.complete(value);
    }
```
`DeferredScalarDisposable` 的 `complete(T value)` 方法
* `get()` 默认值为 `0`, 所以 `state == FUSED_EMPTY` 条件不成立，所以设置为 `TERMINATED`状态， 然后调用 **观察者** 的 `onNext(T value)` 方法。
* 然后再调用 **观察者** 的 `ononComplete()` 方法。
```
static final int TERMINATED = 2;
static final int DISPOSED = 4;
static final int FUSED_EMPTY = 8;
static final int FUSED_READY = 16;
static final int FUSED_CONSUMED = 32;

public final void complete(T value) {
    int state = get();
    if ((state & (FUSED_READY | FUSED_CONSUMED | TERMINATED | DISPOSED)) != 0) {
        return;
    }
    Observer<? super T> a = downstream;
    if (state == FUSED_EMPTY) {
        this.value = value;
        lazySet(FUSED_READY);
        a.onNext(null);
    } else {
        lazySet(TERMINATED);
        a.onNext(value);
    }
    if (get() != DISPOSED) {
        a.onComplete();
    }
}
```

---

#### `ObservableFromFuture`源码：
* 首先构建 `DeferredScalarDisposable` 对象。
* 调用 **观察者** 的 `onSubscribe(Disposable d)` 方法。
* 获取 `future.get(timeout, unit)` 这个 `future.get()` 的返回值，
传给 `DeferredScalarDisposable` 对象的 `complete(T value)` 方法。(和`ObservableFromCallable` 相同)
```
public void subscribeActual(Observer<? super T> observer) {
    DeferredScalarDisposable<T> d = new DeferredScalarDisposable<T>(observer);
    observer.onSubscribe(d);
    if (!d.isDisposed()) {
        T v;
        try {
            v = ObjectHelper.requireNonNull(unit != null ? future.get(timeout, unit) : future.get(), "Future returned null");
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            if (!d.isDisposed()) {
                observer.onError(ex);
            }
            return;
        }
        d.complete(v);
    }
}
```
RxJava提供的 `FutureObserver`类 的 `get` 方法：
```
public final class FutureObserver<T> extends CountDownLatch
        implements Observer<T>, Future<T>, Disposable {
        ...
    public T get() throws InterruptedException, ExecutionException {
        if (getCount() != 0) {
            BlockingHelper.verifyNonBlocking();
            await();
        }

        if (isCancelled()) {
            throw new CancellationException();
        }
        Throwable ex = error;
        if (ex != null) {
            throw new ExecutionException(ex);
        }
        return value;
    }

    @Override
    public T get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        if (getCount() != 0) {
            BlockingHelper.verifyNonBlocking();
            if (!await(timeout, unit)) {
                throw new TimeoutException(timeoutMessage(timeout, unit));
            }
        }

        if (isCancelled()) {
            throw new CancellationException();
        }

        Throwable ex = error;
        if (ex != null) {
            throw new ExecutionException(ex);
        }
        return value;
    }  
        ...
}
```

---

#### `ObservableFromPublisher`源码：
*  构建了一个 `PublisherSubscriber` 对象，重写了 **观察者** 的 `onSubscribe` 方法。
```
public final class ObservableFromPublisher<T> extends Observable<T> {

    final Publisher<? extends T> source;

    public ObservableFromPublisher(Publisher<? extends T> publisher) {
        this.source = publisher;
    }

    @Override
    protected void subscribeActual(final Observer<? super T> o) {
        source.subscribe(new PublisherSubscriber<T>(o));
    }

    static final class PublisherSubscriber<T>
            implements FlowableSubscriber<T>, Disposable {

        final Observer<? super T> downstream;
        Subscription upstream;

        PublisherSubscriber(Observer<? super T> o) {
            this.downstream = o;
        }
            ...
        @Override
        public void onSubscribe(Subscription s) {
            if (SubscriptionHelper.validate(this.upstream, s)) {
                this.upstream = s;
                downstream.onSubscribe(this);
                s.request(Long.MAX_VALUE);
            }
        }
            ...
    }
}
```
`SubscriptionHelper` 的 `validate(Subscription current, Subscription next)` 方法：
* `onSubscribe` 传参不能为 `null`。
* `upstream`默认为 `null`，所以第一次调用直接返回 `true`。
* 当第二次或者多次 调用 `onSubscribe` 方法时，`if (current != null)` 条件成立，直接调用 第二次传参的 `cancel` 方法。
  然后直接结束当前订阅流程。
```
public enum SubscriptionHelper implements Subscription {
        ...
    public static boolean validate(Subscription current, Subscription next) {
        if (next == null) {
            RxJavaPlugins.onError(new NullPointerException("next is null"));
            return false;
        }
        if (current != null) {
            next.cancel();
            reportSubscriptionSet();
            return false;
        }
        return true;
    }
        ...
}
```
以上
