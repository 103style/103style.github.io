>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


### 转换相关的操作符 以及 官方介绍
`RxJava` 之 **转换操作符** 官方介绍 ：[Transforming Observables](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables)
*   [`buffer`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#buffer)
*   [`cast`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#cast)
*   [`concatMap`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmap)
*   [`concatMapXXX`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmap)
*   [`flatMap`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmap)
*   [`flatMapXXX`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmap)
*   [`flattenAsFlowable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flattenasflowable)
*   [`flattenAsObservable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flattenasobservable)
*   [`groupBy`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#groupby)
*   [`map`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#map)
*   [`scan`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#scan)
*   [`switchMap`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#switchmap)
*   [`window`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#window)


以下介绍我们就直接具体实现，中间流程请参考 [RxJava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)。

#### buffer
* 官方示例：
    ```
    Observable.range(0, 10)
        .buffer(4)
        .subscribe((List<Integer> buffer) -> System.out.println(buffer));
    ```
    输出：
    ```
    [0, 1, 2, 3]
    [4, 5, 6, 7]
    [8, 9]
    ```
* 返回对象的 `ObservableBuffer` 的 `subscribeActual` 方法：
    单参数`buffer`的`skip` 和 `count` 是相等的。
    ```
    protected void subscribeActual(Observer<? super U> t) {
        if (skip == count) {
            BufferExactObserver<T, U> bes = new BufferExactObserver<T, U>(t, count, bufferSupplier);
            if (bes.createBuffer()) {//1.0
                source.subscribe(bes);
            }
        } else {
            source.subscribe(new BufferSkipObserver<T, U>(t, count, skip, bufferSupplier));
        }
    }
    ```
    * `(1.0)`  `createBuffer`即新创建了一个`ArrayList`对象 `buffer`。
* `onNext(T t)`和`onComplete()`方法：
    ```
    public void onNext(T t) {
        U b = buffer;
        if (b != null) {
            b.add(t);
            if (++size >= count) {//1.0
                downstream.onNext(b);
                size = 0;
                createBuffer();
            }
        }
    }

    public void onComplete() {
        U b = buffer;
        if (b != null) {//2.0
            buffer = null;
            if (!b.isEmpty()) {
                downstream.onNext(b);
            }
            downstream.onComplete();
        }
    }
    ```
  * `(1.0)`  每次调用`onNext` 就检查缓存的事件数是否 **不小于** `buffer`操作符设置的 值，成立则将缓存的 `buffer` 数组 传给观察者的 `onNext`。
  * `(2.0)`  `onComplete` 是检查缓存的事件数是否不为空，成立则将缓存的 `buffer` 数组 传给观察者的 `onNext`，再调用观察者的 `onComplete`。

---

#### cast
* 官方示例：
    ```
    Observable<Number> numbers = Observable.just(1, 4.0, 3f, 7, 12, 4.6, 5);
    numbers.filter((Number x) -> x instanceof Integer)
            .cast(Integer.class)
            .subscribe((Integer x) -> System.out.println(x));
    ```
    输出：
    ```
    1
    7
    12
    5
    ```
 * `cast` 是通过`map`操作符来实现的，我们直接看`map`。
    ```
    public final <U> Observable<U> cast(final Class<U> clazz) {
        ObjectHelper.requireNonNull(clazz, "clazz is null");
        return map(Functions.castFunction(clazz));
    }
    ```
    *  `apply`方法：
        ```
        public U apply(T t) throws Exception {
            return clazz.cast(t);
        } 
        ```

---

#### map
* 官方示例：
    ```
    Observable.just(1, 2, 3)
            .map(x -> x * x)
            .subscribe(System.out::println);
    ```
    输出：
    ```
    1
    4
    9
    ```
* 返回对象的 `ObservableMap` 的 `subscribeActual` 方法：
    ```
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
    ```
* 继续看 `MapObserver` 的 `onNext(T t)`：
    ```
    public void onNext(T t) {
        if (done) {
            return;
        }
        if (sourceMode != NONE) {
            downstream.onNext(null);
            return;
        }
        U v;
        try {
            v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
        } catch (Throwable ex) {
            fail(ex);
            return;
        }
        downstream.onNext(v);
    }
    ```
    * 即将 `map` 操作符 传入`Function`对象的 **返回值**  传递给 链式调用上一步的返回对象的 `onNext(T t)`。

---

#### concatMap
[RxJava之concatMap系列转换操作符介绍](https://www.jianshu.com/p/db409f7fdd47)


---

#### flatMap
[RxJava之flatMap系列转换操作符介绍](https://www.jianshu.com/p/191a8680b8cc)

---

#### flattenAsFlowable
* 官方示例：
    ```
    Single<Double> source = Single.just(2.0);
    Flowable<Double> flowable = source.flattenAsFlowable(x -> {
        return Arrays.asList(x, Math.pow(x, 2), Math.pow(x, 3));
    });
    flowable.subscribe(x -> System.out.println("onNext: " + x));
    ```
    输出：
    ```
    onNext: 2.0
    onNext: 4.0
    onNext: 8.0
    ```
* 我们先看` Single.just(2.0)`：
    ```
    public static <T> Single<T> just(final T item) {
        ObjectHelper.requireNonNull(item, "value is null");
        return RxJavaPlugins.onAssembly(new SingleJust<T>(item));
    }
    ```
* ` SingleJust`的`subscribeActual`：
    ```
    protected void subscribeActual(SingleObserver<? super T> observer) {
        observer.onSubscribe(Disposables.disposed());
        observer.onSuccess(value);
    }
    ```
* `flattenAsFlowable`返回对象的 `SingleFlatMapIterableFlowable` 的 `subscribeActual` 方法：
    ```
    protected void subscribeActual(Subscriber<? super R> s) {
        source.subscribe(new FlatMapIterableObserver<T, R>(s, mapper));
    }
    ```
* 继续看 `FlatMapIterableObserver` 的`onSubscribe(Disposable d)` 和 `onSuccess(T value)`：
    ```
    public void onSubscribe(Disposable d) {
        if (DisposableHelper.validate(this.upstream, d)) {
            this.upstream = d;
            downstream.onSubscribe(this);//1.0
        }
    }
    public void onSuccess(T value) {
        Iterator<? extends R> iterator;
        boolean has;
        try {
            iterator = mapper.apply(value).iterator();//2.0 调用apply返回的Iterable对象的 iterator()方法。
            has = iterator.hasNext();//3.0
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            downstream.onError(ex);
            return;
        }
        if (!has) {
            downstream.onComplete();//3.1
            return;
        }
        this.it = iterator;//3.2
        drain();
    }
    ```
    * `(1.0)` 通过[Flowable subscribe流程介绍](https://www.jianshu.com/p/0aa24c3ccb4f) 我们知道` downstream.onSubscribe(this)`即调用 `FlowableInternalHelper.RequestMax.INSTANCE` 的 `accept`方法：
        ```
        public enum RequestMax implements Consumer<Subscription> {
            INSTANCE;
            @Override
            public void accept(Subscription t) throws Exception {
                t.request(Long.MAX_VALUE);
            }
        }
        ```
        即：`FlatMapIterableObserver.request(Long.MAX_VALUE)`:  即为设置变量`requested`的`value`为`Long.MAX_VALUE`。`drain()`因为`it` 变量还是`null`，所以没做什么操作。
        ```
        public void request(long n) {
            if (SubscriptionHelper.validate(n)) {
                BackpressureHelper.add(requested, n);
                drain();
            }
        }
        ```
    * `(2.0)` 调用`flattenAsFlowable`传入的`Function`的`apply`返回的`Iterable`对象的 `iterator()`方法。
    * `(3.0)` 检查`Iterable`时候为空，`(3.1)` 为空直接`onComplete()`，`(3.2)` 不为空则将 `iterator()`返回值赋值给当前的 `it` 变量，继续执行`drain()`。
* `drain()`:
    ```
    void drain() {
        ...
        Subscriber<? super R> a = downstream;
        Iterator<? extends R> iterator = this.it;
        ...
        int missed = 1;
        for (; ; ) {
            if (iterator != null) {
                long r = requested.get();
                long e = 0L;
                if (r == Long.MAX_VALUE) {//1.0
                    slowPath(a, iterator);
                    return;
                }
                ...
            }
            ...
        }
    }
    ```
    * 因为上一步`downstream.onSubscribe(this)`调用了`request(Long.MAX_VALUE)`, 所以  `(1.0)` 这里条件成立，执行`slowPath(downstream iterator)`。
* `slowPath(downstream iterator)`：
    ```
    void slowPath(Subscriber<? super R> a, Iterator<? extends R> iterator) {
        for (;;) {
            if (cancelled) {
                return;
            }
            R v;
            try {
                v = iterator.next();//1.0
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                a.onError(ex);
                return;
            }
            a.onNext(v);//1.1
            if (cancelled) {
                return;
            }
            boolean b;
            try {
                b = iterator.hasNext();//1.2
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                a.onError(ex);
                return;
            }
            if (!b) {
                a.onComplete();//1.3
                return;
            }
        }
    }
    ```
    * 将`(1.0)` 获取到的元素，`(1.1)`传递给`downstream ` 的 `onNext`，`(1.2)`然后判断是否还有其他元素，如果有则循环继续，没有的话即调用 `downstream ` 的 `onComplete` 结束。

---

#### flattenAsObservable
* 官方示例：
    ```
    Single<Double> source = Single.just(2.0);
    Observable<Double> observable = source.flattenAsObservable(x -> {
        return Arrays.asList(x, Math.pow(x, 2), Math.pow(x, 3));
    });
    observable.subscribe(x -> System.out.println("onNext: " + x));
    ```
    输出：
    ```
    onNext: 2.0
    onNext: 4.0
    onNext: 8.0
    ```

`subscribeActual`实现的逻辑和 `flattenAsFlowable` 类似，只是返回的对象为 `SingleFlatMapIterableObservable`，就不再赘述了。

---

#### groupBy
* 官方示例：
    ```
    Observable<String> animals = Observable.just(
        "Tiger", "Elephant", "Cat", "Chameleon", "Frog", "Fish", "Turtle", "Flamingo");
    animals.groupBy(animal -> animal.charAt(0), String::toUpperCase)
        .concatMapSingle(Observable::toList)
        .subscribe(System.out::println);
    ```
    输出：
    ```
    [TIGER, TURTLE]
    [ELEPHANT]
    [CAT, CHAMELEON]
    [FROG, FISH, FLAMINGO]
    ```
    
* 我们来看看返回对象的`ObservableGroupBy`:

    ```
    public GroupByObserver(Observer<? super GroupedObservable<K, V>> actual, 
                           Function<? super T, ? extends K> keySelector, 
                           Function<? super T, ? extends V> valueSelector, 
                           int bufferSize, boolean delayError) {
        this.downstream = actual;
        this.keySelector = keySelector;
        this.valueSelector = valueSelector;
        this.bufferSize = bufferSize;
        this.delayError = delayError;
        this.groups = new ConcurrentHashMap<Object, GroupedUnicast<K, V>>();
        this.lazySet(1);
    }

    public void subscribeActual(Observer<? super GroupedObservable<K, V>> t) {
        source.subscribe(new GroupByObserver<T, K, V>(t, keySelector, valueSelector, bufferSize, delayError));
    } 
    ```
* 继续看 `GroupByObserver` 的 `onNext(T t)`：
    ```
    public void onNext(T t) {
        K key;
        try {
            key = keySelector.apply(t);//1.0
        } catch (Throwable e) {
            ...
            return;
        }
        Object mapKey = key != null ? key : NULL_KEY;
        GroupedUnicast<K, V> group = groups.get(mapKey);
        if (group == null) {
            if (cancelled.get()) {
                return;
            }
            group = GroupedUnicast.createWith(key, bufferSize, this, delayError);
            groups.put(mapKey, group);//2.0
            getAndIncrement();
            downstream.onNext(group);//3.0
        }
    
        V v;
        try {
            v = ObjectHelper.requireNonNull(valueSelector.apply(t), "The value supplied is null");//4.0
        } catch (Throwable e) {
            ...
            return;
        }
        group.onNext(v);//4.1
    }
    ```
    * `(1.0)` 通过`keySelector.apply(t)`即官方示例中的 `animal.charAt(0)`获取分组的 `key`。
    * `(2.0)` 如果`GroupedUnicast`不存再这个`key`，则保存进去。
    * `(3.0)` 然后继续调用上一步操作符的 `onNext`方法，即官方示例中的`just`。
    * `(4.0)` 通过`valueSelector.apply(t)`即官方示例中的 `String::toUpperCase)`获取值，`(4.1)`添加到`ToListObserver`的 `collection`中。
* 最后通过 `onComplete()`输出：
    ```
    public void onComplete() {
        List<ObservableGroupBy.GroupedUnicast<K, V>> list = new ArrayList<ObservableGroupBy.GroupedUnicast<K, V>>(groups.values());
        groups.clear();
        for (ObservableGroupBy.GroupedUnicast<K, V> e : list) {
            e.onComplete();
        }
        downstream.onComplete();
    }
    ```
    `ToListObserver`的`onComplete()`:
    ```
    public void onComplete() {
        U c = collection;
        collection = null;
        downstream.onNext(c);
        downstream.onComplete();
    }
    ```

---

#### scan
* 官方示例：
    ```
    Observable.just(5, 3, 8, 1, 7)
            .scan(0, (partialSum, x) -> partialSum + x)
            .subscribe(System.out::println);
    ```
    输出：
    ```
    0
    5
    8
    16
    17
    24
    ```
* 我们来看看返回对象的`ObservableScanSeed`:
    ```
    public ObservableScanSeed(ObservableSource<T> source, Callable<R> seedSupplier, BiFunction<R, ? super T, R> accumulator) {
        super(source);
        this.accumulator = accumulator;
        this.seedSupplier = seedSupplier;
    }

    @Override
    public void subscribeActual(Observer<? super R> t) {
        R r;
        try {
            r = ObjectHelper.requireNonNull(seedSupplier.call(), "The seed supplied is null");//1.0
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            EmptyDisposable.error(e, t);
            return;
        }
        source.subscribe(new ScanSeedObserver<T, R>(t, accumulator, r));
    }
    ```
    * `(1.0)` `seedSupplier.call()`即官方示例中的 `0`，即设置 `r` 的值为`0`.
* 继续看`ScanSeedObserver`的`onNext(T t)`:
    ```
    public void onNext(T t) {
        if (done) {
            return;
        }
        R v = value;//1.0
        R u;
        try {
            u = ObjectHelper.requireNonNull(accumulator.apply(v, t), "The accumulator returned a null value");//2.0
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            upstream.dispose();
            onError(e);
            return;
        }
        value = u;//3.0
        downstream.onNext(u);//4.0
    }
    ```
    * `(1.0)` `value` 即为上一步设置的 `r` 值 `0`. 
    * `(2.0)` `accumulator.apply(v, t)` 即为官方示例中的 `partialSum + x`
    * `(3.0)` 更新`value` 的值
    * `(4.0)` 将`accumulator.apply(v, t)`传递给观察者的`onNext`

---

#### switchMap
* 官方示例：
    ```
    Observable.interval(0, 1, TimeUnit.SECONDS)
            .switchMap(x -> {
                return Observable.interval(0, 750, TimeUnit.MILLISECONDS)
                        .map(y -> x);
            })
            .takeWhile(x -> x < 3)
            .blockingSubscribe(System.out::print);

    ```
    输出：
    ```
    001122
    ```
* 我们来看看返回对象的`ObservableSwitchMap`:
    ```
    public ObservableSwitchMap(ObservableSource<T> source,
                               Function<? super T, ? extends ObservableSource<? extends R>> mapper, int bufferSize,
                                       boolean delayErrors) {
        super(source);
        this.mapper = mapper;
        this.bufferSize = bufferSize;
        this.delayErrors = delayErrors;
    }

    @Override
    public void subscribeActual(Observer<? super R> t) {
        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }
        source.subscribe(new SwitchMapObserver<T, R>(t, mapper, bufferSize, delayErrors));
    }
    ```
* 继续看`SwitchMapObserver`的`onNext`:
    ```
    public void onNext(T t) {
        long c = unique + 1;
        unique = c;
        SwitchMapInnerObserver<T, R> inner = active.get();
        if (inner != null) {
            inner.cancel();
        }
        ObservableSource<? extends R> p;
        try {
            p = ObjectHelper.requireNonNull(mapper.apply(t), "The ObservableSource returned is null");//1.0
        } catch (Throwable e) {
            ...
            return;
        }
        SwitchMapInnerObserver<T, R> nextInner = new SwitchMapInnerObserver<T, R>(this, c, bufferSize);//2.0
        for (;;) {
            inner = active.get();
            if (inner == CANCELLED) {
                break;
            }
            if (active.compareAndSet(inner, nextInner)) {
                p.subscribe(nextInner);//3.0
                break;
            }
        }
    }
    ```
    * `(1.0)` 通过`mapper.apply(t)`即官方示例中的 `Observable.interval(0, 750, TimeUnit.MILLISECONDS).map(y -> x)`返回的`ObservableMap`对象。
    * `(2.0)` 构建`SwitchMapInnerObserver`对象
    * `(3.0)` 用返回的`ObservableMap`订阅`SwitchMapInnerObserver`对象

---

#### window
* 官方示例：
    ```
    Observable.range(1, 10)
            // Create windows containing at most 2 items, and skip 3 items before starting a new window.
            .window(2, 3)
            .flatMapSingle(window -> {
                return window.map(String::valueOf)
                        .reduce(new StringJoiner(", ", "[", "]"), StringJoiner::add);
            })
            .subscribe(System.out::println);
    ```
    输出：
    ```
    [1, 2]
    [4, 5]
    [7, 8]
    [10]
    ```
* 我们来看看返回对象的`ObservableWindow`:
    ```
    public ObservableWindow(ObservableSource<T> source, long count, long skip, int capacityHint) {
        super(source);
        this.count = count;
        this.skip = skip;
        this.capacityHint = capacityHint;
    }

    @Override
    public void subscribeActual(Observer<? super Observable<T>> t) {
        if (count == skip) {
            source.subscribe(new WindowExactObserver<T>(t, count, capacityHint));
        } else {
            source.subscribe(new WindowSkipObserver<T>(t, count, skip, capacityHint));
        }
    }
    ```
* 继续看`WindowSkipObserver`的`onNext`:
    ```
    public void onNext(T t) {
        final ArrayDeque<UnicastSubject<T>> ws = windows;
        long i = index;
        long s = skip;
        if (i % s == 0 && !cancelled) {//3.0
            wip.getAndIncrement();
            UnicastSubject<T> w = UnicastSubject.create(capacityHint, this);
            ws.offer(w);
            downstream.onNext(w);
        }
        long c = firstEmission + 1;
        for (UnicastSubject<T> w : ws) {
            w.onNext(t);//1.0
        }
        if (c >= count) {
            ws.poll().onComplete();//2.0
            if (ws.isEmpty() && cancelled) {
                this.upstream.dispose();
                return;
            }
            firstEmission = c - s;
        } else {
            firstEmission = c;
        }
        index = i + 1;
    }
    ```
    * `(1.0)` 将元素存入 `queue`
    * `(2.0)` 当元素个数到达`count`时，就之前的元素全部输出
    * `(3.0)` 当元素个数到达`skip`时，就重新创建一个`UnicastSubject`来存储元素


以上
