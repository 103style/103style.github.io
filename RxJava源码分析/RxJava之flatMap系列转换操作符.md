# RxJava之flatMap系列转换操作符 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


### 转换相关的操作符 以及 官方介绍
`RxJava` 之 `flatMap` 系列 **转换操作符** 官方介绍 ：[Transforming Observables](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatMap)
*   [`flatMap`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmap)
*   [`flatMapCompletable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmapcompletable)
*   [`flatMapIterable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmapiterable)
*   [`flatMapMaybe`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmapmaybe)
*   [`flatMapObservable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmapobservable)
*   [`flatMapPublisher`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmappublisher)
*   [`flatMapSingle`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmapsingle)
*   [`flatMapSingleElement`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#flatmapsingleelement)


以下介绍我们就直接具体实现，中间流程请参考 [RxJava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)。

#### flatMap

* 官方示例：
    ```
    Observable.just("A", "B", "C")
        .flatMap(new Function<String, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(final String a) throws Exception {
                return Observable.intervalRange(1, 3, 0, 1, TimeUnit.SECONDS)
                        .map(new Function<Long, String>() {
                            @Override
                            public String apply(Long b) throws Exception {
                                return '(' + a + ", " + b + ')';
                            }
                        });
            }
        })
        .blockingSubscribe(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                System.out.println(o);
            }
        });
    ```
    输出：
    ```
    (A, 1)
    (B, 1)
    (C, 1)
    (C, 2)
    (B, 2)
    (A, 2)
    (A, 3)
    (B, 3)
    (C, 3)
    ```
* 返回对象的 `ObservableFlatMap` 的 `subscribeActual` 方法：
    ```
    public void subscribeActual(Observer<? super U> t) {
        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }
        source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }
    ```
* `MergeObserver` 的 `onNext` 方法：
    ```
    public void onNext(T t) {
        // safeguard against misbehaving sources
        ...
        subscribeInner(p);
    }

    void subscribeInner(ObservableSource<? extends U> p) {
        for (;;) {
            if (p instanceof Callable) {
               ...
                        drain();
               ...
            } else {
                InnerObserver<T, U> inner = new InnerObserver<T, U>(this, uniqueId++);
                if (addInner(inner)) {
                    p.subscribe(inner);
                }
                break;
            }
        }
    }
    ```
* `InnerObserver` 的 `onNext` 方法：
    ```
    public void onNext(U t) {
        if (fusionMode == QueueDisposable.NONE) {
            parent.tryEmit(t, this);
        } else {
            parent.drain();
        }
    }
    ```
* `MergeObserver` 的 `drain()` 方法：
    ```
    void drain() {
        if (getAndIncrement() == 0) {
            drainLoop();
        }
    }

    void drainLoop() {
        final Observer<? super U> child = this.downstream;//1.0
        int missed = 1;
        for (;;) {
            ...
            SimplePlainQueue<U> svq = queue;
            if (svq != null) {
                for (;;) {
                    ...
                    U o = svq.poll();
                    if (o == null) {
                        break;
                    }
                    child.onNext(o);//1.1
                }
            }
            ...
            if (d && (svq == null || svq.isEmpty()) && n == 0 && nSources == 0) {
                Throwable ex = errors.terminate();
                if (ex != ExceptionHelper.TERMINATED) {
                    if (ex == null) {
                        child.onComplete();//1.2
                    } else {
                        child.onError(ex);//1.3
                    }
                }
                return;
            }
            ...
        }
    }
    ```
    最终还是调用了 传进来的 `observer` 的 对应方法。

---

#### flatMapCompletable
官方示例：
```
Observable<Integer> source = Observable.just(2, 1, 3);
Completable completable = source.flatMapCompletable(new Function<Integer, CompletableSource>() {
    @Override
    public CompletableSource apply(final Integer x) throws Exception {
        return Completable.timer(x, TimeUnit.SECONDS)
                .doOnComplete(new Action() {
                    @Override
                    public void run() throws Exception {
                        System.out.println("Info: Processing of item \"" + x + "\" completed");
                    }
                });
    }
});
completable.doOnComplete(
        new Action() {
            @Override
            public void run() throws Exception {
                System.out.println("Info: Processing of all items completed");
            }
        })
        .blockingAwait();
```
输出：
```
Info: Processing of item "1" completed
Info: Processing of item "2" completed
Info: Processing of item "3" completed
Info: Processing of all items completed
```

#### flatMapIterable
官方示例：
```
Observable.just(1, 2, 3, 4)
        .flatMapIterable(new Function<Integer, Iterable<?>>() {
            @Override
            public Iterable<?> apply(Integer integer) throws Exception {
                switch (integer % 4) {
                    case 1:
                        return Arrays.asList("A");
                    case 2:
                        return Arrays.asList("B", "B");
                    case 3:
                        return Arrays.asList("C", "C", "C");
                    default:
                        return Arrays.asList();
                }
            }
        })
        .subscribe(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                System.out.println(o);
            }
        });
```
输出：
```
A
B
B
C
C
C
```

---

#### flatMapMaybe
官方示例：
```
Observable.just(9.0, 16.0, -4.0)
        .flatMapMaybe(new Function<Double, MaybeSource<?>>() {
            @Override
            public MaybeSource<?> apply(Double x) throws Exception {
                if (x.compareTo(0.0) < 0) {
                    return Maybe.empty();
                } else {
                    return Maybe.just(Math.sqrt(x));
                }
            }
        })
        .subscribe(new Observer<Object>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(Object o) {
                System.out.println(o);
            }

            @Override
            public void onError(Throwable e) {
                e.printStackTrace();
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete");
            }
        });
```
输出：
```
3.0
4.0
onComplete
```

---

#### flatMapObservable
官方示例：
```
Single<String> source = Single.just("Kirk, Spock, Chekov, Sulu");
Observable<String> names = source.flatMapObservable(new Function<String, ObservableSource<? extends String>>() {
    @Override
    public ObservableSource<? extends String> apply(String text) throws Exception {
        return Observable.fromArray(text.split(","))
                .map(new Function<String, String>() {
                    @Override
                    public String apply(String s) throws Exception {
                        return s;
                    }
                });
    }
});
names.subscribe(new Consumer<String>() {
    @Override
    public void accept(String name) throws Exception {
        System.out.println("onNext: " + name);
    }
});
```
输出：
```
onNext: Kirk
onNext:  Spock
onNext:  Chekov
onNext:  Sulu
```

---

#### flatMapPublisher
官方示例：
```
Single<String> source = Single.just("Kirk, Spock, Chekov, Sulu");
Flowable<String> names = source.flatMapPublisher(new Function<String, Publisher<? extends String>>() {
    @Override
    public Publisher<? extends String> apply(String text) throws Exception {
        return Flowable.fromArray(text.split(","))
                .map(new Function<String, String>() {
                    @Override
                    public String apply(String s) throws Exception {
                        return s;
                    }
                });
    }
});
names.subscribe(new Consumer<String>() {
    @Override
    public void accept(String name) throws Exception {
        System.out.println("onNext: " + name);
    }
});
```
输出：
```
onNext: Kirk
onNext:  Spock
onNext:  Chekov
onNext:  Sulu
```

---

#### flatMapSingle
官方示例：
```
Observable.just(4, 2, 1, 3)
        .flatMapSingle(new Function<Integer, SingleSource<?>>() {
            @Override
            public SingleSource<?> apply(final Integer integer) throws Exception {
                return Single.timer(integer, TimeUnit.SECONDS)
                        .map(new Function<Long, Object>() {
                            @Override
                            public Object apply(Long aLong) throws Exception {
                                return integer;
                            }
                        });
            }
        })
        .blockingSubscribe(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                System.out.println(o);
            }
        });
```
输出：
```
1
2
3
4
```

---

#### flatMapSingleElement
官方示例：
```
Maybe<Integer> source = Maybe.just(-42);
Maybe<Integer> result = source.flatMapSingleElement(new Function<Integer, SingleSource<? extends Integer>>() {
    @Override
    public SingleSource<? extends Integer> apply(Integer x) throws Exception {
        return Single.just(Math.abs(x));
    }
});

result.subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        System.out.println(integer);
    }
});
```
输出：
```
42
```

以上
