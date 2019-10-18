# RxJava之concatMap系列转换操作符源码介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


### 转换相关的操作符 以及 官方介绍
`RxJava` 之 `concatMap` 系列 **转换操作符** 官方介绍 ：[Transforming Observables](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmap)
*   [`concatMap`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmap)
*   [`concatMapCompletable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapcompletable)
*   [`concatMapCompletableDelayError`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapcompletabledelayerror)
*   [`concatMapDelayError`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapdelayerror)
*   [`concatMapEager`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapeager)
*   [`concatMapEagerDelayError`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapeagerdelayerror)
*   [`concatMapIterable`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapiterable)
*   [`concatMapMaybe`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapmaybe)
*   [`concatMapMaybeDelayError`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapmaybedelayerror)
*   [`concatMapSingle`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapsingle)
*   [`concatMapSingleDelayError`](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables#concatmapsingledelayerror)



以下介绍我们就直接具体实现，中间流程请参考 [RxJava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11)。


#### concatMap
* 官方示例：
    ```
    Observable.range(0, 5)
            .concatMap(i -> {
                long delay = Math.round(Math.random() * 2);

                return Observable.timer(delay, TimeUnit.SECONDS).map(n -> i);
            })
            .blockingSubscribe(System.out::print);
    ```
    输出：
    ```
    01234
    ```
* 返回对象的 `ObservableConcatMap` 的 `subscribeActual` 方法：
   单参数的`concatMap`操作符默认的 `delayErrors` 为 `ErrorMode.IMMEDIATE`。
    ```
    public void subscribeActual(Observer<? super U> observer) {
        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, observer, mapper)) {
            return;
        }
        if (delayErrors == ErrorMode.IMMEDIATE) {
            SerializedObserver<U> serial = new SerializedObserver<U>(observer);
            source.subscribe(new SourceObserver<T, U>(serial, mapper, bufferSize));
        } else {
            source.subscribe(new ConcatMapDelayErrorObserver<T, U>(observer, mapper, bufferSize, delayErrors == ErrorMode.END));
        }
    }
    ```
* 继续看 `SourceObserver` 的 `onNext(T t)`：
    ```
    public void onNext(T t) {
        if (done) {
            return;
        }
        if (fusionMode == QueueDisposable.NONE) {
            queue.offer(t);
        }
        drain();
    }

    public void onComplete() {
        if (done) {
            return;
        }
        done = true;
        drain();
    }

    void drain() {
        ...
        for (;;) {
            ...
            if (!active) {
                ...
                if (!empty) {
                    ObservableSource<? extends U> o;
                    try {
                        //1.0
                        o = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        dispose();
                        queue.clear();
                        downstream.onError(ex);
                        return;
                    }
                    active = true;
                    o.subscribe(inner);//2.0
                }
            }
            ...
        }
    }
    ```
  * `(1.0)` 在这里我们看到通过`concatMap`操作符传入`Function`的 `apply`重新构建了一个 `ObservableSource`对象。
  * `(2.0)` 然后新建的 `ObservableSource`对象来 `subscribe(observer)`。

---

#### concatMapXXX
`concatMapCompletable`、`concatMapCompletableDelayError`、`concatMapDelayError`、`concatMapEager`、`concatMapEagerDelayError`、`concatMapIterable`、`concatMapMaybe`、`concatMapMaybeDelayError`、`concatMapSingle`、`concatMapSingleDelayError`
实现逻辑和`concatMap`类似，就不再赘述了。

**官方示例：**
* `concatMapCompletable`
    ```
    Observable<Integer> source = Observable.just(2, 1, 3);
    Completable completable = source.concatMapCompletable(x -> {
        return Completable.timer(x, TimeUnit.SECONDS)
                .doOnComplete(() -> System.out.println("Info: Processing of item \"" + x + "\" completed"));
    });
    completable.doOnComplete(() -> System.out.println("Info: Processing of all items completed"))
            .blockingAwait();
    ```
    输出：
    ```
    Info: Processing of item "2" completed
    Info: Processing of item "1" completed
    Info: Processing of item "3" completed
    Info: Processing of all items completed
    ```

* `concatMapCompletableDelayError`
    ```
    Observable<Integer> source = Observable.just(2, 1, 3);
    Completable completable = source.concatMapCompletableDelayError(x -> {
        if (x.equals(2)) {
            return Completable.error(new IOException("Processing of item \"" + x + "\" failed!"));
        } else {
            return Completable.timer(1, TimeUnit.SECONDS)
                    .doOnComplete(() -> System.out.println("Info: Processing of item \"" + x + "\" completed"));
        }
    });

    completable.doOnError(error -> System.out.println("Error: " + error.getMessage()))
            .onErrorComplete()
            .blockingAwait();
    ```
    输出：
    ```
    Info: Processing of item "1" completed
    Info: Processing of item "3" completed
    Error: Processing of item "2" failed!
    ```
* `concatMapDelayError`
    ```
    Observable.intervalRange(1, 3, 0, 1, TimeUnit.SECONDS)
            .concatMapDelayError(x -> {
                if (x.equals(1L))
                    return Observable.error(new IOException("Something went wrong!"));
                else return Observable.just(x, x * x);
            })
            .blockingSubscribe(
                    x -> System.out.println("onNext: " + x),
                    error -> System.out.println("onError: " + error.getMessage()));
    ```
    输出：
    ```
    onNext: 2
    onNext: 4
    onNext: 3
    onNext: 9
    onError: Something went wrong!
    ```
* `concatMapEager`
    ```
    Observable.range(0, 5)
            .concatMapEager(i -> {
                long delay = Math.round(Math.random() * 3);

                return Observable.timer(delay, TimeUnit.SECONDS)
                        .map(n -> i)
                        .doOnNext(x -> System.out.println("Info: Finished processing item " + x));
            })
            .blockingSubscribe(i -> System.out.println("onNext: " + i));
    ```
    输出：
    ```
    Info: Finished processing item 2
    Info: Finished processing item 3
    Info: Finished processing item 1
    Info: Finished processing item 0
    Info: Finished processing item 4
    onNext: 0
    onNext: 1
    onNext: 2
    onNext: 3
    onNext: 4
    ```
* `concatMapEagerDelayError`
    ```
    Observable<Integer> source = Observable.create(emitter -> {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onError(new Error("Fatal error!"));
    });

    source.doOnError(error -> System.out.println("Info: Error from main source " + error.getMessage()))
            .concatMapEagerDelayError(x -> {
                return Observable.timer(1, TimeUnit.SECONDS).map(n -> x)
                        .doOnSubscribe(it -> System.out.println("Info: Processing of item \"" + x + "\" started"));
            }, true)
            .blockingSubscribe(
                    x -> System.out.println("onNext: " + x),
                    error -> System.out.println("onError: " + error.getMessage()));
    ```
    输出：
    ```
    Info: Processing of item "1" started
    Info: Processing of item "2" started
    Info: Error from main source Fatal error!
    onNext: 1
    onNext: 2
    onError: Fatal error!
    ```
* `concatMapIterable`
    ```
    Observable.just("A", "B", "C")
            .concatMapIterable(item -> Arrays.asList(item, item, item))
            .subscribe(System.out::print);
    ```
    输出：
    ```
    AAABBBCCC
    ```
* `concatMapMaybe`
    ```
    Observable.just("5", "3,14", "2.71", "FF")
            .concatMapMaybe(v -> {
                return Maybe.fromCallable(() -> Double.parseDouble(v))
                        .doOnError(e -> System.out.println("Info: The value \"" + v + "\" could not be parsed."))
                        // Ignore values that can not be parsed.
                        .onErrorComplete();
            })
            .subscribe(x -> System.out.println("onNext: " + x));
    ```
    输出：
    ```
    onNext: 5.0
    Info: The value "3,14" could not be parsed.
    onNext: 2.71
    Info: The value "FF" could not be parsed.
    ```
* `concatMapMaybeDelayError`
    ```
    DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern("dd.MM.uuuu");
    Observable.just("04.03.2018", "12-08-2018", "06.10.2018", "01.12.2018")
            .concatMapMaybeDelayError(date -> {
                return Maybe.fromCallable(() -> LocalDate.parse(date, dateFormatter));
            })
            .subscribe(
                    localDate -> System.out.println("onNext: " + localDate),
                    error -> System.out.println("onError: " + error.getMessage()));
    ```
    输出：
    ```
    onNext: 2018-03-04
    onNext: 2018-10-06
    onNext: 2018-12-01
    onError: Text '12-08-2018' could not be parsed at index 2
    ```
* `concatMapSingle`
    ```
    Observable.just("5", "3,14", "2.71", "FF")
            .concatMapSingle(v -> {
                return Single.fromCallable(() -> Double.parseDouble(v))
                        .doOnError(e -> System.out.println("Info: The value \"" + v + "\" could not be parsed."))

                        // Return a default value if the given value can not be parsed.
                        .onErrorReturnItem(42.0);
            })
            .subscribe(x -> System.out.println("onNext: " + x));
    ```
    输出：
    ```
    onNext: 5.0
    Info: The value "3,14" could not be parsed.
    onNext: 42.0
    onNext: 2.71
    Info: The value "FF" could not be parsed.
    onNext: 42.0
    ```
* `concatMapSingleDelayError`
    ```
    DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern("dd.MM.uuuu");
    Observable.just("24.03.2018", "12-08-2018", "06.10.2018", "01.12.2018")
            .concatMapSingleDelayError(date -> {
                return Single.fromCallable(() -> LocalDate.parse(date, dateFormatter));
            })
            .subscribe(
                    localDate -> System.out.println("onNext: " + localDate),
                    error -> System.out.println("onError: " + error.getMessage()));
    ```
    输出：
    ```
    onNext: 2018-03-24
    onNext: 2018-10-06
    onNext: 2018-12-01
    onError: Text '12-08-2018' could not be parsed at index 2
    ```
