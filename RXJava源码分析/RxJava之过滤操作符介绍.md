>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


### 过滤相关的操作符 以及 官方介绍
`RxJava` 之 **过滤操作符** 官方介绍 ：[Filtering Observables](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables)
*   [`debounce`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#debounce)
*   [`distinct`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#distinct)
*   [`distinctUntilChanged`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#distinctuntilchanged)
*   [`elementAt`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#elementat)
*   [`elementAtOrError`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#elementatorerror)
*   [`filter`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#filter)
*   [`first`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#first)
*   [`firstElement`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#firstelement)
*   [`firstOrError`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#firstorerror)
*   [`ignoreElement`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#ignoreelement)
*   [`ignoreElements`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#ignoreelements)
*   [`last`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#last)
*   [`lastElement`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#lastelement)
*   [`lastOrError`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#lastorerror)
*   [`ofType`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#oftype)
*   [`sample`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#sample)
*   [`skip`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#skip)
*   [`skipLast`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#skiplast)
*   [`take`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#take)
*   [`takeLast`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#takelast)
*   [`throttleFirst`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#throttlefirst)
*   [`throttleLast`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#throttlelast)
*   [`throttleLatest`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#throttlelatest)
*   [`throttleWithTimeout`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#throttlewithtimeout)
*   [`timeout`](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables#timeout)

---


##### debounce
丢弃超过`debounce`设置的时间的事件

官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(1500);
        emitter.onNext("B");

        Thread.sleep(500);
        emitter.onNext("C");

        Thread.sleep(250);
        emitter.onNext("D");

        Thread.sleep(2000);
        emitter.onNext("E");
        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .debounce(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
onNext: A
onNext: D
onNext: E
onComplete
```

---

##### distinct
过滤相同的事件

官方示例：
```
Observable.just(2, 3, 4, 4, 2, 1)
        .distinct()
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
2
3
4
1
```

---


##### distinctUntilChanged
过滤连续的相同事件流

官方示例：
```
Observable.just(1, 1, 2, 1, 2, 3, 3, 4)
        .distinctUntilChanged()
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
1
2
1
2
3
4
```

---


##### elementAt
获取事件流中从零开始的第指定下标的元素

官方示例：
```
Observable.range(0, 10)
        .elementAt(5)
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
5
```

---


##### elementAtOrError
索引不存在则走`onError`

官方示例：
```
Observable<String> source = Observable.just("Kirk", "Spock", "Chekov", "Sulu");
Single<String> element = source.elementAtOrError(4);
element.subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        System.out.println("onSuccess will not be printed!");
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable throwable) throws Exception {
        System.out.println("onError: " + throwable);
    }
});
```
输出：
```
onError: java.util.NoSuchElementException
```

---


##### filter
自定义过滤规则

官方示例：
```
Observable.just(1, 2, 3, 4, 5, 6)
        .filter(new Predicate<Integer>() {
            @Override
            public boolean test(Integer integer) throws Exception {
                return integer % 2 == 0;
            }
        })
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
2
4
6
```

---


##### first
获取事件流中第一个事件，返回值为 `Single`

官方示例：
```
Observable<String> source = Observable.just("A", "B", "C");
Single<String> firstOrDefault = source.first("D");

firstOrDefault.subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        System.out.println(s);
    }
});
```
输出：
```
A
```

---


##### firstElement
获取事件流中第一个事件，返回值为 `Maybe`

官方示例：
```
Observable<String> source = Observable.just("A", "B", "C");
Maybe<String> firstOrDefault = source.firstElement();

firstOrDefault.subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        System.out.println(s);
    }
});
```
输出：
```
A
```

---


##### firstOrError
输出第一个事件并捕获异常。

官方示例：
```
Observable<String> emptySource = Observable.empty();
Single<String> firstOrError = emptySource.firstOrError();

firstOrError.subscribe(
        new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println("onSuccess will not be printed!");
            }
        },
        new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                System.out.println("onError: " + throwable);
            }
        });
```
输出：
```
onError: java.util.NoSuchElementException
```

---


##### ignoreElement
过滤一个事件

官方示例：
```
Single<Long> source = Single.timer(1, TimeUnit.SECONDS);
Completable completable = source.ignoreElement();

completable.doOnComplete(new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Done!");
    }
}).blockingAwait();
```
输出：
```
Done!
```

---


##### ignoreElements
过滤所有事件

官方示例：
```
Observable<Long> source = Observable.intervalRange(1, 5, 1, 1, TimeUnit.SECONDS);
Completable completable = source.ignoreElements();

completable.doOnComplete(new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Done!");
    }
}).blockingAwait();
```
输出：
```
Done!
```

---


##### last
获取事件流中最后一个事件，返回值为 `Single`

官方示例：
```
Observable<String> source = Observable.just("A", "B", "C");
Single<String> lastOrDefault = source.last("D");

lastOrDefault.subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        System.out.println(s);
    }
});
```
输出：
```
C
```

---


##### lastElement
获取事件流中最后一个事件， 返回值为 `Maybe`

官方示例：
```
Observable<String> source = Observable.just("A", "B", "C");
Maybe<String> lastOrDefault = source.lastElement();

lastOrDefault.subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        System.out.println(s);
    }
});
```
输出：
```
C
```

---


##### lastOrError
同 `firstOrError`

官方示例：
```
Observable<String> emptySource = Observable.empty();
Single<String> lastOrError = emptySource.lastOrError();

lastOrError.subscribe(
        new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println("onSuccess will not be printed!");
            }
        },
        new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                System.out.println("onError: " + throwable);
            }
        });
```
输出：
```
onError: java.util.NoSuchElementException
```

---


##### ofType
根据类型过滤

官方示例：
```
Observable<Number> numbers = Observable.just(1, 4.0, 3, 2.71, 2f, 7);
Observable<Integer> integers = numbers.ofType(Integer.class);

integers.subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        System.out.println(integer);
    }
});
```
输出：
```
1
3
7
```

---


##### sample
仅在周期性时间间隔内发出最近发出的事件来过滤事件流中的事件。

官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(500);
        emitter.onNext("B");

        Thread.sleep(200);
        emitter.onNext("C");

        Thread.sleep(800);
        emitter.onNext("D");

        Thread.sleep(600);
        emitter.onNext("E");
        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .sample(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
// 700(500 + 200)   
//1500(500 + 200 + 800)   
//2100（500 + 200 + 800 + 600）
onNext: C
onNext: D
onComplete
```

---


##### skip
跳过事件流中开头的指定个数事件

官方示例：
```
Observable<Integer> source = Observable.just(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

source.skip(4)
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
5
6
7
8
9
10
```

---


##### skipLast
跳过事件流中结尾的指定个数事件

官方示例：
```
Observable<Integer> source = Observable.just(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

source.skipLast(4)
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
1
2
3
4
5
6
```

---


##### take
取事件流中开头的指定个数事件

官方示例：
```
Observable<Integer> source = Observable.just(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

source.take(4)
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
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


##### takeLast
取事件流中结尾的指定个数事件

官方示例：
```
Observable<Integer> source = Observable.just(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

source.takeLast(4)
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
7
8
9
10
```

---


##### throttleFirst
和 `sample`相反   去指定连续时间内的第一个事件

官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(500);
        emitter.onNext("B");

        Thread.sleep(200);
        emitter.onNext("C");

        Thread.sleep(800);
        emitter.onNext("D");

        Thread.sleep(600);
        emitter.onNext("E");
        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .throttleFirst(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
onNext: A
onNext: D
onComplete
```

---


##### throttleLast
和 `sample`一样   去指定连续时间内的最后一个事件

官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(500);
        emitter.onNext("B");

        Thread.sleep(200);
        emitter.onNext("C");

        Thread.sleep(800);
        emitter.onNext("D");

        Thread.sleep(600);
        emitter.onNext("E");
        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .throttleLast(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
onNext: C
onNext: D
onComplete
```

---


##### throttleLatest
发出事件流中的事件，然后在它们之间经过指定的超时时定期发出最新项目（如果有）。

官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(500);
        emitter.onNext("B");

        Thread.sleep(200);
        emitter.onNext("C");

        Thread.sleep(800);
        emitter.onNext("D");

        Thread.sleep(600);
        emitter.onNext("E");
        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .throttleLatest(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
onNext: A
onNext: C
onNext: D
onComplete
```

---


##### throttleWithTimeout
官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(1500);
        emitter.onNext("B");

        Thread.sleep(500);
        emitter.onNext("C");

        Thread.sleep(250);
        emitter.onNext("D");

        Thread.sleep(2000);
        emitter.onNext("E");

        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .throttleWithTimeout(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
onNext: A
onNext: D
onNext: E
onComplete
```

---


##### timeout
在超时时间内发出每一个事件，如果超过超时事件则报错

官方示例：
```
Observable<String> source = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("A");

        Thread.sleep(800);
        emitter.onNext("B");

        Thread.sleep(400);
        emitter.onNext("C");

        Thread.sleep(1200);
        emitter.onNext("D");

        emitter.onComplete();
    }
});

source.subscribeOn(Schedulers.io())
        .timeout(1, TimeUnit.SECONDS)
        .blockingSubscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
            }

            @Override
            public void onNext(String s) {
                System.out.println("onNext: " + s);
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
onNext: A
onNext: B
onNext: C
java.util.concurrent.TimeoutException: The source did not signal an event for 1 seconds and has been terminated.
```

---

以上



