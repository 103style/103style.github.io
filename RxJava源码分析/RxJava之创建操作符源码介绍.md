# RxJava之创建操作符源码介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

###  前言
前置阅读：[Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)

`RxJava` 之 **创建操作符** 官方介绍 ：[Creating-Observables](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables)

### 创建相关的操作符 以及 官方介绍
*   [`create`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#create)
*   [`defer`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#defer)
*   [`empty`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#empty)
*   [`error`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#error)
*   [`from`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#from)
*   [`generate`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#generate)
*   [`interval`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#interval)
*   [`just`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#just)
*   [`never`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#never)
*   [`range`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#range)
*   [`timer`](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables#timer)

前置阅读：[Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)，已经很详细的介绍了 `create`操作符，如果你还没有阅读过，请先阅读 [Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)。

### 创建相关的操作符源码介绍 `为了方便介绍，顺序有变化`

#### `create` 操作符
[Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)

#### `just` 操作符
* 官方提供的使用例子：
    ```
    String greeting = "Hello world!";
    Observable<String> observable = Observable.just(greeting);
    observable.subscribe(item -> System.out.println(item));
    ```
* `just`操作符实际上返回的是一个 `ObservableJust`对象，`just` 操作符也提供了2-10个参数的，不过内部是调用了`fromArray` 操作符。
    ```
    public static <T> Observable<T> just(T item) {
        return RxJavaPlugins.onAssembly(new ObservableJust<T>(item));
    }

    public static <T> Observable<T> just(T item1, T item2) {
        return fromArray(item1, item2);
    }
    ...
    ```
 * `ObservableJust`源码：
    ```
    public final class ObservableJust<T> extends Observable<T> implements ScalarCallable<T> {
    
        private final T value;
        public ObservableJust(final T value) {
            this.value = value;
        }
    
        @Override
        protected void subscribeActual(Observer<? super T> observer) {
            ScalarDisposable<T> sd = new ScalarDisposable<T>(observer, value);
            observer.onSubscribe(sd);
            sd.run();
        }
    
        @Override
        public T call() {
            return value;
        }
    }
    ```
    主要看`subscribeActual`方法，我们看到，
    * 首先构建了一个`ScalarDisposable`对象，
    * 然后调用 **观察者** 的 `onSubscribe`方法，
    * 接着执行 `ScalarDisposable` 的 `run`方法。

* `ScalarDisposable` 相关源码：
    ```
    public static final class ScalarDisposable<T>
    extends AtomicInteger
    implements QueueDisposable<T>, Runnable {
        static final int START = 0;
        static final int FUSED = 1;
        static final int ON_NEXT = 2;
        static final int ON_COMPLETE = 3;
      ...
        @Override
        public void run() {
            if (get() == START && compareAndSet(START, ON_NEXT)) {
                observer.onNext(value);
                if (get() == ON_NEXT) {
                    lazySet(ON_COMPLETE);
                    observer.onComplete();
                }
            }
        }
    }
    ```
    `run`方法里面我们又看到了之前 [Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11) 里介绍的 `get() `和`compareAndSet(int var1, int var2)`方法。
     * 因为是`AtomicInteger`，所以 `get()` 获取到的 `value` 值为 `int` 类型的默认值 `0`，`0`则为 `ObservableJust` 的 `START` 状态。
     * `compareAndSet(int var1, int var2)` 是比较 `var1` 和 `var2` 的值，如果不相等，则更新 `var1` 的值为` var2`，并返回`true`。 
    所以 `ObservableJust` 由 `START` 进入`ON_NEXT ` 状态。
     * `run()`方法的 `if` 条件 `get() == START && compareAndSet(START, ON_NEXT)` 成立。
     所以将 `just`操作符传进来的`value`值 传入 **观察者** 的 `onNext` 方法
     * 因为此时为`onNext`状态，所以 `get() == ON_NEXT` 成立，
    修改 `ObservableJust` 的状态为 `onComplete`， 继续执行 **观察者** 的 `onComplete` 方法。

* 当然我们也可以在 **观察者** 的 `onSubscribe`方法，调用 `dispose()` 修改 `ObservableJust` 的状态。
    ```
    public static final class ScalarDisposable<T>
    extends AtomicInteger
    implements QueueDisposable<T>, Runnable {
        ...        
        @Override
        public void dispose() {
            set(ON_COMPLETE);
        }
    }
    ```

---

#### `defer` 操作符
* 官方提供的使用例子：
    ```
    Observable<Long> observable = Observable.defer(() -> {
        long time = System.currentTimeMillis();
        return Observable.just(time);
    });

    observable.subscribe(time -> System.out.println(time));

    Thread.sleep(1000);

    observable.subscribe(time -> System.out.println(time));
    ```

* `defer`操作符实际上返回的是一个 `ObservableDefer`对象。
    ```
    public static <T> Observable<T> defer(Callable<? extends ObservableSource<? extends T>> supplier) {
        ObjectHelper.requireNonNull(supplier, "supplier is null");
        return RxJavaPlugins.onAssembly(new ObservableDefer<T>(supplier));
    }
    ```

* `ObservableDefer` 源码：
    ```
    public final class ObservableDefer<T> extends Observable<T> {
        final Callable<? extends ObservableSource<? extends T>> supplier;

        public ObservableDefer(Callable<? extends ObservableSource<? extends T>> supplier) {
            this.supplier = supplier;
        }

        @Override
        public void subscribeActual(Observer<? super T> observer) {
            ObservableSource<? extends T> pub;
            try {
                pub = ObjectHelper.requireNonNull(supplier.call(), "null ObservableSource supplied");
            } catch (Throwable t) {
                Exceptions.throwIfFatal(t);
                EmptyDisposable.error(t, observer);
                return;
            }
            pub.subscribe(observer);
        }
    }
    ```
* `defer`操作符 实际上是将 `call()` 方法返回的 `ObservableSource` 和  `subscribe` 中的 **观察者** 建立订阅关系。
    ```
    Observable.defer(new Callable<ObservableSource<Object>>() {
        @Override
        public ObservableSource<Object> call() throws Exception {
            return null;
        }
    }).subscribe(new Observer<Object>() {...});
    ```
* 官方的例子中 第一次调用 `subscribe` 之后，延迟一秒之后再次调用 `subscribe`，
   相当与调用 `call()` 之后，延迟一秒之后再次调用 `call()`，即执行两次`Observable.just(当前毫秒数)`。
   看了上面的 `just` 操作符介绍，我们可以得知，会执行两次 `subscribe`传进来的 `Observer` 的 `onNext(Object o)`方法。
  
----

#### `empty` 操作符
* `empty`操作符实际上返回的是一个 `ObservableEmpty`对象。
    ```
    public static <T> Observable<T> empty() {
        return RxJavaPlugins.onAssembly((Observable<T>) ObservableEmpty.INSTANCE);
    }
    ```
* `ObservableEmpty`源码：
    ```
    public final class ObservableEmpty extends Observable<Object> implements ScalarCallable<Object> {
        public static final Observable<Object> INSTANCE = new ObservableEmpty();
    
        private ObservableEmpty() {
        }
    
        @Override
        protected void subscribeActual(Observer<? super Object> o) {
            EmptyDisposable.complete(o);
        }
    
        @Override
        public Object call() {
            return null; 
        }
    }
    ```

* `EmptyDisposable` 的  `complete(Observer<?> observer)` 方法
    ```
    public enum EmptyDisposable implements QueueDisposable<Object> {
        ...
        public static void complete(Observer<?> observer) {
            observer.onSubscribe(INSTANCE);
            observer.onComplete();
        }
        ...
    }
    ```

* 所以我们看到 `empty` 操作符实际上，在订阅之后直接执行了 **观察者** 的 `onSubscribe(Disposable d)` 和 `onComplete()` 方法。


---
 
#### `error` 操作符
* `error` 操作符实际上返回的是一个 `ObservableError`对象。
    和 `empty` 操作符类似，在订阅之后直接执行了 **观察者** 的 `onSubscribe(Disposable d)` 和 `onError(Throwable e)` 方法。
    ```
    public static <T> Observable<T> error(Callable<? extends Throwable> errorSupplier) {
        ObjectHelper.requireNonNull(errorSupplier, "errorSupplier is null");
        return RxJavaPlugins.onAssembly(new ObservableError<T>(errorSupplier));
    }

    public final class ObservableError<T> extends Observable<T> {
        final Callable<? extends Throwable> errorSupplier;
        public ObservableError(Callable<? extends Throwable> errorSupplier) {
            this.errorSupplier = errorSupplier;
        }
    
        @Override
        public void subscribeActual(Observer<? super T> observer) {
            ...
            EmptyDisposable.error(error, observer);
        }
    }

    public enum EmptyDisposable implements QueueDisposable<Object> {
        ...
        public static void error(Throwable e, Observer<?> observer) {
            observer.onSubscribe(INSTANCE);
            observer.onError(e);
        }
        ...
    }
    ```

---
 
#### `from` 操作符系列
[Rxjava之from系列操作符源码解析](https://www.jianshu.com/p/de1b62568098)

---
 
####  `generate` 操作符
* `generate` 操作符实际上返回的是一个 `ObservableGenerate`对象。
  * 首先获取 `Callable` 对象 `stateSupplier` 的 `call` 方法的返回值。
  * 构建 `GeneratorDisposable` 对象。
  * 然后调用 **观察者** 的  `onSubscribe(Disposable d)`。
  * 执行`GeneratorDisposable` 对象的 `run` 方法。
    ```
    public final class ObservableGenerate<T, S> extends Observable<T> {
        final Callable<S> stateSupplier;
        final BiFunction<S, Emitter<T>, S> generator;
        final Consumer<? super S> disposeState;
    
        public ObservableGenerate(Callable<S> stateSupplier, BiFunction<S, Emitter<T>, S> generator,
                Consumer<? super S> disposeState) {
            this.stateSupplier = stateSupplier;
            this.generator = generator;
            this.disposeState = disposeState;
        }
    
        @Override
        public void subscribeActual(Observer<? super T> observer) {
            S state;
            try {
                state = stateSupplier.call();
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                EmptyDisposable.error(e, observer);
                return;
            }
            GeneratorDisposable<T, S> gd = new GeneratorDisposable<T, S>(observer, generator, disposeState, state);
            observer.onSubscribe(gd);
            gd.run();
        }
        ...
    }
    ```
    `GeneratorDisposable` 源码：
    * `run` 方法里面无限循环执行 `generate`操作符 传进来的 `Consumer`对象的 `accept` 方法。
       直到在 `accept` 中调用 `onComplete()` 或者 `onError`方法 
    * 在`Consumer`对象的 `accept` 方法执行 `onNext` 则会调用观察者的 `onNext` 。
    ```
    static final class GeneratorDisposable<T, S>
    implements Emitter<T>, Disposable {
        final Observer<? super T> downstream;
        final BiFunction<S, ? super Emitter<T>, S> generator;
        final Consumer<? super S> disposeState;
        S state;
        volatile boolean cancelled;
        boolean terminate;
        boolean hasNext;

        GeneratorDisposable(Observer<? super T> actual,
                BiFunction<S, ? super Emitter<T>, S> generator,
                Consumer<? super S> disposeState, S initialState) {
            this.downstream = actual;
            this.generator = generator;
            this.disposeState = disposeState;
            this.state = initialState;
        }

        public void run() {
            S s = state;
            if (cancelled) {
                state = null;
                dispose(s);
                return;
            }
            final BiFunction<S, ? super Emitter<T>, S> f = generator;
            for (;;) {
                if (cancelled) {
                    state = null;
                    dispose(s);
                    return;
                }
                hasNext = false;
                try {
                    s = f.apply(s, this);
                } catch (Throwable ex) {
                    ...
                    onError(ex);
                    dispose(s);
                    return;
                }
                if (terminate) {
                    cancelled = true;
                    state = null;
                    dispose(s);
                    return;
                }
            }
        }
        ...
    }
    ```
    单参数默认构建的 `SimpleGenerator`:
    ```
    static final class SimpleGenerator<T, S> implements BiFunction<S, Emitter<T>, S> {
        final Consumer<Emitter<T>> consumer;

        SimpleGenerator(Consumer<Emitter<T>> consumer) {
            this.consumer = consumer;
        }

        @Override
        public S apply(S t1, Emitter<T> t2) throws Exception {
            consumer.accept(t2);
            return t1;
        }
    }
    ```

---
 
#### `never` 操作符
`never` 操作符实际上返回的是一个 `ObservableNever`对象。
只执行 **观察者** 的 `onSubscribe`方法。
```
public final class ObservableNever extends Observable<Object> {
    public static final Observable<Object> INSTANCE = new ObservableNever();

    private ObservableNever() {
    }

    @Override
    protected void subscribeActual(Observer<? super Object> o) {
        o.onSubscribe(EmptyDisposable.NEVER);
    }
}
```
---
 
#### `range` 操作符  `range(final int start, final int count)`
`count` 参数为 0 时执行 `empty` 操作符，`count` 为 1 时执行  `just` 操作符。

* `range` 操作符实际上返回的是一个 `ObservableRange`对象。
  * 首先初始化了 `起始点` 和  `结束点` 的值。
  * 接着构建 `RangeDisposable` 对象。
  * 接着调用 **观察者** 的 `onSubscribe(Disposable d)`方法。
  * 然后执行 `RangeDisposable` 对象的 `run` 方法。
   ```
    public final class ObservableRange extends Observable<Integer> {
        private final int start;
        private final long end;
    
        public ObservableRange(int start, int count) {
            this.start = start;
            this.end = (long)start + count;
        }
    
        @Override
        protected void subscribeActual(Observer<? super Integer> o) {
            RangeDisposable parent = new RangeDisposable(o, start, end);
            o.onSubscribe(parent);
            parent.run();
        }
    }
   ```
   `RangeDisposable` 对象的 `run`方法：
    * 从 `起始点` 到 `结束点` 的 循环遍历，状态对的话 则调用 **观察者** 的 `onNext` 方法。
    ```
    static final class RangeDisposable
    extends BasicIntQueueDisposable<Integer> {
        final Observer<? super Integer> downstream;
        final long end;
        long index;
        boolean fused;

        RangeDisposable(Observer<? super Integer> actual, long start, long end) {
            this.downstream = actual;
            this.index = start;
            this.end = end;
        }

        void run() {
            if (fused) {
                return;
            }
            Observer<? super Integer> actual = this.downstream;
            long e = end;
            for (long i = index; i != e && get() == 0; i++) {
                actual.onNext((int)i);
            }
            if (get() == 0) {
                lazySet(1);
                actual.onComplete();
            }
        }
        ...
    ```

---
 
#### `interval` 和 `timer` 操作符
[Rxjava之timer和interval操作符源码解析](https://www.jianshu.com/p/316966e3841a)

以上
